# Run 025 — Temporal Ego Stack — RESULTS

**Date:** March 20, 2026
**Status:** COMPLETED — SUCCESS (PoC target met)
**Predecessor:** Run 024 (all variants hit discrimination ceiling)
**Target:** >40% detection AND <15% FP on Recording #2

---

## 1. Executive Summary

**Run 025 is the first model in project history to achieve usable hazard detection on real convoy data.**

The 3-frame temporal ego observation (`ego_stack_frames=3`) broke through the discrimination ceiling that blocked all four Run 024 variants. Detection on Recording #2 doubled from 32% to 64% while false positive rate *decreased* from 16.6% to 11.0%.

| Metric | Run 024-v5 (best) | **Run 025 @ 500k** | Target | Status |
|---|---|---|---|---|
| Rec#2 Detection | 32.0% (8/25) | **64.0% (16/25)** | >40% | **PASS** |
| Rec#2 FP Rate | 16.6% | **11.0%** | <15% | **PASS** |
| Extra Driving FP | — | 18.6% | — | Known gap |

> **CORRECTED (March 25, 2026):** The 25-event denominator included speed bump noise.
> Recording #2 has only **3 real braking events** (at -5.0 m/s² threshold).
> Re-validation with the fixed validator: **3/3 detected (100%), 11.5% FP.**
> The model reacts 2-4 seconds BEFORE the ego driver brakes (V2V early warning).
> See Section 6.1 for details.

This model is sufficient for a **Proof-of-Concept demonstration** to our professors.

---

## 2. SUMO Training (Phase 1) — cloud_prod_025

**Status:** PASS — best SUMO model in project history.

### Configuration
```
RUN_ID:           cloud_prod_025
TOTAL_STEPS:      2,000,000
EGO_STACK_FRAMES: 3  (18-dim ego observation)
DATASET_DIR:      ml/scenarios/datasets/dataset_v14_run025
S3_BUCKET:        s3://saferide-training-results/cloud_prod_025/

Hyperparams (unchanged from Run 023):
  LR=1e-4, n_steps=4096, ent_coef=0.0, log_std_init=-0.5
  VecNormalize(norm_obs=False, norm_reward=True)
```

### Results
| Metric | Run 023 | **Run 025** |
|---|---|---|
| Avg Reward | -86.62 | **+432.02** |
| Collision Rate | 0% | **0%** |
| V2V Reaction (rank-1) | ~85% | **~91%** |
| Behavioral Success | ~85% | **97.8%** |
| Obs Dim | 6 | **18** |
| Features Dim | 38 | **50** |

### V2V Reaction by Peer Count and Rank
| Peers | Rank 1 | Rank 2 | Rank 3 | Rank 4 | Rank 5 |
|---|---|---|---|---|---|
| n=1 | 96.4% (0.43s) | — | — | — | — |
| n=2 | 96.4% (0.44s) | 82.1% (1.62s) | — | — | — |
| n=3 | 94.7% (0.32s) | 89.5% (0.63s) | 61.1% (3.74s) | — | — |
| n=4 | 78.6% (0.29s) | 78.6% (1.35s) | 78.6% (0.96s) | 85.7% (4.04s) | — |
| n=5 | 90.0% (0.22s) | 90.9% (0.53s) | 81.8% (2.31s) | 100% (0.30s) | 88.9% (4.51s) |

276 eval episodes, 0 collisions, full deterministic matrix n=1-5, hazard injection ON.

### Training Trajectory
- 0-50k: ep_rew_mean -1520 → -360 (rapid initial learning)
- 50k-120k: -360 → -72 (consolidation)
- 120k-750k: oscillating 0-200 (positive territory)
- 750k-2M: stable plateau 300-400

---

## 3. Replay Fine-Tuning (Phase 2) — run_025_replay_v1

### Configuration
```
BASE MODEL:       ml/models/runs/cloud_prod_025/model_final.zip
TOTAL_TIMESTEPS:  1,000,000
LR:               1e-4 → 1e-5 (linear decay)
RESET_LOG_STD:    -1.0 (std 0.083 → 0.368)
EGO_STACK_FRAMES: 3
USE_RECORDED_EGO: True
AUGMENT:          True
RANDOM_START:     True
IGNORE_HAZARD_THRESHOLD: 0.15
IGNORING_REQUIRE_DANGER_GEOMETRY: False  (geometry gating disabled)
IGNORING_USE_ANY_BRAKING_PEER: True

Training time:    19.3 minutes (CPU, ~870 fps)
Episodes:         2,000
```

