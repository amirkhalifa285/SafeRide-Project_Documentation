# Run 020 — Deployment-Compatible Observation Design

**Date:** March 14, 2026
**Status:** Run concluded on March 15, 2026. The 2M diagnostic passed in SUMO, but real-data replay failed the deployment gate. Run 020 is not promotable to 10M and not deployable as-is. See `docs/10_PLANS_ACTIVE/RUN_020_POSTMORTEM.md`.
**Predecessor:** Run 019 (best SUMO model, NOT deployable)
**Goal:** Fix sim-to-real deployment gap by removing deployment-incompatible observation features while preserving all Run 018/019 critic-stability fixes.

---

## Outcome Summary (Post-Run Update)

Run 020 produced a mixed result:

- **SUMO diagnostic passed**
  - 2,000,000 timesteps completed successfully
  - `explained_variance` stayed healthy (`0.961` near 500k; `0.925` near run end)
  - final eval: 276 episodes, 0 collisions, 100% reaction across peer counts `n=1..5`
- **Real-data replay failed**
  - Recording #2: `12.0%` sensitivity, `6.07%` false positives
  - Extra Driving: `17.4%` sensitivity, `7.72%` false positives
- **Conclusion**
  - the deployment-incompatible permanent latch problem was removed
  - but the retrained policy became too conservative and missed most real braking events
  - Run 020 did not solve sim-to-real transfer

Additional experimental caveat:

- The plan said to preserve `dataset_v6`, but the actual cloud run used
  `ml/scenarios/datasets/dataset_v10_run020_post_cone`.
- That means Run 020 was not a clean observation-only A/B against Run 019.

Primary references:

- Training artifacts: `roadsense-v2v/ml/models/runs/cloud_prod_020/`
- Replay artifacts: `roadsense-v2v/ml/data/sim_to_real_validation_run020/`
- Postmortem: `docs/10_PLANS_ACTIVE/RUN_020_POSTMORTEM.md`

---

## 1. Problem Statement

Run 019 achieves 100% V2V reaction (276/276) in SUMO with 0% collisions, but **fails on real data**:

| Replay Mode | Recording #2 Sensitivity | Recording #2 FP | Extra Driving Sensitivity | Extra Driving FP |
|-------------|--------------------------|------------------|---------------------------|------------------|
| `latched`, `progress=0.4` | 100.0% | **85.3%** | 91.3% | **92.0%** |
| `off`, `progress=0.0` | **44.0%** | 6.3% | 78.3% | 21.8% |

**Root causes (two deployment-incompatible features):**

1. **`braking_received` (ego[5])** — Sticky boolean latch. In training, resets every 50s episode. In deployment/ESP32, never resets → permanent hazard mode after first routine braking event → 85-92% false positives.

2. **`progress` (ego[6])** — `step/max_steps`. No meaning in continuous deployment. Fixed to constant 0.4 in replay, but still a training-only feature.

---

## 2. Solution Design

### 2.1 Remove `progress` (ego[6])

No replacement needed. Deployment has no episode timer.

### 2.2 Replace sticky `braking_received` with exponential decay

```python
BRAKING_DECAY = 0.95      # per step at 10Hz → half-life ~1.35s
BRAKING_ACCEL_THRESHOLD = -2.5  # m/s² (unchanged)

# Every step:
cone_filtered_peers = obs_builder.filter_observable_peers(...)
if any(peer.accel <= BRAKING_ACCEL_THRESHOLD for peer in cone_filtered_peers):
    braking_received = 1.0          # reset to max
else:
    braking_received *= BRAKING_DECAY  # fade toward 0

# Every episode reset (training) / power-on (ESP32):
braking_received = 0.0
```

### 2.3 Why exponential decay (not removal)

Codex analysis confirmed: the feedforward PPO policy has zero temporal memory. Without _some_ form of "braking happened recently" signal, the model must react purely on the instantaneous `min_peer_accel` value, which returns to ~0 once the target stops. Run 011 got away with this because `setSpeed(0)` produced a persistent state change. `slowDown(0.5-1.5s)` is more realistic but transient.

