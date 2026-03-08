# Agent Handoff: Field Readiness Plan (Magnetometer + Mode 1 Logging + RTT Refinement)

**Created:** February 12, 2026  
**Status:** ACTIVE (Implementation In Progress)  
**Purpose:** Give a fresh agent complete context and an execution plan to make firmware/data logging field-ready before 3-car convoy recording.

---

## 1) Goal

Prepare the system so convoy recording can be done smoothly in one outdoor session.

Primary objectives:
1. Integrate magnetometer into production firmware path (not only standalone test).
2. Validate Mode 1 TX/RX logging reliability using 2 boards + 2 SD cards.
3. Refine RTT measurement/logging outputs (already working; improve fields/quality).
4. Produce clear go/no-go checklist for outdoor convoy session.

---

## 2) Current Truth (Do Not Re-discover)

1. RTT flow already works and was used before; this is **refinement**, not first implementation.
2. Main firmware now populates `msg.sensors.mag` from QMC5883L in `hardware/src/main.cpp`, with graceful zero fallback when mag init/read fails.
3. Reusable production QMC5883L driver now exists in:
   - `roadsense-v2v/hardware/src/sensors/mag/QMC5883LDriver.h`
   - `roadsense-v2v/hardware/src/sensors/mag/QMC5883LDriver.cpp`
   - compatibility shim: `roadsense-v2v/hardware/src/sensors/QMC5883LDriver.h`
4. Mode 1 network characterization logging (TX/RX CSV) is implemented in `hardware/src/logging/DataLogger.cpp` and controlled by button in `hardware/src/main.cpp`.
5. PlatformIO commands must be run from `roadsense-v2v/hardware/`.
6. `V2VMessage.h` defines `sensors.mag[3]` and the firmware now populates it in the production path.
7. Mode 1 TX/RX CSV format is now expanded to 16 columns with gyro+mag in both headers and rows.
8. RTT sender now logs IMU accel + magnetometer, and packet schema includes `mag_*` plus GPS quality fields (`gps_hdop`, `gps_satellites`) while staying 90 bytes.
9. `ml/scripts/analyze_rtt_characterization.py` now parses mag columns and computes `mag_std_*`, `mag_std_ut`, and mag-derived `heading_std_deg`.
10. QMC5883L (0x0D) and MPU6500 (0x68) share the I2C bus (SDA=21, SCL=22). MPU6500Driver already calls `Wire.begin()` internally -- the QMC driver **must reuse the existing Wire instance**, not call `Wire.begin()` again.
11. `config.h` magnetometer comments were updated and `QMC5883L_I2C_ADDR` is defined.
12. `hardware/examples/network_characterization_example.cpp` include path is now supported by compatibility shim `src/sensors/QMC5883LDriver.h`.
13. `config.h` ML defines (`ML_FEATURE_COUNT=30`, `ML_MAX_VEHICLES=3`) are stale vs Deep Sets architecture. **Do NOT change these in this plan** -- they are out of scope and changing them could break things.

---

## 2.1) Execution Status Snapshot (Feb 15, 2026)

- [x] **Phase A complete in code**: QMC driver + production firmware + RTT sender + RTT packet/logging + platformio wiring.
- [x] **Phase B1 complete in code**: Mode 1 TX/RX CSV extended to 16 columns (gyro+mag).
- [x] **Phase C1 complete**: Updated dependent tests (`test_rtt_packet`, `test_rtt_csv_format`, `test_datalogger`).
- [x] **Phase C3 complete**: Added missing tests/coverage:
  - new `test_qmc5883l_driver`
  - sensor integration assertion for non-zero outgoing mag
  - Mode 1 CSV column-count checks
- [x] **Phase D3 complete in code**: RTT analysis script extended for magnetometer noise + heading std.
- [x] **Phase B2 complete (hardware session)**: 2-board TX/RX button workflow validated with SD cards (artifacts: `RXTX_test_20261402/`).
- [x] **Phase C2 complete (user verification)**: user reports full test suite pass (**92 tests passing**).
- [ ] **Phase D4-D5 pending (hardware capture + processing)**: regenerate `emulator_params_measured.json` from fresh 2-board RTT capture.
- [ ] **Phase E pending documentation package**: operator checklist + artifact checklist + on-site integrity checks.

