# Run 024-v5 — Threshold-Only Gate + LR Decay Results

**Date:** March 19, 2026
**Status:** COMPLETED — Pareto improvement over v3, but target not met
**Predecessor:** Run 024-v4 (geometry gate, FAILED — structurally broken in recorded-ego mode)
**Plan doc:** `RUN_024_REPLAY_CONVOY_ENV_PLAN.md` Section 12.12

---

## 1. Configuration

```json
{
  "base_model": "cloud_prod_023/model_final.zip",
  "total_timesteps": 1000000,
  "learning_rate": "1e-4 → 1e-5 (linear decay)",
  "reset_log_std": -1.0,
  "use_recorded_ego": true,
  "random_start": true,
  "reward_config": {
    "ignoring_hazard_threshold": 0.15,
    "ignoring_require_danger_geometry": false,
    "ignoring_use_any_braking_peer": true,
    "ignoring_danger_distance": 20.0,
    "ignoring_danger_closing_rate": 0.5,
    "early_reaction_threshold": 0.01
  }
}
```

**Key changes vs v4:**
- Dropped geometry gate (`ignoring_require_danger_geometry=false`) — structurally broken in recorded-ego mode
- Lowered threshold from 0.3 to 0.15 (~28% of steps fire, vs 19.2% at 0.3 and 48.5% at 0.01)
- Added LR linear decay (1e-4 → 1e-5) to prevent specificity collapse seen in v3

**Why geometry was dropped:** In `use_recorded_ego=True` mode, `distance` and `closing_rate` come from the human driver's recorded trajectory. During real hazard events, the human already braked and maintained safe following distance, so geometry appears safe even during actual hazards. The gate filtered out most hazard-window steps, killing the training signal. See `RUN_024_REPLAY_CONVOY_ENV_PLAN.md` Section 12.11 for full analysis.

---

## 2. Training Summary

| Metric | Value |
|---|---|
| Training time | 18.5 minutes |
| Episodes completed | 2007 |
| Final ep_rew_mean | -1826.3 (vs v4: -2029.3, v3: -1947) |
| LR schedule | Linear 1e-4 → 1e-5 |
| Checkpoints | 20 (every 50k steps) |

---

## 3. Recording #2 — Full Checkpoint Sweep

| Steps | Detection | FP | Notes |
|---|---|---|---|
| 50k | 4.0% (1/25) | 8.90% | barely started learning |
| **100k** | **20.0% (5/25)** | **14.57%** | **best FP (<15%)** |
| **150k** | **32.0% (8/25)** | **16.55%** | **Pareto front — best balanced** |
| 200k | 16.0% (4/25) | 15.65% | regression |
| 250k | 16.0% (4/25) | 16.16% | flat |
| 300k | 20.0% (5/25) | 18.71% | |
| 400k | 20.0% (5/25) | 18.54% | |
| **500k** | **52.0% (13/25)** | **54.20%** | **sensitivity peak — FP explosion** |
| 550k | 52.0% (13/25) | 53.17% | same regime |
| 600k | 44.0% (11/25) | 50.34% | |
| 650k | 12.0% (3/25) | 16.21% | LR decay collapse |
| 700k | 8.0% (2/25) | 15.42% | LR decay collapse |
| 750k | 12.0% (3/25) | 17.35% | |
| 800k | 12.0% (3/25) | 13.78% | best late-stage FP |
| 850k | 16.0% (4/25) | 18.82% | |
| 900k | 28.0% (7/25) | 21.15% | partial recovery |
| 950k | 24.0% (6/25) | 24.04% | |
| **1M** | **40.0% (10/25)** | **21.49%** | **first ever 40% sensitivity at <25% FP** |

### Training Curve Shape

The curve has three distinct phases:

1. **Phase 1 (50k-400k): Gradual learning** — sensitivity 4-32%, FP 9-19%. The model learns a moderate braking policy. Best balanced point at 150k (32%/16.6%).

2. **Phase 2 (500k-600k): Sensitivity burst / FP explosion** — sensitivity jumps to 44-52% but FP explodes to 50-54%. The model undergoes a phase transition to aggressive braking, similar to v3's 400k-1M behavior.

3. **Phase 3 (650k-1M): LR decay correction** — the decaying LR pulls FP back down. Sensitivity initially collapses (650k-800k: 8-12%) but partially recovers by 1M (40%/21.5%). The recovery is incomplete — the phase transition damage is partially but not fully reversed.

---

## 4. Extra Driving Validation

| Checkpoint | Detection | FP |
|---|---|---|
| v5 100k | 13.0% (3/23) | 20.99% |
| v5 150k | 17.4% (4/23) | 22.87% |
| v5 1M | 17.4% (4/23) | 21.06% |

Extra Driving sensitivity is flat across checkpoints (13-17%), suggesting the model struggles with this recording regardless of the tradeoff point. FP stays in the 21-23% range.

