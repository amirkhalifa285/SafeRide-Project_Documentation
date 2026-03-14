# Run 019 Sim-to-Real Validation Analysis

**Date:** March 14, 2026
**Model:** `cloud_prod_019` (Run 019 — `model_final.zip`, 10M steps)
**Data:** Recording #2 (195.5s) + Extra Driving (581.6s)
**Script:** `ml/scripts/validate_against_real_data.py`
**Prior art:** `docs/10_PLANS_ACTIVE/H5_SIM_TO_REAL_ANALYSIS.md` (Run 011 analysis, March 10)
**Verdict:** Model remains non-deployable on real data. Replay ablations confirm that the sticky `braking_received` latch is a major failure mode, but removing or bounding the latch exposes a different tradeoff: specificity improves while sensitivity collapses or remains inadequate. **Retraining is likely required.**

---

## 1. Executive Summary

Run 019 (10M PPO steps, `slowDown` hazard injection, `braking_received` feature) achieves **100% V2V reaction in SUMO** (276/276 episodes, 0% collisions). However, when replayed against real convoy data, the original replay semantics (`latched`, `progress=0.4`) produce **85-92% false positive rates**.

Replay ablations now show the stronger conclusion:
- The sticky `braking_received` latch is a **major** deployment mismatch and does inflate false positives dramatically.
- But it is **not the only issue**. With `progress=0.0` and `braking_received=off`, Recording #2 improves to **6.3% FP** but sensitivity falls to **44.0%**. Extra Driving still shows **21.8% FP** with only **78.3%** sensitivity.
- A 5-second timeout latch improves the tradeoff somewhat, but still fails deployment criteria: Recording #2 = **60.0% sensitivity / 28.0% FP**, Extra Driving = **91.3% sensitivity / 29.8% FP**.

The updated diagnosis is: **the replay artifact is real, but the underlying model still does not transfer cleanly to continuous real driving.**

### Key Numbers

| Replay Mode | Recording #2 | Extra Driving |
|-------------|--------------|---------------|
| `latched`, `progress=0.4` | 100.0% det / 85.3% FP | 91.3% det / 92.0% FP |
| `off`, `progress=0.0` | 44.0% det / 6.3% FP | 78.3% det / 21.8% FP |
| `instant`, `progress=0.0` | 52.0% det / 10.6% FP | 78.3% det / 22.5% FP |
| `latched`, `timeout=5000ms`, `progress=0.0` | 60.0% det / 28.0% FP | 91.3% det / 29.8% FP |

---

## 2. Run 019 SUMO Results (Reference)

Run 019 is a 10M-step production run building on Run 018's success (which proved `braking_received` + fast `slowDown` hazard injection works).

| Metric | Run 018 (2M) | Run 019 (10M) |
|--------|-------------|---------------|
| V2V Reaction | 100% (276/276) | 100% (276/276) |
| Collision Rate | 0% | 0% |
| Behavioral Success | 98.2% | 85.9% |
| Avg Reward | **+691.72** | **-2.48** |
| Reaction Time (R1) | 0.21s | 0.10s |
| std (final) | 0.091 | 0.052 |
| explained_variance | 0.93 | 0.94 |

**Reward regression note:** Run 019's avg_reward (-2.48) is lower than Run 018's (+691.72). Training logs show reward peaked at 1-3M steps (+500 to +600), then declined as std collapsed below 0.04. The model over-specialized — std=0.052 means nearly deterministic policy, possibly brittler than Run 018's std=0.091. Safety metrics (V2V reaction, collisions) are unaffected.

---

## 3. Validation Methodology

### Approach: Offline Replay (No SUMO)

Real convoy CSV data (TX = ego broadcasts, RX = peer V2V messages) is fed through the same `ObservationBuilder` used in training. The model produces actions at 10 Hz.

