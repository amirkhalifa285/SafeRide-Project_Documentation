# Phase 4 Implementation Plan: Packet Loss Modeling

**Document Type:** AI Agent Implementation Prompt
**Target Component:** `ESPNOWEmulator` (Packet Loss Calculation)
**Methodology:** Test-Driven Development (TDD) / Verification of Existing Code
**Priority:** High (Critical for Sim2Real fidelity)

---

## 1. Objective
Verify and validate the `_get_loss_probability` and `_check_burst_loss` methods in `ESPNOWEmulator`. The packet loss model must accurately simulate:
1.  **Distance-Dependent Loss:** Low loss at close range, increasing linearly in medium range, and high loss at far range (tiered model).
2.  **Burst Loss:** Temporal correlation of packet loss (if enabled).
3.  **Domain Randomization:** Base loss rate variation across episodes.

---

## 2. Technical Specification

### 2.1 The Probability Model
The loss probability $P_{loss}$ at distance $d$ is defined by 3 tiers:

*   **Tier 1 (Close Range):** $d < T_1 \implies P = P_{base}$
*   **Tier 2 (Transition):** $T_1 \le d < T_2 \implies P = P_{base} + \frac{d - T_1}{T_2 - T_1} \times (P_{tier2} - P_{base})$
*   **Tier 3 (Far Range):** $d \ge T_2 \implies P = P_{tier3}$

Where:
*   $P_{base}$: Base loss rate (from params or randomized per episode).
*   $T_1, T_2$: Distance thresholds.
*   $P_{tier2}, P_{tier3}$: Loss rates for transition and far range.

### 2.2 Burst Loss Logic
If `burst_loss.enabled` is True:
*   If the *previous* packet from a specific vehicle was lost (`last_loss_state[id] == True`), then `_check_burst_loss` returns `True` with 50% probability (hardcoded in current implementation).
*   If `_check_burst_loss` returns `True`, the transmit logic increases $P_{loss}$ significantly (e.g., $P_{loss} = \min(0.8, P_{loss} \times 3)$).

---

## 3. TDD Plan: Write These Tests FIRST

Create `ml/tests/test_packet_loss_modeling.py`.

### Test Group A: Distance-Dependent Probability
*Explicitly set parameters in `setUp` or test body to avoid dependency on defaults.*
*   `base_rate` = 0.01
*   `threshold_1` = 50, `threshold_2` = 100
*   `rate_tier_2` = 0.10, `rate_tier_3` = 0.50

1.  `test_loss_tier_1_close_range()`:
    *   Distance = 10m ($< T_1$).
    *   Verify probability == `base_rate`.
2.  `test_loss_tier_2_interpolation()`:
    *   Distance = 75m (midpoint between 50 and 100).
    *   Verify probability == approx 0.055 (midpoint between 0.01 and 0.10).
3.  `test_loss_tier_3_far_range()`:
    *   Distance = 150m ($> T_2$).
    *   Verify probability == `rate_tier_3`.
4.  `test_loss_boundary_conditions()`:
    *   Test exactly at $d=50$ and $d=100$.

### Test Group B: Burst Loss Logic
5.  `test_burst_loss_disabled()`:
    *   Set `enabled=False`.
    *   Inject `last_loss_state = True`.
    *   Verify `_check_burst_loss` returns `False`.
6.  `test_burst_loss_enabled_logic()`:
    *   Set `enabled=True`.
    *   Inject `last_loss_state = True`.
    *   Mock `random.random` to return 0.1 (< 0.5).
    *   Verify `_check_burst_loss` returns `True`.
    *   Mock `random.random` to return 0.9 (> 0.5).
    *   Verify `_check_burst_loss` returns `False`.
7.  `test_burst_loss_no_history()`:
    *   Inject `last_loss_state = False` or empty.
    *   Verify `_check_burst_loss` returns `False`.

### Test Group C: Domain Randomization
8.  `test_loss_domain_randomization()`:
    *   Enable DR.
    *   Reset emulator multiple times.
    *   Verify `_get_loss_probability(0)` varies across episodes within `loss_rate_range`.

---

## 4. Execution Instructions

1.  **Context**: Working in `roadsense-v2v/ml`.
2.  **Action 1**: Create `tests/test_packet_loss_modeling.py` with the defined tests.
    *   **Constraint**: Use `pytest.approx()` for float comparisons.
    *   **Constraint**: Explicitly set parameters in tests (do not rely on defaults).
3.  **Action 2**: Run tests.
4.  **Action 3**: If tests fail, fix the implementation in `espnow_emulator.py`.
5.  **Action 4**: Update `docs/ESPNOW_EMULATOR_PROGRESS.md` to mark Phase 4 as complete.
