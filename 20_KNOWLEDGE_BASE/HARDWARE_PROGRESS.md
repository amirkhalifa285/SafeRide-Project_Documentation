# Hardware Firmware Implementation Progress

**Project:** RoadSense V2V Enhanced System
**Status:** Active (Phase 6.3+: Real Data Pipeline — Augmentation & Training)
**Last Updated:** February 28, 2026

---

## Hardware Status Dashboard

| Component | Status | Config | Notes |
|-----------|--------|--------|-------|
| MCU | Ready | ESP32 DevKit V1 | V001, V002, V003 active |
| IMU | Validated | MPU6500 (0x68) | Accel/Gyro driver stable |
| Magnetometer | Integrated | QMC5883L (0x0D) | Production + RTT firmware now populate mag fields |
| GPS | Validated | NEO-6M (UART2) | Must be powered from 5V |
| Storage | Updated | MicroSD (SPI) | Mode 1 TX/RX CSV expanded to 16 columns |
| Network | Refined | ESP-NOW | RTT payload/logging updated with mag + GPS quality |

---

## Active Roadmap (Phase 6.2 — COMPLETED)

### 1. Field Readiness Implementation (Completed in Code)
- [x] Added reusable QMC5883L production driver (`hardware/src/sensors/mag/QMC5883LDriver.*`)
- [x] Enforced shared-I2C policy (`begin(TwoWire&)`, no internal `Wire.begin()`)
- [x] Integrated magnetometer into unified firmware message build path (`hardware/src/main.cpp`)
- [x] Integrated magnetometer into RTT sender (`hardware/test/espnow_rtt/sender_main.cpp`)
- [x] Extended RTT packet/logging schema with mag fields while preserving 90-byte packet
- [x] Extended Mode 1 TX/RX DataLogger schema from 10 to 16 columns (adds gyro + mag)
- [x] Updated RTT analysis script to compute magnetometer noise + mag-derived heading std
- [x] Updated related tests; user-reported test result: 92 tests passing

### 2. Hardware Validation (RTT Complete)
- [x] Run two-board dry run with SD cards (button start/stop workflow validated)
- [x] Verify TX/RX files per board with 16-column headers and non-zero `mag_x/y/z`
- [x] Regenerate `ml/espnow_emulator/emulator_params_measured.json` from fresh RTT capture
  - Completed from `/home/amirkhalifa/RoadSense2/rtt_log.csv` (Deir Hanna → Tiberias drive).
  - Output: `roadsense-v2v/ml/espnow_emulator/emulator_params_measured.json`.
  - Post-processing note: network parameters were extracted from drive-core intervals; sensor-noise parameters were extracted from a clean stationary interval.

### 3. Convoy Recording #1 (Feb 21, 2026) — COMPLETED
- [x] Acquired 3rd microSD card
- [x] Executed 3-car convoy recording (~200s, 6 CSV files)
- [x] GPS fix 100% on all 3 vehicles
- [x] Detected 2 light braking + 1 hard braking event + 42s stationary tail
- **Issue:** V001↔V002 link had ~85% packet loss (board placed near car door — metal shielding)
- **Issue:** Hard braking only reached -1.44 m/s² (too gentle)
- [x] Emulator calibrated from healthy links (V003↔V001, V003↔V002): `emulator_params_measured.json` updated
- [x] Sensor noise calibrated: cruising noise 5-8x higher than previous stationary-only RTT measurements

### 4. Convoy Recording #2 (Feb 28, 2026) — COMPLETED (GO)
- [x] Mesh-aware ego-only recording (V001 logs; V002/V003 run relay firmware)
- [x] Roof-mounted boards for optimal line-of-sight
- [x] Hard brake confirmed: **-8.63 m/s² raw / -4.11 m/s² smoothed** (0.88g, far exceeds -3.0 target)
- [x] Mesh relay confirmed: 953/3768 RX packets with hop>=1, max hop=2
- [x] GPS fix: 100% throughout
- [x] Links: V002→V001 PDR 0.852 (PASS), V003→V001 PDR 0.752 (above 0.70 floor, relay compensates)
- [x] Stationary tail: 301.9s (excellent)
- [x] Data: `V001_tx_004.csv` + `V001_rx_004.csv` (195.5s)
- **Critical discovery:** Board Y-axis is forward (not X). Analyzer requires `--forward-axis y` flag.

