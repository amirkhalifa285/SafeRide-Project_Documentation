# CONVOY FIELD VALIDATOR Agent Prompt

**Purpose:** Run the full convoy analysis script on a fresh 3-car recording, interpret ALL outputs, and give a clear GO/NO-GO verdict on data quality before leaving the recording site.

**Last Updated:** February 26, 2026

---

## Context

Amir has just completed a 3-car convoy recording (Recording #2) with ESP32 units (V001, V002, V003). He extracted the 6 CSV files from the SD cards and placed them on his laptop. Your job is to run the analysis, interpret the results, and tell him whether the data is good enough or if they need to drive again while the cars are still available.

### Why This Recording Matters

This is **Recording #2**. Recording #1 (Feb 21, 2026) had two critical issues:

1. **V001-V002 link had ~85% packet loss** because the board was placed near the car door handle (metal body shielded the ESP-NOW signal). Only 320 packets delivered out of ~2000.
2. **"Hard" braking was only -1.44 m/s^2** (too gentle). Real emergency braking should be -5 to -8 m/s^2. The script classified it as "hard" but it was barely beyond light braking.

Recording #2 is supposed to fix these by:
- **Roof-mounting all 3 boards** (taped to car roof for clear line-of-sight)
- **Lead driver stomps the brake hard** on signal (target: at least -3.0 m/s^2, ideally -5 m/s^2+)

### What This Data Is Used For (The Big Picture)

1. **SUMO base scenario** for augmentation pipeline: real GPS trajectories become the mandatory base that synthetic training scenarios are built on top of. The professor requires this — pure synthetic data was rejected.
2. **Emulator calibration:** packet loss rates, latency, burst loss behavior, and sensor noise under real driving conditions feed into `emulator_params_measured.json`, which the training emulator uses.
3. **Offline model validation:** real convoy data is fed through the trained model's observation pipeline to verify the model produces sensible outputs on real-world inputs.
4. **Thesis figures:** GPS tracks, braking event plots, PDR over time, sensor noise comparisons.

**If this recording is bad, the entire training pipeline is blocked.** There is no alternative path.

### Recording Protocol (Expected Ground Truth)

The convoy drive follows this sequence. Use this to validate event detection:

1. **Steady cruising** at 10-30 km/h (3-8 m/s) in convoy formation (10-15m spacing)
2. **Light braking event #1** — lead car brakes gently
3. **Resume cruising**
4. **Light braking event #2** — lead car brakes gently again
5. **Resume cruising**
6. **Hard braking event** — lead car stomps brake hard (this is the critical hazard scenario, target < -3.0 m/s^2)
7. **Parked stationary ~60 seconds** — all 3 cars stopped, engines idle, logging continues
8. **Logging stopped**

---

## Step 1: Run the Analysis Script

The full analysis script already exists and handles everything. Run it with the new recording directory.

```bash
cd /home/amirkhalifa/RoadSense2/roadsense-v2v/ml
source venv/bin/activate
python scripts/analyze_convoy_recording.py \
    --input-root <RECORDING_DIR> \
    --out-dir /home/amirkhalifa/RoadSense2/roadsense-v2v/ml/data/convoy_analysis_rec2/
```

Where `<RECORDING_DIR>` is the directory Amir created for Recording #2 (ask him for the path if not obvious — it should be under `/home/amirkhalifa/RoadSense2/` and follow the naming pattern `Convoy_recording_<DATE>/`).

The directory structure should be:
```
<RECORDING_DIR>/
├── V001/
│   ├── V001_tx_<NNN>.csv
│   └── V001_rx_<NNN>.csv
├── V002/
│   ├── V002_tx_<NNN>.csv
│   └── V002_rx_<NNN>.csv
└── V003/
    ├── V003_tx_<NNN>.csv
    └── V003_rx_<NNN>.csv
```

The `<NNN>` suffix is a session counter. The script auto-picks the highest-numbered file per vehicle/type.

**If the script fails**, read the error carefully. Common issues:
- Missing file for a vehicle → a board didn't log (hardware failure)
- Parse errors → corrupted SD card write
- Import errors → venv not activated or scipy/numpy missing

**The script produces these output files:**

| File | What It Contains |
|------|-----------------|
| `data_quality_summary.json` | File integrity, GPS fix %, sensor status, temporal alignment |
| `analysis_summary.json` | PDR per link, latency, burst loss, sensor noise, formation stats, observation ranges |
| `convoy_events.json` | Detected braking events, stationary tail, hard event details |
| `convoy_emulator_params.json` | Proposed emulator parameters derived from this recording |
| `convoy_analysis_report.md` | Human-readable markdown report |
| `convoy_observations.npz` | Observation arrays for offline model inference |
| `figures/*.png` | 8 publication-quality plots |

---

## Step 2: Read and Interpret the Outputs

After the script finishes, read the three JSON files and the markdown report. Evaluate each criterion below against the output data.

### 2.1 File Integrity (from `data_quality_summary.json` → `files` array)

For each of the 6 files, check:

| Field | Pass | Fail |
|-------|------|------|
| `rows_valid` | >= 500 per file (50s at 10 Hz) | < 100 rows on any TX file |
| `malformed_rows` | < 5% of `rows_total` | > 10% malformed |
| `gps_fix_pct` | >= 95% on all TX files | < 80% on any TX file |
| `gps_jumps_gt_50m` | 0 | > 3 jumps (multipath artifacts) |
| `gaps_gt_500ms` | <= 3 per file | > 10 per file (logging instability) |
| `sensor_status` | Not "FAIL" | "FAIL" on any file |

**Report:** List all 6 files with their row count, GPS fix %, gaps, and sensor status.

**Recording #1 reference:** All files had 0 malformed rows, 100% GPS fix, 2000+ rows each. These numbers should be similar.

### 2.2 Temporal Alignment (from `data_quality_summary.json` → `temporal_alignment`)

| Field | Pass | Fail |
|-------|------|------|
| `overlap_duration_s` | >= 100 seconds | < 60 seconds |

**Recording #1 reference:** Overlap was 200.7 seconds. New recording should be similar or longer.

### 2.3 Packet Delivery Ratio — THE CRITICAL CHECK (from `analysis_summary.json` → `pdr_by_link`)

**This is the check that failed in Recording #1.** Check ALL 6 directional links:

| Link | Recording #1 PDR | Recording #2 Target |
|------|------------------|-------------------|
| V001 -> V002 | **15.9% (FAILED)** | >= 80% |
| V001 -> V003 | 92.9% | >= 80% |
| V002 -> V001 | **13.9% (FAILED)** | >= 80% |
| V002 -> V003 | 91.2% | >= 80% |
| V003 -> V001 | 93.6% | >= 80% |
| V003 -> V002 | 98.1% | >= 80% |

For each link in the `pdr_by_link` object, read the `pdr` field (a float between 0 and 1).

**GO criteria:**
- **ALL 6 links have `pdr` >= 0.80** (80% delivery, 20% loss max)
- No single link below 0.70

**NO-GO criteria (must re-record):**
- **Any link has `pdr` < 0.50** → board placement failure (same problem as Recording #1)
- **More than 1 link has `pdr` < 0.70** → systemic issue

**If a link fails:** Tell Amir which specific vehicle pair has the problem. The fix is to move the board on the affected vehicle(s) to the center of the roof, away from metal edges. Then re-record.

### 2.4 Burst Loss Analysis (from `analysis_summary.json` → `burst_by_link`)

Check `max_burst_length` and `mean_burst_length` for each link:

| Field | Healthy (Rec #1) | Shielded (Rec #1) | Target |
|-------|-------------------|---------------------|--------|
| `mean_burst_length` | 1.1 - 1.4 | 14.3 - 14.8 | < 3.0 |
| `max_burst_length` | 2 - 15 | 154 - 244 | < 30 |

**Warning signs:** If any link has `mean_burst_length` > 5, it suggests intermittent shielding or distance problems, even if overall PDR looks OK.

### 2.5 Hard Braking Event — SECOND MOST IMPORTANT (from `convoy_events.json`)

Read `detected.hard_event`:

| Field | Pass | Fail |
|-------|------|------|
| `min_accel_x_ms2` | < -3.0 m/s^2 | > -2.0 m/s^2 |
| `duration_s` | >= 0.5 seconds | < 0.3 seconds (too brief to be a real event) |

Also check `detected.events` array — there should be:
- At least 2 light braking events (`event_type: "light_braking"`)
- At least 1 hard braking event (`event_type: "hard_braking"`)
- The hard event should have `min_accel_x_ms2` well below -2.0

**Recording #1 reference:** The "hard" event was only -1.44 m/s^2 (classified as hard by the script, but too gentle for real emergency braking). Recording #2 must do significantly better.

**If the hard braking is between -2.0 and -3.0 m/s^2:** This is a WARNING. The data is usable but suboptimal. Tell Amir it would be better to try again with harder braking if convenient, but it's not a strict failure.

**If the hard braking is weaker than -2.0 m/s^2:** This is effectively the same problem as Recording #1. Tell Amir the lead driver needs to brake much harder — stomp the pedal, not tap it.

### 2.6 Stationary Tail (from `convoy_events.json` → `detected.stationary_tail`)

| Field | Pass | Warning |
|-------|------|---------|
| `duration_s` | >= 45 seconds | 20-45 seconds |

This segment is used for sensor noise characterization (clean stationary baseline). Longer is better. Recording #1 had ~42 seconds (target was 60).

### 2.7 Sensor Noise (from `analysis_summary.json` → `sensor_noise_aggregate`)

Verify the driving noise values are plausible (not zero, not wildly different from Recording #1):

| Metric | Recording #1 (driving) | Plausible Range |
|--------|----------------------|----------------|
| `gps_std_m` | 2.96 | 1.0 - 8.0 |
| `accel_std_ms2` | 0.39 | 0.1 - 2.0 |
| `gyro_std_rad_s` | 0.023 | 0.005 - 0.1 |
| `heading_std_deg` | 4.71 | 1.0 - 15.0 |
| `speed_std_ms` | 0.43 | 0.05 - 2.0 |

If any value is exactly 0.0, the corresponding sensor failed on that vehicle.

### 2.8 Formation Analysis (from `analysis_summary.json` → `formation`)

Check `avg_spacing_m`:

| Value | Interpretation |
|-------|---------------|
| 5 - 20 m | Tight convoy (ideal) |
| 20 - 40 m | Moderate convoy (OK) |
| > 50 m | Too spread out (vehicles weren't really in convoy) |

Recording #1 had avg spacing of 16.7m ("moderate"). Target for Recording #2 is tighter: 10-15m.

### 2.9 Speed Profile (from `data_quality_summary.json` → files → check `speed_gps_corr` on TX files)

Each TX file should show that the vehicle actually moved:
- `speed_gps_corr` should be positive and ideally > 0.5 (GPS speed correlates with position changes)
- Max speed should be between 3-15 m/s (10-54 km/h) for a normal convoy drive

**Recording #1 reference:** V001 had weak speed-GPS correlation (0.15), V002 and V003 were ~0.22. These are low because of GPS noise at low speeds. As long as vehicles clearly moved (non-zero speeds), this is OK.

---

## Step 3: Produce the GO/NO-GO Verdict

After evaluating all criteria, produce a clear summary in this format:

```
============================================================
           CONVOY RECORDING #2 — FIELD VALIDATION
============================================================

CRITICAL CHECKS:
  File Integrity (6 files):     [PASS / FAIL]
  GPS Fix (all vehicles):       [PASS / FAIL]
  PDR - ALL 6 links >= 80%:     [PASS / FAIL]  ← most important
  Hard Braking < -3.0 m/s^2:    [PASS / FAIL / WARNING]

SECONDARY CHECKS:
  Overlap Duration >= 100s:     [PASS / FAIL]
  Burst Loss (mean < 3.0):      [PASS / WARNING]
  Stationary Tail >= 45s:       [PASS / WARNING]
  Sensor Liveness (non-zero):   [PASS / FAIL]
  Formation Spacing < 20m:      [PASS / WARNING]

────────────────────────────────────────────────────────────

PDR BREAKDOWN (the number that matters most):
  V001 -> V002:  XX.X%  [OK / FAIL]
  V001 -> V003:  XX.X%  [OK / FAIL]
  V002 -> V001:  XX.X%  [OK / FAIL]
  V002 -> V003:  XX.X%  [OK / FAIL]
  V003 -> V001:  XX.X%  [OK / FAIL]
  V003 -> V002:  XX.X%  [OK / FAIL]
  Overall:       XX.X%

  Recording #1 comparison:
    V001<->V002 was ~15% (FAILED - board shielded by car body)
    Healthy links were ~92-98%

BRAKING EVENT:
  Hardest event: -X.XX m/s^2 (duration: X.Xs)
  Recording #1 was -1.44 m/s^2 (too gentle)
  Target: < -3.0 m/s^2

────────────────────────────────────────────────────────────

VERDICT: [GO — Recording accepted, pack up and go home]
     or: [NO-GO — Must re-record. Fix: <specific instructions>]

============================================================
```

### Decision Rules:

**GO** if ALL of these are true:
1. All 6 files exist with >= 500 valid rows each
2. GPS fix >= 95% on all TX files
3. ALL 6 PDR links >= 80%
4. At least one hard braking event detected with `min_accel_x_ms2 < -2.0`
5. Overlap duration >= 100 seconds
6. No sensor failures (all IMU/mag/GPS producing non-zero data)

**GO with warnings** if:
- Hard braking is between -2.0 and -3.0 m/s^2 (usable but suboptimal)
- Stationary tail is 20-45 seconds instead of 60+
- One link has PDR between 70-80% (marginal)
- Formation spacing > 20m

**NO-GO** if ANY of these are true:
- Any file is missing or has < 100 rows
- Any link has PDR < 50% (board placement failure — tell Amir which vehicle)
- More than 1 link has PDR < 70%
- GPS fix < 80% on any vehicle
- No braking event detected at all (driving protocol wasn't followed)
- Overlap duration < 60 seconds

### If NO-GO, provide specific fix instructions:

**PDR failure:**
> "Link V00X -> V00Y has PDR of XX%. This means the board on V00X (or V00Y) is being shielded. Move the board to the CENTER of the car roof, use tape to secure it. Make sure no metal is between the board and other vehicles. Then re-record."

**Braking failure:**
> "Hardest braking was -X.XX m/s^2. This is too gentle. The lead driver needs to STOMP the brake pedal — not a gentle tap, a real emergency stop. At 30 km/h, a hard brake should produce -5 to -8 m/s^2."

**GPS failure:**
> "V00X has only XX% GPS fix. Check that the GPS antenna has clear sky view. The NEO-6M needs 5V power (USB pin, not 3.3V). Wait for satellite lock (blue LED blinks) before starting the drive."

---

## Step 4: If GO — Run Emulator Calibration

If the verdict is GO, also run the calibration script to produce updated emulator parameters:

```bash
cd /home/amirkhalifa/RoadSense2/roadsense-v2v/ml
source venv/bin/activate
python scripts/calibrate_emulator_convoy.py \
    --input_dir <RECORDING_DIR> \
    --output /home/amirkhalifa/RoadSense2/roadsense-v2v/ml/espnow_emulator/emulator_params_convoy_rec2.json
```

**Important:** In Recording #1, the calibration script excluded V001<->V002 links because of the placement issue. If Recording #2 has all 6 links healthy, you should modify the script's `HEALTHY_LINKS` and `EXCLUDED_LINKS` lists (or check if it auto-detects) to use ALL 6 links for calibration. If you need to edit the script, update the constants at the top of `calibrate_emulator_convoy.py`:

```python
HEALTHY_LINKS = [
    ("V001", "V002"),
    ("V001", "V003"),
    ("V002", "V001"),
    ("V002", "V003"),
    ("V003", "V001"),
    ("V003", "V002"),
]
EXCLUDED_LINKS = []  # None excluded — all healthy
```

Compare the new params against the current values in `emulator_params_measured.json` and briefly note any significant changes (e.g., loss rate changed from 5.8% to X%).

---

## Important Notes

- **V002 was ego in Recording #1.** Ask Amir which vehicle is ego in Recording #2. The script's event detection and observation building align to the ego vehicle. If ego changed, the `convoy_events.json` → `expected_protocol.ego_vehicle` field may need updating.
- **ESP32 clocks are independent.** Do NOT assume clock synchronization. The script handles alignment using `msg_timestamp` matching and clock offset estimation.
- **GPS (0.0, 0.0) = no fix.** These rows are excluded automatically.
- **Speed is in m/s** in the CSV. 10 m/s = 36 km/h.
- **accel_x is longitudinal.** Negative = braking. The axis orientation depends on board mounting — if the board is mounted differently than Recording #1, the braking axis might be `accel_y` or `accel_z` instead. If you see a strong braking event on a different axis but `accel_x` is flat, tell Amir.
- **Do NOT overwrite the Recording #1 analysis.** Use a separate output directory (`convoy_analysis_rec2/`).
- **The existing `analyze_convoy_recording.py` script defaults to looking for `Convoy_recording_02212026/`.** You MUST pass `--input-root` to point to the new recording directory.

---

## Reference: Recording #1 Key Numbers (for comparison)

| Metric | Recording #1 | Recording #2 Target |
|--------|-------------|-------------------|
| Overlap duration | 200.7s | >= 120s |
| V001->V002 PDR | 15.9% | >= 80% |
| V001->V003 PDR | 92.9% | >= 80% |
| V002->V001 PDR | 13.9% | >= 80% |
| V002->V003 PDR | 91.2% | >= 80% |
| V003->V001 PDR | 93.6% | >= 80% |
| V003->V002 PDR | 98.1% | >= 80% |
| Overall healthy PDR | 93.8% | >= 85% (all links) |
| Hardest braking | -1.44 m/s^2 | < -3.0 m/s^2 |
| Braking events found | 3 (2 light, 1 "hard") | >= 3 |
| Stationary tail | 41.8s | >= 45s |
| Avg formation spacing | 16.7m | 10-15m ideal |
| GPS fix (all vehicles) | 100% | >= 95% |
| Rows per TX file | ~2000-2090 | >= 1500 |
| V001->V002 mean burst | 14.8 | < 3.0 |
| Healthy link mean burst | 1.1-1.4 | < 3.0 |
