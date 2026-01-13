# DataLogger Modification Prompt for AI Agent

**Date:** December 14, 2025
**Priority:** HIGH - Required before real-world data collection
**Estimated Effort:** 1-2 hours

---

## Context

The RoadSense V2V system requires modifying the DataLogger to support the **Single-Agent Architecture** where:
- The ML model runs on ONE vehicle only (V001 - the ego/following vehicle)
- V001 logs **RECEIVED** V2V messages from other vehicles (V002, V003)
- V001 does NOT primarily log its own sensor data (except ego_speed for context)

---

## Current Implementation

**Files to modify:**
- `roadsense-v2v/hardware/src/logging/DataLogger.h`
- `roadsense-v2v/hardware/src/logging/DataLogger.cpp`

**Current behavior:**
- Logs the ego vehicle's OWN V2VMessage (its own sensors, position, etc.)
- Single `logSample(V2VMessage msg, GpsData gps)` method
- CSV format with 20 columns of ego vehicle data

**Current CSV columns:**
```
timestamp_ms,vehicle_id,lat,lon,alt,speed,heading,
long_accel,lat_accel,accel_x,accel_y,accel_z,
gyro_x,gyro_y,gyro_z,mag_x,mag_y,mag_z,
gps_valid,gps_age_ms
```

---

## Required Changes

### 1. New CSV Format

**New columns (25 total):**
```csv
timestamp_ms,ego_speed,
v002_lat,v002_lon,v002_speed,v002_heading,v002_accel_x,v002_accel_y,v002_accel_z,v002_gyro_x,v002_gyro_y,v002_gyro_z,v002_age_ms,
v003_lat,v003_lon,v003_speed,v003_heading,v003_accel_x,v003_accel_y,v003_accel_z,v003_gyro_x,v003_gyro_y,v003_gyro_z,v003_age_ms
```

**Column descriptions:**
| Column | Type | Description |
|--------|------|-------------|
| `timestamp_ms` | uint32_t | millis() when row logged |
| `ego_speed` | float | V001's own speed in m/s (from GPS) |
| `v002_lat` | float | V002's latitude from last received message |
| `v002_lon` | float | V002's longitude |
| `v002_speed` | float | V002's speed in m/s |
| `v002_heading` | float | V002's heading in degrees |
| `v002_accel_x/y/z` | float | V002's IMU accelerometer (m/s^2) |
| `v002_gyro_x/y/z` | float | V002's IMU gyroscope (rad/s) |
| `v002_age_ms` | uint32_t | Time since last V002 message received |
| `v003_*` | ... | Same fields for V003 |

### 2. New Filename Format

**Change from:** `V001_session_XXX.csv`
**Change to:** `scenario_XXX.csv`

The session counter logic remains the same (NVS persistent), just change the filename format.

### 3. New Data Structures

Add to `DataLogger.h`:

```cpp
// Structure to hold last received V2V message from a peer
struct PeerData {
    V2VMessage lastMessage;      // Last received V2VMessage
    uint32_t lastReceivedTime;   // millis() when received
    bool hasData;                // True if we've received at least one message
};

// Maximum number of peers to track
static const int MAX_PEERS = 2;  // V002 and V003
```

Add member variables:
```cpp
private:
    // Peer message storage (indexed: 0=V002, 1=V003)
    PeerData m_peers[MAX_PEERS];

    // Ego vehicle speed (from own GPS)
    float m_egoSpeed;
```

### 4. New Public Methods

**Add method to receive V2V messages from other vehicles:**

```cpp
/**
 * @brief Update stored data for a peer vehicle (called when V2V message received)
 * @param msg The V2VMessage received from another vehicle
 * @note Extracts vehicleId to determine which peer slot (V002=0, V003=1)
 */
void updatePeerData(const V2VMessage& msg);

/**
 * @brief Update ego vehicle speed (called before logging)
 * @param speed Current speed of this vehicle in m/s
 */
void setEgoSpeed(float speed);
```

**Modify existing logSample:**

