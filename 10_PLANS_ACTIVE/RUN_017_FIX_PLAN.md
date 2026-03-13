# Run 017 Fix Plan — Merged Root Cause Review

**Date:** March 12, 2026
**Status:** ✅ IMPLEMENTED — March 12, 2026. All 5 fixes applied, 274 unit tests + Docker integration tests passing.
**Purpose:** Consolidate the latest Grok and Gemini analyses with code-backed findings, then define the minimum structural fixes to test in Run 017.

---

## Post-Run Addendum (March 13, 2026)

Run 017 did **not** cleanly test the intended hypothesis.

A new code-backed issue was found in `ml/envs/convoy_env.py` after launch:

- `reset()` runs 30+ warmup calls to `_step_espnow()` before the episode starts
- `_step_espnow()` can latch `self._braking_received_latched = True`
- that latch was **not** cleared after warmup

Impact:

- **Run 016:** the new `braking_received` observation bit could already be
  contaminated at episode start, before any injected hazard
- **Run 017:** reward was aligned to that same latch, so the reward gate could
  also be contaminated from warmup traffic rather than the actual hazard event

This means Run 017 was still training on a semantically corrupted signal,
independent of the slowDown signal-strength question.

Run 018 follow-up already prepared locally:

- **Option A implemented:** hazard braking duration shortened from `2.0-4.0s`
  to `0.5-1.5s` so the hazard accel cue is much closer to Run 011 strength
- **Option B implemented:** PPO training wrapped with
  `VecNormalize(norm_obs=False, norm_reward=True)` and stats persisted
  to `vecnormalize.pkl`

If Run 018 still fails, revisit whether warmup-time latch contamination or a
related pre-episode observability bug is still corrupting the intended signal.

Historical note:

- `cloud_prod_011` did **not** exceed `explained_variance > 0.1` until
  `1,314,816` timesteps
- `cloud_prod_013` first exceeded `0.1` at `389,120` timesteps

So the earlier `100k/300k` critic wake-up thresholds in this document were
diagnostic targets, not evidence-backed kill thresholds from the winning runs.

---

## Documents reviewed

Primary project context:
- `docs/PROJECT_STATUS_OVERVIEW.md`
- `docs/10_PLANS_ACTIVE/RUN_016_BRAKING_RECEIVED_FEATURE.md`
- `docs/10_PLANS_ACTIVE/RUN_013_ROOT_CAUSE_ANALYSIS.md`
- `docs/10_PLANS_ACTIVE/RUN_011_ANALYSIS.md`

External analyses:
- `docs/10_PLANS_ACTIVE/GROK_CRITIC_ANALYSIS.txt`
- `docs/10_PLANS_ACTIVE/GROK_REFINED_ANALYSIS.txt`
- `/home/amirkhalifa/RoadSense2/GEMINI_CRITIC_ROOTCAUSE_ANALYSIS.md`

Relevant code paths:
- `roadsense-v2v/ml/envs/convoy_env.py`
- `roadsense-v2v/ml/envs/hazard_injector.py`
- `roadsense-v2v/ml/envs/reward_calculator.py`
- `roadsense-v2v/ml/envs/observation_builder.py`
- `roadsense-v2v/ml/envs/action_applicator.py`
- `roadsense-v2v/ml/models/deep_set_extractor.py`
- `roadsense-v2v/ml/scripts/validate_against_real_data.py`

---

## Executive verdict

We do **not** have a guaranteed final fix yet.

We **do** have a strong, code-backed **diagnostic fix package** for Run 017:

1. Align the reward gate with the same latched braking signal the policy sees.
2. Add explicit progress/time context so fixed hazard timing is actually observable.
3. Stop penalizing an ego vehicle that has already safely stopped.
4. Keep additional instrumentation so we can measure whether reward/observation desync is still happening.

This package is stronger than either external analysis alone:
- Grok refined the main structural diagnosis correctly.
- Gemini correctly recovered the stopped-car penalty bug as a secondary issue.
- Current code confirms both of those points, while also narrowing what should and should not change for Run 017.