The decay provides **bounded temporal memory** that is:
- Deployment-identical on ESP32 (one multiply per step)
- Self-clearing (no episode reset needed)
- Gradient-friendly for PPO (smooth continuous signal)
- Consistent with the visible peer set (no hidden pre-cone side-channel)
- Discriminative: sustained hazard braking keeps signal at 1.0 for the duration; brief routine braking produces a spike that fades within 2-3s

### 2.4 Decay signal profile

```
During hazard (0.5-1.5s):   braking_received = 1.0 (reset each step)
1s after braking ends:      0.60
2s after:                   0.36
3s after:                   0.21
5s after:                   0.08 (effectively zero)
```

### 2.5 Reward scaling (obs/reward alignment)

Reward terms scale by the decay value — no hidden hazard state, no obs/reward mismatch:

```python
if braking_received > 0.01:
    ignoring_hazard = PENALTY_IGNORING_HAZARD * braking_received   # -5.0 × decay
    early_reaction  = REWARD_EARLY_REACTION  * braking_received    # +2.0 × decay
```

During active hazard (`braking_received=1.0`): full -5.0 penalty / +2.0 bonus.
1s after (`0.60`): -3.0 / +1.2. Fades smoothly with the observation.

This preserves the Run 017 fix: reward and observation use the **same signal**.

---

## 3. Observation Space

### Before (Run 019, 7-dim ego)

| Dim | Feature | Scaling | Notes |
|-----|---------|---------|-------|
| 0 | speed | /30 | |
| 1 | accel | /10 | |
| 2 | heading | /180 | |
| 3 | peer_count | /8 | |
| 4 | min_peer_accel | /10 | |
| **5** | **braking_received** | **binary {0,1}** | **STICKY LATCH — deployment-incompatible** |
| **6** | **progress** | **raw [0,1]** | **EPISODE TIMER — deployment-incompatible** |

### After (Run 020, 6-dim ego)

| Dim | Feature | Scaling | Range | Notes |
|-----|---------|---------|-------|-------|
| 0 | speed | /30 | [0, 1] | unchanged |
| 1 | accel | /10 | [-1, 1] | unchanged |
| 2 | heading | /180 | [-1, 1] | unchanged |
| 3 | peer_count | /8 | [0, 1] | unchanged |
| 4 | min_peer_accel | /10 | [-1, 0] | unchanged |
| 5 | **braking_received_decay** | raw | **[0, 1]** | **EXPONENTIAL DECAY — deployment-compatible** |

`DeepSetExtractor`: `ego_feature_dim=6`, `features_dim=38` (32 embed + 6 ego).

---

## 4. Code Changes

### 4.1 Files Modified

| File | Changes |
|------|---------|
| `ml/envs/convoy_env.py` | `_braking_received_latched: bool` → `_braking_received_decay: float`; added `BRAKING_DECAY = 0.95`; removed progress computation; decay trigger now uses the cone-filtered peer set; updated obs space 7→6-dim; updated all init/reset/warmup-clear/step logic |
| `ml/envs/observation_builder.py` | Removed `progress` param from `build()`; `braking_received: bool` → `braking_received: float`; added shared `filter_observable_peers()` helper; ego array 7→6 dims |
| `ml/models/deep_set_extractor.py` | `ego_feature_dim=6`, `features_dim=38` |
| `ml/envs/reward_calculator.py` | `braking_received: bool` → `braking_received: float`; reward terms scaled by decay value |
| `ml/scripts/validate_against_real_data.py` | Added `decay` mode (default); replaced `_update_braking_received_state` with `_update_braking_received_decay`; removed progress; replay-side decay trigger now uses the cone-filtered peer set; kept legacy modes for regression analysis |

### 4.2 Signal Flow (Run 020)

```
_step_espnow():            cone_filtered_peers = obs_builder.filter_observable_peers(...)
                           any peer accel <= -2.5 in that set → _braking_received_decay = 1.0
                           else → _braking_received_decay *= 0.95

_step_espnow():            obs_builder.build(braking_received=decay_float)
step():                    reward_calculator.calculate(braking_received=decay_float)
obs_builder:               ego[5] = float(braking_received)
reward_calc:               ignoring_hazard = PENALTY * braking_received  (scaled)
                           early_reaction  = BONUS  * braking_received  (scaled)
replay validation:         uses the same cone-filtered helper before updating decay
```

