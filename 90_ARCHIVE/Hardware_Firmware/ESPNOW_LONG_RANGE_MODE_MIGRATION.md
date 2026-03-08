# ESP-NOW Long Range (LR) Mode Migration Plan

**Date:** January 10, 2026
**Status:** PLANNED - Pending Implementation
**Priority:** HIGH - Addresses critical 5m range limitation
**Author:** Amir (with Claude Code analysis)

---

## Executive Summary

The current ESP-NOW implementation uses default WiFi protocol (802.11b/g/n), limiting reliable range to **~5 meters** with PCB antennas inside vehicles. This document outlines a **software-only fix** using ESP-NOW's Long Range (LR) mode to potentially **double or triple effective range** without hardware changes.

---

## 1. Problem Statement

### 1.1 Current Limitation

| Distance | Packet Loss | Status |
|----------|-------------|--------|
| **5m** | 15.7% | Only usable range |
| **9m** | 70-80% | Unusable |
| **20m** | ~100% | Total failure |

**Root causes:**
1. **PCB antenna weakness:** ESP32 DevKit's small PCB antenna has limited gain
2. **Faraday cage effect:** Metal vehicle chassis blocks/attenuates 2.4GHz signals
3. **Multipath interference:** Ground reflections cause signal cancellation
4. **Default protocol assumption:** 802.11b/g/n prioritizes bandwidth over range

### 1.2 Why 5m is Unacceptable for V2V

- Highway following distance at 50 km/h: **~30-50 meters**
- Urban intersection crossing: **10-30 meters**
- Parking lot scenarios: **5-15 meters**

The current 5m range is only suitable for stationary testing or convoy vehicles physically adjacent (impractical).

---

## 2. Proposed Solution: WIFI_PROTOCOL_LR

### 2.1 What is Long Range Mode?

ESP-NOW Long Range is a **proprietary Espressif modulation scheme** that trades bandwidth for sensitivity:

| Property | Standard (802.11b/g/n) | Long Range (LR) |
|----------|------------------------|-----------------|
| **Data Rate** | 1-54 Mbps | 250-500 kbps |
| **Receiver Sensitivity** | -90 to -95 dBm | -100 to -105 dBm |
| **Sensitivity Gain** | Baseline | **+5 to +10 dB** |
| **Theoretical Range** | 100m (free space) | **200-300m (free space)** |
| **Interoperability** | Standard WiFi devices | **ESP32 LR only** |

### 2.2 Why This Fits Our Use Case

**Bandwidth requirement analysis:**

```
RTTPacket size:     90 bytes
Send rate:          10 Hz
Raw data rate:      90 × 10 × 8 = 7,200 bps = 7.2 kbps

LR mode capacity:   250,000 bps minimum
Utilization:        7.2 / 250 = 2.9%
```

**Conclusion:** We use <3% of LR mode's capacity. The bandwidth trade-off is completely acceptable.

### 2.3 Expected Range Improvement

Signal propagation follows the inverse-square law. Sensitivity gain translates to range:

| Sensitivity Gain | Range Multiplier | New Expected Range |
|------------------|------------------|-------------------|
| +6 dB | 2.0x | 10m |
| +8 dB | 2.5x | 12-13m |
| +10 dB | 3.2x | 15-16m |

**Conservative estimate:** 10-15 meter reliable range with LR mode.

**Important caveat:** LR mode improves weak signal detection but cannot overcome complete signal blockage. If the metal chassis fully blocks the signal path, no amount of sensitivity helps.

---

## 3. System Impact Analysis

### 3.1 Firmware Components Affected

| File | Layer | Change Required | Risk Level |
|------|-------|-----------------|------------|
| `EspNowTransport.cpp` | Transport (Main V2V Stack) | Add LR mode + TX power in `begin()` | Low |
| `sender_main.cpp` | RTT Test Firmware | Add LR mode + TX power in `initEspNow()` | Low |
| `reflector_main.cpp` | RTT Test Firmware | Add LR mode + TX power in `initEspNow()` | Low |

### 3.2 Components NOT Requiring Changes

| File | Layer | Why No Change Needed |
|------|-------|---------------------|
| `PackageManager.cpp` | Application | Operates above transport layer; receives already-decoded V2V messages via callbacks |
| `EspNowTransport.h` | Header | No API changes; LR mode is internal implementation detail |
| `RTTPacket.h` | Data Structure | Protocol mode doesn't affect packet format |
| `RTTBuffer.h`, `RTTLogging.h`, `RTTTimeout.h` | RTT Logic | Handle application logic, not RF configuration |
| Sensor drivers (MPU6500, GPS, etc.) | Hardware | Unrelated to WiFi/ESP-NOW |