### 5. Extra Regular Driving Recording (Feb 28, 2026) — COMPLETED
- [x] 10-minute regular driving (no protocol stops/hazards)
- [x] Data: `V001_tx_005.csv` + `V001_rx_005.csv` (581.6s, 14899+12092 rows)
- [x] V002→V001 PDR: 0.881, V003→V001 PDR: 0.774
- [x] Formation: moderate (21.2m avg spacing) — adds distance diversity vs main recording (14m)
- [x] Braking events present from natural driving (peak -4.58 m/s² smoothed)

### Combined Dataset (Recording #2 + Extra)
- **Total duration:** ~777s (~13 min)
- **Total rows:** 19,883 TX + 15,860 RX
- **Location:** `Convoy_recording_02282026/` and `Convoy_extra_data_02282026/`
- **Analysis:** `ml/data/convoy_analysis_site/` and `ml/data/convoy_analysis_extra/`

---

## Completed Milestones (Archive)

| Date | Milestone | Description |
|------|-----------|-------------|
| Feb 28, 2026 | Convoy Recording #2 + Extra — GO | Mesh-aware ego-only recording (V001). Hard brake -8.63 m/s² confirmed (Y-axis forward). Relay working (hop=2). Extra 10min regular driving captured. Combined ~13 min dataset. |
| Feb 24, 2026 | Convoy Data Analysis + Emulator Calibration | Analyzed 6 CSV files from Recording #1; calibrated emulator with convoy-measured network + sensor noise; identified V001↔V002 placement issue |
| Feb 21, 2026 | Convoy Recording #1 | 3-car convoy, 200s, 6 files — partial success (link issue + mild braking) |
| Feb 20, 2026 | RTT Drive Characterization | Captured fresh RTT field log and regenerated final measured emulator params for Run 002 |
| Feb 15, 2026 | 2-Board Mode 1 Validation | Button-controlled TX/RX logging verified on both boards; 16-column CSV + non-zero mag confirmed |
| Feb 12, 2026 | Field Readiness Code Update | QMC5883L integrated into production + RTT paths; Mode 1 CSV and RTT analysis upgraded |
| Jan 9, 2026 | RTT Field Drive | 20-min drive: 8710 rows, RTT 4-9ms, max 45 km/h |
| Jan 5, 2026 | RTT Home Test | ESP-NOW RTT test passed, SD logging working |
| Dec 22, 2025 | QMC5883L | Magnetometer hardware validated on I2C 0x0D |

---

## Critical Constraints

1. Sim-to-real workflow: collect characterization + validation data, not real-world training labels.
2. GPS modules require 5V power (not 3.3V).
3. **Board placement matters:** metal car body blocks ESP-NOW signal. Roof mount gives best line-of-sight.
4. V002 was used as ego in Recording #1. Final demo will use V001 as ego (per architecture doc).

## Lessons Learned (Recording #1)

1. **ESP-NOW signal is blocked by car body metal.** Board placed near door handle had ~85% loss. Boards near glass or on roof have >90% PDR.
2. **"Hard braking" needs to be genuinely hard.** -1.44 m/s² is gentle. Need -3 to -5 m/s² for meaningful hazard detection.
3. **Stationary tail is useful for sensor noise calibration.** Park for 60s+ at end of every recording.
4. **Sensor noise during driving is 5-8x higher than stationary.** Previous emulator params (from stationary RTT) severely underestimated real-world noise.

## Lessons Learned (Recording #2)

5. **Accelerometer axis mapping is board-dependent.** Our board has Y-axis forward, not X. The analyzer script defaults to accel_x for braking detection. Always use `--forward-axis y`. Without this, hard braking reads as -1.13 m/s² (wrong axis) instead of -8.63 m/s² (correct axis).
6. **Mesh relay works in the real world.** 953 out of 3768 RX packets arrived via relay (hop>=1, max hop=2). V003 data reaches V001 through V002 even when direct V003→V001 link has reduced PDR.
7. **V003→V001 direct PDR at 0.752 is acceptable with relay.** The 0.80 threshold is strict for direct links, but mesh relay supplements the path. Combined effective data availability is higher.
8. **Extra regular driving data is valuable.** The 10-minute no-protocol drive adds variety: wider spacing (21m vs 14m), natural braking patterns, and more total data (3x the main recording).

---

## Next Steps

1. ~~Execute Recording #2 with roof-mounted boards + hard braking.~~ DONE (Feb 28)
2. ~~Validate all links and hard brake quality.~~ DONE — GO verdict
3. Re-calibrate emulator params from Recording #2 data (both recordings).
4. Process convoy logs into `ml/scenarios/base_real/` for final dataset generation.
5. Train final production model with convoy-calibrated emulator params.
6. Run 200-episode evaluation (n=1-5, hazard injection ON).
7. Begin quantization phase (TFLite INT8 for ESP32).
