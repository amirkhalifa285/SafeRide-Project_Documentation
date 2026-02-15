# Phase 6: Real Data Pipeline - Production Model Path

**Created:** January 24, 2026
**Status:** ACTIVE - IMMEDIATE PRIORITY
**Goal:** Complete the real-data grounded training pipeline for Run 002 (production model)

---

## Executive Summary

This plan addresses the professor's critical feedback: training must be grounded in real recorded data, not pure synthetic scenarios. We will:
1. Record real ESP-NOW network characteristics (RTT + sensor noise)
2. Record a real 3-car convoy scenario (trajectory + sensor patterns)
3. Use these as the foundation for augmentation and training

---

## Prerequisites

- [x] RTT firmware validated (5m stationary test complete)
- [x] RTT processing script works
- [x] DataLogger Mode 1 (TX/RX) code exists
- [x] Phase 6.1 firmware/logging implementation integrated (Feb 12, 2026)
- [x] Test suite updated and passing on user machine (92 tests)
- [ ] 3 ESP32 units with SD cards ready
- [ ] Safe location for convoy recording identified

---

## Phase 6.1: Enhanced RTT Recording

**Objective:** Capture network characteristics AND sensor noise in one recording.

Status (Feb 12, 2026):
- [x] 6.1.1 code changes complete
- [ ] 6.1.2 field capture pending
- [ ] 6.1.3 measured params regeneration pending

### 6.1.1 Enhance RTT Firmware with GPS + Mag Logging

**Files to modify:**
- `hardware/test/espnow_rtt/sender_main.cpp`
- `hardware/test/espnow_rtt/RTTLogging.h`
- `hardware/test/espnow_rtt/RTTPacket.h`

**Add to RTTPacket:**
```cpp
// Existing fields...
float mag_x, mag_y, mag_z;    // Magnetometer (heading source)
float gps_hdop;               // GPS accuracy indicator
uint8_t gps_satellites;       // Satellite count
```

**Add to CSV output:**
```
sequence,send_time_ms,recv_time_ms,rtt_ms,lat,lon,speed,heading,
accel_x,accel_y,accel_z,gyro_x,gyro_y,gyro_z,mag_x,mag_y,mag_z,
hdop,satellites,lost
```

### 6.1.2 Execute RTT Recording

**Protocol:**
1. V001 (Sender) + V002 (Reflector) in open area
2. Start at 1m, move apart incrementally: 1m, 5m, 10m, 15m, 20m, 30m
3. At each distance: stand still for 60 seconds
4. Total recording: ~10-15 minutes
5. Include some walking/driving segments for dynamic data

**Exit Criteria:**
- [ ] `/rtt_log.csv` with 5000+ rows
- [ ] Loss rate data at multiple distances
- [ ] GPS, IMU, and magnetometer columns populated

### 6.1.3 Process RTT Recording

**Script:** `ml/scripts/analyze_rtt_characterization.py` (updated for mag analysis)

**Output:** Updated `ml/espnow_emulator/emulator_params_measured.json`
```json
{
  "latency": {
    "base_ms": <measured>,
    "distance_factor": <measured>,
    "jitter_std_ms": <measured>
  },
  "packet_loss": {
    "base_rate": <measured at 1m>,
    "rate_tier_1": <measured at 10m>,
    "rate_tier_2": <measured at 30m>,
    ...
  },
  "sensor_noise": {
    "gps_std_m": <measured from stationary segments>,
    "heading_std_deg": <measured from mag variance>,
    "accel_std_ms2": <measured from IMU variance>,
    "gyro_std_rad_s": <measured from gyro variance>
  }
}
```

---

## Phase 6.2: 3-Car Convoy Recording

**Objective:** Capture real trajectory and sensor data for augmentation base.

### 6.2.1 Firmware Preparation

**Option A (No code changes):**
- Use existing `MODE_NETWORK_CHARACTERIZATION`
- Each vehicle logs TX and RX files
- Post-process alignment by GPS position matching

