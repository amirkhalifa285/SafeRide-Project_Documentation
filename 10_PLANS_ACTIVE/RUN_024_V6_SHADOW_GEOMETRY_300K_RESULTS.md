# Run 024-v6 - Shadow Reward Geometry 300k Probe

**Date:** March 19, 2026
**Status:** COMPLETED - FAILED
**Predecessor:** Run 024-v5 (`threshold-only + LR decay`)
**Plan doc:** `RUN_024_REPLAY_CONVOY_ENV_PLAN.md` Section 12.13

---

## 1. What Changed

This probe implemented the shadow-ego reward path:
- `ReplayConvoyEnv` kept **recorded ego** for observations
- reward `distance`, `closing_rate`, and `ego_speed` came from a parallel
  `EgoKinematics` rollout driven by the policy action
- replay fine-tuning exposed this via `--use_shadow_reward_geometry`

Goal:
- keep the validator-matching observation contract
- restore a **causal** geometry signal for reward
- make the model brake on Recording #2's real hard-brake event without
  re-entering the false-positive explosion

---

## 2. Training Configuration

```json
{
  "base_model": "ml/models/runs/cloud_prod_023/model_final.zip",
  "total_timesteps": 300000,
  "learning_rate": "1e-4 -> 1e-5 (linear decay)",
  "reset_log_std": -1.0,
  "use_recorded_ego": true,
  "use_shadow_reward_geometry": true,
  "reward_config": {
    "ignoring_hazard_threshold": 0.15,
    "ignoring_require_danger_geometry": true,
    "ignoring_use_any_braking_peer": true,
    "ignoring_danger_distance": 20.0,
    "ignoring_danger_closing_rate": 0.5,
    "early_reaction_threshold": 0.01
  }
}
```

**Training summary:**
- Training time: `5.8 min`
- Episodes completed: `606`
- Final `ep_rew_mean`: `-1127.7`

---

## 3. Recording #2 Results

### Checkpoint sweep

| Checkpoint | Detection | FP |
|---|---|---|
| 100k | 12.0% (3/25) | 11.05% |
| **150k** | **16.0% (4/25)** | **15.48%** |
| 200k | 12.0% (3/25) | 12.02% |
| 300k | 0.0% (0/25) | 4.93% |

### Hard-brake thesis event

At the strongest real braking point in Recording #2:
- real ego accel: `-8.63 m/s²`
- timestamp: `235.073s`

Model response:
- `150k` checkpoint: action `0.147` -> `~1.17 m/s²` decel
- `300k` final: action `0.000` -> `0.0 m/s²`

This is the decisive failure: the probe still does **not** brake meaningfully
at the thesis event.

---

## 4. Extra Driving Results

| Checkpoint | Detection | FP |
|---|---|---|
| **150k** | **13.0% (3/23)** | **23.51%** |
| 300k | 8.7% (2/23) | 16.48% |

The run did suppress false positives relative to aggressive replay variants, but
it did so by becoming too conservative overall. Sensitivity collapsed on both
recordings.

---

## 5. Interpretation

### What worked

- The implementation was technically correct:
  - focused replay unit tests passed (`51/51`)
  - reward telemetry clearly separated recorded geometry from shadow geometry
  - training was stable enough to finish the 300k probe

### What failed

- **Shadow reward geometry did not recover Recording #2 sensitivity.**
- The best checkpoint (`150k`) is still far below the v5 frontier
  (`16.0% / 15.5%` vs v5 `32.0% / 16.6%` at 150k).
- Extending to `300k` made the policy even more conservative
  (`0.0% / 4.9%` on Recording #2).

### Likely reason

The geometry signal became causal again, but it was still only available through
the **reward**, not the observation. In practice the policy appears to have
optimized toward "stay calm unless the reward makes danger undeniable," which
reduces FP but does not teach the early braking behavior we need for real
Recording #2 replay.

In short:
- this implementation fixes the **structural masking** bug
- but by itself it does **not** create a strong enough learning signal for the
  real hard-brake response

---

## 6. Conclusion

**Run 024-v6 is a negative result.**

Shadow reward geometry, in this recorded-observation / shadow-reward form, is
**not** the production path by itself. It does not produce a model that brakes
adequately on Recording #2, and it still misses the `-8.63 m/s²` thesis event.

Artifacts:
- `ml/results/run_024_replay_v6_shadow_geometry_300k/`
- `ml/results/run_024_replay_v6_shadow_geometry_300k/validation/`
