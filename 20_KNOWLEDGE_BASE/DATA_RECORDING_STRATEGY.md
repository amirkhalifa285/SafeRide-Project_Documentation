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

## Document History

- **Jan 24, 2026:** Created based on planning session analysis