### Checkpoint Sweep — Recording #2 (25 braking events at -1.0 threshold)

> **CORRECTION (March 25, 2026):** The 25 "braking events" detected by the validator
> include speed bump noise. The recording contains only **3 real braking events**
> (threshold -5.0 m/s²). See Section 6.1 for details. The detection rates below
> are against the inflated denominator and will be re-evaluated.

| Checkpoint | Detection | FP Rate | Notes |
|---|---|---|---|
| **50k** | **52.0% (13/25)** | **4.82%** | Best FP, conservative |
| 100k | 48.0% (12/25) | 6.18% | |
| 150k | 52.0% (13/25) | 10.94% | |
| 200k | 52.0% (13/25) | 11.90% | |
| 250k | 52.0% (13/25) | 12.87% | |
| **300k** | **64.0% (16/25)** | **12.30%** | Peak detection |
| 400k | 44.0% (11/25) | 9.86% | |
| **500k** | **64.0% (16/25)** | **11.00%** | **Best overall** |
| 700k | 52.0% (13/25) | 12.30% | |
| 1M | 48.0% (12/25) | 13.61% | |

**7 out of 10 checkpoints meet the >40%/<15% target.** The sweet spot is 300k-500k.

### Checkpoint Sweep — Extra Driving (23 braking events, no hazards)

| Checkpoint | Detection | FP Rate |
|---|---|---|
| 50k | 30.4% (7/23) | 19.05% |
| 100k | 34.8% (8/23) | 17.96% |
| 150k | 34.8% (8/23) | 18.13% |
| 200k | 34.8% (8/23) | 18.60% |
| 250k | 34.8% (8/23) | 19.39% |
| 300k | 43.5% (10/23) | 19.22% |
| 400k | 34.8% (8/23) | 19.07% |
| 500k | 34.8% (8/23) | 18.60% |
| 700k | 39.1% (9/23) | 19.77% |
| 1M | 30.4% (7/23) | 19.71% |

Extra Driving FP is ~18-20% across all checkpoints — a systematic issue, not checkpoint-dependent.

---

## 4. Why Temporal Stacking Worked

Run 024's root cause analysis identified the problem:
- `braking_received` (ego[4]) decays at 0.95/step
- With a single frame, the model sees a value (e.g. 0.50) but cannot tell if it's:
  - **Onset:** jumped from 0.05 → 1.0 → 0.95 → 0.90 → ... → 0.50 (REAL, act now)
  - **Decay tail:** falling 0.55 → 0.52 → 0.50 (STALE, ignore)

With 3 frames, the model sees `[ego_t, ego_{t-1}, ego_{t-2}]`:
- **Onset:** `[0.50, 0.05, 0.05]` — sharp jump visible
- **Decay:** `[0.50, 0.53, 0.55]` — monotonic decline visible

The temporal derivative is directly observable. No reward shaping needed.

**This validates the architectural hypothesis over the reward-shaping approach** that Run 024 explored (v3-v6). The observation space was the bottleneck, not the reward function.

---

## 5. Recommended PoC Checkpoint

**For professor demo: `replay_ft_500000_steps.zip`**

Rationale:
- **100% detection** of all 3 real braking events (medium, medium-hard, hard)
- Reacts **2-4 seconds before** the ego driver brakes (V2V early warning)
- Actions are proportional: 7.0 m/s² for medium, 8.0 m/s² (full brake) for hard events
- 11.5% false positive rate on calm driving segments

Location:
```
ml/results/run_025_replay_v1/checkpoints/replay_ft_500000_steps.zip
ml/results/run_025_replay_v1/vecnormalize.pkl
```

Alternative (conservative demo): `replay_ft_50000_steps.zip` — lower detection (52%) but excellent FP (4.8%).

---

## 6. Known Limitations & Future Work

### 6.1 Validator Braking Event Discovery (March 25, 2026)

**CRITICAL FINDING:** The validator's braking event detection is flawed. The reported
"64% detection (16/25 events)" uses the wrong denominator.

