# RTT Firmware TDD Progress Tracker

**Session Date:** December 29 - January 10, 2026
**Approach:** Pure TDD (RED → GREEN → REFACTOR)
**Status:** ✅ CHARACTERIZATION COMPLETE | Valid 5m Data Obtained

---

## CURRENT PROJECT STATUS (READ THIS FIRST)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        PROJECT PHASE: PIPELINE VALIDATION                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  WHAT WE HAVE:                                                               │
│  ✅ Valid emulator params from 5m stationary test (emulator_params_5m.json) │
│  ✅ RTT firmware working (42/42 tests passing)                              │
│  ✅ Integration firmware builds (sender + reflector)                         │
│  ✅ ESP-NOW Emulator implemented (84/84 tests passing)                      │
│                                                                              │
│  WHAT WE ARE DOING NOW:                                                      │
│  → Using 5m emulator params to build ConvoyEnv gym environment              │
│  → Training basic PPO/DQN model to validate pipeline                        │
│  → Goal: Prove end-to-end pipeline works before collecting final data       │
│                                                                              │
│  WHAT COMES NEXT (AFTER PIPELINE VALIDATED):                                │
│  1. Apply ESP-NOW LR mode patches (see ESPNOW_LONG_RANGE_MODE_MIGRATION.md) │
│  2. Stationary test to validate LR mode improves range                      │
│  3. Clean characterization drive with LR mode enabled                       │
│  4. Generate final emulator_params_LR_final.json                            │
│  5. Retrain model with production-quality data                              │
│                                                                              │
│  KEY FILES:                                                                  │
│  • Emulator params (current): ml/espnow_emulator/emulator_params_5m.json    │
│  • LR mode plan: docs/10_PLANS_ACTIVE/ESPNOW_LONG_RANGE_MODE_MIGRATION.md   │
│  • ConvoyEnv plan: docs/10_PLANS_ACTIVE/ConvoyEnv_Implementation/           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## TDD Cycle Progress

### 🔴 RED Phase: ✅ COMPLETE

**Objective:** Write tests that FAIL

**Accomplishments:**
- ✅ Created 4 test suites (42 tests total)
- ✅ Created empty header files
- ✅ Removed stub implementations from tests
- ✅ Verified compilation failures (RED phase confirmed)

**Evidence of RED:**
```
test/test_rtt_packet/test_main.cpp:182:5: error: 'RTTPacket' was not declared in this scope
```

---

### 🟢 GREEN Phase: ✅ COMPLETE

**Objective:** Write minimal code to make tests PASS

**Status:** All 42 tests passing after Phase 2 test defect fixes

#### Component 1: RTTPacket.h ✅ IMPLEMENTED

**File:** `test/espnow_rtt/RTTPacket.h`

**Implementation:**
```cpp
#pragma pack(push, 1)
struct RTTPacket {
    uint32_t sequence;          // 4 bytes
    uint32_t send_time_ms;      // 4 bytes
    float sender_lat;           // 4 bytes
    float sender_lon;           // 4 bytes
    float sender_speed;         // 4 bytes
    float sender_heading;       // 4 bytes
    float accel_x;              // 4 bytes
    float accel_y;              // 4 bytes
    float accel_z;              // 4 bytes
    uint8_t padding[54];        // 54 bytes
};  // Total: 90 bytes
#pragma pack(pop)
```

**Tests Affected:** test_rtt_packet (8 tests)
**Status:** ✅ Compiles successfully

---

#### Component 2: RTTLogging.h ✅ IMPLEMENTED

**File:** `test/espnow_rtt/RTTLogging.h`

**Implemented Functions:**
```cpp
void generateCSVHeader(char* buffer, size_t bufferSize);
void formatCSVRow(char* buffer, size_t bufferSize, const RTTRecord* record);
int countChar(const char* str, char c);
```

