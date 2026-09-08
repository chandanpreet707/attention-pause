# Attention Pause

**[Install from the Chrome Web Store](https://chromewebstore.google.com/detail/attention-pause/emihckaonnohmpbnkgpbppkniboknloc)**

A Chrome extension that pauses video when you look away from the screen, and
resumes it when you look back. Everything runs on your own machine — frames are
measured and discarded, nothing is stored, and the extension makes no network
requests at all.

**Milestone 2**: webcam face detection with head direction and eye state. Pauses
when you leave, turn away, or close your eyes.

## Setup

The face detection model isn't in this repo — it's 3.7MB and not ours to
redistribute. Download it into `vendor/` before loading the extension:

```bash
curl -L -o vendor/face_landmarker.task \
  https://storage.googleapis.com/mediapipe-models/face_landmarker/face_landmarker/float16/1/face_landmarker.task
```

Then:

1. Open `chrome://extensions`, turn on **Developer mode**, click **Load unpacked**.
2. Click **Details** on the extension card, then **Extension options**.
3. Click **Turn on camera** and allow the prompt. You should see yourself.
4. Open a YouTube video and switch the extension on from the toolbar icon.

Step 2 isn't optional. An offscreen document has no UI, so it can't show a
permission prompt — the grant has to come from a real extension page first, and
then the offscreen document inherits it.

### Keep the folder path stable

An unpacked extension's ID is derived from its folder path, and camera access is
granted per extension ID. Renaming or moving the folder creates what Chrome
treats as a different extension, and the camera grant does not follow it — the
detector will fail with `NotAllowedError` until you grant access again from the
settings page.

## Try it

Play a video, then turn your head away — roughly 30 degrees is enough. It should
pause after about 1.5 seconds and resume about 0.3s after you look back. Walking
off or closing your eyes works too. The delays are deliberate: without them a
blink or a head scratch makes the video stutter.

The popup shows live yaw and pitch while the feature is on. Turn your head and
watch the numbers to find the angle where it trips, then adjust the limits in
settings.

### Lighting matters more than anything else

The model needs a reasonably lit face. In poor light it finds nothing at all,
and "no face" is indistinguishable from "walked away" — which is why bad
lighting used to read as constant inattention.

The detector now separates the two cases. Once it has seen a face, absence means
absence. But if it has never managed to find one, it treats that as a setup
problem, keeps the video playing, and reports "Can't see you" rather than
pausing forever.

### Neutral position (optional)

Angles are measured as deviation from wherever you told it you normally sit —
not from the camera axis. Sit the way you actually watch, then press **Detect** in settings. The dot in
the popup should settle in the middle of the field.

Anyone sitting off-centre — laptop to one side, screen below eye level, leaning
on an armrest — starts partway to the limit before they've moved, and gets pauses
in one direction while having far too much room in the other. Widening the limits
doesn't fix that; it just makes detection worse both ways. Set neutral again
whenever you change seat or move the screen.

**Reset all to defaults** puts neutral back on the camera axis and restores the
stock limits. Detect is disabled unless a face is currently visible, so it can't
store a position you aren't in.

Worth checking, since these are the behaviours that matter:

- Pause the video yourself. The extension must never resume it for you.
- Cover the camera lens. Video pauses — that's a face-absent reading, correct
  for M1.
- Turn the extension off mid-pause. The video stays paused and the camera
  stops — switching off hands control back rather than pressing play for you.

## How it fits together

```
offscreen document          background.js            content.js
webcam + MediaPipe     ->   service worker      ->   pauses <video>
6fps, hysteresis            holds state              tracks user intent
                                  ^
                              popup.js
                            toggle + Run check
```

The camera lives in an offscreen document because a service worker can't hold a
media stream and a content script would fight the page's CSP. One camera session
serves the whole browser rather than one per tab.

The detector only messages the service worker when the debounced value actually
*changes*. Reporting every frame would wake the worker six times a second.

## Debugging

The popup has a **Run check** button that walks the whole chain and reports
which link is broken — faster than hunting through three separate consoles:

| Part | Where to look |
| --- | --- |
| `content.js` | DevTools on the YouTube tab |
| `background.js` | `chrome://extensions` → **service worker** |
| `offscreen.js` | `chrome://extensions` → **offscreen.html** |

After editing, reload the extension. Open tabs are re-injected automatically on
update, so you no longer have to reload each one by hand — but if **Run check**
reports "Pages reached 0", reload the video tab and check again.

## Design decisions

| Decision | Why |
| --- | --- |
| 6fps inference | 30fps is a battery complaint waiting to happen |
| 1.5s out, 0.3s back | Asymmetric — being slow to pause is fine, being slow to resume is annoying |
| 26° yaw, 30° pitch | Pitch is looser; people tilt down at a keyboard more than they swing sideways |
| Head direction, not gaze | True eye tracking needs per-user calibration and still isn't reliable |
| Fail open while on | No camera, denied permission, closed lid → video just plays |
| No auto-resume when off | Switching off shouldn't start audio you didn't ask for |
| Restricted contexts | `content.js` and `offscreen.js` only get `chrome.runtime`; storage, tabs and scripting route through the service worker |
| Default off | Camera-on-by-default earns one-star reviews |
| Bundled wasm + model | MV3 forbids loading executable code from a CDN |

## Publishing to the Chrome Web Store

The fee is $5, once, ever — not per extension and not annual. Everything else
costs time rather than money.

### Before you submit

1. **Remove the model from `.gitignore` and include it in the upload.** The
   store package must be self-contained; there's no post-install download step.
2. **Gate the 2Hz telemetry.** It wakes the service worker twice a second and
   only exists for tuning. Ship it behind a debug flag or drop it.
3. **Decide the host permissions.** `youtube.com` only is a much easier review
   than `<all_urls>`. Broad host access invites questions about why you need it.
4. **Host the privacy policy** at a public URL. `PRIVACY.md` in this repo is the
   text; GitHub Pages will serve it for free. A policy URL is required because
   the extension uses the camera.

### Listing assets

- Icons at 16, 32, 48 and 128px — already in `icons/`.
- At least one 1280×800 or 640×400 screenshot. Show the popup over a video.
- A 440×280 small promo tile.
- A short description, and a detailed one that says on-device processing in the
  first line. It's the first thing a reviewer and a user will both look for.

### The privacy tab

This is where most first submissions fail. You'll need:

- **Single purpose**: one sentence. "Pauses video playback when the user looks
  away from the screen, using on-device webcam face detection."
- **Permission justifications**: one line each for camera, offscreen, storage,
  scripting, and host access. Say what breaks without it.
- **Data disclosure**: tick nothing. Then make sure that stays true — undisclosed
  collection is grounds for removal after publishing, not just rejection.
- **Limited use certification**: confirm no selling, no ad targeting, no
  transfer to data brokers.

### Expect scrutiny

A camera extension gets a closer look than most, and review can take days rather
than hours. The thing that helps most is that the claim is verifiable: there are
no network requests in the code at all, so a reviewer can confirm the privacy
story by reading it. Keep it that way.

Fees and policies here reflect the store as of late 2026 — check the current
[developer documentation](https://developer.chrome.com/docs/webstore/register)
before submitting, since Google adjusts these over time.

## The extension error list

The extension writes nothing to the console. Every state it can be in — starting,
running, camera denied, can't see you — is reported through the popup and
through Settings › Diagnostics instead, so the error list stays empty in normal
operation.

MediaPipe's wasm does print glog-style startup lines (`xnnpack` acceleration,
`OpenGL error checking is disabled`, the TFLite delegate). Those are
informational and there's no flag to disable them at source, so `offscreen.js`
filters them at the console before Chrome collects them. The patterns match
glog's prefix format only, so genuine JavaScript errors still appear. Set
`SHOW_MEDIAPIPE_LOGS = true` at the top of that file when debugging MediaPipe
itself.

One of those lines is mildly informative: the TFLite delegate line means part of
the graph runs on CPU rather than GPU. That's normal — at 6fps with one face it
makes no practical difference.

Chrome's error list is cumulative, so entries stay after the problem is fixed.
**Clear all** empties it; anything that reappears is current.

---

Built by [Chandanpreet](https://www.chandanpreet.com).
[Privacy policy](PRIVACY.md) · [Chrome Web Store](https://chromewebstore.google.com/detail/attention-pause/emihckaonnohmpbnkgpbppkniboknloc)

## Behaviour in awkward situations

The rule throughout: pause only when confident somebody is present and looking
away. Anything else plays.

| Situation | What happens | Why |
| --- | --- | --- |
| Room goes dark mid-video | Keeps playing, popup says "Too dark to see" | The frame is measured; a dark frame is a lighting problem, not an absent person |
| You walk away | Pauses | Frame is lit but has no face in it |
| Lens covered | Keeps playing | Reads as too dark, same as unlit |
| Video call takes the camera | Keeps playing, retries every 5s | Track fires `mute`; sitting on the last verdict would strand the video |
| Lid closed, machine sleeps | Keeps playing, reclaims camera on wake | Track ends, or frames stall for 8s |
| Camera permission denied | Keeps playing, popup links to settings | Nothing to detect with |
| You pause it yourself | Stays paused | The extension never resumes what it didn't pause |
| You switch the feature off | Stays paused, camera stops | Switching off shouldn't start audio you didn't ask for |
| Another tab is focused | Still tracked | You're facing the screen either way |

The one case that genuinely can't be resolved is sitting in the dark *and*
walking away — both read as "can't see", so it keeps playing. Failing that way
round is the right trade: a video that won't pause is an annoyance, one that
won't play is a broken extension.

## Known rough edges

- Every YouTube tab reacts, not just the active one.
- Videos inside iframes are ignored (`all_frames` is false).
- If you manually press play while a video is being held, tracking gets slightly
  confused and it may pause again on the next signal change.
