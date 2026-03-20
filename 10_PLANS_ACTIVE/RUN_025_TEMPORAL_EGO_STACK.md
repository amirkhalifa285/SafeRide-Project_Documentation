# Run 025 — Temporal Ego Stack

**Date:** March 19, 2026
**Status:** LAUNCHED on EC2 (`cloud_prod_025`)
**Predecessor:** Run 024 (all variants hit discrimination ceiling)
**Target:** >40% detection AND <15% FP on Recording #2

---

## 1. Why This Run Exists

Run 024 explored four reward-shaping variants on the replay fine-tuning pipeline.
All hit the same ceiling: the model cannot discriminate real hazard onsets from
stale braking signal decay using a single-timestep observation.

| Variant | Best Detection | FP at Best | Failure Mode |
|---------|---------------|------------|--------------|
| v3 1M   | 72.0%         | 68.2%      | FP explosion — brakes on everything |
| v4 300k | 8.0%          | 12.4%      | Geometry gate killed signal (recorded-ego bug) |
| v5 150k | 32.0%         | 16.6%      | Pareto front but can't go higher without FP blowup |
| v6 150k | 16.0%         | 15.5%      | Shadow geometry made learning harder |

**Root cause:** The ego observation is 6-dim, single-timestep. The `braking_received`
signal (ego[4]) decays at 0.95/step, but the model sees only the current value. It
cannot distinguish:
- **Onset:** braking_received jumps from 0.05 → 1.0 (REAL, act now)
- **Decay tail:** braking_received at 0.50 and falling (STALE, ignore)

Reward shaping cannot fix an observation space that lacks discriminative power.

## 2. What Changed (Code)

**Single change:** Stack the last N ego observation vectors into a flat vector.

```
Before: ego = [speed, accel, peer_count, min_peer_accel, braking_received, max_closing_speed]  → (6,)
After:  ego = [ego_t, ego_{t-1}, ego_{t-2}]  → (18,)  (N=3)
```

Peers and peer_mask remain single-frame (unchanged).

### Files Modified

| File | What Changed |
|------|-------------|
| `ml/envs/replay_convoy_env.py` | `ego_stack_frames` param, `deque` history, `_stack_ego()` helper, tiled obs space bounds |
| `ml/envs/convoy_env.py` | Same stacking + `empty_obs` ego dim fix for early-exit paths |
| `ml/policies/deep_set_policy.py` | `ego_feature_dim` kwarg passthrough in `create_deep_set_policy_kwargs()` |
| `ml/models/deep_set_extractor.py` | **No change** — already supported `ego_feature_dim` param |
| `ml/envs/observation_builder.py` | **No change** — stays stateless, builds single-frame (6,) ego |
| `ml/training/train_convoy.py` | `--ego_stack_frames` CLI arg, wired to env kwargs + policy kwargs + metrics |
| `ml/scripts/run_replay_fine_tuning.py` | `--ego_stack_frames` CLI arg, wired to env factory + summary |
| `ml/scripts/validate_against_real_data.py` | `--ego_stack_frames` CLI arg, deque stacking in `run_replay()` |
| `ml/scripts/cloud/run_training.sh` | Updated for Run 025: RUN_ID, dataset, `--ego_stack_frames 3`, removed `--eval_hazard_step 200` |
| `ml/tests/unit/test_ego_stack.py` | **19 new tests** (stacking shape, order, history, bounds, backward compat, extractor, policy kwargs) |
| `ml/tests/unit/test_train_convoy.py` | Added `ego_stack_frames=1` to `_make_args()` defaults |

### Key Design Properties

- **Backward compatible:** `ego_stack_frames=1` (default) produces identical behavior to pre-Run 025
- **Stacking order:** `[ego_t, ego_{t-1}, ..., ego_{t-N+1}]` — most recent first, so `obs["ego"][0:6]` is always the current frame
- **DeepSetExtractor:** features_dim = 32 + 18 = **50** (was 38)
- **History in env, not builder:** `ObservationBuilder` stays a pure function. Episode-scoped deque lives in the env.
- **Fresh SUMO training:** No weight transfer from Run 023 — obs dim change breaks checkpoint compat

## 3. How to Revert if This Goes Wrong

Everything is backward compatible. To go back to Run 023 behavior:

1. **Don't pass `--ego_stack_frames`** (or pass `--ego_stack_frames 1`) — all code defaults to 1
2. **Use Run 023 model** (`s3://saferide-training-results/cloud_prod_023/model_final.zip`)
3. **Use Run 023 cloud script** — `git show HEAD~1:ml/scripts/cloud/run_training.sh` has the original

No files need to be reverted. The `ego_stack_frames=1` default preserves the exact Run 023 observation pipeline.

## 4. EC2 Training Configuration

```
RUN_ID:           cloud_prod_025
TOTAL_STEPS:      2,000,000
EGO_STACK_FRAMES: 3
DATASET_DIR:      ml/scenarios/datasets/dataset_v14_run025
S3_BUCKET:        s3://saferide-training-results/cloud_prod_025/

Hyperparams (unchanged from Run 023):
  LR=1e-4, n_steps=4096, ent_coef=0.0, log_std_init=-0.5
  VecNormalize(norm_obs=False, norm_reward=True)

Eval config:
  --eval_force_hazard (every eval episode gets a hazard)
  --eval_use_deterministic_matrix (covers n=1-5 peer counts)
  --eval_matrix_episodes_per_bucket 10
  NO --eval_hazard_step (state-triggered onset timing for more realistic eval)
```

## 5. Test Results Before Launch

```
Unit tests:        426 passed (19 new + 407 existing)
Integration tests:  20 passed (Docker + SUMO)
Codex review:      No blocking findings
```

## 6. What Happens After SUMO Training

1. **Check SUMO eval:** V2V reaction >= ~85%, 0% collisions (Run 023 baseline)
2. **Replay fine-tune:** `--ego_stack_frames 3 --use_recorded_ego --reset_log_std -1.0 --ignore_hazard_threshold 0.15`
3. **Validate on Recording #2 + Extra Driving**
4. **Success criterion:** any checkpoint achieves >40% detection AND <15% FP
5. **If ceiling holds:** escalate to action-dependent observations (kinematic ego training)

## 7. Known Residual Risks (non-blocking)

1. `ego_feature_dim = ego_stack_frames * 6` is manually coupled in `train_convoy.py` and `deep_set_policy.py`. Fine unless single-frame ego dim changes (it hasn't since Run 023).
2. No metadata assertion that a trained model expects N stacked frames. Mismatch crashes loudly (dimension error in forward pass), so it's not silent — just an operator burden.
