<template>
  <div>
    <p class="qr_import_info">
      {{ i18n.import_backup_qr_in_batches }}
    </p>

    <div class="qr_camera">
      <a-button v-if="!cameraActive" @click="startCamera">
        {{ i18n.import_camera_scan }}
      </a-button>
      <div v-show="cameraActive" class="qr_camera_view">
        <video
          ref="video"
          class="qr_camera_video"
          autoplay
          muted
          playsinline
        ></video>
        <p class="qr_camera_status">{{ statusMessage }}</p>
        <div class="qr_camera_actions">
          <a-button :disabled="!scannedCodes.length" @click="importScanned">
            {{ i18n.import_camera_finish }}
          </a-button>
          <a-button @click="stopCamera">
            {{ i18n.import_camera_stop }}
          </a-button>
        </div>
      </div>
    </div>

    <a-file-input
      button-type="file"
      accept="image/*"
      @change="importQr($event, true)"
      multiple="true"
      :label="i18n.import_backup_qr"
    />
  </div>
</template>
<script lang="ts">
import Vue from "vue";
// @ts-ignore
import QRCode from "qrcode-reader";
import jsQR from "jsqr";
import { getEntryDataFromOTPAuthPerLine } from "../../import";
import { EntryStorage } from "../../models/storage";
import { Encryption } from "../../models/encryption";

interface QrImportData {
  cameraActive: boolean;
  cameraError: string;
  scannedCodes: string[];
  mediaStream: MediaStream | null;
  scanCanvas: HTMLCanvasElement | null;
  animationFrameId: number | null;
}

export default Vue.extend({
  data(): QrImportData {
    return {
      cameraActive: false,
      cameraError: "",
      scannedCodes: [],
      mediaStream: null,
      scanCanvas: null,
      animationFrameId: null,
    };
  },
  computed: {
    statusMessage(): string {
      if (this.cameraError) {
        return this.cameraError;
      }
      if (this.scannedCodes.length) {
        return `${this.i18n.import_camera_found}: ${this.scannedCodes.length}`;
      }
      return this.i18n.import_camera_scanning;
    },
  },
  beforeDestroy() {
    this.stopCamera();
  },
  methods: {
    async importQr(event: Event, closeWindow: Boolean) {
      const target = event.target as HTMLInputElement;
      if (!target || !target.files) {
        return;
      }
      if (target.files.length) {
        const otpUrlList: string[] = [];
        let hasFailedResults = false;
        for (let fileIndex = 0; fileIndex < target.files.length; fileIndex++) {
          const file = target.files[fileIndex];
          const otpUrl = await getOtpUrlFromQrFile(file);
          if (otpUrl !== null) {
            otpUrlList.push(otpUrl);
          } else {
            hasFailedResults = true;
          }
        }

        await this.importOtpUrlList(otpUrlList, hasFailedResults, closeWindow);
      } else {
        alert(this.i18n.updateFailure);
        if (closeWindow) {
          window.alert(this.i18n.updateFailure);
          window.close();
        }
      }
      return;
    },

    async importOtpUrlList(
      otpUrlList: string[],
      hasFailedResults: boolean,
      closeWindow: Boolean
    ) {
      const result = await getEntryDataFromOTPAuthPerLine(
        otpUrlList.join("\n")
      );

      const importData: {
        // @ts-ignore
        key?: { enc: string; hash: string };
        [hash: string]: RawOTPStorage;
      } = result.exportData;

      const { failedCount, succeededCount } = result;

      const decryptedFileData: { [hash: string]: RawOTPStorage } = importData;

      if (Object.keys(decryptedFileData).length) {
        await EntryStorage.import(
          this.$encryption as Encryption,
          decryptedFileData
        );

        if (hasFailedResults) {
          alert(this.i18n.import_backup_qr_partly_failed);
        } else if (failedCount && succeededCount) {
          alert(this.i18n.migration_partly_fail);
        } else if (succeededCount) {
          alert(this.i18n.updateSuccess);
        } else {
          alert(this.i18n.migration_fail);
        }

        if (closeWindow) {
          window.close();
        }
      } else {
        alert(this.i18n.errorqr);
        if (closeWindow) {
          window.close();
        }
      }
      return;
    },

    async startCamera() {
      this.cameraError = "";
      this.scannedCodes = [];

      if (
        !navigator.mediaDevices ||
        typeof navigator.mediaDevices.getUserMedia !== "function"
      ) {
        this.cameraError = this.i18n.import_camera_unsupported;
        return;
      }

      let stream: MediaStream;
      try {
        stream = await navigator.mediaDevices.getUserMedia({
          audio: false,
          video: {
            facingMode: "environment",
            width: { ideal: 1280 },
            height: { ideal: 720 },
          },
        });
      } catch (error) {
        console.error(error);
        this.cameraError = getCameraErrorMessage(error, this.i18n);
        return;
      }

      const video = (this.$refs.video as unknown) as
        | HTMLVideoElement
        | undefined;
      if (!video) {
        stopStream(stream);
        this.cameraError = this.i18n.import_camera_error;
        return;
      }

      this.mediaStream = stream;
      video.srcObject = stream;
      this.cameraActive = true;

      try {
        await video.play();
      } catch (error) {
        // Autoplay can be rejected while the element is not visible yet. The
        // scan loop below still picks up frames once the video starts playing.
        console.error(error);
      }

      this.scanFrame();
    },

    scanFrame() {
      if (!this.cameraActive) {
        return;
      }

      this.animationFrameId = window.requestAnimationFrame(() =>
        this.scanFrame()
      );

      const video = (this.$refs.video as unknown) as
        | HTMLVideoElement
        | undefined;
      if (!video || video.readyState !== video.HAVE_ENOUGH_DATA) {
        return;
      }

      let canvas = this.scanCanvas;
      if (!canvas) {
        canvas = document.createElement("canvas");
        this.scanCanvas = canvas;
      }

      canvas.width = video.videoWidth;
      canvas.height = video.videoHeight;
      const ctx = canvas.getContext("2d");
      if (!ctx) {
        return;
      }

      ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
      const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
      const code = jsQR(imageData.data, imageData.width, imageData.height, {
        inversionAttempts: "dontInvert",
      });

      if (code && code.data && isOtpAuthCode(code.data)) {
        const otpUrl = code.data.trim();
        if (this.scannedCodes.indexOf(otpUrl) === -1) {
          this.scannedCodes.push(otpUrl);
        }
      }
    },

    async importScanned() {
      if (!this.scannedCodes.length) {
        return;
      }
      const otpUrlList = this.scannedCodes.slice();
      this.stopCamera();
      await this.importOtpUrlList(otpUrlList, false, false);
    },

    stopCamera() {
      this.cameraActive = false;

      if (this.animationFrameId !== null) {
        window.cancelAnimationFrame(this.animationFrameId);
        this.animationFrameId = null;
      }

      const video = (this.$refs.video as unknown) as
        | HTMLVideoElement
        | undefined;
      if (video) {
        video.pause();
        video.srcObject = null;
      }

      stopStream(this.mediaStream);
      this.mediaStream = null;
    },
  },
});

