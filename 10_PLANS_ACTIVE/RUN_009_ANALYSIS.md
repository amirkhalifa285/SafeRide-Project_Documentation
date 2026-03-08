# Run 009 Post-Training Analysis

**Date:** March 7, 2026
**Analyst:** Claude (automated analysis from `RUN_009_ANALYSIS_PROMPT.md`)
**Run ID:** cloud_prod_009
**Status:** COMPLETED (10M steps)

---

## Executive Summary

Run 009 is a **PARTIAL SUCCESS** — the best training run to date by every stability metric, but it **still fails the primary objective**: 0% V2V hazard reaction rate, identical to Run 004. The model learned excellent passive driving (0% collisions, 72% behavioral success) but does not actively brake when receiving V2V hazard warnings.

| Metric | Run 004 | Run 008 | Run 009 | Target |
|--------|---------|---------|---------|--------|
| avg_reward | +352 | -7340 | -3460 | positive |
| collision_rate | 19% | N/A (killed) | **0%** | 0% |
| behavioral_success | 81% | N/A | **72%** | >90% |
| V2V_reaction_rate | **0%** | N/A | **0%** | **>50%** |
| std (final) | N/A | 4.36 (exploded) | **0.118** | <0.3 |
| explained_variance | N/A | 0 (dead) | **0.97** | >0.9 |

**Verdict: B — PARTIAL SUCCESS with critical V2V failure. See Section 7 for recommended next steps.**

---

## 1. Training Curve Analysis

### 1.1 Reward Trajectory

The reward oscillated in the **-3,450 to -9,400** range throughout training with no clear convergence trend. The mean stayed roughly constant across all 10M steps:

| Phase | Mean Reward | Min | Max | Std(policy) |
|-------|------------|-----|-----|-------------|
| 0-1M  | -6,328 | -11,300 | -4,250 | 0.521 |
| 1-2M  | -6,155 | -8,180 | -3,890 | 0.374 |
| 2-3M  | -6,361 | -9,110 | -3,970 | 0.319 |
| 3-4M  | -6,433 | -8,110 | -4,430 | 0.271 |
| 4-5M  | -6,614 | -8,680 | -3,650 | 0.235 |
| 5-6M  | -6,235 | -8,240 | -4,290 | 0.202 |
| 6-7M  | -6,305 | -8,770 | -4,190 | 0.183 |
| 7-8M  | -6,266 | -9,400 | -3,620 | 0.163 |
| 8-9M  | -5,964 | -9,170 | -4,270 | 0.141 |
| 9-10M | -6,395 | -9,350 | -3,450 | 0.127 |

**Key observations:**
- The reward **plateaued** around -6,300 by ~1M steps and never improved meaningfully.
- Oscillation range stayed constant (best ~-3,500, worst ~-9,300) throughout training.
- The oscillation is NOT random — it's caused by the **scenario mix** (some scenarios trigger hazards, others don't).
- Best single reading: -3,450 at ~9.85M steps. Worst: -11,300 at step 0 (initial random policy).

### 1.2 Convergence Assessment

**Not converged, but plateaued.** The model found a stable local optimum (passive driving) early (~1M steps) and made no further progress for the remaining 9M steps. Extending training beyond 10M would NOT help — the policy has converged to a deterministic strategy that cannot discover hazard-reactive behavior through further optimization.

---

## 2. PPO Stability Assessment

### 2.1 Standard Deviation (Action Noise)

```
Step      std
0         0.615  (initial, from log_std_init=-0.5)
1M        0.429
2M        0.339
3M        0.279
5M        0.202
7M        0.169
9M        0.130
10M       0.118  (final)
```

**EXCELLENT.** Monotonic decline from 0.615 to 0.118 with ZERO drift. Compare to Run 008 where std exploded 0.75 -> 4.36. The stability fixes (log_std_init=-0.5, ent_coef=0.0, LR=1e-4) completely solved the std explosion problem.

### 2.2 Other PPO Health Metrics

| Metric | Early (0-1M) | Mid (5M) | Late (10M) | Assessment |
|--------|-------------|----------|------------|------------|
| clip_fraction | 0.001-0.038 | 0.006-0.035 | 0.016-0.032 | Healthy, PPO actively clipping |
| explained_variance | -0.0005 -> 0.54 | 0.957 | 0.969 | Near-perfect value function |
| approx_kl | variable | 0.002-0.007 | 0.003-0.005 | Healthy, no divergence |
| policy_gradient_loss | variable | active | active (-0.001 to +0.0005) | Active, not dead |
| entropy_loss | ~0.71 | ~0.71 | ~0.71 | Stable (ent_coef=0.0 so this is cosmetic) |

