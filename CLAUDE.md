# CLAUDE.md

Guided facial exercise timer for Bell's palsy (facial nerve palsy) recovery. It is a static web page, used mainly on phones.

## Structure

- `index.html` holds the whole app (HTML, CSS, vanilla JS). There is no build step, no dependencies and no framework. Keep it that way unless there's a strong reason.
- `docs/biofeedback.md` covers the design of hardware biofeedback (surface EMG): options, electrode placement, signal processing, safety.
- `README.md` is a short description. Keep it UTF-8; it used to be UTF-16.

## How `index.html` is organised (script sections, in order)

1. **Translations (`I18N`)**: `de` and `en`. Every key must exist in **both** languages. Use `t("key", { var })` with `{var}` placeholders. Static elements use `data-i18n="key"` and are filled in by `applyLanguage()`.
2. **Exercises**:
   - `WARMUP` and `COOLDOWN` are always first and last.
   - `POOL` holds the random exercises: `{ id, group, reps, hold, name: {de,en}, desc: {de,en} }`.
   - Optional fields:
     - `cues`: spoken words per rep instead of "Hold"
     - `titleReps`: the title shows "10 reps" instead of "10 × 3 s"
     - `meter`: blendshape names measured left/right
     - `watch`: blendshapes that must stay still (synkinesis)
     - `synkinesis: true`: used only when synkinesis mode is on
     - `avoidWithSynkinesis`: left out when synkinesis mode is on
   - The rep cycle is `floor(duration / reps)` seconds and must be greater than `hold`. Every exercise lasts 60 s.
   - `GROUPS` sets the face areas, top to bottom. Routines are sorted by it.
3. **`buildRoutine(lengthKey)`**: in synkinesis mode it first picks `ceil(count / 3)` synkinesis exercises. It fills the rest round-robin over the groups (eyes and mouth first), then sorts by group.
4. **Storage**: `localStorage` keys are prefixed `bpt.` (lang, length, sound, voice, meter, side, synkinesis, history). The history is an array of `{ date, ts, length, completed, symmetry?, perExercise?, coactivation? }`.
5. **Sound and voice**: Web Audio tones (`sounds.*`) and `speechSynthesis` in the UI language. `cueState(step, elapsed)` drives both the visible rep cue and the spoken cue.
6. **Symmetry meter**: MediaPipe Face Landmarker (`@mediapipe/tasks-vision`, version pinned in `MEDIAPIPE_BASE`), loaded via dynamic `import()` only when the meter is enabled.
   - `cameraSource` implements the *activity source* interface: `load / start(onSample) / stop / read(names) → {left, right}`. A future EMG source must implement the same interface.
   - The resting baseline is recorded during the 6 s prep. The per-side peak is taken during each hold. Rep symmetry is `min / max`.
   - Co-movement: a `watch` signal on the affected side rising ≥ `COACT_THRESHOLD` during a hold.
   - `SWAP_SIDES = false` has been verified on a real device (a left wink moves the "Left" bar).
7. **Session layout**: the `.stage` holds the video, the ring, the cue and the meter. When the camera works, `#sessionCard` gets `.has-camera`. The mirror then fills about 58% of the screen height, with the timer in the top-right corner and the cue pill plus a compact meter strip at the bottom. Without a camera, the large ring layout is used. The language switch is hidden during a session (`body[data-screen]`). The whole session must fit on a phone screen (about 412×915 CSS px) without scrolling.
8. **Session flow**: `startSession → beginPhase("prep" | "active") → advance → finishSession`. `stopSession` clears the timer, speech, camera, wake lock and meter.

## Conventions

- The UI language is German first, then English. Write exercise texts in plain, friendly language: "du" in German, "you" in English.
- Keep exercise advice gentle and conservative: slow, small movements, stop if it hurts. Strengthening the neck (platysma) and synkinesis work carry a "check with your therapist" note. No electrical stimulation.
- The affected side defaults to **right**.
- Nothing personal leaves the device. Video is processed locally, and the history stays in `localStorage`.

## Testing (no test suite)

- **Syntax:** pull out the `<script>` block and run `node --check`.
- **Logic:** pull the pure sections (I18N, POOL, `buildRoutine`, `cueState`, meter functions) into a Node script with stubs. Check that both languages have the same keys, that every `data-i18n` key exists, that rep cycles are valid, and that routine size and composition match each length and synkinesis mode.
- **Browser:** serve with `python3 -m http.server` and run `chromium --headless=new --use-fake-device-for-media-stream --use-fake-ui-for-media-stream --enable-logging=stderr`. Log state with `console.log` from an injected script.
  - Use **real time**, not `--virtual-time-budget`: virtual time barely runs `requestAnimationFrame`, and the fake video never starts.
  - The fake camera shows no face, so expect "Face not detected".
  - `--screenshot` captures the page before scripts that run later. For mid-session or phone-sized screenshots, launch Chromium with `--remote-debugging-port` and drive it from Node 24, which has a built-in `WebSocket`. Call `Emulation.setDeviceMetricsOverride({ width: 412, height: 915, mobile: true })`, click and skip with `Runtime.evaluate`, then `Page.captureScreenshot`.
  - Stop background servers by PID. `pkill -f <pattern>` matches and kills its own shell.
- Real face tracking, voice and wake lock can only be checked on a real device.

## Git

- Remote `origin` is `https://github.com/soffensive/bell-palsy-timer`, and work goes on `main`.