---

## Claude review outcome

Claude reviewed the merged plan and explicitly approved the Run 017 diagnostic package.

Code verification confirmed all four central claims:
- observation and reward currently use different latches
- a stopped ego can still be penalized for "ignoring" the hazard
- `braking_received` is currently pre-cone
- CF override prevents free-riding on SUMO car-following once active

Claude-approved decisions:
1. Keep the diagnostic sequence as written:
   - align reward gate
   - add progress feature
   - fix hazard timing for the diagnostic run
   - speed-gate the ignoring penalty
   - keep instrumentation
2. Keep `braking_received` **pre-cone** for Run 017 to avoid adding another confounding variable.
3. Use **ego speed only** to suppress the stopped-car penalty in Run 017.
4. Implement diagnostic hazard timing directly in `HazardInjector` defaults / code path for Run 017, not by relying on per-reset options from training scripts.

Claude watch items:
- The progress feature is acceptable for the diagnostic run, but it is **not** a deployable ESP32 feature as-is.
- `validate_against_real_data.py` will need an explicit progress proxy during replay.
- A successful Run 017 should be followed by gradual re-randomization, not immediate production use of the fixed-timing setup.

---

## External analysis synthesis

| Source | What it gets right | What is overstated or still open |
|--------|--------------------|----------------------------------|
| `GROK_CRITIC_ANALYSIS.txt` | Hidden hazard timing and observation/reward mismatch are real. | Earlier version overstated certainty and claimed fixed timing alone restored the Markov property. |
| `GROK_REFINED_ANALYSIS.txt` | Best current high-level diagnosis: hidden timing alias becomes fatal only after `slowDown()` weakens the onset signal; progress feature and gate alignment are sensible next steps. | The cone-filter recommendation is still an open design choice, not a confirmed requirement. |
| `GEMINI_CRITIC_ROOTCAUSE_ANALYSIS.md` | Correctly identifies the stopped-car penalty bug as real and worth fixing. | Fixed `hazard_step` alone still does not fully solve timing aliasing without explicit progress in the observation. |
| Codex review of docs + code | Confirms the reward/observation latch split in code, confirms the stopped-car penalty bug, and confirms Run 013 had a healthy critic despite the reward family. | Does not claim Run 017 is solved until the structural patch set passes local smoke and Docker integration. |

---

## Code-backed findings

### 1. Hidden hazard timing is real and still partly unobservable

`HazardInjector` currently uses:
- `HAZARD_PROBABILITY = 0.5`
- `hazard_step` randomized inside `[150, 350]`

That means pre-hazard observations can map to very different future returns depending on hidden episode timing.

**Files:**
- `roadsense-v2v/ml/envs/hazard_injector.py`

### 2. Observation and reward are driven by different latches

Current environment logic uses two different variables:

- Observation path:
  - `_braking_received_latched`
  - driven by `any_braking_peer_received`
  - passed to `ObservationBuilder.build(..., braking_received=...)`

- Reward path:
  - `_hazard_source_braking_latched`
  - driven only by the injected hazard source message
  - passed to `RewardCalculator.calculate(..., hazard_source_braking_received=...)`

This is a real source of noise exactly where the critic most needs a stable signal.

**Files:**
- `roadsense-v2v/ml/envs/convoy_env.py`
- `roadsense-v2v/ml/envs/reward_calculator.py`

### 3. `braking_received` is currently computed before cone filtering

In both training and real-data validation, the latch is set before `ObservationBuilder` applies the cone filter:

- Training:
  - `convoy_env.py` computes `any_braking_peer_received` from the raw valid `received_map`
  - then passes the latched boolean into `ObservationBuilder`

- Real-data validation:
  - `validate_against_real_data.py` computes `braking_received_latched` from `peer_obs`
  - then passes it into `ObservationBuilder`

So the current semantics of `ego[5]` are:

> "At least one valid received peer message reported hard braking"

not:

> "At least one cone-filtered peer in the observation is hard braking"

