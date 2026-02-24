# CONVOY_DATA_INTERPRETER Agent Prompt

**Purpose:** Systematic analysis of the 3-car convoy field recording (6 CSV files) to extract maximum value for model validation, emulator refinement, and thesis content.

**Last Updated:** February 24, 2026

---

## Context

A 3-car convoy recording has been completed. Three ESP32 units (V001, V002, V003) drove in convoy formation, each logging transmitted and received V2V messages to SD card at ~10 Hz.

**Critical:** In this recording, **V002 is the ego vehicle** (the car that will run the RL model in the final demo). V001 and V003 are peer broadcasters.

### Recording Protocol (Known Ground Truth)

The convoy drive followed this sequence — use this as ground truth when validating event detection:

1. **Steady cruising** — convoy driving in formation
2. **Light braking event #1** — lead car braked gently (mid-drive)
3. **Resume cruising**
4. **Light braking event #2** — lead car braked gently again
5. **Resume cruising**
6. **Hard braking event** — lead car braked hard at end of drive (this is the critical hazard scenario)
7. **Parked stationary ~60 seconds** — all 3 cars stopped, engines idle, logging continued
8. **Logging stopped** — end of recording

**Why this matters:**
- The **hard braking event** is the most valuable segment — it's the exact scenario the model must detect in the final demo. Analyze packet delivery, latency, and sensor behavior during this event in detail.
- The **two light braking events** provide intermediate reference points for comparing model response at different deceleration magnitudes.
- The **60-second stationary window** at the end is a gift for sensor noise characterization — use it the same way the RTT stationary window was used, but now with all 3 vehicles and convoy-warmed sensors.
- Event detection (Task 5.3) should find exactly these events. If it finds fewer or more, investigate why.

---

## Input Files

You will be given 6 CSV files:

| File | Description |
|------|-------------|
| `v001_tx.csv` | Messages V001 broadcast |
| `v001_rx.csv` | Messages V001 received from V002 and V003 |
| `v002_tx.csv` | Messages V002 (ego) broadcast |
| `v002_rx.csv` | Messages V002 (ego) received from V001 and V003 |
| `v003_tx.csv` | Messages V003 broadcast |
| `v003_rx.csv` | Messages V003 received from V001 and V002 |

### CSV Column Format (16 columns)

```
timestamp_local_ms, msg_timestamp, vehicle_id, lat, lon, speed, heading, accel_x, accel_y, accel_z, gyro_x, gyro_y, gyro_z, mag_x, mag_y, mag_z
```

| Col | Name | Unit | Notes |
|-----|------|------|-------|
| 1 | timestamp_local_ms | ms | Local ESP32 `millis()` at log time |
| 2 | msg_timestamp | ms | Timestamp embedded in the V2V message |
| 3 | vehicle_id / from_vehicle_id | string | Sender ID. In TX logs: self. In RX logs: originating peer |
| 4 | lat | degrees | WGS84 latitude |
| 5 | lon | degrees | WGS84 longitude |
| 6 | speed | m/s | Vehicle speed |
| 7 | heading | degrees | 0-359, true north |
| 8-10 | accel_x/y/z | m/s^2 | MPU6500 accelerometer |
| 11-13 | gyro_x/y/z | rad/s | MPU6500 gyroscope |
| 14-16 | mag_x/y/z | uT | QMC5883L magnetometer |

---

## Task 1: Data Validation & Sanity Checks

Before any analysis, verify the data is usable. Check and report:

### 1.1 File Integrity
- Row count per file (expect ~6000 rows per minute at 10 Hz)
- Any malformed rows (wrong column count, parse errors)
- Missing or empty fields (especially GPS: lat/lon = 0.0 means no fix)
- Duplicate timestamps

### 1.2 Temporal Alignment
- Recording start/end time per vehicle (from `timestamp_local_ms`)
- Overlap window: what time range do all 3 vehicles have data?
- Clock drift: ESP32 `millis()` clocks are independent. Estimate drift between vehicles by correlating GPS positions or RX timestamps vs TX timestamps.
- Flag any large gaps (>500ms) in any TX log — indicates logging hiccup or reboot.