### 3.3 Architecture Layer Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                      APPLICATION LAYER                           │
│  ┌─────────────────┐  ┌─────────────────┐                       │
│  │ PackageManager  │  │ RTTBuffer/      │  NO CHANGES NEEDED    │
│  │ (deduplication) │  │ RTTLogging/etc  │  (receive decoded     │
│  └────────┬────────┘  └────────┬────────┘   messages only)      │
│           │                    │                                 │
├───────────┼────────────────────┼─────────────────────────────────┤
│           ▼                    ▼       TRANSPORT LAYER           │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                    EspNowTransport.cpp                      ││
│  │                    (Main V2V Stack)                         ││
│  │                                                             ││
│  │    *** NEEDS LR MODE PATCH in begin() method ***            ││
│  └─────────────────────────────────────────────────────────────┘│
│                              OR                                  │
│  ┌─────────────────────┐  ┌─────────────────────┐               │
│  │  sender_main.cpp    │  │ reflector_main.cpp  │  STANDALONE   │
│  │  (RTT test sender)  │  │ (RTT test reflector)│  TEST FIRMWARE│
│  │                     │  │                     │               │
│  │  *** NEEDS PATCH ***│  │  *** NEEDS PATCH ***│               │
│  └─────────────────────┘  └─────────────────────┘               │
├─────────────────────────────────────────────────────────────────┤
│                      ESP-IDF / WiFi LAYER                        │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  WiFi.mode(WIFI_STA)                                        ││
│  │  WiFi.disconnect()                                          ││
│  │  esp_wifi_set_protocol(WIFI_IF_STA, WIFI_PROTOCOL_LR) ← NEW ││
│  │  esp_wifi_set_max_tx_power(84)                        ← NEW ││
│  │  esp_wifi_set_channel(...)                                  ││
│  │  esp_now_init()                                             ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

### 3.4 Critical Requirement: Both Devices Must Use LR

**WIFI_PROTOCOL_LR is a proprietary modulation.** A standard WiFi receiver cannot decode LR transmissions and vice versa.

```
Sender (LR) → Receiver (Standard) = NO COMMUNICATION
Sender (Standard) → Receiver (LR) = NO COMMUNICATION
Sender (LR) → Receiver (LR) = SUCCESS
```

**This is non-negotiable.** Both ESP32 devices MUST be flashed with LR-enabled firmware.

### 3.5 Protocol Compatibility Matrix

| V001 Mode | V002 Mode | V003 Mode | Result |
|-----------|-----------|-----------|--------|
| LR | LR | LR | All communicate |
| LR | Standard | LR | V002 cannot receive/send |
| Standard | Standard | Standard | Works (short range) |
| Mixed | Mixed | Mixed | **BROKEN - DO NOT MIX** |

### 3.6 No Impact On

- Packet structure (RTTPacket unchanged)
- ESP-NOW callbacks (same API)
- Logging format (CSV unchanged)
- ML emulator parameters (may need re-characterization)
- SD card operations

---

## 4. Implementation Details

### 4.1 Files Requiring Code Changes

Three files need identical LR mode patches:

| File | Function to Modify | Location (Line) |
|------|-------------------|-----------------|
| `roadsense-v2v/hardware/src/network/transport/EspNowTransport.cpp` | `begin()` | Lines 76-86 |
| `roadsense-v2v/hardware/test/espnow_rtt/sender_main.cpp` | `initEspNow()` | Lines 136-146 |
| `roadsense-v2v/hardware/test/espnow_rtt/reflector_main.cpp` | `initEspNow()` | Lines 79-89 |

### 4.2 Code Changes Required

All three files have the same initialization pattern. The patch is identical for each:

**Current code pattern (in all three files):**

```cpp
bool initEspNow() {
    logger.info("ESP-NOW", "Initializing ESP-NOW transport...");

    // Step 1: Set WiFi to station mode
    WiFi.mode(WIFI_STA);
    logger.debug("ESP-NOW", "WiFi mode: WIFI_STA");

    // Step 2: Disconnect from any saved WiFi networks
    WiFi.disconnect();
    logger.debug("ESP-NOW", "WiFi disconnected (prevents auto-connect)");

    // Step 3: Set WiFi channel  <-- CURRENT STEP 3
    esp_wifi_set_channel(ESPNOW_CHANNEL, WIFI_SECOND_CHAN_NONE);
    // ...
```

**Modified code (insert after WiFi.disconnect(), before esp_wifi_set_channel()):**

