# Run 021 — Hazard Distribution Fix

**Date:** March 15, 2026
**Status:** Ready to launch
**Predecessor:** Run 020 (deployment-compatible obs, SUMO pass, replay fail)
**Goal:** Fix sim-to-real sensitivity collapse by diversifying hazard braking intensity and adding resolved-hazard episodes.

---

## 1. Problem Statement

Run 020 proved the deployment-compatible observation design works in SUMO (100% reaction, 0 collisions, healthy critic) but failed decisively on real-data replay:

| Recording | Sensitivity | Target | False Positive Rate | Target |
|-----------|-------------|--------|---------------------|--------|
| Recording #2 | **12.0%** | >60% | 6.07% | <15% |
| Extra Driving | **17.4%** | >75% | 7.72% | <20% |

### Root Cause: Two Mismatches

**Mismatch 1 — Accel distribution gap:**
- Training: hazard peer always brakes at -9.3 to -10.0 m/s² (extreme)
- Real data: 80%+ of braking events are -2.5 to -5.0 m/s² (moderate)
- The model never sees moderate braking → decision boundary only activates at extreme values
- Confirmed by real-data distribution: Recording #2 had 108/128 threshold-crossing samples in [-5, -2.5], only 20 at ≤ -5

**Mismatch 2 — Persistent-hazard semantics:**
- Training: after braking pulse, hazard target is pinned at speed 0 for the rest of the episode
- Real data: braking events are transient — peer brakes, slows, then resumes normal driving
- Model learned to react to the combined pattern (braking signal + closing gap with stopped vehicle), not just the braking signal

---

## 2. Solution Design

### 2.1 Domain-Randomize Desired Deceleration

Instead of always braking to speed 0, randomize the **desired deceleration** per episode:

```python
HAZARD_DECEL_MIN = 3.0    # m/s² — moderate braking
HAZARD_DECEL_MAX = 10.0   # m/s² — emergency (clamped by SUMO)

desired_decel = rng.uniform(HAZARD_DECEL_MIN, HAZARD_DECEL_MAX)
target_speed = max(0.0, current_speed - desired_decel * braking_duration)
sumo.slow_down(target, target_speed, braking_duration)
```

Resulting training distribution at cruise speed 13.9 m/s:

| Desired Decel | Duration | Peer Accel | Normalized | Covers |
|--------------|----------|-----------|-----------|--------|
| 3.0 m/s² | 1.0s | -3.0 | -0.30 | Moderate braking (bulk of real data) |
| 5.0 m/s² | 1.0s | -5.0 | -0.50 | Strong braking |
| 8.0 m/s² | 1.0s | -8.0 | -0.80 | Hard braking |
| 10.0 m/s² | 0.5s | -10.0 | -1.00 | Emergency (current training) |

HAZARD_DECEL_MIN=3.0 is 6x above normal CF adjustments (-0.5 m/s²), preserving the critic's ability to distinguish hazard from calm episodes.

### 2.2 Resolved-Hazard Episodes

40% of hazards **resolve** — the target releases to CF after a short delay:

```python
HAZARD_RESOLVE_PROB = 0.4       # 40% resolve
RESOLVE_DELAY_MIN = 20          # steps (2.0s) after slowdown
RESOLVE_DELAY_MAX = 50          # steps (5.0s) after slowdown
```

Execution:
1. Slowdown completes → target pinned at hazard_target_speed
2. After resolve delay → `release_vehicle_speed(target)` returns to CF
3. Target accelerates back to normal cruise speed

What this teaches:
- **60% persistent**: peer brakes → stops/slows → stays → ego must brake and hold (strong learning signal)
- **40% resolved**: peer brakes → slows → resumes → ego should brake briefly, then release

The decay signal handles the transition naturally: during braking, decay=1.0 → penalty fires → model brakes. After braking stops, decay fades over ~3s → penalty fades → model releases.

### 2.3 After Slowdown: Pin at Target Speed (not always 0)

`maintain_hazard()` now pins at the computed `hazard_target_speed` instead of always 0.0. For persistent hazards, this creates a "slow-moving obstacle" when target_speed > 0 (more realistic than always stopping). For resolved hazards, the pin is temporary before CF release.

---

## 3. What Changed (Code)

| File | Change |
|------|--------|
| `ml/envs/hazard_injector.py` | Added `HAZARD_DECEL_MIN/MAX`, `HAZARD_RESOLVE_PROB`, `RESOLVE_DELAY_MIN/MAX`. `maybe_inject()` randomizes desired decel, computes target speed from current speed, decides resolve. `maintain_hazard()` pins at target speed (Phase 1), releases to CF for resolved hazards (Phase 2). |
| `ml/envs/convoy_env.py` | Added `hazard_desired_decel`, `hazard_target_speed`, `hazard_resolved` to step info dict for diagnostics. |
| `ml/tests/unit/test_hazard_injector.py` | Updated existing tests for variable target speed. Added 7 new tests: decel range, target speed computation, zero clamp, resolve fraction, resolve delay range, resolved-hazard CF release, persistent-hazard no-release. |
| `ml/scripts/cloud/run_training.sh` | Updated for Run 021 (RUN_ID, comments, dataset name). |