**Tests Affected:** test_rtt_csv_format (10 tests)
**Status:** ✅ All tests passing (after Phase 2 GPS precision fix)

**Phase 2 Fix:**
- Changed GPS test values to clean binary floats (32.125, 34.75)
- Avoids IEEE 754 precision issues in string comparison

---

#### Component 3: RTTBuffer.h ✅ IMPLEMENTED

**File:** `test/espnow_rtt/RTTBuffer.h`

**Implemented Functions:**
```cpp
void initCircularBuffer();
void addPendingPacket(const RTTRecord* record);
void markPacketReceived(uint32_t sequence, uint32_t recv_time_ms);
bool isPacketPending(uint32_t sequence);
bool isPacketReceived(uint32_t sequence);
int getBufferIndex(uint32_t sequence);
```

**Tests Affected:** test_rtt_circular_buffer (12 tests)
**Status:** ✅ All tests passing (after Phase 2 modulo math fix)

**Phase 2 Fix:**
- Corrected wraparound logic for stress test (250 packets in 100 slots)
- Slots 0-49: sequences 200-249 (overwritten twice)
- Slots 50-99: sequences 150-199 (overwritten once)

---

#### Component 4: RTTTimeout.h ✅ IMPLEMENTED

**File:** `test/espnow_rtt/RTTTimeout.h`

**Implemented Functions:**
```cpp
bool shouldWriteRecord(const RTTRecord* record, uint32_t current_time_ms);
bool isTimedOut(uint32_t send_time_ms, uint32_t current_time_ms);
bool isRecentlySent(uint32_t send_time_ms, uint32_t current_time_ms);
uint32_t getElapsedTime(uint32_t start_time_ms, uint32_t current_time_ms);
```

**Critical Feature:** Handles millis() wraparound correctly

**Tests Affected:** test_rtt_timeout (12 tests)
**Status:** ✅ All tests passing

---

### 🔵 REFACTOR Phase: ⏳ NOT STARTED

**Objective:** Improve code quality while maintaining passing tests

**Planned Activities:**
- Code review for clarity
- Remove code duplication
- Add inline documentation
- Performance optimization
- Memory optimization

---

## Implementation Priority

### Next Steps (In Order):

1. **Implement RTTLogging.h** (10 tests)
   - Foundation for CSV output
   - Depends on: RTTRecord struct

2. **Implement RTTTimeout.h** (12 tests)
   - Independent component
   - Critical: millis() wraparound handling

3. **Implement RTTBuffer.h** (12 tests)
   - Depends on: RTTRecord struct
   - Most complex logic

4. **Verify ALL tests PASS**
   - Run full test suite
   - Confirm GREEN phase complete

5. **Begin REFACTOR phase**
   - Review all code
   - Optimize and document

---

## Test Statistics

| Component | Tests | Status |
|-----------|-------|--------|
| RTTPacket.h | 8 | ✅ PASS |
| RTTLogging.h | 10 | ✅ PASS (Phase 2 GPS precision fixed) |
| RTTBuffer.h | 12 | ✅ PASS (Phase 2 modulo math fixed) |
| RTTTimeout.h | 12 | ✅ PASS |
| **TOTAL** | **42** | **✅ 100% Complete (GREEN)** |

---

## TDD Benefits Observed

### What TDD Gave Us:

1. **Clear Requirements**
   - 42 tests define EXACTLY what code must do
   - No ambiguity about expected behavior

2. **Immediate Feedback**
   - Compilation errors show missing components
   - Test failures show incorrect logic

3. **Regression Prevention**
   - Future changes verified against 42 tests
   - Can't break existing functionality unknowingly

4. **Documentation**
   - Tests serve as executable documentation
   - Show how components should be used

5. **Design Quality**
   - Tests forced clean interfaces
   - Minimal coupling between components

---

## Lessons Learned

### ✅ What Worked Well:

1. **Stub implementations in tests** (for initial validation)
   - Allowed tests to compile immediately
   - Verified test logic before real implementation