### 1.3 GPS Quality
- How many rows have valid GPS fix (lat != 0, lon != 0)?
- GPS fix percentage per vehicle
- Plot GPS tracks for all 3 vehicles on same axes — do they follow the same road?
- Any GPS jumps > 50m between consecutive readings (multipath/cold start artifacts)?

### 1.4 Sensor Liveness
- Are IMU fields (accel, gyro) non-zero and varying? A flat line means sensor failure.
- Are mag fields non-zero? (QMC5883L may report zeros if initialization failed)
- Speed: does it correlate with GPS position changes?

**Output:** A validation summary table: file, rows, GPS fix %, time range, sensor status (OK/WARN/FAIL).

---

## Task 2: Packet Delivery Analysis (Convoy-Specific Emulator Params)

This is the primary networking analysis. Compare what was sent vs what was received.

### 2.1 Packet Delivery Ratio (PDR)

For each directional link, compute PDR:

| Link | Method |
|------|--------|
| V001 → V002 | Count V001 rows in `v002_rx.csv` / total rows in `v001_tx.csv` (within overlap window) |
| V001 → V003 | Count V001 rows in `v003_rx.csv` / total rows in `v001_tx.csv` |
| V002 → V001 | Count V002 rows in `v001_rx.csv` / total rows in `v002_tx.csv` |
| V002 → V003 | Count V002 rows in `v003_rx.csv` / total rows in `v002_tx.csv` |
| V003 → V001 | Count V003 rows in `v001_rx.csv` / total rows in `v003_tx.csv` |
| V003 → V002 | Count V003 rows in `v002_rx.csv` / total rows in `v003_tx.csv` |

Match packets by `msg_timestamp` (should be unique per sender).

**Report:**
- PDR per link (percentage)
- Overall PDR (all links combined)
- Is PDR symmetric? (V001→V002 vs V002→V001)
- PDR over time (sliding window of 10 seconds) — plot it. Look for dropout zones.

### 2.2 Latency Estimation

For each received packet, compute one-way latency:
```
latency = rx.timestamp_local_ms - rx.msg_timestamp
```

**Caveat:** This mixes clock domains (receiver's local clock vs sender's message timestamp). If `msg_timestamp` is the sender's `millis()`, this latency includes clock offset. Compute **relative latency** (distribution shape, jitter) rather than absolute values.

**Report:**
- Latency distribution: mean, median, p95, p99, std
- Per-link latency comparison
- Latency vs time plot — any trends or spikes?

### 2.3 Burst Loss Analysis

Examine consecutive lost packets:
- For each TX sequence, mark received/lost
- Compute burst lengths (consecutive lost packets)
- Mean burst length, max burst length
- Compare against current emulator params: `mean_burst_length = 2.106`

### 2.4 Distance-Dependent Loss

Using GPS positions from TX logs:
- For each packet, compute inter-vehicle distance at send time
- Bin by distance (0-20m, 20-50m, 50-100m, 100m+)
- Compute PDR per distance bin
- Compare against current emulator tiers: base=0.21, tier1=0.31, tier2=0.57

**Output:** A proposed update to the emulator params JSON with convoy-measured values:
```json
{
  "latency": { "base_ms": ..., "jitter_std_ms": ... },
  "packet_loss": { "base_rate": ..., ... },
  "burst_loss": { "mean_burst_length": ..., ... }
}
```

---

## Task 3: Sensor Noise Characterization (Driving Conditions)

Current emulator sensor noise params were measured from a **stationary window** in the RTT log. The convoy data provides noise under **real driving vibration**.

### 3.1 Identify Segments

Segment the convoy data by driving state:
- **Stationary:** speed < 0.5 m/s (parked/stopped)
- **Cruising:** speed > 5 m/s, |accel_x| < 0.5 m/s^2
- **Braking:** accel_x < -1.0 m/s^2 (or longitudinal deceleration)
- **Accelerating:** accel_x > 1.0 m/s^2

### 3.2 Per-Segment Noise Analysis

For each segment type, on each vehicle's TX log:

**GPS:**
- Compute std of lat/lon differences from a smoothed trajectory (e.g., Savitzky-Golay)
- Convert to meters using Haversine
- Compare to current `gps_std_m = 1.506`

