# Blinker Tracker

A real-time eye-fatigue monitor that runs in the browser. The webcam feed is processed locally with MediaPipe Tasks Vision, blinks are detected from the Eye Aspect Ratio (EAR), and a HUD overlays live tracking data on top of the video. When blink rate stays low for too long, the app surfaces a "take a break" prompt.

Built as a hands-on exploration of in-browser computer vision and embedded web apps. Deployed to Vercel and embedded into a Framer site via `<iframe>`, with bidirectional `postMessage` control (remote shutdown, fatigue notifications).

**Live demo:** embedded inside [my Framer site](https://framer.com) — the Vercel build is the iframe source.

---

## What it does

- **Blink detection** from a 468-landmark face mesh, using EAR thresholds.
- **Adaptive calibration** — collects ~90 open-eye samples, sets the threshold at 65% of the personal baseline. Works across face shapes and camera angles without tuning.
- **Rotation-invariant EAR** via `Math.hypot()` so blinks register reliably with a tilted head (an early bug — Y-only distance broke detection when the head was upright).
- **Blinks-per-minute** rolling counter; flags fatigue when BPM drops below threshold for sustained periods.
- **Rest mode** — pauses tracking, resets state (counts, BPM, baseline), waits for the user to resume.
- **John Cena mode** — swaps the camera feed for two static images (open/closed eyes) that flip in sync with the user's blinks. A toy mode for users who don't want to see themselves.
- **Pose-aware HUD** — shoulder/wrist landmarks from MediaPipe Pose drive floating coordinate labels next to the head.
- **Framer integration** — listens for `SHUTDOWN` / `RESUME` from the parent window, emits `FATIGUE_ALERT` upward; the parent surfaces a Web Notification (since notifications inside an iframe are blocked).
- **Responsive HUD** — every dimension is `clamp()`-based so the embed scales cleanly inside Framer breakpoints.

---

## Tech stack

- **React 19** + **Vite 5**
- **@mediapipe/tasks-vision** — `FaceLandmarker` + `PoseLandmarker`, loaded via CDN-hosted WASM
- **Web Worker** — drives the 33ms (~30 fps) frame tick so the main thread doesn't drift under load
- **getUserMedia** for camera capture, with explicit track teardown on shutdown
- **Canvas 2D** for the HUD overlay
- **postMessage** for iframe ↔ parent communication
- **Vercel** for hosting, **Framer** as the embed host

No backend. Everything runs client-side; no frames leave the browser.

---

## Running locally

```bash
npm install
npm run dev
```

The app needs camera permission. Open in a browser, allow access, and the tracker initializes once the WASM models load.

```bash
npm run build      # production build
npm run preview    # serve the build
```

---

## Architecture notes

**EAR calculation.** For each eye, vertical distance between top and bottom lid landmarks divided by horizontal distance between inner and outer corners. Both distances use `Math.hypot()` so head rotation doesn't skew the ratio.

**Adaptive threshold.** Cold start uses a fixed default; the first 90 frames where avg EAR > 0.32 are treated as open-eye samples. The baseline is the mean of those samples; the trigger threshold is `baseline * 0.65`. Threshold is *not* drifted with an EMA — earlier versions did, but the EMA pulled the baseline upward over time and made the detector progressively too strict (the "works for 15 seconds, then breaks" bug).

**Frame timing.** A Web Worker fires `tick` every 33ms; the main thread skips the tick if the previous frame is still being processed. This decouples model latency from the cadence, so dropped frames don't desynchronize the blink counter.

**Iframe shutdown.** When the parent sends `SHUTDOWN`, the worker is paused and `streamRef.current.getTracks().forEach(t => t.stop())` releases the camera. `RESUME` re-acquires via `getUserMedia`. Important for Framer's editor, which renders multiple preview frames and otherwise hits the WebMediaPlayer cap.

---

## Project status

Personal project; no longer under active development. The `src/App.jsx` is a single file by design — the goal was a tight, end-to-end build, not a production architecture. If I were continuing it, splitting the EAR/threshold logic out of the React tree and running the model in the worker (instead of just the tick) would be the next pass.