function stopStream(stream: MediaStream | null) {
  if (!stream) {
    return;
  }
  for (const track of stream.getTracks()) {
    track.stop();
  }
}

function isOtpAuthCode(text: string): boolean {
  const value = text.trim();
  return (
    value.indexOf("otpauth://") === 0 ||
    value.indexOf("otpauth-migration://") === 0
  );
}

function getCameraErrorMessage(
  error: unknown,
  i18n: { [key: string]: string }
): string {
  const name =
    error && typeof error === "object" && "name" in error
      ? String((error as { name: unknown }).name)
      : "";

  switch (name) {
    case "NotAllowedError":
    case "PermissionDeniedError":
    case "SecurityError":
      return i18n.import_camera_denied;
    case "NotFoundError":
    case "DevicesNotFoundError":
      return i18n.import_camera_not_found;
    default:
      return i18n.import_camera_error;
  }
}

async function getOtpUrlFromQrFile(file: File): Promise<string | null> {
  return new Promise((resolve) => {
    const reader = new FileReader();
    reader.onload = () => {
      const imageUrl = reader.result as string;
      const qrReader = new QRCode();
      qrReader.callback = (
        error: string,
        text: {
          result: string;
          points: Array<{
            x: number;
            y: number;
            count: number;
            estimatedModuleSize: number;
          }>;
        }
      ) => {
        if (error) {
          console.error(error);

          const image: HTMLImageElement = document.createElement("img");
          image.onload = () => {
            const canvas: HTMLCanvasElement = document.createElement("canvas");
            const ctx: CanvasRenderingContext2D = canvas.getContext(
              "2d"
            ) as CanvasRenderingContext2D;

            canvas.width = image.width;
            canvas.height = image.height;
            ctx.drawImage(image, 0, 0);

            const qrImageData = ctx.getImageData(
              0,
              0,
              canvas.width,
              canvas.height
            );
            const jsQrCode = jsQR(
              qrImageData.data,
              canvas.width,
              canvas.height
            );

            if (jsQrCode && jsQrCode.data) {
              if (!isOtpAuthCode(jsQrCode.data)) {
                return resolve(null);
              }
              return resolve(jsQrCode.data);
            } else {
              return resolve(null);
            }
          };
          image.src = imageUrl;
        } else {
          if (!isOtpAuthCode(text.result)) {
            return resolve(null);
          }
          return resolve(text.result);
        }
      };
      qrReader.decode(imageUrl);
    };
    reader.readAsDataURL(file);
  });
}
</script>
