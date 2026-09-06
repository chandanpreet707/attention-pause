# Privacy Policy — Attention Pause

_Last updated: September 2026_

Attention Pause does not collect, store, transmit, or share any personal data.

## Camera

The extension uses your webcam to determine whether you are looking at the
screen. Each frame is analysed on your device and discarded immediately. Frames
are never recorded, saved to disk, or transmitted anywhere.

The only values derived from a frame are two head angles and an eyes-open
estimate. These exist in memory for a fraction of a second and are not stored.
No images, biometric templates, or identifying information are retained.

## Data collection

None. Specifically, Attention Pause does not collect personally identifiable
information, health information, financial information, authentication
information, personal communications, location, web history, or user activity.
There is no analytics, no telemetry, and no crash reporting.

## Network

The extension makes no network requests of any kind. The face detection model
and runtime are bundled inside the extension package rather than downloaded, so
nothing is fetched at runtime.

## Local storage

Your preferences — theme, sensitivity thresholds, and whether the feature is on
— are stored locally in your browser using the Chrome storage API. They never
leave your device.

## Permissions

- **Camera** — to detect whether you are looking at the screen.
- **offscreen** — to run the camera and detection in a background document.
- **storage** — to save your preferences locally.
- **scripting** and **youtube.com host access** — to pause and resume video
  elements on pages you are watching.

## Revoking access

Camera access can be revoked at any time through Chrome's site settings. Without
it, the extension does nothing and video plays as normal.

## Contact

Questions can be raised as an issue on the project's GitHub repository.