2. **Removing stubs for pure RED**
   - Forces disciplined TDD cycle
   - Clear RED → GREEN transition

3. **Component-by-component approach**
   - RTTPacket first (foundation)
   - Then logging, timeout, buffer

### 🚧 Challenges:

1. **PlatformIO test quirks**
   - `--without-testing` flag behavior
   - Hardware upload attempts
   - Solution: Focus on compilation success

2. **Header dependencies**
   - Multiple tests need RTTRecord struct
   - Solution: Define in each header, or shared header

---

## ✅ Phase 2 Completion Summary

### Test Defects Fixed:

1. **test_gps_precision** (test_rtt_csv_format)
   - **Issue:** IEEE 754 precision - `32.085123f` doesn't represent exactly
   - **Fix:** Changed to clean binary values: `32.125f`, `34.75f`
   - **Result:** ✅ Test now passes

2. **test_buffer_stress_rapid_adds** (test_rtt_circular_buffer)
   - **Issue:** Wrong expectation for modulo buffer wraparound
   - **Fix:** Corrected math - slots 0-49 contain seq 200-249, slots 50-99 contain seq 150-199
   - **Result:** ✅ Test now passes

### Final Test Results:
```
✅ test_rtt_packet:           8/8   PASSED
✅ test_rtt_csv_format:      10/10  PASSED
✅ test_rtt_circular_buffer: 12/12  PASSED
✅ test_rtt_timeout:         12/12  PASSED
────────────────────────────────────────────
   TOTAL:                    42/42  🟢 GREEN
```

---

## ✅ Phase 3 - Integration: COMPLETE (January 4, 2026)

### Integration Tasks:
- [x] Implement `sender_main.cpp` (complex - uses all verified components) ✅
- [x] Implement `reflector_main.cpp` (simple - receive and echo) ✅
- [x] Update `platformio.ini` with `env:rtt_sender` and `env:rtt_reflector` ✅
- [x] Build verification: `pio run -e rtt_sender && pio run -e rtt_reflector` ✅
- [ ] Hardware bench test (60s @ 1m) ⏳ *Pending field test*

### Build Results:
```
rtt_sender:    SUCCESS - RAM: 15.1% (49.5KB), Flash: 61.9% (812KB)
rtt_reflector: SUCCESS - RAM: 13.3% (43.5KB), Flash: 57.1% (749KB)
```

### Files Created:
| File | Purpose |
|------|---------|
| `test/espnow_rtt/sender_main.cpp` | 10Hz RTT sender with GPS+IMU logging |
| `test/espnow_rtt/reflector_main.cpp` | Minimal echo reflector |
| `platformio.ini` (updated) | Added `env:rtt_sender` and `env:rtt_reflector` |

---

## Session Update: January 4, 2026 (Evening)

### Architecture Review: ✅ VALIDATED

Reviewed `RTT_ARCH_REVIEW_CRITIC.md` criticisms:

| Criticism | Status | Resolution |
|-----------|--------|------------|
| A) Distance correlation not logged | ✅ Accepted | Option 1: Use domain randomization, no distance model needed |
| B) No sessionization | ⚠️ Deferred | Acceptable for characterization (not scenario logging) |
| C) Destructive logging (O_TRUNC) | ✅ **FIXED** | Changed to O_AT_END + smart header handling |
| D) Measurement contamination | ✅ Acceptable | Timestamps captured before sensor reads |

**Decision:** Proceeding with **Option 1** (random drive + domain randomization) - distance-bucketed model not required for RL observation space.

### Bug Fix Applied:

**File:** `test/espnow_rtt/sender_main.cpp` (line 217)

```cpp
// BEFORE (data loss on reboot):
logFile.open(RTT_LOG_FILENAME, O_RDWR | O_CREAT | O_TRUNC)

// AFTER (preserves existing data):
logFile.open(RTT_LOG_FILENAME, O_RDWR | O_CREAT | O_AT_END)
```

