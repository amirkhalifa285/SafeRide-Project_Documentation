# Run 010 Post-Training Analysis Agent

**Role:** Analyze Run 010 training results, determine if the CF override fix produced V2V reaction, and diagnose the persistent negative reward.
**Date:** March 8, 2026

---

## Context

Run 010 is the 8th training attempt. It uses the same reward structure and hyperparameters as Run 009 (which had perfect stability but 0% V2V reaction) with ONE critical architectural change: **CF override** -- SUMO's car-following model is disabled during hazard events, forcing the RL agent to be the sole source of braking.

### Why Run 010 Exists

Runs 004 and 009 both converged to a passive "do nothing" local optimum (0% V2V reaction). The root cause: SUMO's CF model brakes automatically during hazard events, giving the ego the same reward as RL-commanded braking but without comfort penalty. The RL agent rationally learns to let CF handle everything. **The CF override removes this free ride.**

### Run 010 Configuration

**The ONLY change vs Run 009:**
- **CF override in `action_applicator.py`:** When `cf_override=True` and RL action is below release threshold, ego speed is HELD at current value (`setSpeed(ego, current_speed)`) instead of releasing to CF model (`setSpeed(ego, -1)`).
- **CF override activation in `convoy_env.py`:** Activates 3 steps after hazard injection (`CF_OVERRIDE_GRACE_STEPS = 3`). Persists for rest of episode. Resets between episodes.
- **Eval matrix tracking fix:** `hazard_injection_attempted` and `hazard_injection_failed_reason` now accumulated in `episode_details`.

**Unchanged from Run 009 (all carry forward):**
- Reward: linear ramp -5 at 5m to +4 at 20m, safe plateau +4 at 20-35m, far zone -2 beyond 35m
- Comfort penalty: graduated by distance multiplier `max(0.1, (d-5)/15)`
- Hyperparams: LR=1e-4, n_steps=4096, log_std_init=-0.5, ent_coef=0.0
- Architecture: Deep Sets, Box(1,) action, MAX_DECEL=8.0, 5-dim ego, 6-dim peers
- Dataset: dataset_v3/base_real, 25 train + 10 eval scenarios, n=1-5 peers
- Eval: 200 episodes via deterministic eval matrix, hazard injection ON
- total_timesteps: 10,000,000, seed=42
- 228 unit tests passing

### Previous Run History

| Run | Reward | V2V Reaction | Key Issue |
|-----|--------|-------------|-----------|
| 004 | +352 | **0%** | Passive -- CF model did all braking for free |
| 005 | +175 | 0% | Regressed -- unit tests mocked SUMO |
| 006 | -6700 | 0% | Reward terms fired during normal driving |
| 007 | -7620 | 0% | Comfort penalty drowned safety signal |
| 007.1 | -5190 | 0% | Discrete zone boundaries = poverty trap |
| 008 | -7340 | 0% | std exploded (LR too high + ent_coef drift) |
| 009 | **-3460** | **0%** | Perfect stability, but passive optimum. CF model did all braking. |

### Run 010 In-Training Observations (Live Monitoring)

**1M steps:**
- ep_rew_mean: ~-4000 (promising early signal)
- std: declining normally from 0.6

**2M steps (dip):**
- ep_rew_mean: -6700 to -7800 (regressed from 1M peak)
- std: 0.317-0.321 (healthy, faster decline than Run 009)
- clip_fraction: 0.000244-0.025 (some very low values -- yellow flag)
- explained_variance: 0.80-0.97

**9.8M steps (near completion):**
- ep_rew_mean: -4160 to -4810 (recovered, still improving in final readings)
- std: 0.122-0.123 (rock solid)
- clip_fraction: 0.014-0.043 (STRONGER than at 2M -- PPO still active)
- explained_variance: 0.93-0.98
- approx_kl: 0.0008-0.009 (healthy)

**Key observations:**
1. Reward followed V-shape: -4000 (1M) -> -7200 (2M dip) -> -4200 (9.8M recovery)
2. The 2M dip is likely the model "feeling the pain" of CF override for the first time
3. std monotonic 0.6 -> 0.12 (stability fixes validated again, 2nd consecutive run)
4. clip_fraction recovered from near-zero at 2M to healthy 0.01-0.04 at 9.8M
5. Final reward ~-4500 is ~1000 worse than Run 009's -3460. Expected because hazard episodes are genuinely harder with CF override.

---

## Your Analysis Tasks

### 1. THE Critical Question: Did V2V Reaction Happen?

This is the ONLY metric that determines success vs failure. Every run before this had 0%.

**Steps:**
1. Read `eval_results.json` or `metrics.json` -- find `v2v_reaction_rate` (or `reaction_rate`)
2. If V2V reaction > 0%: **this is a breakthrough**. Extract reaction_time_s, which scenarios, which peer counts.
3. If V2V reaction = 0%: the CF override didn't work as intended. Proceed to root cause analysis (Task 5).

### 2. Reward Decomposition (Root Cause for -4500 Mean)

The mean reward is ~-4500. To understand WHY it's negative, you MUST decompose it. Do NOT just report the mean.