```cpp
bool initEspNow() {
    logger.info("ESP-NOW", "Initializing ESP-NOW transport...");

    // Step 1: Set WiFi to station mode
    WiFi.mode(WIFI_STA);
    logger.debug("ESP-NOW", "WiFi mode: WIFI_STA");

    // Step 2: Disconnect from any saved WiFi networks
    WiFi.disconnect();
    logger.debug("ESP-NOW", "WiFi disconnected (prevents auto-connect)");

    // =========================================================================
    // Step 2.5: Enable Long Range (LR) Mode and Max TX Power
    // =========================================================================
    // WIFI_PROTOCOL_LR: Proprietary Espressif mode (~250-500kbps)
    // - Increases receiver sensitivity by +5-10 dB
    // - Potentially doubles/triples range compared to standard 802.11
    // - CRITICAL: Both sender and receiver MUST use LR mode
    //
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
    // =========================================================================

    // Step 3: Set WiFi channel
    esp_wifi_set_channel(ESPNOW_CHANNEL, WIFI_SECOND_CHAN_NONE);
    logger.info("ESP-NOW", "WiFi channel set to " + String(ESPNOW_CHANNEL));
    // ... rest unchanged
```

### 4.3 Include Files

All three files already include the required header:
```cpp
#include <esp_wifi.h>  // Already present in all files
```

No additional includes needed.

### 4.4 Build Configuration

No changes to `platformio.ini` required. The ESP-IDF functions are part of the standard ESP32 Arduino core.

---

## 5. Post-Implementation: Re-Characterization Required

### 5.1 Why Re-characterization is Needed

The current `emulator_params_5m.json` was measured with standard protocol at 5m. After enabling LR mode:

1. **New range capability** - Test at 5m, 10m, 15m, 20m
2. **Latency may change** - LR mode uses different modulation timing
3. **Loss characteristics may differ** - Different error correction behavior
4. **Optimal distance needs discovery** - Find sweet spot for convoy operation

### 5.2 Re-characterization Test Plan

After flashing both devices with LR-enabled firmware:

| Test | Distance | Duration | Success Criteria |
|------|----------|----------|------------------|
| Baseline | 5m | 2 min | Loss ≤15% (same or better than before) |
| Extended 1 | 10m | 2 min | Loss ≤25% |
| Extended 2 | 15m | 2 min | Loss ≤35% |
| Extended 3 | 20m | 2 min | Loss ≤50% (usable for degraded mode) |
| Optimal | 10m | 10 min | Full characterization for new params |

### 5.3 Expected Output

If LR mode works as expected, generate new emulator parameters:

```
ml/espnow_emulator/emulator_params_LR_10m.json
```

Compare against current `emulator_params_5m.json` to validate improvement.

---

## 6. Post-Mortem: Why Was This Missed?

### 6.1 Root Cause Analysis

This oversight reveals several process gaps that should be addressed for future embedded projects:

#### A. Assumption of "Default is Good Enough"

**What happened:** ESP-NOW documentation mentions "up to 200m+ in open air." We assumed the default protocol would provide sufficient range.

**Reality:** The 200m figure is for:
- Free space (no obstacles)
- External antennas with good gain
- Not inside a metal vehicle chassis

**Lesson:** Always question "typical" performance claims against your specific deployment environment.

#### B. Indoor Testing Masked the Problem

**What happened:** Initial validation tests at 1m distance worked perfectly. Home tests at 1m also worked.

**Reality:** The range limitation only became apparent during field testing at realistic vehicle distances.

**Lesson:** Test at deployment-realistic conditions early. A "hardware validation checklist" should include range testing before field deployment.

#### C. TDD Cannot Catch RF Issues

**What happened:** Our TDD approach produced 42 passing tests covering packet handling, buffer management, timeout logic, and logging - all of which worked correctly.

**Reality:** Unit tests verify software logic, not RF propagation physics.

**Lesson:** TDD is necessary but not sufficient for embedded systems. Hardware characterization tests must be separate from unit tests.

#### D. LR Mode is Not Well-Publicized

**What happened:** Most ESP-NOW tutorials (Arduino examples, PlatformIO guides, YouTube videos) use default WiFi protocol. LR mode is documented but not prominently featured.

**Reality:** ESP-IDF documentation mentions WIFI_PROTOCOL_LR but it's in the WiFi section, not the ESP-NOW section.

**Lesson:** When working with resource-constrained wireless protocols, always check for "range extension" or "long range" modes in the vendor documentation.

#### E. No Pre-Deployment RF Checklist

**What happened:** We had checklists for software (TDD tests, code review) but not for RF deployment.

**Missing checklist items:**
- [ ] TX power maximized
- [ ] Protocol mode appropriate for range requirements
- [ ] Antenna placement optimized
- [ ] Distance-vs-loss characterized before field use
- [ ] Both devices using identical RF configuration

