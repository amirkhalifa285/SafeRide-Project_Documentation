# Phase 3 & 4 Builder Report (Causality Fix + Training Pipeline)

**Date:** 2026-01-15
**Agent Role:** BUILDER
**Spec:** `docs/10_PLANS_ACTIVE/N_Element_Implementation/PHASE_3_4_SPEC.md`

## Scope Implemented
- Added emulator receive synchronization (`sync_and_get_messages`) to enforce message causality.
- Refactored `ConvoyEnv` to decouple ESP-NOW transmit from receive and compute message staleness at observation time.
- Added Deep Sets policy kwargs helper, training script, and integration test for the pipeline.
- Updated unit test mocks to include new emulator API.

## Files Changed
- `roadsense-v2v/ml/espnow_emulator/espnow_emulator.py`
  - Added `sync_and_get_messages(current_time_ms)` to deliver queued messages and return a snapshot buffer.
- `roadsense-v2v/ml/envs/convoy_env.py`
  - `_step_espnow` now: transmit all peers -> sync receive -> compute `age_ms` and `valid` -> build obs.
  - Uses emulator staleness threshold if available, otherwise ObservationBuilder default.
- `roadsense-v2v/ml/policies/deep_set_policy.py` (new)
  - `create_deep_set_policy_kwargs(peer_embed_dim=32)` returns SB3 `features_extractor_*` config.
- `roadsense-v2v/ml/policies/__init__.py` (new)
  - Exports `create_deep_set_policy_kwargs`.
- `roadsense-v2v/ml/training/train_convoy.py` (new)
  - PPO training script using `MultiInputPolicy` + DeepSetExtractor; checkpointing + save.
- `roadsense-v2v/ml/tests/integration/test_training_pipeline.py` (new)
  - Short training loop (100 steps) with model save assertion.
- `roadsense-v2v/ml/tests/unit/test_n_element_env.py`
  - Mock emulator now supports `sync_and_get_messages` with queued ReceivedMessage.
- `roadsense-v2v/ml/tests/unit/test_convoy_env.py`
  - Mock emulator now includes `sync_and_get_messages` stub.

## Key Logic Changes
### Emulator Causality
- `sync_and_get_messages()` drains `pending_messages` by arrival time and updates `last_received`.
- Returns a shallow copy of `last_received` for env-side formatting and staleness handling.

### ConvoyEnv ESP-NOW Flow
- **Transmit phase:** loop over peer IDs and call `emulator.transmit()` (fire-and-forget).
- **Receive phase:** call `sync_and_get_messages(current_time_ms)` to get only arrived messages.
- **Age/validity:** compute `age_ms = now - received_at + latency` and mark `valid` using staleness threshold.
- Observation builder receives `age_ms` and `valid` per peer.

### Training Pipeline
- `train_convoy.py` uses `create_deep_set_policy_kwargs(peer_embed_dim=32)` with SB3 PPO.
- Saves checkpoints under `ml/logs/checkpoints/` and final model to `ml/models/saved/`.
- Integration test runs short training and asserts model zip exists.

## Potential Risk Areas / Deep Review Targets
1. **Age calculation correctness**
   - Uses `age_ms = current_time_ms - received_at_ms + received.age_ms`.
   - This matches previous get_observation logic but should be verified with expected latency behavior.

2. **Staleness threshold source**
   - Env pulls from `emulator.params['observation']['staleness_threshold_ms']` when available.
   - If emulator params are missing or mismatched, ObservationBuilder defaults apply.

3. **sync_and_get_messages vs get_observation parity**
   - get_observation still exists; ensure no tests or scripts rely on mixed usage in a way that breaks ordering assumptions.

4. **Training pipeline integration**
   - `ml/policies/deep_set_policy.py` uses `ml.models.deep_set_extractor` import path; confirm when running from repo root.
   - `train_convoy.py` expects SUMO scenario at `ml/scenarios/base/scenario.sumocfg`.

5. **Integration test runtime**
   - `test_training_pipeline.py` requires SUMO and may be slow; marked as integration+slow.

## Tests To Run (Reviewer)
From `roadsense-v2v/`:
```bash
pytest ml/tests/unit/test_n_element_env.py
pytest ml/tests/unit/test_convoy_env.py
```
Optional (requires SUMO):
```bash
pytest -m integration ml/tests/integration/test_training_pipeline.py
```

## Tests Run By Builder
- Not run (sandbox constraints).
