# Hardware Firmware Implementation Progress

**Project:** RoadSense V2V Enhanced System
**Status:** Active (Phase 6.2: Recording #2 Prep — Roof Mount + Hard Braking)
**Last Updated:** February 24, 2026

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

## Active Roadmap (Phase 6.2)

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

### 4. Convoy Recording #2 (NEXT — Planned)
- [ ] Roof-mount all 3 boards (strong tape, low speed) for optimal line-of-sight
- [ ] Verify all 6 links >80% PDR before driving
- [ ] Hard brake event: lead car stomps brake on signal (target: < -3.0 m/s²)
- [ ] Maintain 10-15m convoy spacing
- [ ] 60s stationary tail at end

---

## Completed Milestones (Archive)

| Date | Milestone | Description |
|------|-----------|-------------|
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

---

## Next Steps

1. Execute Recording #2 with roof-mounted boards + hard braking.
2. Validate all 6 links >80% PDR and hard brake < -3.0 m/s².
3. Process convoy logs into `ml/scenarios/base_real/` for final dataset generation.
4. Train final production model with convoy-calibrated emulator params.
5. Run 200-episode evaluation (n=1-5, hazard injection ON).
6. Begin quantization phase (TFLite INT8 for ESP32).