**All stability fixes from Run 008 post-mortem WORKED:**
1. LR 3e-4 -> 1e-4: No oscillation
2. n_steps 2048 -> 4096: Smoother gradients
3. log_std_init 0.0 -> -0.5: Controlled std from start
4. ent_coef 0.01 -> 0.0: No entropy-driven std drift

---

## 3. Evaluation Results

### 3.1 Overall Metrics

| Metric | Value |
|--------|-------|
| Total eval episodes | 250 |
| avg_reward | **-3,459.8** |
| std_reward | 7,767.6 |
| median_reward | **-870.5** |
| max_reward | +1,993.6 |
| min_reward | -31,017.0 |
| collision_rate | **0.0%** |
| truncation_rate | 100% |
| success_rate | 100% |
| behavioral_success_rate | **72%** |
| avg_pct_time_unsafe | 16.6% |

**NOTE:** Rewards are NOT directly comparable to Run 004 (+352) because Run 009 uses harsher penalties (REWARD_FAR=-2, always-on hazard injection).

### 3.2 Reward Distribution (Bimodal)

The reward distribution is **sharply bimodal**, not Gaussian:

| Bucket | Episodes | % |
|--------|----------|---|
| Positive (>0) | 54 | 22% |
| -1,000 to 0 | 126 | 50% |
| -2,000 to -1,000 | 24 | 10% |
| -5,000 to -2,000 | 8 | 3% |
| -10,000 to -5,000 | 4 | 2% |
| Below -10,000 | 34 | **14%** |

**The bimodality is caused by hazard injection:**
- **No hazard received (204 episodes):** avg_reward = **-293**, median = -845
- **Hazard received (46 episodes):** avg_reward = **-17,506**, median = -18,101
- **Reward gap: 17,214 points** between hazard and non-hazard episodes

### 3.3 Performance by Peer Count

| n peers | Episodes | Avg Reward | Behavioral Success | Pct Time Unsafe |
|---------|----------|-----------|-------------------|-----------------|
| 1 | 50 | -963 | 56% | 0.5% |
| 2 | 50 | -5,314 | 74% | 23.6% |
| 3 | 50 | **-854** | **100%** | 0.6% |
| 4 | 50 | -4,713 | 70% | 23.1% |
| 5 | 50 | -5,455 | 60% | 35.1% |

**Pattern:** n=3 is perfect (100% success, 0.6% unsafe). n=1 and n=3 scenarios never received hazard injections. The scenarios that DO receive hazards (n=2,4,5) have massive unsafe percentages.

### 3.4 Scenario-Level Bimodality

| Scenario | Episodes | Good (<-1000) | Bad (<-5000) | Avg Reward | Hazard Works? |
|----------|----------|---------------|-------------|-----------|---------------|
| eval_n1_000 | 25 | 25 | 0 | -862 | No hazard injected |
| eval_n1_001 | 25 | 3 | 0 | -1,065 | No hazard injected |
| eval_n2_000 | 25 | 12 | **12** | -9,795 | YES (hazard fires) |
| eval_n2_001 | 25 | 25 | 0 | -834 | No hazard injected |
| eval_n3_000 | 25 | 25 | 0 | -883 | No hazard injected |
| eval_n3_001 | 25 | 25 | 0 | -824 | No hazard injected |
| eval_n4_000 | 25 | 12 | **11** | -8,491 | YES (hazard fires) |
| eval_n4_001 | 25 | 23 | 0 | -935 | No hazard injected |
| eval_n5_000 | 25 | 15 | **7** | -5,210 | YES (hazard fires) |
| eval_n5_001 | 25 | 15 | **8** | -5,700 | YES (hazard fires) |

**Critical finding:** Only 4 of 10 eval scenarios successfully inject hazards. The other 6 scenarios (n=1 both, n=2_001, n=3 both, n=4_001) have hazard_source_id=null. This means the hazard injector fails to find a valid target in those scenarios.

---

## 4. V2V Reaction Analysis (THE Critical Metric)

### 4.1 Overall V2V Reaction

| Metric | Value |
|--------|-------|
| Hazard episodes (ego received braking msg) | 46 / 250 (18.4%) |
| V2V reaction detected | **0 / 250 (0.0%)** |
| Avg reaction time | **N/A (no reactions)** |

**This is the same result as Run 004.** Despite 7 training runs of fixes, the model has never learned to actively brake in response to V2V hazard warnings.

### 4.2 Source-Specific Reaction

