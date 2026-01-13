# ESP-NOW Network Characterization Guide

**Date:** December 23, 2025
**Purpose:** Measure real ESP-NOW performance for Python emulator calibration

---

## 📊 **What We're Measuring**

To build an accurate Python ESP-NOW emulator for SUMO training, we need **real-world measurements** of:

1. **Latency Distribution:** Time from TX to RX (mean, std_dev, p50, p95, p99)
2. **Packet Loss Rate:** Percentage of sent messages that never arrive
3. **Jitter:** Variance in latency (for realistic timing simulation)
4. **Distance Effects:** How latency/loss change at 5m, 50m, 100m
5. **Sensor Noise Floor:** Stationary baseline noise for GPS and Accelerometer (to calibrate Phase 5 Emulator)

---

## 🔧 **Hardware Setup**
...
## 📁 **Output File Format**

> ⚠️ **IMPORTANT:** The `DataLogger.cpp` currently only logs GPS and speed in Mode 1. Before running characterization, **Mode 1 must be updated** to include `accel_x, accel_y, accel_z` columns to support Phase 5 Emulator calibration.

### **TX Log (v001_tx_001.csv)**
```csv
timestamp_local_ms,msg_timestamp,vehicle_id,lat,lon,speed,heading,accel_x,accel_y,accel_z
1000,1000,V001,32.085300,34.781800,15.5,45.0,0.01,0.02,9.81
1100,1100,V001,32.085320,34.781820,15.7,45.2,0.05,-0.01,9.79
```

### **RX Log (v001_rx_001.csv)**
```csv
timestamp_local_ms,msg_timestamp,from_vehicle_id,lat,lon,speed,heading,accel_x,accel_y,accel_z
1052,1000,V002,32.086300,34.782800,14.8,225.0,-0.02,0.01,9.82
1151,1100,V002,32.086280,34.782780,14.9,224.5,0.03,0.04,9.80
```