**IMU (Accelerometer):**
- In stationary segments: std of accel_x/y/z (should be close to zero + noise)
- In cruising segments: std of accel residuals after detrending
- Compare to current `accel_std_ms2 = 0.059`

**Gyroscope:**
- Same approach: stationary noise floor vs driving noise
- Compare to current `gyro_std_rad_s = 0.01`

**Magnetometer:**
- Std of mag_x/y/z per segment
- Compute heading from mag: `heading_mag = atan2(mag_y, mag_x)` (simplified, assumes level)
- Heading std in cruising segments
- Compare to current `heading_std_deg = 0.580`, `mag_std_ut = 0.612`

**Output:** Updated `sensor_noise` block for emulator params, with separate columns for stationary vs driving noise.

---

## Task 4: Offline Model Validation (Critical)

Feed real convoy data through the model's observation pipeline to check if the trained model produces sensible outputs on real-world inputs.

### 4.1 Build Observations from V002 RX Log

V002 is the ego vehicle. Using `v002_tx.csv` (ego state) and `v002_rx.csv` (peer messages):

For each timestamp in V002's TX log:
1. **Ego state** (from `v002_tx.csv`):
   ```
   ego = [
       speed / 30.0,
       accel_x / 10.0,       # longitudinal acceleration
       heading_rad / pi,      # convert degrees → radians, normalize
       peer_count / 8.0       # number of peers heard in last 500ms
   ]
   ```

2. **Peer states** (from `v002_rx.csv`, messages received within last 500ms):
   For each peer message not older than 500ms:
   ```
   peer = [
       rel_x / 100.0,        # relative position in ego frame (requires coordinate transform)
       rel_y / 100.0,
       rel_speed / 30.0,     # peer.speed - ego.speed
       rel_heading / pi,     # heading difference, normalized
       peer_accel / 10.0,
       age_ms / 500.0        # staleness of this message
   ]
   ```

3. **Coordinate transform for rel_x, rel_y:**
   ```python
   dx = (peer_lon - ego_lon) * 111320 * cos(ego_lat_rad)  # approx meters
   dy = (peer_lat - ego_lat) * 110540                      # approx meters
   # Rotate into ego heading frame:
   ego_heading_rad = ego_heading * pi / 180
   rel_x = dx * cos(ego_heading_rad) + dy * sin(ego_heading_rad)
   rel_y = -dx * sin(ego_heading_rad) + dy * cos(ego_heading_rad)
   ```

### 4.2 Value Range Check

Before running inference, verify the observations fall within expected ranges:

| Field | Training Range | Check |
|-------|---------------|-------|
| speed / 30.0 | [0, 1] | Real speeds stay under 30 m/s (~108 km/h)? |
| accel_x / 10.0 | [-1, 1] | Accelerations stay under 10 m/s^2? |
| heading / pi | [-1, 1] | Heading conversion correct? |
| rel_x / 100.0 | [-1, 1] | Inter-vehicle distances under 100m? |
| rel_y / 100.0 | [-1, 1] | Lateral offsets reasonable? |
| age_ms / 500.0 | [0, 1] | Message ages under 500ms? Anything stale getting through? |

**Flag any values outside [-1.5, 1.5] — these would be out-of-distribution for the model.**

### 4.3 Run Model Inference (if model available)

If the Run 002 model checkpoint is accessible:
1. Load the model (`DeepSetPolicy`)
2. Feed the real observation sequences
3. Record the action distribution at each timestep
4. **Key question:** During steady cruising, does the model output "maintain" (action 0)? During peer braking events, does it output "brake" (action 3)?

If the model is not available for loading, still produce the observation sequences as `.npy` or `.csv` for later offline inference.

### 4.4 Identify Sim-to-Real Gaps

Compare real observation statistics to training data statistics:
- Distribution of ego speeds (real vs SUMO scenarios)
- Distribution of inter-vehicle distances
- Distribution of message ages / staleness
- Frequency of zero-peer windows (no messages for >500ms)

Any significant distributional shift is a sim-to-real gap that should be documented and potentially addressed in training.

---

## Task 5: Convoy Trajectory Extraction (SUMO Base Scenario)

Extract clean trajectories for use as a real-data base scenario in the augmentation pipeline.