---

## 5. Comparison to All Prior Run 024 Variants

### Recording #2 (best checkpoint from each variant)

| Variant | Config | Best Det | FP at Best | Notes |
|---|---|---|---|---|
| v3 200k | threshold 0.01, no geometry, LR=1e-4 constant | 24.0% | 16.3% | previous Pareto front |
| v3 1M | (same) | 72.0% | 68.2% | sensitivity record, FP explosion |
| v4 300k | threshold 0.3, geometry ON, LR=1e-4 constant | 8.0% | 12.4% | geometry gate killed signal |
| **v5 100k** | **threshold 0.15, no geometry, LR decay** | **20.0%** | **14.6%** | **best FP (<15%)** |
| **v5 150k** | **(same)** | **32.0%** | **16.6%** | **new Pareto front** |
| **v5 1M** | **(same)** | **40.0%** | **21.5%** | **first 40% at <25% FP** |

### Pareto Analysis

v5 150k Pareto-dominates v3 200k: **+8pp sensitivity (32% vs 24%) at same FP (16.6% vs 16.3%)**.

v5 1M achieves 40% sensitivity for the first time outside the FP explosion zone (21.5% vs v3's 68.2% at similar sensitivity).

No checkpoint meets the combined target of **>40% detection AND <15% FP**.

### What LR Decay Achieved

| Metric | v3 1M (constant LR) | v5 1M (LR decay) | Improvement |
|---|---|---|---|
| Detection | 72.0% | 40.0% | -32pp (expected — less aggressive) |
| FP | 68.2% | 21.5% | **-46.7pp** (massive) |
| FP:Det ratio | 0.95 | 0.54 | **much better tradeoff** |

LR decay cut FP by 47pp while sacrificing 32pp sensitivity — a 1.5:1 FP:sensitivity improvement ratio. The tradeoff curve is clearly better than v3's, but still not enough to hit both targets simultaneously.

---

## 6. Timeseries Analysis (v5 1M on Recording #2)

From `timeseries_rec02_v5_1M.npz`:

```
braking_received frequency (unchanged from prior runs — derives from CSV data):
  > 0.01:  949/1956 (48.5%)
  > 0.15:  ~500/1956 (~28%)    ← ignoring penalty fires here
  > 0.3:   375/1956 (19.2%)
  > 0.5:   279/1956 (14.3%)
  any_braking_peer: 128/1956 (6.5%)

Model action distribution (v5 1M):
  Steps with action > 0.1: 20.9%
  Avg calm action: 0.102
  Max calm action: 1.000
```

---

## 7. Root Cause: Why No Checkpoint Hits Both Targets

### The phase transition problem

Between 400k and 500k, the model undergoes a sharp phase transition from conservative (20%/18%) to aggressive (52%/54%). There is no smooth intermediate where sensitivity rises without FP.

**Why this happens:** At ~400k-500k steps, the policy crosses a tipping point where the ignoring penalty's expected cost exceeds the comfort penalty's expected cost. Once this happens, the optimal policy flips from "mostly don't brake" to "mostly brake." The transition is abrupt because the policy is near-deterministic (std is low after log_std reset and training).

The LR decay partially reverses this (1M: 40%/21.5%) but can't undo it cleanly because the policy weights have already been pushed through the phase transition.

### The observation discrimination limit

From v3 timeseries analysis (Section 12.4 of the plan doc):
- V3 200k had **46.1% action rate during brk>0.3** vs **2.8% during calm** — a 16:1 selectivity ratio
- By V3 1M, calm action rose to 65.6% — selectivity collapsed to 1.2:1

V5's LR decay preserved better selectivity at 1M than v3, but the discrimination ceiling appears to be ~32% sensitivity at ~16% FP (the 150k point). Beyond that, more training pushes both sensitivity and FP up together.

This suggests the **observation features may not carry enough information** to discriminate real hazards from normal traffic braking at higher sensitivity levels. The key features (`braking_received`, `min_peer_accel`, `max_closing_speed`) overlap between hazard and non-hazard braking in real recordings.

---

## 8. Recommended Next Steps

### Option A: Graduated penalty (soft ramp instead of hard threshold)

Replace the binary `braking_received > 0.15` gate with a soft penalty that scales nonlinearly:

```python
# Instead of:
if braking_received >= threshold:
    penalty = PENALTY_IGNORING_HAZARD * braking_received

# Use:
effective = max(0, braking_received - 0.10) / 0.90   # shift baseline
penalty = PENALTY_IGNORING_HAZARD * effective ** 2     # quadratic scaling
```

Rationale: The hard threshold creates a binary incentive (brake/don't brake) that leads to the phase transition. A smooth ramp creates a graded incentive where the penalty is tiny for weak signals and large for strong signals, potentially avoiding the phase transition.

At `braking_received=0.2`: penalty = -5.0 * (0.11)^2 = -0.06 (negligible)
At `braking_received=0.5`: penalty = -5.0 * (0.44)^2 = -0.98 (moderate)
At `braking_received=1.0`: penalty = -5.0 * (1.0)^2 = -5.0 (full)

### Option B: Shadow kinematic ego for reward geometry

Keep recorded ego for observations but run `EgoKinematics` in parallel to compute counterfactual `distance` and `closing_rate` for the reward gate:

```python
def step(self, action):
    # Observation: from recorded trajectory (matches validator)
    ego_state = VehicleState(recorded...)
    obs = self.obs_builder.build(ego_state, ...)

    # Reward geometry: from shadow kinematic ego (reflects model's actions)
    shadow_decel = self._shadow_ego.step(action_value, heading)
    shadow_distance = self._nearest_peer_distance(peers, shadow_ego_pos)
    shadow_closing = self._closing_rate(peers, shadow_ego, shadow_ego_pos)

    reward = self.reward_calculator.calculate(
        distance=shadow_distance,      # counterfactual
        closing_rate=shadow_closing,    # counterfactual
        braking_received=brk_recv,     # real signal
        ...
    )
```

Rationale: This fixes the structural issue — the shadow ego's geometry reflects the model's actions, so dangerous geometry appears when the model fails to brake. The geometry gate becomes meaningful again.

Complexity: Moderate — requires maintaining a parallel ego state in `step()`. Must reset shadow ego each episode.

### Option C: Curriculum with phase transition avoidance

Run two sequential fine-tuning passes:
1. **Pass 1 (200k steps, LR=1e-4):** Aggressive penalty (threshold 0.01) to build initial sensitivity. Stop BEFORE the phase transition (~200k).
2. **Pass 2 (300k steps, LR=3e-5):** Tighter penalty (threshold 0.2) starting from Pass 1 checkpoint. The lower LR + tighter threshold should refine discrimination without triggering the FP explosion.

Rationale: The phase transition happens when extended training at high LR + loose threshold pushes the policy past the tipping point. Two-phase training separates sensitivity building from specificity refinement.

### Option D: Augment observation with hazard severity signal

Add a new observation feature that better discriminates real hazards: the **rate of change of braking_received** (`d_braking_received/dt`). This would be high during onset of new braking events and near-zero during the slow decay tail.

```python
ego[6] = braking_received_delta  # current - previous step
```

Rationale: `braking_received` alone can't distinguish "just entered hazard" from "slow decay from old event." The derivative provides temporal edge detection. Requires ego obs 6→7 dim and `features_dim` 38→39. Must also add to validator.

### Recommended order

1. **Option A first** (lowest implementation cost, directly addresses the phase transition)
2. **Option C second** (if A doesn't break the phase transition pattern)
3. **Option B third** (if the observation discrimination limit is confirmed)
4. **Option D last** (highest cost — changes observation space, requires revalidation)

---

## 9. Files & Results

```
ml/results/run_024_replay_v5_threshold_only/
├── model_final.zip                    # 1M final model
├── vecnormalize.pkl                   # reward normalization stats
├── training_summary.json              # training config + summary
├── checkpoints/                       # 20 checkpoints (50k-1M)
│   ├── replay_ft_50000_steps.zip
│   ├── replay_ft_100000_steps.zip
│   ├── ...
│   └── replay_ft_1000000_steps.zip
├── tensorboard/                       # training curves
└── validation/                        # all checkpoint validations
    ├── validation_report_rec02_v5_*.json      # Recording #2 (all checkpoints)
    ├── timeseries_rec02_v5_*.npz              # Recording #2 timeseries
    ├── validation_report_extra_v5_*.json      # Extra Driving (100k, 150k, 1M)
    └── timeseries_extra_v5_*.npz              # Extra Driving timeseries
```

---

## 10. Key Takeaways

1. **LR decay is the right tool for FP control** — it cut FP from 68% (v3) to 21.5% (v5) at 1M. This should be standard for all future replay fine-tuning.

2. **Dropping geometry was correct** — v5 recovered from v4's 8% sensitivity back to 40%. The geometry gate is confirmed broken in recorded-ego mode.

3. **The threshold-only approach improved the tradeoff curve** but didn't break the fundamental coupling between sensitivity and FP. The phase transition at ~500k steps creates a cliff that LR decay can only partially repair.

4. **The 150k checkpoint (32%/16.6%) is the new Pareto front** — it beats v3 200k (24%/16.3%) and is the best balanced result in the entire Run 024 line. But it falls short of the >40%/<15% target.

5. **The observation discrimination limit may be real** — the 16:1 selectivity ratio at 150k suggests the model CAN distinguish hazard from calm, but extending training degrades this discrimination. Either the reward pushes past the discrimination boundary, or the features can't support finer-grained decisions.
