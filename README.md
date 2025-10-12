<div align="center">
  <h2>ODIM Android</h2>
  <p>Operationalized Digital Interaction Mining for Android — capture, curate, and upload interaction traces for research.</p>
  
  <p>
    <a href="https://shields.io/">
      <img src="https://img.shields.io/badge/platform-Android-3DDC84?logo=android&logoColor=white" alt="platform" />
    </a>
    <a>
      <img src="https://img.shields.io/badge/minSdk-30-blue" alt="minSdk" />
    </a>
    <a>
      <img src="https://img.shields.io/badge/targetSdk-33-blue" alt="targetSdk" />
    </a>
    <a>
      <img src="https://img.shields.io/badge/AGP-8.6.x-3DDC84?logo=android" alt="AGP" />
    </a>
    <a>
      <img src="https://img.shields.io/badge/Kotlin-1.7.x-7F52FF?logo=kotlin&logoColor=white" alt="Kotlin" />
    </a>
    <a href="LICENSE">
      <img src="https://img.shields.io/badge/license-NCSA-green" alt="license" />
    </a>
  </p>
  
  <p>
    <a href="#about">About</a>
    · <a href="#getting-started">Getting Started</a>
    · <a href="#configuration">Configuration</a>
    · <a href="#storage-layout">Storage</a>
    · <a href="#privacy--permissions">Privacy</a>
    · <a href="#networking">Networking</a>
  </p>
</div>


## About

ODIM (Operationalized Digital Interaction Mining) is an Android app that captures user interactions across installed apps to generate traces for research and analysis. It runs an AccessibilityService to observe interaction events and, on each relevant touch, records:

- a full-screen screenshot
- the view hierarchy (VH) at that moment as JSON
- a lightweight gesture record describing where and how the user interacted

Captured data is organized locally on-device and can be reviewed, curated (redact sensitive UI text), edited (fix incomplete gestures), and uploaded to a server.


## Highlights

- Capture cross-app interaction traces with screenshots, view hierarchy, and gestures
- Curate traces: redact sensitive UI text/regions and fix incomplete interactions
- Associate traces with capture tasks via QR codes
- Split/merge flows and upload full traces to a server


## Tech stack

- Android, Kotlin, Coroutines
- AndroidX, RecyclerView, ViewBinding
- CameraX and ML Kit (barcode-scanning) for QR
- OkHttp (+ logging), Jackson (Kotlin module) for networking/JSON


## Project layout

Top-level layout (this directory):

- `app/` – the main Android application module
- `gradle/`, `gradlew*`, `settings.gradle`, `build.gradle`, `LICENSE`, `CODEOWNERS`

Module structure (`app/src/main/java/edu/illinois/odim`):

- `activities/` – entry points and screens for browsing, editing, and uploading captures
- `adapters/` – RecyclerView adapters for list UIs
- `dataclasses/` – data models persisted locally and exchanged with the server
- `fragments/` – custom views/overlays used by activities
- `utils/` – storage, upload, and screen dimension helpers
- `CoreAccessibilityService.kt` – the core AccessibilityService implementation (`MyAccessibilityService`)

Android resources live in `app/src/main/res/` (layouts, drawables, strings, menus, themes, etc.).


## How it works

1) Enable the ODIM accessibility service. When the user interacts with other apps, `MyAccessibilityService` listens for events and on first touch of a screen:

- takes a screenshot
- serializes the current view hierarchy tree to JSON
- infers an initial gesture label (click, long click, scroll, or unknown) and stores normalized coordinates
- groups screens into a "trace" per target app

2) Browse on-device captures in the ODIM app:

- `AppActivity` shows installed app packages that have captured traces
- `TraceActivity` shows traces for a selected app (label is the timestamp of first screen; task metadata is shown if present)
- `EventActivity` shows the sequence of captured screens (events) in a grid; from here you can:
  - open a screen to redact sensitive text or UI regions
  - fix incomplete gestures (when the service couldn’t infer a unique target)
  - split selected screens into a new trace
  - upload the full trace to the server

3) Redact or edit details:

- `ScreenShotActivity` overlays VH bounding boxes over the screenshot; you can redact text via a list of VH nodes or draw redaction rectangles and save them
- `IncompleteScreenActivity` helps resolve incomplete gesture events by selecting the intended VH node and defining the gesture type (click/long click/scroll)

4) Upload a trace:

- From `EventActivity`, ODIM uploads each screen (image + VH) and a metadata document describing gestures and redactions via `UploadDataOps`


