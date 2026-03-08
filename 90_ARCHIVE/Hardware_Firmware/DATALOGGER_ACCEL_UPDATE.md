# DataLogger Accelerometer Update for Network Characterization

**Date:** December 29, 2025
**Priority:** HIGH
**Status:** Planned

## Objective
To build an accurate ESP-NOW emulator, we need to characterize not just the network latency and loss, but also the **sensor noise floor** during real driving conditions. This requires adding raw accelerometer data to the Network Characterization (Mode 1) logs.

## Current Mode 1 Format (Incomplete)
Currently, Mode 1 logs the following columns:
- `timestamp_local_ms`
- `msg_timestamp`
- `vehicle_id`
- `lat`
- `lon`
- `speed`
- `heading`

## Target Mode 1 Format (Updated)
The updated format will include 3-axis accelerometer data:
- `timestamp_local_ms`
- `msg_timestamp`
- `vehicle_id`
- `lat`
- `lon`
- `speed`
- `heading`
- **`accel_x`**
- **`accel_y`**
- **`accel_z`**

## Implementation Tasks
1.  **Modify `DataLogger.h`**: Update the header string for Mode 1 TX and RX logs.
2.  **Modify `DataLogger.cpp`**: Update `logTxMessage` and `logRxMessage` to extract and log accelerometer values from the `V2VMessage`.
3.  **Update `V2VMessage.h`**: Ensure the structure correctly holds sensor data (already verified as complete).
4.  **Verification**: Run the `test_datalogger` unit test and verify the CSV output format.

## Why This Matters
Training an RL agent in SUMO requires "synthesizing" sensor data. By measuring the real noise floor during the RTT drive, we can calibrate the `SensorSynthesizer` in the emulator to match the real MPU6500 performance, preventing "God Mode" overfitting.
