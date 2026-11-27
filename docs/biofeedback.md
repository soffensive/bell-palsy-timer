# Biofeedback for facial exercises

Facial exercises work better when you can see what the muscles are doing. The weak side often moves less than it feels like it does. And in later recovery, *synkinesis* is common: unwanted movement that comes along with the intended one, such as the eye closing during a smile. Biofeedback makes both visible.

This document covers what is already built (camera) and how surface EMG hardware could be added later.

> **Not a medical device.** Everything here is for practice support only. Agree on the use of biofeedback, and especially on any strengthening of the neck (platysma) or work on synkinesis, with your doctor or therapist.

## What we want to measure

| Signal | Why it matters |
| --- | --- |
| Activation per side (left / right) for the muscle being trained | Shows whether the affected side moves, and how much compared with the healthy side |
| Symmetry = weaker side ÷ stronger side | One simple number to track over weeks |
| Co-activation of other muscles (e.g. eye during a smile) | Detects synkinesis early, so you can practise moving muscles separately |
| Resting tone | High resting activity on the affected side can point to contracture or hypertonia |

## Option 1: Camera (implemented)

The app can use **MediaPipe Face Landmarker** in the browser. It tracks 478 face points and outputs 52 "blendshapes" (ARKit-style), many of them per side: `mouthSmileLeft/Right`, `eyeBlinkLeft/Right`, `browOuterUpLeft/Right`, `mouthFrownLeft/Right` and so on.

How it works in the app:

1. During the 6-second "up next" countdown the face is at rest, so the app records a **baseline** for each side.
2. During each *hold* phase it records the **peak activity per side** above that baseline.
3. Rep symmetry = `min(left, right) / max(left, right)`. Reps where neither side moved noticeably are ignored.
4. **Synkinesis exercises** also *watch* a second signal on the affected side, for example the eye (`eyeSquint`, `eyeBlink`) during a smile. If it rises ≥ 0.2 above rest during a hold, the app warns ("Co-movement – smaller and slower") and counts that rep. The end screen shows e.g. "co-movement 2/6".
5. Exercise score = mean of its reps. Session score = mean of the measured exercises. The session score is saved in the history, and the start page shows it with the change compared with about a week earlier.

The camera image is processed locally and never uploaded. The model and the WebAssembly runtime (about 20 MB) are downloaded from jsDelivr / Google Storage the first time.

**Limits:**
- It measures visible *movement*, not muscle activity. A muscle can fire without visible motion, especially early in recovery.
- Only some exercises have a left/right blendshape pair. For pucker and whistle, MediaPipe only gives one overall `mouthPucker` value. The app therefore works out each side itself from the face landmarks: how far each mouth corner moves toward the face midline (nose bridge to chin), measured in face widths. Head turns change this distance, so keep your head still. Cheek puff, cheek suck-in, P/B/M, chin wrinkle and the neck stretch aren't measured.
- It depends on lighting, head pose and glasses. Face the camera straight on.
- Check the left/right assignment once on your device: wink with your left eye during a blink exercise and the **left** bar must move. If it doesn't, set `SWAP_SIDES = true` in `index.html`.

### Tension the camera can't see

