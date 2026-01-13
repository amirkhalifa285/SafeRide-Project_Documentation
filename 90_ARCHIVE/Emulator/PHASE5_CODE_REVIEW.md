# Phase 5 Code Review: Sensor Noise Modeling

**Date:** December 28, 2025
**Reviewer:** Merciless Critic (via Gemini)
**Target:** `ml/espnow_emulator/espnow_emulator.py` and `ml/tests/test_sensor_noise.py`
**Status:** ⚠️ **APPROVED WITH REQUIRED CHANGES**

Phase 5 implementation is functionally correct but contains unacceptable "magic numbers" and test smells that violate project standards for maintainability and determinism.

---

## 🚨 Critical Issues (Must Fix)

### 1. Magic Numbers in `_add_sensor_noise`
**Severity:** High
**Location:** `espnow_emulator.py`, lines ~327-329

The code hardcodes gyroscope noise to `0.01` regardless of configuration:
```python
gyro_x=msg.gyro_x + random.gauss(0, 0.01),  # <--- MAGIC NUMBER
```
**Why this is bad:**
*   **Configuration Breakage:** If a user sets `sensor_noise` parameters to `0.0` (expecting perfect data for debugging), they will *still* get noisy gyro data.
*   **Determinism:** It makes it impossible to fully control the environment via the configuration file.

**Fix:**
1.  Add `gyro_std_rad_s` to `_default_params` (e.g., default `0.01`).
2.  Use `noise.get('gyro_std_rad_s', 0.01)` in `_add_sensor_noise`.

### 2. "Giving Up" in Tests
**Severity:** High
**Location:** `tests/test_sensor_noise.py`, `test_noise_zero_std`

The test explicitly documents that it ignores a failure instead of fixing the root cause:
```python
# Gyro fields have hardcoded 0.01 noise in implementation
# So we don't check exact equality for gyro
```
**This is Anti-TDD.** Never write a test that accepts broken behavior. The test exposed the bug (hardcoded noise), and the correct action was to fix the implementation, not relax the test.

**Fix:**
*   Once Issue #1 is fixed, update this test to ASSERT that gyro fields are identical when noise is set to 0.

### 3. Physics Accuracy (Longitude Scaling)
**Severity:** Medium
**Location:** `espnow_emulator.py`

You use `111,000` meters per degree for both Latitude and Longitude:
```python
lat=msg.lat + random.gauss(0, noise['gps_std_m'] / 111000),
lon=msg.lon + random.gauss(0, noise['gps_std_m'] / 111000),
```
**The Physics:**
*   Latitude: ~111km per degree (Constant-ish) -> **Correct**
*   Longitude: ~111km * cos(lat) (Varies significantly) -> **Incorrect**

At Tel Aviv latitude (~32°N), 1 degree longitude is **~94km**. Your model underestimates longitudinal noise by ~15%.

**Fix:**
*   Define `METERS_PER_DEG_LAT = 111000.0`.
*   Acknowledge the longitude approximation in a comment or implement `math.cos` scaling.
*   Replace the raw number `111000` with the constant.

---

## ⚠️ Code Quality Improvements

### 1. DRY Violation
The magic number `111000` appears twice.
**Fix:** Define a class constant `METERS_PER_DEG_LAT`.

### 2. Missing Default Parameter
The `gyro_std_rad_s` parameter is completely missing from `_default_params`, making the configuration schema incomplete.

---

## ✅ What Was Done Well
*   **Mocking:** `random.gauss` was mocked correctly in tests, ensuring deterministic execution.
*   **Defensive Coding:** `speed=max(0, ...)` is a good practice.
*   **Edge Cases:** `test_noise_heading_wraparound` correctly validates the modulo logic.

---

## 🛠️ Refactoring Instructions

1.  **Modify `espnow_emulator.py`**:
    *   Add `METERS_PER_DEG_LAT = 111000.0` class constant.
    *   Add `'gyro_std_rad_s': 0.01` to `_default_params`.
    *   Update `_add_sensor_noise` to use `self.METERS_PER_DEG_LAT` and `noise['gyro_std_rad_s']`.

2.  **Modify `tests/test_sensor_noise.py`**:
    *   Update `test_noise_zero_std` to include `'gyro_std_rad_s': 0.0` in the params.
    *   Uncomment/add assertions for gyro equality.
    *   Update `mock_gauss.side_effect` lists in other tests to account for the gyro noise calls (now that they are using the config).

**Action Required:** Proceed with these fixes immediately.
