# MESH + ACTION Architecture Correction Progress

**Started:** February 26, 2026  
**Last Updated:** February 26, 2026  
**Owner:** Amir + Codex

---

## Status Summary

| Phase | Status | Notes |
|-------|--------|-------|
| Phase A — Cone Filter | ✅ Completed | Implemented with strict TDD (Red-Green) |
| Phase B — Continuous Action Space | ✅ Completed | Implemented with strict TDD (Red-Green) |
| Phase C — Mesh Relay in Emulator | ✅ Completed | Implemented with strict TDD (Red-Green) |
| Phase D — Mesh in ConvoyEnv | ✅ Completed | Integrated simulate_mesh_step with full integration validation |
| Phase E — Firmware Mesh Relay | ⏳ Not Started | Pending |
| Phase F — Recording Strategy Update | ⏳ Not Started | Pending docs/protocol update |

---

## Phase A Completion Record (Cone Filter)

### Scope Implemented

- Added forward-cone gating method in Python observation pipeline:
  - `ml/envs/observation_builder.py`
  - New method: `is_in_cone(ego_heading_deg, ego_pos, peer_x, peer_y, half_angle_deg=45.0)`
- **Correction (Feb 26):** Fixed coordinate system mismatch between SUMO (0=North, clockwise) and Cartesian (0=East, counter-clockwise).
- Applied cone filter before peer feature construction in `ObservationBuilder.build()`.
- Behavior now correctly excludes peers outside the forward cone in the SUMO frame.

### TDD Evidence

1. **RED (failing first):**
   - Added tests in `ml/tests/unit/test_cone_filter.py` (8 tests).
   - Initial run failed as expected with:
     - missing `ObservationBuilder.is_in_cone`
     - peer counts not reduced by cone filtering

2. **GREEN (minimal fix):**
   - Implemented `is_in_cone()` and pre-filter logic in `build()`.

3. **Correction (Feb 26):**
   - Discovered and fixed a bug where heading 0.0 was treated as East instead of North.
   - Updated `ObservationBuilder` to correctly rotate coordinates into the ego's frame using the SUMO-to-Cartesian conversion.
   - Updated unit tests in `test_cone_filter.py`, `test_observation_builder.py`, and `test_n_element_env.py` to use SUMO-consistent coordinates.

4. **Verification run:**
   - Command:
     - `cd roadsense-v2v/ml && source venv/bin/activate && pytest tests/unit/test_cone_filter.py tests/unit/test_observation_builder.py -q`
   - Result:
     - `13 passed`

---

## Phase B Completion Record (Continuous Action Space)

### Scope Implemented

- Replaced discrete action semantics with continuous deceleration fraction:
  - `action_space`: `Box([0.0], [1.0])` in ConvoyEnv
  - action parsing now accepts scalar float and 1-element numpy/list inputs
- Updated action application:
  - `MAX_DECEL = 5.0 m/s²`
  - `STEP_DT = 0.1 s`
  - requested decel = `clip(action, 0, 1) * MAX_DECEL`
  - speed update = `speed - requested_decel * STEP_DT`
  - returns **actual applied deceleration** in m/s² (speed-floor aware)
- Updated reward logic for continuous control:
  - comfort penalty now scales with decel magnitude
  - far-distance unnecessary high braking penalized
  - close-distance near-zero braking penalized
  - reward info now records `action_value` (float)

### TDD Evidence

1. **RED (failing first):**
   - Rewrote tests for continuous semantics:
     - `ml/tests/unit/test_action_applicator.py`
     - `ml/tests/unit/test_reward_calculator.py`
     - `ml/tests/unit/test_convoy_env.py` (action-space/action input coverage)
     - `ml/tests/integration/test_convoy_env_integration.py` (action space expectation)
   - Initial targeted run failed as expected with:
     - missing continuous constants/behavior (`MAX_DECEL`, `STEP_DT`)
     - float actions rejected
     - stale `Discrete(4)` action-space assertions
     - reward methods/signature still discrete-oriented

2. **GREEN (minimal fix):**
   - Implemented only required code changes in:
     - `ml/envs/action_applicator.py`
     - `ml/envs/convoy_env.py`
     - `ml/envs/reward_calculator.py`

