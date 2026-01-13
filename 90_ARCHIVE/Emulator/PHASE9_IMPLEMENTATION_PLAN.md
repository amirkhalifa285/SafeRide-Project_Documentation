# Phase 9 Implementation Plan: Integration & Sim2Real Validation

**Document Type:** AI Agent Implementation Prompt
**Target Component:** `ESPNOWEmulator` (Integration Tests)
**Methodology:** Statistical Verification TDD
**Priority:** High (Sim2Real Validation)

---

## 1. Objective
Validate that the `ESPNOWEmulator` accurately reproduces the statistical properties of the real-world ESP-NOW measurements. This phase moves beyond unit testing individual components to testing the **holistic behavior** of the system over long runs (Monte Carlo simulations).

**Key Question:** Does the emulator *actually* behave like the real radio?

---

## 2. Technical Requirements

### 2.1 Latency Distribution Validation
- **Requirement:** The observed latency distribution over N samples must match the theoretical distribution defined by the parameters.
- **Metrics:** Mean, Standard Deviation, Min/Max.
- **Tolerance:** ±5% for Mean/Std (Sim2Real gap tolerance).

### 2.2 Packet Loss Validation
- **Requirement:** The observed packet loss rate over N samples must converge to the expected probability.
- **Metrics:** `count(lost) / count(total)`.
- **Tolerance:** ±1% (e.g., if target is 2%, observed should be 1-3%).

### 2.3 Burst Loss Validation (Optional/Stretch)
- **Requirement:** If burst loss is enabled, the run length of consecutive losses should approximate a geometric distribution with mean `mean_burst_length`.

### 2.4 Domain Randomization Integration
- **Requirement:** Verify that running multiple episodes produces a distribution of *distributions*. (e.g., Episode A has mean latency 15ms, Episode B has mean latency 40ms).

---

## 3. TDD Plan: Write These Tests FIRST

Create `ml/tests/test_integration.py`.

### Test Group A: Statistical Correctness (Fixed Params)
1.  **`test_integration_latency_statistics`**:
    *   **Setup:** Disable DR. Set fixed params (Base=20, Dist=0, Jitter=5).
    *   **Action:** Simulate 10,000 transmissions.
    *   **Verify:**
        *   Observed Mean ≈ 20.0 (±0.5ms).
        *   Observed Std Dev ≈ 5.0 (±0.5ms).
        *   Observed Min >= 1.0ms.

2.  **`test_integration_packet_loss_rate`**:
    *   **Setup:** Disable DR. Set Loss=0.05 (5%).
    *   **Action:** Simulate 10,000 transmissions.
    *   **Verify:** Loss Count between 400 and 600 (4-6%).

### Test Group B: Distance Modeling Integration
3.  **`test_integration_distance_scaling`**:
    *   **Setup:** Disable DR. Set `distance_factor` = 0.1ms/m. Base=10ms.
    *   **Action:**
        *   Run 1000 tx at 0m.
        *   Run 1000 tx at 100m.
    *   **Verify:** `Mean(100m) - Mean(0m)` ≈ 10.0ms.

### Test Group C: Domain Randomization Integration
4.  **`test_integration_dr_variation`**:
    *   **Action:** Run 20 episodes. In each episode, run 100 tx and calculate mean latency.
    *   **Verify:** The standard deviation of the *means* is significant (> 0).
    *   **Verify:** All means fall within the DR range.

---

## 4. Execution Guidelines

1.  **Use Large N:** Statistical tests are flaky with small N. Use N >= 1000.
2.  **Seed Everything:** Set `seed=42` in `setUp` to ensure tests are deterministic and don't fail randomly in CI.
3.  **Performance:** These tests are slower. Mark them with `@pytest.mark.slow` if possible (optional).

---

**Output:**
-   `ml/tests/test_integration.py`
