# 3-Vehicle Network Characterization Plan

**Date:** January 17, 2026
**Status:** PLANNED - Pending LR Mode Implementation
**Priority:** HIGH - Required for emulator_params_LR_final.json
**Author:** Claude (with Amir)

---

## Executive Summary

This document analyzes whether using 3 vehicles for the final network characterization drive yields better data than the 2-vehicle RTT approach. **Conclusion: Yes, 3-vehicle characterization is definitively superior.**

The main V2V stack with DataLogger Mode 1 is already designed for multi-vehicle characterization - no new firmware development needed beyond the LR mode patch.

---

## 1. Context: ESP-NOW LR Mode Migration

The LR Mode Migration plan addresses a **critical range limitation**:

| Distance | Current Loss | Target with LR |
|----------|--------------|----------------|
| 5m | 15.7% | Baseline (should not regress) |
| 10m | 70-80% | ≤25% |
| 15m | ~100% | ≤40% |

**Key change:** Add `esp_wifi_set_protocol(WIFI_IF_STA, WIFI_PROTOCOL_LR)` + max TX power.

**Reference:** `ESPNOW_LONG_RANGE_MODE_MIGRATION.md`

---

## 2. Analysis: Is 3-Vehicle Characterization Better?

### Answer: **YES, significantly better**

### Comparison Table

| Aspect | 2-Vehicle RTT | 3-Vehicle Convoy |
|--------|--------------|------------------|
| **Links measured** | 1 (V001↔V002) | 6 directional links |
| **Firmware used** | Test (sender/reflector) | Production (main.cpp) |
| **Spatial diversity** | None | Different distances/angles |
| **Real-world realism** | Ping-pong test | Actual convoy scenario |
| **Network load** | Light (10 Hz one-way) | Production (30 Hz total) |
| **Burst collision data** | None | Multiple simultaneous TX |
| **Matches deployment** | No | Yes (Deep Sets trains on n peers) |

### The 6 Directional Links

```
V001 ←────→ V002    (links 1-2)
 ↑            ↑
 │            │
 └────────────┼─────→ V003    (links 3-4, 5-6)
```

Each link has TWO directions (A→B may differ from B→A due to antenna orientation).

---

## 3. Critical Finding: Use Main V2V Stack, NOT RTT Firmware

### Two Firmware Options Exist:

| Option | Files | Purpose | 3-Vehicle Support |
|--------|-------|---------|-------------------|
| **RTT Test Firmware** | `sender_main.cpp`, `reflector_main.cpp` | Simple RTT ping-pong | **NO** (2 devices only) |
| **Main V2V Stack** | `main.cpp` + `DataLogger.cpp` | Production convoy | **YES** (built for this) |

### Why RTT Firmware Doesn't Work for 3 Vehicles:

1. **Ambiguous echoes:** If V002 AND V003 both reflect, V001 receives two responses - can't distinguish
2. **No peer identification:** RTT packet doesn't include reflector ID
3. **Designed for 1:1 only:** Sender assumes single reflector

### Main V2V Stack Already Supports:

1. **Broadcast to all peers** - ESP-NOW FF:FF:FF:FF:FF:FF
2. **Receive from multiple peers** - PackageManager tracks up to 20 MACs
3. **Mode 1 TX/RX logging** - Separate CSV files for sent vs received
4. **Timestamps for latency** - `local_timestamp - msg_timestamp = latency`
5. **Vehicle ID in messages** - Can distinguish V002 vs V003 packets

---

## 4. Data Collection: What Each Vehicle Logs

### Mode 1 Output Files

```
V001:
├── v001_tx_001.csv  (what V001 sent: ~600 msgs/min @ 10Hz)
└── v001_rx_001.csv  (what V001 received from V002 + V003: ~1200 msgs/min)

V002:
├── v002_tx_001.csv  (what V002 sent)
└── v002_rx_001.csv  (what V002 received from V001 + V003)

V003:
├── v003_tx_001.csv  (what V003 sent)
└── v003_rx_001.csv  (what V003 received from V001 + V002)
```

### CSV Format (Mode 1)

**TX file:**
```csv
timestamp_local_ms,msg_timestamp,vehicle_id,lat,lon,speed,heading,accel_x,accel_y,accel_z
12345,12340,V001,40.123,-75.456,5.2,45.0,0.1,-0.2,9.8
```

**RX file:**
```csv
timestamp_local_ms,msg_timestamp,from_vehicle_id,lat,lon,speed,heading,accel_x,accel_y,accel_z
12350,12340,V002,40.124,-75.455,5.1,44.8,0.2,-0.1,9.7
```

**Latency calculation:** `timestamp_local_ms - msg_timestamp = 10ms`

---

## 5. Data Pooling and Analysis

### Step 1: Collect 6 Files from SD Cards

After the drive, retrieve:
- `v001_tx_001.csv`, `v001_rx_001.csv`
- `v002_tx_001.csv`, `v002_rx_001.csv`
- `v003_tx_001.csv`, `v003_rx_001.csv`

### Step 2: Compute Per-Link Latency

```python
# Example: V002 → V001 latency
v002_tx = pd.read_csv("v002_tx_001.csv")
v001_rx = pd.read_csv("v001_rx_001.csv")

# Filter for messages from V002
v001_rx_from_v002 = v001_rx[v001_rx.from_vehicle_id == "V002"]

# Compute latency
latency = v001_rx_from_v002.timestamp_local_ms - v001_rx_from_v002.msg_timestamp
print(f"V002→V001 latency: mean={latency.mean():.1f}ms, std={latency.std():.1f}ms")
```