---

## 3) Read First (Context Files)

1. `docs/PROJECT_STATUS_OVERVIEW.md`
2. `docs/_MAPS/INDEX.md`
3. `docs/10_PLANS_ACTIVE/PHASE_6_REAL_DATA_PIPELINE.md`
4. `docs/00_ARCHITECTURE/DEEP_SETS_N_ELEMENT_ARCHITECTURE.md`
5. `docs/10_PLANS_ACTIVE/PROFESSOR_COMMENTS_RESPONSE_PLAN.md`
6. `docs/20_KNOWLEDGE_BASE/HARDWARE_PROGRESS.md`

---

## 4) Key Code Files To Review

**Review before modifying:**
1. `roadsense-v2v/hardware/src/main.cpp`
2. `roadsense-v2v/hardware/src/logging/DataLogger.h`
3. `roadsense-v2v/hardware/src/logging/DataLogger.cpp`
4. `roadsense-v2v/hardware/src/config.h`
5. `roadsense-v2v/hardware/src/sensors/gps/NEO6M_Driver.h`
6. `roadsense-v2v/hardware/src/sensors/gps/NEO6M_Driver.cpp`
7. `roadsense-v2v/hardware/src/sensors/imu/MPU6500Driver.h`
8. `roadsense-v2v/hardware/src/sensors/imu/MPU6500Driver.cpp`
9. `roadsense-v2v/hardware/test/test_qmc5883l/test_main.cpp`
10. `roadsense-v2v/hardware/platformio.ini`
11. `roadsense-v2v/hardware/test/espnow_rtt/sender_main.cpp`
12. `roadsense-v2v/hardware/test/espnow_rtt/RTTPacket.h`
13. `roadsense-v2v/hardware/test/espnow_rtt/RTTLogging.h`
14. `roadsense-v2v/hardware/src/network/protocol/V2VMessage.h` *(verify `sensors.mag[3]` field exists)*
15. `roadsense-v2v/ml/scripts/analyze_rtt_characterization.py` *(understand current analysis scope)*

---

## 5) Constraints

1. Only 2 microSD cards currently available. **A 3rd SD card must be acquired before the 3-car convoy recording (Phase 6.2).**
2. Initial validation must be done with 2 boards before 3-car convoy recording.
3. Keep existing working RTT behavior; avoid risky rewrites.
4. Do not break Mode 1 button workflow (start/stop logging on device).
5. QMC5883L driver **must reuse** the existing `Wire` instance initialized by MPU6500Driver. Do NOT call `Wire.begin()` inside the QMC driver -- accept a `TwoWire&` reference or assume Wire is already started.
6. Do NOT touch `config.h` ML defines (`ML_FEATURE_COUNT`, `ML_MAX_VEHICLES`) -- they are stale but out of scope.

---

## 6) Execution Plan

## Phase A: Magnetometer Integration (Production Firmware + RTT Firmware)

### A1. Create reusable QMC5883L driver

Create `roadsense-v2v/hardware/src/sensors/mag/`:
   - `QMC5883LDriver.h`
   - `QMC5883LDriver.cpp`

Implement:
   - `begin(TwoWire& wire)` -- accept existing Wire reference, do NOT call `Wire.begin()` (MPU6500 already initializes I2C)
   - `readRaw()` / `read()` returning X/Y/Z as floats
   - basic health checks (device present at 0x0D, chip ID 0xFF, config write/read)
   - Reference implementation: `hardware/test/test_qmc5883l/test_main.cpp` has working register-level code

### A2. Update `config.h`

   - Add `#define QMC5883L_I2C_ADDR 0x0D`
   - Update/remove misleading comments at lines 75-79 ("Magnetometer calibration NOT APPLICABLE") to reflect QMC5883L integration

### A3. Integrate into production firmware (`hardware/src/main.cpp`)

   - `#include "sensors/mag/QMC5883LDriver.h"`
   - Create global `QMC5883LDriver mag;`
   - Initialize in `setup()` after IMU init: `mag.begin(Wire)` -- reuse Wire instance
   - Keep system robust if mag init fails (log warning, continue with zeros)
   - In `buildV2VMessage()`: replace `memset(msg.sensors.mag, 0, ...)` with actual readings