## Getting started

Requirements:

- Android Studio (AGP 8.6.x), Kotlin 1.7.x
- minSdk 30, targetSdk 33, compileSdk 34

Steps:

1) Open this directory in Android Studio
2) Build the project and install the `app` module
3) In system settings, enable the ODIM accessibility service
4) Launch ODIM and scan a QR code in `CaptureActivity` to load a task (optional)
5) Use your device normally; ODIM will capture screens for non-ODIM apps you interact with


## Configuration

- Base URL is defined via `BuildConfig.API_URL_PREFIX`:
  - Debug: `${API_URL_PREFIX}` Gradle property or defaults to `https://dev.interactionmining.org`
  - Release: `${API_URL_PREFIX}` Gradle property or defaults to `https://interactionmining.org`
- `usesCleartextTraffic` is set via manifest placeholders (enabled for debug, disabled for release).

To override the base URL, set a Gradle property when building:

```
API_URL_PREFIX=https://your.server
```


## Storage layout

All captured data is stored under the app's internal storage, rooted at `files/TRACES`.

Directory pattern:

`TRACES/<appPackage>/<traceLabel>/<eventLabel>/`

Within each `eventLabel` directory:

- `<eventLabel>.png` – the screenshot
- `vh-<eventLabel>.json` – the view hierarchy at capture time
- `gesture-<eventLabel>.json` – the normalized gesture (when available)
- `redact-<eventLabel>.json` – accumulated redactions for this screen

Trace-level metadata:

- `TRACES/<appPackage>/<traceLabel>/task.json` – `CaptureStore { id, description }` set when the trace is associated with a capture task

Key helpers in `utils/LocalStorageOps.kt` provide file I/O for listing apps/traces/events, saving and renaming artifacts, and splitting selected events into a new trace.


## Activities

- `CaptureActivity` – starting point for ingesting a capture task by scanning a QR code; shows task instructions and shortcuts to enable accessibility and launch the target app
- `AppActivity` – lists apps that have captured traces; navigate into traces or multi-select to delete app data
- `TraceActivity` – lists traces for the chosen app; shows task description and number of screens; supports multi-select delete
- `EventActivity` – grid of events (screens) for a trace; supports edit gesture, split trace, delete screens, and upload trace
- `ScreenShotActivity` – redact screen contents using VH-driven text selection or freeform rectangles; updates both VH JSON and PNG
- `IncompleteScreenActivity` – resolve incomplete or unknown gestures: choose a VH candidate and gesture type; persists updated gesture and renames the event accordingly


## Accessibility service

- `MyAccessibilityService` (declared as a `<service>` in `AndroidManifest.xml`) captures:
  - package transitions to segment traces
  - screenshots and VH on touch
  - gestures from `AccessibilityEvent` to label interactions
  - writes to on-device storage via `LocalStorageOps`


## Privacy & permissions

- Permissions: `INTERNET`, `ACCESS_NETWORK_STATE` (uploads), `CAMERA` (QR), `QUERY_ALL_PACKAGES` (enumerate app directories for display)
- Data stays on-device under `files/TRACES` until the user explicitly uploads a trace
- Screenshots and VH are only captured for non-ODIM apps; system UI and the launcher are excluded


## Key packages and components

- `activities/` – `CaptureActivity`, `AppActivity`, `TraceActivity`, `EventActivity`, `ScreenShotActivity`, `IncompleteScreenActivity`
- `adapters/` – `AppAdapter`, `TraceAdapter`, `EventAdapter`, `VHAdapter`
- `dataclasses/` – `Gesture`, `Redaction`, `ScreenShotPreview`, `CaptureStore`, `CaptureTask`, etc.
- `fragments/` – overlays and floating widgets (e.g., `ScrubbingScreenshotOverlay`, `MovableFloatingActionButton`)
- `utils/` – `LocalStorageOps`, `UploadDataOps`, `ScreenDimensionsOps`


## Networking

- Uploads use `UploadDataOps` with base URL from `BuildConfig.API_URL_PREFIX`.
- Endpoints:
  - `POST /api/capture/{captureId}/upload/frames`
  - `POST /api/capture/{captureId}/upload/metadata`


## Acknowledgements

- CameraX (`androidx.camera:*`)
- Google ML Kit (Barcode Scanning)
- OkHttp (+ logging interceptor)
- Jackson (Kotlin module)


## License

University of Illinois/NCSA Open Source License. See `LICENSE`.