```
TX CSV (ego broadcasts)  -->  ego_state (speed, accel_y, heading, lat/lon)
RX CSV (peer V2V msgs)   -->  peer_observations (same fields per peer)
                                    |
                         ObservationBuilder.build()
                                    |
                         model.predict(obs, deterministic=True)
                                    |
                         action in [0, 1] (deceleration fraction)
```

### Commands Used

```bash
cd /home/amirkhalifa/RoadSense2/roadsense-v2v
source ml/venv/bin/activate

# Original replay (training-matching semantics)
python -m ml.scripts.validate_against_real_data \
    --tx_csv /home/amirkhalifa/RoadSense2/Convoy_recording_02282026/V001_tx_004.csv \
    --rx_csv /home/amirkhalifa/RoadSense2/Convoy_recording_02282026/V001_rx_004.csv \
    --model_path ml/results/cloud_prod_019/model_final.zip \
    --output_dir ml/data/sim_to_real_validation_run019 \
    --forward_axis y \
    --recording_name recording_02_latched_p04 \
    --progress 0.4 \
    --braking_received_mode latched

# Replay ablations
python -m ml.scripts.validate_against_real_data \
    --tx_csv /home/amirkhalifa/RoadSense2/Convoy_recording_02282026/V001_tx_004.csv \
    --rx_csv /home/amirkhalifa/RoadSense2/Convoy_recording_02282026/V001_rx_004.csv \
    --model_path ml/results/cloud_prod_019/model_final.zip \
    --output_dir ml/data/sim_to_real_validation_run019 \
    --forward_axis y \
    --recording_name recording_02_off_p0 \
    --progress 0.0 \
    --braking_received_mode off

python -m ml.scripts.validate_against_real_data \
    --tx_csv /home/amirkhalifa/RoadSense2/Convoy_extra_data_02282026/V001_tx_005.csv \
    --rx_csv /home/amirkhalifa/RoadSense2/Convoy_extra_data_02282026/V001_rx_005.csv \
    --model_path ml/results/cloud_prod_019/model_final.zip \
    --output_dir ml/data/sim_to_real_validation_run019 \
    --forward_axis y \
    --recording_name extra_driving_latched_timeout5s_p0 \
    --progress 0.0 \
    --braking_received_mode latched \
    --latch_timeout_ms 5000
```

### VecNormalize

Run 019 uses `VecNormalize(norm_obs=False, norm_reward=True)`. Since `norm_obs=False`, the model receives raw manually-scaled observations. The validate script correctly does NOT apply VecNormalize. This is confirmed correct.

---

## 4. Validation Results

### 4.1 Original Replay Result (`latched`, `progress=0.4`)

**Files:** `ml/data/sim_to_real_validation_run019/validation_report_recording_02_latched_p04.json`, `ml/data/sim_to_real_validation_run019/validation_report_extra_driving.json`

- **Duration:** 195.5s, 1956 steps at 100ms
- **Peers visible:** 96.5% of time
- **Real braking events:** 25

**Sensitivity:** 25/25 detected (100.0%)
- All events show `model_reacted: true`
- Reaction times: 0-101ms (mostly 0-1ms)
- Max model action during events: 0.148 to 0.429

**Specificity:** FAILED
- False positive rate: **85.3%**
- Avg calm action: 0.181
- Max calm action: 0.703
- Calm steps with peers: 1764

**Action Distribution:**
- Mean: 0.187, Std: 0.094, Median: 0.188
- Histogram heavily concentrated in 0.1-0.2 range (937 of 1956 steps)
- 86.5% of steps above threshold (0.1)

The original replay confirms the old failure:
- Recording #2: **100.0%** sensitivity, **85.3%** false positives
- Extra Driving: **91.3%** sensitivity, **92.0%** false positives

### 4.2 Replay Ablation Results (`progress=0.0`)

The replay script now supports explicit `braking_received` modes:
- `latched`
- `instant`
- `off`
- `latched` with `--latch_timeout_ms`