```cpp
/**
 * @brief Log a row containing all peer data + ego speed
 * @note Call this at 10Hz from main loop (does NOT take V2VMessage parameter anymore)
 */
void logSample();
```

### 5. Implementation Details

**updatePeerData():**
```cpp
void DataLogger::updatePeerData(const V2VMessage& msg) {
    int peerIndex = -1;

    // Determine peer index from vehicleId
    if (strncmp(msg.vehicleId, "V002", 4) == 0) {
        peerIndex = 0;
    } else if (strncmp(msg.vehicleId, "V003", 4) == 0) {
        peerIndex = 1;
    }

    if (peerIndex >= 0 && peerIndex < MAX_PEERS) {
        m_peers[peerIndex].lastMessage = msg;
        m_peers[peerIndex].lastReceivedTime = millis();
        m_peers[peerIndex].hasData = true;
    }
}
```

**logSample() (new implementation):**
```cpp
void DataLogger::logSample() {
    if (!m_isLogging) return;

    // Build CSV row with all peer data
    char rowBuffer[LOG_ROW_SIZE_BYTES];
    buildCsvRow(rowBuffer, sizeof(rowBuffer));

    // ... rest same as before (buffer append, auto-flush logic)
}
```

**buildCsvRow() (new implementation):**
```cpp
void DataLogger::buildCsvRow(char* outBuffer, size_t bufferSize) {
    uint32_t now = millis();

    // Calculate age for each peer
    uint32_t v002_age = m_peers[0].hasData ? (now - m_peers[0].lastReceivedTime) : 99999;
    uint32_t v003_age = m_peers[1].hasData ? (now - m_peers[1].lastReceivedTime) : 99999;

    // Get peer data (or zeros if no data received yet)
    const V2VMessage& v002 = m_peers[0].lastMessage;
    const V2VMessage& v003 = m_peers[1].lastMessage;

    snprintf(outBuffer, bufferSize,
        "%lu,%.2f,"                                    // timestamp, ego_speed
        "%.6f,%.6f,%.2f,%.1f,%.2f,%.2f,%.2f,%.3f,%.3f,%.3f,%lu,"  // v002 data
        "%.6f,%.6f,%.2f,%.1f,%.2f,%.2f,%.2f,%.3f,%.3f,%.3f,%lu\n", // v003 data

        now,
        m_egoSpeed,

        // V002 data
        m_peers[0].hasData ? v002.position.lat : 0.0f,
        m_peers[0].hasData ? v002.position.lon : 0.0f,
        m_peers[0].hasData ? v002.dynamics.speed : 0.0f,
        m_peers[0].hasData ? v002.dynamics.heading : 0.0f,
        m_peers[0].hasData ? v002.sensors.accel[0] : 0.0f,
        m_peers[0].hasData ? v002.sensors.accel[1] : 0.0f,
        m_peers[0].hasData ? v002.sensors.accel[2] : 0.0f,
        m_peers[0].hasData ? v002.sensors.gyro[0] : 0.0f,
        m_peers[0].hasData ? v002.sensors.gyro[1] : 0.0f,
        m_peers[0].hasData ? v002.sensors.gyro[2] : 0.0f,
        v002_age,

        // V003 data
        m_peers[1].hasData ? v003.position.lat : 0.0f,
        m_peers[1].hasData ? v003.position.lon : 0.0f,
        m_peers[1].hasData ? v003.dynamics.speed : 0.0f,
        m_peers[1].hasData ? v003.dynamics.heading : 0.0f,
        m_peers[1].hasData ? v003.sensors.accel[0] : 0.0f,
        m_peers[1].hasData ? v003.sensors.accel[1] : 0.0f,
        m_peers[1].hasData ? v003.sensors.accel[2] : 0.0f,
        m_peers[1].hasData ? v003.sensors.gyro[0] : 0.0f,
        m_peers[1].hasData ? v003.sensors.gyro[1] : 0.0f,
        m_peers[1].hasData ? v003.sensors.gyro[2] : 0.0f,
        v003_age
    );
}
```

