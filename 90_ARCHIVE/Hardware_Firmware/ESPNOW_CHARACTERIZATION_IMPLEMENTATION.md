# ESP-NOW Characterization Implementation Guide

**Document Type:** AI Agent Implementation Prompt
**Created:** December 23, 2025
**Author:** Amir Khalifa (with Claude analysis)
**Priority:** CRITICAL - Blocking RL training pipeline
**Target Completion:** Implementation this week, data collection next week

> **UPDATE (December 26, 2025):** IMU/mag logging during RTT characterization is now **OPTIONAL**. Focus on network characterization (RTT, loss, jitter). Sensor noise measurement is a stretch goal. See `docs/ARCHITECTURE_V2_MINIMAL_RL.md`.

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Project Context](#project-context)
3. [What Exists vs What Needs Building](#what-exists-vs-what-needs-building)
4. [End Goals and Desired Outcomes](#end-goals-and-desired-outcomes)
5. [RTT Measurement Approach](#rtt-measurement-approach)
6. [Implementation Plan](#implementation-plan)
7. [Firmware Changes Per Unit](#firmware-changes-per-unit)
8. [Data Collection Procedure](#data-collection-procedure)
9. [Analysis Pipeline](#analysis-pipeline)
10. [Integration with Existing Codebase](#integration-with-existing-codebase)
11. [Testing Checklist](#testing-checklist)
12. [Connection to RL Pipeline](#connection-to-rl-pipeline)

---

## Executive Summary

### What This Document Is

This is a detailed implementation guide for an AI agent to assist with building the ESP-NOW RTT (Round-Trip Time) characterization system. The goal is to measure real ESP-NOW communication characteristics (latency, packet loss, jitter) from actual ESP32 hardware, then use these measurements to build an accurate emulator for RL training.

### Why This Matters

- **Problem**: SUMO simulation provides perfect vehicle data. Real ESP-NOW has latency (10-80ms), packet loss (2-10%), and jitter.
- **Impact**: Training on perfect data → Sim2Real failure (model won't work on real hardware)
- **Solution**: Measure real ESP-NOW → Build emulator → Train with realistic communication effects

### Key Decision Made

**RTT-based measurement** chosen over DataLogger Mode 1 because:
- All timing measured on ONE device (no clock synchronization needed)
- More accurate results
- Cleaner measurement (dedicated test, not mixed with normal operation)

### Practical Constraints

- **No exact distance measurements** during drive (too distracting/dangerous)
- Will log GPS position instead → calculate distances in post-processing
- Drive will be "random" - varying distances naturally as vehicles move
- Implementation happens THIS WEEK, data collection NEXT WEEK

---

## Project Context

### The RoadSense V2V System

RoadSense is a Vehicle-to-Vehicle hazard detection system using:
- 3 ESP32 DevKit units (V001, V002, V003)
- MPU6500 6-axis IMU (accelerometer + gyroscope)
- NEO-6M GPS modules
- QMC5883L magnetometer (hardware validated, driver needed)
- ESP-NOW mesh networking
- SD card data logging
- On-device ML inference (TensorFlow Lite)

### 🆕 Sensor Characterization (December 25, 2025) - OPTIONAL

**OPTIONAL/STRETCH:** The RTT characterization drive CAN log IMU/mag data to measure real sensor noise for the SensorSynthesizer. This is NOT blocking - implement only if time permits.

This allows us to:
1. Derive gyro_z from GPS heading changes
2. Compare to actual MPU6500 gyro_z readings
3. Extract REAL noise profile (not guessed from datasheets)
4. Use measured noise in SensorSynthesizer for realistic training data

### The RL Training Pipeline

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         COMPLETE RL PIPELINE                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  [YOU ARE HERE]                                                              │
│       ↓                                                                      │
│  1. ESP-NOW Characterization ← CURRENT TASK                                  │
│     - Measure RTT, packet loss, jitter                                       │
│     - Output: emulator_params.json                                           │
│       ↓                                                                      │
│  2. ESP-NOW Emulator (Python)                                                │
│     - Uses measured parameters                                               │
│     - Adds realistic communication effects to SUMO data                      │
│       ↓                                                                      │
│  3. SUMO Gym Environment                                                     │
│     - ConvoyEnv class                                                        │
│     - Integrates SUMO + Emulator                                             │
│       ↓                                                                      │
│  4. Scenario Generation (50+ scenarios)                                      │
│     - Parameter variations                                                   │
│     - Emergency braking injection                                            │
│       ↓                                                                      │
│  5. RL Training (100,000+ episodes)                                          │
│     - stable-baselines3 PPO                                                  │
│     - Agent learns from rewards                                              │
│       ↓                                                                      │
│  6. TFLite Conversion → Deploy on ESP32                                      │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Single-Agent Architecture (Professor Requirement)

- ML model runs in **ONE vehicle only** (V001 - the follower/rear vehicle)
- V001 receives V2V messages from V002, V003
- Model decides actions for V001 only (Maintain, Caution, Brake, Emergency)
- For characterization: V001 = sender/measurer, V002 = reflector

---

## What Exists vs What Needs Building

### Already Implemented (DO NOT RECREATE)

| Component | Location | Status |
|-----------|----------|--------|
| MPU6500 Driver | `hardware/src/sensors/imu/MPU6500Driver.*` | ✅ Complete |
| NEO6M GPS Driver | `hardware/src/sensors/gps/NEO6M_Driver.*` | ✅ Complete |
| EspNowTransport | `hardware/src/network/transport/EspNowTransport.*` | ✅ Complete |
| V2VMessage Protocol | `hardware/src/network/protocol/V2VMessage.h` | ✅ Complete |
| PackageManager | `hardware/src/network/mesh/PackageManager.*` | ✅ Complete |
| DataLogger (Mode 1 & 2) | `hardware/src/logging/DataLogger.*` | ✅ Complete |
| Logger Utility | `hardware/src/utils/Logger.*` | ✅ Complete |
| Config System | `hardware/src/config.h` | ✅ Complete |
| QMC5883L Hardware | I²C 0x0D, tested | ✅ Validated |
| SUMO Exercise 1-3 | `sumo_learning/exercise1-3_*` | ✅ Complete |

### Needs Building (THIS TASK)

| Component | Priority | Notes |
|-----------|----------|-------|
| RTT Sender Firmware | **CRITICAL** | For V001 - sends packets, measures RTT |
| RTT Reflector Firmware | **CRITICAL** | For V002 - echoes packets immediately |
| Analysis Script | **CRITICAL** | Python script to process RTT data |
| `emulator_params.json` | **CRITICAL** | Output of analysis |
| QMC5883L Driver (stub) | MEDIUM | To compile cleanly with no warnings |

### Related but Separate Tasks (DO NOT IMPLEMENT NOW)

| Component | Priority | When |
|-----------|----------|------|
| ESPNOWEmulator Python | HIGH | After characterization complete |
| ConvoyEnv Gym | HIGH | After emulator complete |
| SUMO Exercise 4-5 | HIGH | Parallel with emulator |
| DataLogger v2 (Training) | MEDIUM | Before real-world validation |
| QMC5883L Full Driver | LOW | After RL pipeline working |

---

## End Goals and Desired Outcomes

### Immediate Goal (This Task)

**Create firmware and analysis tools to measure real ESP-NOW characteristics**

Output files:
1. `test/espnow_rtt/sender_main.cpp` - RTT sender firmware for V001
2. `test/espnow_rtt/reflector_main.cpp` - Reflector firmware for V002
3. `scripts/analyze_rtt_characterization.py` - Analysis script
4. `data/characterization/emulator_params.json` - Measured parameters

### Success Criteria

After implementation, the user should be able to:
1. Flash sender firmware to V001
2. Flash reflector firmware to V002
3. Drive around for 20-30 minutes (varying distances naturally)
4. Retrieve SD card from V001
5. Run analysis script
6. Get `emulator_params.json` with measured parameters

### Output Format: emulator_params.json

```json
{
  "metadata": {
    "collection_date": "2025-12-XX",
    "duration_minutes": 25,
    "total_packets": 15000,
    "firmware_version": "1.0.0"
  },
  "latency": {
    "description": "One-way latency parameters (RTT/2)",
    "min_ms": 5,
    "max_ms": 45,
    "mean_ms": 15,
    "median_ms": 12,
    "std_ms": 8,
    "p95_ms": 35,
    "p99_ms": 42
  },
  "packet_loss": {
    "description": "Packet loss characteristics",
    "overall_rate": 0.023,
    "burst_detected": true,
    "max_burst_length": 3,
    "mean_burst_length": 1.5
  },
  "distance_correlation": {
    "description": "How latency/loss correlate with GPS distance",
    "distances_m": [10, 30, 50, 80, 100],
    "latency_at_distance_ms": [12, 14, 18, 25, 35],
    "loss_at_distance_pct": [1.5, 2.0, 3.5, 8.0, 15.0]
  },
  "domain_randomization": {
    "description": "Wider ranges for robust training",
    "latency_range_ms": [3, 60],
    "loss_rate_range": [0.0, 0.20]
  }
}
```

---

## RTT Measurement Approach

### Why RTT (Not One-Way Latency)

**Problem**: To measure one-way latency, both ESP32s need synchronized clocks.
**ESP32 Issue**: Internal clocks drift ~20ppm (milliseconds per minute).
**Solution**: Use Round-Trip Time (RTT) - all timing on ONE device.

### RTT Measurement Flow

```
V001 (Sender)                         V002 (Reflector)
     │                                      │
     │  1. Record T1 = millis()             │
     │  2. Send packet (seq=N, T1)          │
     │─────────────────────────────────────→│
     │                                      │ 3. Receive packet
     │                                      │ 4. Echo IMMEDIATELY
     │←─────────────────────────────────────│    (same packet, no modification)
     │  5. Record T2 = millis()             │
     │  6. RTT = T2 - T1                    │
     │  7. One-way ≈ RTT / 2                │
     │                                      │
     │  8. Log to SD: seq, T1, T2, RTT,     │
     │     GPS lat/lon (for distance calc)  │
```

### Key Design Decisions

1. **V001 does ALL timing** - No clock sync needed
2. **V002 echoes immediately** - Minimal processing delay
3. **Include GPS position** - Calculate distance in post-processing (not during drive)
4. **Log every packet** - No buffering, need accurate timing
5. **10 Hz rate** - Matches real system (100ms between packets)
6. **Packet size matches V2VMessage** - ~90 bytes (realistic payload)

### What We're Measuring

| Metric | How | Why |
|--------|-----|-----|
| RTT | T2 - T1 on V001 | One-way latency ≈ RTT/2 |
| Packet Loss | Missing sequence numbers | Emulator loss probability |
| Jitter | Variance in RTT | Latency distribution shape |
| Distance Correlation | GPS positions at send time | Loss/latency vs distance |
| Burst Loss | Consecutive missing packets | Bursty vs random loss |

---

## Implementation Plan

### Directory Structure

```
roadsense-v2v/hardware/
├── test/
│   └── espnow_rtt/                    ← NEW DIRECTORY
│       ├── sender_main.cpp            ← V001 firmware (sender + RTT measurement)
│       ├── reflector_main.cpp         ← V002 firmware (echo reflector)
│       └── platformio_envs.ini        ← Build configurations (reference)
├── src/
│   ├── sensors/
│   │   └── magnetometer/              ← NEW DIRECTORY (for clean build)
│   │       ├── QMC5883LDriver.h       ← Interface stub (compiles cleanly)
│   │       └── QMC5883LDriver.cpp     ← Minimal implementation
│   └── ... (existing files unchanged)
└── platformio.ini                     ← Add new build environments

scripts/
└── analyze_rtt_characterization.py    ← NEW FILE (analysis script)

data/
└── characterization/                  ← NEW DIRECTORY (created by script)
    ├── raw/                           ← Raw CSV files from SD card
    ├── analysis/                      ← Analysis outputs
    └── emulator_params.json           ← FINAL OUTPUT
```

### Build Environments (platformio.ini additions)

```ini
; Add to existing platformio.ini

[env:rtt_sender]
platform = espressif32
board = esp32dev
framework = arduino
monitor_speed = 115200
build_flags =
    -DVEHICLE_ID=\"V001\"
    -DRTT_MODE_SENDER=1
build_src_filter = +<../test/espnow_rtt/sender_main.cpp>
lib_deps =
    greiman/SdFat@^2.2.0
    mikalhart/TinyGPSPlus@^1.0.3

[env:rtt_reflector]
platform = espressif32
board = esp32dev
framework = arduino
monitor_speed = 115200
build_flags =
    -DVEHICLE_ID=\"V002\"
    -DRTT_MODE_REFLECTOR=1
build_src_filter = +<../test/espnow_rtt/reflector_main.cpp>
```

---

## Firmware Changes Per Unit

### V001 (Sender Unit) - DETAILED

**Purpose**: Send packets, measure RTT, log to SD card with GPS position AND IMU/mag data

**Hardware Required**:
- ESP32 DevKit
- SD card module (CS=5, already wired)
- NEO-6M GPS (TX=16, RX=17, VCC=5V USB pin!)
- MPU6500 IMU (I2C: SDA=21, SCL=22, address 0x68)
- QMC5883L Magnetometer (I2C: SDA=21, SCL=22, address 0x0D)
- Power: USB or battery

**Firmware File**: `test/espnow_rtt/sender_main.cpp`

**Key Functions**:
1. Initialize SD card (10 MHz SPI - conservative speed)
2. Initialize GPS (for position + heading logging)
3. Initialize MPU6500 (for gyro/accel logging)
4. Initialize QMC5883L (for magnetometer logging)
5. Initialize ESP-NOW (broadcast mode)
6. Send packets at 10 Hz with sequence number and timestamp
7. Listen for echoes, record receive time
8. Read IMU/mag sensors at each packet send
9. Log to SD: sequence, send_time, recv_time, RTT, lat, lon, heading, gyro[3], accel[3], mag[3], lost
10. Print progress to Serial every 100 packets

**Config Changes for V001**:
```cpp
// In sender_main.cpp or passed via build flags
#define VEHICLE_ID "V001"
#define PACKET_INTERVAL_MS 100  // 10 Hz
#define TEST_DURATION_MS (30 * 60 * 1000)  // 30 minutes
#define LOG_FILE "/rtt_data.csv"

// I2C addresses
#define MPU6500_ADDR 0x68
#define QMC5883L_ADDR 0x0D
```

### V002 (Reflector Unit) - DETAILED

**Purpose**: Receive packets, echo immediately (no processing)

**Hardware Required**:
- ESP32 DevKit
- Power: USB or battery
- NO SD card needed
- NO GPS needed (V001 logs both positions via V2V messages)

**Firmware File**: `test/espnow_rtt/reflector_main.cpp`

**Key Functions**:
1. Initialize ESP-NOW (receive + send)
2. On packet receive: echo immediately (same data, no modification)
3. Print progress to Serial every 100 packets (optional)

**Config Changes for V002**:
```cpp
// In reflector_main.cpp or passed via build flags
#define VEHICLE_ID "V002"
// No other config needed - reflector is simple
```

### What Changes Between Units (Summary)

| Setting | V001 (Sender) | V002 (Reflector) |
|---------|---------------|------------------|
| VEHICLE_ID | "V001" | "V002" |
| Build environment | `rtt_sender` | `rtt_reflector` |
| SD Card | ✅ Required | ❌ Not needed |
| GPS | ✅ Required | ❌ Not needed |
| MPU6500 IMU | ✅ Required | ❌ Not needed |
| QMC5883L Mag | ✅ Required | ❌ Not needed |
| Firmware file | `sender_main.cpp` | `reflector_main.cpp` |
| Role | Send, measure, log (GPS + IMU + mag) | Echo only |

---

## Data Collection Procedure

### Pre-Drive Checklist

1. **V001 (Sender)**:
   - [ ] Flash `rtt_sender` firmware: `pio run -e rtt_sender -t upload`
   - [ ] Insert SD card (FAT32 formatted, SanDisk 32GB recommended)
   - [ ] Connect GPS (VCC to 5V USB pin!)
   - [ ] Verify Serial output shows "Ready to start"
   - [ ] Press button or wait 10s to start

2. **V002 (Reflector)**:
   - [ ] Flash `rtt_reflector` firmware: `pio run -e rtt_reflector -t upload`
   - [ ] Verify Serial output shows "Reflector ready"
   - [ ] No SD card or GPS needed

### During Drive

**IMPORTANT: No exact distance measurements needed!**

The drive should be "random" - just drive normally with varying distances:
- Start with vehicles close (parking lot)
- Drive on roads with varying gaps
- Sometimes close (10-20m), sometimes far (50-100m)
- Include some line-of-sight and some with obstacles
- Duration: 20-30 minutes
- GPS will log positions → distances calculated in post-processing

**What to Observe (Optional)**:
- Serial monitor on V001 shows progress
- Packet count increases steadily
- Loss percentage displayed every 100 packets

### Post-Drive

1. **Stop V001** - Press button or power off (auto-flushes data)
2. **Retrieve SD card from V001**
3. **Copy `/rtt_data.csv` to computer**
4. **Run analysis script**

---

## Analysis Pipeline

### Analysis Script: `scripts/analyze_rtt_characterization.py`

**Input**: `rtt_data.csv` from SD card
**Output**: `emulator_params.json` AND `sensor_noise_params.json`

**Analysis Steps**:

1. **Load CSV data**
2. **Calculate packet loss**:
   - Total packets sent vs received
   - Identify missing sequence numbers
   - Detect burst patterns (consecutive losses)
3. **Analyze RTT distribution**:
   - Min, max, mean, median, std
   - Percentiles (p95, p99)
   - Check for bimodal distribution (indicates retransmissions)
4. **Calculate distance from GPS**:
   - Haversine formula between V001 and V002 positions
   - Correlate distance with RTT and loss
5. **🆕 Analyze sensor noise** (for SensorSynthesizer):
   - Compare `gyro_z` to `d(heading)/dt` → extract gyro noise std
   - Analyze magnetometer variance vs expected heading-based values
   - Extract accelerometer noise profile
6. **Generate emulator parameters**:
   - Base latency model
   - Loss probability model
   - Domain randomization ranges (train wider than measured)
7. **🆕 Generate sensor noise parameters**:
   - `gyro_z_std_rad_s`: Measured gyro noise
   - `mag_xy_std_ut`: Measured mag noise
   - `accel_std_ms2`: Measured accel noise
8. **Create visualizations**:
   - RTT histogram
   - Loss vs distance plot
   - Time series of RTT
   - 🆕 Gyro noise distribution
   - 🆕 Mag noise vs heading

### 🆕 Sensor Noise Analysis Functions

```python
def analyze_gyro_noise(df: pd.DataFrame) -> dict:
    """
    Compare actual gyro_z to expected (from GPS heading changes).

    Returns noise parameters for SensorSynthesizer.
    """
    # Calculate expected gyro_z from heading changes
    dt = (df['send_time_ms'].diff() / 1000.0)  # seconds
    dheading = df['heading'].diff()

    # Handle wraparound (e.g., 359 -> 1 = +2, not -358)
    dheading = dheading.apply(lambda x: x if abs(x) < 180 else x - np.sign(x) * 360)

    expected_gyro_z = np.radians(dheading) / dt  # rad/s
    actual_gyro_z = df['gyro_z']

    # Calculate residual (noise)
    residual = actual_gyro_z - expected_gyro_z
    residual = residual.dropna()

    return {
        'gyro_z_mean_offset': float(residual.mean()),
        'gyro_z_std_rad_s': float(residual.std()),
        'gyro_z_p95': float(np.percentile(np.abs(residual), 95)),
    }

def analyze_mag_noise(df: pd.DataFrame) -> dict:
    """
    Compare actual mag readings to expected (from GPS heading).

    Returns noise parameters for SensorSynthesizer.
    """
    # Expected magnetic field from heading
    heading_rad = np.radians(df['heading'])
    mag_intensity = 50.0  # Approximate Earth's field (µT)

    expected_mag_x = mag_intensity * np.cos(heading_rad)
    expected_mag_y = mag_intensity * np.sin(heading_rad)

    # Calculate residuals
    residual_x = df['mag_x'] - expected_mag_x
    residual_y = df['mag_y'] - expected_mag_y

    # Combined XY noise (what matters for heading calculation)
    combined_std = np.sqrt(residual_x.var() + residual_y.var()) / np.sqrt(2)

    return {
        'mag_x_std_ut': float(residual_x.std()),
        'mag_y_std_ut': float(residual_y.std()),
        'mag_xy_std_ut': float(combined_std),
        'mag_z_std_ut': float(df['mag_z'].std()),
    }

def analyze_accel_noise(df: pd.DataFrame) -> dict:
    """
    Analyze accelerometer noise during steady-state driving.

    Focus on periods of constant speed (low expected acceleration).
    """
    # Find periods of ~constant speed (speed change < 0.5 m/s over 1s)
    speed_change = df['speed'].diff().abs() if 'speed' in df else None

    # Use accel_z as baseline (should be ~9.81 always)
    accel_z_residual = df['accel_z'] - 9.81

    return {
        'accel_x_std_ms2': float(df['accel_x'].std()),
        'accel_y_std_ms2': float(df['accel_y'].std()),
        'accel_z_std_ms2': float(accel_z_residual.std()),
        'accel_std_ms2': float(np.mean([
            df['accel_x'].std(),
            df['accel_y'].std(),
            accel_z_residual.std()
        ])),
    }
```

### Output: sensor_noise_params.json

```json
{
  "metadata": {
    "source": "RTT characterization drive",
    "date": "2025-12-XX",
    "samples": 15000
  },
  "gyro": {
    "z_mean_offset_rad_s": 0.002,
    "z_std_rad_s": 0.015,
    "z_p95_rad_s": 0.028
  },
  "mag": {
    "x_std_ut": 4.2,
    "y_std_ut": 4.8,
    "xy_std_ut": 4.5,
    "z_std_ut": 2.1
  },
  "accel": {
    "x_std_ms2": 0.18,
    "y_std_ms2": 0.22,
    "z_std_ms2": 0.15,
    "combined_std_ms2": 0.18
  }
}
```

### CSV Format (Logged by V001)

```csv
sequence,send_time_ms,recv_time_ms,rtt_ms,lat,lon,heading,gyro_x,gyro_y,gyro_z,accel_x,accel_y,accel_z,mag_x,mag_y,mag_z,lost
0,1000,1012,12,32.085123,34.781234,45.2,0.01,-0.02,0.15,0.12,-0.08,9.78,25.3,42.1,38.5,0
1,1100,1115,15,32.085125,34.781236,45.5,0.02,-0.01,0.18,0.10,-0.05,9.81,25.5,42.3,38.2,0
2,1200,-1,-1,32.085127,34.781238,45.8,0.01,-0.02,0.12,0.08,-0.06,9.79,25.4,42.2,38.4,1
3,1300,1318,18,32.085129,34.781240,46.1,0.02,-0.01,0.20,0.15,-0.10,9.80,25.6,42.0,38.3,0
...
```

**Column Descriptions**:
- `sequence`: Packet sequence number (0, 1, 2, ...)
- `send_time_ms`: V001's millis() when packet sent
- `recv_time_ms`: V001's millis() when echo received (-1 if lost)
- `rtt_ms`: Round-trip time in ms (-1 if lost)
- `lat`, `lon`: V001's GPS position at send time
- `heading`: GPS heading in degrees (for gyro_z comparison)
- `gyro_x`, `gyro_y`, `gyro_z`: MPU6500 gyroscope readings (rad/s)
- `accel_x`, `accel_y`, `accel_z`: MPU6500 accelerometer readings (m/s²)
- `mag_x`, `mag_y`, `mag_z`: QMC5883L magnetometer readings (µT)
- `lost`: 1 if packet lost, 0 if received

**🆕 Why IMU/Mag Columns?**
These allow post-processing to:
1. Compare `gyro_z` to `d(heading)/dt` → extract gyro noise profile
2. Compare `mag` to expected values from heading → extract mag noise profile
3. Feed measured noise into SensorSynthesizer for realistic training data

---

## Integration with Existing Codebase

### Reusing Existing Components

The RTT firmware should reuse existing components where possible:

```cpp
// REUSE these existing components:
#include "../src/utils/Logger.h"           // Logging utility
#include "../src/config.h"                  // Pin definitions (SD_CS_PIN, etc.)

// DO NOT include these (not needed for RTT test):
// #include "../src/sensors/imu/MPU6500Driver.h"
// #include "../src/network/mesh/PackageManager.h"
// #include "../src/logging/DataLogger.h"  // Using custom simpler logging
```

### ESP-NOW Initialization (Must Match Existing)

**CRITICAL**: Use same ESP-NOW init sequence as `EspNowTransport.cpp`:

```cpp
// This sequence is FRAGILE - do not change order
WiFi.mode(WIFI_STA);
WiFi.disconnect();  // CRITICAL - removes auto-connect interference
esp_wifi_set_channel(1, WIFI_SECOND_CHAN_NONE);
esp_now_init();
esp_now_register_recv_cb(onReceive);
esp_now_register_send_cb(onSent);

// Add broadcast peer
esp_now_peer_info_t peerInfo = {};
memset(peerInfo.peer_addr, 0xFF, 6);  // Broadcast address
peerInfo.channel = 0;
peerInfo.encrypt = false;
esp_now_add_peer(&peerInfo);
```

### Packet Structure (Match V2VMessage Size)

The test packet should be similar size to real V2VMessage (~90 bytes):

```cpp
#pragma pack(push, 1)
struct RTTPacket {
    uint32_t sequence;      // Packet sequence number
    uint32_t send_time_ms;  // Sender's millis() at send
    float sender_lat;       // Sender's GPS lat
    float sender_lon;       // Sender's GPS lon
    uint8_t padding[74];    // Pad to ~90 bytes (matches V2VMessage)
};
#pragma pack(pop)

static_assert(sizeof(RTTPacket) == 90, "RTTPacket must be 90 bytes");
```

### QMC5883L Driver Stub (For Clean Build)

To compile the main firmware cleanly with no warnings, create a minimal QMC5883L driver stub:

```cpp
// QMC5883LDriver.h - Minimal stub for clean compilation
#ifndef QMC5883L_DRIVER_H
#define QMC5883L_DRIVER_H

#include <Arduino.h>

class QMC5883LDriver {
public:
    struct MagData {
        float x, y, z;  // Magnetic field in µT
        float heading;  // Calculated heading in degrees
        bool valid;
    };

    bool begin() { return false; }  // Stub - returns false (not available)
    MagData read() { return {0, 0, 0, 0, false}; }
    bool isAvailable() const { return false; }  // Stub
};

#endif
```

This allows the main firmware to compile cleanly while the magnetometer driver is not yet implemented.

---

## Testing Checklist

### Before Data Collection Drive

**Firmware Compilation**:
- [ ] `pio run -e rtt_sender` compiles with no errors
- [ ] `pio run -e rtt_reflector` compiles with no errors
- [ ] No warnings in compilation (clean build)

**V001 (Sender) Hardware Test**:
- [ ] SD card initializes (check Serial output)
- [ ] GPS gets fix (may take 1-2 minutes outdoors)
- [ ] ESP-NOW initializes
- [ ] First few packets show in Serial monitor
- [ ] RTT values displayed (should be 2-50ms)

**V002 (Reflector) Hardware Test**:
- [ ] ESP-NOW initializes
- [ ] "Packet received, echoing" messages in Serial
- [ ] Packet count increases

**Communication Test (Bench)**:
- [ ] Place V001 and V002 1m apart on desk
- [ ] Run for 60 seconds (600 packets)
- [ ] Check Serial: loss should be <1%
- [ ] Check Serial: RTT should be 2-15ms
- [ ] Stop test, check SD card has data

### After Data Collection Drive

**Data Validation**:
- [ ] CSV file exists on SD card
- [ ] File has expected number of rows (duration × 10)
- [ ] GPS coordinates are valid (not 0,0)
- [ ] RTT values are reasonable (2-100ms)
- [ ] Loss rate is reasonable (1-10%)

**Analysis Validation**:
- [ ] Script runs without errors
- [ ] `emulator_params.json` generated
- [ ] Latency values match expectations
- [ ] Loss rates match expectations
- [ ] Plots look reasonable

---

## Connection to RL Pipeline

### How This Connects to Training

After characterization is complete:

```
emulator_params.json
        │
        ▼
┌───────────────────────────────────────┐
│     ESPNOWEmulator (Python)           │
│                                       │
│  class ESPNOWEmulator:                │
│      def __init__(self, params_file): │
│          self.params = load(params)   │
│                                       │
│      def transmit(self, msg, dist):   │
│          # Add measured latency       │
│          # Apply measured loss prob   │
│          # Return None if lost        │
│          return modified_msg          │
└───────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────────┐
│     ConvoyEnv (Gym Environment)       │
│                                       │
│  SUMO → ESPNOWEmulator → Observation  │
│                                       │
│  Agent sees "realistic" V2V messages  │
│  with latency, loss, noise            │
└───────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────────┐
│     RL Training (stable-baselines3)   │
│                                       │
│  Agent learns robust policy that      │
│  works despite communication issues   │
└───────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────────┐
│     Deploy on ESP32                   │
│                                       │
│  Model works in real world because    │
│  it was trained with realistic        │
│  communication effects                │
└───────────────────────────────────────┘
```

### Why Accurate Characterization Matters

| If Emulator Has | But Real ESP-NOW Has | Result |
|-----------------|----------------------|--------|
| Gaussian latency (25ms mean) | Bimodal (15ms OR 80ms) | Model learns wrong timing |
| Independent 2% loss | Bursty loss (5 in a row) | Model underestimates danger |
| Linear loss vs distance | Cliff at 80m (sudden drop) | Model overconfident at range edge |

**Measuring real characteristics prevents these Sim2Real failures.**

---

## Implementation Instructions for AI Agent

### Step 1: Create Directory Structure

```bash
mkdir -p roadsense-v2v/hardware/test/espnow_rtt
mkdir -p roadsense-v2v/hardware/src/sensors/magnetometer
mkdir -p scripts
mkdir -p data/characterization/raw
mkdir -p data/characterization/analysis
```

### Step 2: Create QMC5883L Driver Stub

Create minimal stub for clean compilation:
- `hardware/src/sensors/magnetometer/QMC5883LDriver.h`
- `hardware/src/sensors/magnetometer/QMC5883LDriver.cpp`

### Step 3: Create RTT Sender Firmware

Create `hardware/test/espnow_rtt/sender_main.cpp`:
- Initialize SD card (reuse config.h pin definitions)
- Initialize GPS (reuse NEO6M patterns)
- Initialize ESP-NOW (match EspNowTransport.cpp sequence)
- Main loop: send packets at 10 Hz
- On echo receive: calculate RTT, log to SD
- Include GPS position in each log row

### Step 4: Create RTT Reflector Firmware

Create `hardware/test/espnow_rtt/reflector_main.cpp`:
- Initialize ESP-NOW only
- On packet receive: echo immediately
- Minimal code, fast response

### Step 5: Update platformio.ini

Add `rtt_sender` and `rtt_reflector` build environments.

### Step 6: Create Analysis Script

Create `scripts/analyze_rtt_characterization.py`:
- Load CSV from SD card
- Calculate statistics
- Detect burst patterns
- Correlate with GPS distance
- Generate `emulator_params.json`
- Create visualization plots

### Step 7: Test on Bench

Before real drive, test with both units on desk:
- 60 seconds, 1m apart
- Verify communication works
- Verify data logging works
- Verify reasonable RTT values

---

## Critical Reminders

1. **GPS VCC must use 5V USB pin** (NOT 3.3V - causes GPS failure)

2. **SD card SPI speed must be 10 MHz** (not 25 MHz - SdFat v2 requirement)

3. **ESP-NOW init sequence is fragile** - copy exactly from EspNowTransport.cpp

4. **No exact distance measurements during drive** - GPS positions logged, distances calculated in post-processing

5. **Packet size must match V2VMessage** (~90 bytes) for realistic measurement

6. **Reflector must echo IMMEDIATELY** - no processing, no logging, just echo

7. **Domain randomization** - train WIDER than measured (emulator params include expanded ranges)

---

## Files to Create (Summary)

| File | Purpose |
|------|---------|
| `hardware/test/espnow_rtt/sender_main.cpp` | V001 sender firmware |
| `hardware/test/espnow_rtt/reflector_main.cpp` | V002 reflector firmware |
| `hardware/src/sensors/magnetometer/QMC5883LDriver.h` | Stub header |
| `hardware/src/sensors/magnetometer/QMC5883LDriver.cpp` | Stub implementation |
| `scripts/analyze_rtt_characterization.py` | Analysis script |
| Updates to `hardware/platformio.ini` | New build environments |

---

## Document Version

**Version:** 1.1
**Created:** December 23, 2025
**Updated:** December 26, 2025 - IMU/mag logging marked optional (see ARCHITECTURE_V2_MINIMAL_RL.md)
**Author:** Amir Khalifa (with Claude analysis)
**Status:** Ready for implementation

---

## Appendix: Expected Output Examples

### Serial Output - V001 (Sender)

```
=== ESP-NOW RTT Characterization (Sender) ===
SD card initialized (10 MHz)
GPS initializing...
GPS fix acquired: 32.0851, 34.7812
ESP-NOW initialized
Waiting 10 seconds to start...
STARTING MEASUREMENT!
Sent: 100, Received: 98, Loss: 2.0%, Avg RTT: 14.2ms
Sent: 200, Received: 196, Loss: 2.0%, Avg RTT: 15.1ms
...
Sent: 15000, Received: 14650, Loss: 2.3%, Avg RTT: 16.8ms
MEASUREMENT COMPLETE!
Data saved to: /rtt_data.csv
```

### Serial Output - V002 (Reflector)

```
=== ESP-NOW RTT Characterization (Reflector) ===
ESP-NOW initialized
Reflector MAC: 24:6F:28:XX:XX:XX
Waiting for packets...
Packets echoed: 100
Packets echoed: 200
...
Packets echoed: 15000
```

### emulator_params.json (Example)

```json
{
  "metadata": {
    "collection_date": "2025-12-28",
    "duration_minutes": 25,
    "total_packets_sent": 15000,
    "total_packets_received": 14650,
    "overall_loss_rate": 0.0233
  },
  "latency": {
    "min_ms": 4,
    "max_ms": 78,
    "mean_ms": 16.8,
    "median_ms": 14,
    "std_ms": 9.2,
    "p95_ms": 35,
    "p99_ms": 52
  },
  "packet_loss": {
    "overall_rate": 0.0233,
    "burst_detected": true,
    "max_burst_length": 4,
    "mean_burst_length": 1.8
  },
  "distance_correlation": {
    "method": "gps_haversine",
    "bins_m": [0, 20, 40, 60, 80, 100, 150],
    "latency_by_bin_ms": [12, 14, 17, 22, 30, 45],
    "loss_by_bin_pct": [1.2, 1.8, 2.5, 4.0, 8.5, 18.0]
  },
  "domain_randomization": {
    "latency_range_ms": [2, 100],
    "loss_rate_range": [0.0, 0.25]
  }
}
```

---

**END OF DOCUMENT**