Also added smart header handling - only writes CSV header if file is empty (new file).

---

## ✅ Home Validation Test: PASSED (January 5, 2026)

### Test Setup:
- **Sender (V001):** ESP32 connected to **Linux laptop** (Fedora)
- **Reflector (V002):** ESP32 connected to **Windows laptop**
- **Distance:** ~1m indoor (home environment)

### Flash Commands Used:

**On Linux (V001 - Sender):**
```bash
pio run -e rtt_sender --target upload --upload-port /dev/ttyUSB0
pio device monitor -p /dev/ttyUSB0 -b 115200
```

**On Windows (V002 - Reflector):**
- Files copied to `C:\HARDWARE_TESTING\`
- Required files: `platformio.ini`, `src/utils/Logger.h`, `src/utils/Logger.cpp`, `test/espnow_rtt/reflector_main.cpp`, `test/espnow_rtt/RTTPacket.h`
```powershell
pio run -e rtt_reflector --target upload
pio device monitor -b 115200
```

### Validation Checklist:
- [x] Sender boots and shows "RTT Sender Starting..."
- [x] Reflector boots and shows "RTT Reflector Starting..."
- [x] Reflector serial shows "Echoed X packets" (increasing count) ✅
- [x] Sender serial shows "Sent seq=X" messages ✅
- [x] After 30s: Check SD card for `/rtt_log.csv` ✅
- [x] CSV contains rows with `lost=0` and valid `rtt_ms` values ✅
- [ ] **Append test:** Power cycle sender, verify file grows *(Pending - user will test later)*

### Actual Results (January 5, 2026):

| Metric | Expected | Actual | Status |
|--------|----------|--------|--------|
| Total packets | N/A | **1127 rows** | ✅ |
| RTT | 2-15ms | **4-9ms typical** | ✅ Excellent |
| Packet loss | < 2% | ~14% (sample) | ⚠️ Higher than expected |
| IMU accel_z | ~9.81 m/s² | **~9.78-9.80** | ✅ Correct |
| GPS | 0 (indoor) | **0** | ✅ Expected |

### Sample Data Captured:
```csv
sequence,send_time_ms,recv_time_ms,rtt_ms,lat,lon,speed,heading,accel_x,accel_y,accel_z,lost
0,831,840,9,0,0,0,0,1.22,0.51,9.789,0
1,931,936,5,0,0,0,0,1.223,0.511,9.793,0
2,1031,1036,5,0,0,0,0,1.219,0.517,9.784,0
3,1131,-1,-1,0,0,0,0,1.226,0.51,9.774,1
```

### Notes:
- **GPS=NO** displayed on serial (expected indoors)
- Lost packets correctly logged with `recv_time_ms=-1`, `rtt_ms=-1`, `lost=1`
- Packet loss higher than expected (~14%) - possibly due to WiFi interference indoors
- Sequence numbers out-of-order in CSV is correct (lost packets written after 500ms timeout)

---

## ✅ Field Characterization Drive: COMPLETED (January 9, 2026)

### Test Setup:
- **Sender (V001):** ESP32 in vehicle 1 (Linux laptop for flash)
- **Reflector (V002):** ESP32 in vehicle 2 (Windows laptop for flash)
- **Duration:** ~20 minutes
- **Route:** Open road + area near military base (RF interference observed)

### Flash Commands Used:
**Linux (V001 - Sender):**
```bash
pio run -e rtt_sender --target upload --upload-port /dev/ttyUSB0
```

**Windows (V002 - Reflector):**
```powershell
pio run -e rtt_reflector --target upload
```

### Results Summary:

| Metric | Value | Notes |
|--------|-------|-------|
| **Total rows** | 8710 | Excellent coverage |
| **Duration** | ~20 min | ~839 seconds |
| **RTT (when received)** | 4-9ms | Consistent with home test |
| **Max speed** | 12.49 m/s (45 km/h) | Good range captured |
| **Packet loss** | ~27% overall | Higher near military base |
| **GPS coverage** | Good | Fix acquired after ~20s outdoors |
| **IMU data** | Valid | accel_z ~9.8 when level, ~7.5 when tilted at end |

### Key Observations:
1. **Military Base Interference:** Heavy packet loss in middle section - consecutive lost=1 rows visible in CSV around rows 4000-4100
2. **Speed in m/s:** CSV `speed` column is meters/second (from `gps.speed.mps()`), multiply by 3.6 for km/h
3. **Training Implication:** High packet loss is actually valuable for training robustness (domain randomization principle)

### Data Location:
- **File:** `/home/amirkhalifa/RoadSense2/rtt_log.csv`
- **Size:** ~639 KB
- **Format:** CSV with 12 columns (sequence, send_time_ms, recv_time_ms, rtt_ms, lat, lon, speed, heading, accel_x, accel_y, accel_z, lost)

---

## ✅ Step 1: Analysis Script - COMPLETE (January 9, 2026)

### Script Implementation
- [x] Create `roadsense-v2v/ml/scripts/analyze_rtt_characterization.py`
- [x] Compute RTT statistics (mean, std, min, max, percentiles)
- [x] Calculate packet loss rate (overall + by time segment)
- [x] Extract IMU noise floor (std of accel when stationary)
- [x] Identify high-loss zones (military base segment)
- [x] Generate plots (RTT distribution, loss over time, GPS track)

### Output Files Generated:
| File | Location |
|------|----------|
| `analysis_summary.json` | `roadsense-v2v/ml/data/rtt_analysis/` |
| `loss_by_segment.csv` | `roadsense-v2v/ml/data/rtt_analysis/` |
| `rtt_distribution.png` | `roadsense-v2v/ml/data/rtt_analysis/` |
| `loss_over_time.png` | `roadsense-v2v/ml/data/rtt_analysis/` |
| `gps_track.png` | `roadsense-v2v/ml/data/rtt_analysis/` |

---

## ✅ Step 2: Emulator Parameters - COMPLETE (January 9, 2026)

- [x] Output `emulator_params.json` with:
  - Latency: base_ms, jitter_std_ms
  - Packet loss: base_rate, burst parameters
  - Sensor noise: accel_std_ms2
  - Domain randomization ranges

**File:** `roadsense-v2v/ml/espnow_emulator/emulator_params.json`

---

## 🚨 CRITICAL: Field Data Analysis Review (January 9, 2026)

### ⚠️ DATA QUALITY ASSESSMENT - READ BEFORE USING

| Metric | Measured Value | Expected Value | Usable? |
|--------|----------------|----------------|---------|
| **RTT Latency** | 4-5ms mean, 7ms p95 | 4-15ms | ✅ YES - Excellent |
| **IMU Noise** | 0.4-0.9 m/s² std | 0.1-1.0 m/s² | ✅ YES - Valid |
| **Packet Loss** | **63.5% overall** | 5-20% | ❌ NO - Catastrophic |
| **GPS Noise** | **195m std** | 2-10m | ❌ NO - Invalid |
| **Heading Noise** | **153.7° std** | 3-10° | ❌ NO - Invalid |
| **Max Burst Loss** | 133 packets (13.3s) | 10-20 packets | ⚠️ Extreme |

### Root Cause Analysis

**Why 63.5% packet loss?** (Normal ESP-NOW: 5-20%)
1. **Inter-vehicle distance likely too large** - No distance logging to confirm
2. **PCB antenna weakness** - Metal car body blocks signal
3. **Military base RF interference** - Confirmed in segments 330-630s
4. **No external antennas used**

**Why 195m GPS noise?** (Normal NEO-6M: 2-5m)
- "Stationary" detection (speed ≤ 0.5 m/s) captured slow-moving traffic
- GPS multipath errors from nearby structures
- Vehicle was not truly stationary during noise samples

### What IS Valid and Usable

```
✅ RTT Latency:
   mean=4.86ms, p50=5ms, p95=7ms, std=1.19ms
   → USE THIS for emulator latency model