**Option B (Recommended - GPS time):**
- Add GPS UTC time to log format (~20 lines change)
- Provides automatic time alignment across vehicles

**Files to modify (Option B):**
- `hardware/src/logging/DataLogger.cpp` - Add `getGpsTimeMs()` function

### 6.2.2 Execute Convoy Recording

**Protocol:**
1. V001, V002, V003 in convoy formation (10-20m spacing)
2. Safe location: parking lot or quiet road
3. Maneuvers to include:
   - Steady-state driving (30 seconds)
   - Acceleration from stop
   - Gradual braking
   - Hard braking (simulated hazard)
   - Gentle curves (if possible)
4. Duration: 5-10 minutes
5. Speed: 10-30 km/h

**Exit Criteria:**
- [ ] 6 files total (2 per vehicle: TX + RX)
- [ ] GPS fix on all vehicles throughout
- [ ] At least one hard braking event captured
- [ ] Trajectories show realistic convoy behavior

### 6.2.3 Process Convoy Recording

**New Script:** `ml/scripts/process_convoy_recording.py`

**Inputs:**
- `v001_tx.csv`, `v001_rx.csv`
- `v002_tx.csv`, `v002_rx.csv`
- `v003_tx.csv`, `v003_rx.csv`

**Processing Steps:**
1. Load all 6 CSV files
2. Align timestamps (GPS time or position matching)
3. Extract trajectories for each vehicle
4. Convert GPS coordinates to SUMO format
5. Generate route file with real depart times and positions

**Output:** `ml/scenarios/base_real/`
```
base_real/
├── network.net.xml      (from OSM of recording location)
├── routes.rou.xml       (from parsed trajectories)
├── scenario.sumocfg
└── recording_metadata.json
```

---

## Phase 6.3: Update Augmentation Pipeline

**Objective:** Use real recording as mandatory base for augmentation.

### 6.3.1 Modify gen_scenarios.py

**Current behavior:** Takes synthetic SUMO base, augments parameters
**New behavior:** Requires real base, augments on top

**Changes:**
- Add `--real_base` flag (required for production runs)
- Validate base has `recording_metadata.json`
- Preserve core trajectory shape, only jitter parameters

### 6.3.2 Generate Dataset v2

**Command:**
```bash
./ml/run_docker.sh generate \
  --base_dir ml/scenarios/base_real \
  --output_dir ml/scenarios/datasets/dataset_v2 \
  --seed 42 \
  --train_count 20 \
  --eval_count 5 \
  --peer_drop_prob 0.3 \
  --min_peers 1 \
  --emulator_params ml/espnow_emulator/emulator_params_measured.json
```

**Exit Criteria:**
- [ ] 20 training scenarios with variable n (1 to 8+ peers)
  - Base recording provides 2 real peers
  - Augmentation DROPS peers (n=1, n=2)
  - Augmentation ADDS synthetic peers (n=3, 4, 5, 6, 7, 8) based on real trajectory patterns
- [ ] 5 eval scenarios with variable n
- [ ] All scenarios grounded in real trajectory base
- [ ] Manifest includes `real_base: true` flag

---

## Phase 6.4: Cone Filtering Implementation

**Objective:** Filter peers to front FOV (V001 only) per professor's requirement.

### 6.4.1 Simulation Side

**File:** `ml/envs/observation_builder.py`

**Add function:**
```python
def is_in_cone(ego_heading_deg, bearing_to_peer_deg, half_angle=45.0) -> bool:
    """Return True if peer is within ±half_angle of ego's heading."""
    diff = (bearing_to_peer_deg - ego_heading_deg + 180) % 360 - 180
    return abs(diff) <= half_angle
```

**Modify `build()`:** Filter `peer_observations` before processing.

### 6.4.2 Embedded Side

**New files:**
- `hardware/src/inference/ConeFilter.h`
- `hardware/src/inference/ConeFilter.cpp`

**Integration point:** Before AI inference, filter `peerStates` vector.