### A4. Integrate into RTT sender firmware (`hardware/test/espnow_rtt/sender_main.cpp`)

   **CRITICAL: Phase 6.1 of PHASE_6_REAL_DATA_PIPELINE.md requires mag data in RTT captures for sensor noise characterization.**
   - Add `#include "sensors/mag/QMC5883LDriver.h"` to sender_main.cpp
   - Initialize QMC5883L in sender setup (after IMU, reusing Wire)
   - Read mag X/Y/Z each cycle alongside IMU data

### A5. Update `RTTPacket.h`

   - Add `float mag_x, mag_y, mag_z;` fields (12 bytes, taken from the 54-byte padding)
   - Add `float gps_hdop; uint8_t gps_satellites;` if GPS driver exposes them
   - Update compile-time size assertion accordingly

### A6. Update `RTTLogging.h`

   - Add `mag_x,mag_y,mag_z` columns to CSV header
   - Update `formatRow()` to include mag data in output

### A7. Update `platformio.ini`

   - Add `+<sensors/mag/QMC5883LDriver.cpp>` to `rtt_sender` `build_src_filter` (line ~80)
   - The `esp32dev` env uses `+<**/*.cpp>` so it picks up the driver automatically

### A8. Keep heading policy explicit

   - Primary heading from GPS where valid (current behavior)
   - Log raw mag X/Y/Z for offline analysis -- do NOT fuse mag heading into GPS heading in this plan
   - Optional mag heading fallback only if intentionally added and documented in a future plan

## Phase B: Mode 1 TX/RX CSV Format Update + Logging Validation (2 Boards)

### B1. Extend Mode 1 DataLogger CSV format

**CRITICAL: Current Mode 1 CSV only logs 10 columns (no mag, no gyro, no HDOP/satellites). The convoy recording will be useless for sensor noise analysis without these fields.**

Modify `DataLogger.cpp`:
   - `logTxMessage()` (~line 401): Add gyro_x, gyro_y, gyro_z, mag_x, mag_y, mag_z columns
   - `logRxMessage()` (~line 447): Add the same columns
   - `writeTxHeader()` (~line 555): Update CSV header string
   - `writeRxHeader()` (~line 570): Update CSV header string

New TX CSV format:
```
timestamp_local_ms,msg_timestamp,vehicle_id,lat,lon,speed,heading,accel_x,accel_y,accel_z,gyro_x,gyro_y,gyro_z,mag_x,mag_y,mag_z
```

New RX CSV format:
```
timestamp_local_ms,msg_timestamp,from_vehicle_id,lat,lon,speed,heading,accel_x,accel_y,accel_z,gyro_x,gyro_y,gyro_z,mag_x,mag_y,mag_z
```

### B2. Flash and validate (2 boards)

1. Flash both boards with unified firmware (different `VEHICLE_ID` values).
2. Insert SD cards in both boards.
3. Test button workflow:
   - press once: start logging
   - press again: stop logging
4. Run two dry runs:
   - static run: ~3 minutes
   - motion run: ~5 minutes with varying distance
5. Validate generated files per board:
   - both TX and RX files exist
   - headers are correct and include new mag/gyro columns
   - rows are parseable (no truncated/corrupt lines)
   - timestamps monotonic within each file
   - counts are plausible (RX <= peer TX; loss explainable)
   - **mag_x/y/z columns contain non-zero values** (confirms Phase A integration works end-to-end)

## Phase C: Test Suite Pass (Firmware + Sensor/Logging)

**IMPORTANT: Run Phase C tests AFTER Phase A and Phase D code changes are complete.** Phases A and D modify structs and CSV formats -- running tests before those changes will give false failures.

### C1. Update tests that depend on changed structs/formats

These tests will break after Phase A/D changes and must be updated first:
   - `test_rtt_packet` -- RTTPacket struct gains mag fields, size assertion changes
   - `test_rtt_csv_format` -- CSV header and row format now include mag columns
   - `test_datalogger` -- Mode 1 CSV format now includes gyro + mag columns

