# Camera-based QR import — feasibility assessment & implementation

**Repo:** `Avinash99b/Authenticator` (fork of Authenticator-Extension/Authenticator, branch `dev`, v8.0.2)
**Date:** 2026-11-18

## Verdict

**Feasible — and implemented.** Importing accounts straight from the device camera is
supported by the platform, needs **no new extension permissions**, and reuses libraries
the project already ships. Additions: 3 files, ~340 lines.

## How importing worked before

| Path | Where | Mechanism |
| --- | --- | --- |
| QR image backup | `src/components/Import/QrImport.vue` | `<input type="file" accept="image/*">` (cropped screenshot) → `qrcode-reader`, with a `jsQR` fallback on a canvas |
| Tab capture overlay | `src/content.ts` + `background.ts` | Injected overlay; user drags a box over the page, `chrome.tabs.captureVisibleTab` crops it |
| Text / file backup | `TextImport.vue`, `FileImport.vue` | `otpauth://` lines, backup files |

So QR acquisition was always a **still image** — either a screenshot the user had to crop
by hand, or a region of the current tab. There was no camera path anywhere in the codebase.

## Feasibility findings

1. **The import page is a valid context for `getUserMedia`.** `view/import.html` is opened
   as a normal extension tab (`window.open("import.html?QrImport", "_blank")` from the popup).
   Extension pages are top-level documents on a secure `chrome-extension://` (or
   `moz-extension://`) origin, which is exactly the context browsers allow to prompt for a
   camera. The Chrome extensions samples issue #821 documents the same pattern: a
   `getUserMedia()` call from a visible extension page triggers the "…wants to use your
   camera" prompt and grants the origin's media permission.
2. **No manifest change is required.** Chrome/Edge MV3 and Firefox MV3 have **no camera
   manifest permission**; consent is a runtime, per-origin browser prompt. That means: no new
   install-time warning, no change to the CWS/AMO listing and no store review impact.
   (`videoCapture` is a Chrome *Apps* permission, not an extension one.)
3. **Offscreen documents are the wrong place** — MV3 offscreen documents cannot call
   `getUserMedia` until a visible extension page has been granted the permission. The scanner
   is therefore implemented in the import page itself, which is already a visible tab.
4. **`jsQR` is already a dependency and already used by this repo** (`content.ts`,
   `QrImport.vue`). `jsQR` decodes from raw RGBA frame data, which is the standard way to scan
   live video: `getUserMedia` → `video.srcObject` → canvas frame → `jsQR(imageData.data, …)`.
   Nothing new to add to `package.json`.
5. **The extension's strict CSP does not block the preview.** The extension pages use
   `default-src 'none'`. A camera preview attached via `video.srcObject = stream` loads no
   URL, so `media-src` does not apply to it — no CSP change needed.
6. **Firefox works from a tab-bound extension page.** Mozilla's own guidance is that
   `getUserMedia` from an extension page opened in a tab works (it is bound to that tab being
   open — which is precisely the import page's situation).

**Overall risk: low.** No new permissions, no new dependencies, no CSP relaxation.

## What was implemented

### `src/components/Import/QrImport.vue`
* **"Scan with Camera"** button starts `navigator.mediaDevices.getUserMedia({ video: { facingMode: "environment", width/height } , audio: false })`.
* Live preview in a `<video>` (`autoplay muted playsinline`) inside a `v-show` container.
* A `requestAnimationFrame` loop draws frames onto one reusable canvas and calls `jsQR`
  (with `inversionAttempts: "dontInvert"` for speed).
* Recognised `otpauth://` / `otpauth-migration://` payloads are collected and **de-duplicated**,
  so a user can scan several codes in one session (matches the existing batch-import UX).
* Live status line: scanning hint → `QR codes recognized: n`.
* **"Import Scanned Codes"** and **"Stop Camera"** buttons; the import button is disabled until
  something has been recognised.
* Explicit error mapping: permission denied (`NotAllowedError`/`PermissionDeniedError`/
  `SecurityError`), no camera (`NotFoundError`), `getUserMedia` unavailable → clear i18n messages.
* Correct teardown: cancels the animation frame, pauses the video, clears `srcObject`, stops
  every media track — on stop *and* on `beforeDestroy`. No camera light left on.
* The image-file path was **refactored, not replaced**: both paths now share
  `importOtpUrlList()`, which runs the existing `getEntryDataFromOTPAuthPerLine` +
  `EntryStorage.import` logic and the existing success/partial/failure alerts. Existing file
  import behaviour is byte-for-byte equivalent.

### `_locales/en/messages.json`
Nine new strings (`import_camera_scan`, `_stop`, `_scanning`, `_found`, `_finish`,
`_unsupported`, `_denied`, `_not_found`, `_error`). Only English is authored; every other
locale falls back to English automatically through `chrome.i18n.getMessage`, so the UI is
never blank and translators can pick the keys up via Crowdin later.

### `sass/import.scss`
Styling for `.qr_camera`, `.qr_camera_view`, `.qr_camera_video`, `.qr_camera_status`,
`.qr_camera_actions`, and `.qr_import_info`.

## Verification performed

| Check | Command | Result |
| --- | --- | --- |
| Lint | `eslint . --ext .js,.ts` | clean |
| Format | `prettier --check src/…` | clean (repo-configured Prettier 2.2.1) |
| Type-check + bundle | `CI=1 bash scripts/build.sh chrome` | **`webpack 5.94.0 compiled successfully`**, `Done!`; ForkTsChecker reports **no** errors |
| Bundle contents | `grep` in `dist/import.js` | `getUserMedia` and camera i18n keys present |
| Locales in build | `chrome/_locales/en/messages.json` | all 9 camera keys present |
| Styles compiled | `css/import.css` | camera rules present |

Two build-time type errors were found and fixed during development (`vue` `data()` inference
needed an explicit `QrImportData` interface; a narrowed canvas/stream reference needed a local).

**Not verified:** the camera behaving on real hardware. A headless end-to-end test was
attempted (fake camera fed a generated `otpauth://` QR code via
`--use-file-for-fake-video-capture`), but Chrome could not start in this container
(`libatk-1.0.so.0` missing), so a quick manual check with a real webcam is still recommended.

## Notes / limits

* Camera requires the user to grant consent once per origin; if denied, the browser's site
  settings for the extension (`chrome://settings/content/camera`) must be changed and the page
  says so.
* Images embedded in the page (the old capture overlay) still work unchanged.
* A physical camera and a user gesture are both required — no silent capture.
* Batch scanning collects only *distinct* codes; scanning the same QR twice adds one entry.

## Suggested follow-up (optional)

1. Surface a camera entry point from the add-account page (`AddMethodPage.vue`) alongside
   "Scan QR Code".
2. Use `BarcodeDetector` when available and fall back to `jsQR`.
3. Upstream the change to `Authenticator-Extension/Authenticator` if desired.
