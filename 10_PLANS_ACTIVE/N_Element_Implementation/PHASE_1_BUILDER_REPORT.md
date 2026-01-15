# Phase 1 Builder Report (ConvoyEnv + n-Element Observations)

**Date:** 2026-01-13
**Agent Role:** BUILDER
**Spec:** `docs/10_PLANS_ACTIVE/N_Element_Implementation/PHASE_1_SPEC.md`

## Scope Implemented
- Converted ConvoyEnv observation space from fixed `Box(11,)` to `Dict` with keys `ego`, `peers`, and `peer_mask`.
- Reworked ObservationBuilder to accept variable peer observations and emit padded peer features with a mask.
- Replaced hardcoded peer IDs with dynamic peer discovery via TraCI.
- Added new unit tests for n-element behavior and updated existing unit/integration tests.

## Files Changed
- `roadsense-v2v/ml/envs/convoy_env.py`
  - Observation space now `gym.spaces.Dict` with shapes:
    - `ego`: `(4,)`
    - `peers`: `(8, 6)`
    - `peer_mask`: `(8,)`
  - Added dynamic peer discovery using `traci.vehicle.getIDList()` and removed fixed lead IDs.
  - Added `_step_espnow()` to transmit all peer states and build Dict observations.
  - `_all_vehicles_active()` now only checks ego vehicle status.
- `roadsense-v2v/ml/envs/observation_builder.py`
  - `build()` signature updated to `(ego_state, peer_observations, ego_pos)`.
  - Computes ego-normalized features and peer features with relative position, relative speed, relative heading, accel, and age.
  - Produces `peers` padded to 8 slots and a `peer_mask` for valid peers.
- `roadsense-v2v/ml/tests/unit/test_n_element_env.py` (new)
  - Validates Dict observation space.
  - Ensures peer mask is all zeros with no peers.
  - Ensures peer mask has exactly three valid entries with 3 peers.
- `roadsense-v2v/ml/tests/unit/test_observation_builder.py`
  - Updated to validate Dict output shapes and dtypes.
  - Added mask count and invalid peer handling checks.
- `roadsense-v2v/ml/tests/unit/test_convoy_env.py`
  - Updated reset expectations for Dict observations.
  - Patched TraCI ID list for deterministic peer discovery in tests.
  - Adjusted truncation test to reflect ego-only active check.
- `roadsense-v2v/ml/tests/integration/test_convoy_env_integration.py`
  - Updated observation shape assertions for Dict output.
- `roadsense-v2v/ml/demo_convoy_gui.py`
  - Updated ego speed readout to use `obs["ego"][0]`.

## Behavior Notes
- Peer observations are built from emulator transmission results (received messages). Only received messages contribute to peers.
- Peers are truncated to `MAX_PEERS = 8` and padded with zeros. `peer_mask` marks valid peers.

## Tests To Run (Reviewer)
From `roadsense-v2v/`:
```bash
pytest ml/tests/unit/test_n_element_env.py
pytest ml/tests/unit/test_observation_builder.py
pytest ml/tests/unit/test_convoy_env.py
```
Optional (requires SUMO):
```bash
pytest -m integration ml/tests/integration/test_convoy_env_integration.py
```

## Tests Run By Builder
- Not run (sandbox constraints).