### C2. Run and verify all relevant tests from `roadsense-v2v/hardware/`

1. `test_qmc5883l` (hardware test -- requires physical sensor)
2. `test_datalogger` (updated for new CSV format)
3. `test_sensor_integration`
4. `test_mpu6500_driver`
5. RTT-related tests:
   - `test_rtt_packet` (updated for new RTTPacket fields)
   - `test_rtt_csv_format` (updated for new CSV columns)
   - `test_rtt_timeout`
   - `test_rtt_circular_buffer`

### C3. Add missing test coverage

1. Add tests for QMC5883L driver: init success, init failure (sensor absent), read valid data, read when sensor failed.
2. Add test/assertion that outgoing V2V message contains non-zero mag values when sensor active.
3. Add test that Mode 1 TX/RX CSV rows contain correct column count (16 columns, not old 10).

## Phase D: RTT Refinement (Not Rebuild)

### D1. RTT struct/logging changes (done in Phase A4-A6)

Phase A already covers the RTTPacket.h and RTTLogging.h modifications. Verify:
   - `RTTPacket.h` has mag_x/y/z fields
   - `RTTLogging.h` CSV header includes mag columns
   - `sender_main.cpp` populates mag data each cycle
   - Reflector firmware does NOT need mag (it only echoes packets)

### D2. Keep existing RTT send/reflect workflow intact

   - Do not change the ping-pong timing, sequence numbering, or loss detection logic
   - Only the payload content and CSV output are extended

### D3. Update `ml/scripts/analyze_rtt_characterization.py`

**CRITICAL: The existing script cannot produce magnetometer noise parameters. Without this update, `emulator_params_measured.json` will be incomplete.**

Add to the script:
   - Parse new `mag_x,mag_y,mag_z` columns from RTT CSV
   - During stationary segments, compute `mag_std_x`, `mag_std_y`, `mag_std_z`
   - Compute heading from mag (atan2(y,x)) and derive `heading_std_deg` from mag variance
   - Add `sensor_noise.mag_std_ut` and `sensor_noise.heading_std_deg` to output JSON

### D4. Run short refinement capture with 2 boards

### D5. Process with updated script to regenerate measured emulator params

   - Output target: `roadsense-v2v/ml/espnow_emulator/emulator_params_measured.json`
   - Verify output JSON contains: latency, packet_loss, AND sensor_noise (including mag fields)

## Phase E: Field-Day Readiness Package

Produce:
1. operator checklist (power-on, GPS lock criteria, button sequence, stop sequence),
2. expected artifact list (which files from which board),
3. quick integrity checks to run immediately after recording before leaving site,
4. **hardware procurement note: a 3rd microSD card is required before the 3-car convoy recording (Phase 6.2).**

---

## 7) Suggested Command Baseline

From project root:

```bash
cd /home/amirkhalifa/RoadSense2/roadsense-v2v/hardware
```

Build examples:

```bash
pio run -e esp32dev
pio run -e rtt_sender
pio run -e rtt_reflector
```

Test examples:

```bash
pio test -e esp32dev -f test_datalogger
pio test -e esp32dev -f test_qmc5883l
pio test -e esp32dev -f test_sensor_integration
```

Upload/monitor examples (adjust serial port):

```bash
pio run -e esp32dev --target upload --upload-port /dev/ttyUSB0
pio device monitor --baud 115200 --port /dev/ttyUSB0
```

---

## 8) Definition of Done (Go/No-Go)

Ready for outdoor convoy only if **all** are true:
1. QMC5883L driver created and integrated in both production firmware (`main.cpp`) and RTT sender firmware (`sender_main.cpp`).
2. Magnetometer logs non-zero X/Y/Z in V2V broadcasts and in RTT CSV captures.
3. Mode 1 TX/RX CSV format includes gyro + mag columns (16 columns, not old 10).
4. Mode 1 TX/RX logging works reliably on both boards through button start/stop.
5. All tests pass after code changes (including updated RTT and DataLogger tests).
6. RTT refinement output (`emulator_params_measured.json`) is regenerated successfully and includes `sensor_noise.heading_std_deg` from magnetometer data.
7. `analyze_rtt_characterization.py` updated to parse and analyze mag columns.
8. Field checklist and expected artifacts are documented and verified in a dry run.
9. **Blocker for convoy (not for this plan):** 3rd microSD card acquired.