**writeHeader() update:**
```cpp
bool DataLogger::writeHeader() {
    const char* header =
        "timestamp_ms,ego_speed,"
        "v002_lat,v002_lon,v002_speed,v002_heading,v002_accel_x,v002_accel_y,v002_accel_z,v002_gyro_x,v002_gyro_y,v002_gyro_z,v002_age_ms,"
        "v003_lat,v003_lon,v003_speed,v003_heading,v003_accel_x,v003_accel_y,v003_accel_z,v003_gyro_x,v003_gyro_y,v003_gyro_z,v003_age_ms\n";

    // ... rest same
}
```

**createLogFile() update:**
```cpp
bool DataLogger::createLogFile(const char* vehicleId) {
    // Change filename format: scenario_XXX.csv
    snprintf(m_filename, sizeof(m_filename), "%s/scenario_%03d.csv",
             LOG_DIRECTORY, m_sessionNum);

    // ... rest same
}
```

### 6. Integration with main.cpp

In `main.cpp`, update the ESP-NOW receive callback and main loop:

```cpp
// When V2V message received from another vehicle:
void onV2VMessageReceived(const V2VMessage& msg) {
    // Update DataLogger with peer data
    dataLogger.updatePeerData(msg);

    // ... existing processing
}

// In main loop (10Hz):
void loop() {
    // Update ego speed from GPS
    dataLogger.setEgoSpeed(gps.getSpeed());

    // Log sample (uses stored peer data)
    dataLogger.logSample();

    // ... rest of loop
}
```

### 7. Initialization

In `begin()` or constructor, initialize peer data:

```cpp
DataLogger::DataLogger() : /* ... existing ... */ {
    // Initialize peer data
    for (int i = 0; i < MAX_PEERS; i++) {
        memset(&m_peers[i], 0, sizeof(PeerData));
        m_peers[i].hasData = false;
    }
    m_egoSpeed = 0.0f;
}
```

In `startLogging()`:
```cpp
bool DataLogger::startLogging(const char* vehicleId, bool gpsReady) {
    // ... existing code ...

    // Reset peer data for new session
    for (int i = 0; i < MAX_PEERS; i++) {
        m_peers[i].hasData = false;
        m_peers[i].lastReceivedTime = 0;
    }
    m_egoSpeed = 0.0f;

    // ... rest of function
}
```

---

## Testing Checklist

1. [ ] Compile successfully with no errors
2. [ ] SD card initialization works (10 MHz SPI)
3. [ ] File created as `scenario_XXX.csv`
4. [ ] Header row has correct 25 columns
5. [ ] `updatePeerData()` correctly identifies V002 vs V003
6. [ ] Peer age calculates correctly (increases between messages)
7. [ ] Zeros logged for missing peer data (hasData=false)
8. [ ] 10 Hz logging rate maintained
9. [ ] Button start/stop still works (GPIO 4)
10. [ ] Buffer flush works correctly

---

## Config Changes (if needed)

In `config.h`, update buffer size if row is longer:

```cpp
// Old: 20 columns ~200 bytes per row
// New: 25 columns ~300 bytes per row
#define LOG_ROW_SIZE_BYTES 350  // Increase from 256 if needed
```

---

## Summary of Changes

| File | Change |
|------|--------|
| `DataLogger.h` | Add `PeerData` struct, `m_peers[]`, `m_egoSpeed`, new methods |
| `DataLogger.cpp` | Implement `updatePeerData()`, `setEgoSpeed()`, update `logSample()`, `buildCsvRow()`, `writeHeader()`, `createLogFile()` |
| `main.cpp` | Call `updatePeerData()` on receive, `setEgoSpeed()` before logging |
| `config.h` | Increase `LOG_ROW_SIZE_BYTES` if needed |

---

## Questions for Human Review

1. Should we log V001's own position/lat/lon in addition to ego_speed?
2. Should we include a `peer_count` column (number of peers with valid data)?
3. Maximum acceptable `*_age_ms` before considering data stale? (Currently just logging raw age)