| n Peers | Source | Episodes | Reception | Reaction | Braking Signal Rx | Avg Min Distance |
|---------|--------|----------|-----------|----------|-------------------|-----------------|
| 1 | unknown | 50 | 0% | 0% | 0% | N/A |
| 2 | rank_1 | 13 | 100% | **0%** | 100% | 1.40m |
| 2 | unknown | 37 | 0% | 0% | 0% | N/A |
| 3 | unknown | 50 | 0% | 0% | 0% | N/A |
| 4 | rank_1 | 7 | 100% | **0%** | 100% | 0.87m |
| 4 | rank_2 | 6 | 100% | **0%** | 100% | 2.36m |
| 5 | rank_1 | 10 | 100% | **0%** | 100% | 1.67m |
| 5 | rank_2 | 10 | 100% | **0%** | 100% | 2.15m |

**When the model receives the hazard braking signal (braking_signal_reception_rate=100%), it still does NOT react.** The signal is there in the observation space (min_peer_accel), but the model's learned policy ignores it.

### 4.3 Near-Miss Analysis (Hazard Episodes Sorted by Closest Approach)

| Min Distance | Reward | Scenario | Rank |
|-------------|--------|----------|------|
| **0.06m** | -31,017 | eval_n4_000 | 1 |
| **0.06m** | -27,696 | eval_n5_001 | 2 |
| **0.08m** | -28,360 | eval_n5_000 | 2 |
| 0.16m | -28,019 | eval_n5_001 | 1 |
| 0.17m | -23,135 | eval_n4_000 | 2 |
| 0.19m | -26,389 | eval_n5_001 | 1 |
| 0.19m | -26,100 | eval_n2_000 | 1 |
| 0.20m | -29,063 | eval_n4_000 | 1 |

The model gets within **6 centimeters of collision** and still does not brake. Only SUMO's ground-truth car-following model prevents the actual crash. This is an extremely dangerous policy for real-world deployment.

---

## 5. Eval Matrix Issues

### 5.1 Coverage Failure

```
coverage_ok: false
injection_attempted: 0  (BUG - should be >0)
injection_succeeded: 0  (BUG - should be >0)
missing_buckets: 12/15
```

The eval matrix tracking reports 0 injection attempts, but 46 episodes clearly received hazard messages. This is a **tracking bug** in the eval matrix code — the hazard injector fires correctly for some scenarios, but the matrix telemetry doesn't record it.

### 5.2 Bucket Coverage

Only 3 of 15 (n,rank) buckets have any observed episodes:
- n2_rank1: 13 (expected 10)
- n4_rank1: 7, n4_rank2: 6 (expected 10 each)
- n5_rank1: 10, n5_rank2: 10 (expected 10 each)

Missing entirely: n1_rank1, n2_rank2, all n3 buckets, n4_rank3, n4_rank4, n5_rank3, n5_rank4, n5_rank5.

**Root cause:** Hazard injection only works in certain scenario geometries. The injector cannot find valid peers to target in the n=1, n=3, and some n=2/n=4 scenarios.

---

## 6. Root Cause Analysis: Why 0% V2V Reaction (Again)

### 6.1 The "Passive Optimum" Problem

The model has found a **stable local optimum**: do nothing and let SUMO's car-following (CF) model handle everything. This strategy achieves:
- 0% collision rate (CF model prevents crashes)
- Low comfort penalty (no RL-commanded braking)
- Moderate following distance (~15-35m in calm traffic)

When a hazard fires:
- The lead vehicle suddenly brakes (-9 m/s^2)
- The ego vehicle's CF model reacts automatically (not RL-commanded)
- Distance closes rapidly (0.06m-5m)
- The ego receives massive penalties (unsafe zone, PENALTY_IGNORING_HAZARD)
- But the model cannot learn from this because:

### 6.2 Why The Model Cannot Escape The Local Optimum

**Problem 1: Hazard rarity during training.**
Only ~18% of eval episodes even received a hazard. During training (25 scenarios, not all with hazard injection), hazard episodes are rare. With std=0.118 (very deterministic), the model has almost zero probability of randomly trying a large braking action during a hazard to discover it gets rewarded.

**Problem 2: Temporal credit assignment.**
The hazard fires at a random step (32-73). The model must learn to associate `min_peer_accel << 0` in the observation with "brake now". But the reward is continuous (per-step distance-based ramp), not episodic. The model sees the same ramp reward whether or not it caused the braking — SUMO's CF model often provides the braking for free.

