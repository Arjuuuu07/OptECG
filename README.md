# optECG

> **Non-Invasive, Camera-Based ECG Monitoring for Clinical Support**

![Status](https://img.shields.io/badge/status-in%20development-orange)
![Phase](https://img.shields.io/badge/current%20phase-1-blue)
![Type](https://img.shields.io/badge/type-clinical%20support-green)

optECG acquires ECG waveforms optically from existing bedside monitors — no electrodes, no hardware modifications, no patient contact. A fixed camera captures the monitor display, extracts a calibrated millivolt signal in real time, and runs rhythm analysis with a parallel safety watchdog layer.

> ⚠️ **This project is actively under development.** Architecture and APIs are subject to change. Not ready for clinical use.

---

## What it will do

- Capture ECG display footage at good fps using a fixed-mount camera
- Detect and correct monitor tilt using YOLO OBB (Oriented Bounding Box)
- Calibrate pixel amplitudes to real millivolt values using the monitor's own 1 mV reference pulse
- Extract a 1-D digital signal via column scanning across the waveform region
- Assemble 10-second windows for rhythm analysis
- Detect tachycardia, bradycardia, ST deviation, arrhythmia, and absent P-wave
- Produce human-readable reason strings for every alert — fully explainable, no black box
- Run two independent safety watchdogs (gyroscope + YOLO re-check) on separate threads

---

## Planned Pipeline

```
Camera Input
    │
    ▼
Phase 1 — Image Capture & Orientation        [🔄 In Progress]
    YOLO OBB → tilt angle θ → warpAffine deskew → aligned ROI
    │
    ▼
Phase 2 — Calibration                        [🔲 Planned]
    Detect 1mV cal pulse → measure A_cal → k = A_cal / A_screen
    │
    ▼
Phase 3 — Signal Extraction                  [🔲 Planned]
    ResNet + ECA-Net (wave classification) → column scan → mV vector
    │
    ▼
Phase 4 — Analysis & Detection               [🔲 Planned]
    Rule-based thresholds + trained vector matcher → fused alert + reason string
    │
    ▼
Clinical Alert Output

════════════════════════════════════════════════════════
Safety Watchdog (parallel, independent)      [🔲 Planned]
    Gyroscope monitor + YOLO re-check (every 60s, 3-strike)
```

---

## Phase Details

### Phase 1 — Image Capture & Orientation `[🔄 In Progress]`

A fixed-mount camera captures the ECG monitor at 30–50 cm, perpendicular to the display. YOLO OBB detects the screen region and outputs the oriented bounding box plus tilt angle θ. OpenCV's `warpAffine` corrects angles within ±3°; angles beyond this prompt the operator rather than silently correcting.

The YOLO model is trained exclusively on in-house photos taken under the intended deployment setup — making it an implicit setup validator that rejects frames from incorrect positioning.

A **startup gate** will run YOLO once before any session begins. If confidence is below threshold, the session is blocked entirely.

**Current focus:**
- Data collection under controlled setup conditions
- YOLO OBB model training on in-house images
- Deskew pipeline implementation

---

### Phase 2 — Calibration `[🔲 Planned]`

Every ECG monitor displays a standard 1 mV calibration square wave. A dedicated YOLO model will detect this mark and measure its pixel area (`A_cal`). The screen area (`A_screen`) is locked from the first frame mask and held constant throughout the session.

```
k = A_cal_pixels / A_screen_pixels
Amplitude (mV) = pixel_height / k
```

Calibration will run over a 5–10 frame acceptance window. k will be accepted only when the bounding box is consistently detected; `A_cal` is averaged before division. k will be logged with session ID and timestamp for retrospective audit.

---

### Phase 3 — Signal Extraction & Vectorisation `[🔲 Planned]`

For each frame, ResNet + ECA-Net will classify the visible waveform components (P wave, QRS complex, T wave). A column scan will then extract the waveform amplitude at each horizontal pixel position — the brightest foreground pixel per column gives `y[x]`. Dividing by k produces a 1-D float array in millivolts.

Thirty frames will be assembled into a 10-second window with boundary continuity checks to prevent stitching artefacts. A sliding window (5-second overlap) will ensure events at window boundaries are never missed.

---

### Phase 4 — Analysis & Detection `[🔲 Planned]`

A hybrid detector will run two parallel paths:

| Path | Method | Purpose |
|------|--------|---------|
| Rule-based | Threshold checks on R-R intervals, HR, ST deviation | Tachycardia, bradycardia, ST changes |
| Trained vector | Pattern matching on waveform morphology | Arrhythmia, absent P-wave, complex patterns |

Every alert will include a human-readable reason string — no black box output:

```
"HR = 115 bpm — above 100 bpm threshold → TACHYCARDIA"
"ST deviation = +0.18 mV in window → ST ELEVATION FLAG"
"RR variance = 0.24s — irregular rhythm pattern → ARRHYTHMIA"
```

**Target detectable conditions:** Tachycardia · Bradycardia · ST Elevation · ST Depression · Arrhythmia · Absent P-wave

---

### Safety Layer `[🔲 Planned]`

Two watchdogs will run on independent threads, completely decoupled from the analysis pipeline.

**Gyroscope watchdog** — detects physical disturbance to the camera mount. Motion must exceed threshold continuously for 300–500 ms (debounce) before session shutdown to avoid false kills from brief knocks.

**YOLO re-check watchdog** — verifies setup validity every 60 seconds. Three consecutive failures (3-strike rule) trigger graceful degradation or hard shutdown. One failure alone is treated as transient occlusion.

All safety events will be written to an audit log with timestamp, strike count, session ID, and k value at time of event.

---

## Planned Model Architecture

| Component | Model | Role |
|-----------|-------|------|
| Screen detection | YOLO OBB | Oriented bounding box + tilt angle |
| Calibration mark | YOLO (dedicated) | 1 mV square wave detection |
| Wave classification | ResNet + ECA-Net | P / QRS / T wave feature extraction |
| Rhythm analysis | Rule engine + vector matcher | Threshold + morphology detection |

---

## Design Principles

**Explainability over accuracy** — every alert will include a reason string. A black-box classifier is not acceptable in a clinical context.

**In-house training data only** — YOLO models will be trained on images captured under the exact deployment setup, making the model an implicit validator of correct positioning.

**Calibration anchored to physics** — the 1 mV reference pulse already present on the monitor is used as the calibration source. No external reference hardware required.

**Safety cannot be blocked** — watchdog threads will be independent of the analysis pipeline. A stalled Phase 4 computation cannot prevent the safety layer from triggering a shutdown.

**Clinical support, not replacement** — optECG is designed to assist clinical staff, not replace physician interpretation or standard ECG equipment.

---



---

## Requirements

- Python 3.10+
- OpenCV
- PyTorch
- Ultralytics (YOLO)
- A fixed-mount camera with consistent lighting setup
- ECG monitor with standard 1 mV calibration pulse

---

## Clinical Disclaimer

optECG is a clinical **support** tool under development. It does not replace physician interpretation, standard 12-lead ECG equipment, or continuous patient monitoring systems. **Do not use in any clinical setting until validation is complete.**

---

## License

[To be defined]