### 6.2 Process Improvements Going Forward

1. **Add RF Configuration Section to Hardware Setup Guides**
   - Document TX power settings
   - Document protocol mode selection
   - Include range expectations with PCB vs external antennas

2. **Create "RF Deployment Checklist"**
   - Similar to software test checklists
   - Must be completed before field testing

3. **Early Range Testing**
   - Test at 10m, 20m, 50m indoors before any field deployment
   - Identify range limitations before driving to remote test locations

4. **Document ESP-IDF "Hidden" Features**
   - WIFI_PROTOCOL_LR
   - esp_wifi_set_max_tx_power()
   - Other range/reliability optimizations

---

## 7. Files to Update After Implementation

### 7.1 Firmware Files

| File | Action |
|------|--------|
| `roadsense-v2v/hardware/src/network/transport/EspNowTransport.cpp` | Add LR mode code in `begin()` |
| `roadsense-v2v/hardware/test/espnow_rtt/sender_main.cpp` | Add LR mode code in `initEspNow()` |
| `roadsense-v2v/hardware/test/espnow_rtt/reflector_main.cpp` | Add LR mode code in `initEspNow()` |

### 7.2 Documentation Files

| File | Action |
|------|--------|
| `docs/20_KNOWLEDGE_BASE/NETWORK_CHARACTERIZATION_GUIDE.md` | Add LR mode section |
| `docs/10_PLANS_ACTIVE/RTT_Implementation/RTT_PROGRESS_TRACKER.md` | Update status |
| `docs/00_ARCHITECTURE/ESPNOW_EMULATOR_DESIGN.md` | Note LR mode impact |
| This file | Update status to COMPLETED |

### 7.3 Future Firmware Templates

Any future ESP-NOW firmware should use LR mode by default:
- `src/main.cpp` (main V2V firmware when implemented)
- Any new vehicle unit firmware
- Test harnesses using ESP-NOW

---

## 8. Checklist

### 8.1 Implementation Checklist

**Firmware Changes:**
- [ ] Modify `EspNowTransport.cpp` begin() with LR mode + TX power
- [ ] Modify `sender_main.cpp` initEspNow() with LR mode + TX power
- [ ] Modify `reflector_main.cpp` initEspNow() with LR mode + TX power

**Build Verification:**
- [ ] Build RTT test firmware: `pio run -e rtt_sender && pio run -e rtt_reflector`
- [ ] Build main V2V firmware (if applicable): `pio run -e esp32dev`

**Flash & Verify:**
- [ ] Flash sender (V001) on Linux
- [ ] Flash reflector (V002) on Windows
- [ ] Verify serial logs show "Long Range (LR) mode ENABLED" on both devices
- [ ] Verify serial logs show "TX power set to maximum" on both devices

### 8.2 Validation Checklist

- [ ] Test at 5m: Packet loss ≤15% (baseline, should not regress)
- [ ] Test at 10m: Packet loss ≤25% (improvement over current ~75%)
- [ ] Test at 15m: Packet loss ≤40% (major improvement)
- [ ] Test at 20m: Document results (any success is improvement)

### 8.3 Post-Validation Checklist

- [ ] If improved: Generate `emulator_params_LR_10m.json` from best distance
- [ ] Update `NETWORK_CHARACTERIZATION_GUIDE.md` with LR mode instructions
- [ ] Update `RTT_PROGRESS_TRACKER.md` status
- [ ] Mark this plan as COMPLETED

---

## 9. Rollback Plan

If LR mode causes unexpected issues:

1. **Remove the LR mode lines** from all three firmware files:
   - `EspNowTransport.cpp`
   - `sender_main.cpp`
   - `reflector_main.cpp`
2. **Re-flash all devices** with standard protocol
3. **Document the failure** (what went wrong, error codes, observed behavior)
4. **Escalate to hardware solution** (external antennas)

LR mode should not cause any compatibility issues since all devices are using the same change, but if serial output shows communication failures or unexpected behavior, rolling back is straightforward.

---

## 10. References

- [ESP-IDF WiFi API - Long Range Mode](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-guides/wifi.html#lr)
- [ESP-NOW Programming Guide](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/network/esp_now.html)
- [ESP32 TX Power Settings](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/network/esp_wifi.html#_CPPv425esp_wifi_set_max_tx_power6int8_t)

---

**Last Updated:** January 10, 2026
**Status:** PLANNED - Awaiting implementation approval
**Next Action:** Apply patches to EspNowTransport.cpp, sender_main.cpp, and reflector_main.cpp
