# Run 006 Fix Plan — 7 Root Causes, Docker-Validated

**Created:** March 5, 2026
**Status:** ALL FIXES IMPLEMENTED AND TESTED
**Result:** 211 unit tests + 19 integration tests passing

---

## Context

Run 005 regressed from Run 004 across all metrics:
- Reward: +175 vs +352
- Collisions: 26.4% vs 19%
- V2V reaction: still 0%

All 5 Run 005 fixes failed despite 201 unit tests passing. The unit tests mock SUMO/emulator behavior and never test real physics. Every fix in this plan was validated with Docker/SUMO integration tests before going to EC2.

**Key lesson:** Unit tests with mocked physics prove nothing about real SUMO behavior. Docker integration tests are MANDATORY.

---

## Phase 1: Collision Detection — Use SUMO Ground Truth [DONE]

**Root cause:** `_calculate_min_distance_and_closing_rate()` uses GPS-noisy emulated peer positions (noise std 3-4m) for the collision/termination check. With real following gaps of 7-10m, there's a 60-100% chance of false collision from noise alone.

**File:** `ml/envs/convoy_env.py`

**Fix:**
- [x] Split collision detection (termination logic) from observation
- [x] `step()` uses NEW `_calculate_ground_truth_distance()` that reads SUMO positions directly via `sumo.get_vehicle_state()` for all active vehicles
- [x] Ground-truth distance used ONLY for `distance < COLLISION_DIST` termination check
- [x] Existing mesh-based `_calculate_min_distance_and_closing_rate()` continues to feed reward calculator
- [x] In `reset()`: changed `self.emulator.clear()` → `self.emulator.reset()` so GPS noise params re-randomize per episode

**Integration test:** `test_no_false_collision_from_gps_noise` — 50 episodes, collision rate < 10% in first 30 steps

---

## Phase 2: Route-End Guard — Move to Top of step() [DONE]

**Root cause:** The guard at line 263 ran AFTER `action_applicator.apply()` and `hazard_injector.maybe_inject()`, both of which call TraCI on V001. If V001 left at end of previous step, these crash before the guard runs.

**File:** `ml/envs/convoy_env.py`

**Fix:**
- [x] Added pre-action guard at very top of `step()`, before ANY TraCI calls on V001
- [x] If V001 not active, returns empty obs + truncated=True with `truncated_reason: ego_route_ended`
- [x] Kept existing post-`sumo.step()` guard too (catches V001 leaving during current step)

**Integration test:** `test_route_end_no_crash` — 500 steps, no TraCIException

---

## Phase 3: Warmup — Feed Emulator + Verify Safe Distance [DONE]

**Root cause:** Warmup ran 30 SUMO steps but never fed the emulator. After warmup, emulator had zero message history. Also, V001-V002 gap could close to <5m with staggered departs.

**File:** `ml/envs/convoy_env.py`

**Fix:**
- [x] Feed emulator during warmup loop with `_step_espnow()` each iteration
- [x] After warmup, verify GT distance to nearest peer is above safety margin
- [x] If too close, extend warmup by additional steps (up to 30 more)

**Integration test:** `test_warmup_produces_safe_distance` — 20 episodes, no immediate collision after warmup

---

## Phase 4: Observation — Add Braking Peer Signal [DONE]

**Root cause:** `any_braking_peer` was computed in convoy_env but NOT in the observation. The model literally cannot "see" when a peer is braking as a clean signal.

**Files:** `ml/envs/observation_builder.py`, `ml/envs/convoy_env.py`, `ml/models/deep_set_extractor.py`

**Fix:**
- [x] Added `min_peer_accel` to ego observation vector (ego dims 4→5)
- [x] Computed as minimum acceleration across cone-filtered peers, normalized by MAX_ACCEL
- [x] Updated `gymnasium.spaces.Dict` ego space from `Box(4,)` to `Box(5,)`
- [x] Updated Deep Sets extractor: `ego_feature_dim` 4→5, `features_dim` 36→37

**Integration test:** `test_braking_peer_in_observation` — forces hazard, checks ego obs[4] < -0.03

---

## Phase 5: Reward Restructure — Penalize Ignoring Hazard [DONE]

**Root cause:** The +2.0 bonus required the model to already be braking (chicken-and-egg). Over an episode, comfort penalty for unnecessary braking vastly outweighed the hazard bonus. The model correctly learned to NEVER brake.

**File:** `ml/envs/reward_calculator.py`

**Fix:**
- [x] New `PENALTY_IGNORING_HAZARD = -5.0` — fires every step when:
  - `any_braking_peer` is True AND
  - `abs(deceleration) < 0.5` (ego NOT braking) AND
  - `distance < 30m` (close enough to matter)
- [x] Comfort penalty zeroed when `any_braking_peer=True` AND ego is braking
- [x] Added `reward_ignoring_hazard` component to info dict

**Economics flip:**
- Before: not braking = +1.0/step, braking during hazard = ~+2.6/step (mild incentive)
- After: not braking during hazard = -4.0/step, braking during hazard = +3.0/step (-7.0 differential)

**Integration test:** `test_hazard_penalty_fires` — forces hazard with no braking, checks `reward_ignoring_hazard` < 0