---

## 4. What Is Preserved (DO NOT CHANGE)

| Component | Why |
|-----------|-----|
| Decay observation (ego[5]) | Deployment-compatible, proven trainable in Run 020 |
| `BRAKING_DECAY = 0.95` | Correct half-life for deployment semantics |
| `BRAKING_ACCEL_THRESHOLD = -2.5` | Matches real-data event definition |
| Reward structure (decay-scaled) | Obs/reward aligned, proven in Run 020 |
| `BRAKING_DURATION_MIN/MAX = 0.5/1.5` | Signal stays sharp and distinct from CF |
| `VecNormalize(norm_obs=False, norm_reward=True)` | Reward normalization stability |
| CF override during hazards | Required for V2V reaction (Runs 004, 009) |
| `HAZARD_PROBABILITY=1.0` | Every episode gets a hazard |
| All PPO hyperparams | LR=1e-4, n_steps=4096, ent_coef=0.0, log_std_init=-0.5 |
| Deep Sets architecture | features_dim=38 (32 embed + 6 ego) |
| Episode 500 steps, hazard window 150-350 | Formation-fixed geometry |
| Warmup contamination fix | Clear braking_received_decay=0.0 after warmup |

---

## 5. Training Plan

### Phase 1: 2M Diagnostic Run

**Kill criteria:**
- `explained_variance > 0.1` by 500k steps → if 0 at 500k, KILL
- V2V reaction at 2M must exceed 50%

### Phase 2: Real-Data Replay Validation (MANDATORY before 10M)

```bash
cd /home/amirkhalifa/RoadSense2/roadsense-v2v
source ml/venv/bin/activate

python -m ml.scripts.validate_against_real_data \
    --tx_csv /home/amirkhalifa/RoadSense2/Convoy_recording_02282026/V001_tx_004.csv \
    --rx_csv /home/amirkhalifa/RoadSense2/Convoy_recording_02282026/V001_rx_004.csv \
    --model_path ml/models/runs/cloud_prod_021/model_final.zip \
    --output_dir ml/data/sim_to_real_validation_run021 \
    --forward_axis y \
    --recording_name recording_02_decay

python -m ml.scripts.validate_against_real_data \
    --tx_csv /home/amirkhalifa/RoadSense2/Convoy_extra_data_02282026/V001_tx_005.csv \
    --rx_csv /home/amirkhalifa/RoadSense2/Convoy_extra_data_02282026/V001_rx_005.csv \
    --model_path ml/models/runs/cloud_prod_021/model_final.zip \
    --output_dir ml/data/sim_to_real_validation_run021 \
    --forward_axis y \
    --recording_name extra_driving_decay
```

**Acceptance targets:**

| Recording | Sensitivity | False Positive Rate |
|-----------|-------------|---------------------|
| Recording #2 | > 60% | < 15% |
| Extra Driving | > 75% | < 20% |

### Phase 3: 10M Production Run (only if Phase 1 AND Phase 2 pass)

Same config, longer run. Do NOT extend to 10M without replay validation passing.

---

## 6. Risk Register

| Risk | Likelihood | Mitigation |
|------|-----------|-----------|
| Critic collapse from mixed intensities | Low — decay feature distinguishes hazard/calm; HAZARD_DECEL_MIN=3.0 is 6x above CF | Kill at 500k. Fallback: raise DECEL_MIN to 5.0 |
| SUMO reaction drops on mild hazards | Medium — mild hazards produce small closing rate | PENALTY_IGNORING_HAZARD fires whenever braking_received > 0.01, independent of closing rate |
| FP increase from over-sensitized model | Medium — model may brake at every minor peer decel | Current FP is 6-7%, headroom to 15-20%. Replay gate catches it. Resolved-hazard episodes teach "release" behavior which helps contain FP tails |
| Resolved hazards produce weak gradient signal | Low — 60% of episodes are still persistent (strong signal), resolved episodes add breadth | If reaction drops, reduce HAZARD_RESOLVE_PROB to 0.2 |

---

## 7. Predecessor Comparison

| Aspect | Run 020 | Run 021 |
|--------|---------|---------|
| Hazard target speed | Always 0.0 | max(0, current - decel × dur) |
| Desired decel | Fixed ~10 m/s² | Uniform [3.0, 10.0] m/s² |
| Hazard persistence | Always pinned at 0 forever | 60% persistent, 40% resolve after 2-5s |
| Observation | 6-dim ego + decay (unchanged) | Same |
| Reward | Decay-scaled (unchanged) | Same |
| Architecture | Deep Sets (unchanged) | Same |