The camera only sees movement. Muscle tension without visible movement (early recovery, resting hypertonia in synkinesis, a muscle that doesn't fully let go) needs EMG. Until then the app adds two proxies:

- **Tension diary:** after each session you rate eye, cheek, mouth, chin and neck from 0 to 10. The start page shows the trend.
- **Resting asymmetry:** during the relaxed countdown before each exercise, the camera measures each eye's opening (lid gap ÷ eye width) and how high each mouth corner sits along the face midline. These are reported as affected ÷ healthy eye opening and as the mouth-corner height difference in % of face width. A narrowed eye or a raised mouth corner at rest is a typical outward sign of hypertonia; a lower corner points more to weakness. These are indirect clues, not tension measurements.

## Option 2: Surface EMG (sEMG)

sEMG measures the electrical activity of the muscle through electrodes on the skin. It can detect activation **before** it becomes visible, which makes it the classic tool for facial biofeedback therapy.

### Hardware options

| Option | Channels | Link to the app | Rough cost | Notes |
| --- | --- | --- | --- | --- |
| **ESP32-S3 + Upside Down Labs BioAmp EXG Pill** | 1 per pill (use 2–4) | Web Bluetooth or USB Web Serial | ~€10 per pill + ~€10 ESP32 + electrodes | Small, made for EMG/EEG, open source. **Recommended DIY starter.** |
| ESP32 + MyoWare 2.0 Muscle Sensor | 1 per sensor | same | ~€40 per sensor | Easy, but the board snaps directly onto large electrodes, which is awkward on the face. Better suited to limbs. |
| OpenBCI Ganglion | 4 | Bluetooth LE (has been used from the browser) | several hundred € | Ready-made, good signal quality, no soldering. |
| OpenBCI Cyton | 8 | USB dongle (serial) | higher | Enough channels for both sides plus synkinesis monitoring. |
| Clinical sEMG biofeedback device | varies | usually its own software | via therapist | Validated; ask your therapist whether they use one. |

Prices are approximate and change. Check current listings.

### Recommended starter kit (no hardware yet)

- 1 × ESP32-S3 dev board (has BLE + native USB)
- 2 × BioAmp EXG Pill (one per side for the same muscle), later 4 for synkinesis monitoring
- Small pre-gelled snap electrodes (about 1–2 cm, "paediatric" or "facial" size) and shielded snap cables
- A USB power bank or LiPo battery. **Never run it from a mains-connected laptop while electrodes are on the face** (see Safety).
- Skin prep: alcohol wipes; optionally a mild abrasive gel

### Electrode placement

Place a pair (two electrodes about 1–2 cm apart, along the muscle fibres) over the same muscle on **both** sides. Put the reference/ground electrode on a bony spot: the mastoid behind the ear, or the bony centre of the forehead.

| Muscle | Exercise | Placement |
| --- | --- | --- |
| Frontalis | Eyebrow raise | Mid-forehead, about 2 cm above the eyebrow, in line with the pupil |
| Orbicularis oculi | Eye closure, squint | Just below and lateral to the outer eye corner (not on the eyelid) |
| Zygomaticus major | Smile | On the line from the mouth corner to the cheekbone, about halfway |
| Depressor anguli oris / platysma | Mouth corners down, neck tense | Below and lateral to the mouth corner; for the platysma, on the side of the neck below the jaw |

For **synkinesis** monitoring, add a pair on orbicularis oculi while training the mouth (smile, pucker). If eye activity rises during a mouth exercise, the app can warn you.

### Signal processing (on the ESP32)

1. Sample each channel at ≥ 500 Hz (1 kHz is better) with the ESP32 ADC.
2. Band-pass filter about 20–250 Hz (up to 450 Hz at 1 kHz) and add a 50 Hz notch filter (mains hum in Europe).
3. Full-wave rectify, then take an RMS envelope over about 100 ms.
4. Send the envelope for each channel to the browser at about 50 Hz.

The BioAmp project publishes Arduino sketches for EMG envelope extraction that can serve as a starting point.

**Calibration in the app:** record resting level (baseline) and maximum voluntary contraction (MVC) per side and muscle. Then show activity as % of MVC, and symmetry as a ratio, like the camera meter does.

### Connecting to the web app

- **Web Bluetooth:** the ESP32 exposes a GATT service with one *notify* characteristic carrying `[uint16 left, uint16 right, …]` envelope values. Works in Chrome/Edge on desktop and Android. **Not** in iOS Safari or Firefox.
- **Web Serial:** plain USB serial lines such as `L,R\n`. Chrome/Edge desktop only.

Because of these browser limits, the camera stays the default and EMG is an optional extra.

**App integration:** the meter in `index.html` reads from an *activity source* with `load()`, `start(onSample)`, `stop()` and `read(names) → { left, right }`. An `emgSource` would implement the same interface. Each exercise's `meter` field would then name the channel pair instead of blendshapes (e.g. `meter: ["zygomaticus"]`). Baseline, rep detection, scoring and history work unchanged.

## Safety

- **Measurement only.** These setups never send current into the body. Electrical stimulation is generally *not* recommended for Bell's palsy unless a doctor prescribes it, and it is out of scope here.
- **Electrical isolation.** Power EMG hardware from a battery or power bank, and use a wireless link (BLE). Don't connect electrodes to a circuit that is attached to a mains-powered computer or charger. If USB is needed, use a USB isolator.
- **Skin.** Use fresh electrodes, clean the skin before and after, and don't use them on broken or irritated skin. Keep away from the eye itself.
- **Synkinesis and the platysma.** Overactive neck bands are common after facial palsy. For them, therapists often focus on *relaxing and stretching* rather than strengthening. Biofeedback should then show you how to keep activity *low*.

## Possible next steps

1. Collect a few weeks of camera symmetry data and check that the trend looks plausible.
2. Build the 2-channel ESP32 + BioAmp prototype and log the raw envelopes over Web Serial.
3. Add `emgSource` to the app (Web Bluetooth) with a calibration screen (rest + MVC per side).
4. Add synkinesis alerts using a second channel pair.