---

## Phase 6: Eval Matrix — Fix Silent Injection Failures [DONE]

**Root cause:** Hazard injection fails silently when target rank doesn't exist. Coverage gaps invisible.

**Files:** `ml/envs/hazard_injector.py`, `ml/eval_matrix.py`

**Fix:**
- [x] `maybe_inject()` tracks `_hazard_injection_attempted`, `_hazard_injection_failed`, `_hazard_injection_failed_reason`
- [x] Three new properties exposed: `hazard_injection_attempted`, `hazard_injection_failed`, `hazard_injection_failed_reason`
- [x] `summarize_deterministic_eval_coverage()` counts attempted vs successful vs failed injections per bucket

**Integration test:** `test_eval_matrix_coverage_mini` — 10 episodes, checks injection tracking fields appear

---

## Phase 7: Docker/SUMO Integration Tests [DONE]

**File:** `ml/tests/integration/test_run006_fixes.py` (NEW)

All 8 tests passing via `./run_docker.sh integration`:

| # | Test | Validates |
|---|------|-----------|
| 1 | `test_no_false_collision_from_gps_noise` | GT collision eliminates GPS-noise false collisions (50 episodes) |
| 2 | `test_route_end_no_crash` | Pre-action guard prevents TraCIException (500 steps) |
| 3 | `test_warmup_produces_safe_distance` | Warmup extension ensures safe starting distance (20 episodes) |
| 4 | `test_braking_peer_in_observation` | min_peer_accel flows from emulator to observation (30 episodes) |
| 5 | `test_hazard_penalty_fires` | PENALTY_IGNORING_HAZARD fires correctly (30 episodes) |
| 6 | `test_eval_matrix_coverage_mini` | Injection tracking fields present (10 episodes) |
| 7 | `test_emulator_resets_per_episode` | Clean emulator state between episodes |
| 8 | `test_smoke_training_5000_steps` | PPO + DeepSetExtractor, 5000 steps, no crashes |

**Other integration test files also updated for ego 4→5 dim change:**
- `test_convoy_env_integration.py` (6 tests)
- `test_scenario_switching.py` (1 test)

Total: **19/19 integration tests passing**

---

## Files Modified

| File | Changes |
|------|---------|
| `ml/envs/convoy_env.py` | GT collision, pre-action guard, warmup feeds emulator, emulator.reset(), ego space 4→5 |
| `ml/envs/observation_builder.py` | min_peer_accel in ego vector (4→5 dims) |
| `ml/envs/reward_calculator.py` | PENALTY_IGNORING_HAZARD, comfort zeroed during hazard braking |
| `ml/envs/hazard_injector.py` | Injection failure tracking (attempted/failed/reason) |
| `ml/eval_matrix.py` | Report attempted vs succeeded vs failed injections per bucket |
| `ml/models/deep_set_extractor.py` | ego_feature_dim 4→5, features_dim 36→37 |
| `ml/tests/unit/test_convoy_env.py` | Updated mocks + ego dims + GT collision tests |
| `ml/tests/unit/test_observation_builder.py` | Ego 4→5, new min_peer_accel tests |
| `ml/tests/unit/test_reward_calculator.py` | 8 new ignoring-hazard penalty + comfort zeroing tests |
| `ml/tests/unit/test_deep_set_extractor.py` | Ego 4→5, features 36→37, all shapes updated |
| `ml/tests/unit/test_n_element_env.py` | Mock fixes for get_active_vehicle_ids + emulator.reset |
| `ml/tests/integration/test_run006_fixes.py` | NEW — 8 Docker/SUMO integration tests |
| `ml/tests/integration/test_convoy_env_integration.py` | Ego dim 4→5 |
| `ml/tests/integration/test_scenario_switching.py` | Ego dim 4→5 |

---

## Expected Run 006 Improvements

| Metric | Run 005 | Run 006 Target |
|--------|---------|----------------|
| Collision rate | 26.4% | <5% (false collisions eliminated) |
| V2V reaction | 0% | >60% (penalty + observation fix) |
| Behavioral success | 73.6% | >85% |
| Eval matrix coverage | 25% | >90% |
| Training crashes | 2/3 attempts | 0 (guard at top of step) |

---

## Execution Order (Completed)

| Step | Phase | Depends On | Docker Test | Status |
|------|-------|------------|-------------|--------|
| 1 | Phase 2: Route-end guard | None | test_route_end_no_crash | ✅ Done |
| 2 | Phase 1: Ground truth collision | None | test_no_false_collision | ✅ Done |
| 3 | Phase 3: Warmup fix | Phase 1 | test_warmup_safe_distance | ✅ Done |
| 4 | Phase 4: Observation update | None | test_braking_peer_in_obs | ✅ Done |
| 5 | Phase 5: Reward restructure | Phase 4 | test_hazard_penalty_fires | ✅ Done |
| 6 | Phase 6: Eval matrix logging | Phase 1 | test_eval_matrix_coverage | ✅ Done |
| 7 | Phase 7: Integration smoke | All above | 5000-step smoke train | ✅ Done |
| 8 | Push to git, launch Run 006 on EC2 | All tests green | — | 🔜 Next |