**Key Columns:**
- `timestamp_local_ms` - Local millis() when event occurred
- `msg_timestamp` - Timestamp from V2VMessage (sender's clock)
- `vehicle_id` / `from_vehicle_id` - Who sent it

---

## 🐍 **Post-Processing Script**

### **Run Analysis (Python)**
```bash
cd ml/scripts/
python analyze_espnow_performance.py \
  --v001-tx ../data/v001_tx_001.csv \
  --v001-rx ../data/v001_rx_001.csv \
  --v002-tx ../data/v002_tx_001.csv \
  --v002-rx ../data/v002_rx_001.csv \
  --output emulator_params.json
```

### **Expected Output: `emulator_params.json`**
```json
{
  "latency_ms": {
    "mean": 23.5,
    "std_dev": 8.2,
    "min": 12,
    "max": 68,
    "p50": 22,
    "p95": 45,
    "p99": 62
  },
  "packet_loss_rate": 0.023,
  "jitter_ms": 8.2,
  "distance_model": {
    "0-50m": {"loss": 0.01, "latency_mean": 18},
    "50-100m": {"loss": 0.02, "latency_mean": 25},
    "100-150m": {"loss": 0.05, "latency_mean": 35}
  }
}
```

---

## 🎯 **Using Emulator Parameters in SUMO**

```python
# In SUMO environment
from espnow_emulator import ESPNOWEmulator

# Load measured parameters
emulator = ESPNOWEmulator.from_file("emulator_params.json")

# Apply to SUMO FCD output
for timestep in sumo_fcd:
    v002_message = get_v002_state(timestep)
    v003_message = get_v003_state(timestep)

    # Add realistic ESP-NOW effects
    v002_delivered = emulator.transmit(v002_message, distance=calc_distance(v001, v002))
    v003_delivered = emulator.transmit(v003_message, distance=calc_distance(v001, v003))

    # Log only delivered messages (realistic packet loss)
    if v002_delivered:
        log_received_message(v002_message, age=emulator.get_latency())
```

---

## ✅ **Success Criteria**

After characterization drive, you should have:
- [ ] 4 CSV files (2 TX, 2 RX)
- [ ] ~12,000-18,000 rows per file (10 Hz × 20-30 min)
- [ ] GPS coordinates valid (not all zeros)
- [ ] Packet delivery rate 85-98% (depending on distance)
- [ ] `emulator_params.json` generated successfully

---

## 🚨 **CRITICAL LESSONS LEARNED (January 9, 2026 Field Test)**

> **READ THIS BEFORE YOUR NEXT FIELD TEST**

### **What Went Wrong: 63.5% Packet Loss**

Our first field characterization drive produced **catastrophically high packet loss (63.5%)** instead of the expected 5-20%. This made the packet loss data **unusable for emulator calibration**.

### **Root Causes Identified**

| Issue | Impact | Fix |
|-------|--------|-----|
| **Inter-vehicle distance too large** | Signal attenuation | Keep 10-50m max |
| **PCB antenna inside metal car** | Signal blocked by chassis | Add external 2.4GHz antenna |
| **Military base RF interference** | 87-99% loss in certain zones | Avoid high-EMI areas |
| **ESP32 TX power not maximized** | Reduced range | Call `esp_wifi_set_max_tx_power(84)` |
| **No distance logging** | Cannot correlate loss vs distance | Log both vehicle GPS coordinates |

### **What Worked Well**

| Metric | Result | Usable? |
|--------|--------|---------|
| RTT Latency | 4-5ms mean, 7ms p95 | ✅ Excellent |
| IMU Noise | 0.4-0.9 m/s² std | ✅ Valid |
| Data Pipeline | 8726 rows logged correctly | ✅ Working |

### **GPS/Heading Noise Was Invalid**

The "stationary" noise measurements produced **195m GPS std** (should be 2-5m) because:
- Speed threshold (≤0.5 m/s) captured slow-moving traffic
- No true stationary calibration period

**Fix:** Add 60 seconds of parked calibration before starting drive.

### **Checklist for Next Field Test**

- [ ] **Distance:** Stay within 10-50m of other vehicle
- [ ] **Antennas:** External 2.4GHz dipole or patch antenna on both ESP32s
- [ ] **TX Power:** Verify `esp_wifi_set_max_tx_power(84)` in firmware
- [ ] **Route:** Avoid military bases, industrial areas, dense WiFi zones
- [ ] **Calibration:** 60s parked at start for stationary noise baseline
- [ ] **Distance Logging:** Have V002 also log GPS to compute inter-vehicle distance
- [ ] **Target Loss Rate:** Should be 5-20%, not 60%+
- [x] **Keep laptops awake:** Prevent sleep mode on BOTH laptops during test

---

## ✅ **STATIONARY TEST SUCCESS (January 10, 2026)**

### **Test Setup**
- **Location:** Abba Hillel Silver Road 28, Ramat Gan
- **Duration:** 22 minutes
- **Distances tested:** 20m → 9m → 5m progressively

### **Critical Discovery: PCB Antenna Range**

| Distance | Packet Loss | Verdict |
|----------|-------------|---------|
| **20m** | ~100% | ❌ Total failure |
| **9m** | 70-80% | ❌ Unusable |
| **5m** | **15.7%** | ✅ **USABLE** |

**Conclusion:** ESP32 DevKit PCB antenna only works reliably within **5-10 meters**.

### **Additional Root Cause: Laptop Sleep Mode**

The field drive failure was likely caused by **Windows laptop going to sleep**:
- Reflector ESP32 lost power when laptop slept
- Sender kept logging but received no echoes → appeared as 63% "loss"
- **Fix:** Keep laptops awake (play video, disable sleep, etc.)

### **Valid Data Obtained at 5m**

| Metric | Value | Status |
|--------|-------|--------|
| Packet Loss | 15.7% | ✅ Usable for emulator |
| GPS Noise | 3.63m std | ✅ Valid |
| IMU Noise | 0.07-0.17 m/s² | ✅ Excellent |
| RTT Latency | 6.74ms mean | ✅ Good |

### **Emulator Parameters Generated**

Valid parameters saved to: `ml/espnow_emulator/emulator_params_5m.json`

```json
{
  "packet_loss.base_rate": 0.08,      // 8% baseline
  "sensor_noise.gps_std_m": 3.63,     // Valid GPS noise
  "sensor_noise.accel_std_ms2": 0.12, // Valid IMU noise
  "latency.base_ms": 3.1              // One-way latency
}
```

---

## 🔧 **Troubleshooting**

### **Problem: GPS coordinates all zeros**
- **Cause:** GPS no lock
- **Fix:** Ensure clear sky view, wait 60+ seconds

### **Problem: Packet loss > 20%**
- **Cause:** Too far apart or bad antenna orientation
- **Fix:** Stay within 100m, ensure antennas not blocked
- **NEW:** Check for external interference (military bases, industrial RF)
- **NEW:** Consider external antennas if using inside vehicle

### **Problem: Packet loss > 50%**
- **Cause:** CRITICAL - Something fundamentally wrong
- **Fix:** DO NOT use this data for emulator calibration
- **Action:** Check distance, antennas, TX power, interference sources

### **Problem: GPS noise > 50m std**
- **Cause:** "Stationary" samples were actually moving
- **Fix:** Add explicit parked calibration period (60s minimum)
- **Fix:** Lower speed threshold or use variance-based detection

### **Problem: SD write failed errors**
- **Cause:** SD card full or corrupted
- **Fix:** Format SD card (FAT32), use Class 10 or better

### **Problem: Different number of TX/RX messages**
- **Cause:** Expected! This measures packet loss
- **Fix:** This is normal - emulator will model this

---

## 📚 **Next Steps**

1. ✅ Complete characterization drive (field test - issues found)
2. ✅ Complete stationary test (valid data at 5m)
3. ✅ Run `analyze_rtt_characterization.py`
4. ✅ Get valid `emulator_params_5m.json`
5. → Copy 5m params to main `emulator_params.json`
6. → Integrate emulator into SUMO pipeline
7. → Generate 50+ augmented training scenarios
8. → Train RL model
9. → (Optional) Re-test with external antennas for longer range

---

**Last Updated:** January 10, 2026
**Status:** ✅ Valid characterization data obtained at 5m | PCB antenna range limitation documented | Ready for emulator integration
