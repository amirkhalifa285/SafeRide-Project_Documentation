# ESP-NOW Communication Emulator Design

**Date Created:** December 19, 2025
**Author:** Amir Khalifa
**Status:** Planning Phase
**Target Completion:** 2 weeks (January 2, 2026)

> **UPDATE (December 26, 2025):** Sensor synthesis (gyro/mag) is now **OPTIONAL** - not blocking RL training. The minimal RL observation uses only peer pos/speed/accel + message age + ego speed. See `docs/ARCHITECTURE_V2_MINIMAL_RL.md` for the updated architecture.

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [The Sim2Real Problem](#the-sim2real-problem)
3. [Phase 1: ESP-NOW Characterization Experiment](#phase-1-esp-now-characterization-experiment)
4. [Phase 2: Building the Emulator](#phase-2-building-the-emulator)
5. [Phase 3: SUMO Integration](#phase-3-sumo-integration)
6. [Phase 4: Validation & Augmentation](#phase-4-validation--augmentation)
7. [2-Week Implementation Timeline](#2-week-implementation-timeline)
8. [Risk Mitigation](#risk-mitigation)
9. [Success Criteria](#success-criteria)

---

## Executive Summary

### The Problem

SUMO provides perfect vehicle state data at every timestep. Real ESP-NOW communication introduces:
- **Latency** (10-80ms variable)
- **Packet loss** (2-10% depending on distance/conditions)
- **Jitter** (latency variance)
- **Message staleness** (old data due to delays)

Training an RL agent on perfect SUMO data will fail in the real world (**Sim2Real gap**).

### The Solution

Build a **Python ESP-NOW emulator** based on **measured real-world characteristics**:

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Training Pipeline                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────┐    ┌─────────────────────┐    ┌──────────────────┐   │
│  │   SUMO   │───→│  ESP-NOW Emulator   │───→│   RL Environment │   │
│  │  (FCD)   │    │  (Based on REAL     │    │  (Gym Interface) │   │
│  │          │    │   measurements)     │    │                  │   │
│  └──────────┘    └─────────────────────┘    └──────────────────┘   │
│                                                                      │
│  Perfect data         Adds realistic:          Agent learns         │
│  at 10 Hz             - Measured latency       robust policy        │
│                       - Measured loss                               │
│                       - Domain randomization                        │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Principle: Measure First, Emulate Second

**DO NOT GUESS** ESP-NOW parameters. Measure them from real hardware, then build emulator to match.

---

## The Sim2Real Problem

### What Can Go Wrong

| If Emulator Has | But Real ESP-NOW Has | Result |
|-----------------|----------------------|--------|
| Gaussian latency (25ms mean) | Bimodal (15ms OR 80ms retransmit) | Model learns wrong timing |
| Independent 2% packet loss | Bursty loss (5 in a row) | Model underestimates danger |
| Linear loss vs distance | Cliff at 80m (sudden drop) | Model overconfident at range edge |
| Gaussian GPS noise ±5m | Multipath jumps of 20m | Model not robust to outliers |

### The Mitigation Strategy

1. **Characterize real ESP-NOW** (Phase 1) - Don't guess, measure
2. **Build emulator from measurements** (Phase 2) - Match real distributions
3. **Add domain randomization** (Phase 2) - Train WIDER than reality
4. **Validate iteratively** (Phase 4) - Test on real hardware frequently

---

## The Sensor Data Gap (Sim2Real Problem #2) - OPTIONAL

> **Note:** This section describes sensor synthesis which is now **optional/stretch goal**. The minimal RL observation does not require gyro/mag. Keep this for future ablation studies.

### Problem: SUMO Doesn't Provide All Sensor Data

SUMO simulation provides:
- ✅ Position (lat/lon from FCD output)
- ✅ Speed (m/s)
- ✅ Heading (degrees)
- ✅ Longitudinal acceleration (m/s²)

But V2VMessage struct (90 bytes) contains:
- ❌ `gyro[3]` - gyroscope readings (rad/s)
- ❌ `mag[3]` - magnetometer readings (µT)
- ❌ `accel[3]` - full 3-axis accelerometer (not just longitudinal)

**Impact**: If we train on SUMO data with zeros/guesses for gyro/mag, the RL agent won't learn to use these sensors. Real ESP32 units HAVE these sensors (MPU6500 + QMC5883L).

### Solution: Sensor Synthesis from SUMO + Measured Noise

**Key Insight**: We can DERIVE gyro/mag from SUMO kinematics, then add REAL noise measured from hardware.

```
┌───────────────────────────────────────────────────────────────────────────────┐
│                    SENSOR SYNTHESIS PIPELINE                                    │
├───────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  SUMO FCD Output              Sensor Synthesizer              RL Agent Input    │
│  ─────────────────           ──────────────────               ──────────────    │
│                                                                                 │
│  speed, heading          →   gyro_z = d(heading)/dt          →  gyro[3]        │
│  accel_longitudinal      →   accel_x = accel_long            →  accel[3]       │
│  position, speed         →   accel_y = v²/R (centripetal)    →                 │
│  heading                 →   mag = f(heading, declination)   →  mag[3]         │
│                                                                                 │
│                              + REAL noise from characterization                 │
│                              (measured during RTT drive with IMU/mag)           │
│                                                                                 │
└───────────────────────────────────────────────────────────────────────────────┘
```

### Physics-Based Derivation

| Sensor | Derivation from SUMO | Notes |
|--------|----------------------|-------|
| `gyro_z` | `d(heading)/dt` | Yaw rate = heading change per timestep |
| `gyro_x`, `gyro_y` | ~0 (flat road assumption) | Roll/pitch minimal on roads |
| `accel_x` | `longitudinal_accel` from SUMO | Already provided by FCD |
| `accel_y` | `v²/R` where R = turn radius | Centripetal acceleration on curves |
| `accel_z` | ~9.81 m/s² | Gravity (constant for flat roads) |
| `mag_x`, `mag_y` | `f(heading, intensity)` | Heading → magnetic field components |
| `mag_z` | ~local_vertical_intensity | Roughly constant (~40-50 µT) |

### Why Measure Real Noise?

**Don't guess noise from datasheets!** Measure it during characterization.

During the RTT characterization drive, V001 logs:
1. GPS heading (from NEO-6M)
2. Real gyro_z (from MPU6500)
3. Real mag_x, mag_y, mag_z (from QMC5883L)

Post-processing compares:
- `expected_gyro_z = d(gps_heading)/dt`
- `actual_gyro_z = MPU6500 reading`
- `noise = actual - expected`

This gives us REAL noise distribution, not guessed values.

### SensorSynthesizer Class (Added to Emulator)

```python
class SensorSynthesizer:
    """
    Synthesizes gyro/mag/accel from SUMO kinematics + measured noise.
    """

    def __init__(self, noise_params_file: str = None):
        """
        Args:
            noise_params_file: Path to sensor_noise_params.json from characterization
        """
        if noise_params_file:
            with open(noise_params_file, 'r') as f:
                self.noise_params = json.load(f)
        else:
            # Defaults (will be replaced by measurements)
            self.noise_params = {
                'gyro_z_std_rad_s': 0.01,
                'mag_xy_std_ut': 5.0,
                'accel_std_ms2': 0.2,
            }

        self.prev_heading = None
        self.prev_time = None

    def synthesize(self, sumo_state: dict, current_time_s: float) -> dict:
        """
        Generate realistic sensor values from SUMO state.

        Args:
            sumo_state: {speed, heading, accel, lat, lon}
            current_time_s: Current simulation time

        Returns:
            {gyro: [x,y,z], accel: [x,y,z], mag: [x,y,z]}
        """
        sensors = {}

        # Gyroscope synthesis
        if self.prev_heading is not None and self.prev_time is not None:
            dt = current_time_s - self.prev_time
            if dt > 0:
                # heading change -> yaw rate (convert degrees to radians)
                dheading = self._angle_diff(sumo_state['heading'], self.prev_heading)
                gyro_z = math.radians(dheading) / dt
            else:
                gyro_z = 0.0
        else:
            gyro_z = 0.0

        self.prev_heading = sumo_state['heading']
        self.prev_time = current_time_s

        # Add measured noise
        sensors['gyro'] = [
            random.gauss(0, self.noise_params['gyro_z_std_rad_s'] * 0.1),  # x (roll)
            random.gauss(0, self.noise_params['gyro_z_std_rad_s'] * 0.1),  # y (pitch)
            gyro_z + random.gauss(0, self.noise_params['gyro_z_std_rad_s']),  # z (yaw)
        ]

        # Accelerometer synthesis
        accel_x = sumo_state.get('accel', 0.0)  # longitudinal
        # Centripetal: a_y = v²/R, approximated from turn rate
        accel_y = sumo_state['speed'] * sensors['gyro'][2]  # v * omega
        accel_z = 9.81  # gravity

        sensors['accel'] = [
            accel_x + random.gauss(0, self.noise_params['accel_std_ms2']),
            accel_y + random.gauss(0, self.noise_params['accel_std_ms2']),
            accel_z + random.gauss(0, self.noise_params['accel_std_ms2']),
        ]

        # Magnetometer synthesis (heading -> magnetic field)
        heading_rad = math.radians(sumo_state['heading'])
        mag_intensity = 50.0  # µT (Earth's field, approximate)

        sensors['mag'] = [
            mag_intensity * math.cos(heading_rad) + random.gauss(0, self.noise_params['mag_xy_std_ut']),
            mag_intensity * math.sin(heading_rad) + random.gauss(0, self.noise_params['mag_xy_std_ut']),
            40.0 + random.gauss(0, self.noise_params['mag_xy_std_ut'] * 0.5),  # vertical
        ]

        return sensors

    def _angle_diff(self, a1: float, a2: float) -> float:
        """Calculate angle difference handling wraparound"""
        diff = a1 - a2
        while diff > 180:
            diff -= 360
        while diff < -180:
            diff += 360
        return diff
```

### Updated Training Pipeline

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    COMPLETE TRAINING PIPELINE (Updated)                          │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌──────────┐    ┌─────────────────────┐    ┌─────────────────────┐             │
│  │   SUMO   │───→│  ESP-NOW Emulator   │───→│  SensorSynthesizer  │             │
│  │  (FCD)   │    │  (latency, loss)    │    │  (gyro, mag, accel) │             │
│  │          │    │                     │    │  + measured noise   │             │
│  └──────────┘    └─────────────────────┘    └─────────────────────┘             │
│       │                   │                            │                         │
│       │         Adds communication effects    Adds sensor realism               │
│       │         from RTT measurement          from IMU/mag measurement          │
│       │                   │                            │                         │
│       └───────────────────┴────────────────────────────┘                        │
│                                    │                                             │
│                                    ▼                                             │
│                          ┌──────────────────┐                                   │
│                          │   RL Environment │                                   │
│                          │  (Gym Interface) │                                   │
│                          │                  │                                   │
│                          │  Observation:    │                                   │
│                          │  - pos, speed    │                                   │
│                          │  - gyro[3] ✓     │                                   │
│                          │  - mag[3] ✓      │                                   │
│                          │  - accel[3] ✓    │                                   │
│                          │  - age_ms        │                                   │
│                          └──────────────────┘                                   │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Implementation Effort

| Task | Effort | Notes |
|------|--------|-------|
| Add IMU/mag logging to RTT sender | ~30 lines | Extend existing firmware |
| Add sensor analysis to Python script | ~80 lines | Extract noise parameters |
| Add SensorSynthesizer to emulator | ~100 lines | Class above |
| **Total** | **~1 day** | Not weeks! |

---

## Phase 1: ESP-NOW Characterization Experiment

### Objective

Measure actual ESP-NOW behavior AND real sensor noise with 2 ESP32 boards to establish ground truth for emulator.

### Equipment Needed

- 2 ESP32 boards (V001, V002) - Already have
- SD cards in both boards - Already have
- Measuring tape (for distance tests)
- Open outdoor area (parking lot, field)
- ~2 hours of testing time

### The Clock Synchronization Problem

**Challenge:** To measure one-way latency, both boards need synchronized clocks.
**ESP32 Problem:** Internal clocks drift ~20ppm (milliseconds per minute).

**Solution: Round-Trip Time (RTT) Measurement**

```
V001 (Sender)                    V002 (Reflector)
     │                                │
     │──── Packet (seq=1, t1) ───────→│
     │                                │ (immediate echo)
     │←─── Echo (seq=1, t1) ──────────│
     │                                │
     │ RTT = t2 - t1                  │
     │ One-way ≈ RTT / 2              │
```

No clock sync needed - V001 measures its own timestamps.

### Firmware: V001 (Sender + RTT Calculator)

```cpp
// File: test/espnow_characterization/sender_rtt.cpp

#include <Arduino.h>
#include <esp_now.h>
#include <WiFi.h>
#include <SD.h>
#include <SPI.h>

// SD Card pins
#define SD_CS 5

// Test configuration
#define PACKETS_PER_BATCH 1000
#define PACKET_INTERVAL_MS 100  // 10 Hz like real system
#define LOG_FILE "/espnow_rtt.csv"

// Packet structure (matches V2VMessage size roughly)
struct TestPacket {
    uint32_t sequence;
    uint32_t send_time_ms;
    uint8_t padding[82];  // Pad to ~90 bytes like real V2VMessage
};

// RTT tracking
struct RTTRecord {
    uint32_t sequence;
    uint32_t send_time;
    uint32_t receive_time;
    bool received;
};

RTTRecord rtt_records[PACKETS_PER_BATCH];
volatile uint32_t echo_sequence = 0;
volatile uint32_t echo_receive_time = 0;
volatile bool echo_received = false;

// Peer MAC (V002) - UPDATE THIS
uint8_t peerMAC[] = {0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF};  // Broadcast for now

File logFile;
uint32_t current_sequence = 0;
uint32_t packets_sent = 0;
uint32_t packets_received = 0;

// ESP-NOW receive callback (for echoes)
void onDataRecv(const esp_now_recv_info_t *info, const uint8_t *data, int len) {
    if (len >= sizeof(TestPacket)) {
        TestPacket* pkt = (TestPacket*)data;
        echo_sequence = pkt->sequence;
        echo_receive_time = millis();
        echo_received = true;
    }
}

// ESP-NOW send callback
void onDataSent(const uint8_t *mac_addr, esp_now_send_status_t status) {
    // Optional: track send failures
}

void setup() {
    Serial.begin(115200);
    delay(1000);

    Serial.println("=== ESP-NOW RTT Characterization (Sender) ===");

    // Initialize SD card
    SPI.begin(18, 19, 23, SD_CS);
    if (!SD.begin(SD_CS)) {
        Serial.println("ERROR: SD card init failed!");
        while(1);
    }
    Serial.println("SD card initialized");

    // Create/open log file
    logFile = SD.open(LOG_FILE, FILE_WRITE);
    if (!logFile) {
        Serial.println("ERROR: Cannot create log file!");
        while(1);
    }

    // Write CSV header
    logFile.println("sequence,send_time_ms,receive_time_ms,rtt_ms,lost");
    logFile.flush();

    // Initialize WiFi in station mode
    WiFi.mode(WIFI_STA);
    Serial.print("Sender MAC: ");
    Serial.println(WiFi.macAddress());

    // Initialize ESP-NOW
    if (esp_now_init() != ESP_OK) {
        Serial.println("ERROR: ESP-NOW init failed!");
        while(1);
    }

    // Register callbacks
    esp_now_register_recv_cb(onDataRecv);
    esp_now_register_send_cb(onDataSent);

    // Add broadcast peer
    esp_now_peer_info_t peerInfo = {};
    memcpy(peerInfo.peer_addr, peerMAC, 6);
    peerInfo.channel = 0;
    peerInfo.encrypt = false;

    if (esp_now_add_peer(&peerInfo) != ESP_OK) {
        Serial.println("ERROR: Failed to add peer");
        while(1);
    }

    Serial.println("Setup complete. Starting RTT measurement...");
    Serial.println("Press button or wait 10 seconds to start...");
    delay(10000);

    Serial.println("STARTING MEASUREMENT!");
}

void loop() {
    static uint32_t last_send = 0;
    static uint32_t batch_start = 0;
    static bool batch_complete = false;

    if (batch_complete) {
        // All done - just idle
        delay(1000);
        return;
    }

    uint32_t now = millis();

    // Send packet every PACKET_INTERVAL_MS
    if (now - last_send >= PACKET_INTERVAL_MS) {
        last_send = now;

        // Prepare packet
        TestPacket pkt;
        pkt.sequence = current_sequence;
        pkt.send_time_ms = now;
        memset(pkt.padding, 0xAA, sizeof(pkt.padding));

        // Record send time
        rtt_records[current_sequence % PACKETS_PER_BATCH].sequence = current_sequence;
        rtt_records[current_sequence % PACKETS_PER_BATCH].send_time = now;
        rtt_records[current_sequence % PACKETS_PER_BATCH].received = false;

        // Send packet
        esp_err_t result = esp_now_send(peerMAC, (uint8_t*)&pkt, sizeof(pkt));

        if (result == ESP_OK) {
            packets_sent++;
        }

        current_sequence++;

        // Progress indicator
        if (current_sequence % 100 == 0) {
            Serial.printf("Sent: %u, Received: %u, Loss: %.1f%%\n",
                packets_sent, packets_received,
                100.0 * (packets_sent - packets_received) / packets_sent);
        }

        // Check if batch complete
        if (current_sequence >= PACKETS_PER_BATCH) {
            Serial.println("Batch complete! Waiting for final echoes...");
            delay(1000);  // Wait for stragglers

            // Write all records to SD
            Serial.println("Writing results to SD card...");
            for (int i = 0; i < PACKETS_PER_BATCH; i++) {
                logFile.printf("%u,%u,%u,%d,%d\n",
                    rtt_records[i].sequence,
                    rtt_records[i].send_time,
                    rtt_records[i].receive_time,
                    rtt_records[i].received ? (int)(rtt_records[i].receive_time - rtt_records[i].send_time) : -1,
                    rtt_records[i].received ? 0 : 1);
            }
            logFile.close();

            // Print summary
            Serial.println("\n=== RESULTS ===");
            Serial.printf("Packets sent: %u\n", packets_sent);
            Serial.printf("Packets received: %u\n", packets_received);
            Serial.printf("Packet loss: %.2f%%\n", 100.0 * (packets_sent - packets_received) / packets_sent);

            // Calculate RTT statistics
            uint32_t rtt_sum = 0;
            uint32_t rtt_count = 0;
            uint32_t rtt_min = UINT32_MAX;
            uint32_t rtt_max = 0;

            for (int i = 0; i < PACKETS_PER_BATCH; i++) {
                if (rtt_records[i].received) {
                    uint32_t rtt = rtt_records[i].receive_time - rtt_records[i].send_time;
                    rtt_sum += rtt;
                    rtt_count++;
                    if (rtt < rtt_min) rtt_min = rtt;
                    if (rtt > rtt_max) rtt_max = rtt;
                }
            }

            if (rtt_count > 0) {
                Serial.printf("RTT min: %u ms\n", rtt_min);
                Serial.printf("RTT max: %u ms\n", rtt_max);
                Serial.printf("RTT avg: %.1f ms\n", (float)rtt_sum / rtt_count);
                Serial.printf("One-way latency (est): %.1f ms\n", (float)rtt_sum / rtt_count / 2);
            }

            Serial.println("\nData saved to SD card: " LOG_FILE);
            Serial.println("EXPERIMENT COMPLETE!");

            batch_complete = true;
        }
    }

    // Check for echo responses
    if (echo_received) {
        echo_received = false;

        uint32_t idx = echo_sequence % PACKETS_PER_BATCH;
        if (!rtt_records[idx].received) {
            rtt_records[idx].receive_time = echo_receive_time;
            rtt_records[idx].received = true;
            packets_received++;
        }
    }
}
```

### Firmware: V002 (Reflector/Echo)

```cpp
// File: test/espnow_characterization/reflector_echo.cpp

#include <Arduino.h>
#include <esp_now.h>
#include <WiFi.h>

// Packet structure (must match sender)
struct TestPacket {
    uint32_t sequence;
    uint32_t send_time_ms;
    uint8_t padding[82];
};

uint32_t packets_received = 0;
uint32_t packets_echoed = 0;

// Store sender MAC for echo response
uint8_t senderMAC[6];
bool sender_known = false;

void onDataRecv(const esp_now_recv_info_t *info, const uint8_t *data, int len) {
    if (len >= sizeof(TestPacket)) {
        packets_received++;

        // Store sender MAC on first packet
        if (!sender_known) {
            memcpy(senderMAC, info->src_addr, 6);

            // Add sender as peer for echo
            esp_now_peer_info_t peerInfo = {};
            memcpy(peerInfo.peer_addr, senderMAC, 6);
            peerInfo.channel = 0;
            peerInfo.encrypt = false;
            esp_now_add_peer(&peerInfo);

            sender_known = true;
            Serial.printf("Sender registered: %02X:%02X:%02X:%02X:%02X:%02X\n",
                senderMAC[0], senderMAC[1], senderMAC[2],
                senderMAC[3], senderMAC[4], senderMAC[5]);
        }

        // Echo packet back immediately (don't modify - sender needs original timestamp)
        esp_err_t result = esp_now_send(senderMAC, data, len);
        if (result == ESP_OK) {
            packets_echoed++;
        }

        // Progress indicator
        if (packets_received % 100 == 0) {
            Serial.printf("Received: %u, Echoed: %u\n", packets_received, packets_echoed);
        }
    }
}

void setup() {
    Serial.begin(115200);
    delay(1000);

    Serial.println("=== ESP-NOW RTT Characterization (Reflector) ===");

    // Initialize WiFi in station mode
    WiFi.mode(WIFI_STA);
    Serial.print("Reflector MAC: ");
    Serial.println(WiFi.macAddress());

    // Initialize ESP-NOW
    if (esp_now_init() != ESP_OK) {
        Serial.println("ERROR: ESP-NOW init failed!");
        while(1);
    }

    // Register receive callback
    esp_now_register_recv_cb(onDataRecv);

    Serial.println("Reflector ready. Waiting for packets...");
}

void loop() {
    // Just wait for packets - all work done in callback
    delay(100);
}
```

### Test Procedure

#### Test 1: Close Range Baseline (5 minutes)

1. Place V001 and V002 **1 meter apart** on a table
2. Flash `sender_rtt.cpp` to V001
3. Flash `reflector_echo.cpp` to V002
4. Power both boards
5. Wait for V001 to complete (1000 packets @ 10Hz = ~100 seconds)
6. Retrieve SD card from V001, copy `espnow_rtt.csv`
7. Rename to `rtt_1m.csv`

**Expected results:**
- Packet loss: <1%
- RTT: 2-10ms
- One-way latency: 1-5ms

#### Test 2: Variable Distance (30 minutes)

Repeat Test 1 at each distance:
- 10 meters → `rtt_10m.csv`
- 30 meters → `rtt_30m.csv`
- 50 meters → `rtt_50m.csv`
- 80 meters → `rtt_80m.csv`
- 100 meters → `rtt_100m.csv`
- 120 meters → `rtt_120m.csv` (expect high loss)

**What we're measuring:**
- How latency changes with distance
- At what distance packet loss increases significantly
- The "cliff" point where communication degrades

#### Test 3: Interference Test (15 minutes)

At 30m distance:
- Test near WiFi router
- Test near Bluetooth devices
- Test with phone hotspot active nearby

**What we're measuring:**
- Impact of 2.4 GHz interference

### Data Analysis Script

```python
# File: scripts/analyze_espnow_characterization.py

import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from scipy import stats
import json

def analyze_rtt_file(filepath, distance_m):
    """Analyze a single RTT measurement file"""
    df = pd.read_csv(filepath)

    # Separate received and lost packets
    received = df[df['lost'] == 0]
    lost = df[df['lost'] == 1]

    results = {
        'distance_m': distance_m,
        'total_packets': len(df),
        'packets_received': len(received),
        'packets_lost': len(lost),
        'loss_rate': len(lost) / len(df),
    }

    if len(received) > 0:
        rtt_values = received['rtt_ms'].values
        results.update({
            'rtt_min': float(np.min(rtt_values)),
            'rtt_max': float(np.max(rtt_values)),
            'rtt_mean': float(np.mean(rtt_values)),
            'rtt_median': float(np.median(rtt_values)),
            'rtt_std': float(np.std(rtt_values)),
            'rtt_p95': float(np.percentile(rtt_values, 95)),
            'rtt_p99': float(np.percentile(rtt_values, 99)),
            'oneway_latency_est': float(np.mean(rtt_values) / 2),
        })

        # Check for bimodal distribution (indicates retransmissions)
        # Simple heuristic: if std > mean/2, likely bimodal
        results['possibly_bimodal'] = results['rtt_std'] > results['rtt_mean'] / 2

        # Fit distribution
        # Try normal first
        norm_params = stats.norm.fit(rtt_values)
        # Try exponential
        exp_params = stats.expon.fit(rtt_values)

        # KS test for goodness of fit
        _, norm_p = stats.kstest(rtt_values, 'norm', norm_params)
        _, exp_p = stats.kstest(rtt_values, 'expon', exp_params)

        results['best_fit'] = 'normal' if norm_p > exp_p else 'exponential'
        results['fit_params'] = {
            'normal': {'loc': norm_params[0], 'scale': norm_params[1]},
            'exponential': {'loc': exp_params[0], 'scale': exp_params[1]}
        }

    return results, df

def analyze_burst_loss(df):
    """Analyze packet loss patterns - check for bursty behavior"""
    lost = df['lost'].values

    # Find consecutive loss sequences
    burst_lengths = []
    current_burst = 0

    for l in lost:
        if l == 1:
            current_burst += 1
        else:
            if current_burst > 0:
                burst_lengths.append(current_burst)
            current_burst = 0

    if current_burst > 0:
        burst_lengths.append(current_burst)

    return {
        'num_loss_bursts': len(burst_lengths),
        'max_burst_length': max(burst_lengths) if burst_lengths else 0,
        'mean_burst_length': np.mean(burst_lengths) if burst_lengths else 0,
        'burst_lengths': burst_lengths
    }

def main():
    distances = [1, 10, 30, 50, 80, 100, 120]
    all_results = []

    for d in distances:
        filepath = f'data/characterization/rtt_{d}m.csv'
        try:
            results, df = analyze_rtt_file(filepath, d)
            burst_info = analyze_burst_loss(df)
            results.update(burst_info)
            all_results.append(results)

            print(f"\n=== Distance: {d}m ===")
            print(f"  Packets: {results['total_packets']}")
            print(f"  Loss rate: {results['loss_rate']*100:.2f}%")
            if 'rtt_mean' in results:
                print(f"  RTT: {results['rtt_mean']:.1f}ms (±{results['rtt_std']:.1f}ms)")
                print(f"  RTT range: {results['rtt_min']:.1f} - {results['rtt_max']:.1f}ms")
                print(f"  Est. one-way latency: {results['oneway_latency_est']:.1f}ms")
            if results['max_burst_length'] > 1:
                print(f"  WARNING: Bursty loss detected! Max burst: {results['max_burst_length']}")

        except FileNotFoundError:
            print(f"File not found: {filepath}")

    # Save results
    with open('data/characterization/analysis_results.json', 'w') as f:
        json.dump(all_results, f, indent=2)

    # Generate emulator parameters
    generate_emulator_params(all_results)

    # Plot results
    plot_results(all_results)

def generate_emulator_params(results):
    """Generate parameters for ESP-NOW emulator based on measurements"""

    params = {
        'latency': {
            'description': 'One-way latency parameters',
            'base_ms': None,
            'distance_factor': None,  # ms per meter
            'jitter_std_ms': None,
        },
        'packet_loss': {
            'description': 'Packet loss probability vs distance',
            'base_rate': None,
            'distance_threshold_1': None,
            'distance_threshold_2': None,
            'rate_tier_1': None,
            'rate_tier_2': None,
            'rate_tier_3': None,
        },
        'burst_loss': {
            'description': 'Burst loss parameters (if detected)',
            'enabled': False,
            'mean_burst_length': 1,
        },
        'domain_randomization': {
            'description': 'Wider ranges for training robustness',
            'latency_range_ms': None,
            'loss_rate_range': None,
        }
    }

    # Analyze latency vs distance
    latencies = [(r['distance_m'], r.get('oneway_latency_est', 0))
                 for r in results if 'oneway_latency_est' in r]

    if len(latencies) >= 2:
        distances = [l[0] for l in latencies]
        lats = [l[1] for l in latencies]

        # Linear regression for distance factor
        slope, intercept, _, _, _ = stats.linregress(distances, lats)

        params['latency']['base_ms'] = intercept
        params['latency']['distance_factor'] = slope

        # Jitter from std
        stds = [r.get('rtt_std', 0)/2 for r in results if 'rtt_std' in r]
        params['latency']['jitter_std_ms'] = np.mean(stds) if stds else 5

    # Analyze packet loss vs distance
    loss_rates = [(r['distance_m'], r['loss_rate']) for r in results]

    if loss_rates:
        # Find threshold distances
        params['packet_loss']['base_rate'] = min(r[1] for r in loss_rates)

        for d, rate in sorted(loss_rates):
            if rate > 0.05 and params['packet_loss']['distance_threshold_1'] is None:
                params['packet_loss']['distance_threshold_1'] = d
            if rate > 0.15 and params['packet_loss']['distance_threshold_2'] is None:
                params['packet_loss']['distance_threshold_2'] = d

    # Domain randomization (train wider than measured)
    if params['latency']['base_ms']:
        measured_max_latency = max(r.get('rtt_p99', 0)/2 for r in results if 'rtt_p99' in r)
        params['domain_randomization']['latency_range_ms'] = [
            max(1, params['latency']['base_ms'] - 5),
            measured_max_latency * 1.5  # 50% wider than measured
        ]

    measured_max_loss = max(r['loss_rate'] for r in results)
    params['domain_randomization']['loss_rate_range'] = [
        0.0,
        min(0.3, measured_max_loss * 2)  # Up to 2x measured, max 30%
    ]

    # Check for burst loss
    max_burst = max(r.get('max_burst_length', 0) for r in results)
    if max_burst > 2:
        params['burst_loss']['enabled'] = True
        params['burst_loss']['mean_burst_length'] = np.mean([
            r.get('mean_burst_length', 1) for r in results
        ])

    # Save parameters
    with open('data/characterization/emulator_params.json', 'w') as f:
        json.dump(params, f, indent=2)

    print("\n=== EMULATOR PARAMETERS GENERATED ===")
    print(json.dumps(params, indent=2))

    return params

def plot_results(results):
    """Generate visualization of characterization results"""
    fig, axes = plt.subplots(2, 2, figsize=(12, 10))

    distances = [r['distance_m'] for r in results]

    # Plot 1: Latency vs Distance
    ax1 = axes[0, 0]
    latencies = [r.get('oneway_latency_est', 0) for r in results]
    lat_stds = [r.get('rtt_std', 0)/2 for r in results]
    ax1.errorbar(distances, latencies, yerr=lat_stds, marker='o', capsize=5)
    ax1.set_xlabel('Distance (m)')
    ax1.set_ylabel('One-way Latency (ms)')
    ax1.set_title('ESP-NOW Latency vs Distance')
    ax1.grid(True, alpha=0.3)

    # Plot 2: Packet Loss vs Distance
    ax2 = axes[0, 1]
    loss_rates = [r['loss_rate'] * 100 for r in results]
    ax2.bar(distances, loss_rates, width=5)
    ax2.set_xlabel('Distance (m)')
    ax2.set_ylabel('Packet Loss (%)')
    ax2.set_title('ESP-NOW Packet Loss vs Distance')
    ax2.grid(True, alpha=0.3)

    # Plot 3: RTT Distribution (first file with data)
    ax3 = axes[1, 0]
    for r in results:
        if r['distance_m'] == 30:  # Use 30m as representative
            filepath = f'data/characterization/rtt_{r["distance_m"]}m.csv'
            try:
                df = pd.read_csv(filepath)
                received = df[df['lost'] == 0]['rtt_ms']
                ax3.hist(received, bins=50, alpha=0.7, edgecolor='black')
                ax3.axvline(r['rtt_mean'], color='r', linestyle='--', label=f'Mean: {r["rtt_mean"]:.1f}ms')
                ax3.axvline(r['rtt_p95'], color='orange', linestyle='--', label=f'P95: {r["rtt_p95"]:.1f}ms')
            except:
                pass
    ax3.set_xlabel('RTT (ms)')
    ax3.set_ylabel('Count')
    ax3.set_title('RTT Distribution at 30m')
    ax3.legend()
    ax3.grid(True, alpha=0.3)

    # Plot 4: Summary
    ax4 = axes[1, 1]
    ax4.axis('off')
    summary_text = "ESP-NOW Characterization Summary\n" + "="*40 + "\n\n"

    for r in results:
        summary_text += f"Distance: {r['distance_m']}m\n"
        summary_text += f"  Loss: {r['loss_rate']*100:.1f}%\n"
        if 'oneway_latency_est' in r:
            summary_text += f"  Latency: {r['oneway_latency_est']:.1f}ms\n"
        summary_text += "\n"

    ax4.text(0.1, 0.9, summary_text, transform=ax4.transAxes,
             fontfamily='monospace', fontsize=10, verticalalignment='top')

    plt.tight_layout()
    plt.savefig('data/characterization/espnow_characterization.png', dpi=150)
    plt.show()

    print("\nPlot saved to: data/characterization/espnow_characterization.png")

if __name__ == "__main__":
    main()
```

---

## Phase 2: Building the Emulator

### Based on Measured Parameters

After Phase 1, we'll have `emulator_params.json` with real measurements. The emulator uses these:

```python
# File: ml/espnow_emulator.py

import json
import random
import numpy as np
from dataclasses import dataclass
from typing import Optional, Dict, List
from collections import deque

@dataclass
class V2VMessage:
    """Simulated V2V message (matches ESP32 struct)"""
    vehicle_id: str
    lat: float
    lon: float
    speed: float
    heading: float
    accel_x: float
    accel_y: float
    accel_z: float
    gyro_x: float
    gyro_y: float
    gyro_z: float
    timestamp_ms: int

@dataclass
class ReceivedMessage:
    """Message as received by ego vehicle (with communication effects)"""
    message: V2VMessage
    age_ms: int
    received_at_ms: int

class ESPNOWEmulator:
    """
    Emulates ESP-NOW communication effects on perfect SUMO data.

    Uses measured parameters from real ESP32 characterization + domain randomization.
    """

    def __init__(self, params_file: str = None, domain_randomization: bool = True):
        """
        Initialize emulator with measured parameters.

        Args:
            params_file: Path to emulator_params.json from characterization
            domain_randomization: If True, randomize parameters for robust training
        """
        self.domain_randomization = domain_randomization

        # Load measured parameters or use defaults
        if params_file:
            with open(params_file, 'r') as f:
                self.params = json.load(f)
        else:
            # Conservative defaults (will be replaced by measurements)
            self.params = self._default_params()

        # State tracking
        self.last_received: Dict[str, ReceivedMessage] = {}
        self.message_buffer: Dict[str, deque] = {}  # For modeling delays
        self.current_time_ms = 0

        # Per-episode randomized parameters (if domain_randomization enabled)
        self._randomize_episode_params()

    def _default_params(self) -> dict:
        """Default parameters (conservative estimates)"""
        return {
            'latency': {
                'base_ms': 15,
                'distance_factor': 0.1,  # ms per meter
                'jitter_std_ms': 8,
            },
            'packet_loss': {
                'base_rate': 0.02,
                'distance_threshold_1': 50,
                'distance_threshold_2': 100,
                'rate_tier_1': 0.05,
                'rate_tier_2': 0.15,
                'rate_tier_3': 0.40,
            },
            'burst_loss': {
                'enabled': False,
                'mean_burst_length': 1,
            },
            'sensor_noise': {
                'gps_std_m': 5.0,
                'speed_std_ms': 0.5,
                'accel_std_ms2': 0.2,
                'heading_std_deg': 3.0,
            },
            'domain_randomization': {
                'latency_range_ms': [10, 80],
                'loss_rate_range': [0.0, 0.15],
            }
        }

    def _randomize_episode_params(self):
        """Randomize parameters for this episode (domain randomization)"""
        if not self.domain_randomization:
            self.episode_latency_base = self.params['latency']['base_ms']
            self.episode_loss_base = self.params['packet_loss']['base_rate']
            return

        # Randomize within domain randomization ranges
        dr = self.params['domain_randomization']

        self.episode_latency_base = random.uniform(
            dr['latency_range_ms'][0],
            dr['latency_range_ms'][1]
        )

        self.episode_loss_base = random.uniform(
            dr['loss_rate_range'][0],
            dr['loss_rate_range'][1]
        )

    def reset(self):
        """Reset emulator state for new episode"""
        self.last_received = {}
        self.message_buffer = {}
        self.current_time_ms = 0
        self._randomize_episode_params()

    def _calculate_distance(self, pos1: tuple, pos2: tuple) -> float:
        """Calculate Euclidean distance between two positions (x, y in meters)"""
        return np.sqrt((pos1[0] - pos2[0])**2 + (pos1[1] - pos2[1])**2)

    def _get_loss_probability(self, distance_m: float) -> float:
        """Get packet loss probability based on distance"""
        pl = self.params['packet_loss']

        base = self.episode_loss_base if self.domain_randomization else pl['base_rate']

        if distance_m < pl['distance_threshold_1']:
            return base
        elif distance_m < pl['distance_threshold_2']:
            # Linear interpolation
            t = (distance_m - pl['distance_threshold_1']) / (pl['distance_threshold_2'] - pl['distance_threshold_1'])
            return base + t * (pl['rate_tier_2'] - base)
        else:
            # Beyond threshold 2
            return pl['rate_tier_3']

    def _get_latency(self, distance_m: float) -> float:
        """Get one-way latency based on distance"""
        lat = self.params['latency']

        base = self.episode_latency_base if self.domain_randomization else lat['base_ms']
        distance_component = distance_m * lat['distance_factor']
        jitter = random.gauss(0, lat['jitter_std_ms'])

        return max(1, base + distance_component + jitter)

    def _add_sensor_noise(self, msg: V2VMessage) -> V2VMessage:
        """Add realistic sensor noise to message"""
        noise = self.params['sensor_noise']

        # Create copy with noise
        noisy = V2VMessage(
            vehicle_id=msg.vehicle_id,
            lat=msg.lat + random.gauss(0, noise['gps_std_m'] / 111000),  # ~degrees
            lon=msg.lon + random.gauss(0, noise['gps_std_m'] / 111000),
            speed=max(0, msg.speed + random.gauss(0, noise['speed_std_ms'])),
            heading=(msg.heading + random.gauss(0, noise['heading_std_deg'])) % 360,
            accel_x=msg.accel_x + random.gauss(0, noise['accel_std_ms2']),
            accel_y=msg.accel_y + random.gauss(0, noise['accel_std_ms2']),
            accel_z=msg.accel_z + random.gauss(0, noise['accel_std_ms2']),
            gyro_x=msg.gyro_x + random.gauss(0, 0.01),
            gyro_y=msg.gyro_y + random.gauss(0, 0.01),
            gyro_z=msg.gyro_z + random.gauss(0, 0.01),
            timestamp_ms=msg.timestamp_ms,
        )

        return noisy

    def _check_burst_loss(self, vehicle_id: str) -> bool:
        """Check if we're in a burst loss period"""
        if not self.params['burst_loss']['enabled']:
            return False

        # Simple burst model: if last packet was lost, higher chance this one is too
        if vehicle_id in self.message_buffer:
            # 50% chance to continue burst
            return random.random() < 0.5
        return False

    def transmit(self,
                 sender_msg: V2VMessage,
                 sender_pos: tuple,
                 receiver_pos: tuple,
                 current_time_ms: int) -> Optional[ReceivedMessage]:
        """
        Simulate transmission of a V2V message through ESP-NOW.

        Args:
            sender_msg: The message being sent (perfect SUMO data)
            sender_pos: Sender position (x, y) in meters
            receiver_pos: Receiver position (x, y) in meters
            current_time_ms: Current simulation time

        Returns:
            ReceivedMessage if successful, None if packet lost
        """
        self.current_time_ms = current_time_ms

        # Calculate distance
        distance = self._calculate_distance(sender_pos, receiver_pos)

        # Check for packet loss
        loss_prob = self._get_loss_probability(distance)

        # Burst loss check
        if self._check_burst_loss(sender_msg.vehicle_id):
            loss_prob = min(0.8, loss_prob * 3)  # Increase loss during burst

        if random.random() < loss_prob:
            # Packet lost
            return None

        # Calculate latency
        latency_ms = self._get_latency(distance)

        # Add sensor noise
        noisy_msg = self._add_sensor_noise(sender_msg)

        # Create received message
        received = ReceivedMessage(
            message=noisy_msg,
            age_ms=int(latency_ms),
            received_at_ms=current_time_ms + int(latency_ms)
        )

        # Update last received
        self.last_received[sender_msg.vehicle_id] = received

        return received

    def get_observation(self,
                        ego_speed: float,
                        current_time_ms: int) -> Dict:
        """
        Get current observation state for RL agent.

        Returns dictionary with latest received messages from each peer,
        including message age (staleness).

        Args:
            ego_speed: Current ego vehicle speed
            current_time_ms: Current simulation time

        Returns:
            Observation dict matching RL environment spec
        """
        obs = {
            'ego_speed': ego_speed,
            'timestamp_ms': current_time_ms,
        }

        for vehicle_id in ['V002', 'V003']:
            if vehicle_id in self.last_received:
                recv = self.last_received[vehicle_id]
                msg = recv.message

                # Calculate current age (time since received)
                age_ms = current_time_ms - recv.received_at_ms + recv.age_ms

                obs[f'{vehicle_id.lower()}_lat'] = msg.lat
                obs[f'{vehicle_id.lower()}_lon'] = msg.lon
                obs[f'{vehicle_id.lower()}_speed'] = msg.speed
                obs[f'{vehicle_id.lower()}_heading'] = msg.heading
                obs[f'{vehicle_id.lower()}_accel_x'] = msg.accel_x
                obs[f'{vehicle_id.lower()}_age_ms'] = age_ms
                obs[f'{vehicle_id.lower()}_valid'] = age_ms < 500  # Stale after 500ms
            else:
                # No message received yet
                obs[f'{vehicle_id.lower()}_lat'] = 0.0
                obs[f'{vehicle_id.lower()}_lon'] = 0.0
                obs[f'{vehicle_id.lower()}_speed'] = 0.0
                obs[f'{vehicle_id.lower()}_heading'] = 0.0
                obs[f'{vehicle_id.lower()}_accel_x'] = 0.0
                obs[f'{vehicle_id.lower()}_age_ms'] = 9999
                obs[f'{vehicle_id.lower()}_valid'] = False

        return obs


# Convenience function for testing
def test_emulator():
    """Test emulator with synthetic data"""
    emulator = ESPNOWEmulator(domain_randomization=True)

    # Simulate 100 transmissions
    received_count = 0
    total_latency = 0

    for i in range(100):
        msg = V2VMessage(
            vehicle_id='V002',
            lat=32.0 + i * 0.0001,
            lon=34.0,
            speed=15.0,
            heading=90.0,
            accel_x=0.0,
            accel_y=0.0,
            accel_z=9.8,
            gyro_x=0.0,
            gyro_y=0.0,
            gyro_z=0.0,
            timestamp_ms=i * 100,
        )

        result = emulator.transmit(
            sender_msg=msg,
            sender_pos=(i * 0.5, 0),  # V002 position
            receiver_pos=(0, 0),      # V001 position (ego)
            current_time_ms=i * 100
        )

        if result:
            received_count += 1
            total_latency += result.age_ms

    print(f"Received: {received_count}/100")
    print(f"Loss rate: {(100 - received_count)}%")
    print(f"Avg latency: {total_latency / received_count:.1f}ms")

if __name__ == "__main__":
    test_emulator()
```

---

## Phase 3: SUMO Integration

### Gym Environment with ESP-NOW Emulator

```python
# File: ml/envs/convoy_env.py

import gymnasium as gym
import numpy as np
import traci
from typing import Dict, Tuple, Optional
from ..espnow_emulator import ESPNOWEmulator, V2VMessage

class ConvoyEnv(gym.Env):
    """
    SUMO-based convoy environment with ESP-NOW communication emulation.

    Agent (V001) follows V002 and V003, receiving V2V messages with
    realistic communication effects (latency, loss, noise).
    """

    metadata = {'render_modes': ['human', 'rgb_array']}

    def __init__(self,
                 sumo_cfg: str,
                 espnow_params: str = None,
                 domain_randomization: bool = True,
                 render_mode: str = None):
        """
        Args:
            sumo_cfg: Path to SUMO .sumocfg file
            espnow_params: Path to emulator_params.json (from characterization)
            domain_randomization: Enable DR for robust training
            render_mode: 'human' for GUI, None for headless
        """
        super().__init__()

        self.sumo_cfg = sumo_cfg
        self.render_mode = render_mode

        # Initialize ESP-NOW emulator
        self.espnow = ESPNOWEmulator(
            params_file=espnow_params,
            domain_randomization=domain_randomization
        )

        # Action space: 4 discrete actions
        # 0 = Maintain, 1 = Caution, 2 = Brake, 3 = Emergency
        self.action_space = gym.spaces.Discrete(4)

        # Observation space
        # [ego_speed, v002_*, v003_*, ...]
        self.observation_space = gym.spaces.Box(
            low=-np.inf, high=np.inf,
            shape=(15,),  # Adjust based on actual features
            dtype=np.float32
        )

        # Reward parameters (from CLAUDE.md)
        self.COLLISION_REWARD = -100
        self.UNSAFE_PROXIMITY_REWARD = -5
        self.SAFE_DISTANCE_REWARD = +1
        self.HARSH_BRAKING_REWARD = -10
        self.UNNECESSARY_ALERT_REWARD = -2

        # Distance thresholds (meters)
        self.COLLISION_DISTANCE = 5
        self.UNSAFE_DISTANCE = 15
        self.SAFE_DISTANCE_MIN = 20
        self.SAFE_DISTANCE_MAX = 40

        self.sumo_running = False
        self.step_count = 0
        self.max_steps = 1000

    def _start_sumo(self):
        """Start SUMO simulation"""
        sumo_binary = 'sumo-gui' if self.render_mode == 'human' else 'sumo'

        traci.start([
            sumo_binary,
            '-c', self.sumo_cfg,
            '--step-length', '0.1',  # 10 Hz
            '--collision.action', 'warn',
            '--no-warnings', 'true',
        ])
        self.sumo_running = True

    def _get_vehicle_state(self, vehicle_id: str) -> Optional[V2VMessage]:
        """Get vehicle state from SUMO as V2VMessage"""
        try:
            pos = traci.vehicle.getPosition(vehicle_id)
            speed = traci.vehicle.getSpeed(vehicle_id)
            angle = traci.vehicle.getAngle(vehicle_id)
            accel = traci.vehicle.getAcceleration(vehicle_id)

            return V2VMessage(
                vehicle_id=vehicle_id,
                lat=pos[1],  # SUMO y -> lat
                lon=pos[0],  # SUMO x -> lon
                speed=speed,
                heading=angle,
                accel_x=accel,  # Longitudinal
                accel_y=0,
                accel_z=9.8,
                gyro_x=0,
                gyro_y=0,
                gyro_z=0,
                timestamp_ms=int(traci.simulation.getTime() * 1000),
            )
        except traci.exceptions.TraCIException:
            return None

    def _calculate_distance_to_leader(self) -> float:
        """Calculate distance from ego (V001) to nearest leader"""
        try:
            ego_pos = traci.vehicle.getPosition('V001')
            v002_pos = traci.vehicle.getPosition('V002')

            return np.sqrt(
                (ego_pos[0] - v002_pos[0])**2 +
                (ego_pos[1] - v002_pos[1])**2
            )
        except:
            return 100  # Default safe distance

    def _apply_action(self, action: int):
        """Apply agent's action to ego vehicle"""
        current_speed = traci.vehicle.getSpeed('V001')

        if action == 0:  # Maintain
            pass  # Do nothing, let SUMO car-following handle it
        elif action == 1:  # Caution
            traci.vehicle.setSpeed('V001', max(0, current_speed - 1))
        elif action == 2:  # Brake
            traci.vehicle.setSpeed('V001', max(0, current_speed - 3))
        elif action == 3:  # Emergency
            traci.vehicle.setSpeed('V001', max(0, current_speed - 5))

    def _get_reward(self, action: int, distance: float, decel: float) -> float:
        """Calculate reward based on current state and action"""
        reward = 0

        # Collision check
        if distance < self.COLLISION_DISTANCE:
            return self.COLLISION_REWARD

        # Unsafe proximity
        if distance < self.UNSAFE_DISTANCE:
            reward += self.UNSAFE_PROXIMITY_REWARD

        # Safe following distance
        if self.SAFE_DISTANCE_MIN < distance < self.SAFE_DISTANCE_MAX:
            reward += self.SAFE_DISTANCE_REWARD

        # Harsh braking penalty
        if decel > 4.5:  # m/s²
            reward += self.HARSH_BRAKING_REWARD

        # Unnecessary alert penalty
        if distance > self.SAFE_DISTANCE_MAX and action > 1:
            reward += self.UNNECESSARY_ALERT_REWARD

        return reward

    def _obs_to_array(self, obs: Dict) -> np.ndarray:
        """Convert observation dict to numpy array"""
        return np.array([
            obs['ego_speed'],
            obs.get('v002_lat', 0),
            obs.get('v002_lon', 0),
            obs.get('v002_speed', 0),
            obs.get('v002_heading', 0),
            obs.get('v002_accel_x', 0),
            obs.get('v002_age_ms', 9999) / 1000,  # Normalize to seconds
            obs.get('v002_valid', False),
            obs.get('v003_lat', 0),
            obs.get('v003_lon', 0),
            obs.get('v003_speed', 0),
            obs.get('v003_heading', 0),
            obs.get('v003_accel_x', 0),
            obs.get('v003_age_ms', 9999) / 1000,
            obs.get('v003_valid', False),
        ], dtype=np.float32)

    def reset(self, seed=None, options=None):
        """Reset environment for new episode"""
        super().reset(seed=seed)

        if self.sumo_running:
            traci.close()
            self.sumo_running = False

        self._start_sumo()
        self.espnow.reset()
        self.step_count = 0

        # Run a few steps to initialize
        for _ in range(10):
            traci.simulationStep()

        # Get initial observation
        obs = self._step_espnow()

        return self._obs_to_array(obs), {}

    def _step_espnow(self) -> Dict:
        """Process ESP-NOW communication for current timestep"""
        current_time_ms = int(traci.simulation.getTime() * 1000)
        ego_pos = traci.vehicle.getPosition('V001')
        ego_speed = traci.vehicle.getSpeed('V001')

        # Simulate V002 -> V001 transmission
        v002_msg = self._get_vehicle_state('V002')
        if v002_msg:
            v002_pos = traci.vehicle.getPosition('V002')
            self.espnow.transmit(
                sender_msg=v002_msg,
                sender_pos=v002_pos,
                receiver_pos=ego_pos,
                current_time_ms=current_time_ms
            )

        # Simulate V003 -> V001 transmission
        v003_msg = self._get_vehicle_state('V003')
        if v003_msg:
            v003_pos = traci.vehicle.getPosition('V003')
            self.espnow.transmit(
                sender_msg=v003_msg,
                sender_pos=v003_pos,
                receiver_pos=ego_pos,
                current_time_ms=current_time_ms
            )

        # Get observation (what agent sees after ESP-NOW effects)
        return self.espnow.get_observation(ego_speed, current_time_ms)

    def step(self, action: int):
        """Execute one environment step"""
        self.step_count += 1

        # Get pre-action state
        prev_speed = traci.vehicle.getSpeed('V001')

        # Apply action
        self._apply_action(action)

        # Step SUMO
        traci.simulationStep()

        # Get post-action state
        current_speed = traci.vehicle.getSpeed('V001')
        decel = (prev_speed - current_speed) / 0.1  # Deceleration in m/s²

        # Process ESP-NOW
        obs = self._step_espnow()

        # Calculate distance and reward
        distance = self._calculate_distance_to_leader()
        reward = self._get_reward(action, distance, decel)

        # Check termination
        terminated = distance < self.COLLISION_DISTANCE
        truncated = self.step_count >= self.max_steps

        # Check if simulation ended
        if traci.simulation.getMinExpectedNumber() <= 0:
            truncated = True

        return self._obs_to_array(obs), reward, terminated, truncated, {
            'distance': distance,
            'ego_speed': current_speed,
            'action': action,
        }

    def close(self):
        """Clean up"""
        if self.sumo_running:
            traci.close()
            self.sumo_running = False


# Register environment
gym.register(
    id='RoadSense-Convoy-v0',
    entry_point='ml.envs.convoy_env:ConvoyEnv',
)
```

---

## Phase 4: Validation & Augmentation

### Validation Against Real ESP32

After training, validate on real hardware:

1. Run same scenario on real ESP32 convoy
2. Log actual latency/loss/age_ms values
3. Compare distributions against emulator
4. If gap > 20%, refine emulator and retrain

### Augmentation Pipeline

Once emulator validated, generate training scenarios:

```python
# File: ml/augmentation/scenario_generator.py

import subprocess
import os
from typing import List, Dict

class ScenarioGenerator:
    """Generate augmented SUMO scenarios for training"""

    def __init__(self, base_network: str, output_dir: str):
        self.base_network = base_network
        self.output_dir = output_dir
        os.makedirs(output_dir, exist_ok=True)

    def generate_convoy_variations(self,
                                   n_scenarios: int,
                                   speed_range: tuple = (10, 25),
                                   gap_range: tuple = (15, 50),
                                   brake_time_range: tuple = (30, 90)) -> List[str]:
        """
        Generate convoy scenarios with parameter variations.

        Returns list of scenario config file paths.
        """
        scenarios = []

        for i in range(n_scenarios):
            # Randomize parameters
            import random

            speed = random.uniform(*speed_range)
            initial_gap = random.uniform(*gap_range)
            brake_time = random.uniform(*brake_time_range)
            brake_decel = random.uniform(3.0, 6.0)

            # Generate scenario files
            scenario_name = f"convoy_{i:04d}"
            scenario_dir = os.path.join(self.output_dir, scenario_name)
            os.makedirs(scenario_dir, exist_ok=True)

            # Generate route file
            route_file = self._generate_route(
                scenario_dir, speed, initial_gap
            )

            # Generate config
            cfg_file = self._generate_config(
                scenario_dir, route_file, brake_time, brake_decel
            )

            scenarios.append(cfg_file)

        return scenarios

    def _generate_route(self, output_dir: str, speed: float, gap: float) -> str:
        """Generate vehicles.rou.xml with specified parameters"""
        route_content = f'''<?xml version="1.0" encoding="UTF-8"?>
<routes>
    <vType id="car_convoy" accel="2.6" decel="4.5" sigma="0.5"
           length="5" maxSpeed="{speed + 5}" tau="1.0">
        <param key="has.ssm.device" value="true"/>
        <param key="device.ssm.measures" value="TTC DRAC"/>
        <param key="device.ssm.thresholds" value="4.0 3.4"/>
    </vType>

    <route id="convoy_route" edges="636924784#1 636924784#0"/>

    <vehicle id="V003" type="car_convoy" route="convoy_route"
             depart="0" departPos="{gap * 2}" departSpeed="{speed}" color="red"/>
    <vehicle id="V002" type="car_convoy" route="convoy_route"
             depart="0" departPos="{gap}" departSpeed="{speed}" color="green"/>
    <vehicle id="V001" type="car_convoy" route="convoy_route"
             depart="0" departPos="0" departSpeed="{speed}" color="blue"/>
</routes>
'''
        route_file = os.path.join(output_dir, 'vehicles.rou.xml')
        with open(route_file, 'w') as f:
            f.write(route_content)

        return route_file

    def _generate_config(self, output_dir: str, route_file: str,
                         brake_time: float, brake_decel: float) -> str:
        """Generate scenario.sumocfg"""
        # Also generate additional file for emergency brake event
        additional_content = f'''<?xml version="1.0" encoding="UTF-8"?>
<additional>
    <variableSpeedSign id="emergency_brake" lanes="636924784#1_0">
        <step time="{brake_time}" speed="0"/>
        <step time="{brake_time + 5}" speed="20"/>
    </variableSpeedSign>
</additional>
'''
        additional_file = os.path.join(output_dir, 'additional.xml')
        with open(additional_file, 'w') as f:
            f.write(additional_content)

        cfg_content = f'''<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <input>
        <net-file value="{self.base_network}"/>
        <route-files value="{route_file}"/>
        <additional-files value="{additional_file}"/>
    </input>
    <time>
        <begin value="0"/>
        <end value="150"/>
        <step-length value="0.1"/>
    </time>
    <output>
        <fcd-output value="fcd_output.xml"/>
        <fcd-output.geo value="true"/>
        <device.ssm.file value="ssm_output.xml"/>
    </output>
</configuration>
'''
        cfg_file = os.path.join(output_dir, 'scenario.sumocfg')
        with open(cfg_file, 'w') as f:
            f.write(cfg_content)

        return cfg_file
```

---

## 2-Week Implementation Timeline

### Week 1: Characterization & Emulator

| Day | Task | Deliverable |
|-----|------|-------------|
| **Day 1** | Flash characterization firmware to V001/V002 | `sender_rtt.cpp`, `reflector_echo.cpp` working |
| **Day 2** | Run all distance tests (1m, 10m, 30m, 50m, 80m, 100m) | `rtt_*.csv` files |
| **Day 3** | Analyze data, generate emulator parameters | `emulator_params.json`, plots |
| **Day 4** | Implement `ESPNOWEmulator` class | `espnow_emulator.py` tested |
| **Day 5** | Integrate with SUMO (TraCI) | Basic SUMO+emulator working |
| **Day 6** | Create `ConvoyEnv` Gym environment | Can step through episodes |
| **Day 7** | Buffer / catch-up day | Documentation |

### Week 2: Training & Validation

| Day | Task | Deliverable |
|-----|------|-------------|
| **Day 8** | Test RL training (short run, 10k steps) | Verify training loop works |
| **Day 9** | First validation on real ESP32 | Compare emulator vs reality |
| **Day 10** | Refine emulator if needed | Updated parameters |
| **Day 11** | Augmentation pipeline | `ScenarioGenerator` working |
| **Day 12** | Generate 50 training scenarios | `scenarios/` directory |
| **Day 13** | Full training run (100k+ steps) | Trained model checkpoint |
| **Day 14** | Final validation, documentation | Report on Sim2Real gap |

---

## Risk Mitigation

### Risk 1: Characterization Results Unexpected

**Symptoms:** Bimodal latency, extreme packet loss, etc.
**Mitigation:**
- Document findings honestly
- Adjust emulator to match reality
- May need to increase domain randomization range

### Risk 2: SUMO Integration Issues

**Symptoms:** TraCI errors, timing issues
**Mitigation:**
- Start with simple scenario (straight road)
- Test emulator standalone first
- Have fallback: process FCD files offline instead of real-time TraCI

### Risk 3: Training Diverges

**Symptoms:** Reward doesn't improve, agent learns bad policy
**Mitigation:**
- Start with simpler reward (binary: collision vs no collision)
- Tune hyperparameters (learning rate, batch size)
- Reduce observation space if needed

### Risk 4: Sim2Real Gap Too Large

**Symptoms:** >30% performance drop on real hardware
**Mitigation:**
- Increase domain randomization
- Fine-tune on small real dataset
- Document gap honestly in thesis

---

## Success Criteria

### Phase 1 (Characterization) - Day 3
- [ ] RTT measured at 6+ distances
- [ ] Packet loss characterized
- [ ] `emulator_params.json` generated

### Phase 2 (Emulator) - Day 5
- [ ] `ESPNOWEmulator` class implemented
- [ ] Matches measured latency distribution
- [ ] Matches measured loss rates

### Phase 3 (SUMO Integration) - Day 7
- [ ] `ConvoyEnv` runs without errors
- [ ] Agent can take actions and receive observations
- [ ] Episodes terminate correctly

### Phase 4 (Validation) - Day 14
- [ ] Sim2Real gap < 25%
- [ ] 50+ training scenarios generated
- [ ] Model trains without divergence

---

## References

- Domain Randomization: Tobin et al., 2017 - "Domain Randomization for Transferring Deep Neural Networks from Simulation to the Real World"
- SUMO FCD Output: https://sumo.dlr.de/docs/Simulation/Output/FCDOutput.html
- ESP-NOW Documentation: https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/network/esp_now.html
- Gymnasium (Gym): https://gymnasium.farama.org/

---

**Document Version:** 1.1
**Created:** December 19, 2025
**Updated:** December 26, 2025 - Sensor synthesis marked optional (see ARCHITECTURE_V2_MINIMAL_RL.md)
**Status:** Ready for Implementation
