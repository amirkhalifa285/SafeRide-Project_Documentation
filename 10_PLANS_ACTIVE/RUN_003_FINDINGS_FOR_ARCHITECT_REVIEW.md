# Run 003 Findings for Architect Review

**Date:** March 2, 2026
**Prepared by:** Codex analysis pass
**Scope:** Post-run diagnosis of `cloud_prod_003` training artifacts and metrics
**Status:** ALL P0 AND P1 FIXES IMPLEMENTED AND TESTED (March 2, 2026)
**Session Addendum:** See `RUN_003_SESSION_FIXES_FOR_ARCHITECT_REVIEW.md` for post-review follow-up fixes applied later in the same day.

---

## 1) Executive Summary

Run 003 completed technically (10M steps, full artifacts, 200-episode eval with hazard injection ON and n=1..5 coverage), but the learned policy quality is poor.

The key issue is not missing files or failed training execution. The issue is behavioral: the policy is evaluated in a reward landscape where most episodes are dominated by persistent unsafe-distance penalties, and the current speed-control semantics likely prevent meaningful recovery.

Result: `collision_rate=0` but very negative returns (`avg_reward=-6511.45`) and `truncation_rate=1.0`.

**Root cause:** Three compounding bugs create an inescapable penalty trap where the policy can only decelerate (never recover), and gets punished ~-8/step for being close to the leader.

---

## 2) What Was Verified

### 2.1 Artifacts (Run 003)

Location: `roadsense-v2v/ml/models/runs/cloud_prod_003/results`

- Present:
  - `metrics.json`
  - `model_final.zip`
  - `checkpoints/` (1000 checkpoint files from 10k to 10,000,000 steps)
  - `tensorboard/PPO_1/events.out.tfevents...`
- Missing:
  - `training-run.log` in this folder

Note on missing log: this is consistent with manual `tmux` launch flow (not necessarily a failed run). The model/eval artifacts themselves are present.

### 2.2 Run completion and eval coverage

From `ml/models/runs/cloud_prod_003/results/metrics.json`:

- `training_timesteps = 10000000`
- `evaluation.eval_episodes = 200`
- `evaluation.hazard_injection = true`
- `peer_count_summary` contains keys `1,2,3,4,5` with `40` episodes each

So the mandatory eval envelope (hazards ON, variable peer counts n=1..5) is satisfied.

---

## 3) Observed Performance Pattern

From `metrics.json`:

- `avg_reward = -6511.4475`
- `collision_rate = 0.0`
- `truncation_rate = 1.0`
- `success_rate = 1.0` (non-collision definition; see Section 5.3)

### 3.1 Scenario-level behavior

For 9 out of 10 eval scenarios, all 20 episodes produce either exactly one reward value or near-constant extreme negative values:

- `eval_n1_000`: all 20 = `-7107.5`
- `eval_n1_001`: all 20 = `-7141.5`
- `eval_n2_000`: all 20 = `-7196.5`
- `eval_n2_001`: all 20 = `-7099.0`
- `eval_n3_001`: all 20 = `-7116.0`
- `eval_n4_000`: all 20 = `-7143.0`
- `eval_n4_001`: all 20 = `-7107.5`
- `eval_n5_000`: all 20 = `-7111.0`
- `eval_n5_001`: all 20 = `-7135.0`

Only `eval_n3_000` shows variance (16 positive episodes at `188.5`, 4 strongly negative).

### 3.2 Distribution summary

- 200 total episodes
- 184 episodes with reward `<= -1000`
- 16 episodes with reward `> 0` (all in `eval_n3_000`)
- Median reward around `-7111.0`

This is a strong indicator that behavior is mostly locked into a deterministic penalty regime, not adaptive hazard-response control.

### 3.3 Why eval_n3_000 was the outlier

`eval_n3_000` has V004 at `departPos=38.8m`, creating a larger initial gap than other scenarios. This confirms the penalty trap is distance-threshold driven — when the gap starts large enough, the policy earns positive rewards.

---

## 4) Why This Happened (Root-Cause Analysis)

### RC-1: Action=0 permanently pins speed (THE KILLER BUG) — FIXED

In `ml/envs/action_applicator.py`:

- `ActionApplicator.apply()` called `sumo.set_vehicle_speed(V001, current_speed)` when action=0
- In TraCI, `setSpeed()` overrides SUMO's car-following model **permanently**
- The ego can only brake, never accelerate. Speed ratchets down monotonically over 1000 steps
- Math: ~900 steps at -8/step = -7200, matching observed -7100 range

