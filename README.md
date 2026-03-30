
# 📷 OptECG — Non-Invasive Optical ECG Digitization & Clinical Decision Support System

> **MSc Research Project — Concept & Design Document**
> This is a **planned research system**, currently in the idea and design phase.  This document outlines the intended architecture, methodology, and research goals.

---

## ⚠️ Disclaimer

This system is a **research prototype concept** being developed for academic purposes. It is **not a certified medical device** and must **not** be used as a replacement for clinical-grade cardiac monitoring equipment. All clinical decisions must be made by qualified medical professionals. This tool is intended solely to assist — not replace — clinical judgment.

---

## 📌 Table of Contents

- [Motivation](#motivation)
- [What This System Plans To Do](#what-this-system-plans-to-do)
- [System Architecture Overview](#system-architecture-overview)
- [Phase 1 — Image Pre-Processing & Calibration Lock](#phase-1--image-pre-processing--calibration-lock)
- [Phase 2 — Signal Extraction & Ratiometric Normalization](#phase-2--signal-extraction--ratiometric-normalization)
- [Phase 3 — Rule-Based Clinical Decision Support (XAI)](#phase-3--rule-based-clinical-decision-support-xai)
- [Phase 4 — Data Storage & Trend Buffer](#phase-4--data-storage--trend-buffer)
- [Phase 5 — Trend Display & Pattern Flagging](#phase-5--trend-display--pattern-flagging)
- [Phase 6 — Short-Term Deterioration Forecasting (The 20-Minute Window)](#phase-6--short-term-deterioration-forecasting-the-20-minute-window)
- [Known Limitations & Engineering Trade-offs](#known-limitations--engineering-trade-offs)
- [Tech Stack](#tech-stack)

---

## Motivation

Modern hospitals use cardiac monitors from vendors like **GE, Philips, and Schiller**. These devices run on **proprietary closed software** with no public API, no data export, and no integration hooks.

This means:

- A nurse must **physically watch the monitor** to track trends
- There is **no automated alert** unless the machine's own threshold is crossed
- In resource-limited hospitals (rural clinics, field hospitals, low-income countries), there is **no secondary layer of decision support**
- Gradual deterioration — happening slowly over 10–20 minutes — is **nearly impossible to catch by human observation alone**

This project plans to build a **non-invasive optical bridge** — a camera pointed at the monitor screen — that extracts the live ECG signal without touching the hospital's proprietary system, and runs both a transparent rule-based decision support layer and a proactive deterioration forecasting layer on top of it.

---

## What This System Plans To Do

```
Live ECG Monitor Screen
        │
        ▼  (standard camera, no hardware modification)
Optical Calibration & Distance Lock
        │
        ▼
Grayscale + De-Flicker Pre-Processing
        │
        ▼
ResUNet + CBAM Segmentation  →  Binary Mask
        │
        ▼
Scanline Extraction  →  1D Voltage Vector [mV]
        │
        ▼
Rule-Based Clinical Decision Support (Phase 3)
   ├── Heart Rate (BPM)
   ├── ST-Segment Analysis
   ├── Arrhythmia Flags
   └── Transparent Explainable Alerts
        │
        ▼
Sliding Window Buffer — 30–60 min rolling history (Phase 4)
        │
        ▼
Trend Display & Deterioration Flagging (Phase 5)
        │
        ▼
Short-Term Deterioration Forecasting — 20-Minute Pre-Warning (Phase 6)
   ├── Micro-shift detection in extracted signal
   ├── Time-series pattern matching (LSTM / Transformer)
   └── Early alert before rule-based threshold is crossed
```

---

## System Architecture Overview

The pipeline is divided into **6 phases**, each responsible for a distinct processing stage. The key innovation is the combination of:

1. **Non-invasive optical extraction** — no hospital IT integration required
2. **Transparent rule-based clinical flags** — fully auditable by clinicians
3. **Proactive deterioration forecasting** — catching downward trends before they become crises

---

## Phase 1 — Image Pre-Processing & Calibration Lock

### Objective
Ensure the camera is at the correct distance before any data is captured or stored. The system acts as a **smart scanner**, not just a passive camera.

### 1.1 Real-Time Sweet Spot Check

The system continuously analyzes the live camera feed and measures the **Calibration Pulse Height** ($H_{pixels}$) — the pixel height of one standard ECG calibration pulse (1mV reference marker).

| Condition | $H_{pixels}$ | System Action |
|---|---|---|
| Too Far | $< 40$ px | ⚠️ Warning: "Move Closer" |
| Too Close | $> 200$ px | ⚠️ Warning: "Move Back" |
| **Locked** | $\approx 100$ px | ✅ Lock acquired — capture begins |

When locked, the system records $H_{curr} = H_{pixels}$ for use in normalization downstream.

### 1.2 Grayscale Conversion

The ECG monitor screen displays a waveform in high contrast (typically bright green or white on black). Converting to **grayscale (0–255)** removes color channel noise and lets the segmentation model focus purely on **luminance contrast**.

### 1.3 De-Flicker Pre-Processing

A monitor screen refreshes at 60Hz. A camera filming at 30fps creates **Moiré interference** — flickering horizontal bands. A temporal averaging step across 3–5 consecutive frames is planned to suppress this artifact before the image enters the segmentation model.

---

## Phase 2 — Signal Extraction & Ratiometric Normalization

### Objective
Convert the cleaned image into a **1D array of medical voltage values** using scanline extraction.

### 2.1 ResUNet + CBAM Segmentation

The pre-processed grayscale frame will be fed into a **ResUNet with Convolutional Block Attention Module (CBAM)**. The output is a **Binary Mask**:

```
Pixel Value = 1  →  ECG wave (white)
Pixel Value = 0  →  Background (black)
```

CBAM attention allows the model to focus on the spatially thin ECG trace while suppressing residual screen artifacts or glare.

### 2.2 Scanline Extraction (Left → Right)

Each vertical column of the binary mask is scanned independently. The **Center of Mass** of the white pixels in that column gives a single sub-pixel-accurate $y$ position.

```
Column x: white pixels at rows [150, 151, 152]
→  y_raw = mean([150, 151, 152]) = 151.0
```

### 2.3 Coordinate Transformation

Image coordinates place (0,0) at the **top-left**, meaning "up" is a smaller number. This is inverted to match medical convention (upward deflection = positive voltage):

$$V_{raw} = u - y_{raw}$$

Where $u$ is the **vertical midpoint** of the image frame (the baseline reference).

**Example:**
- Wave at top of frame: $u - 50 = +150$ → Positive deflection ✅
- Wave at bottom of frame: $u - 350 = -150$ → Negative deflection ✅

### 2.4 Ratiometric Normalization (Scale Factor)

The physical distance between camera and monitor varies. The calibration pulse from Phase 1 corrects for this:

$$S = \frac{H_{ref}}{H_{curr}}$$

Where:
- $H_{ref}$ = reference calibration pulse height in pixels (standard condition)
- $H_{curr}$ = actual measured calibration pulse height at lock time

The final normalized voltage value for each time step:

$$V_{final} = (u - y_{raw}) \times S$$

This ensures that **regardless of camera distance**, the output is always in standardized, medically interpretable units.

### 2.5 Vector Stitching (Cross-Fade at Boundaries)

Each captured frame produces one 1D vector segment. To prevent hard jumps between segments (which would appear as false arrhythmias to downstream analysis), an **overlap-add crossfade** is planned at every boundary:

```
Capture 1:  |─────────── 1.2s ────────────|
Capture 2:        |─────────── 1.2s ───────────|
                  |◄── 0.2s overlap ──►|
```

```python
overlap_samples = 50
blend_weights = np.linspace(0, 1, overlap_samples)
stitched = (V1_tail * (1 - blend_weights)) + (V2_head * blend_weights)
```

### 2.6 Final Output (Planned)

A continuous 1D array of normalized voltage values:

```python
Vector = [0.02, 0.05, 0.88, 1.25, 0.40, -0.12, ...]  # units: mV (normalized)
```

---

## Phase 3 — Rule-Based Clinical Decision Support (XAI)

### Objective
Provide **transparent, explainable, real-time clinical flags** based on the normalized 1D signal. This is a core academic contribution of the project.

> This is **not** a black-box AI classifier. Every alert is backed by an explicitly coded clinical rule that a doctor can audit, challenge, and override.

### 3.1 Feature Point Detection

The system will scan the 1D vector to locate standard ECG morphological landmarks:

| Feature | Definition | Clinical Use |
|---|---|---|
| **R-Peak** | Highest positive deflection | Heart rate, R-R interval |
| **P-Wave** | Small deflection before QRS | Atrial activity |
| **QRS Complex** | Sharp spike cluster | Ventricular conduction |
| **ST-Segment** | Flat section after QRS | Ischemia detection |
| **T-Wave** | Rounded wave after ST | Repolarization |

### 3.2 Clinical IF-THEN Rules (Planned)

```
RULE 1 — Tachycardia
  IF   R-R Interval < 0.6s
  THEN Alert: "High Heart Rate (>100 BPM) — Tachycardia Pattern Detected"

RULE 2 — Bradycardia
  IF   R-R Interval > 1.0s
  THEN Alert: "Low Heart Rate (<60 BPM) — Bradycardia Pattern Detected"

RULE 3 — ST Elevation (STEMI Indicator)
  IF   ST-Segment Mean > baseline + 0.1 mV
  THEN Alert: "ST Elevation Detected — Review for Acute Myocardial Infarction"

RULE 4 — ST Depression (Ischemia Indicator)
  IF   ST-Segment Mean < baseline - 0.05 mV
  THEN Alert: "ST Depression — Possible Subendocardial Ischemia"

RULE 5 — QRS Widening
  IF   QRS Duration > 0.12s
  THEN Alert: "Wide QRS Complex — Possible Bundle Branch Block"
```

### 3.3 Explainability Output Format (Planned)

Every alert will include a **plain-language explanation** with the measured value and the threshold it crossed:

```
⚠️  ALERT — ST Elevation Detected
    Measured ST-Segment: +0.15 mV above isoelectric baseline
    Clinical Threshold:  +0.10 mV
    Excess:              +0.05 mV
    Pattern Match:       Consistent with Acute Myocardial Infarction (STEMI)
    Action:              Recommend immediate physician review.
```

This transparency is the defining feature of the system. No score. No black box. Every flag is traceable to a specific measurement.

---

## Phase 4 — Data Storage & Trend Buffer

### Objective
Store a rolling history of the digitized signal to support trend analysis.

### 4.1 Sliding Window Buffer (FIFO)

The system will maintain a **circular buffer** of the last 30–60 minutes of data:

```python
Buffer stores per time step:
  - timestamp         (real-world clock time)
  - mV_vector         (normalized 1D signal)
  - heart_rate_bpm    (calculated R-R)
  - flags             (list of active clinical rules)
```

Old data is automatically dropped as new data arrives (First In, First Out).

### 4.2 Real-Time Sync & Frame Drop Logic

The camera capture and AI processing will run on **separate threads** (Producer-Consumer pattern):

```
Camera Thread (Producer)      AI Thread (Consumer)
      │                             │
  Frame @ t=0  ──────────►  Buffer Slot A  ──► Process
  Frame @ t=1  ──► DROP     Buffer Slot B  (busy)
  Frame @ t=2  ──────────►  Buffer Slot A  ──► Process
```

- Each frame is **timestamped at capture**, not at processing
- If the AI is busy, frames are **silently dropped** to prevent queue buildup
- The output is always synced to the **real-world clock**, not the processing clock
- Acceptable drop rate: ~17% (1 in 6 frames) at 1.2s processing / 1s capture cadence

---

## Phase 5 — Trend Display & Pattern Flagging

### Objective
Display historical signal trends and flag **progressive deterioration patterns** visible over 5–15 minute windows.

### 5.1 Temporal Trend Monitoring

The system will compare current signal metrics against historical checkpoints:

| Metric | Tracked Over |
|---|---|
| R-Peak amplitude (mV) | 5 min / 10 min / 15 min rolling windows |
| R-R interval variance | Same windows |
| ST-segment mean | Same windows |
| QRS morphology stability | Shape correlation score |

### 5.2 Progressive Deterioration Flags (Planned)

```
FLAG — Declining R-Peak Amplitude
  IF   R-Peak height decreasing consistently over 10-minute window
  THEN Flag: "R-Peak amplitude trend: ↓ 8% over 10 min — Monitor closely"

FLAG — Rhythm Instability
  IF   R-R interval variance increasing over time
  THEN Flag: "Rhythm variability increasing — Possible onset of arrhythmia"

FLAG — ST Trend Worsening
  IF   ST-segment elevation increasing across consecutive 5-min windows
  THEN Flag: "ST elevation trend worsening — Escalate immediately"
```

> **Note:** Phase 5 provides **trend-based flagging**, not predictive forecasting. The system observes and reports what is measurably happening — it does not claim to predict what will happen. That is the role of Phase 6.

---

## Phase 6 — Short-Term Deterioration Forecasting (The 20-Minute Window)

### Objective
Move the system from **Reactive** (responding to a problem that has already occurred) to **Proactive** (predicting a problem before it shows up as a threshold violation on the monitor).

This is the most research-intensive phase of the project and is currently at the **conceptual design stage**.

---

### 6.1 The Core Idea: Velocity of Change

Instead of looking at a single heartbeat in isolation, the forecasting layer looks at the **rate of change** of key signal metrics over time.

**Example — Trend Velocity:**
```
Heart Rate @ t-10min:  70 BPM
Heart Rate @ t-5min:   75 BPM
Heart Rate @ t-now:    80 BPM
→  Trend velocity: +5 BPM / 5 minutes → Rising
→  Projected @ t+20min:  ~100 BPM  →  PRE-WARNING triggered
```

The system does not wait for 100 BPM to be reached. It flags the trajectory *toward* it.

---

### 6.2 Micro-Shift Detection

The forecasting layer is designed to catch **"Micro-Deteriorations"** — tiny, systematic changes in the digitized signal that are invisible to human observation in real time:

| Micro-Shift | What It Looks Like | Why It Matters |
|---|---|---|
| R-Peak height loss | Amplitude dropping 0.05mV per minute | Gradual contractile weakness |
| T-Wave shape change | Flattening or inversion developing slowly | Evolving ischaemia |
| R-R interval drift | Subtle lengthening or shortening of intervals | Pre-arrhythmic state |
| ST-segment creep | ST level slowly migrating from baseline | Early ischaemia |
| QRS duration widening | 2–3ms broader per 5-minute window | Conduction deterioration |

These shifts are quantified from the normalized 1D vector produced in Phase 2 and tracked across the rolling buffer from Phase 4.

---

### 6.3 The 20-Minute Pre-Warning Window

**Why 20 minutes?**

20 minutes is chosen as the forecasting horizon because:
- It is a clinically meaningful window — enough time for a nurse to escalate, call a physician, and prepare an intervention
- It is short enough that the model's projection remains statistically grounded in recent signal behavior
- It balances lead time against false-positive risk


---

### 6.4 Deep Learning Pattern Matching (Conceptual)

The specific model architecture is not yet decided. The leading candidates are:

| Architecture | Why It's Suitable |
|---|---|
| **LSTM (Long Short-Term Memory)** | Designed for sequential time-series data; well-established in physiological signal forecasting |
| **Transformer (Time-Series)** | Captures long-range temporal dependencies; increasingly favored for medical time-series |
| **Hybrid CNN-LSTM** | CNN extracts local morphological features; LSTM models the temporal trajectory |

The model will be trained to recognize **"Failure Patterns"** — sequences of micro-shifts that historically precede clinical deterioration events. At inference time:

```
Live Vector (last 10 min of metrics)
        │
        ▼
Pattern Matching against learned Failure Vectors
        │
        ├── High similarity to Failure Pattern → 🔴 HIGH RISK — Pre-Warning
        ├── Moderate similarity                → 🟡 WATCH — Elevated monitoring
        └── Low similarity                     → 🟢 STABLE — No flag
```

The comparison is not a single number — it is a **trajectory match**. The model asks: *"Does the shape of change in the last 10 minutes look like the shape of change that precedes deterioration?"*

---

### 6.5 Relationship to Phase 3 (Rule-Based Layer)

Phase 6 and Phase 3 are **complementary, not competing**:

| Layer | What It Catches | When It Fires |
|---|---|---|
| Phase 3 — Rule-Based | Threshold violations that have already occurred | Reactive — now |
| Phase 6 — Forecasting | Trajectories heading toward threshold violations | Proactive — before |


---

## Known Limitations & Engineering Trade-offs

These are acknowledged openly as part of the research scope.

| Limitation | Root Cause | Planned Mitigation |
|---|---|---|
| **Processing Lag** | ResUNet inference time (~800ms on CPU) | Double-buffer FIFO with frame dropping |
| **Vector Stitching Artifacts** | Hard boundaries between capture windows | Overlap-Add crossfade (0.2s overlap) |
| **Resolution vs. Speed** | Full-frame ResUNet is slow | 2-Stage ROI crop: edge detection → crop → ResUNet on thin strip only |
| **Screen Flicker (Moiré)** | Monitor Hz vs camera fps mismatch | Temporal frame averaging (3–5 frames) |
| **Glare / Specular Reflection** | Shiny monitor surface + overhead lighting | Over-saturation masking pre-processing |
| **Domain Gap (Training vs Live)** | Model trained on clean data, receives noisy camera-extracted input | Validate against MIT-BIH on digitized frames |
| **Phase 6 False Positives** | Forecasting over 20 min introduces uncertainty | Confidence thresholding + alert severity tiering |
| **Phase 6 Training Data** | Need labeled "deterioration trajectory" datasets | PTB-XL + MIT-BIH + synthetic augmentation |

---

## Tech Stack

| Component | Planned Technology |
|---|---|
| Camera Interface | OpenCV (`cv2`) |
| Segmentation Model | PyTorch — ResUNet + CBAM |
| Signal Processing | NumPy, SciPy |
| Feature Detection | Custom peak-finding on 1D vector |
| Forecasting Model | PyTorch — LSTM or Transformer (TBD) |
| Trend Buffer | Python `collections.deque` |
| Visualization | Matplotlib / Streamlit (dashboard) |
| Validation Dataset | MIT-BIH Arrhythmia Database (PhysioNet), PTB-XL |

---



## Validation Strategy

To demonstrate that the signal extracted by this pipeline is medically accurate, the system will be validated against the **MIT-BIH Arrhythmia Database** (PhysioNet).

### Method (Planned)

1. Take known clean ECG records from MIT-BIH
2. Render them visually on screen at standard ECG display parameters
3. Point the camera at the rendered screen
4. Run the full pipeline (Phases 1–2) to extract the 1D vector
5. Compare the extracted vector against the original MIT-BIH ground truth using:
   - **R-Peak location error** (ms)
   - **Amplitude error** (mV)
   - **Pearson correlation** between extracted and ground-truth signals


---


## License

MIT License — see `LICENSE` for details.

This software is provided for research and educational purposes only. The authors accept no liability for clinical use.

---

*Developed as part of MSc dissertation research — currently in the concept and planning stage. For academic inquiries, open an issue or contact via GitHub.*
