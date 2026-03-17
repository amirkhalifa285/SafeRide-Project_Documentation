# Run 022 — Remove Ego Heading from Observation

**Date:** March 15, 2026
**Status:** Complete — SUMO pass, replay fail
**Completed:** March 16, 2026
**Predecessor:** Run 021 (hazard distribution fix — SUMO pass, replay FAIL)
**Goal:** Fix sim-to-real false positive catastrophe by removing ego heading (spurious feature).
**Outcome:** Heading removal fixed the Run 021 false-positive blow-up, but replay sensitivity remained too low for promotion (`12.0% / 2.55%` on Recording #2, `26.1% / 4.58%` on Extra Driving).
**See also:** `docs/10_PLANS_ACTIVE/RUN_022_POSTMORTEM.md`, `docs/10_PLANS_ACTIVE/RUN_023_HYPOTHESIS_SET.md`

---

## 1. Problem Statement

Run 021 improved sensitivity significantly (12%→40% Rec#2, 17%→91% Extra) via hazard decel randomization, but introduced catastrophic false positives:

| Recording | Sensitivity | Target | FP Rate | Target | Verdict |
|-----------|-------------|--------|---------|--------|---------|
| Recording #2 | 40.0% | >60% | 19.16% | <15% | FAIL |
| Extra Driving | 91.3% | >75% | **93.83%** | <20% | FAIL |

The model outputs near-maximum braking 88.6% of the time on Extra Driving (mean action = 0.69, median = 0.80), while staying calm on Recording #2 (mean action = 0.07).

### Root Cause: Ego Heading is a Spurious Feature

**Discovery method:** Systematic model probing — swept each observation dimension independently while holding others constant.

**Heading sweep results (all other features at training baseline):**

| Heading (normalized) | Model Action |
|---------------------|-------------|
| -1.00 (-180°) | **1.000** |
| -0.50 (-90°) | **0.996** |
| -0.25 (-45°) | **0.718** |
| 0.00 (0°) | 0.406 |
| +0.25 (+45°) | 0.071 |
| +0.50 (+90°) | **0.000** |
| +1.00 (+180°) | **0.000** |

**Heading is the single strongest predictor of model action**, swinging output from 0.0 to 1.0 across its range — far more influential than braking_received (0.41→0.68), min_peer_accel (0.41→0.87), or peer distance.

**Why this happened:** In SUMO training, all episodes run on a fixed route (base_real). Hazards always occur at specific route positions, which correspond to specific headings. The model learned: "heading in range X → hazard zone → brake hard." This is a textbook spurious correlation that breaks on any route with different headings.

**Recording heading distributions confirm:**

| Recording | Heading Mean (norm) | Dominant Zone | Model Behavior |
|-----------|-------------------|---------------|----------------|
| Recording #2 | +0.78 | [+0.75, +1.0) → action≈0.05 | Calm (correct zone) |
| Extra Driving | -0.33 | [-0.5, 0.0) → action≈0.63-0.70 | Max braking (wrong zone) |

Even Recording #2's 42 steps with heading < -0.5 show action=1.0, confirming the pattern is universal.

**Why heading should NOT be in ego obs:** Ego's absolute heading has zero causal relationship with V2V hazards. The information that matters — relative geometry between ego and peers — is already captured in peer observations (peer[3] = relative heading). The cone filter handles directional filtering.

---

## 2. Solution Design

### Remove Ego Heading from Observation

Ego vector changes from 6-dim to 5-dim:

| Index | Run 021 (6-dim) | Run 022 (5-dim) |
|-------|----------------|----------------|
| 0 | speed/30 | speed/30 |
| 1 | accel/10 | accel/10 |
| 2 | **heading/180** | peer_count/8 |
| 3 | peer_count/8 | min_peer_accel/10 |
| 4 | min_peer_accel/10 | braking_received |
| 5 | braking_received | *(removed)* |

Deep Sets features_dim: 38 → 37 (32 embed + 5 ego).

No other changes. Peer observations (6-dim per peer, including relative heading) are unchanged.

---

## 3. What Changed (Code)

| File | Change |
|------|--------|
| `ml/envs/observation_builder.py` | Removed heading line from ego array. Docstring updated. |
| `ml/envs/convoy_env.py` | Ego observation space shape (6,)→(5,), low/high arrays updated. Two `np.zeros(6)` → `np.zeros(5)` for empty_obs. Comment ego[5]→ego[4]. |
| `ml/models/deep_set_extractor.py` | Default `ego_feature_dim=5` (was 6). Docstring: batch shapes and features_dim comment. |
| `ml/envs/reward_calculator.py` | Comment: ego[5]→ego[4]. |
| `ml/tests/unit/test_observation_builder.py` | ego shape (6,)→(5,), index shifts for peer_count (3→2), min_peer_accel (4→3), braking_received (5→4). |
| `ml/tests/unit/test_deep_set_extractor.py` | Observation space (6,)→(5,), batch ego dim, features_dim 38→37 in test name and assertion. |
| `ml/tests/unit/test_convoy_env.py` | ego shape assertions (6,)→(5,), braking_received index 5→4. |
| `ml/tests/integration/test_run006_fixes.py` | ego shape, min_peer_accel index 4→3, braking_received index 5→4. |
| `ml/tests/integration/test_scenario_switching.py` | ego shape (6,)→(5,). |
| `ml/tests/integration/test_convoy_env_integration.py` | ego shape assertions (6,)→(5,), observation_space ego shape. |
| `ml/scripts/validate_against_real_data.py` | **Launch hotfix (March 15):** replay validator updated for the 5-dim ego obs. `min_peer_accel` now reads from `ego[3]`, `braking_received` from `ego[4]`. Usage/help text updated so Phase 2 validation matches Run 022 obs layout. |
| `ml/tests/unit/test_validate_against_real_data.py` | Added regression test that replays a braking peer and verifies exported `min_peer_accel`/`braking_received` use the current 5-dim ego indices. |
| `ml/scripts/cloud/run_training.sh` | Updated for Run 022 (RUN_ID, comments, dataset name). |

---

## 4. What Is Preserved (DO NOT CHANGE)

| Component | Why |
|-----------|-----|
| Hazard decel randomization [3.0, 10.0] m/s² | Run 021 fix — diversifies braking intensity |
| Resolved-hazard episodes (40%) | Run 021 fix — teaches transient braking response |
| Decay observation (ego[4] now) | Deployment-compatible, proven trainable |
| `BRAKING_DECAY = 0.95` | Correct half-life for deployment semantics |
| `BRAKING_ACCEL_THRESHOLD = -2.5` | Matches real-data event definition |
| Reward structure (decay-scaled) | Obs/reward aligned, proven in Run 020 |
| `BRAKING_DURATION_MIN/MAX = 0.5/1.5` | Signal stays sharp and distinct from CF |
| `VecNormalize(norm_obs=False, norm_reward=True)` | Reward normalization stability |
| CF override during hazards | Required for V2V reaction (Runs 004, 009) |
| `HAZARD_PROBABILITY=1.0` | Every episode gets a hazard |
| All PPO hyperparams | LR=1e-4, n_steps=4096, ent_coef=0.0, log_std_init=-0.5 |
| Peer observations (6-dim per peer) | Relative heading still in peer[3] |
| Episode 500 steps, hazard window 150-350 | Formation-fixed geometry |
| Warmup contamination fix | Clear braking_received_decay=0.0 after warmup |

---

## 5. Training Plan

### Phase 1: 2M Diagnostic Run

**Kill criteria:**
- `explained_variance > 0.1` by 500k steps → if 0 at 500k, KILL
- V2V reaction at 2M must exceed 50%

### Phase 2: Real-Data Replay Validation (MANDATORY before 10M)

**Pre-flight note (March 15 hotfix):**
- `validate_against_real_data.py` originally still assumed the old 6-dim ego layout from Run 021 (`ego[4]`/`ego[5]`).
- This is fixed before launch. The replay validator now matches Run 022's 5-dim ego obs (`ego[3]` = `min_peer_accel`, `ego[4]` = `braking_received`).

```bash
cd /home/amirkhalifa/RoadSense2/roadsense-v2v
source ml/venv/bin/activate

python -m ml.scripts.validate_against_real_data \
    --tx_csv /home/amirkhalifa/RoadSense2/Convoy_recording_02282026/V001_tx_004.csv \
    --rx_csv /home/amirkhalifa/RoadSense2/Convoy_recording_02282026/V001_rx_004.csv \
    --model_path ml/models/runs/cloud_prod_022/model_final.zip \
    --output_dir ml/data/sim_to_real_validation_run022 \
    --forward_axis y \
    --recording_name recording_02_decay

python -m ml.scripts.validate_against_real_data \
    --tx_csv /home/amirkhalifa/RoadSense2/Convoy_extra_data_02282026/V001_tx_005.csv \
    --rx_csv /home/amirkhalifa/RoadSense2/Convoy_extra_data_02282026/V001_rx_005.csv \
    --model_path ml/models/runs/cloud_prod_022/model_final.zip \
    --output_dir ml/data/sim_to_real_validation_run022 \
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
| Critic collapse without heading | Very Low — heading had no causal signal, was pure overfitting. Removing it gives the critic cleaner features. | Kill at 500k if explained_variance=0. |
| V2V reaction drops | Low — Run 021 achieved 95.3% V2V reaction using min_peer_accel + braking_received, neither of which depends on heading. | Monitor at 2M; must exceed 50%. |
| FP improvement insufficient | Medium — heading was the dominant FP source, but other spurious correlations may exist (e.g., speed, peer distance distributions). | Replay gate catches this. If FP still high, investigate other feature correlations. |
| Sensitivity drops vs Run 021 | Low — sensitivity was driven by hazard decel randomization (unchanged). Heading removal should not affect sensitivity. | Replay gate checks both directions. |

---

## 7. Predecessor Comparison

| Aspect | Run 020 | Run 021 | Run 022 |
|--------|---------|---------|---------|
| Ego observation dim | 6 (with heading) | 6 (with heading) | **5 (no heading)** |
| features_dim | 38 (32+6) | 38 (32+6) | **37 (32+5)** |
| Hazard target speed | Always 0.0 | max(0, current - decel × dur) | Same as 021 |
| Desired decel | Fixed ~10 m/s² | Uniform [3.0, 10.0] m/s² | Same as 021 |
| Hazard persistence | Always pinned at 0 | 60% persistent, 40% resolve | Same as 021 |
| Rec#2 sensitivity | 12.0% | 40.0% | TBD |
| Rec#2 FP | 6.07% | 19.16% | TBD |
| Extra sensitivity | 17.4% | 91.3% | TBD |
| Extra FP | 7.72% | **93.83%** | TBD |

---

## 8. Evidence: Heading Probing Data

Full probing results that confirmed heading as root cause:

```
=== Heading sweep (training-like obs, 1 peer 30m ahead) ===
heading=-1.00 (-180°): action=1.000
heading=-0.50 (-90°):  action=0.996
heading=-0.25 (-45°):  action=0.718
heading= 0.00 (  0°):  action=0.406
heading=+0.25 (+45°):  action=0.071
heading=+0.50 (+90°):  action=0.000
heading=+0.75 (+135°): action=0.000
heading=+1.00 (+180°): action=0.000

=== Recording heading → action correlation ===
Recording #2:
  heading [+0.75, +1.00): 1808 steps (92%), mean_action=0.050
  heading [-1.00, -0.50):   42 steps ( 2%), mean_action=1.000

Extra Driving:
  heading [-1.00, -0.50): 1605 steps (28%), mean_action=0.887
  heading [-0.50, -0.25): 1638 steps (28%), mean_action=0.703
  heading [-0.25, +0.00): 1642 steps (28%), mean_action=0.634
  heading [+0.75, +1.00):  237 steps ( 4%), mean_action=0.000
```

Even with braking_received=0 (no hazard signal at all), the model outputs mean action=0.74 on Extra Driving, purely driven by heading.