**Fix applied:** Action <= 0.02 now calls `sumo.release_vehicle_speed(V001)` which invokes `traci.vehicle.setSpeed(V001, -1)`, handing control back to SUMO's Krauss CF model. This means action=0 = "I don't need to intervene, let normal driving happen."

### RC-2: Missed-warning penalty fires every step regardless of closing rate — FIXED

In `ml/envs/reward_calculator.py`:

- `PENALTY_MISSED_WARNING = -3` triggered EVERY step when `distance < 15m AND decel <= 0.5`
- Combined with `REWARD_UNSAFE = -5`, this creates -8/step that the policy can't escape
- The penalty fired even when the gap was **opening** (leader pulling away)

**Fix applied:** Added `closing_rate` parameter. Missed-warning now only fires when `closing_rate > 0.5` (gap is actually shrinking at >0.5 m/s AND ego isn't braking). "You're getting CLOSER and not braking" vs "you're close but gap is stable/opening."

### RC-3: Success metric masks quality issues — FIXED

In `ml/training/train_convoy.py`:

- `success_rate = (episode_count - collisions) / episode_count`
- So all non-collision truncations are treated as success
- This explains why `success_rate=1.0` coexists with `avg_reward=-6511`

**Fix applied:** Added `behavioral_success_rate` (requires both no collision AND reward > -1000) and `pct_time_unsafe` (% of episode steps with distance < 10m). Original `success_rate` kept for backward compat.

### RC-4: Distance thresholds too aggressive for convoy following — FIXED

- `UNSAFE_DIST = 15m` was too far for real convoy driving (real-world safe following is ~10m)
- Combined with short initial spacings (V002 at ~10.9m in eval_n1_000), episodes started in penalty zone
- SUMO's Krauss CF model desired gap: `speed * tau + minGap`. At low speed (2 m/s), desired gap is ~5m — far below 15m

**Fix applied:** Lowered thresholds:
- `UNSAFE_DIST`: 15m → 10m
- `SAFE_DIST_MIN`: 20m → 15m
- `SAFE_DIST_MAX`: 40m → 35m

### RC-5: Hazard injector hardcoded target — FIXED

- `HAZARD_TARGET_VEHICLE = "V002"` always targeted V002, regardless of which peer was nearest
- In n=3,4,5 scenarios, nearest peer might be V003 or others

**Fix applied:** Complete rewrite of `HazardInjector` to dynamically find nearest peer via `_find_nearest_peer(sumo)`. Gracefully handles no-peers and ego-not-active cases.

### RC-6: Test infrastructure bug masking 22 convoy_env test failures — FIXED

- `pytest.ini` has `pythonpath = . ..` which creates two module identities: `envs.convoy_env` and `ml.envs.convoy_env`
- `unittest.mock.patch("envs.convoy_env.SUMOConnection")` patches one module but `ConvoyEnv` is imported from the other
- Result: mock never takes effect, 22 tests in `test_convoy_env.py` always failed silently (required real SUMO)

**Fix applied:** Changed all patches in `test_convoy_env.py` to `"ml.envs.convoy_env.SUMOConnection"` etc. All 22 tests now pass without SUMO.

---

## 5) Important Clarifications

### 5.1 This is not a lost-training artifact issue

Training output is present and complete enough for diagnosis. Missing `training-run.log` does not explain performance.

### 5.2 This is not necessarily "Run 003 regressed from a good smoke baseline"

Smoke run on dataset_v3 already showed similar negative-reward pattern.
This indicates the issue likely predates long training and is rooted in env/reward/control setup.

### 5.3 Collision-only success metric is insufficient

For this project stage, "no collision" is necessary but not sufficient. We need behavior quality metrics that reflect appropriate deceleration and gap management.

---

## 6) Fixes Implemented (March 2, 2026 Session)

### Files Modified

| File | Change | Bug Fixed |
|------|--------|-----------|
| `ml/envs/sumo_connection.py` | Added `release_vehicle_speed()` method (calls `setSpeed(-1)`) | RC-1 |
| `ml/envs/action_applicator.py` | Added `RELEASE_THRESHOLD=0.02`, action<=threshold releases to CF model | RC-1 |
| `ml/envs/reward_calculator.py` | Added `closing_rate` param, gated missed-warning on `closing_rate > 0.5`, lowered distance thresholds | RC-2, RC-4 |
| `ml/envs/convoy_env.py` | Replaced `_calculate_min_distance()` with `_calculate_min_distance_and_closing_rate()`, passes closing_rate to reward calculator | RC-2 |
| `ml/envs/hazard_injector.py` | Rewrote to dynamically find nearest peer instead of hardcoded V002 | RC-5 |
| `ml/training/train_convoy.py` | Added `behavioral_success_rate`, `pct_time_unsafe`, `BEHAVIORAL_SUCCESS_REWARD_THRESHOLD=-1000` | RC-3 |
| `ml/tests/unit/test_action_applicator.py` | Rewrote: 11 tests covering release semantics, sensitivity | RC-1 |
| `ml/tests/unit/test_reward_calculator.py` | Rewrote: 15 tests covering closing-rate gating, new thresholds | RC-2, RC-4 |
| `ml/tests/unit/test_hazard_injector.py` | Rewrote: 10 tests covering dynamic targeting, edge cases | RC-5 |
| `ml/tests/unit/test_observation_builder.py` | Added 3 edge case tests (zero-peer, behind-ego cone, stale messages) | Safety net |
| `ml/tests/unit/test_convoy_env.py` | Fixed mock patch paths (`ml.envs.convoy_env.*` instead of `envs.convoy_env.*`) | RC-6 |

### Test Results After Fixes

**169/169 tests pass** (up from 133 passing + 22 broken + 14 missing)

All tests run without SUMO (pure unit tests with mocks):
```bash
cd roadsense-v2v/ml && source venv/bin/activate && python -m pytest tests/unit/ -v
```

---

## 7) Acceptance Gates Before Run 004

Before starting full training, confirm ALL are true:

1. ✅ Action sensitivity test shows clear behavioral divergence between release (action=0) and max brake (action=1)
2. ✅ Missed-warning penalty only fires when gap is actually closing (closing_rate > 0.5)
3. ✅ `behavioral_success_rate` metric distinguishes quality (not just collision avoidance)
4. ✅ Distance thresholds match real-world convoy behavior (10m unsafe, not 15m)
5. ✅ Hazard injector targets nearest peer dynamically (not hardcoded V002)
6. ✅ All 169 unit tests pass
7. ○ Short smoke train (200k-500k steps) confirms reward trend improves over Run 003
8. ○ `pct_time_unsafe` decreases versus Run 003 baseline

Items 7-8 require a training run.

---

## 8) Answered Design Questions

> 1. Should continuous action remain decel-only, or should we move to signed longitudinal control ([-1,1])?

**Answer:** Decel-only with CF release. Action=0 releases to SUMO's car-following model for natural acceleration/following. The ego doesn't need explicit acceleration control — the CF model handles that. The RL model only decides how much to brake.

> 2. Should reward be reframed around time-gap/TTC rather than raw distance bands?

**Deferred:** Keep distance bands for now (with corrected thresholds). The closing_rate gating addresses the worst penalty trap. TTC-based reward is a potential future improvement.

> 3. Should "success" require both no collision and bounded unsafe-time ratio?

**Answer:** Yes. `behavioral_success_rate` requires reward > -1000 AND no collision. `pct_time_unsafe` tracks time spent in dangerous proximity. Both added.

> 4. Should hazard injection probability remain 0.3 during eval, or be stratified and reported by hazard/no-hazard cohorts?

**Deferred:** Keep 0.3 for now. Stratified reporting is a future improvement.

---

## 9) Reproduction Commands (Used for This Analysis)

From repo root:

```bash
# Core metrics
jq '.evaluation | {eval_episodes,hazard_injection,avg_reward,collision_rate,truncation_rate,success_rate}' \
  ml/models/runs/cloud_prod_003/results/metrics.json

# Per peer-count summary
jq '.evaluation.peer_count_summary' \
  ml/models/runs/cloud_prod_003/results/metrics.json

# Scenario-level summary
jq '.evaluation.scenario_summary' \
  ml/models/runs/cloud_prod_003/results/metrics.json

# Reward distribution sanity
jq '.evaluation.episode_details | map(.reward) | group_by(.) | map({reward:.[0],count:length}) | sort_by(-.count)[:12]' \
  ml/models/runs/cloud_prod_003/results/metrics.json
```

---

## 10) Run Unit Tests (Verify Fixes)

```bash
cd roadsense-v2v/ml && source venv/bin/activate && python -m pytest tests/unit/ -v
# Expected: 169 passed, 0 failed
```

---

## 11) Bottom Line

Run 003 infrastructure worked; model quality did not meet behavioral intent.
The highest-probability blockers were reward/control semantics, not missing artifacts.

**All identified root causes (RC-1 through RC-6) have been fixed and tested.**
Next step: Run 004 with these fixes on EC2.
