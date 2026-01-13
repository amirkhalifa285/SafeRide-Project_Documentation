# Phase 6 Implementation Plan: Core Transmission & Observation Logic

**Document Type:** AI Agent Implementation Prompt
**Target Component:** `ESPNOWEmulator` (Core Logic)
**Methodology:** Test-Driven Development (TDD)
**Priority:** Critical (Core Functionality)

---

## 1. Objective
Implement and verify the core `transmit()` and `get_observation()` methods. This phase bridges the gap between the physics models (Latency/Loss/Noise) and the RL agent interface.

**Critical Goal:** prevent **Causality Violations**. The agent must NEVER see a message before `current_time + latency`.

---

## 2. Technical Specification

### 2.1 State Management
The emulator maintains three critical state structures:
1.  `pending_messages: PriorityQueue`: Stores `(arrival_time, vehicle_id, message)`. Sorted by arrival time.
2.  `last_received: Dict[str, ReceivedMessage]`: The latest *delivered* message for each vehicle.
3.  `last_loss_state: Dict[str, bool]`: Tracks if the last packet from `vehicle_id` was lost (for burst modeling).

### 2.2 Transmit Logic (`transmit`)
*   **Input:** Sender state (perfect), positions, time.
*   **Physics:** Calculate Distance -> Loss Prob -> Latency -> Noise.
*   **Action:**
    *   If Lost: Return `None`, update `last_loss_state[id] = True`.
    *   If Success:
        *   Create `ReceivedMessage` with `received_at_ms = now + latency`.
        *   **CRITICAL:** Push to `pending_messages`. Do **NOT** update `last_received` yet.
        *   Update `last_loss_state[id] = False`.
        *   Return `ReceivedMessage` (for logging only).

### 2.3 Observation Logic (`get_observation`)
*   **Input:** Ego speed, current time.
*   **Action:**
    *   **Process Queue:** Pop messages from `pending_messages` where `arrival_time <= current_time`. Update `last_received`.
    *   **Build Dict:** For each vehicle (V002, V003):
        *   If in `last_received`:
            *   Calculate `age = current_time - received_at + transmission_latency`.
            *   `valid = age < 500`.
        *   Else: Zero fill, `valid = False`.

---

## 3. TDD Plan: Write These Tests FIRST

Create `ml/tests/test_core_logic.py`.

### Test Group A: Transmission Logic (`transmit`)
1.  `test_transmit_packet_loss()`:
    *   Force loss (mock `random.random`).
    *   Verify returns `None`.
    *   Verify `pending_messages` is empty.
    *   Verify `last_loss_state` is True.
2.  `test_transmit_success()`:
    *   Force success.
    *   Verify returns `ReceivedMessage`.
    *   Verify `pending_messages` has 1 item.
    *   Verify `last_loss_state` is False.
    *   **CRITICAL:** Verify `last_received` is EMPTY (causality check).
3.  `test_transmit_latency_application()`:
    *   Mock latency to 50ms.
    *   Send at T=1000.
    *   Verify `received_at_ms` in returned message is 1050.
    *   Verify queue item priority is 1050.

### Test Group B: Observation Logic (`get_observation`)
4.  `test_observation_causality()`:
    *   Queue message to arrive at T=1050.
    *   Call `get_observation(time=1049)`.
    *   Verify `last_received` is still empty.
    *   Verify observation shows `valid=False` (or zeros).
5.  `test_observation_delivery()`:
    *   Queue message to arrive at T=1050.
    *   Call `get_observation(time=1050)`.
    *   Verify `last_received` is updated.
    *   Verify observation matches message data.
6.  `test_observation_staleness()`:
    *   Deliver message at T=1000 (Latency=50).
    *   Call `get_observation(time=1600)`.
    *   `age` should be `1600 - 1000 + 50 = 650`.
    *   Verify `valid=False` (age > 500).

---

## 4. Execution Instructions

1.  **Context**: Working in `roadsense-v2v/ml`.
2.  **Action 1**: Create `tests/test_core_logic.py` with the tests above.
3.  **Action 2**: Run tests. They might pass (due to previous refactor), but verify coverage.
4.  **Action 3**: Refine `espnow_emulator.py` if needed to ensure robustness.
5.  **Action 4**: Update `docs/ESPNOW_EMULATOR_PROGRESS.md`.
