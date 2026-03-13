# Run 014 Post-Mortem & Run 015 Handoff

**Date:** March 12, 2026
**Run 014 Status:** KILLED at ~987k steps
**Run 015 Status:** Ready to launch

---

## Run 014 Outcome

### Diagnostics at ~987k steps

| Metric | Value | Signal |
|--------|-------|--------|
| ep_rew_mean | -3.4k → -2.75k (improving) | Good |
| std | stable ~0.47 | Neutral |
| clip_fraction | 0.003–0.037 | Good |
| approx_kl | 0.0003–0.009 | Good |
| **explained_variance** | **0 (entire run)** | **Bad** |

### 1M Checkpoint V2V Reaction

```
Episodes:                  40
Hazard Injected:           40/40 (100.0%)
Hazard Source Received:    40/40 (100.0%)
Braking Signal Received:   40/40 (100.0%)
RL Reactions:              0/40 (0.0%)
  n=1: 0/8, n=2: 0/8, n=3: 0/8, n=4: 0/8, n=5: 0/8
```

### Kill decision

Two compounding red flags:

1. **explained_variance = 0** for the entire run (~987k steps). The critic learned nothing — it predicted a constant value for all states. PPO fell back to raw Monte Carlo returns (essentially REINFORCE).
2. **V2V reaction = 0%** at 1M with no improvement from the 100k smoke test.

The reward was improving (-3.4k → -2.75k), but this was likely from better non-hazard behavior, not hazard response learning.

### Root cause: critic starvation

`HAZARD_PROBABILITY = 1.0` meant every episode had a massive latched hazard penalty. The critic saw uniformly bad returns with no contrastive "calm" episodes to anchor value estimates. Without variance in episode types, the value function could not learn to distinguish states.

In Run 011 (the winner), `HAZARD_PROBABILITY = 0.5` gave the critic ~50% calm episodes where returns were predictable, allowing explained_variance to reach 0.956.

### Important context: the slowDown question

Three consecutive runs since switching from `setSpeed(0)` to gradual `slowDown(0, 2-4s)` have failed with 0% V2V reaction:

| Run | Hazard injection | HAZARD_PROB | Reward bug | Critic | Reaction |
|-----|-----------------|-------------|------------|--------|----------|
| 011 | `setSpeed(0)` | 0.5 | None | 0.956 | **100%** |
| 012 | `slowDown` | 0.5 | Penalty dead code | ? | 0% |
| 013 | `slowDown` | 0.5 | Flag only 8-16% | ? | 0% |
| 014 | `slowDown` | **1.0** | None (latch works) | **0** | 0% |

However, **no run has yet tested gradual slowDown with both correct rewards AND a working critic**. Each failure had a different bug. Run 015 is the first clean test.

If Run 015 shows a working critic (explained_variance > 0.5) but still 0% V2V reaction, then gradual slowDown itself is the problem — the observation signal (`min_peer_accel ≈ -0.46 normalized` vs `-13.9` with instant stop) may be too weak for the model to learn from.

---

## Run 015 Setup

### Single change from Run 014

`HAZARD_PROBABILITY`: `1.0` → `0.5` in `ml/envs/hazard_injector.py`

### Everything else retained from Run 014

- Latch `_hazard_source_braking_latched` in `convoy_env.py` (~99% penalty coverage)
- Gradual hazard injection (`slowDown(0, 2-4s)`, domain randomized)
- CF override after grace period
- `base_real`: `sigma=0.0`, `speedFactor=1.0`, `speedDev=0`, 25m spacing
- Hazard-gated reward: `PENALTY_IGNORING_HAZARD=-5.0`, `REWARD_EARLY_REACTION=+2.0`
- PPO: `LR=1e-4`, `n_steps=4096`, `ent_coef=0.0`, `log_std_init=-0.5`
- Dataset regenerated on EC2 from formation-fixed `base_real` (same params as v8)

### EC2 details

- RUN_ID: `cloud_prod_015`
- Training script: `ml/scripts/cloud/run_training.sh`
- S3 bucket: `s3://saferide-training-results/cloud_prod_015/`
- Instance type: c6i.xlarge
- Expected training time: ~18-19h

---

## Early diagnostic checkpoints

### At ~500k steps: critic alive check

```bash
grep "explained_variance" /var/log/training-run.log | tail -10
```

- If explained_variance > 0: critic is learning. Major improvement over Run 014.
- If explained_variance = 0: something else is wrong beyond HAZARD_PROBABILITY.

### At 1M steps: V2V reaction check

Copy checkpoint locally and run:

```bash
cd /home/amirkhalifa/RoadSense2/roadsense-v2v/ml

./run_docker.sh python3 -m ml.scripts.check_v2v_reaction \
  --model_path /work/results/cloud_prod_015_checkpoint_1M.zip \
  --dataset_dir /work/ml/scenarios/datasets/dataset_v6_formation_fix \
  --emulator_params /work/ml/espnow_emulator/emulator_params_measured.json \
  --output /work/results/cloud_prod_015_checkpoint_1M_v2v.json
```

### Interpretation matrix

| explained_variance at 1M | V2V reaction at 1M | Verdict |
|--------------------------|-------------------|---------|
| > 0.3 | > 0% | Run is alive — keep training |
| > 0.3 | 0% | Inconclusive — slowDown signal may be too weak, wait to 5M |
| 0 | 0% | Run is dead — slowDown is likely the fundamental problem |
| 0 | > 0% | Unlikely but would mean actor learned without critic |

### If Run 015 fails with working critic but 0% reaction

The gradual `slowDown` observation signal is the problem. Next steps would be:

1. Add explicit binary feature `hazard_braking_received` to ego observation (0/1 flag)
2. Or increase the observation signal strength (e.g., scale peer accel differently)
3. Or use a faster initial deceleration phase in the slowDown profile

---

## Pass criteria for Run 015

1. `explained_variance > 0.5` by 5M steps
2. V2V reaction > 90% in final eval (target 100%, Run 011 baseline)
3. `avg_reward > -200` (Run 011 was -86.62)
4. 0% collision rate
5. H5 re-validation: model action > 0.1 within 3s of peer braking onset

---

## Relevant files

- `ml/envs/hazard_injector.py` — `HAZARD_PROBABILITY = 0.5` (changed for Run 015)
- `ml/envs/convoy_env.py` — latch fix (from Run 014)
- `ml/envs/reward_calculator.py` — hazard-gated penalties
- `ml/scripts/cloud/run_training.sh` — EC2 launch script (updated for Run 015)
- `ml/scripts/check_v2v_reaction.py` — checkpoint evaluation tool
- `docs/10_PLANS_ACTIVE/RUN_013_ROOT_CAUSE_ANALYSIS.md` — Run 013 failure context
- `docs/10_PLANS_ACTIVE/RUN_011_ANALYSIS.md` — Run 011 (winner) reference
