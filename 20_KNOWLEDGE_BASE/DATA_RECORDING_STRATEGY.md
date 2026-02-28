# Data Recording Strategy for RoadSense V2V

**Created:** January 24, 2026
**Purpose:** Document the two-recording strategy and technical constraints for real data collection.

---

## Overview

Training Run 002 (production model) requires real recorded data. This document captures the strategy and technical details discussed on Jan 24, 2026.

---

## Two-Recording Strategy

### Why Two Separate Recordings?

| Recording | Purpose | Vehicles | Output |
|-----------|---------|----------|--------|
| RTT Recording | Network params + sensor noise | 2 (Sender + Reflector) | `emulator_params_measured.json` |
| Convoy Recording | Trajectory base for augmentation | 3 (V001, V002, V003) | `base_real/` scenario |

**Rationale:**
- RTT measurement requires **same-device timestamps** (sender measures both send and receive time)
- Convoy recording has **cross-device timing** which is limited by GPS clock accuracy
- Mixing both compromises RTT precision

---

## Clock Synchronization Analysis

### GPS Time Accuracy (NEO-6M)

| Method | Accuracy | Availability |
|--------|----------|--------------|
| NMEA sentence parsing | ±100-500ms | Always (when GPS fix) |
| PPS (Pulse Per Second) | ±1-10ms | Requires hardware wiring |
| Atomic clock (satellite) | ±50ns | Not directly accessible |

### Why NMEA Has Latency

```
GPS Satellite → Radio → NEO-6M → [Compute Fix] → [Format NMEA] → [UART TX] → ESP32
                                      ↓
                            Time in sentence = fix time
                            But 100-300ms have passed before you read it
```

### Impact on Measurements

| Measurement | Required Accuracy | GPS NMEA Sufficient? |
|-------------|-------------------|----------------------|
| ESP-NOW latency | ~15-50ms | ❌ No (use RTT) |
| Trajectory alignment | ~500ms | ✅ Yes |
| Braking event timing | ~100ms | ⚠️ Marginal |

---

## RTT Recording Details

### What RTT Firmware Measures

The RTT firmware uses **ping-pong protocol**:
```
Sender (V001) → RTTPacket → Reflector (V002)
Sender (V001) ← RTTPacket ← Reflector (V002)

RTT = recv_time_local - send_time_local (both on sender!)
One-way latency ≈ RTT / 2
```

### Current RTT Log Format

```csv
sequence,send_time_ms,recv_time_ms,rtt_ms,lat,lon,speed,heading,accel_x,accel_y,accel_z,lost
```

### Enhancement Needed (Phase 6.1)

Add magnetometer and GPS metadata:
```csv
sequence,send_time_ms,recv_time_ms,rtt_ms,lat,lon,speed,heading,
accel_x,accel_y,accel_z,gyro_x,gyro_y,gyro_z,mag_x,mag_y,mag_z,
hdop,satellites,lost
```

This enables sensor noise characterization from the same recording.

---

## Convoy Recording Details

### What We Capture

Each vehicle logs TX and RX files using `MODE_NETWORK_CHARACTERIZATION`:

- `v001_tx.csv` - All messages V001 sent
- `v001_rx.csv` - All messages V001 received (from V002, V003)
- `v002_tx.csv`, `v002_rx.csv`
- `v003_tx.csv`, `v003_rx.csv`

### Log Format

```csv
timestamp_local_ms,msg_timestamp,vehicle_id,lat,lon,speed,heading,accel_x,accel_y,accel_z
```

### Post-Processing Alignment

Since clocks aren't synchronized, align by GPS position:
```python
# Find when V001 passes a landmark
# Find when V002 passes the same landmark
# Offset = time difference
offset_v002 = find_offset_by_position_matching(v001_log, v002_log)
```

---

## Simulation Data Flow (Reference)

Understanding how SUMO data becomes V2V messages:

```
SUMO TraCI                     to_v2v_message()              ESP-NOW Emulator
┌──────────────┐              ┌──────────────────┐          ┌──────────────┐
│ x, y (meters)│ ────────────►│ lat = y / 111000 │ ────────►│ + GPS noise  │
│ speed (m/s)  │ ────────────►│ speed            │ ────────►│ + speed noise│
│ angle (deg)  │ ────────────►│ heading          │ ────────►│ + heading noise│
│ accel (m/s²) │ ────────────►│ accel_x          │ ────────►│ + accel noise│
│              │              │ accel_y = 0.0    │ ────────►│ (hardcoded)  │
│              │              │ accel_z = 9.81   │ ────────►│ (hardcoded)  │
│              │              │ gyro_* = 0.0     │ ────────►│ (hardcoded)  │
└──────────────┘              └──────────────────┘          └──────────────┘
```

**Key Finding:** `accel_y`, `accel_z`, and `gyro_*` are hardcoded in simulation.

**Current Design Choice:** The observation builder only uses `accel_x` (longitudinal acceleration). Gyro and lateral accel are not in the observation space.

**Open Question (Jan 24, 2026):** Should we expand the observation to include:
- `gyro_z` (yaw rate) - predicts turn trajectory
- `accel_y` (lateral accel) - indicates turning

This would require:
1. Update observation_builder.py (add 2 dims per peer: 6 → 8)
2. Update embedded inference code
3. Derive gyro from heading change rate in simulation: `gyro_z ≈ Δheading / Δt`

Decision pending: Simplicity vs richer observation for Run 002.

---

## Emulator Parameters from Recording

### From RTT Recording

| Parameter | How to Calculate |
|-----------|------------------|
| `latency.base_ms` | Mean RTT/2 at 1m distance |
| `latency.distance_factor` | Slope of RTT vs distance |
| `latency.jitter_std_ms` | Std dev of RTT/2 |
| `packet_loss.base_rate` | Loss rate at 1m |
| `packet_loss.rate_tier_1` | Loss rate at 10m |
| `packet_loss.rate_tier_2` | Loss rate at 30m |
| `sensor_noise.gps_std_m` | Std dev of GPS position during stationary |
| `sensor_noise.heading_std_deg` | Std dev of magnetometer heading during stationary |
| `sensor_noise.accel_std_ms2` | Std dev of accelerometer during stationary |

### From Convoy Recording

- Not used for emulator params (clock sync insufficient)
- Used only for trajectory extraction

---

## Recording Checklist

### Before Recording

- [ ] SD cards formatted (FAT32)
- [ ] SD cards tested (write/read works)
- [ ] GPS fix confirmed on all vehicles
- [ ] Batteries/power sufficient for recording duration
- [ ] Safe location identified

### RTT Recording Protocol

1. Place V001 (Sender) and V002 (Reflector) 1m apart
2. Start logging on both
3. Stand still 60 seconds
4. Move to 5m, repeat
5. Move to 10m, 15m, 20m, 30m
6. Include some walking/driving segments
7. Total: 10-15 minutes

### Convoy Recording Protocol

1. V001, V002, V003 in convoy (10-20m spacing)
2. Start logging on all three
3. Maneuvers:
   - Steady driving (30s)
   - Acceleration from stop
   - Gradual braking
   - Hard braking
   - Gentle curves
4. Duration: 5-10 minutes
5. Speed: 10-30 km/h

---

---

## Mesh-Aware Recording (Feb 26, 2026 Update)

### What Changed

With mesh relay implemented in firmware (Phase E complete, Feb 27 2026), the recording strategy is simplified:

**Old approach:** Record TX/RX on all three vehicles, align clocks post-hoc.
**New approach:** Only record from the ego vehicle. Through mesh relay, ego's RX log captures data from ALL reachable vehicles (direct + relayed).

### How Mesh Changes Recording

```
Convoy: A (front) → B (middle) → C (ego, rear)

A broadcasts its own state (hop=0)
B receives A → cone filter: A is ahead → rebroadcast A (hop=1) + broadcast B's own (hop=0)
C (ego) receives: A via B (hop=1) + B direct (hop=0)

C's RX log contains EVERYTHING.
```

### Recording Protocol (Mesh-Aware)

1. **Only ego (V001) needs to log.** Its RX file captures:
   - Direct messages from immediate neighbors (hop=0)
   - Relayed messages from further vehicles (hop >= 1)
   - Each message includes `sourceMAC` (original sender) and `hopCount`
