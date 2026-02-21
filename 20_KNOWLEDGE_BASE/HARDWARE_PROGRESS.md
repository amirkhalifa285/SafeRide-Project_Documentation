# Hardware Firmware Implementation Progress

**Project:** RoadSense V2V Enhanced System
**Status:** Active (Phase 6.2: 3-Car Convoy Base Recording Prep)
**Last Updated:** February 20, 2026

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

### 3. Convoy Recording Gate
- [ ] Acquire 3rd microSD card before 3-car convoy recording (Phase 6.2 blocker)

---

## Completed Milestones (Archive)

| Date | Milestone | Description |
|------|-----------|-------------|
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
3. Only 2 SD cards are currently available; 3rd card required before 3-car convoy session.
4. V001 remains the intelligent/logging ego unit; peers are dumb broadcasters.

---

## Next Steps

1. Acquire 3rd SD card and verify all three vehicles can log TX/RX artifacts.
2. Execute 3-car convoy base recording (5-10 min) with at least one hard-braking event.
3. Process convoy logs into `ml/scenarios/base_real/` for dataset_v2 generation.
4. Start Phase 6.3 augmentation pipeline using `ml/espnow_emulator/emulator_params_measured.json`.
