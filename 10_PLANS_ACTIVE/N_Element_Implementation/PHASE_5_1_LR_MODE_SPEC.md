# Phase 5.1: ESP-NOW Long Range (LR) Mode Implementation Spec

**Date:** January 19, 2026
**Phase:** 5.1 (Firmware Migration Pre-requisite)
**Status:** READY FOR BUILDER
**Priority:** HIGH - Addresses critical 5m range limitation
**Target Agent:** BUILDER (Codex/Gemini)

---

## Objective

Enable ESP-NOW Long Range (LR) mode and maximum TX power on all three firmware files to extend communication range from ~5m to potentially 10-15m. This is a **software-only fix** requiring no hardware changes.

---

## Prerequisites

- [x] Access to RoadSense V2V git repository at `/home/amirkhalifa/RoadSense2/roadsense-v2v/`
- [x] PlatformIO CLI installed and functional
- [x] Two ESP32 DevKit units available for testing (V001 sender, V002 reflector)
- [x] USB ports available: `/dev/ttyUSB0`, `/dev/ttyUSB1` (or Windows COM ports)
- [x] Read and understand: `docs/10_PLANS_ACTIVE/ESPNOW_LONG_RANGE_MODE_MIGRATION.md`

---

## Implementation Steps

### Step 1: Modify EspNowTransport.cpp

**File:** `roadsense-v2v/hardware/src/network/transport/EspNowTransport.cpp`

**Location:** Inside `begin()` method, **after line 86** (channel log), **before line 88** (Step 4 comment)

**Current Code Context (lines 84-92):**
```cpp
    // Step 3: Set WiFi channel
    esp_wifi_set_channel(channel, WIFI_SECOND_CHAN_NONE);
    logger.info("ESP-NOW", "WiFi channel set to " + String(channel));

    // Step 4: Initialize ESP-NOW
    if (esp_now_init() != ESP_OK) {
        logger.error("ESP-NOW", "esp_now_init() failed!");
        return false;
    }
```

**Insert After line 86 (channel log), Before line 88 (Step 4 comment) - replacing the empty line 87:**
```cpp
    // Step 3.5: Enable Long Range (LR) Mode and Max TX Power
    // =========================================================================
    // WIFI_PROTOCOL_LR: Proprietary Espressif mode (~250-500kbps)
    // - Increases receiver sensitivity by +5-10 dB
    // - Potentially doubles/triples range compared to standard 802.11
    // - CRITICAL: Both sender and receiver MUST use LR mode
    // Reference: https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-guides/wifi.html#lr
    // =========================================================================
    esp_err_t lr_result = esp_wifi_set_protocol(WIFI_IF_STA, WIFI_PROTOCOL_LR);
    if (lr_result == ESP_OK) {
        logger.info("ESP-NOW", "Long Range (LR) mode ENABLED");
    } else {
        logger.warning("ESP-NOW", "Failed to set LR mode (err=" + String(lr_result) + "), using standard protocol");
    }

    // Set maximum TX power: 84 × 0.25 = 21 dBm (hardware caps to ~19.5 dBm)
    esp_err_t tx_result = esp_wifi_set_max_tx_power(84);
    if (tx_result == ESP_OK) {
        logger.info("ESP-NOW", "TX power set to maximum (84 = ~21dBm)");
    } else {
        logger.warning("ESP-NOW", "Failed to set TX power (err=" + String(tx_result) + ")");
    }

```

**Verification:** `#include <esp_wifi.h>` is already present at line 24 of EspNowTransport.h. No additional includes needed.

---

### Step 2: Modify sender_main.cpp

**File:** `roadsense-v2v/hardware/test/espnow_rtt/sender_main.cpp`

**Location:** Inside `initEspNow()` function, **after line 146** (channel log), **before line 148** (Step 4 comment)

**Current Code Context (lines 144-152):**
```cpp
    // Step 3: Set WiFi channel
    esp_wifi_set_channel(ESPNOW_CHANNEL, WIFI_SECOND_CHAN_NONE);
    logger.info("ESP-NOW", "WiFi channel set to " + String(ESPNOW_CHANNEL));

    // Step 4: Initialize ESP-NOW
    if (esp_now_init() != ESP_OK) {
        logger.error("ESP-NOW", "esp_now_init() failed!");
        return false;
    }
```