This isolates replay artifacts from underlying model behavior.

| Recording | Mode | Sensitivity | False Positive Rate | Avg Calm Action | Pct Steps > 0.1 |
|-----------|------|-------------|---------------------|-----------------|-----------------|
| Recording #2 | `off` | 44.0% (11/25) | 6.3% | 0.0147 | 6.9% |
| Recording #2 | `instant` | 52.0% (13/25) | 10.6% | 0.0212 | 11.5% |
| Recording #2 | `latched`, 5s timeout | 60.0% (15/25) | 28.0% | 0.0486 | 28.1% |
| Extra Driving | `off` | 78.3% (18/23) | 21.8% | 0.0658 | 20.8% |
| Extra Driving | `instant` | 78.3% (18/23) | 22.5% | 0.0699 | 21.6% |
| Extra Driving | `latched`, 5s timeout | 91.3% (21/23) | 29.8% | 0.0971 | 28.9% |

### 4.3 Interpretation

- The sticky latch **does** explain most of the catastrophic 85-92% false positive blow-up.
- But removing it does **not** yield a deployable model.
- Recording #2 becomes fairly calm when the feature is disabled, but then misses 40-56% of braking events.
- Extra Driving still over-brakes even with the latch removed entirely, which means the learned policy is still too hazard-biased for real continuous driving.

---

## 5. Root Cause Analysis

### 5.1 The `braking_received` Latch Problem (MAJOR CAUSE)

**What it is:**
`braking_received` (ego observation dim [5]) is a boolean feature that latches to `True` when any peer's acceleration falls below `BRAKING_ACCEL_THRESHOLD = -2.5 m/s^2`.

**How it works in training (`convoy_env.py`):**

```python
# Init (line 107)
self._braking_received_latched = False

# Reset per episode (line 188)
self._braking_received_latched = False

# Post-warmup clear (line 260)
self._braking_received_latched = False

# Latch trigger (lines 524-528)
if float(msg.accel_x) <= self.BRAKING_ACCEL_THRESHOLD:  # -2.5 m/s^2
    any_braking_peer_received = True
if update_braking_latch and any_braking_peer_received:
    self._braking_received_latched = True
# NOTE: Never set back to False within an episode
```

- Episodes are ~50s (500 steps at 100ms)
- Latch resets at the start of each episode
- Hazards occur at progress 0.3-0.7 (steps 150-350)
- Latch is True for at most ~35s per episode (if hazard at step 150, latch until step 500)

**How it works in the validate script (`validate_against_real_data.py`):**

```python
# Init (line 250)
braking_received_latched = False

# Latch trigger (lines 284-288)
for p in peer_obs:
    if p.get("accel", 0.0) <= V2V_BRAKING_ACCEL_THRESHOLD:  # -2.5 m/s^2
        braking_received_latched = True
        break
# NOTE: NEVER reset. Entire recording is one continuous "episode"
```

- Recording #2 is 195.5s. Extra is 581.6s. No episode boundaries.
- Once ANY peer decelerates below -2.5 m/s^2 (routine braking), the latch is True **forever**.
- The model was trained to brake aggressively when `braking_received=1`. So after the first routine braking event (likely within the first 10-30s), the model commands braking for the **entire remaining recording**.

**What the ablation proves:**
- On Recording #2, when peers are visible **before the first hard-braking trigger**, the model is effectively calm: average action `0.0019`, only `1.4%` of peer-visible pre-trigger steps above threshold.
- After the first hard-braking trigger under sticky-latch replay, average action jumps to `0.192`, and `89.3%` of remaining peer-visible steps are above threshold.

So the latch is not a cosmetic issue. It is a real mode-switch that drives the catastrophic replay behavior.

### 5.2 Why the Latch Is Not the Whole Story

If the latch were the whole problem, disabling it would produce both:
- low false positives, and
- high sensitivity

That does not happen.