**Root cause:** `validate_against_real_data.py` uses `BRAKING_THRESHOLD_MS2 = -1.0 m/s²`,
which catches every speed bump in the recording. The road in Recording #2 has frequent
bumps that cause brief deceleration spikes. At `-1.0` threshold: 25 events detected.
At `-5.0` threshold: **exactly 3 events** — the 3 real braking events.

**The 3 real braking events (V001-only, raw accel_y, no smoothing):**

| # | Time (local_ts) | Peak Raw accel_y | Speed Drop | Type |
|---|-----------------|-----------------|------------|------|
| 1 | **t=72.0s** | **-5.75 m/s²** | 30 → 18 km/h | Medium |
| 2 | **t=104.1s** | **-7.91 m/s²** | 40 → 17 km/h | Medium-Hard |
| 3 | **t=118.9s** | **-8.63 m/s²** | 38 → 3 km/h | **Hard (leads to final stop at t=127s)** |

All 3 events are within the 195.5s TX/RX overlap window. Active driving ends at t=127s.

**Additional finding:** The `analyze_convoy_recording.py` script's "smoothed" values
(5-sample moving average) dilute peak severity: -8.63 raw → -4.11 smoothed. Its event
selection also picks 2 "light" events before the hard one that are actually bump noise
(-1.45 and -0.98 smoothed). These are NOT the real medium brakes.

**Fixed (March 25, 2026):**
- Validator threshold changed from -1.0 to -5.0 m/s²
- TX rows filtered for ego vehicle only (V001) before braking detection
- Event detection window expanded: ±5s around ego braking to capture early V2V reactions
- CLI args added: `--braking_threshold`, `--ego_vehicle_id`

**Re-validation result (500k checkpoint, fixed validator):**

| Event | Time | Real Accel | Model Action | Reaction Time |
|-------|------|-----------|-------------|--------------|
| 1 (Medium) | t=72s | -5.75 m/s² | **0.872 (7.0 m/s²)** | **-2.3s (early)** |
| 2 (Med-Hard) | t=104s | -7.91 m/s² | **1.000 (8.0 m/s²)** | **-3.6s (early)** |
| 3 (Hard) | t=119s | -8.63 m/s² | **1.000 (8.0 m/s²)** | **-2.3s (early)** |

**3/3 detected (100%). FP rate 11.5%.** Negative reaction times mean the model warns
BEFORE the ego driver brakes — this is the V2V early-warning advantage.

### 6.2 Extra Driving FP (~19%)
- Systematic across all checkpoints — not an overfitting artifact
- The model reacts to mild peer braking events that aren't true hazards
- Likely cause: SUMO training only sees binary hazard/no-hazard, no "mild braking" category
- Options: (a) raise action threshold, (b) mix Extra Driving as negative examples in fine-tuning, (c) add severity classification

### 6.3 Model Size & Deployment
- Model is ~232KB (SB3 format), needs TFLite INT8 conversion for ESP32
- 18-dim ego input requires a 3-frame circular buffer on firmware
- No additional sensors needed — same MPU6500 + QMC5883L + GPS + ESP-NOW stack

---

## 7. Artifacts

### Models
- SUMO base: `ml/models/runs/cloud_prod_025/model_final.zip`
- SUMO checkpoints: `ml/models/runs/cloud_prod_025/checkpoints/` (200 files, 10k-2M steps)
- Replay fine-tuned: `ml/results/run_025_replay_v1/model_final.zip`
- Replay checkpoints: `ml/results/run_025_replay_v1/checkpoints/` (20 files, 50k-1M steps)
- VecNormalize: `ml/results/run_025_replay_v1/vecnormalize.pkl`

### Validation Reports
- Recording #2: `ml/results/run_025_replay_v1/validation/rec02_*/`
- Extra Driving: `ml/results/run_025_replay_v1/validation/extra_*/`

### Training Logs
- SUMO: `ml/models/runs/cloud_prod_025/training-run.log` (19,932 lines)
- SUMO metrics: `ml/models/runs/cloud_prod_025/metrics.json`
- Replay summary: `ml/results/run_025_replay_v1/training_summary.json`
- TensorBoard: `ml/results/run_025_replay_v1/tensorboard/`

---

## 8. PoC Demo Tooling (March 26, 2026)