**Problem 3: The reward signal for "correct V2V braking" is identical to "CF model braked for you".**
When the lead vehicle brakes and the CF model slows the ego automatically, the ego gets the EXACT SAME reward as if the RL model had commanded braking. The RL model literally cannot distinguish its own actions from the CF model's automatic behavior in reward terms.

**Problem 4: Comfort penalty makes active braking net-negative in most states.**
At std=0.118, the model outputs nearly the same action every step (~0.0, "do nothing"). To brake, it would need to output ~0.5-1.0, incurring comfort penalty. Even though REWARD_SAFE=4.0, the comfort penalty for hard braking often exceeds the reward difference between safe and unsafe zones, especially in the ramp region.

### 6.3 The Fundamental Design Problem

**The RL agent competes with SUMO's built-in car-following model**, and the CF model is free (no comfort penalty). The RL model has learned that any action it takes is either redundant with CF (when CF already handles it) or costly (comfort penalty for unnecessary braking). The rational optimum is: do nothing.

This is NOT a hyperparameter problem. It's an **architecture/environment design problem**. The CF model must be weakened or disabled during hazard scenarios for the RL agent to have a reason to act.

---

## 7. Verdict & Recommended Next Steps

### Verdict: B — PARTIAL SUCCESS

**What worked:**
- PPO stability fully solved (std 0.615 -> 0.118, no drift)
- Value function excellent (explained_variance 0.97)
- Zero collisions (improved from 19% in Run 004)
- Linear ramp reward provides gradient (proven in both Run 008 and 009)
- All 4 stability hyperparameter fixes validated

**What failed:**
- 0% V2V reaction rate (unchanged from Run 004)
- Model converged to passive "do nothing" local optimum
- Reward cannot distinguish RL-commanded braking from CF-model braking
- Hazard injection coverage incomplete (only 4/10 eval scenarios)

### Recommended Next Steps (for Run 010)

**The core fix needed is to make the CF model unable to save the ego during hazards**, so the RL agent MUST act. Two approaches:

#### Option A: Disable CF Model During Hazard Window (RECOMMENDED)
When a hazard is injected:
1. Call `traci.vehicle.setSpeed(ego, current_speed)` to **freeze the CF model's braking**
2. The ego will coast at current speed unless the RL model actively brakes
3. This forces the RL model to learn that `min_peer_accel << 0` means "brake or crash"
4. Outside hazard windows, CF model operates normally

**Implementation:** In `convoy_env.py`, when hazard is active and `hazard_message_received_by_ego=True`, call `setSpeed` to hold current speed before applying RL action. The RL action then becomes the ONLY source of deceleration.

**Risk:** May need to tune the window to avoid training instability. Start with disabling CF for 50 steps after hazard injection.

#### Option B: Direct Hazard-Reactive Reward Bonus
Add a large bonus specifically for RL-commanded braking during hazard:
- If `min_peer_accel < -2.0` AND `rl_action > 0.3` → bonus of +10.0/step
- This makes active braking during hazard clearly superior to passive behavior
- Simpler to implement but more "reward-hacky"

#### Fix Eval Matrix Tracking
- Debug why `injection_attempted=0` when 46 episodes got hazards
- Debug why n=1 and n=3 scenarios never get hazard_source_id set
- Ensure all 15 (n,rank) buckets get covered

#### Do NOT Change
- Keep stability hyperparameters (LR=1e-4, n_steps=4096, log_std_init=-0.5, ent_coef=0.0)
- Keep linear ramp reward structure
- Keep REWARD_SAFE=4.0, REWARD_FAR=-2.0
- Do NOT extend training past 10M — the policy has plateaued

---

## 8. Run 010 Fixes — IMPLEMENTED (March 7, 2026)

All three recommended fixes have been coded, unit-tested (228 tests passing), and are ready for Docker integration testing.

### Fix 1: CF Override in ActionApplicator (`ml/envs/action_applicator.py`)

**What:** Added `cf_override` parameter to `ActionApplicator.apply()`.

