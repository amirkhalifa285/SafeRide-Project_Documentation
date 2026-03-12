# Run 014 Preparation Checklist

**Date:** March 12, 2026
**Goal:** Latch fix (Run 013 root cause) + formation-fixed base_real + `HAZARD_PROBABILITY=1.0`. First run where the ignoring-hazard penalty covers ~99% of post-hazard steps instead of 8-16%, and every training episode contains a hazard.

---

## What changed since Run 013

| Change | File | Effect |
|--------|------|--------|
| Latch `_hazard_source_braking_latched` | `convoy_env.py` | Reward signal persists for entire hazard event, not just 2-4s slowDown |
| base_real: sigma=0.0 | `vehicles.rou.xml` | No random braking/accel perturbations |
| base_real: speedFactor=1.0 speedDev=0 | `vehicles.rou.xml` | All vehicles cruise at same desired speed — formation holds |
| base_real: 25m departPos spacing | `vehicles.rou.xml` | Above SUMO safe-insertion threshold (9.6m) |
| Integration tests on base_real | `conftest.py`, `test_run006_fixes.py` | Docker tests now validate actual training scenario |
| `HAZARD_PROBABILITY = 1.0` | `hazard_injector.py` | Doubles hazard exposure vs Run 013 (every training episode has a hazard) |

**Unchanged from Run 013:** gradual hazard (slowDown 2-4s), CF override, Deep Sets, 5-dim ego obs, all PPO hyperparams (LR=1e-4, n_steps=4096, ent_coef=0.0, log_std_init=-0.5).

---

## Pre-flight checklist

### Step 1: Set HAZARD_PROBABILITY = 1.0

**Decision:** DONE (March 11, 2026).

- At 0.5, only 50% of episodes had hazards. With the latch fix, that still wastes half of training on non-hazard episodes.
- At 1.0, every episode has a hazard. The model already knows how to cruise; Run 014 should spend its sample budget on hazard-response learning.
- **Interpretation after local smoke checks:** `100k` timesteps with `HAZARD_PROBABILITY=1.0` still showed `0%` RL reaction, but that is not a contradiction. `100k` timesteps is only a few hundred episodes and still early-training territory, not evidence that 1.0 is wrong.

```python
# ml/envs/hazard_injector.py line 15
HAZARD_PROBABILITY = 1.0
```

### Step 2: Push all changes to master

The EC2 script does `git pull origin master` before regenerating datasets. These files MUST be on master:

- [ ] `ml/scenarios/base_real/vehicles.rou.xml` (sigma=0, speedDev=0, 25m spacing)
- [ ] `ml/scenarios/base_real/recording_metadata.json` (updated notes)
- [ ] `ml/envs/convoy_env.py` (latch fix)
- [ ] `ml/tests/integration/conftest.py` (base_real fixtures)
- [ ] `ml/tests/integration/test_run006_fixes.py` (base_real fixture + latch test)
- [ ] `ml/envs/hazard_injector.py` (`HAZARD_PROBABILITY = 1.0`)
- [ ] `ml/scripts/check_v2v_reaction.py` (local/EC2 checkpoint reaction verdict helper)

### Step 3: Update cloud training script

```bash
# ml/scripts/cloud/run_training.sh — changes needed:
RUN_ID="cloud_prod_014"
# Update header comments to describe Run 014
# Dataset dir stays dataset_v8 (regenerated from fixed base_real)
# Everything else stays the same
```

### Step 4: Docker integration tests (MANDATORY)

```bash
cd /home/amirkhalifa/RoadSense2/roadsense-v2v/ml
./run_docker.sh integration
```

**Already done (March 11) — ALL PASSING.** Re-run only if you change HAZARD_PROBABILITY or any other code.

### Step 5: Docker smoke checks (MANDATORY before EC2)

#### 5A. Pipeline smoke train (completed)

Short training to verify the full pipeline works end-to-end with the latch fix and `HAZARD_PROBABILITY=1.0`:

```bash
cd /home/amirkhalifa/RoadSense2/roadsense-v2v/ml
./run_docker.sh train \
    --total_timesteps 5000 \
    --run_id smoke_014 \
    --output_dir /work/results
```

**Result (completed):**
- Training started and completed without errors
- Model file saved successfully
- This run was a pipeline sanity check only
- Important: this command falls back to the single default scenario (`base`), so it is **NOT** a meaningful V2V behavior check

#### 5B. Dataset-based local smoke run on `dataset_v6_formation_fix` (completed)

Run used:

