# N-Element Deep Sets Implementation Progress Tracker

**Tracking Document:** `N_ELEMENT_PROGRESS_TRACKER.md`
**Parent Plan:** `../N_ELEMENT_IMPLEMENTATION_PLAN.md`
**Architecture:** `../../00_ARCHITECTURE/DEEP_SETS_N_ELEMENT_ARCHITECTURE.md`
**Status:** 🟡 IN PROGRESS

---

## 🛑 TDD Protocol (REQUIRED)

For every task below, the **BUILDER** must follow this sequence:
1.  **RED:** Create/Modify a test file (e.g., `tests/test_variable_peers.py`) that asserts the new behavior. **Run it. It MUST fail.**
2.  **GREEN:** Implement the minimum code to pass the test.
3.  **REFACTOR:** Clean up code, add comments, ensure type safety.
4.  **VERIFY:** Run the full test suite to ensure no regressions.

---

## Phase 1: ConvoyEnv & Observation Space (📍 CURRENT)

**Spec:** `PHASE_1_SPEC.md`
**Goal:** Convert `ConvoyEnv` from fixed `Box(11,)` to `Dict` space with variable peers.

- [ ] **1.1 Test Scaffolding**
    - [ ] Create `ml/tests/test_n_element_env.py`
    - [ ] Test `env.reset()` returns `Dict` keys: `ego`, `peers`, `peer_mask`
    - [ ] Test `env.observation_space` structure matches architecture

- [ ] **1.2 Observation Builder Update**
    - [ ] Modify `ml/envs/observation_builder.py` class
    - [ ] Implement `build(ego, peers_list)` -> `Dict`
    - [ ] Implement normalization and relative coordinate transformation
    - [ ] Verify `peer_mask` correctly identifies valid vs padded slots

- [ ] **1.3 ConvoyEnv Integration**
    - [ ] Update `ml/envs/convoy_env.py` observation space definition
    - [ ] Update `_step_espnow()` to loop over `traci.vehicle.getIDList()` (ignoring ego)
    - [ ] Remove hardcoded `V002`, `V003` references
    - [ ] Pass variable peer list to `ObservationBuilder`

---

## Phase 2: ESP-NOW Emulator Updates

**Goal:** Remove hardcoded vehicle IDs from the network emulation layer.

- [ ] **2.1 Emulator Refactoring**
    - [ ] Create test case: `ml/tests/test_emulator_n_peers.py` with 3+ peers
    - [ ] Modify `ml/espnow_emulator/espnow_emulator.py`
    - [ ] Replace `['V002', 'V003']` loop with dynamic dictionary tracking
    - [ ] Implement `get_all_peer_observations()`
    - [ ] Verify staleness/age logic works for dynamic peers

---

## Phase 3: Deep Sets Policy Network

**Goal:** Implement the PyTorch neural network that handles the set-based input.

- [ ] **3.1 Custom Feature Extractor**
    - [ ] Create `ml/policies/deep_set_policy.py`
    - [ ] Implement `DeepSetFeatureExtractor` (SB3 compatible)
    - [ ] Test: Permutation invariance (swapping peer order = same output)
    - [ ] Test: Masking (adding zero-padded peers = same output)

- [ ] **3.2 Policy Integration**
    - [ ] Create `create_deep_set_policy_kwargs()` helper
    - [ ] Verify shape compatibility with SB3 PPO

---

## Phase 4: Training Pipeline

**Goal:** End-to-end training loop validation.

- [ ] **4.1 Training Script**
    - [ ] Update `ml/training/train_convoy.py` to use `MultiInputPolicy`
    - [ ] Verify training starts and runs for 1000 steps without error
    - [ ] Verify TensorBoard logging

---

## Phase 5: Firmware Migration

**Goal:** Port the N-element logic to C++ for the ESP32.

- [ ] **5.1 Data Structures**
    - [ ] Create `roadsense-v2v/hardware/include/ml_types.h`
    - [ ] Define `MLObservation` struct and `PeerFeatures` struct

- [ ] **5.2 Inference Logic**
    - [ ] Create `roadsense-v2v/hardware/src/ml/ml_inference.cpp`
    - [ ] Implement "Shared Encoder" loop and "Max Pooling"
    - [ ] Integrate TFLite Micro calls

- [ ] **5.3 V2V Integration**
    - [ ] Create/Update `roadsense-v2v/hardware/src/network/v2v_receiver.cpp`
    - [ ] Populate `MLObservation` from received buffer

---

## Verification Log

| Date | Phase | Tests Passed | Notes |
|------|-------|--------------|-------|
|      |       |              |       |