When `cf_override=True` and the action is below `RELEASE_THRESHOLD` (i.e., "no brake"), instead of calling `release_vehicle_speed()` (which hands control back to SUMO's CF model), it calls `set_vehicle_speed(ego, current_speed)` — holding the ego at its current speed. This means the CF model **cannot** decelerate the ego; only the RL agent's braking commands can slow it down.

When `cf_override=False` (the default), behavior is unchanged from Run 009.

**Tests added:** 4 new tests in `test_action_applicator.py`:
- `test_cf_override_zero_action_holds_speed`
- `test_cf_override_below_threshold_holds_speed`
- `test_cf_override_braking_still_works`
- `test_cf_override_false_is_default_behavior`

### Fix 2: CF Override Activation in ConvoyEnv (`ml/envs/convoy_env.py`)

**What:** Added hazard-tracking and CF override activation logic to the environment.

- New class constant: `CF_OVERRIDE_GRACE_STEPS = 3` — allows 3 simulation steps after hazard injection for the V2V mesh message to propagate to the ego before disabling CF.
- New instance variables: `_hazard_injection_step` (records when hazard first fires) and `_cf_override_active` (bool).
- In `step()`: once `_step_count >= _hazard_injection_step + CF_OVERRIDE_GRACE_STEPS`, sets `_cf_override_active = True` for the rest of the episode.
- Passes `cf_override=self._cf_override_active` to `action_applicator.apply()`.
- Both variables reset in `reset()`.

**Design rationale:** The 3-step grace period ensures the ego has received the V2V hazard braking signal via mesh before the CF model is disabled. Without this, the ego would be coasting blind before the signal arrives — an unfair test. After the grace period, the RL model is the sole source of deceleration. If it doesn't brake, the ego crashes.

**Tests added:** 3 new tests in `test_convoy_env.py` (`TestCFOverride` class):
- `test_cf_override_inactive_before_hazard`
- `test_cf_override_activates_after_hazard_grace_period`
- `test_cf_override_resets_between_episodes`

### Fix 3: Eval Matrix Tracking Bug (`ml/training/train_convoy.py` + `ml/scripts/evaluate_model.py`)

**What:** `hazard_injection_attempted` and `hazard_injection_failed_reason` were present in per-step `info` dicts from `convoy_env.step()` but were never accumulated into the `episode_details` dicts written to metrics.json. This caused `summarize_deterministic_eval_coverage()` to always report 0 injection attempts.

**Fix:** Both `train_convoy.py` and `evaluate_model.py` now:
1. Initialize `hazard_injection_attempted = False` and `hazard_injection_failed_reason = None` per episode.
2. Accumulate from step info: if `info.get("hazard_injection_attempted")` is True, set the episode-level flag.
3. Write both fields to `episode_details`.

### What Was NOT Changed (Carry Forward from Run 009)

- PPO hyperparameters: LR=1e-4, n_steps=4096, log_std_init=-0.5, ent_coef=0.0
- Linear ramp reward: -5 at 5m → +4 at 20m, plateau +4 at 20-35m, -2 beyond 35m
- REWARD_SAFE=4.0, REWARD_FAR=-2.0
- Comfort graduated by distance multiplier
- Deep Sets architecture, observation space (5-dim ego + 6-dim peers)

---

## 9. Files & Artifacts

| File | Location |
|------|----------|
| Training log | `run_009_results/training-run.log` (14.5 MB, 2442 rollout entries) |
| Metrics JSON | `run_009_results/metrics.json` (250 eval episode details) |
| Final model | `run_009_results/model_final.zip` |
| Best model | `run_009_results/checkpoints/deep_sets_10000000_steps.zip` |
| TensorBoard | `run_009_results/tensorboard/PPO_1/` |
| Checkpoints | `run_009_results/checkpoints/` (1000 checkpoints, every 10K steps) |
| S3 backup | `s3://saferide-training-results/cloud_prod_009/` |

---

## Appendix A: Key Data Points for Future Agents (Updated March 7)

```
QUICK REFERENCE — Run 009 Results
==================================
avg_reward:             -3,459.8
median_reward:          -870.5
collision_rate:         0.0%
behavioral_success:    72%
V2V_reaction_rate:     0.0%  ← STILL THE PROBLEM
std_final:             0.118
explained_variance:    0.969
hazard_episodes:       46/250 (18.4%)
hazard_avg_reward:     -17,506
non_hazard_avg_reward: -293
near_miss_min:         0.06m (!)
eval_matrix_coverage:  false (12/15 buckets missing)
```

```
WHAT WORKS IN RUN 009 (carry forward):
  - LR=1e-4, n_steps=4096, log_std_init=-0.5, ent_coef=0.0
  - Linear ramp reward (-5 at 5m to +4 at 20m)
  - REWARD_SAFE=4.0, REWARD_FAR=-2.0
  - Comfort graduated by distance multiplier

IMPLEMENTED FOR RUN 010 (March 7, 2026):
  - CF override: CF model disabled after hazard injection + 3-step grace period
  - Eval matrix: hazard_injection_attempted now tracked in episode_details
  - 228 unit tests passing, Docker integration tests pending
```