Status snapshot (Feb 15, 2026):
- [x] Item 1 complete
- [ ] Item 2 pending RTT-side confirmation (Mode 1 TX/RX logs now confirmed with non-zero mag; RTT capture still pending)
- [x] Item 3 complete
- [x] Item 4 complete (button start/stop + TX/RX files validated on both boards)
- [x] Item 5 complete (user reported 92 tests passing)
- [ ] Item 6 pending refreshed capture + processing
- [x] Item 7 complete
- [ ] Item 8 pending
- [ ] Item 9 pending procurement

---

## 9) Parallel Work Split (Two Agents)

1. Agent A (Firmware/Sensors):
   - QMC5883L driver creation (A1-A2)
   - Production firmware integration (A3)
   - RTT sender firmware integration (A4-A7)
   - Sensor tests (C3 items 1-2)
2. Agent B (Logging/RTT/Validation):
   - Mode 1 DataLogger CSV format update (B1)
   - Mode 1 dry-run validation (B2)
   - RTT processing script update (D3)
   - DataLogger test updates (C1, C3 item 3)
   - Field readiness package (E)

**Dependency:** Agent B's CSV validation (B2) requires Agent A's mag driver (A1-A3) to be complete first -- mag columns will be zeros until the driver is wired in.

Shared checkpoint:
1. Joint go/no-go decision using Section 8 before outdoor 3-car recording.

---

## 10) Files Modified By This Plan (Summary)

| File | Change |
|------|--------|
| `hardware/src/sensors/mag/QMC5883LDriver.h` | **NEW** -- driver header |
| `hardware/src/sensors/mag/QMC5883LDriver.cpp` | **NEW** -- driver implementation |
| `hardware/src/sensors/QMC5883LDriver.h` | **NEW** -- compatibility include shim for existing example include path |
| `hardware/src/config.h` | Add QMC5883L defines, fix comments |
| `hardware/src/main.cpp` | Add mag driver init + populate sensors.mag |
| `hardware/test/espnow_rtt/sender_main.cpp` | Add mag init + read |
| `hardware/test/espnow_rtt/RTTCommon.h` | Add mag fields to RTTRecord used by CSV writer |
| `hardware/test/espnow_rtt/RTTPacket.h` | Add mag fields, adjust padding |
| `hardware/test/espnow_rtt/RTTLogging.h` | Add mag columns to CSV |
| `hardware/src/logging/DataLogger.h` | Add Mode 1 row formatter helpers + updated schema docs |
| `hardware/src/logging/DataLogger.cpp` | Extend Mode 1 TX/RX CSV with gyro+mag |
| `hardware/platformio.ini` | Add QMC driver to rtt_sender build_src_filter |
| `ml/scripts/analyze_rtt_characterization.py` | Add mag noise analysis |
| `hardware/test/test_rtt_packet/*` | Update for new RTTPacket size |
| `hardware/test/test_rtt_csv_format/*` | Update for new CSV columns |
| `hardware/test/test_datalogger/*` | Update for new Mode 1 CSV columns |
| `hardware/test/test_qmc5883l_driver/*` | **NEW** -- QMC driver init/read/failure coverage |
| `hardware/test/test_sensor_integration/*` | Add non-zero outgoing magnetometer assertion |

---

## 11) Notes For Fresh Chat Prompt

When starting a new chat, include:
1. this file path: `docs/10_PLANS_ACTIVE/AGENT_HANDOFF_FIELD_READINESS_PLAN.md`
2. constraint: only 2 SD cards available
3. instruction: do not restart from architecture redesign; execute this plan directly
4. requirement: prioritize reliability and repeatable field workflow over new features
5. constraint: QMC5883L driver must NOT call Wire.begin() -- reuse existing Wire instance from MPU6500
6. constraint: do NOT change config.h ML defines (ML_FEATURE_COUNT, ML_MAX_VEHICLES) -- out of scope.