### 4.3 Test Coverage

- 285 unit tests passing locally (0 failures)
- Docker integration suite passed after the final post-cone update (20/20)
- New tests added for decay logic in `test_validate_against_real_data.py`
- New tests added for exact reward fade cutoff and front-vs-behind cone behavior
- Updated dim assertions across all test files (7→6, 39→38)

---

## 5. What Is Preserved (DO NOT CHANGE)

| Component | Why |
|-----------|-----|
| `slowDown(0.5-1.5s)` hazard injection | Signal strength fix that ended 6-run dead-critic streak |
| `VecNormalize(norm_obs=False, norm_reward=True)` | Returns span -2500 to +2000 → unit-scale for value function |
| CF override during hazards | Without it → 0% V2V reaction (Runs 004, 009) |
| `HAZARD_PROBABILITY=1.0` | Worked with fast slowDown in Runs 018/019 |
| All PPO hyperparams | LR=1e-4, n_steps=4096, log_std_init=-0.5, ent_coef=0.0 |
| dataset_v6 (formation-fixed scenarios) | 25 train + 40 eval, seed=42 |
| Episode 500 steps, hazard window 150-350 | Formation-fixed geometry |
| Warmup contamination fix | Clear braking_received_decay=0.0 after warmup |

---

## 6. Training Plan

### Phase 1: 2M Diagnostic Run

Same as Run 018 diagnostic — proves the fixes work before committing to 10M.

**Kill criteria:**
- `explained_variance` must exceed 0.1 by 500k steps (Run 018 hit 0.71-0.93 by 2M)
- `explained_variance = 0` at 500k → KILL, diagnose
- V2V reaction at 2M checkpoint must exceed 50% (Run 018 hit 100%)

**Actual outcome:** PASS in SUMO. The cloud run reached 2,000,000 timesteps, kept a healthy critic, and finished with 100% reaction / 0 collisions in the eval matrix.

### Phase 2: 10M Production Run (if Phase 1 passes)

Same config as Phase 1, longer run. Monitor for reward regression (expected with std collapse, but V2V reaction and collision rate should hold).

**Actual outcome:** BLOCKED. Real-data replay failed, so 10M promotion should not proceed.

### Phase 3: Real-Data Re-Validation

```bash
cd /home/amirkhalifa/RoadSense2/roadsense-v2v
source ml/venv/bin/activate

python -m ml.scripts.validate_against_real_data \
    --tx_csv /path/to/V001_tx_004.csv \
    --rx_csv /path/to/V001_rx_004.csv \
    --model_path ml/results/cloud_prod_020/model_final.zip \
    --output_dir ml/data/sim_to_real_validation_run020 \
    --forward_axis y \
    --recording_name recording_02_decay

# Default mode is now 'decay' — no extra flags needed
# Legacy modes still available: --braking_received_mode latched/instant/off
```

**Targets:**
- Recording #2: sensitivity > 60%, false positives < 15%
- Extra Driving: sensitivity > 75%, false positives < 20%

**Actual outcome:** FAIL.

- Recording #2: `12.0%` sensitivity, `6.07%` false positives
- Extra Driving: `17.4%` sensitivity, `7.72%` false positives

The false-positive objective was met, but sensitivity collapsed on both recordings.

---

## 7. Risk Register

| Risk | Likelihood | Mitigation |
|------|-----------|-----------|
| Dead critic (explained_variance=0) | Medium — fast slowDown + decay + reward normalization are promising, but PPO learning dynamics remain unproven until the 2M diagnostic | Kill at 500k. Fallback: try decay=0.97 (longer memory) |
| 0% V2V reaction | Medium — decay provides bounded temporal memory, but this still needs a real training run to confirm | Kill at 2M. Fallback: bounded sliding-window binary feature (K=30 steps) |
| Still too many FP on real data (>25%) | Medium — model might learn too-low threshold | Lower decay to 0.90 (faster fade) in follow-up |
| Low sensitivity on real data (<50%) | Medium — real braking (-5 to -8 m/s²) slightly weaker than training (-9 to -10 m/s²) | Widen `BRAKING_DURATION_MAX` to 2.0s |