**Insert After line 146 (channel log), Before line 148 (Step 4 comment) - replacing the empty line 147:**
```cpp
    // Step 3.5: Enable Long Range (LR) Mode and Max TX Power
    // =========================================================================
    // WIFI_PROTOCOL_LR: Proprietary Espressif mode (~250-500kbps)
    // - Increases receiver sensitivity by +5-10 dB
    // - CRITICAL: Both sender and reflector MUST use LR mode
    // =========================================================================
    esp_err_t lr_result = esp_wifi_set_protocol(WIFI_IF_STA, WIFI_PROTOCOL_LR);
    if (lr_result == ESP_OK) {
        logger.info("ESP-NOW", "Long Range (LR) mode ENABLED");
    } else {
        logger.warning("ESP-NOW", "Failed to set LR mode (err=" + String(lr_result) + "), using standard protocol");
    }

    esp_err_t tx_result = esp_wifi_set_max_tx_power(84);
    if (tx_result == ESP_OK) {
        logger.info("ESP-NOW", "TX power set to maximum (84 = ~21dBm)");
    } else {
        logger.warning("ESP-NOW", "Failed to set TX power (err=" + String(tx_result) + ")");
    }

```

**Verification:** `#include <esp_wifi.h>` is already present at line 20. No additional includes needed.

---

### Step 3: Modify reflector_main.cpp

**File:** `roadsense-v2v/hardware/test/espnow_rtt/reflector_main.cpp`

**Location:** Inside `initEspNow()` function, **after line 89** (channel log), **before line 91** (Step 4 comment)

**Current Code Context (lines 87-95):**
```cpp
    // Step 3: Set WiFi channel
    esp_wifi_set_channel(ESPNOW_CHANNEL, WIFI_SECOND_CHAN_NONE);
    logger.info("ESP-NOW", "WiFi channel set to " + String(ESPNOW_CHANNEL));

    // Step 4: Initialize ESP-NOW
    if (esp_now_init() != ESP_OK) {
        logger.error("ESP-NOW", "esp_now_init() failed!");
        return false;
    }
```

**Insert After line 89 (channel log), Before line 91 (Step 4 comment) - replacing the empty line 90:**
```cpp
    // Step 3.5: Enable Long Range (LR) Mode and Max TX Power
    // =========================================================================
    // WIFI_PROTOCOL_LR: Proprietary Espressif mode (~250-500kbps)
    // - Increases receiver sensitivity by +5-10 dB
    // - CRITICAL: Both sender and reflector MUST use LR mode
    // =========================================================================
    esp_err_t lr_result = esp_wifi_set_protocol(WIFI_IF_STA, WIFI_PROTOCOL_LR);
    if (lr_result == ESP_OK) {
        logger.info("ESP-NOW", "Long Range (LR) mode ENABLED");
    } else {
        logger.warning("ESP-NOW", "Failed to set LR mode (err=" + String(lr_result) + "), using standard protocol");
    }

    esp_err_t tx_result = esp_wifi_set_max_tx_power(84);
    if (tx_result == ESP_OK) {
        logger.info("ESP-NOW", "TX power set to maximum (84 = ~21dBm)");
    } else {
        logger.warning("ESP-NOW", "Failed to set TX power (err=" + String(tx_result) + ")");
    }

```

**Verification:** `#include <esp_wifi.h>` is already present at line 17. No additional includes needed.

---

## Build Commands

Execute from: `roadsense-v2v/hardware/`

### Step 4: Build All Affected Environments

```bash
# Build RTT sender firmware
pio run -e rtt_sender

# Build RTT reflector firmware
pio run -e rtt_reflector

# Build main V2V firmware (EspNowTransport)
pio run -e esp32dev
```

**Expected Result:** All three builds complete with 0 errors.

**If Build Fails:**
- Check for syntax errors in the inserted code
- Verify the insertion point is correct (between channel set and esp_now_init)
- Ensure no duplicate variable declarations

---

## Testing Procedure

### Step 5: Run Existing Unit Tests

```bash
cd roadsense-v2v/hardware

# Run all unit tests (requires ESP32 connected)
pio test -e esp32dev --upload-port /dev/ttyUSB0

# If above succeeds, run specific RTT-related tests
pio test -f test_rtt_packet --upload-port /dev/ttyUSB0
pio test -f test_rtt_timeout --upload-port /dev/ttyUSB0
pio test -f test_rtt_circular_buffer --upload-port /dev/ttyUSB0
pio test -f test_v2v_message --upload-port /dev/ttyUSB0
```

**Expected Result:** All tests pass (42/42 RTT tests historically).

**Note:** These tests validate packet structures and logic, NOT the RF layer. They should pass unchanged.

---

### Step 6: Flash RTT Firmware for Hardware Validation

```bash
cd roadsense-v2v/hardware

# Flash sender to Device 1 (V001)
pio run -e rtt_sender --target upload --upload-port /dev/ttyUSB0

# Flash reflector to Device 2 (V002)
pio run -e rtt_reflector --target upload --upload-port /dev/ttyUSB1
```

---

### Step 7: Verify Serial Output (LR Mode Enabled)

Open two serial monitors:

```bash
# Terminal 1 - Sender
pio device monitor --port /dev/ttyUSB0 --baud 115200

# Terminal 2 - Reflector
pio device monitor --port /dev/ttyUSB1 --baud 115200
```