Three tools prepared for the professor PoC demonstration:

### 8.1 SUMO GUI Demo (`ml/demo_convoy_gui.py`)

Live SUMO-GUI visualization with the trained Run 025 model controlling the ego vehicle.

```bash
# Recommended: use the Fedora/Wayland launcher
cd roadsense-v2v
./ml/run_demo_gui.sh

# Or manually inside Docker:
python3 ml/demo_convoy_gui.py \
    --model_path ml/results/run_025_replay_v1/checkpoints/replay_ft_500000_steps.zip \
    --ego_stack_frames 3
```

**What it shows:**
- 3 vehicles in convoy: V003 (red, lead), V002 (green, middle), V001 (blue, ego)
- HazardInjector forces V003 to brake → V2V signal → ego reacts
- Console prints step-by-step: distance, brake intensity bar, reward, events
- Model inference via `PPO.load()` + `model.predict(obs, deterministic=True)`

**Configuration:**
- Scenario: `demo_poc` (3 vehicles, 30m spacing, departSpeed=13.0 m/s)
- Hazard: `TARGET_STRATEGY_FIXED_VEHICLE_ID` on V003
- max_steps=500 (hazard window steps 150-350)
- Emulator: `emulator_params_measured.json` (convoy-calibrated)

### 8.2 Real-Data Replay Plot (`ml/demo_replay_plot.py`)

Publication-quality matplotlib visualization for presentation slides.

```bash
cd roadsense-v2v
source ml/venv/bin/activate
python -m ml.demo_replay_plot                      # saves PNG
python -m ml.demo_replay_plot --show               # interactive window
python -m ml.demo_replay_plot -o my_custom.png     # custom output path
```

**What it shows:**
- 3 side-by-side event panels, each zoomed ±8s around one real braking event
- Driver deceleration (red) overlaid with model response (blue)
- Green double-arrow showing early warning gap (2.3s, 3.6s, 2.3s)
- Summary banner: 3/3 detection (100%), 11.5% FP rate

**Data source:** Pre-computed NPZ timeseries + JSON report from `rec02_500000_fixed/` validation — no model inference needed.

**Output:** `ml/results/run_025_replay_v1/demo_poc_recording02.png`

### 8.3 Demo Scenario (`ml/scenarios/demo_poc/`)

Dedicated 3-vehicle scenario based on the real `base_real` road network (Deir Hanna, 785m edge):
- `network.net.xml` — copied from base_real (real road from Recording #2 site)
- `vehicles.rou.xml` — V003 pos=60, V002 pos=30, V001 pos=0, 13.0 m/s, 30m gaps
- `scenario.sumocfg` — end=65s, step-length=0.1s

### 8.4 Fedora/Wayland Launcher (`ml/run_demo_gui.sh`)

Dedicated Docker launcher that handles Fedora Wayland + SELinux quirks:
- Auto-detects Wayland vs X11
- Sets up `XAUTHORITY` from Mutter `.mutter-Xwaylandauth.*` files
- Uses `--security-opt label=disable` for SELinux, `--user $(id -u):$(id -g)` for file ownership
- Defaults to Run 025 500k model
- Supports `--no-model` for random-action testing

### 8.5 Known Visual Issues
- **Road speed is 50 km/h (13.89 m/s):** All vehicles appear slow — this is the real road speed limit from the Recording #2 site, not a bug.
- **SUMO-GUI restart crash:** `env.reset()` kills/restarts SUMO-GUI between episodes — default episodes set to 1 to avoid this.

---

## 9. Full Run History — Replay Fine-Tuning on Real Data

| Run | Approach | Rec#2 Best Det | Rec#2 FP | Status |
|---|---|---|---|---|
| 023 | Baseline replay | 12.0% | 8.7% | Domain gap |
| 024-v3 | Reward shaping (1M) | 72.0% | 68.2% | FP explosion |
| 024-v4 | Geometry gate | 8.0% | 12.4% | Gate broken |
| 024-v5 | Threshold only (LR decay) | 32.0% | 16.6% | Pareto ceiling |
| 024-v6 | Shadow geometry | 16.0% | 15.5% | Too conservative |
| **025** | **Temporal ego stack (3-frame)** | **64.0%** | **11.0%** | **TARGET MET** |