✅ IMU Noise Floor:
   accel_x_std=0.44, accel_y_std=0.91, accel_z_std=0.37 m/s²
   → USE THIS for sensor noise model

✅ Data Pipeline:
   → Logging, CSV format, analysis script all working
```

### What MUST Be Corrected Before Training

The generated `emulator_params.json` contains **invalid values**:

```json
// WRONG - DO NOT USE AS-IS:
"packet_loss.base_rate": 0.5175    // Should be 0.05-0.15
"sensor_noise.gps_std_m": 195.14   // Should be 3-10
"sensor_noise.heading_std_deg": 153.74  // Should be 3-10
```

### Recommended Corrections

**Option A: Manual Override (Quick)**
Edit `emulator_params.json`:
```json
{
  "packet_loss": {
    "base_rate": 0.08,
    "rate_tier_1": 0.15,
    "rate_tier_2": 0.30,
    "rate_tier_3": 0.50
  },
  "sensor_noise": {
    "gps_std_m": 5.0,
    "heading_std_deg": 5.0
  },
  "domain_randomization": {
    "loss_rate_range": [0.02, 0.25]
  }
}
```

**Option B: Re-run Field Test (Recommended)**
1. Reduce inter-vehicle distance: 10-50m max
2. Add external 2.4GHz antennas
3. Set ESP32 TX power to max: `esp_wifi_set_max_tx_power(84)`
4. Avoid RF interference zones
5. Log inter-vehicle distance (both GPSes)
6. Include 60s parked calibration for true stationary noise

**Option C: Dual-Use Strategy**
- Use current data as "adversarial worst-case"
- Collect "normal conditions" data separately
- Train with domain randomization between both

---

## ✅ Stationary Test: COMPLETED (January 10, 2026)

### Test Setup
- **Location:** Abba Hillel Silver Road 28, Ramat Gan (Diamond Exchange area)
- **Sender (V001):** Outside with GPS fix, Linux laptop
- **Reflector (V002):** Windows laptop, various distances
- **Duration:** ~22 minutes total
- **Distances tested:** 20m → 9m → 5m

### Critical Discovery: PCB Antenna Range Limitation

| Distance | Packet Loss | Status |
|----------|-------------|--------|
| **20m** | ~100% | ❌ Unusable - nearly total loss |
| **9m** | 70-80% | ❌ Bad - too much loss |
| **5m** | **15.7%** | ✅ **USABLE** |

**Root cause of field drive failure confirmed:** Likely combination of:
1. **Laptop sleep mode** - Reflector laptop may have slept, killing V002
2. **Distance too large** - PCB antenna only works reliably within ~10m
3. **NOT primarily military base interference** (though that contributed)

### 5m Data Analysis Results

Extracted data from 850-1345 seconds (when at 5m distance):

| Metric | Field Drive (Invalid) | **5m Stationary (VALID)** |
|--------|----------------------|---------------------------|
| **Packet Loss** | 63.5% | **15.7%** ✅ |
| **GPS Noise** | 195m ❌ | **3.63m** ✅ |
| **IMU Noise** | 0.4-0.9 m/s² | **0.07-0.17 m/s²** ✅ |
| **RTT Mean** | 4.86ms | **6.74ms** ✅ |
| **RTT p95** | 7ms | **13ms** ✅ |

### Valid Emulator Parameters Generated

**File:** `roadsense-v2v/ml/espnow_emulator/emulator_params_5m.json`

```json
{
  "packet_loss": {
    "base_rate": 0.08,           // 8% baseline
    "rate_tier_1": 0.14,         // 14% at medium range
    "rate_tier_2": 0.29          // 29% at longer range
  },
  "sensor_noise": {
    "gps_std_m": 3.63,           // Valid GPS noise
    "accel_std_ms2": 0.12        // Excellent IMU noise
  },
  "latency": {
    "base_ms": 3.1,              // One-way latency
    "jitter_std_ms": 1.05        // Jitter
  },
  "domain_randomization": {
    "loss_rate_range": [0.04, 0.16]  // 4-16% for training variation
  }
}
```

### Data Files

| File | Description | Location |
|------|-------------|----------|
| `rtt_log.csv` | Full stationary test (22 min) | `/home/amirkhalifa/RoadSense2/` |
| `rtt_log_5m_only.csv` | Filtered 5m data (8 min) | `/home/amirkhalifa/RoadSense2/` |
| `emulator_params_5m.json` | Valid emulator params | `ml/espnow_emulator/` |
| `rtt_analysis_5m/` | Analysis of 5m data | `ml/data/` |

### Backups Preserved

| File | Location |
|------|----------|
| `rtt_log_field_drive_sheba.csv` | `data_backups/jan9_field_drive/` |
| `emulator_params_field_drive.json` | `data_backups/jan9_field_drive/` |
| `rtt_analysis_field_drive/` | `data_backups/jan9_field_drive/` |

---

## 🔧 Hardware Limitation: PCB Antenna Range

### The Problem
ESP32 DevKit PCB antenna has **effective range of only 5-10 meters** in real conditions:
- Buildings/obstacles attenuate signal
- Metal car bodies block signal
- Indoor environments have multipath interference

### Solutions

**Option 1: External Antenna (Recommended for Vehicles)**
- Cost: ~$5-10 per unit
- Improvement: 10x range (50-100m+)
- Requirements: ESP32 with U.FL/IPEX connector + 2.4GHz SMA antenna

**Option 2: Accept Short Range**
- Use 5m data (15% loss) as baseline
- Train RL model with 5-30% loss domain randomization
- Note: 10m range may be insufficient for highway V2V

**Option 3: Different Protocol**
- LoRa for longer range (but higher latency)
- 802.11p dedicated V2V hardware
- Major redesign required

### Recommendation for Project
1. Use `emulator_params_5m.json` for training (valid data)
2. Document PCB antenna limitation in thesis
3. Consider external antennas for future field tests
4. Train with domain randomization for 4-30% loss range

---

## 🔜 Next Steps: Emulator Integration

### Step 3: Build/Test ESP-NOW Emulator
- [ ] Copy `emulator_params_5m.json` to main `emulator_params.json`
- [ ] Implement `ESPNOWEmulator` class in `roadsense-v2v/ml/`
- [ ] Test with synthetic data
- [ ] Verify distribution matches 5m measured data

### Step 4: SUMO + RL Integration
- [ ] Create `ConvoyEnv` Gym environment
- [ ] Integrate emulator with SUMO TraCI
- [ ] Begin PPO/DQN training with realistic communication effects

---

## Commands Reference

### Run all RTT tests:
```bash
pio test -f "test_rtt*" -e esp32dev
```

### Run specific test:
```bash
pio test -e esp32dev -f test_rtt_csv_format
```

### Build integration firmware (Phase 3):
```bash
pio run -e rtt_sender
pio run -e rtt_reflector
```

---

**Last Updated:** January 10, 2026 (Stationary Test Complete - Valid Data Obtained)
**Current Phase:** ✅ CHARACTERIZATION COMPLETE - VALID EMULATOR PARAMS GENERATED
**Status:** 5m data valid (15.7% loss, 3.63m GPS noise, 0.12 m/s² IMU noise) | PCB antenna limited to ~10m range
**Next Phase:** 🛠️ EMULATOR INTEGRATION
**Next Steps:** 1) Copy emulator_params_5m.json to main params, 2) Build ESPNOWEmulator class, 3) Integrate with SUMO ConvoyEnv
