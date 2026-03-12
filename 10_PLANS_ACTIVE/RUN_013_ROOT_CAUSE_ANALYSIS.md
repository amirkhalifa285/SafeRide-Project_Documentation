# Run 013 Root Cause Analysis

**Date:** March 11, 2026
**Author:** Claude (Opus 4.6)
**Scope:** Full postmortem of Run 013 (hazard-gated reward fix), root cause identification
**Runs compared:** `cloud_prod_011` vs `cloud_prod_012` vs `cloud_prod_013`
**Status:** Root cause confirmed locally by Codex. Primary fix implemented in code; Docker integration re-validation pending.

---

> Update (March 11, 2026, Codex):
> Claude's primary diagnosis was correct. The timing bug was confirmed against
> the current `convoy_env.py` implementation and local Run 013 artifacts.
> An episode-level latch for `hazard_source_braking_received` has now been
> implemented in code, along with regression tests that would have failed on
> the pre-fix behavior. Docker integration validation still needs to be run.

## Executive Verdict

**Run 013 FAILED.** V2V reaction rate: **0/276 (0%)** — identical to Run 012.

The hazard-gated reward fix (`PENALTY_IGNORING_HAZARD`, `REWARD_EARLY_REACTION`) was
architecturally correct but had a **critical timing flaw**: the reward signal
only fires during the 2-4 second active deceleration phase of `slowDown()`,
**not during the entire hazard event**.

Once the hazard source reaches speed 0 and its acceleration returns to 0.0,
the condition `accel_x <= BRAKING_ACCEL_THRESHOLD (-2.5)` becomes False. The
ignoring penalty and early reaction bonus switch off. The model then faces
the exact same reward landscape as Run 012 for the remaining 210-230 steps
of the hazard — and Run 012's reward landscape produces 0% reaction.

**The reward signal covers only 8-16% of the post-hazard steps.** This is
too brief and too weak to train a robust braking response via PPO.

---

## Head-to-Head Results

| Metric | Run 011 | Run 012 | Run 013 | Trend |
|--------|---------|---------|---------|-------|
| avg_reward | -86.62 | -1135.22 | **-1351.68** | Worsening |
| V2V reaction | 276/276 (100%) | 0/276 (0%) | **0/276 (0%)** | Same as 012 |
| collision_rate | 0% | 0% | 0% | Stable |
| behavioral_success | 84.1% | 53.6% | **42.4%** | Worsening |
| avg_pct_time_unsafe | 16.7% | 28.2% | 27.6% | Flat |
| min_dist_post_hazard (mean) | 4.15m | 2.82m | **2.75m** | Slightly worse |
| min_dist < 1m | 23/276 | 54/276 | 49/276 | Flat |
| PPO std (final) | 0.043 | 0.086 | **0.060** | Between 011/012 |

**Run 013 is the worst of the three runs on reward and behavioral success.**
The ignoring penalty adds ~-100 to -200 per hazard episode without teaching
the model to avoid it, dragging avg_reward below Run 012.

---

## The Root Cause: Reward Signal Window Is Too Brief

### Timeline of a hazard episode (example: 3s braking, injection at step 200)

```
Step:   0       150       200    230         350       500
        |         |         |      |           |         |
        |  normal |  normal | SLOW |  TARGET   | normal  |
        |  driving|  driving| DOWN |  AT 0 m/s | driving |
        |         |         |      |           |         |
        |         |         | accel_x = -4.63  | accel_x = 0.0
        |         |         | FLAG = True      | FLAG = False
        |         |         | PENALTY fires    | NO penalty
        |         |         |      |           |
        |         |         |<-30->|<---270--->|
        |         |         | 12%  |    88%    |
        |         |         |reward|  no reward|
        |         |         |signal|   signal  |
```

### The math

| Braking duration | Signal steps | Post-hazard steps | Coverage |
|-----------------|-------------|-------------------|----------|
| 2.0s | 20 | 250 (avg) | **8.0%** |
| 3.0s | 30 | 250 (avg) | **12.0%** |
| 4.0s | 40 | 250 (avg) | **16.0%** |