2. **Other vehicles (V002, V003) only need firmware running.** No SD card required on non-ego.
3. **Deduplication is handled by PackageManager.** If the same message arrives via multiple paths, only the first copy is stored.

### Data Augmentation Strategy

From a 3-vehicle recording (A, B, C where C=ego):
1. C's RX data contains: B's state (direct) + A's state (relayed via B)
2. To augment: add a virtual vehicle D behind C
3. D's input = C's relayed data (C's own state + what C relayed from A and B)
4. This creates a 4-vehicle scenario from a 3-vehicle recording

### Implications for Phase 6 (Recording #2)

- Mount boards on roof (better ESP-NOW range, confirmed by Recording #1 analysis)
- Only V001 needs SD card + logging firmware
- V002/V003 run standard mesh-relay firmware (Phase E)
- Include hard braking maneuvers for hazard scenarios
- Verify mesh relay is working: check for hop=1 messages in V001's RX log

### On-Site Post-Drive Validation (Updated)

Use `analyze_convoy_recording.py` immediately after recording, before leaving the site:

```bash
cd /home/amirkhalifa/RoadSense2/roadsense-v2v/ml
source venv/bin/activate
python scripts/analyze_convoy_recording.py \
  --input-root <RECORDING_DIR> \
  --out-dir /home/amirkhalifa/RoadSense2/roadsense-v2v/ml/data/convoy_analysis_site \
  --no-plots
```

Notes:
- Script now auto-detects input mode:
  - `ego_only`: only `V001_tx_*.csv` + `V001_rx_*.csv` (current workflow)
  - `full`: legacy 6-file layout (`V001/V002/V003` each with tx/rx)
- In ego-only mode, link metrics in `analysis_summary.json -> pdr_by_link` are estimated from V001 RX message spacing (sufficient for on-site GO/NO-GO).

Minimum on-site acceptance checks:
- `data_quality_summary.json -> files`: V001 TX and RX both have `rows_valid >= 500`, low malformed rows, and no logging instability.
- V001 TX shows good GPS (`gps_fix_pct >= 95%`).
- `analysis_summary.json -> pdr_by_link`: both peer links to V001 are healthy (`pdr >= 0.80`). Links above 0.70 are acceptable if mesh relay is confirmed working.
- `V001_rx_*.csv` includes `hop_count` column and has some rows with `hop_count >= 1`.
- `convoy_events.json -> detected.hard_event.min_accel_x_ms2 < -2.0` (target `< -3.0`).

### Accelerometer Axis Mapping (CRITICAL — Feb 28, 2026)

**The board's forward/braking axis is Y, not X.** The analyzer script defaults to checking `accel_x` for braking detection. Without correction, hard braking reads as ~-1.1 m/s² (lateral noise on wrong axis) instead of the real -8.63 m/s² on the correct axis.

**Always use the `--forward-axis y` flag:**
```bash
python scripts/analyze_convoy_recording.py \
  --input-root <RECORDING_DIR> \
  --out-dir <OUT_DIR> \
  --no-plots \
  --forward-axis y
```

The flag swaps `accel_x` and `accel_y` arrays at load time so all downstream code (braking detection, segment classification, observation building, sensor noise estimation) uses the correct forward axis.

**How this was discovered:** Recording #2 (Feb 28) showed min_accel_x of only -1.13 m/s² despite the driver stomping the brakes. Manual inspection of accel_y revealed peaks of -8.63 m/s² correlated with GPS speed drops from 43 km/h. The board is physically mounted with Y-axis pointing forward along the vehicle.

---

## Document History

- **Jan 24, 2026:** Created based on planning session analysis
- **Feb 27, 2026:** Added mesh-aware recording section (Phase E firmware complete)
- **Feb 27, 2026:** Added on-site post-drive validation flow for analyzer auto-mode (`ego_only` + `full`)
- **Feb 28, 2026:** Added accelerometer axis mapping section. Board Y-axis is forward; `--forward-axis y` flag added to analyzer. Documented Recording #2 GO verdict and discovery process.