**Step 2a -- Hazard vs Non-Hazard Split:**
This is the single most informative decomposition. In Run 009:
- Non-hazard episodes: avg = -293 (acceptable)
- Hazard episodes: avg = -17,506 (catastrophic)
- Reward gap: 17,214 points

Do the same split for Run 010. If non-hazard episodes are near 0 or positive, the negative mean is ENTIRELY driven by hazard episodes (expected with CF override). If non-hazard episodes are also deeply negative, something else is wrong.

**How to do this:**
- `eval_results.json` should have per-episode details with `hazard_source_id` or `hazard_message_received_by_ego`
- Alternatively, parse `monitor.csv` (per-episode rewards) cross-referenced with hazard injection info
- Group episodes: hazard-received vs no-hazard. Compute avg/median/min/max for each group.

**Step 2b -- Peer Count Breakdown:**
Report avg_reward, behavioral_success, pct_time_unsafe by n=1,2,3,4,5. In Run 009, n=3 was 100% success while n=2,4,5 with hazards were terrible.

**Step 2c -- Per-Scenario Breakdown:**
Which specific eval scenarios produce the worst rewards? Cross-reference with hazard injection success/failure per scenario.

### 3. Collision Analysis

With CF override, the model CAN now crash (CF can't save it). This is new territory.

**Check:**
- What is the collision_rate? (Run 009 was 0% because CF saved everything)
- If collisions > 0: Are they spawn-time (steps 0-32) or mid-episode? Spawn-time collisions are a known artifact; mid-episode collisions during hazard events are EXPECTED if the model hasn't learned to brake.
- Collision breakdown by scenario and peer count
- Are collisions concentrated in hazard episodes? (They should be)

### 4. Training Curve Deep Dive

**Step 4a -- Full reward trajectory:**
Read `training-run.log`. Extract ep_rew_mean at regular intervals (every 500K steps). Describe the curve shape. Compare to Run 009:

| Phase | Run 009 | Run 010 |
|-------|---------|---------|
| 1M | -6328 | ~-4000 |
| 2M | -6155 | ~-7200 |
| 5M | -6614 | ? |
| 8M | -6266 | ? |
| 10M | -6395 | ~-4500 |

**Step 4b -- Is the model still improving at 10M?**
Look at the last 1M steps (9M-10M). Is the trend:
- Downward (still improving)? -> Consider extending to 15M or 20M
- Flat (plateaued)? -> No benefit from more steps
- Oscillating with no trend? -> Scenario mix noise

**Step 4c -- Policy gradient loss:**
Extract `policy_gradient_loss` over training. In Run 009 it was -0.000808 (strongest ever). Is Run 010 comparable?

### 5. CF Override Effectiveness Analysis

This is critical regardless of V2V reaction result.

**Step 5a -- Did CF override actually activate?**
- Check `hazard_injection_attempted` in episode_details (this was the tracking bug fixed for Run 010)
- How many episodes had `cf_override_active = True`?
- Run 009 had only 4/10 eval scenarios successfully injecting hazards. Did Run 010 improve this?

**Step 5b -- Behavioral difference during hazard episodes:**
Compare hazard episode behavior between Run 009 and Run 010:
- Run 009 hazard episodes: model did nothing, CF braked, min distance 0.06m, reward -17,506
- Run 010 hazard episodes: What happened? Did the model brake? Did it crash? What's the min distance?

**Step 5c -- If V2V reaction is still 0%:**
The CF override should have made "do nothing" catastrophic during hazards (ego coasts into lead vehicle). If the model STILL doesn't brake:
1. Is it crashing instead? (collision_rate during hazard episodes)
2. Is the CF override actually disabled? (Check if `setSpeed(ego, current_speed)` is being called vs `setSpeed(ego, -1)`)
3. Is the 3-step grace period too short? (Signal might not arrive in time)
4. Is the model braking just below the detection threshold? (REACTION_DECEL_THRESHOLD = 0.5 m/s^2)

### 6. Stability Assessment

**Confirm these are still healthy (expected yes based on live monitoring):**
- std: monotonic decline, no late-training drift?
- clip_fraction: nonzero through end of training?
- explained_variance: >0.9?
- approx_kl: no spikes above 0.01?

### 7. Eval Matrix Coverage

Run 009 had coverage_ok=false with 12/15 buckets missing. The tracking fix should help.

- Is `coverage_ok` now true?
- How many of 15 (n,rank) buckets are covered?
- Which buckets are still missing and why?
- Does `hazard_injection_attempted` now show nonzero values?

---

## Verdict Framework

Based on your analysis, recommend ONE of:

### A. SUCCESS -- V2V reaction > 0% and model is viable
The CF override worked. Model actively brakes during hazards. Proceed to:
- Full characterization: reaction time, peer count scaling, scenario coverage
- H5 sim-to-real validation against real convoy recordings
- Quantization (TFLite INT8 for ESP32)
- Fix any remaining eval matrix coverage gaps

### B. PARTIAL SUCCESS -- V2V reaction > 0% but needs improvement
Model shows some V2V reaction but inconsistently. Consider:
- Extend training to 15M or 20M from the 10M checkpoint (do NOT change hyperparams)
- Identify which scenarios/peer counts have reaction and which don't
- Check if reaction_time_s is reasonable (<2s)

### C. IMPROVING BUT NOT THERE -- Reward still improving at 10M, V2V unclear
The training curve suggests more steps would help. Recommend:
- Extend to 15M or 20M: `model.load("path_to_10M_checkpoint")` then `model.learn(additional_steps)`
- Do NOT change hyperparameters mid-run

### D. CF OVERRIDE WORKED BUT MODEL CRASHES -- Collisions replaced passive behavior
The CF override successfully removed the free ride, but instead of learning to brake, the model crashes. This means:
- The reward gradient for "brake to avoid collision" isn't strong enough relative to comfort penalty
- Consider: increase collision penalty, reduce comfort penalty during hazard, or add explicit braking bonus
- May need reward tuning for Run 011

### E. CF OVERRIDE DIDN'T ACTIVATE -- Environment bug
The tracking shows CF override never activates, or hazard injection still fails in most scenarios. Debug:
- Check `hazard_injection_attempted` / `hazard_injection_failed_reason`
- Check `_cf_override_active` activation logic
- This is a code bug, not a training problem

### F. FAILED -- No V2V reaction, no improvement over Run 009
Despite CF override, model found another way to avoid braking. Possible causes:
- CF override has a bug (not actually disabling CF)
- Grace period too long (CF brakes before override kicks in)
- Model learned to stay far enough that hazard events never close the gap

---

## File Locations

Sync results from S3 first:
```bash
export AWS_DEFAULT_REGION=il-central-1
mkdir -p ~/RoadSense2/run_010_results
aws s3 sync s3://saferide-training-results/cloud_prod_010/ ~/RoadSense2/run_010_results/
```

Expected files:
```
~/RoadSense2/run_010_results/
├── training-run.log                    <- Full training log (parse for reward/std/clip curves)
├── eval_results.json                   <- Eval metrics (primary analysis target)
├── metrics.json                        <- Training config + final metrics + episode_details
├── best_model.zip                      <- Best model checkpoint
├── model_final.zip                     <- Final 10M model
├── deep_sets_*.zip                     <- Periodic checkpoints (every 10K steps)
├── tensorboard/                        <- TensorBoard logs
│   └── PPO_1/
│       └── events.out.tfevents.*
├── monitor.csv                         <- Per-episode reward log (reward, length, time)
└── config.json                         <- Run configuration
```

---

## Analysis Methodology Guide

### Best approach for root-causing negative reward:

**1. Start with eval_results.json (or metrics.json episode_details)**
This has per-episode data with hazard info. Parse it to split hazard vs non-hazard episodes immediately. This single split explains most of the reward variance.

**2. Use monitor.csv for training-time reward trajectory**
`monitor.csv` has one row per episode: `r` (reward), `l` (length), `t` (wall time). Plot/describe the reward distribution over time. Look for:
- Is the distribution tightening (std decreasing)?
- Are there two clusters (bimodal = hazard vs non-hazard)?
- Is the lower cluster (hazard episodes) improving over training?

**3. Parse training-run.log for PPO diagnostics**
Extract at regular intervals (every 250K-500K steps):
- ep_rew_mean, std, clip_fraction, explained_variance, policy_gradient_loss, approx_kl
- Look for inflection points (where reward trend changes)
- The 2M dip is already known -- confirm recovery and check for any late-training issues

**4. Cross-reference per-episode reward with hazard injection tracking**
For each eval episode, determine:
- Was hazard injected? (hazard_injection_attempted)
- Did ego receive the V2V message? (hazard_message_received_by_ego)
- Did CF override activate? (cf_override_active or equivalent)
- Did the model brake? (v2v_reaction_detected, or check rl_action > threshold)
- What was the minimum distance?
- What was the episode reward?

This gives you a complete picture of what happens during each hazard episode.

**5. If V2V reaction = 0%, check action distributions**
Look at the actual actions the model outputs during hazard episodes:
- Are actions near 0 (no braking)? -> Model hasn't learned
- Are actions > 0 but below REACTION_DECEL_THRESHOLD (0.5 m/s^2 / 8.0 = 0.0625)? -> Threshold too high
- Are actions large but only after collision? -> Too late

---

## Important Context

- **Reward numbers are NOT comparable across runs** -- Run 010 has CF override which makes hazard episodes genuinely harder. A worse average reward than Run 009 is EXPECTED and not a regression.
- **Run 009 baseline:** avg=-3460, median=-871, collision_rate=0%, V2V_reaction=0%, behavioral_success=72%
- **The whole point of Runs 005-010** is V2V reaction > 0%. Everything else (reward, stability, collisions) is secondary to this ONE metric.
- **CF override is environment-level, not reward-level.** It doesn't change what the model is rewarded for -- it changes what happens physically when the model doesn't act. This is the first run where "do nothing" has real consequences during hazards.
- **Board Y-axis is forward** -- always use `--forward-axis y` with analysis tools.
- **If extending training**, do NOT change hyperparameters mid-run -- just add more steps.
