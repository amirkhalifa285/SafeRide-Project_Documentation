# Phase 8 Implementation Plan: Domain Randomization & Lifecycle Management

**Document Type:** AI Agent Implementation Prompt
**Target Component:** `ESPNOWEmulator` (Randomization & Reset Logic)
**Methodology:** Verification-First TDD
**Priority:** High (Required for Robust RL Training)

---

## 1. Objective
Verify and harden the `reset()` and `_randomize_episode_params()` methods.
While some logic exists, it must be **rigorously tested** to ensure:
1.  **State Hygiene:** No data leaks between episodes (critical for RL).
2.  **Parameter Diversity:** The agent actually encounters varied environments.
3.  **Determinism:** Experiments must be reproducible via seeding.

---

## 2. Technical Requirements

### 2.1 State Cleanup (`reset`)
The `reset()` method must act as a "hard wipe". The following **MUST** be cleared:
-   `pending_messages` (PriorityQueue): Must be empty.
-   `last_received` (Dict): Must be empty.
-   `last_loss_state` (Dict): Must be empty (stop burst loss carrying over).
-   `msg_counter` (int): Must reset to 0 (prevent integer overflow in infinite training).
-   `current_time_ms`: Reset to 0.

### 2.2 Domain Randomization Logic
When `domain_randomization=True`:
-   **Latency Base:** Sampled uniformly from `latency_range_ms`.
-   **Loss Base:** Sampled uniformly from `loss_rate_range`.
-   **Sampling Frequency:** MUST happen exactly once per `reset()`.

When `domain_randomization=False`:
-   **Latency Base:** Locked to `latency.base_ms`.
-   **Loss Base:** Locked to `packet_loss.base_rate`.
-   **Constraint:** Values must NOT change across resets.

### 2.3 Reproducibility (Seeding)
The emulator relies on Python's global `random` state. We must verify that `random.seed(N)` produces identical sequences of "random" parameters across runs. This is non-negotiable for debugging RL policies.

---

## 3. TDD Plan: Write These Tests FIRST

Create `ml/tests/test_domain_randomization.py`.

### Test Group A: State Hygiene (The "Clean Slate" Test)
1.  **`test_reset_clears_all_state`**:
    *   Setup: Pump data into emulator (messages in queue, received msgs, active burst loss state, incremented msg_counter).
    *   Action: Call `reset()`.
    *   Verify:
        *   Queue is empty.
        *   `last_received` is empty.
        *   `last_loss_state` is empty.
        *   `msg_counter` == 0.

### Test Group B: Randomization Logic
2.  **`test_dr_enabled_variability`**:
    *   Setup: `domain_randomization=True`.
    *   Action: Loop 50 times, call `reset()`, collect `episode_latency_base`.
    *   Verify: `len(unique_values) > 1` (It varies).
    *   Verify: All values fall within `latency_range_ms`.

3.  **`test_dr_disabled_consistency`**:
    *   Setup: `domain_randomization=False`.
    *   Action: Loop 50 times, call `reset()`.
    *   Verify: `episode_latency_base` is EXACTLY `params['latency']['base_ms']` every time.

4.  **`test_seeding_reproducibility`**:
    *   Action:
        1. `random.seed(42)` -> `reset()` -> Record params A.
        2. `random.seed(42)` -> `reset()` -> Record params B.
    *   Verify: `Params A == Params B`. (Critical for deterministic replay).

---

## 4. Execution Guidelines

1.  **Do NOT trust the existing implementation.** It may have been written hastily.
2.  **Strict Bounds Checking:** Ensure that even with randomization, we don't generate invalid parameters (e.g., negative latency if range is bad).
3.  **Logging:** The `reset()` method currently does not log. Consider adding an optional debug log showing the chosen parameters for the episode.

---

**Output:**
-   `ml/tests/test_domain_randomization.py`
-   (Optional) Refactored `espnow_emulator.py` if bugs are found.