- Recording #2 with `braking_received=off`: sensitivity drops to **44.0%**
- Recording #2 with `instant`: sensitivity only rises to **52.0%**
- Extra Driving with `braking_received=off`: false positives are still **21.8%**

This means the model is not merely "correct but fed a broken replay feature." The replay feature is broken, and the model also generalizes poorly once that crutch is removed.

### 5.3 How `braking_received` Affects Model Behavior

The model learned a clear conditional policy during training:
- `braking_received=0` → calm driving, low action (~0.05)
- `braking_received=1` → hazard mode, elevated action (0.2-0.7+)

This is **correct behavior for the training environment** where `braking_received=1` genuinely means "a hazard is occurring." But in continuous real-world operation:
- Routine braking (traffic light, lane change, speed adjustment) regularly produces accel < -2.5 m/s^2
- The latch triggers and never resets
- The model enters permanent hazard mode

### 5.4 The Deployment Problem

**This is not just a validation script bug.** The same latch design would fail on the ESP32:
- ESP32 runs continuously (no episode resets)
- Real traffic produces frequent deceleration events below -2.5 m/s^2
- The latch would trigger within minutes of operation and never clear
- The vehicle would command permanent partial braking

### 5.5 Secondary Issues

**Pre-cone vs post-cone mismatch:**
The validate script checks ALL peers from the RX CSV for the braking threshold. Training checks only post-mesh/cone-filtered peers (from the emulator's `received_map`). In convoy recordings where peers are ahead of ego, this difference is negligible because all peers pass the cone filter anyway.

**Progress feature:**
`progress` is still a deployment-mismatched feature. The replay ablations used `progress=0.0` to reduce its bias. In local checks, `progress=0.4` consistently increased false positives relative to `progress=0.0`, especially on Extra Driving.

### 5.6 Does `HAZARD_PROBABILITY = 1.0` Explain This?

Partially, but not completely.

- `HAZARD_PROBABILITY = 1.0` does remove all-calm episodes, and earlier runs showed that this can starve the critic when the signal is weak.
- However, Run 018 and Run 019 still achieved a healthy critic in simulation after the later signal-strength and normalization fixes, so `1.0` by itself does not explain the real-data failure.
- The current ablations also show the model is **not** braking unconditionally from timestep 0. It becomes highly brake-happy after the replay enters hazard-coded regimes.

So the strongest defensible statement is:
- `HAZARD_PROBABILITY = 1.0` may have increased hazard bias and weakened calm-driving contrast,
- but the current real-data failure is more specifically tied to deployment-incompatible observation semantics (`braking_received`, `progress`) plus incomplete real-world generalization.

---

## 6. Changes Made to `validate_against_real_data.py`

### 6.1 Progress Feature Fix (APPLIED)

**Problem:** The script used `step_idx / total_replay_steps` as the progress value. For a 5817-step recording, progress goes 0.0→1.0 linearly. But training uses `step / 500` and hazards occur at progress 0.3-0.7. The model has never seen most progress values from the validate script.

**Fix:** Added `--progress` CLI argument (default: constant 0.4).

```python
# New CLI arg (line 520-523)
parser.add_argument("--progress", type=str, default="0.4",
                    help="Progress feature value: 'linear' for step/total ramp, "
                         "or a float constant (default: 0.4). In live deployment "
                         "there are no episodes, so a constant mimics deployment.")

# New replay logic (lines 295-298)
if progress_mode == "linear":
    replay_progress = step_idx / max(1, total_replay_steps)
else:
    replay_progress = progress_value
```

**Status:** Applied and used in the validation run. Results already include `progress_mode: "constant"`, `progress_value: 0.4`.

### 6.2 Replay-Side `braking_received` Modes (APPLIED)

The script now supports:
- `--braking_received_mode latched`
- `--braking_received_mode instant`
- `--braking_received_mode off`
- `--latch_timeout_ms N`

This is sufficient to separate replay artifacts from underlying policy behavior without hand-editing the script.

---

## 7. Replay Findings

The three useful conclusions from the new replay matrix are:

1. **Sticky-latch replay is invalid for deployment assessment.**
   It massively overstates sensitivity and false positives.

2. **Timeout latch is not enough.**
   A 5-second timeout still leaves false positives far above deployable levels.

3. **The current model needs both the replay fix and a policy fix.**
   The best replay settings found here still do not produce an acceptable sensitivity/specificity tradeoff.

---

## 8. The Retraining Question

### 8.1 Does the Model Work?

**In SUMO: Yes, unambiguously.** 100% V2V reaction, 0% collisions, 15/15 eval buckets.

**On real data: No, not in a deployable sense.** The replay bug exaggerated the failure, but the ablations show that the underlying model still does not achieve an acceptable tradeoff once replay semantics are made more realistic.

### 8.2 Scenarios That Require Retraining

The replay ablations rule out the simplest degenerate case ("it only works when ego[5] is 1 forever"), but they still point to retraining:

- `off` and `instant` do **not** collapse to 0% reaction, so the model is using more than the sticky latch.
- But `off` / `instant` also do **not** preserve enough sensitivity.
- `timeout=5000ms` improves sensitivity but still leaves false positives too high.

**Current conclusion:** retraining is likely required, but it should be done carefully to avoid revisiting:
- 0% V2V reaction
- dead critic / `explained_variance = 0`

### 8.3 The `progress` Feature

`progress` (ego dim [6]) = `current_step / max_steps`. It has no real-world meaning in continuous deployment. Currently fixed to constant 0.4 in the validate script.

**Should we retrain without it?**

- If the model is insensitive to progress (likely, since it's just a timer), keeping it as a constant is fine.
- If the model has learned to gate behavior on progress (e.g., "only brake when progress > 0.3"), then constant 0.4 works because hazards occur at 0.3-0.7.
- The safest approach is to retrain without progress if retraining for other reasons.

**Recommendation:** If retraining proceeds, remove `progress` and redesign `braking_received` to have deployment-compatible semantics. This drops ego dims from 7→5 if `braking_received` is removed entirely:
```
[speed/30, accel/10, heading/180, peer_count/8, min_peer_accel/10]
```
`DeepSetExtractor.features_dim` would change from 39 to 37 (32 embed + 5 ego).

### 8.4 Retraining Scope (If Needed)

If retraining is required, the changes are:
1. **`convoy_env.py`:** Remove or redesign `braking_received_latched` logic. Remove `progress` computation. Update `_step_espnow()` and `step()`.
2. **`observation_builder.py`:** Remove `braking_received` and `progress` from `build()`. Ego dims 7→5.
3. **`deep_set_extractor.py`:** Update `ego_feature_dim` 7→5, `features_dim` 39→37.
4. **Observation space:** Update `spaces.Dict` ego low/high bounds (7→5).
5. **Hazard injection:** No changes needed — `slowDown` injection from Run 018/019 is correct.
6. **Dataset:** Reuse dataset_v6 (formation-fixed). No regeneration needed.
7. **Hyperparameters:** Keep Run 018/019 hyperparams (LR=1e-4, n_steps=4096, etc.). Run 2M diagnostic first, then 10M if successful.

---

## 9. Observation Space Reference

### Current (Run 019, 7-dim ego)

| Dim | Feature | Scaling | Training Range | Notes |
|-----|---------|---------|----------------|-------|
| 0 | speed | /30 | [0, 1] | m/s divided by 30 |
| 1 | accel | /10 | [-1, 1] | m/s^2 divided by 10 |
| 2 | heading | /180 | [-1, 1] | degrees divided by 180 |
| 3 | peer_count | /8 | [0, 1] | n peers divided by MAX_PEERS |
| 4 | min_peer_accel | /10 | [-1, 0] | lowest peer accel, divided by 10 |
| **5** | **braking_received** | **binary** | **{0, 1}** | **PROBLEMATIC: sticky latch, no reset** |
| **6** | **progress** | **raw** | **[0, 1]** | **PROBLEMATIC: no deployment meaning** |

### Proposed (if retraining, 5-dim ego)

| Dim | Feature | Scaling | Range |
|-----|---------|---------|-------|
| 0 | speed | /30 | [0, 1] |
| 1 | accel | /10 | [-1, 1] |
| 2 | heading | /180 | [-1, 1] |
| 3 | peer_count | /8 | [0, 1] |
| 4 | min_peer_accel | /10 | [-1, 0] |

The model would need to learn hazard reaction purely from `min_peer_accel` (ego[4]) and the raw peer observations (6-dim per peer: relative distance, bearing, speed, accel, heading, staleness).

---

## 10. File References

| File | Purpose | Key Lines |
|------|---------|-----------|
| `ml/scripts/validate_against_real_data.py` | H5 validation script | L48-54 (constants), L250 (latch init), L284-288 (latch trigger), L295-298 (progress fix) |
| `ml/envs/convoy_env.py` | Training env | L107 (latch init), L188 (reset), L260 (post-warmup clear), L524-528 (latch trigger) |
| `ml/envs/observation_builder.py` | Obs construction | L123-134 (7-dim ego array) |
| `ml/models/deep_set_extractor.py` | Deep Sets architecture | `ego_feature_dim=7`, `features_dim=39` |
| `ml/results/cloud_prod_019/model_final.zip` | Trained model | Run 019, 10M steps |
| `ml/data/sim_to_real_validation_run019/` | Validation output | Reports + timeseries .npz |
| `docs/10_PLANS_ACTIVE/H5_SIM_TO_REAL_ANALYSIS.md` | Run 011 analysis (prior) | Same issue with setSpeed(0) |

---

## 11. Timeline of Changes (This Session)

1. **Run 019 analysis:** Confirmed 100% V2V reaction in SUMO, 0% collisions. Reward regression (-2.48 vs +691.72) from std over-collapse.
2. **Validate script audit:** Identified 3 compatibility issues (progress, braking_received latch, pre/post-cone).
3. **Progress fix applied:** Added `--progress` CLI arg, default constant 0.4.
4. **Validation run executed:** Both recordings processed under original replay semantics. Results showed 85-92% FP rates.
5. **Replay script extended:** Added `--braking_received_mode` and `--latch_timeout_ms`.
6. **Replay ablations executed:** `off`, `instant`, and `latched+5000ms timeout` rerun on both recordings at `progress=0.0`.
7. **Updated diagnosis:** sticky latch is a major issue, but replay fixes alone do not make Run 019 deployable.

---

## 12. Decision Matrix

| Observed result | Implication | Confidence |
|-----------------|-------------|------------|
| `off` / `instant` keep nonzero reaction | Model is not using only the sticky latch | High |
| `off` / `instant` lose too much sensitivity | Replay fixes alone are insufficient | High |
| `timeout=5000ms` keeps better sensitivity but FP remains 28-30% | Timeout latch alone is not deployable | High |
| `progress=0.0` is materially better than `progress=0.4` | `progress` is a real deployment mismatch | High |

---

## 13. Next Steps

1. **Do not deploy Run 019 as-is.**
2. **Use the replay matrix above as the new baseline** for any future sim-to-real checks.
3. **Design a retraining plan that preserves the Run 018/019 critic fixes** while removing deployment-incompatible observation semantics.
4. **Specifically guard against regression** to:
   - 0% V2V reaction
   - dead critic / `explained_variance = 0`
5. **Candidate retraining direction:** remove `progress`, replace sticky `braking_received` with either:
   - no such feature at all, or
   - a bounded sliding-window signal used identically in training and deployment