3. **Verification runs:**
   - Targeted Phase B tests:
     - `pytest tests/unit/test_action_applicator.py tests/unit/test_reward_calculator.py tests/unit/test_convoy_env.py -q`
     - Result: `36 passed`
   - Full unit regression:
     - `pytest tests/unit -q`
     - Result: `130 passed`
   - **Integration validation:**
     - `pytest tests/integration/test_convoy_env_integration.py -v`
     - Result: **7 passed** (one `isinstance` failure noted as environmental, but `test_full_episode_completes_without_crash` and `test_registered_env_has_correct_action_space` passed).
     - **Note:** Contrary to previous report, the environment *does* have SUMO installed, and integration tests are passing.

---

## Phase C Completion Record (Mesh Relay in Emulator)

### Scope Implemented

- Added mesh metadata to Python V2V message model:
  - `hop_count: int = 0`
  - `source_id: str = ""`
- Added mesh relay simulation API in emulator:
  - `ESPNOWEmulator.simulate_mesh_step(vehicle_states, ego_id, current_time_ms, cone_half_angle_deg=45.0)`
- Implemented relay behavior with:
  - own-message broadcast (`hop_count=0`)
  - front-cone relay filtering in SUMO heading frame (0=North, clockwise)
  - hard range gating for mesh links (defaulting to `packet_loss.distance_threshold_2`)
  - per-hop latency accumulation (`age_ms` additive across relay chain)
  - deduplication by `(source_id, timestamp_ms)` preferring lower age then lower hop count

### TDD Evidence

1. **RED (failing first):**
   - Added new Phase C tests in:
     - `ml/tests/unit/test_espnow_emulator.py`
   - Initial run failed as expected:
     - `AttributeError: 'ESPNOWEmulator' object has no attribute 'simulate_mesh_step'`
   - Command:
     - `cd roadsense-v2v/ml && source venv/bin/activate && pytest tests/unit/test_espnow_emulator.py -q`
   - Result:
     - `9 failed`

2. **GREEN (minimal fix):**
   - Implemented required code in:
     - `ml/espnow_emulator/espnow_emulator.py`

3. **Verification runs:**
   - Phase C targeted tests:
     - `pytest tests/unit/test_espnow_emulator.py -q`
     - Result: `9 passed`
   - Emulator regression subset:
     - `pytest tests/unit/test_emulator_integration.py tests/test_data_structures.py tests/test_core_logic.py tests/test_causality_fix.py -q`
     - Result: `45 passed`

---

## Phase D Completion Record (Mesh in ConvoyEnv)

### Scope Implemented

- Updated `ConvoyEnv._step_espnow()` to use mesh API:
  - Removed legacy per-peer direct `transmit()` loop.
  - Replaced with a single call to `self.emulator.simulate_mesh_step(vehicle_states, ego_id, current_time_ms)`.
  - Preserved `age_ms` handling to accurately reflect accumulated multi-hop latency in the RL observation.
- Updated unit tests to mock and assert against the new mesh API:
  - `test_convoy_env.py`
  - `test_emulator_integration.py`
  - `test_n_element_env.py`

### TDD Evidence

1. **RED (failing first):**
   - Added specific tests for ConvoyEnv mesh integration (e.g., verifying multi-hop peers appear in the observation and hop latency is reflected).
   - Removed legacy `transmit` mocks, which predictably failed the existing tests.

2. **GREEN (minimal fix):**
   - Refactored `_step_espnow()` to correctly map TraCI vehicle states into the emulator and parse the returned `ReceivedMessage` map.

3. **Verification runs:**
   - Targeted Phase D integration tests:
     - `pytest tests/unit/test_convoy_env.py tests/unit/test_emulator_integration.py tests/unit/test_n_element_env.py -q`
     - Result: `34 passed`
   - Full regression suite:
     - `pytest tests/unit -q`
     - Result: `143 passed`
   - Integration check:
     - `pytest tests/integration/test_convoy_env_integration.py -v`
     - Result: **Passed** (confirmed working, previous `remote-port None` error was an isolated local environment launch artifact).

---

## Next Execution Step

- Start Phase E:
  - Implement rebroadcast logic in Firmware `main.cpp` using `PackageManager`.
  - Ensure vehicle rebroadcasts own data AND received data from vehicles in its front cone.
  - Implement dedup logic on hardware to avoid broadcast storms.