```bash
cd /home/amirkhalifa/RoadSense2/roadsense-v2v/ml
./run_docker.sh train \
    --dataset_dir ml/scenarios/datasets/dataset_v6_formation_fix \
    --emulator_params ml/espnow_emulator/emulator_params_measured.json \
    --total_timesteps 100000 \
    --run_id smoke_014_v6_100k \
    --output_dir /work/results/smoke_014_v6_100k
```

Follow-up reaction check used:

```bash
cd /home/amirkhalifa/RoadSense2/roadsense-v2v/ml
./run_docker.sh python3 -m ml.scripts.check_v2v_reaction \
    --model_path /work/results/smoke_014_v6_100k/model_final.zip \
    --dataset_dir /work/ml/scenarios/datasets/dataset_v6_formation_fix \
    --emulator_params /work/ml/espnow_emulator/emulator_params_measured.json \
    --output /work/results/smoke_014_v6_100k/v2v_reaction_check.json
```

**Result (March 12, 2026):**
- 40/40 hazards injected
- 40/40 hazard-source messages received by ego
- 40/40 braking signals received by ego
- **0/40 RL reactions after receipt (0.0%)**
- 0/40 collisions
- All peer counts covered sequentially: n=1,2,3,4,5 each had 0/8 reactions

**Interpretation:**
- This is **not** evidence of a broken signal path. The full hazard/V2V path works end-to-end.
- This is also **not** evidence that `HAZARD_PROBABILITY=1.0` is wrong.
- It means `100k` timesteps is still too early to expect nonzero learned V2V braking.
- Therefore this smoke run does **not** block EC2 launch.

**Local evaluation note:**
- The built-in deterministic eval matrix path in `train_convoy.py` / `evaluate_model.py` hit a scenario-plan mismatch during local smoke eval.
- For local checkpoint checks, use `check_v2v_reaction.py` with sequential forced-hazard eval instead of deterministic matrix mode.

### Step 6: Launch on EC2

```bash
# Standard launch — see EC2 Training Procedure in PROJECT_STATUS_OVERVIEW.md
# Use the updated run_training.sh with RUN_ID=cloud_prod_014
```

**Decision:** GO.

- Local smoke evidence is sufficient to show the pipeline is healthy.
- `100k`-timestep zero-reaction result is too early to invalidate the run.
- Proceed to full EC2 training without changing any other knobs.

**Expected training time:** ~18-19h on c6i.xlarge (10M steps).

---

## Pass criteria (after training completes)

| Metric | Target | Rationale |
|--------|--------|-----------|
| V2V reaction rate | >90% (target 100%) | Run 011 achieved 100%; latch fix should restore this |
| avg_reward | > -200 | Run 011 was -86.62; Run 013 was -1351 |
| collision_rate | 0% | Must maintain |
| behavioral_success | >80% | Run 011 was 84.1% |
| Training convergence | Stable plateau by 5-7M steps | Run 013 oscillated; latch should fix |
| H5 sim-to-real | action > 0.1 within 3s of peer braking | The whole point of gradual hazard |
| FP rate (Extra Recording) | < 5% | Safety check |

---

## Risk assessment

| Risk | Likelihood | Mitigation |
|------|-----------|------------|
| Latch makes penalty too strong, model over-brakes | Low | PENALTY_IGNORING_HAZARD is -5.0/step, same as cruise reward +4.0. Magnitudes comparable. |
| Formation still broken in augmented scenarios | None | EC2 regenerates from fixed base_real. vType attributes copied as-is. |
| EC2 doesn't get latest code | Low | Verify `git pull` output in training log shows the latch commit. |
| HAZARD_PROBABILITY=1.0 hurts calm driving | Low | Model learned calm driving by Run 004. Reward ramp teaches following regardless. |
| Local `100k` smoke run showed 0% reaction | Medium | Treat as "too early to judge," not as a blocker. Full run is 10M timesteps, not 100k. |
| Local deterministic eval matrix mismatch during smoke eval | Medium | Use `ml/scripts/check_v2v_reaction.py` for checkpoint reaction checks instead of deterministic matrix mode. |

---

## Quick reference: what NOT to change

- Hyperparams (LR, n_steps, ent_coef, log_std_init) — proven in Run 011
- CF override — it's what enabled V2V reaction
- Gradual hazard (slowDown 2-4s) — required for sim-to-real transfer
- Deep Sets architecture — validated n=1-5
- Reward constants (PENALTY_IGNORING_HAZARD=-5.0, REWARD_EARLY_REACTION=+2.0) — isolate latch fix first