This is important because it means the cone-filter question is still an explicit design decision for Run 017.

**Files:**
- `roadsense-v2v/ml/envs/convoy_env.py`
- `roadsense-v2v/ml/scripts/validate_against_real_data.py`
- `roadsense-v2v/ml/envs/observation_builder.py`

### 4. A stopped ego can still be penalized for "ignoring" the hazard

`RewardCalculator` currently treats the ego as "braking" only when:

```python
abs(deceleration) > GENTLE_DECEL_THRESHOLD
```

But once the ego reaches zero speed, `actual_decel` becomes `0.0` in `ActionApplicator`, so the ego can continue taking `PENALTY_IGNORING_HAZARD` even after it has already stopped safely.

This is a real reward bug.

**Files:**
- `roadsense-v2v/ml/envs/reward_calculator.py`
- `roadsense-v2v/ml/envs/action_applicator.py`

### 5. Low action does not hand braking back to SUMO once CF override is active

After hazard injection plus the grace period, `cf_override=True` causes low actions to hold current speed instead of releasing to SUMO car-following.

That means the old "do nothing and SUMO will brake for free" explanation is no longer valid under current code.

**Files:**
- `roadsense-v2v/ml/envs/convoy_env.py`
- `roadsense-v2v/ml/envs/action_applicator.py`

### 6. The stopped-car bug is secondary, not the whole explanation

Run 013 already used the hazard-specific reward family and still had a healthy critic:
- `explained_variance = 0.90-0.97`

So the stopped-car bug cannot be the sole reason the critic died in Runs 014-016.

The most defensible causal chain is:

1. `slowDown()` weakens the onset signal
2. hidden hazard timing becomes much harder for the critic to resolve
3. observation/reward desync adds noise at the onset step
4. the stopped-car penalty bug worsens the reward landscape after braking succeeds

**Files:**
- `docs/10_PLANS_ACTIVE/RUN_013_ROOT_CAUSE_ANALYSIS.md`
- `docs/10_PLANS_ACTIVE/RUN_016_BRAKING_RECEIVED_FEATURE.md`

---

## Proposed Run 017 change set

Run 017 should be treated as a **diagnostic run**, not yet a final production training recipe.

### Fix 1 — Align reward with the observation gate

**Goal:** remove the largest confirmed mismatch.

Change the reward path to use the same latched braking signal that drives `ego[5]`.

Recommended implementation:

- Add a new reward argument, for example `braking_received: bool`
- Pass `self._braking_received_latched` into `RewardCalculator.calculate()`
- Compute `PENALTY_IGNORING_HAZARD` and `REWARD_EARLY_REACTION` from this aligned signal
- Keep `hazard_source_braking_received` and `_hazard_source_braking_latched` alive for instrumentation only

**Do not delete the hazard-source path yet.**

It should stay in `info` so Run 017 can measure divergence between:
- the aligned training signal
- the old hazard-source-only signal

### Fix 2 — Add explicit progress to the ego observation

**Goal:** make remaining episode horizon observable for the critic.

Add:

```text
ego[6] = current_step / max_steps
```

Required updates:
- `observation_builder.py`: ego feature dim `6 -> 7`
- `convoy_env.py`: pass current progress into observation builder
- `deep_set_extractor.py`: ego feature dim `6 -> 7`, features dim `38 -> 39`
- tests that assert observation and extractor shape
- `validate_against_real_data.py`: synthesize the same normalized progress feature during replay, using `step_index / total_replay_steps` for the diagnostic path

Why this is included:
- fixed `hazard_step` without explicit progress still leaves pre-hazard time aliasing
- the critic needs a way to distinguish "hazard 150 steps away" from "hazard 50 steps away"

Important note:
- this feature is for the **diagnostic run**
- it should not be treated as final ESP32 deployment input without a later replacement or removal plan

### Fix 3 — Use fixed hazard timing for the diagnostic run

**Goal:** reduce hidden episode-type variance while we verify critic recovery.

