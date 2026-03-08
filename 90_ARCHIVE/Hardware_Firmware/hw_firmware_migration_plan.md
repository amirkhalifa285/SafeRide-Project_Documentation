# Hardware Firmware Implementation Plan
**RoadSense V2V Enhanced System**

**Status**: Active Optimization & Characterization
**Last Updated**: January 1, 2026
**Current Phase**: Phase 4 (ML Integration & RTT Characterization)

---

## ⚡ Hardware Configuration (The Truth)

**Vehicle Units (V001, V002, V003)**
- **MCU**: ESP32 DevKit V1
- **IMU**: **MPU6500** (6-axis: Accel + Gyro) @ 0x68
  - *Note:* Driver uses modified `MPU9250_WE` library.
- **Magnetometer**: **QMC5883L** (3-axis Compass) @ 0x0D
  - *Status:* Hardware validated. Driver implementation pending integration.
- **GPS**: **NEO-6M** (UART2)
  - ⚠️ **CRITICAL:** Must be powered by **5V (USB pin)**. 3.3V is insufficient.
- **Storage**: MicroSD Module (SPI)
  - Used for **RTT Characterization Logging** and **Model Inference Logging** (Debug).
- **Network**: **ESP-NOW** (2.4GHz)
  - Latency: 10-50ms (stochastic).
  - Payload: 90 bytes (Current V2VMessage struct).

---

## 🏗️ Firmware Architecture

### Layered Design

```
┌────────────────────────────────────────────────┐
│  APPLICATION LAYER (core/)                     │
│  VehicleState, HazardDetector, AlertManager   │
└────────────────┬───────────────────────────────┘
                 │
┌────────────────▼───────────────────────────────┐
│  ML LAYER (ml/)                                │
│  MLInference (TFLite), FeatureExtractor       │
└────────────────┬───────────────────────────────┘
                 │
┌────────────────▼───────────────────────────────┐
│  SENSOR LAYER (sensors/)                       │
│  IImuSensor (MPU6500 + QMC5883L), IGpsSensor  │
└────────────────┬───────────────────────────────┘
                 │
┌────────────────▼───────────────────────────────┐
│  NETWORK LAYER (network/)                      │
│  ├─ Protocol: V2VMessage (BSM format)         │
│  ├─ Transport: EspNowTransport                │
│  └─ Mesh: PeerManager, PackageManager         │
└────────────────────────────────────────────────┘
```

---

## 📅 Active Implementation Roadmap

### 1. Network Characterization (Current Focus)
- [ ] **Goal**: Measure real-world ESP-NOW behavior to tune the SUMO Emulator.
- [ ] **Firmware**: `RTT_Characterization_Firmware` (Ping-Pong logic).
- [ ] **Action**: Record RTT, RSSI, and Packet Loss at various distances (10m, 50m, 100m).
- [ ] **Output**: `emulator_params.json` (Distribution parameters for Python).
- [ ] **See**: `10_PLANS_ACTIVE/ESPNOW_CHARACTERIZATION_IMPLEMENTATION.md`

### 2. Sim-to-Real ML Deployment
- [ ] **Training**: Done purely in SUMO using the Python Emulator.
- [ ] **Model Conversion**: `model.tflite` (Quantized INT8).
- [ ] **Firmware Integration**:
    - `FeatureExtractor`: Must match the Python training environment exactly.
    - `MLInference`: Run TFLite Micro interpreter on ESP32.
- [ ] **Validation Drive**:
    - Drive real cars to *test* the model.
    - Log model predictions vs reality for analysis.

### 3. Magnetometer Integration
- [ ] **Driver**: Integrate `QMC5883LDriver` into `IImuSensor`.
- [ ] **Fusion**: Fuse Mag data with Gyro/Accel for robust heading (Tilt-Compensated Compass).
- [ ] **Calibration**: Implement hard/soft iron calibration routine.

---

## 📦 V2VMessage Structure (Current)

**Size: 90 Bytes** (Fits well within 250 byte limit)

```cpp
struct V2VMessage {
    // Header (13 bytes)
    uint8_t version;
    char vehicleId[8];
    uint32_t timestamp;

    // Position (12 bytes)
    float lat, lon, alt;

    // Dynamics (16 bytes)
    float speed, heading, longAccel, latAccel;

    // Sensors (36 bytes)
    float accel[3]; // x,y,z
    float gyro[3];  // x,y,z
    float mag[3];   // x,y,z (Currently 0, pending QMC5883L)

    // Alert (6 bytes)
    uint8_t riskLevel;
    uint8_t scenarioType;
    float confidence;

    // Mesh Metadata (7 bytes)
    uint8_t hopCount;
    uint8_t sourceMAC[6];
};
```

---

## 🛠️ Verification Checklist

### Before Characterization Drive
- [ ] **Power Check**: GPS VCC connected to 5V?
- [ ] **SD Check**: Card inserted? `test_sd_card` passes?
- [ ] **Firmware**: Flashed with RTT Logging logic?
- [ ] **Battery**: Sufficient power for 20 mins of testing?

### Field Protocol (RTT Only)
1.  **Static Test**: Place units 10m apart. Record 5 mins.
2.  **Distance Test**: Move units to 50m, then 100m. Record.
3.  **Motion Test**: One unit drives away while logging.
