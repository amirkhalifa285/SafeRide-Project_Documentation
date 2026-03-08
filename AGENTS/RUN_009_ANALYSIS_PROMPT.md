# Run 009 Post-Training Analysis Agent

**Role:** Analyze Run 009 training results and determine next steps.
**Date:** March 7, 2026

---

## Context

Run 009 is the 7th training attempt (Runs 003-008 all failed). It uses the proven linear ramp reward structure from Run 008 combined with 4 stability fixes and stronger reward incentives.

### Run 009 Configuration

**Reward structure (linear ramp, no discrete zones):**
- Linear ramp: -5 at 5m → +4 at 20m (RAMP_LOW=-5, RAMP_HIGH=4, span=9)
- Safe plateau: +4/step at 20-35m
- Far zone: -2/step beyond 35m
- Comfort penalty: graduated by distance multiplier `max(0.1, (d-5)/15)`
- Hazard injection: ON by default in both training and eval

**Hyperparameters:**
- `learning_rate`: 1e-4 (was 3e-4 in Run 008 — caused oscillation)
- `n_steps`: 4096 (was 2048 in Run 008 — too noisy)
- `log_std_init`: -0.5 (was 0.0/unconstrained in Run 008 — std exploded to 4.36)
- `ent_coef`: 0.0 (was 0.01 in Run 008 — entropy pressure dominated weak policy gradient, drove std upward)
- `total_timesteps`: 10,000,000
- `seed`: 42

**Reward changes vs Run 008:**
- REWARD_SAFE: 3.0 → 4.0 (active braking must clearly beat passive "do nothing")
- REWARD_FAR: -1.0 → -2.0 (stronger anti-laziness penalty)
- RAMP_HIGH: 3.0 → 4.0 (steeper gradient)

**Architecture:**
- Deep Sets with permutation-invariant pooling (n-element problem)
- Continuous action space: Box(1,) — deceleration fraction [0, 1]
- MAX_DECEL = 8.0 m/s² (measured from real braking at -8.63)
- Ego observation: 5-dim (speed, accel, heading, peer_count, min_peer_accel)
- Peer observation: up to 8 peers, 6-dim each + peer_mask
- features_dim = 37 (embed_dim 32 + ego 5)

**Dataset:**
- dataset_v3/base_real (real convoy recording grounded)
- 25 train + 10 eval scenarios
- Peer counts n=1 through n=5

**Eval:**
- 200 episodes via deterministic eval matrix
- Peer counts 1,2,3,4,5
- Hazard injection ON

### Previous Run History (for comparison)

| Run | Reward | Key Issue |
|-----|--------|-----------|
| 004 | +352 | First "success" — but 0% V2V reaction (passive, let SUMO CF model drive) |
| 005 | +175 | Regressed — unit tests mocked SUMO |
| 006 | -6700 | Reward terms fired during normal driving |
| 007 | -7620 | Comfort penalty drowned safety signal (explained_variance=0) |
| 007.1 | -5190 | Discrete zone boundaries created poverty trap |
| 008 | -7340 | Ramp worked early (hit -4120 at 1M) but std exploded due to LR too high + ent_coef=0.01 entropy drift |

### Run 009 In-Training Observations (Live Monitoring)

**500K steps:**
- ep_rew_mean: -5040 to -5400
- std: 0.522-0.528 (dropping from initial 0.6)
- clip_fraction: 0.001 (low but nonzero)
- explained_variance: 0.56-0.72

**1M steps:**
- ep_rew_mean: -6440 (recovering from dip, was climbing: -9400 → -7100 → -6440)
- std: 0.429 (steady decline)
- clip_fraction: 0.004-0.049 (10-50x higher than 500K — PPO actively learning)
- explained_variance: 0.67-0.97

**7.15M steps:**
- ep_rew_mean: oscillating between -4710 and -7670 (bouncy but trending down in magnitude)
- std: 0.167-0.171 (rock solid, still declining)
- clip_fraction: 0.001-0.068 (still active)
- explained_variance: 0.95-0.98 (near-perfect value function)
- policy_gradient_loss: -0.000808 (strongest seen across all runs)
- approx_kl: 0.001-0.007 (healthy)

**Key observations:**
1. std went 0.6 → 0.52 → 0.43 → 0.17 monotonically — NO DRIFT (ent_coef=0.0 fix worked)
2. explained_variance reached 0.97 — model understands reward structure
3. Reward oscillates due to variable scenarios (n=1-5, hazard/no-hazard mix)
4. Policy gradient loss is active and strong throughout
5. This is by far the best run: Run 008 was dead at 7M (std=4.36), Run 007 was dead at 500K

---

## Your Analysis Tasks

### 1. Training Curve Analysis
- Read the full training log: `results/training-run.log`
- Plot or describe the reward trajectory over all 10M steps
- Identify convergence pattern: still improving? plateaued? diverged?
- Compare final metrics to Run 004 (+352) and Run 008 (failed at 7.6M)

### 2. Eval Results Analysis
- Read eval results: `results/eval_results.json` (if exists)
- Key metrics to extract:
  - **avg_reward** (final)
  - **collision_rate** (and breakdown: spawn-time vs mid-episode)
  - **V2V_reaction_rate** (THE critical metric — has been 0% in every run)
  - **behavioral_success_rate**
  - **peer_count_summary** (performance by n=1,2,3,4,5)
  - **reaction_time_s** (if V2V reaction exists)

### 3. Stability Assessment
- Did std stay stable or start drifting late?
- Did clip_fraction remain nonzero through end of training?
- Any signs of policy collapse or entropy issues?

### 4. Behavioral Assessment
- Is this model actively braking during hazards? (V2V reaction > 0%)
- Or is it another passive "do nothing" model like Run 004?
- Does it maintain safe following distance?

### 5. Verdict & Next Steps
Based on your analysis, recommend ONE of:

**A. SUCCESS** — Model works. Proceed to:
- H5 sim-to-real validation against real convoy recordings
- Quantization (TFLite INT8 for ESP32)

**B. PARTIAL SUCCESS** — Model improved but needs more training:
- Extend to 15M or 20M steps from the 10M checkpoint
- Specify: `model.load("path_to_10M_checkpoint")` then `model.learn(additional_steps)`

**C. NEEDS ADJUSTMENT** — Identify specific changes needed:
- Reward tuning? Which constants?
- Hyperparameter changes?
- Architecture changes?

**D. FAILED** — Explain why and what's fundamentally wrong

---

## File Locations

All results should be in the synced S3 folder:

```
~/RoadSense2/run_009_results/          ← S3 sync destination
├── training-run.log                    ← Full training log
├── eval_results.json                   ← Eval metrics (if generated)
├── metrics.json                        ← Training config + final metrics
├── best_model.zip                      ← Best model checkpoint
├── deep_sets_*.zip                     ← Periodic checkpoints (every 10K steps)
├── tensorboard/                        ← TensorBoard logs
│   └── PPO_1/
│       └── events.out.tfevents.*
├── monitor.csv                         ← Per-episode reward log
└── config.json                         ← Run configuration
```

---

## Important Context

- **Run 004 is the only "successful" model** (+352 reward, 81% behavioral success, but 0% V2V reaction)
- **The whole point of Runs 005-009** is to get a model that ACTIVELY brakes during hazards (V2V reaction > 0%)
- **Reward numbers are NOT comparable** across runs — Run 009 has harsher penalties (REWARD_FAR=-2, hazard injection always ON)
- **Board Y-axis is forward** — always use `--forward-axis y` with analysis tools
- **If extending training**, do NOT change hyperparameters mid-run — just add more steps