### 6.4.3 Unit Tests

- `ml/tests/test_cone_filter.py` - Python tests
- `hardware/test/test_cone_filter/` - Native C++ tests

**Exit Criteria:**
- [ ] Python cone filter implemented and tested
- [ ] C++ cone filter implemented and tested
- [ ] Identical behavior verified (same inputs → same filter decisions)

---

## Phase 6.5: ML Code Finalization

**Objective:** Ensure training pipeline is production-ready.

### 6.5.1 Verify Deep Sets Policy

- [ ] Handles variable n (1 to 8 peers)
- [ ] Cone-filtered observations integrated
- [ ] Evaluation callback logs per-scenario results

### 6.5.2 Update Training Script

- [ ] Load `emulator_params_measured.json`
- [ ] Use `dataset_v2` (real-grounded)
- [ ] Set timesteps to 10M
- [ ] Configure checkpointing every 500K steps

---

## Phase 6.6: Training Run 002 (Production)

**Objective:** Train production model with real-grounded data.

### 6.6.1 Pre-flight Checklist

- [ ] `emulator_params_measured.json` created from RTT recording
- [ ] `dataset_v2` generated from real convoy base
- [ ] Cone filtering integrated in ConvoyEnv
- [ ] EC2 instance ready (or local GPU)

### 6.6.2 Execute Training

**Command:**
```bash
python ml/scripts/train.py \
  --dataset ml/scenarios/datasets/dataset_v2 \
  --emulator_params ml/espnow_emulator/emulator_params_measured.json \
  --timesteps 10000000 \
  --eval_freq 100000 \
  --checkpoint_freq 500000 \
  --output_dir ml/models/runs/run_002
```

### 6.6.3 Success Criteria

- [ ] Training completes 10M timesteps
- [ ] Eval success rate > 85% (target: 90%+)
- [ ] Model handles variable n (tested on n=1,2,3,4,5)
- [ ] No regression on n=2 (compare to Run 001)

---

## Timeline Summary

| Phase | Description | Estimated Effort |
|-------|-------------|------------------|
| 6.1 | Enhanced RTT Recording | 1 session (firmware + recording) |
| 6.2 | 3-Car Convoy Recording | 1 session (prep + recording) |
| 6.3 | Augmentation Pipeline Update | 0.5 session |
| 6.4 | Cone Filtering | 1 session |
| 6.5 | ML Code Finalization | 0.5 session |
| 6.6 | Training Run 002 | 1 session (launch + monitor) |

**Total:** ~5 sessions to production model

---

## Dependencies Graph

```
[6.1 RTT Recording] ──────────────────────────┐
        │                                      │
        ▼                                      ▼
[6.1.3 Process RTT] ──► [emulator_params_measured.json]
                                               │
[6.2 Convoy Recording]                         │
        │                                      │
        ▼                                      │
[6.2.3 Process Convoy] ──► [base_real/]        │
        │                                      │
        ▼                                      │
[6.3 Update Augmentation] ◄────────────────────┘
        │
        ▼
[6.3.2 Generate dataset_v2]
        │
        ├──────────────────────────────────────┐
        │                                      │
        ▼                                      ▼
[6.4 Cone Filtering]                   [6.5 ML Finalization]
        │                                      │
        └──────────────┬───────────────────────┘
                       │
                       ▼
              [6.6 Training Run 002]
                       │
                       ▼
              [Production Model]
```

---

## Risk Mitigation

| Risk | Mitigation |
|------|------------|
| GPS fix issues during recording | Test GPS lock before convoy starts; abort if < 4 satellites |
| SD card write failures | Use 10 MHz SPI; test cards before recording |
| Clock sync drift | Use GPS time (Option B) or position-based alignment |
| Convoy safety | Use empty parking lot; maintain safe speed |
| Recording location map unavailable | Use OSM export; simplify to straight road if needed |

---

## Document History

- **Jan 24, 2026:** Created based on planning session analysis
