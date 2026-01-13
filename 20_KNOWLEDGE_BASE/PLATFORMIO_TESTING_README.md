# RoadSense V2V Unit Tests

This directory contains the unit tests for the RoadSense V2V firmware, implemented using the [Unity](http://www.throwtheswitch.org/unity) framework via PlatformIO.

## Test Organization

| Directory | Description | Key Test Files |
|-----------|-------------|----------------|
| `test_network/` | Network layer tests (ESP-NOW, Protocol) | `test_package_manager.cpp`, `test_v2v_message.cpp`, `test_esp_now_transport.cpp` |
| `test_sensors/` | Hardware driver tests (IMU, GPS) | `test_mpu6500_driver.cpp`, `test_neo6m_driver.cpp` |
| `test_utils/` | Utility component tests | `test_logger.cpp`, `test_mac_helper.cpp` |
| `test_integration/` | Multi-component integration tests | `test_sensor_integration.cpp`, `test_full_system.cpp` |

## Running Tests

### Run All Tests (Unit + Hardware)
```bash
pio test
```

### Run Specific Test Suite
```bash
# Run only Logger tests (folder name)
pio test -f test_utils

# Run with specific upload port (useful for Linux/Fedora)
pio test -f test_utils --upload-port /dev/ttyUSB0
```

### Run Integration Tests
Integration tests are skipped by default (as configured in `platformio.ini`). To run them:

```bash
pio test -e esp32dev_integration
```

## Writing Tests

See `test_logger.cpp` for a simple example of how to structure a test file.

Each test file must:
1. Include `<unity.h>`
2. Include the header of the component being tested
3. Implement `setUp()` and `tearDown()`
4. Implement test functions `void test_feature_name()`
5. Implement `setup()` calling `UNITY_BEGIN()` and `RUN_TEST()`
6. Implement empty `loop()`

## Troubleshooting

- **"A fatal error occurred: Failed to connect to ESP32"**: Check your USB connection and permissions (`/dev/ttyUSB0`).
- **Test failures**: Use `pio test -v` for verbose output to see exact assertion failures.