---

## 7.1 If Run 020 Is Still Too Sticky On Real Data

Run 020 is designed to eliminate the catastrophic permanent-latch failure, but it may still leave a short "afterglow" of caution after a hard braking event because the policy is feedforward and the training hazard remains active after the initial braking pulse.

If replay or deployment testing shows that the model still over-alerts or over-brakes for too long after traffic has normalized, the next fixes should be tried in this order:

1. **Faster decay**
   - Lower `BRAKING_DECAY` from `0.95` to `0.90`
   - Effect: shorter temporal memory, faster self-clear, lower false-positive tail
   - Tradeoff: may reduce sensitivity if the policy loses too much post-braking context

2. **Reward gating on both memory and current geometry**
   - Keep the recent-braking memory signal, but require the reward shaping to also depend on current danger geometry such as:
     - closing distance
     - closing rate
     - visible stopped/slow peer ahead
   - Effect: discourages the policy from treating "recent brake happened" as sufficient by itself once the scene is no longer dangerous

3. **Resolved-hazard training scenarios**
   - Add scenarios where the hazard brake event ends and traffic returns to normal instead of keeping the lead hazard pinned for the rest of the episode
   - Effect: teaches the policy an explicit "release back to normal driving" behavior
   - This is the cleanest long-term fix if post-event conservatism remains too high

These are follow-up levers, not part of the initial Run 020 diagnostic. The first goal of Run 020 is to confirm that post-cone decay preserves critic health and V2V reaction while removing the worst deployment mismatch.

---

## 8. Why This Is More Likely To Transfer To Real Data

| Signal Source | Magnitude (m/s²) | Normalized (/10) | vs Normal CF |
|---------------|-------------------|-------------------|-------------|
| Training slowDown (0.5s) | -10.0 (clamped) | -1.00 | 20x |
| Training slowDown (1.5s) | -9.3 | -0.93 | 18x |
| Real hard braking (measured) | -8.6 | -0.86 | 17x |
| Real moderate braking | -5.0 | -0.50 | 10x |
| Normal CF adjustment | -0.3 to -0.5 | -0.03 to -0.05 | baseline |

Real hard braking (-8.6 m/s²) is close to the training distribution. More importantly, the decay feature now provides the same high-level semantics in training, replay, and planned deployment: no sticky episode latch, no progress feature, and no broader-than-visible pre-cone trigger. This does **not** guarantee transfer, but it removes a major observation mismatch that previously inflated false positives.

---

## 9. Predecessor Analysis

| Feature | Run 019 (NOT deployable) | Run 020 (deployment-compatible) |
|---------|--------------------------|--------------------------------|
| Ego dims | 7 | 6 |
| braking_received | Sticky boolean (permanent after first trigger) | Cone-filtered exponential decay (half-life 1.35s) |
| progress | step/max_steps [0,1] | Removed |
| Reward gating | `if braking_received:` full penalty/bonus | `penalty * decay_value` — scaled |
| features_dim | 39 | 38 |
| Deployment semantics | Training ≠ deployment | Training = deployment |
| ESP32 compatibility | Requires episode reset (impossible) | Self-clearing decay (one multiply/step) |

---

## 10. File References

| File | Purpose |
|------|---------|
| `ml/envs/convoy_env.py` | Core env: decay logic, obs space, reward call |
| `ml/envs/observation_builder.py` | Ego obs construction (6-dim) |
| `ml/models/deep_set_extractor.py` | Architecture dims (ego=6, features=38) |
| `ml/envs/reward_calculator.py` | Decay-scaled reward terms |
| `ml/scripts/validate_against_real_data.py` | Real-data replay with decay default |
| `docs/10_PLANS_ACTIVE/RUN_019_SIM_TO_REAL_ANALYSIS.md` | Predecessor failure analysis |
| `docs/10_PLANS_ACTIVE/RUN_020_POSTMORTEM.md` | Final result analysis and decision |
| `docs/PROJECT_STATUS_OVERVIEW.md` | Project status (updated after Run 020 results) |
