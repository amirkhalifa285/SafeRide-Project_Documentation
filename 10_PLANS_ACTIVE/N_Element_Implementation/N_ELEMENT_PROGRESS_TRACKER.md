# N-Element Deep Sets Implementation Progress Tracker

**Tracking Document:** `N_ELEMENT_PROGRESS_TRACKER.md`
**Parent Plan:** `../N_ELEMENT_IMPLEMENTATION_PLAN.md`
**Architecture:** `../../00_ARCHITECTURE/DEEP_SETS_N_ELEMENT_ARCHITECTURE.md`
**Status:** 🟢 Phase 3 & 4 COMPLETE (2026-01-15) — Training Pipeline Ready

---

## 🛑 TDD Protocol (REQUIRED)

For every task below, the **BUILDER** must follow this sequence:
1.  **RED:** Create/Modify a test file (e.g., `tests/test_variable_peers.py`) that asserts the new behavior. **Run it. It MUST fail.**
2.  **GREEN:** Implement the minimum code to pass the test.
3.  **REFACTOR:** Clean up code, add comments, ensure type safety.
4.  **VERIFY:** Run the full test suite to ensure no regressions.

---

## Phase 1: ConvoyEnv & Observation Space ✅ COMPLETE

**Spec:** `PHASE_1_SPEC.md`
**Builder Report:** `PHASE_1_BUILDER_REPORT.md`
**Review:** `PHASE_1_REVIEW.md` — **APPROVED** (2026-01-15)
**Goal:** Convert `ConvoyEnv` from fixed `Box(11,)` to `Dict` space with variable peers.

- [x] **1.1 Test Scaffolding**
    - [x] Create `ml/tests/unit/test_n_element_env.py`
    - [x] Test `env.reset()` returns `Dict` keys: `ego`, `peers`, `peer_mask`
    - [x] Test `env.observation_space` structure matches architecture

- [x] **1.2 Observation Builder Update**
    - [x] Modify `ml/envs/observation_builder.py` class
    - [x] Implement `build(ego_state, peer_observations, ego_pos)` -> `Dict`
    - [x] Implement normalization and relative coordinate transformation
    - [x] Verify `peer_mask` correctly identifies valid vs padded slots

- [x] **1.3 ConvoyEnv Integration**
    - [x] Update `ml/envs/convoy_env.py` observation space definition
    - [x] Update `_step_espnow()` to loop over `traci.vehicle.getIDList()` (ignoring ego)
    - [x] Remove hardcoded `V002`, `V003` references
    - [x] Pass variable peer list to `ObservationBuilder`

- [x] **1.4 Post-Review Fix** (2026-01-15)
    - [x] Added `clear()` method to `ESPNOWEmulator` for API consistency
    - [x] Simplified `convoy_env.py` to use `emulator.clear()` directly

**Tests:** 74/74 unit tests pass

---

## Phase 2: Deep Sets Policy Network ✅ COMPLETE

**Spec:** `PHASE_2_SPEC.md`
**Builder Report:** `PHASE_2_BUILDER_REPORT.md`
**Review:** `PHASE_2_REVIEW.md` — **APPROVED** (2026-01-15)
**Goal:** Implement the PyTorch neural network that handles the set-based input.

- [x] **2.1 Custom Feature Extractor**
    - [x] Create `ml/models/deep_set_extractor.py`
    - [x] Implement `DeepSetExtractor` (SB3 `BaseFeaturesExtractor` subclass)
    - [x] Shared peer encoder MLP (6→32→32)
    - [x] Masked max pooling over peer embeddings
    - [x] Handle n=0 case (all peers masked)
    - [x] Test: `features_dim == 36` (32 peer latent + 4 ego)

- [x] **2.2 TDD Tests**
    - [x] Create `ml/tests/unit/test_deep_set_extractor.py`
    - [x] Test: Output shape is `(batch, 36)`
    - [x] Test: Handles zero peers (no NaN/Inf)
    - [x] Test: Handles max peers (8 valid)
    - [x] Test: Permutation invariant
    - [x] Test: Works with SB3 PPO

- [x] **2.3 Policy Integration**
    - [x] Verify `MultiInputPolicy` + `DeepSetExtractor` works
    - [x] Verify `model.predict()` returns valid actions

**Tests:** 6/6 unit tests pass

---

## Phase 3: Emulator Causality Fix ✅ COMPLETE

**Spec:** `PHASE_3_4_SPEC.md`
**Builder Report:** `PHASE_3_4_BUILDER_REPORT.md`
**Review:** `PHASE_3_4_REVIEW.md` — **APPROVED** (2026-01-15)
**Goal:** Fix Sim2Real causality violation where `ConvoyEnv` used future messages immediately.

- [x] **3.1 Emulator Updates**
    - [x] Modify `ml/espnow_emulator/espnow_emulator.py`
    - [x] Implement `sync_and_get_messages(current_time_ms)`
    - [x] Ensure it processes `pending_messages` correctly

- [x] **3.2 ConvoyEnv Causality Fix**
    - [x] Modify `ml/envs/convoy_env.py`
    - [x] Refactor `_step_espnow` to separate Transmit loop from Receive logic
    - [x] Use `sync_and_get_messages()` to get only physically arrived messages
    - [x] Verify `age_ms` calculation is correct

---

## Phase 4: Training Pipeline ✅ COMPLETE

**Spec:** `PHASE_3_4_SPEC.md`
**Builder Report:** `PHASE_3_4_BUILDER_REPORT.md`
**Review:** `PHASE_3_4_REVIEW.md` — **APPROVED** (2026-01-15)
**Goal:** End-to-end training loop validation.

- [x] **4.1 Training Script**
    - [x] Create `ml/training/train_convoy.py`
    - [x] Implement PPO setup with `MultiInputPolicy`
    - [x] Configure `DeepSetExtractor` as custom feature extractor
    - [x] Use `create_deep_set_policy_kwargs(peer_embed_dim=32)`

- [x] **4.2 Integration Test**
    - [x] Create `ml/tests/integration/test_training_pipeline.py`
    - [x] Verify SUMO availability check uses env var (Docker compatible)
    - [x] Verify model saving works

- [x] **4.3 Post-Review Fixes** (2026-01-15)
    - [x] Replaced `os.system()` SUMO check with `SUMO_AVAILABLE` env var
    - [x] Added `sync_and_get_messages` test case to `test_causality_fix.py`
    - [x] Added `ml/training/__init__.py` for conventional packaging

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
| 2026-01-15 | Phase 1 | 74/74 | ConvoyEnv Dict observation space - APPROVED |
| 2026-01-15 | Phase 2 | 6/6 | Deep Sets Policy Network - APPROVED |
| 2026-01-15 | Phase 3 & 4 | All | Causality fix + Training pipeline - APPROVED |
| 2026-01-15 | Phase 3 & 4 | 90/90 | **Final Verification:** Fixed Docker mount, `ml/__init__.py`, and SUMO accel bug. Pipeline runs end-to-end. |
