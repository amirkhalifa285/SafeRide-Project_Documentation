# ESP-NOW RTT Characterization Firmware Implementation Guide

**Document Type:** AI Agent Implementation Prompt
**Created:** December 29, 2025
**Author:** Amir Khalifa
**Priority:** CRITICAL - Blocking RL training pipeline
**Estimated Implementation Time:** 2-3 hours

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Why RTT Mode is Required](#why-rtt-mode-is-required)
3. [Current State vs Target State](#current-state-vs-target-state)
4. [Reference Files (MUST READ)](#reference-files-must-read)
5. [Directory Structure](#directory-structure)
6. [Sender Firmware Specification](#sender-firmware-specification)
7. [Reflector Firmware Specification](#reflector-firmware-specification)
8. [PlatformIO Configuration](#platformio-configuration)
9. [RTT Packet Protocol](#rtt-packet-protocol)
10. [SD Card Logging Format](#sd-card-logging-format)
11. [Build and Flash Instructions](#build-and-flash-instructions)
12. [Test Procedure](#test-procedure)
13. [Expected Output](#expected-output)
14. [Common Pitfalls to Avoid](#common-pitfalls-to-avoid)

---

## Executive Summary

### What We Need

Build **dedicated RTT characterization firmware** for measuring ESP-NOW latency accurately:

1. **`sender_main.cpp`** - Runs on V001: Sends packets, waits for echo, measures RTT, logs to SD
2. **`reflector_main.cpp`** - Runs on V002: Receives packets, echoes immediately (no logging)

### Why We Can't Use Existing Firmware

The current `main.cpp` firmware does **bidirectional broadcasts** where each ESP32 sends its own messages. This creates a **clock synchronization problem**:

```
CURRENT (WRONG):                           RTT MODE (CORRECT):
─────────────────                          ────────────────────

V001 Clock: 47518ms                        V001 Clock: 47518ms
V002 Clock: 38604ms                        V002 Clock: ??? (DOESN'T MATTER!)
     ↑
  ~9 second drift!

V001 sends (47518) → V002 receives         V001 sends (T1=100) → V002 echoes → V001 receives (T2=107)
V002 sends (38604) → V001 receives         RTT = T2 - T1 = 7ms
                                           One-way latency = RTT / 2 = 3.5ms
Latency = ??? (clocks not synced!)
                                           ALL timestamps from ONE device!
```

### Key Principle

**RTT measures round-trip on ONE device** - no clock sync needed because V001 records both T1 (send) and T2 (receive) using its own clock.

---

## Why RTT Mode is Required

### The Clock Problem in Detail

ESP32's internal RTC drifts approximately **20 ppm** (parts per million), which means:
- ~1.7 seconds per day
- ~70ms per hour
- **Unpredictable offset** between two devices

From the test data we collected:
- V001 first message: `timestamp_local_ms = 47518`
- V002 first message: `timestamp_local_ms = 38604`
- **Offset: ~8914ms** (almost 9 seconds!)

This makes calculating one-way latency from bidirectional broadcasts **impossible** without complex clock synchronization.

### The RTT Solution

```
V001 (Sender)                              V002 (Reflector)
     │                                           │
     │  1. T1 = millis()                         │
     │  2. Send RTTPacket(seq=42, T1)            │
     │────────────────────────────────────────→  │
     │                                           │  3. Receive packet
     │                                           │  4. Echo IMMEDIATELY
     │  ←────────────────────────────────────────│     (same bytes, no change)
     │  5. T2 = millis()                         │
     │  6. RTT = T2 - T1                         │
     │  7. Log: seq, T1, T2, RTT, GPS...         │
     │                                           │
```

**Result:** Accurate latency measurement using only V001's clock!

---

## Current State vs Target State

### Current State (What Exists)

| Component | Status | Location |
|-----------|--------|----------|
| Main firmware (bidirectional) | ✅ Working | `hardware/src/main.cpp` |
| EspNowTransport | ✅ Working | `hardware/src/network/transport/EspNowTransport.*` |
| DataLogger | ✅ Working | `hardware/src/logging/DataLogger.*` |
| V2VMessage struct | ✅ Working | `hardware/src/network/protocol/V2VMessage.h` |
| Config (pins, etc.) | ✅ Working | `hardware/src/config.h` |
| RTT Sender firmware | ❌ **NOT BUILT** | `hardware/test/espnow_rtt/sender_main.cpp` |
| RTT Reflector firmware | ❌ **NOT BUILT** | `hardware/test/espnow_rtt/reflector_main.cpp` |

### Target State (What We Need)

```
roadsense-v2v/hardware/
├── test/
│   └── espnow_rtt/                    ← NEW DIRECTORY
│       ├── sender_main.cpp            ← NEW FILE (V001)
│       └── reflector_main.cpp         ← NEW FILE (V002)
├── src/
│   └── ... (existing, DO NOT MODIFY)
└── platformio.ini                     ← ADD new build environments
```

---

## Reference Files (MUST READ)

Before implementing, the AI agent MUST read these files to understand existing patterns:

### Critical Reference Files

| File | Purpose | Key Patterns to Reuse |
|------|---------|----------------------|
| `hardware/src/config.h` | Pin definitions | SD_CS_PIN, LED_STATUS_PIN, BUTTON_CALIB_PIN, I2C pins |
| `hardware/src/network/transport/EspNowTransport.cpp` | ESP-NOW initialization | **EXACT** init sequence (lines 66-140) - DO NOT DEVIATE |
| `hardware/src/logging/DataLogger.cpp` | SD card logging | SdFat initialization, file creation patterns |
| `hardware/src/main.cpp` | Button handling | Debounce pattern, LED control |
| `hardware/src/sensors/imu/MPU6500Driver.h` | IMU driver | **REUSE THIS** - don't rewrite! |
| `hardware/src/sensors/imu/MPU6500Driver.cpp` | IMU implementation | Calibration, read() method |

### ESP-NOW Initialization Sequence (CRITICAL - COPY EXACTLY)

From `EspNowTransport.cpp` lines 74-127:

```cpp
// CRITICAL: This initialization sequence order must not be changed!
// Step 1: Set WiFi to station mode
WiFi.mode(WIFI_STA);

// Step 2: Disconnect from any saved WiFi networks
// CRITICAL: This removes STA auto-connect interference
WiFi.disconnect();

// Step 3: Set WiFi channel
esp_wifi_set_channel(1, WIFI_SECOND_CHAN_NONE);

// Step 4: Initialize ESP-NOW
esp_now_init();

// Step 5: Register send callback
esp_now_register_send_cb(onDataSent);

// Step 6: Register receive callback
esp_now_register_recv_cb(onDataRecv);

// Step 7: Add broadcast peer (FF:FF:FF:FF:FF:FF)
esp_now_peer_info_t peerInfo;
memset(&peerInfo, 0, sizeof(peerInfo));
memset(peerInfo.peer_addr, 0xFF, 6);  // Broadcast address
peerInfo.channel = 0;
peerInfo.encrypt = false;
esp_now_add_peer(&peerInfo);
```

**WARNING:** Changing this order WILL cause ESP-NOW to fail silently!

### SD Card Initialization (10 MHz SPI - IMPORTANT)

From existing DataLogger patterns:

```cpp
#include <SdFat.h>

SdFat sd;

// IMPORTANT: Use 10 MHz, NOT 25 MHz (SdFat v2 requirement for reliability)
if (!sd.begin(SD_CS_PIN, SD_SCK_MHZ(10))) {
    // Handle error
}
```

---

## Directory Structure

### Create These Directories and Files

```bash
# From roadsense-v2v/hardware/
mkdir -p test/espnow_rtt
touch test/espnow_rtt/sender_main.cpp
touch test/espnow_rtt/reflector_main.cpp
```

### Final Structure

```
roadsense-v2v/hardware/
├── platformio.ini                      ← MODIFY (add 2 new environments)
├── src/
│   ├── config.h                        ← READ ONLY (use for pin definitions)
│   ├── main.cpp                        ← DO NOT MODIFY
│   ├── network/
│   │   └── transport/
│   │       └── EspNowTransport.cpp     ← READ ONLY (copy init pattern)
│   └── logging/
│       └── DataLogger.cpp              ← READ ONLY (copy SD patterns)
└── test/
    └── espnow_rtt/                     ← NEW
        ├── sender_main.cpp             ← NEW (V001 firmware)
        └── reflector_main.cpp          ← NEW (V002 firmware)
```

---

## Sender Firmware Specification

### File: `test/espnow_rtt/sender_main.cpp`

### Purpose

- Send RTT packets at 10 Hz
- Wait for echo from reflector
- Calculate RTT = T2 - T1
- Log results to SD card with GPS position (for distance correlation)
- **Read IMU data (REQUIRED - for sensor noise characterization)**

### IMPORTANT: Reuse Existing Drivers

**DO NOT rewrite drivers from scratch!** Use the existing project drivers:

```cpp
// Include the existing MPU6500 driver
#include "sensors/imu/MPU6500Driver.h"

// In setup():
MPU6500Driver imu;
if (!imu.begin()) {
    Serial.println("ERROR: IMU init failed!");
}

// In loop():
auto imuData = imu.read();
float accel_x = imuData.accel[0];
float accel_y = imuData.accel[1];
float accel_z = imuData.accel[2];
```

**Note:** Since this is in `test/espnow_rtt/`, you may need to adjust include paths or copy the driver. The simplest approach is to add to `build_src_filter` in platformio.ini to include the driver source files.

### Required Includes

```cpp
#include <Arduino.h>
#include <WiFi.h>
#include <esp_now.h>
#include <esp_wifi.h>
#include <SdFat.h>
#include <TinyGPSPlus.h>
#include <Wire.h>

// Use config.h for pin definitions
// Note: Cannot include directly from test/, copy needed constants
```

### Pin Definitions (Copy from config.h)

```cpp
// SD Card (SPI)
#define SD_CS_PIN         5
#define SD_SPI_MHZ        10    // IMPORTANT: 10 MHz, not 25!

// GPS (UART)
#define GPS_RX_PIN        16
#define GPS_TX_PIN        17
#define GPS_BAUD_RATE     9600

// I2C (for MPU6500 - optional)
#define I2C_SDA_PIN       21
#define I2C_SCL_PIN       22
#define MPU6500_ADDR      0x68

// Status LED
#define LED_STATUS_PIN    2

// Button (start/stop)
#define BUTTON_PIN        4
```

### RTT Packet Structure

**CRITICAL: Include IMU data for sensor noise characterization!**

```cpp
#pragma pack(push, 1)
struct RTTPacket {
    // Network characterization fields
    uint32_t sequence;          // Packet sequence number (0, 1, 2, ...)  - 4 bytes
    uint32_t send_time_ms;      // Sender's millis() at send time        - 4 bytes

    // GPS fields (for distance correlation)
    float sender_lat;           // GPS latitude                          - 4 bytes
    float sender_lon;           // GPS longitude                         - 4 bytes
    float sender_speed;         // GPS speed (m/s)                       - 4 bytes
    float sender_heading;       // GPS heading (degrees)                 - 4 bytes

    // IMU fields (REQUIRED for sensor noise characterization)
    float accel_x;              // Accelerometer X (m/s²)                - 4 bytes
    float accel_y;              // Accelerometer Y (m/s²)                - 4 bytes
    float accel_z;              // Accelerometer Z (m/s²)                - 4 bytes

    // Padding to match V2VMessage size (90 bytes)
    uint8_t padding[54];        // 90 - 36 = 54 bytes
};
#pragma pack(pop)

// Size verification:
// 4 + 4 + 4 + 4 + 4 + 4 + 4 + 4 + 4 + 54 = 90 bytes ✓
static_assert(sizeof(RTTPacket) == 90, "RTTPacket must be 90 bytes");
```

**Why IMU Data is REQUIRED:**
- Stationary/home tests are the PERFECT opportunity to measure IMU noise floor
- The emulator's `SensorSynthesizer` needs real noise parameters
- Without this, we'd have to guess noise values from datasheets (inaccurate)

### State Variables

```cpp
// RTT tracking
struct RTTRecord {
    uint32_t sequence;
    uint32_t send_time_ms;
    uint32_t recv_time_ms;      // 0 if not received (lost)
    float lat, lon, speed;      // GPS at send time
    float heading;              // GPS heading at send time
    float accel_x, accel_y, accel_z;  // IMU at send time (REQUIRED!)
    bool received;
};

// Use circular buffer or simple array
static const int MAX_PENDING = 100;  // Track up to 100 pending packets
RTTRecord pendingPackets[MAX_PENDING];

// Statistics
uint32_t packetsSent = 0;
uint32_t packetsReceived = 0;
uint32_t currentSequence = 0;

// Timing
uint32_t lastSendTime = 0;
const uint32_t SEND_INTERVAL_MS = 100;  // 10 Hz

// State
bool isLogging = false;

// Drivers (reuse existing!)
MPU6500Driver imu;
TinyGPSPlus gps;
```

### ESP-NOW Receive Callback (For Echo)

```cpp
// Called when we receive the echoed packet back
void onDataRecv(const uint8_t* mac, const uint8_t* data, int len) {
    if (len != sizeof(RTTPacket)) return;

    uint32_t recvTime = millis();
    const RTTPacket* pkt = (const RTTPacket*)data;

    // Find matching pending packet by sequence number
    int idx = pkt->sequence % MAX_PENDING;
    if (pendingPackets[idx].sequence == pkt->sequence && !pendingPackets[idx].received) {
        pendingPackets[idx].recv_time_ms = recvTime;
        pendingPackets[idx].received = true;
        packetsReceived++;
    }
}
```

### Main Loop Logic

```cpp
void loop() {
    uint32_t now = millis();

    // Update GPS
    while (Serial1.available()) {
        gps.encode(Serial1.read());
    }

    // Handle button (start/stop logging)
    handleButton();

    // Update LED
    updateLED();

    if (!isLogging) return;

    // Send packet every 100ms (10 Hz)
    if (now - lastSendTime >= SEND_INTERVAL_MS) {
        lastSendTime = now;

        // Read IMU data (REQUIRED!)
        auto imuData = imu.read();

        // Build packet
        RTTPacket pkt;
        pkt.sequence = currentSequence;
        pkt.send_time_ms = now;
        pkt.sender_lat = gps.location.isValid() ? gps.location.lat() : 0.0f;
        pkt.sender_lon = gps.location.isValid() ? gps.location.lng() : 0.0f;
        pkt.sender_speed = gps.speed.isValid() ? gps.speed.mps() : 0.0f;
        pkt.sender_heading = gps.course.isValid() ? gps.course.deg() : 0.0f;
        pkt.accel_x = imuData.accel[0];
        pkt.accel_y = imuData.accel[1];
        pkt.accel_z = imuData.accel[2];
        memset(pkt.padding, 0xAA, sizeof(pkt.padding));

        // Store in pending buffer
        int idx = currentSequence % MAX_PENDING;
        pendingPackets[idx].sequence = currentSequence;
        pendingPackets[idx].send_time_ms = now;
        pendingPackets[idx].lat = pkt.sender_lat;
        pendingPackets[idx].lon = pkt.sender_lon;
        pendingPackets[idx].speed = pkt.sender_speed;
        pendingPackets[idx].heading = pkt.sender_heading;
        pendingPackets[idx].accel_x = pkt.accel_x;
        pendingPackets[idx].accel_y = pkt.accel_y;
        pendingPackets[idx].accel_z = pkt.accel_z;
        pendingPackets[idx].received = false;
        pendingPackets[idx].recv_time_ms = 0;

        // Send via ESP-NOW (broadcast)
        uint8_t broadcast[6] = {0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF};
        esp_now_send(broadcast, (uint8_t*)&pkt, sizeof(pkt));

        packetsSent++;
        currentSequence++;

        // Write completed records to SD (older packets that have been echoed or timed out)
        writeCompletedRecords();
    }

    // Print progress every second
    if (now - lastPrintTime >= 1000) {
        printProgress();
        lastPrintTime = now;
    }
}
```

### Writing to SD Card

```cpp
void writeCompletedRecords() {
    // Check packets from ~1 second ago (allow time for echo)
    uint32_t now = millis();
    const uint32_t TIMEOUT_MS = 500;  // Wait up to 500ms for echo

    for (int i = 0; i < MAX_PENDING; i++) {
        RTTRecord& rec = pendingPackets[i];

        // Skip empty slots
        if (rec.send_time_ms == 0) continue;

        // Skip if too recent (might still receive echo)
        if (now - rec.send_time_ms < TIMEOUT_MS) continue;

        // Write to SD (now includes IMU data!)
        char line[200];  // Increased buffer for IMU columns
        if (rec.received) {
            int rtt = rec.recv_time_ms - rec.send_time_ms;
            snprintf(line, sizeof(line),
                     "%u,%u,%u,%d,%.6f,%.6f,%.2f,%.1f,%.3f,%.3f,%.3f,0\n",
                     rec.sequence, rec.send_time_ms, rec.recv_time_ms, rtt,
                     rec.lat, rec.lon, rec.speed, rec.heading,
                     rec.accel_x, rec.accel_y, rec.accel_z);
        } else {
            // Lost packet: recv_time = -1, rtt = -1, lost = 1
            snprintf(line, sizeof(line),
                     "%u,%u,-1,-1,%.6f,%.6f,%.2f,%.1f,%.3f,%.3f,%.3f,1\n",
                     rec.sequence, rec.send_time_ms,
                     rec.lat, rec.lon, rec.speed, rec.heading,
                     rec.accel_x, rec.accel_y, rec.accel_z);
        }

        logFile.print(line);

        // Clear slot
        rec.send_time_ms = 0;
    }

    // Flush periodically
    logFile.flush();
}
```

### CSV Header (Write on Start)

```cpp
// Updated header with IMU columns!
const char* CSV_HEADER = "sequence,send_time_ms,recv_time_ms,rtt_ms,lat,lon,speed,heading,accel_x,accel_y,accel_z,lost\n";
```

---

## Reflector Firmware Specification

### File: `test/espnow_rtt/reflector_main.cpp`

### Purpose

**MINIMAL**: Receive packets, echo immediately. No SD card, no GPS, no processing.

### Why Minimal?

Any processing delay on the reflector adds to the RTT measurement. The reflector should:
1. Receive packet
2. Echo **immediately** (same bytes, no modification)
3. Nothing else

### Complete Implementation

```cpp
/**
 * @file reflector_main.cpp
 * @brief ESP-NOW RTT Reflector for RoadSense Characterization
 *
 * PURPOSE: Echo received packets immediately for RTT measurement.
 * CRITICAL: Do NOT add any processing - it increases measured RTT!
 */

#include <Arduino.h>
#include <WiFi.h>
#include <esp_now.h>
#include <esp_wifi.h>

// Statistics (for Serial output only)
volatile uint32_t packetsReceived = 0;
volatile uint32_t packetsEchoed = 0;

// Store sender MAC for reply
uint8_t senderMAC[6];
bool senderKnown = false;

// ESP-NOW receive callback - ECHO IMMEDIATELY
void onDataRecv(const uint8_t* mac, const uint8_t* data, int len) {
    packetsReceived++;

    // On first packet, register sender as peer
    if (!senderKnown) {
        memcpy(senderMAC, mac, 6);

        esp_now_peer_info_t peerInfo;
        memset(&peerInfo, 0, sizeof(peerInfo));
        memcpy(peerInfo.peer_addr, senderMAC, 6);
        peerInfo.channel = 0;
        peerInfo.encrypt = false;
        esp_now_add_peer(&peerInfo);

        senderKnown = true;

        Serial.printf("Sender registered: %02X:%02X:%02X:%02X:%02X:%02X\n",
                      mac[0], mac[1], mac[2], mac[3], mac[4], mac[5]);
    }

    // ECHO IMMEDIATELY - no processing, no delay!
    esp_err_t result = esp_now_send(senderMAC, data, len);
    if (result == ESP_OK) {
        packetsEchoed++;
    }
}

void onDataSent(const uint8_t* mac, esp_now_send_status_t status) {
    // Optional: track send failures
}

void setup() {
    Serial.begin(115200);
    delay(1000);

    Serial.println("========================================");
    Serial.println("  ESP-NOW RTT Reflector");
    Serial.println("  Role: Echo packets immediately");
    Serial.println("========================================");

    // Initialize WiFi (EXACT sequence from EspNowTransport.cpp)
    WiFi.mode(WIFI_STA);
    WiFi.disconnect();
    esp_wifi_set_channel(1, WIFI_SECOND_CHAN_NONE);

    // Initialize ESP-NOW
    if (esp_now_init() != ESP_OK) {
        Serial.println("ERROR: esp_now_init() failed!");
        while (1) delay(1000);
    }

    esp_now_register_recv_cb(onDataRecv);
    esp_now_register_send_cb(onDataSent);

    // Print MAC address
    uint8_t mac[6];
    WiFi.macAddress(mac);
    Serial.printf("Reflector MAC: %02X:%02X:%02X:%02X:%02X:%02X\n",
                  mac[0], mac[1], mac[2], mac[3], mac[4], mac[5]);

    Serial.println("Waiting for packets...");
}

void loop() {
    // Print stats every second
    static uint32_t lastPrint = 0;
    if (millis() - lastPrint >= 1000) {
        Serial.printf("Received: %u, Echoed: %u\n", packetsReceived, packetsEchoed);
        lastPrint = millis();
    }

    delay(10);  // Small delay to prevent watchdog
}
```

**That's it!** The reflector is intentionally simple.

---

## PlatformIO Configuration

### Add to `platformio.ini`

```ini
; ============================================================================
; ESP-NOW RTT Characterization Environments
; ============================================================================

[env:rtt_sender]
platform = espressif32
board = esp32dev
framework = arduino
monitor_speed = 115200

build_flags =
    -DVEHICLE_ID=\"V001\"
    -DRTT_MODE_SENDER=1
    -Isrc  ; Allow includes from src/ directory

; Include sender firmware + required drivers
build_src_filter =
    -<*>
    +<../test/espnow_rtt/sender_main.cpp>
    +<sensors/imu/MPU6500Driver.cpp>    ; REQUIRED: IMU driver
    +<utils/Logger.cpp>                  ; Required by MPU6500Driver

lib_deps =
    greiman/SdFat@^2.3.1
    mikalhart/TinyGPSPlus@^1.0.3


[env:rtt_reflector]
platform = espressif32
board = esp32dev
framework = arduino
monitor_speed = 115200

build_flags =
    -DVEHICLE_ID=\"V002\"
    -DRTT_MODE_REFLECTOR=1

; IMPORTANT: Only compile the reflector firmware, not main.cpp
build_src_filter =
    -<*>
    +<../test/espnow_rtt/reflector_main.cpp>

; No lib_deps needed - reflector doesn't use SD or GPS
```

---

## RTT Packet Protocol

### Packet Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          RTT MEASUREMENT PROTOCOL                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  SENDER (V001)                              REFLECTOR (V002)                 │
│                                                                              │
│  1. T1 = millis()                                                           │
│  2. Build RTTPacket:                                                        │
│     - sequence = N                                                          │
│     - send_time_ms = T1                                                     │
│     - lat, lon from GPS                                                     │
│     - padding to 90 bytes                                                   │
│                                                                              │
│  3. Send via ESP-NOW                                                        │
│     ─────────────────────────────────────────────────────→                  │
│                                              4. Receive packet              │
│                                              5. Echo IMMEDIATELY            │
│     ←─────────────────────────────────────────────────────                  │
│                                                 (same bytes)                │
│                                                                              │
│  6. T2 = millis()                                                           │
│  7. RTT = T2 - T1                                                           │
│  8. One-way latency ≈ RTT / 2                                               │
│  9. Log: seq, T1, T2, RTT, lat, lon, speed, lost=0                          │
│                                                                              │
│  (If no echo after 500ms: log lost=1)                                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Why 90 Bytes?

The existing V2VMessage struct is 90 bytes. To get realistic ESP-NOW performance measurements, the RTT packet should be the same size.

---

## SD Card Logging Format

### Output File: `/rtt_data.csv`

```csv
sequence,send_time_ms,recv_time_ms,rtt_ms,lat,lon,speed,heading,accel_x,accel_y,accel_z,lost
0,1000,1007,7,32.085123,34.781234,0.00,0.0,-0.05,0.12,9.81,0
1,1100,1108,8,32.085123,34.781234,0.00,0.0,-0.08,0.10,9.79,0
2,1200,-1,-1,32.085123,34.781234,0.00,0.0,-0.03,0.11,9.82,1
3,1300,1312,12,32.085125,34.781236,1.50,45.2,-0.06,0.09,9.80,0
...
```

### Column Descriptions

| Column | Type | Description |
|--------|------|-------------|
| `sequence` | uint32 | Packet sequence number (0, 1, 2, ...) |
| `send_time_ms` | uint32 | V001's millis() when packet sent |
| `recv_time_ms` | int32 | V001's millis() when echo received (-1 if lost) |
| `rtt_ms` | int32 | Round-trip time in ms (-1 if lost) |
| `lat` | float | GPS latitude at send time (0.0 if no fix) |
| `lon` | float | GPS longitude at send time (0.0 if no fix) |
| `speed` | float | GPS speed in m/s at send time |
| `heading` | float | GPS heading in degrees (0.0 if no fix) |
| `accel_x` | float | **IMU X-axis acceleration (m/s²) - for noise analysis** |
| `accel_y` | float | **IMU Y-axis acceleration (m/s²) - for noise analysis** |
| `accel_z` | float | **IMU Z-axis acceleration (m/s²) - should be ~9.81 when stationary** |
| `lost` | int | 1 if packet lost, 0 if received |

### Why IMU Data Matters for Analysis

During a **stationary/home test**:
- `accel_z` should be ~9.81 m/s² (gravity)
- The **standard deviation** of accel_x, accel_y, accel_z = **sensor noise floor**
- This feeds directly into the emulator's `SensorSynthesizer` for realistic training

---

## Build and Flash Instructions

### Prerequisites

1. PlatformIO installed (VS Code extension or CLI)
2. Two ESP32 DevKit boards
3. SD card inserted in V001 (FAT32 formatted)
4. USB cables for both boards

### Step 1: Build Sender Firmware

```bash
cd roadsense-v2v/hardware

# Build sender (for V001)
pio run -e rtt_sender
```

### Step 2: Flash Sender to V001

```bash
# Linux
pio run -e rtt_sender -t upload --upload-port /dev/ttyUSB0

# Windows
pio run -e rtt_sender -t upload --upload-port COM3
```

### Step 3: Build Reflector Firmware

```bash
# Build reflector (for V002)
pio run -e rtt_reflector
```

### Step 4: Flash Reflector to V002

```bash
# Flash to the OTHER ESP32
pio run -e rtt_reflector -t upload --upload-port /dev/ttyUSB1  # or COM4
```

### Step 5: Monitor Serial Output

```bash
# Terminal 1: V001 (Sender)
pio device monitor --port /dev/ttyUSB0 --baud 115200

# Terminal 2: V002 (Reflector)
pio device monitor --port /dev/ttyUSB1 --baud 115200
```

---

## Test Procedure

### Bench Test (Indoor, No GPS)

1. Place both ESP32s on desk, ~1 meter apart
2. Power both via USB
3. Open Serial monitors for both
4. **V002 (Reflector)** should show: `"Waiting for packets..."`
5. **V001 (Sender)** should show: `"Press button to start..."`
6. Press button on V001 (GPIO 4)
7. Watch for:
   - V001: `"Sent: 100, Received: 98, Loss: 2.0%, Avg RTT: 8ms"`
   - V002: `"Received: 100, Echoed: 100"`
8. After ~60 seconds (600 packets), press button again to stop
9. Check SD card for `rtt_data.csv`

### Expected Results (Bench Test)

| Metric | Expected Value |
|--------|----------------|
| Packet Loss | < 1% |
| RTT Min | 2-5 ms |
| RTT Max | 10-20 ms |
| RTT Mean | 5-10 ms |
| One-way Latency | 2.5-5 ms |

### Real Drive Test (Outdoor, With GPS)

1. Follow bench test setup
2. Mount V001 in one vehicle, V002 in another
3. Ensure GPS has fix before starting (wait 1-2 min outdoors)
4. Press button to start logging
5. Drive for 20-30 minutes with varying distances
6. Press button to stop
7. Retrieve SD card from V001

---

## Expected Output

### Serial Output - Sender (V001)

```
========================================
  ESP-NOW RTT Characterization (Sender)
  Vehicle: V001
========================================
SD Card: OK (10 MHz SPI)
GPS: Initializing...
ESP-NOW: OK
MAC: 24:6F:28:XX:XX:XX

Press button (GPIO 4) to start logging...

[LOGGING STARTED] Session: 001
GPS Fix: NO (waiting...)
Sent: 100, Received: 98, Loss: 2.0%, Avg RTT: 8.2ms
GPS Fix: YES (Sats: 7)
Sent: 200, Received: 196, Loss: 2.0%, Avg RTT: 7.8ms
...
Sent: 6000, Received: 5880, Loss: 2.0%, Avg RTT: 8.5ms

[LOGGING STOPPED]
Total: 6000 sent, 5880 received
Loss Rate: 2.0%
RTT: min=3ms, max=45ms, avg=8.5ms
Data saved: /rtt_data.csv
```

### Serial Output - Reflector (V002)

```
========================================
  ESP-NOW RTT Reflector
  Role: Echo packets immediately
========================================
MAC: 30:AE:A4:XX:XX:XX
Waiting for packets...

Sender registered: 24:6F:28:XX:XX:XX
Received: 100, Echoed: 100
Received: 200, Echoed: 200
...
Received: 6000, Echoed: 6000
```

---

## Common Pitfalls to Avoid

### 1. Wrong ESP-NOW Init Sequence

**WRONG:**
```cpp
esp_now_init();
WiFi.mode(WIFI_STA);  // Too late!
```

**CORRECT:**
```cpp
WiFi.mode(WIFI_STA);
WiFi.disconnect();
esp_wifi_set_channel(1, WIFI_SECOND_CHAN_NONE);
esp_now_init();
```

### 2. SD Card Speed Too High

**WRONG:**
```cpp
sd.begin(SD_CS_PIN, SD_SCK_MHZ(25));  // May fail on some cards
```

**CORRECT:**
```cpp
sd.begin(SD_CS_PIN, SD_SCK_MHZ(10));  // Reliable
```

### 3. Processing in Reflector

**WRONG:**
```cpp
void onDataRecv(...) {
    Serial.println("Received!");  // ADDS DELAY
    processPacket(data);          // ADDS DELAY
    esp_now_send(...);
}
```

**CORRECT:**
```cpp
void onDataRecv(...) {
    esp_now_send(...);  // ECHO FIRST!
    packetsReceived++;  // Stats after
}
```

### 4. Not Waiting for Echo Timeout

**WRONG:**
```cpp
// Write to SD immediately after sending
sendPacket();
writeToSD();  // Packet might still be in flight!
```

**CORRECT:**
```cpp
// Wait 500ms before marking as lost
if (now - record.send_time > 500 && !record.received) {
    record.lost = true;
    writeToSD(record);
}
```

### 5. Forgetting to Flush SD Card

**WRONG:**
```cpp
logFile.print(line);  // Data in buffer, not on disk!
```

**CORRECT:**
```cpp
logFile.print(line);
logFile.flush();  // Or flush every N rows
```

---

## Summary

### Files to Create

| File | Purpose |
|------|---------|
| `test/espnow_rtt/sender_main.cpp` | V001 firmware - sends, measures, logs |
| `test/espnow_rtt/reflector_main.cpp` | V002 firmware - echoes immediately |
| Update `platformio.ini` | Add `rtt_sender` and `rtt_reflector` environments |

### Key Requirements

1. **RTT measurement on ONE device** - V001 records T1 and T2
2. **Reflector echoes immediately** - no processing delay
3. **90-byte packets** - matches V2VMessage size
4. **10 MHz SD SPI** - reliable writes
5. **Button start/stop** - GPIO 4 with debounce
6. **GPS logging** - for distance correlation (optional if no fix)

### After Implementation

1. Build and flash both firmwares
2. Run bench test (60 seconds, 1m apart)
3. Verify RTT values (5-15ms expected)
4. Run real drive test (20-30 minutes, varying distances)
5. Process data with `analyze_rtt_characterization.py` script
6. Generate `emulator_params.json` for SUMO emulator

---

## Important Clarification: ONE Recording Session Only

**You only need ONE recording with the RTT firmware.** Do NOT also record with main.cpp!

```
❌ WRONG APPROACH:
   Session 1: RTT firmware (sender + reflector)
   Session 2: main.cpp (bidirectional broadcast)  ← USELESS! Clock sync problem!

✅ CORRECT APPROACH:
   Session 1: RTT firmware (sender + reflector)   ← This gives EVERYTHING you need!
```

**The RTT firmware provides ALL characterization data:**

| Data Type | How RTT Firmware Provides It |
|-----------|------------------------------|
| One-way latency | RTT / 2 (accurate, no clock sync needed!) |
| Packet loss rate | Missing sequence numbers |
| Jitter | std(RTT) |
| Distance correlation | GPS lat/lon (outdoor drive) |
| IMU noise floor | std(accel_x), std(accel_y), std(accel_z) |

**The main.cpp bidirectional broadcast CANNOT provide accurate latency** due to the clock sync problem (~9 second drift observed!). There is NO reason to record with it for characterization purposes.

---

**Document Version:** 1.1
**Created:** December 29, 2025
**Updated:** December 29, 2025 - Fixed padding bug, made IMU REQUIRED, added driver reuse instructions
**Status:** Ready for Implementation