**Per-episode impact of ignoring penalty (if model doesn't brake):**
- 2s braking: 20 steps x -5.0 = **-100** per episode
- 3s braking: 30 steps x -5.0 = **-150** per episode
- 4s braking: 40 steps x -5.0 = **-200** per episode

**Per-episode impact of non-hazard cruising reward:**
- 350 steps x +4.0 = **+1400** per episode

**Ratio: The ignoring penalty is only 7-14% of the cruising reward.**
PPO sees this as noise, not signal.

### Why the flag turns False after slowDown completes

```python
# convoy_env.py lines 374-386
hazard_source_braking_received = False
if (
    self.hazard_injector._hazard_injected        # True ✓
    and hazard_source_id is not None              # True ✓
):
    for received in received_map.values():
        msg = received.message
        source = msg.source_id or msg.vehicle_id
        if source == hazard_source_id:
            if float(msg.accel_x) <= self.BRAKING_ACCEL_THRESHOLD:  # -2.5
                hazard_source_braking_received = True   # ← ONLY during active decel
            break
```

After `slowDown()` completes:
1. `maintain_hazard()` calls `setSpeed(target, 0.0)` — target pinned at speed 0
2. SUMO reports `getAcceleration(target)` = 0.0 (no speed change)
3. V2V message has `accel_x = 0.0`
4. `0.0 <= -2.5` is **False**
5. `hazard_source_braking_received = False`
6. **No ignoring penalty. No early reaction bonus.**

The model then faces 210-230 steps with ONLY the old geometry-based ramp
reward — the same reward landscape that produced 0% reaction in Run 012.

### Compounding factor: HAZARD_PROBABILITY = 0.5

Only 50% of training episodes have hazards. Combined with 8-16% coverage:

**Effective reward signal: ~4-8% of total training steps.**

This is far too diluted for PPO to learn a reliable braking response.

---

## Training Trajectory Analysis

### Reward trajectory (sampled at intervals)

| Phase | Timesteps | ep_rew_mean | Interpretation |
|-------|-----------|-------------|----------------|
| Startup | 0-400K | -1950 to -938 | Random policy, learning basics |
| Improvement | 400K-2M | -938 to -124 | Learning safe following |
| **Oscillation** | **2M-8M** | **-461 to +252** | **Cycling between strategies** |
| Late | 8M-10M | -296 to +252 | No convergence, still oscillating |

**Compare Run 011:** -1890 → **+593** (smooth convergence by 5M, stable plateau 5-10M)

**Run 013 never stabilized.** The reward oscillates with 500+ range throughout
training. This is consistent with the weak, intermittent reward signal:
- Batches where the brief braking signal aligns with model exploration → positive
- Batches where the model misses the brief window → penalty adds negative reward
- Net result: policy oscillates, never converges to a clean braking strategy

### PPO diagnostics (healthy but insufficient)

| Metric | Run 013 | Interpretation |
|--------|---------|----------------|
| std | 0.60 → 0.060 | Monotonic decrease, policy sharpened |
| clip_fraction | 0.07-0.12 | Active learning throughout |
| approx_kl | 0.003-0.015 | Healthy update sizes |
| explained_variance | 0.90-0.97 | Value function accurate |

PPO training was mechanically healthy. The issue is not PPO stability — it's
that the reward signal is too weak to drive the desired behavioral change.

---

## Why behavioral_success_rate dropped from 53.6% to 42.4%

The ignoring penalty adds -100 to -200 per hazard episode. Since the model
did NOT learn to avoid it, the penalty simply shifts the reward distribution
downward without improving behavior. More episodes fall below the -1000
threshold that defines behavioral success.

This is a measurement artifact of the added penalty, not worse driving.
The actual driving behavior (min_distance, pct_time_unsafe) is nearly
identical to Run 012.

---

## What the Evidence Definitively Rules Out

1. **Signal path broken?** No — `braking_signal_reception_rate: 1.0` across all ranks.
   The V2V message reaches ego with negative accel_x every hazard episode.

2. **Fix not deployed?** Almost certainly deployed. Merge commit `1463d87` on
   `origin/master` at 20:18:40 UTC. Training started at 20:20:07 UTC (1m27s
   later). `git pull` shows "Already up to date." indicating the EC2 had the
   latest code. The worse avg_reward (-1351 vs -1135) is consistent with the
   ignoring penalty firing during the brief signal window.

3. **PPO instability?** No — std is monotonically decreasing, explained_variance
   is >0.90, clip_fraction is healthy. Training mechanics are fine.

4. **Wrong field (accel_x vs accel_y)?** No — in SUMO simulation, `accel_x` is
   correctly populated from `VehicleState.acceleration` via `to_v2v_message()`.
   The axis mismatch only matters for real hardware (board Y-axis forward),
   not for SUMO training/eval.

5. **Penalty too small?** It IS too small in aggregate effect, but the per-step
   value (-5.0) is adequate. The problem is DURATION, not magnitude.

---

## Root Cause Summary

| # | Root Cause | Impact |
|---|-----------|--------|
| **1** | **`hazard_source_braking_received` gates on `accel_x <= -2.5`, which is only True during active slowDown (2-4s). After target stops, accel_x = 0, flag = False.** | **Reward signal fires for 8-16% of post-hazard steps. Too brief for PPO.** |
| 2 | HAZARD_PROBABILITY = 0.5 halves effective signal | Only 50% of episodes contribute hazard reward signal |
| 3 | No "latch" mechanism — flag resets to False each step | Model cannot learn from sustained penalty across hazard event |

---

## Fix Implemented: Latch the Hazard Flag

The primary fix from this analysis has now been implemented.

Once `hazard_source_braking_received` becomes True (braking message from the
injected source arrives), it now **stays True for the rest of the episode**
instead of dropping back to False as soon as `slowDown()` completes.

### Implemented code change

`ml/envs/convoy_env.py` now:

- adds `self._hazard_source_braking_latched`
- resets that latch in `reset()`
- sets the latch the first time the injected hazard source is received with
  `accel_x <= BRAKING_ACCEL_THRESHOLD`
- keeps `hazard_source_braking_received=True` on later steps even when the
  source is already pinned at speed 0 and reports `accel_x = 0.0`
- clears the latch when no injected hazard is active

This is the exact fix Claude proposed as "Option A".

### Regression tests added

To stop this exact class of bug from slipping through again, the following
tests were added:

- `ml/tests/unit/test_convoy_env.py`
  - `test_hazard_source_braking_signal_latches_across_steps`
  - `test_reset_clears_hazard_source_braking_latch`
- `ml/tests/integration/test_run006_fixes.py`
  - `test_ignoring_hazard_penalty_persists_past_active_slowdown`

The new unit regression specifically covers the pre-fix failure mode:

- step 1: injected source message arrives with negative `accel_x`
- step 2: same source is still the active hazard but now reports `accel_x=0.0`
- expected: `hazard_source_braking_received` and `reward_ignoring_hazard`
  remain active on step 2

That would have failed before this patch.

Integration-test note:

- the first draft of the new Docker regression used `hazard_step=160`
  on the same base scenario
- that was incorrect because it left too little post-hazard horizon to observe
  sustained penalty coverage
- the regression was corrected without changing the scenario itself:
  it now temporarily relaxes the injector's lower window bound inside the test
  and forces an earlier `hazard_step=80`
- this keeps the production hazard window unchanged while letting the test
  verify persistence past the 4.0s maximum `slowDown()` window
- it also asserts that once the ignoring-hazard penalty turns on, it must not
  drop back to zero before episode termination/truncation

Scenario-coverage note (new finding, March 11, 2026):

- the integration fixture in `ml/tests/integration/test_run006_fixes.py`
  ~~still points to `ml/scenarios/base/scenario.sumocfg`, not `base_real`~~
  **FIXED** — all integration fixtures now point to `base_real`
- ~~therefore the Docker integration suite was **not** validating the
  `base_real` scenario geometry/formation that training and eval datasets are
  derived from~~ **FIXED** — Docker integration now validates `base_real`
- `base_real` itself is **not** a pure 3-vehicle replay of Recording #2:
  `recording_metadata.json` explicitly says V001/V002/V003 are recorded and
  V004/V005/V006 are synthetic peers placed ahead of V003
- ~~this explains why `base_real`-specific issues seen in SUMO-GUI were not
  caught by the current integration tests~~ **FIXED** — see fixes below

### base_real scenario fixes (March 11, 2026)

Three formation issues found via SUMO-GUI inspection and fixed:

1. **departPos gaps too small** — V002-V003 gap was 5.9m, below SUMO
   safe-insertion threshold (9.6m). Fixed: redistributed to 25m uniform
   spacing (V001=0m, V002=25m, ..., V006=125m).
2. **sigma=0.5** — SUMO driver imperfection caused random braking/accel.
   Fixed: `sigma="0.0"` for deterministic car-following.
3. **speedDev=0.1 (SUMO default)** — Each vehicle got a random desired speed
   factor from N(1.0, 0.1). V001 cruised at 11.54 m/s while V006 at
   14.72 m/s on a 13.89 m/s road, causing the convoy to spread apart.
   Fixed: `speedFactor="1.0" speedDev="0"` in vType.

### Integration test fixture update (March 11, 2026)

All integration test fixtures switched from `ml/scenarios/base/` to
`ml/scenarios/base_real/`:

- `ml/tests/integration/conftest.py`: `scenario_path` fixture
- `ml/tests/integration/conftest.py`: `dataset_dir` fixture
- `ml/tests/integration/test_run006_fixes.py`: `base_scenario_path` fixture

### Verification

- Unit tests: `37 passed` (includes latch regression tests)
- **Docker integration tests: ALL PASSING** (validated March 11, 2026)
  - All tests now run against `base_real` with correct formation dynamics

**Effect:** Once the braking signal arrives (20-40 steps after injection), the
ignoring penalty can now fire every step until episode end. If injected at
step 250 with 3s braking, signal arrives ~step 253, latch triggers. Then:
- Steps 253-500 = **247 steps** of active reward signal
- vs current: only 30 steps (12% → **99% coverage**)

### Remaining secondary recommendation: Raise HAZARD_PROBABILITY to 1.0

Every training episode should have a hazard. There is no benefit to training
50% of episodes without hazards — the model already knows how to cruise.
This doubles the effective reward signal immediately.

This has **not** been changed yet. It remains a separate training-economics
decision for the next run.

### Still not recommended as a standalone fix: Increase penalty magnitude

Raising `PENALTY_IGNORING_HAZARD` from -5 to -20 would increase the brief
signal's magnitude but not its duration. The fundamental timing problem
remains. Only combine with the now-implemented latch fix.

### Why this won't re-introduce Run 006/007 poison

The latch is still gated to:
1. `hazard_injector._hazard_injected` (training-only construct)
2. The specific hazard target's V2V message having arrived
3. The message showing hard braking (accel_x < -2.5) at detection time

During normal driving, condition 1 is always False. The latch only
activates during injected hazard events. This preserves the same safety
guarantees as the current (non-latched) implementation.

---

## Suggested Run 014 Configuration

1. **Use the current code with latched `hazard_source_braking_received`**.
2. **Decide whether to set `HAZARD_PROBABILITY = 1.0`** for training.
3. **Do not add more reward changes yet** — isolate the latch fix first.
4. **Validate with Docker integration tests before EC2**.
5. **Pass criteria:**
   - SUMO eval: >90% V2V reaction (target: 100%)
   - H5 re-validation: model action > 0.1 within 3s of peer braking onset
   - FP rate < 5% in Extra Recording

---

## Appendix: Full Eval Data

### Per-peer-count summary

| Peers | Episodes | Reaction | avg_reward | behavioral_success | avg_min_dist |
|-------|----------|----------|------------|-------------------|--------------|
| n=1 | 56 | 0% | -1756 | 30.4% | 3.04m |
| n=2 | 56 | 0% | -1416 | 44.6% | 2.82m |
| n=3 | 56 | 0% | -1241 | 46.4% | 2.64m |
| n=4 | 56 | 0% | -1163 | 44.6% | 2.52m |
| n=5 | 52 | 0% | -1170 | 46.2% | 2.69m |

### Eval matrix coverage

- **coverage_ok: True** — 15/15 buckets filled
- **injection_attempted: 276, succeeded: 276, failed: 0**
- All ranks at all peer counts: 0% reaction, 100% reception

### Training trajectory key points

```
Timesteps    ep_rew_mean    std       Phase
~100K        -1950          0.600     Random policy
~1M          -938           0.535     Learning basics
~2M          -124           0.403     Improving
~4M          -13.8          0.273     Near plateau...
~5M          -439           0.230     DROPS — oscillation begins
~6M          -170           0.150     Unstable
~8M          -242           0.077     Still oscillating
~10M         +18.2          0.060     Final value — not stable
```

Run 011 for comparison: converged to +593 by 5M, held plateau through 10M.