### 5.1 Trajectory Extraction
- Use TX logs from all 3 vehicles (these are ground truth positions)
- Resample to uniform 10 Hz if needed
- Smooth GPS noise (moving average or Kalman filter)
- Output: per-vehicle trajectory as `[(t, lat, lon, speed, heading), ...]`

### 5.2 Coordinate Conversion
- Convert GPS (lat/lon) to local XY coordinates (meters) using a reference point (e.g., first GPS fix of V002)
- This local frame is needed for SUMO `.rou.xml` generation

### 5.3 Event Annotation
- Identify key events in the trajectory:
  - **Hard braking:** accel_x < -2.0 m/s^2 for > 0.5s
  - **Stops:** speed < 0.5 m/s for > 2s
  - **Lane changes:** lateral acceleration spikes or heading changes > 5 deg/s
  - **Formation changes:** inter-vehicle distance changing significantly
- Annotate timestamps of these events — they become the "interesting" segments for evaluation

### 5.4 Formation Analysis
- Plot inter-vehicle distances over time (V001↔V002, V002↔V003, V001↔V003)
- Average convoy spacing
- Was the convoy tight (10-15m) or loose (30-50m)?
- Did the formation hold or did vehicles drift apart?

**Output:** Cleaned trajectory files + event annotations + formation statistics.

---

## Task 6: Thesis-Quality Figures

Generate publication-ready plots. Use matplotlib with clean styling (no default blue, use a consistent color palette).

### Required Figures:

1. **GPS Track Map** — All 3 vehicles on same plot, color-coded, with start/end markers
2. **Speed Profiles** — Speed vs time for all 3 vehicles, stacked or overlaid
3. **Packet Delivery Over Time** — PDR in 10-second sliding windows, per link
4. **Latency Distribution** — Histogram + CDF of one-way latency
5. **Inter-Vehicle Distance** — Distance between each pair over time
6. **Braking Event Detail** — Zoomed plot of a braking event showing:
   - Lead vehicle deceleration
   - Ego vehicle's received messages (dots)
   - Message latency during the event
   - Missing packets highlighted
7. **Sensor Noise Comparison** — Bar chart: stationary noise vs driving noise vs current emulator params
8. **Observation Value Distributions** — Histograms of real observation values vs expected training ranges

---

## Output Deliverables

At the end of analysis, produce:

1. **`convoy_analysis_report.md`** — Full written report covering Tasks 1-5 with key findings
2. **`convoy_emulator_params.json`** — Updated emulator parameters derived from convoy data
3. **`convoy_observations.npz`** — Pre-built observation arrays from V002's perspective for offline model inference
4. **`convoy_events.json`** — Annotated timeline of braking/acceleration events
5. **`figures/`** — Directory of all plots (PNG, 300 DPI)
6. **`data_quality_summary.json`** — Validation results from Task 1

---

## Reference: Current Emulator Parameters (for comparison)

These are from the Feb 20 RTT drive (2-car, highway). Your convoy analysis should compare against these:

```json
{
  "latency": { "base_ms": 3.84, "jitter_std_ms": 0.971 },
  "packet_loss": { "base_rate": 0.21, "rate_tier_1": 0.31, "rate_tier_2": 0.57 },
  "burst_loss": { "mean_burst_length": 2.106, "max_loss_cap": 0.717 },
  "sensor_noise": {
    "gps_std_m": 1.506,
    "speed_std_ms": 0.021,
    "accel_std_ms2": 0.059,
    "gyro_std_rad_s": 0.01,
    "heading_std_deg": 0.580,
    "mag_std_ut": 0.612
  }
}
```

---

## Key Rules

- **V002 is ego.** All ego-perspective analysis uses V002 data.
- **Do not assume clock synchronization.** ESP32 `millis()` clocks are independent. Use relative timing or GPS-based correlation.
- **GPS coordinates (0.0, 0.0) mean no fix** — exclude these rows from position-based analysis.
- **Heading is in degrees (0-359)** in the CSV. Convert to radians for trigonometric operations.
- **Speed is in m/s** in the CSV. Convert to km/h only for display purposes.
- **All analysis scripts should be saved** in `roadsense-v2v/ml/scripts/` for reproducibility.