### Step 3: Compute Per-Link Packet Loss

```python
# V002 → V001 loss rate
v002_sent_count = len(v002_tx)
v001_received_from_v002 = len(v001_rx[v001_rx.from_vehicle_id == "V002"])

loss_rate = 1.0 - (v001_received_from_v002 / v002_sent_count)
print(f"V002→V001 loss rate: {loss_rate*100:.1f}%")
```

### Step 4: Aggregate for Emulator Parameters

**Options:**
- **Option A:** Use worst-case link (conservative)
- **Option B:** Use average across all 6 links (balanced)
- **Option C:** Use ego-centric links only (V001 RX) - **RECOMMENDED**

**Recommendation:** Option C - Use V001 RX data since V001 is the ego vehicle in Deep Sets training.

### Step 5: Generate emulator_params_LR_final.json

Modify `analyze_rtt_characterization.py` or write new pooling script.

---

## 6. Additional Benefits of 3-Vehicle Approach

### 6.1 Burst Loss Under Load
With 3 vehicles broadcasting at 10 Hz each, the network sees **30 packets/sec total**. This creates realistic collision/contention patterns that 2-vehicle testing misses.

### 6.2 Validate Deep Sets Training Data
Your Deep Sets model needs variable n peers. With 3 vehicles:
- V001 sees n=2 (V002, V003)
- This is **actual n=2 data**, not simulated

### 6.3 Antenna Orientation Effects
Real vehicles have different antenna orientations. 3 vehicles capture:
- Front-to-back (convoy following)
- Side-to-side (adjacent lanes)
- Diagonal (turning scenarios)

### 6.4 Distance Variation
GPS in each log allows:
- Distance reconstruction between any two vehicles
- Correlation of loss rate with actual measured distance
- Geographic loss zone identification

---

## 7. Recommended Workflow

### Phase A: Pre-Drive Preparation

1. **Flash LR mode firmware** to all 3 units:
   ```bash
   # Apply LR mode patch to EspNowTransport.cpp (see ESPNOW_LONG_RANGE_MODE_MIGRATION.md)
   pio run --target upload --upload-port /dev/ttyUSB0  # V001
   pio run --target upload --upload-port /dev/ttyUSB1  # V002
   pio run --target upload --upload-port /dev/ttyUSB2  # V003
   ```

2. **Verify serial logs show:**
   ```
   [INFO] ESP-NOW: Long Range (LR) mode ENABLED
   [INFO] ESP-NOW: TX power set to maximum (84 = ~21dBm)
   ```

3. **Test Mode 1 logging:**
   - Press button on each unit
   - Check SD cards create `vXXX_tx_001.csv` and `vXXX_rx_001.csv`
   - Verify files contain data

4. **Sync clocks:**
   - All units use `millis()` from boot
   - Press start buttons simultaneously (or note offset for correction)

### Phase B: Characterization Drive

1. **Start logging** on all 3 units (button press)
2. **Drive convoy formation:**
   - Vary distances: 10m, 20m, 30m gaps
   - Include straight roads, curves, intersections
   - Mix of speeds: 0-50 km/h
3. **Duration:** 20-30 minutes (~36K packets per vehicle)
4. **Stop logging** (button press or power off)

### Phase C: Post-Drive Analysis

1. Collect 6 CSV files from SD cards
2. Run pooling analysis script
3. Generate `emulator_params_LR_final.json`
4. Compare with `emulator_params_5m.json` baseline
5. Document results in `CLOUD_TRAINING_RUN_002_RESULTS.md`

---

## 8. Files Involved

### Firmware (Need LR Mode Patch)

| File | Change |
|------|--------|
| `hardware/src/network/transport/EspNowTransport.cpp` | Add LR mode + TX power |

### Already Working (No Changes)

| File | Purpose |
|------|---------|
| `hardware/src/main.cpp` | Mode 1 support exists |
| `hardware/src/logging/DataLogger.cpp` | TX/RX CSV logging |

### Analysis (May Need Update)

| File | Change |
|------|--------|
| `ml/scripts/analyze_rtt_characterization.py` | Modify to accept 6-file pooled input |

---

## 9. Checklist

### Pre-Drive
- [ ] LR mode patch applied to `EspNowTransport.cpp`
- [ ] Firmware flashed to V001, V002, V003
- [ ] Serial logs confirm LR mode enabled on all units
- [ ] Mode 1 logging tested (files created on SD)
- [ ] SD cards have sufficient space (~50MB each)

### During Drive
- [ ] All 3 units started simultaneously (note start time)
- [ ] Varied distances during drive (10m, 20m, 30m)
- [ ] Duration ≥20 minutes
- [ ] Drive included curves, intersections, speed changes

### Post-Drive
- [ ] 6 CSV files collected from SD cards
- [ ] Files renamed/organized by vehicle
- [ ] Pooling analysis script run
- [ ] `emulator_params_LR_final.json` generated
- [ ] Results compared with 5m baseline
- [ ] Documentation updated

---

## 10. Conclusion

**3-vehicle characterization is definitively better** because:

1. Uses **production firmware** (not test firmware)
2. Captures **6 directional links** instead of 1
3. Creates **realistic network load** (30 Hz vs 10 Hz)
4. Matches **deployment scenario** (convoy with multiple peers)
5. **No new firmware development** - just apply LR mode patch

The main V2V stack with DataLogger Mode 1 was designed for exactly this use case.

---

**Document Version:** 1.0
**Last Updated:** January 17, 2026