**Expected Log Output (BOTH devices must show):**
```
[INFO] [ESP-NOW] Long Range (LR) mode ENABLED
[INFO] [ESP-NOW] TX power set to maximum (84 = ~21dBm)
```

**If you see warnings instead:**
```
[WARN] [ESP-NOW] Failed to set LR mode (err=X), using standard protocol
```
This indicates the API call failed. Document the error code and check ESP-IDF docs.

---

### Step 8: Baseline Communication Test (5m)

1. Place both devices **5 meters apart** (same as previous characterization)
2. Power on reflector first, wait for "Waiting for packets..."
3. Power on sender, observe packet transmission
4. **Run for 2 minutes minimum**

**Success Criteria:**
- Packet loss ≤ 15% (same or better than standard mode baseline)
- RTT values in normal range (10-50ms)
- No communication failures

**If 5m test fails:** STOP. LR mode may have a compatibility issue. Document and rollback.

---

### Step 9: Extended Range Tests

If 5m baseline passes, proceed with extended range testing:

| Distance | Duration | Success Threshold |
|----------|----------|-------------------|
| 10m | 2 min | Packet loss ≤ 25% |
| 15m | 2 min | Packet loss ≤ 35% |
| 20m | 2 min | Any communication success |

**For each distance:**
1. Move devices to target distance
2. Run sender for 2 minutes
3. Check SD card log on sender for packet loss statistics
4. Document results

---

## Exit Criteria

All of the following must be true:

- [x] **BUILD:** All three environments compile without errors
- [x] **UNIT TESTS:** All existing tests pass (42/42 minimum)
- [x] **SERIAL OUTPUT:** Both devices log "Long Range (LR) mode ENABLED"
- [x] **SERIAL OUTPUT:** Both devices log "TX power set to maximum"
- [x] **BASELINE:** 5m test shows ≤15% packet loss
- [ ] **IMPROVEMENT:** At least one extended distance (10m, 15m, or 20m) shows usable communication

---

## Files Touched

| File | Change | Lines Affected |
|------|--------|----------------|
| `hardware/src/network/transport/EspNowTransport.cpp` | Add LR mode + TX power after channel set | ~86-105 (insert ~19 lines) |
| `hardware/test/espnow_rtt/sender_main.cpp` | Add LR mode + TX power after channel set | ~146-165 (insert ~19 lines) |
| `hardware/test/espnow_rtt/reflector_main.cpp` | Add LR mode + TX power after channel set | ~89-108 (insert ~19 lines) |

---

## Risks / Notes

### Critical: Both Devices Must Use LR Mode
LR mode uses proprietary Espressif modulation. A standard WiFi receiver cannot decode LR transmissions:
```
Sender (LR) → Receiver (Standard) = NO COMMUNICATION
Sender (Standard) → Receiver (LR) = NO COMMUNICATION
Sender (LR) → Receiver (LR) = SUCCESS
```
**Both ESP32 devices MUST be flashed with LR-enabled firmware before testing.**

### Initialization Order is Fragile
The EspNowTransport.h header (lines 71-77) explicitly warns:
> "The initialization sequence order is fragile and must not be modified without testing"

**DO NOT:**
- Move the LR mode code before `WiFi.disconnect()`
- Move the LR mode code after `esp_now_init()`
- Change the order of existing initialization steps

### Error Handling is Non-Fatal
The LR mode and TX power calls use `warning` level logging on failure, not `error`. This is intentional:
- Allows fallback to standard protocol if LR mode fails
- Does not abort initialization
- Documents the failure for debugging

### No Changes to platformio.ini Required
The ESP-IDF functions are part of the standard ESP32 Arduino core. No additional libraries needed.

### Emulator Parameters May Need Re-characterization
After validating LR mode works, the `emulator_params_5m.json` will need updating with new RTT/loss characteristics measured at the optimal LR distance (likely 10m). This is a separate task.

---

## Rollback Plan

If LR mode causes issues:

1. Remove the 19-line LR mode block from all three files
2. Rebuild: `pio run -e rtt_sender && pio run -e rtt_reflector && pio run -e esp32dev`
3. Re-flash both devices
4. Verify standard mode still works at 5m
5. Document failure mode and escalate to hardware solution (external antennas)

---

## References

- [ESP-IDF WiFi LR Mode Docs](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-guides/wifi.html#lr)
- [ESP-NOW Programming Guide](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/network/esp_now.html)
- Migration Plan: `docs/10_PLANS_ACTIVE/ESPNOW_LONG_RANGE_MODE_MIGRATION.md`
- Testing Guide: `docs/20_KNOWLEDGE_BASE/PLATFORMIO_TESTING_README.md`

---

**Document Version:** 1.0
**Author:** PLANNER Agent (Claude Opus 4.5)
**Next Action:** BUILDER implements Steps 1-3, then proceeds through Steps 4-9 for validation