For Run 017 only, change the diagnostic defaults directly in `HazardInjector`:
- set `HAZARD_PROBABILITY = 1.0`
- use default fixed `hazard_step = 200` instead of `randint(150, 350)`
- keep braking duration randomized (`2.0-4.0 s`)

This should be treated as a diagnostic setting, not a permanent training policy.

If the critic wakes up under this controlled setup, hazard timing can be re-randomized later once the value function is healthy.

### Fix 4 — Gate the ignoring-hazard penalty on ego speed

**Goal:** stop punishing a vehicle that has already reacted successfully.

Recommended implementation:
- extend `RewardCalculator.calculate()` to accept `ego_speed`
- add `STOPPED_SPEED_THRESHOLD`, for example `0.5 m/s`
- only apply `PENALTY_IGNORING_HAZARD` when:
  - aligned braking signal is active
  - ego is not braking
  - `ego_speed > STOPPED_SPEED_THRESHOLD`

For Run 017, this is the minimum reward fix.

Claude review decision:
- use the **speed gate alone**
- do not add distance or closing-rate conditions in Run 017

**Do not unlatch the hazard entirely when ego stops.**

That would hide the underlying event and risks creating a brake-then-coast exploit.

### Fix 5 — Add instrumentation instead of deleting old logic

Add diagnostics to `info` / reward telemetry:

- `obs_reward_gate_divergence`
  - `1` when aligned reward bit differs from hazard-source bit
- `stop_while_hazard_active`
  - `1` when aligned hazard signal is active and ego speed is below stopped threshold
- `ignoring_penalty_suppressed_for_stop`
  - `1` when the speed gate prevented a penalty on that step
- `aligned_braking_latched`
  - current value of the reward-driving aligned latch
- `hazard_source_braking_latched`
  - current value of the old hazard-source latch
- first flip step for:
  - aligned braking latch
  - hazard-source latch

This turns Run 017 into a measurable diagnosis instead of another blind training attempt.

Note:
- both latches remaining high for much of the episode is acceptable for the diagnostic run
- the important thing is to log how often they diverge and how often the stopped-speed suppression fires

---

## Cone semantics decision for Run 017

Claude approved keeping the current semantics for the diagnostic run.

### Option A — Keep current pre-cone semantics for Run 017

Meaning:
- `ego[5]` stays "any valid received hard-brake message"
- reward aligns to that same bit
- real-data validation stays consistent with training

Pros:
- minimal variable count for the diagnostic run
- easiest way to isolate critic recovery
- training and validation remain aligned with current Python semantics

Cons:
- the bit is broader than the cone-filtered peer set embedded in the observation
- final deployment semantics for an ESP32 implementation remain unspecified

### Option B — Move `braking_received` to cone-filtered semantics now

Meaning:
- compute the bit only from peers that survive the observation cone filter
- update both training and `validate_against_real_data.py`

Pros:
- observation bit matches the visible peer set exactly
- probably cleaner long-term behavior for deployment

Cons:
- introduces another structural change into the same diagnostic run
- makes Run 017 harder to attribute if learning still fails

### Approved choice for Run 017

Use **Option A** for the diagnostic run:
- keep `braking_received` pre-cone
- align reward to it exactly
- instrument the mismatch

Rationale:
- current training and current real-data validation already share this semantics
- the critic problem should be isolated before changing the semantics of the bit itself
- cone-filtered semantics can be tested in a follow-up ablation once the critic is alive

---

## Changes explicitly not recommended for Run 017

- Do not delete the hazard-source latch and related telemetry yet.
- Do not unlatch the hazard event just because ego stopped.
- Do not change PPO hyperparameters before testing the structural fixes.
- Do not change the semantics of `braking_received` and the reward gate and the timing regime all at once unless the goal is a full ablation, not a diagnostic run.

---

## Required file changes

Core environment:
- `roadsense-v2v/ml/envs/convoy_env.py`
- `roadsense-v2v/ml/envs/reward_calculator.py`
- `roadsense-v2v/ml/envs/observation_builder.py`
- `roadsense-v2v/ml/envs/hazard_injector.py`
- `roadsense-v2v/ml/envs/action_applicator.py` (read path only; likely no logic change beyond reward inputs elsewhere)

