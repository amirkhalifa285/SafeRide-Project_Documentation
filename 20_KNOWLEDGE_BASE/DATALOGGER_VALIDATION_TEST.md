# DataLogger Validation Test Guide (At Home)

**Date:** December 29, 2025
**Purpose:** Verify DataLogger functionality before the field characterization drive.

## 1. Bench Test Objectives
- [ ] Verify SD card initialization at 4-10 MHz.
- [ ] Confirm file creation for `v001_tx_XXX.csv` and `v001_rx_XXX.csv`.
- [ ] Validate CSV column headers and data formatting.
- [ ] Ensure `millis()` timestamps are incrementing correctly.
- [ ] Test the "Log Toggle" button (GPIO 4) to start/stop sessions.

## 2. Setup
- **Hardware**: 1 ESP32 with SD card module + MicroSD card.
- **Power**: USB connection to laptop.
- **Software**: PlatformIO Serial Monitor (115200 baud).

## 3. Test Procedure
1.  **Preparation**:
    - Format MicroSD card as FAT32.
    - Insert card into the ESP32 module.
2.  **Flash Test Firmware**:
    - Build and upload the `test_datalogger` project.
    - Or flash the unified firmware in "Bench Mode".
3.  **Execution**:
    - Power on the ESP32.
    - Observe Serial Monitor for "SD card OK" message.
    - Press the button (GPIO 4) to start logging.
    - Wait 30 seconds.
    - Press the button again to stop logging.
4.  **Data Inspection**:
    - Remove SD card and insert into laptop.
    - Open the latest `.csv` files in the `logs/` directory.
    - **Verify Columns**: Ensure `accel_x, accel_y, accel_z` are present and populated.
    - **Verify Timing**: Check that rows are roughly 100ms apart (for 10Hz logging).

## 4. Troubleshooting
- **"SD card initialization failed"**: Check wiring (CS=5, SCK=18, MISO=19, MOSI=23) and ensure the card is FAT32.
- **"Failed to create file"**: Ensure the card is not write-protected and has space.
- **Zeroes in data**: Ensure IMU (MPU6500) is initialized before the logger starts.

## 5. Success Criteria
A successful test produces two CSV files with correct headers and at least 300 rows of valid, timestamped data.
