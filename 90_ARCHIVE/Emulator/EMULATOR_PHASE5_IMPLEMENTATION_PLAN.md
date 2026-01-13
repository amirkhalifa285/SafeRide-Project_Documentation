# Phase 5 Implementation Plan: Sensor Noise Modeling

**Document Type:** AI Agent Implementation Prompt
**Target Component:** `ESPNOWEmulator` (Sensor Noise Injection)
**Methodology:** Test-Driven Development (TDD)
**Priority:** High (Critical for preventing "God Mode" overfitting)

---

## 1. Objective
Implement the `_add_sensor_noise` method to inject Gaussian noise into the "perfect" V2V messages coming from SUMO. This forces the RL agent to learn robustness against sensor jitter (GPS wander, accelerometer vibration).

**Scope:** Minimal RL observation features only:
*   Position (`lat`, `lon`)
*   Dynamics (`speed`, `accel`)
*   Orientation (`heading`)
*   *Excluded:* Magnetometer, Gyroscope (not used in current RL architecture).

---

## 2. Technical Specification

### 2.1 The Noise Model
For each feature $x$, the noisy value $\hat{x}$ is:
$$ \hat{x} = x + \mathcal{N}(0, \sigma_{sensor}) $$

Where $\sigma_{sensor}$ is configured in `params['sensor_noise']`:
*   **GPS:** `gps_std_m` (Converted to degrees lat/lon).
*   **Speed:** `speed_std_ms`.
*   **Accel:** `accel_std_ms2`.
*   **Heading:** `heading_std_deg`.

### 2.2 GPS Conversion Logic
Noise is defined in *meters*, but data is in *degrees*.
*   Approximation: 1 degree latitude $\approx$ 111,000 meters.
*   $\sigma_{degrees} = \sigma_{meters} / 111000$.

---

## 3. TDD Plan: Write These Tests FIRST

Create `ml/tests/test_sensor_noise.py`.

### Test Group A: Noise Injection Logic
*Use `unittest.mock.patch` to control `random.gauss`.*

1.  `test_noise_gps_injection()`:
    *   Set `gps_std_m = 111.0` (for easy math, 111m ≈ 0.001 deg).
    *   Mock `random.gauss` to return $111.0$.
    *   Verify `lat` increases by exactly `0.001`.
2.  `test_noise_dynamics_injection()`:
    *   Set `speed_std_ms = 1.0`.
    *   Mock `random.gauss` to return `1.0`.
    *   Verify `speed` increases by 1.0.
    *   Verify `speed` is clamped (cannot be negative).
3.  `test_noise_heading_wraparound()`:
    *   Set `heading = 359`, noise = `+2`.
    *   Verify result is `1` (not 361).
    *   Set `heading = 1`, noise = `-2`.
    *   Verify result is `359` (not -1).

### Test Group B: Configuration Validation
4.  `test_noise_zero_std()`:
    *   Set all std devs to 0.
    *   Verify output equals input exactly.

---

## 4. Execution Instructions

1.  **Context**: Working in `roadsense-v2v/ml`.
2.  **Action 1**: Create `tests/test_sensor_noise.py`.
    *   **Constraint**: Use `pytest.approx()` for float comparisons.
    *   **Constraint**: Explicitly set parameters in test fixtures.
3.  **Action 2**: Run tests (expect failure initially or verifying existing implementation).
4.  **Action 3**: Refine `_add_sensor_noise` in `espnow_emulator.py` if needed (e.g., ensuring speed clamping and heading wraparound).
5.  **Action 4**: Update `docs/ESPNOW_EMULATOR_PROGRESS.md`.