Model:
- `roadsense-v2v/ml/models/deep_set_extractor.py`

Validation:
- `roadsense-v2v/ml/scripts/validate_against_real_data.py`

Tests:
- observation shape tests
- extractor shape tests
- reward unit tests for stopped-speed gate
- env unit tests for aligned reward signal
- integration tests covering forced hazards with the new ego dim and telemetry

---

## Validation plan before EC2

### 1. Unit and Docker integration tests

Must pass before any launch:
- targeted unit tests for new ego dimension
- reward tests for stopped-speed penalty suppression
- env tests for aligned reward/observation gate
- full Docker integration suite

### 2. Local short smoke training

Run a controlled local smoke train before EC2:
- fixed hazard timing
- full hazard probability
- same dataset as current formation-fixed runs

Expected signal:
- `explained_variance > 0.1` by `100k`
- `explained_variance > 0.3` by `300k`

If `explained_variance` remains at numerical noise around `0`, kill immediately and postmortem.

### 3. Reaction check

Use the sequential forced-hazard check path already preferred after the deterministic eval mismatch:
- `ml/scripts/check_v2v_reaction.py`

Diagnostic threshold:
- nonzero reaction at `100k`
- clearly positive reaction at `300k`
- target `>= 70%` at `1M`

### 4. H5 replay path

If the critic wakes up and early reaction appears in SUMO:
- rerun `validate_against_real_data.py`
- verify the new progress feature is wired correctly in replay
- confirm no shape mismatch or silent defaulting

---

## Kill criteria for Run 017

Kill early and do not wait for 10M if either of these is still true:

1. `explained_variance` stays effectively `0` through `300k`
2. forced-hazard reaction remains `0%` at the first meaningful checkpoint

If both improve:
- continue to `1M`
- evaluate reaction
- only then decide whether to extend to `10M`

---

## Post-Run 017 follow-up if diagnostic succeeds

Claude recommended reintroducing randomness gradually instead of jumping straight back to the full Run 015/016 regime:

1. Run 018:
   - `hazard_step = 200`
   - `HAZARD_PROBABILITY = 0.7`
2. Run 019:
   - `hazard_step` in `[175, 225]`
   - `HAZARD_PROBABILITY = 0.7`
3. Run 020:
   - `hazard_step` in `[150, 350]`
   - `HAZARD_PROBABILITY = 0.5`

This should only happen if Run 017 first proves that the critic can wake up under the controlled diagnostic setup.

---

## Implementation record

All 5 fixes were implemented on March 12, 2026. 274 unit tests + full Docker integration suite passing.

### Files modified (14 total)

**Core environment (6 files):**

| File | Changes |
|------|---------|
| `ml/envs/convoy_env.py` | Ego obs 6→7 dim (obs space bounds, both empty_obs guards). Reward call uses `braking_received=self._braking_received_latched` + `ego_speed=ego_speed` (Fix 1). Computes `progress = self._step_count / self.max_steps` and passes to obs builder (Fix 2). Instrumentation fields in info dict: `hazard_source_braking_latched`, `obs_reward_gate_divergence`, `ego_speed` (Fix 5). |
| `ml/envs/reward_calculator.py` | Parameter renamed `hazard_source_braking_received` → `braking_received`. Added `ego_speed: float` param + `STOPPED_SPEED_THRESHOLD = 0.5`. Speed gate suppresses `PENALTY_IGNORING_HAZARD` when ego stopped. New info field `ignoring_penalty_suppressed_for_stop` (Fix 4). |
| `ml/envs/observation_builder.py` | Added `progress: float = 0.0` parameter. Ego array extended from 6 to 7 elements with clamped progress `max(0.0, min(1.0, float(progress)))` at index 6 (Fix 2). |
| `ml/envs/hazard_injector.py` | `HAZARD_PROBABILITY = 1.0` (was 0.5). Added `DEFAULT_HAZARD_STEP = 200`. Default path uses fixed step instead of `rng.randint(150, 350)` (Fix 3). |
| `ml/models/deep_set_extractor.py` | `ego_feature_dim` default 6→7, `features_dim` now 32+7=39 (Fix 2). |
| `ml/scripts/validate_against_real_data.py` | Synthesizes `progress = step_idx / max(1, total_replay_steps)` during replay. Passes to `obs_builder.build()` (Fix 2). |

**Unit tests (4 files):**

| File | Changes |
|------|---------|
| `ml/tests/unit/test_observation_builder.py` | Ego shape `(6,)` → `(7,)`. Added `test_build_observation_progress_feature()` and `test_build_observation_progress_clamped()`. |
| `ml/tests/unit/test_deep_set_extractor.py` | Ego shape `(6,)` → `(7,)`. Feature shapes `38` → `39`. Renamed dim test to `test_extractor_features_dim_is_39`. |
| `ml/tests/unit/test_reward_calculator.py` | All `hazard_source_braking_received=` → `braking_received=`. Added 4 new tests: stopped ego not penalized, moving ego still penalized, threshold boundary, info contains new fields. |
| `ml/tests/unit/test_convoy_env.py` | Ego shape `(6,)` → `(7,)`. Updated latch test to check `braking_received_latched` + `hazard_source_braking_latched`. |
| `ml/tests/unit/test_hazard_injector.py` | Probability test: statistical 50% → deterministic 100%. Added `test_hazard_injector_default_step_is_fixed`. |

**Integration tests (3 files):**

| File | Changes |
|------|---------|
| `ml/tests/integration/test_convoy_env_integration.py` | All ego shape assertions `(6,)` → `(7,)`. |
| `ml/tests/integration/test_run006_fixes.py` | Ego shape `(6,)` → `(7,)`. `test_ignoring_hazard_penalty_persists_past_active_slowdown` rewritten: allows penalty gaps when `ignoring_penalty_suppressed_for_stop=True` (speed gate), fails only on unexplained gaps. |
| `ml/tests/integration/test_scenario_switching.py` | Ego shape `(6,)` → `(7,)`. |

### Test results

- **274 unit tests:** ALL PASSING
- **Docker integration tests:** ALL PASSING (20/20)
- Integration test for stopped-car speed gate validated: penalty gaps during stopped state are correctly explained by `ignoring_penalty_suppressed_for_stop=True`, zero unexplained gaps.

### Key implementation decisions

1. **Fix 1 (reward alignment):** `convoy_env.py` now passes `self._braking_received_latched` to both the observation builder AND the reward calculator. The old `_hazard_source_braking_latched` remains in `info` for instrumentation only.
2. **Fix 2 (progress):** `ego[6] = step_count / max_steps`, clamped [0,1]. Ego obs is now 7-dim. `validate_against_real_data.py` synthesizes `step_idx / total_replay_steps`.
3. **Fix 3 (fixed timing):** `HAZARD_PROBABILITY=1.0`, `DEFAULT_HAZARD_STEP=200`. Braking duration still randomized 2-4s.
4. **Fix 4 (speed gate):** `STOPPED_SPEED_THRESHOLD=0.5 m/s`. Penalty suppressed when ego is stopped, NOT unlatched.
5. **Fix 5 (instrumentation):** `obs_reward_gate_divergence`, `hazard_source_braking_latched`, `ego_speed`, `ignoring_penalty_suppressed_for_stop` all in step info dict.

### Ready for EC2

Code is validated and ready for Docker training run or EC2 launch per the validation plan above.

---

## Bottom line

The best current interpretation is:

- **Primary failure mode:** hidden timing alias + weak post-`slowDown()` onset signal + reward/observation gate mismatch
- **Secondary failure mode:** stopped-car penalty bug

Run 017 tests the smallest combined patch set that addresses all of those at once, while preserving enough telemetry to prove which part actually fixed the critic.

All fixes are implemented and validated. 274 unit tests + Docker integration tests passing.
