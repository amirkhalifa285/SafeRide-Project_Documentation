# Run 016 — braking_received Binary Feature

**Date:** March 12, 2026
**Status:** Killed at ~300k steps on March 12, 2026
**Purpose:** Test whether a binary `braking_received` observation feature would wake the dead critic from Runs 014 and 015. It did not.

---

## Required related reading

Any agent working on this failure mode should read these documents before proposing the next training run:

1. `docs/PROJECT_STATUS_OVERVIEW.md` — project-wide chronology and current run status.
2. `docs/10_PLANS_ACTIVE/H5_SIM_TO_REAL_ANALYSIS.md` — why gradual braking and early V2V timing matter.
3. `docs/10_PLANS_ACTIVE/RUN_012_ROOT_CAUSE_ANALYSIS.md` — first post-Run-011 diagnosis: hazard signal reached ego, reward ignored early V2V usage.
4. `docs/10_PLANS_ACTIVE/RUN_013_ROOT_CAUSE_ANALYSIS.md` — reward timing bug and latch motivation.
5. `docs/10_PLANS_ACTIVE/RUN_014_CHECKPOINT_HANDOFF.md` — first dead-critic postmortem after the gradual `slowDown()` hazard change.
6. `docs/10_PLANS_ACTIVE/RUN_011_ANALYSIS.md` — last healthy baseline (`explained_variance` ~0.956, 100% V2V reaction).
7. `docs/00_ARCHITECTURE/DEEP_SETS_N_ELEMENT_ARCHITECTURE.md` — observation architecture constraints and the n-element problem.

---

## Why this change is necessary

### The pattern: three consecutive dead critics

Every run since the gradual `slowDown` change has failed with 0% V2V reaction:

| Run | Hazard injection | HAZARD_PROB | Reward bug | explained_var | Reaction |
|-----|-----------------|-------------|------------|---------------|----------|
| **011** | `setSpeed(0)` | 0.5 | None | **0.956** | **100%** |
| 012 | `slowDown(0, 2-4s)` | 0.5 | Penalty dead code | ? | 0% |
| 013 | `slowDown(0, 2-4s)` | 0.5 | Flag only 8-16% | ? | 0% |
| 014 | `slowDown(0, 2-4s)` | **1.0** | None (latch works) | **0** | 0% |
| 015 | `slowDown(0, 2-4s)` | **0.5** | None (latch works) | **0** | 0% |

Run 014 hypothesis: HAZARD_PROBABILITY=1.0 starved the critic of contrastive episodes.
Run 015 disproved this: HAZARD_PROBABILITY=0.5 gave bimodal episodes (ep_rew_mean improved from -3200 to -1500) but explained_variance was still exactly 0 at 300k steps.

### The root cause: observation-return disconnect

The critic's job is to predict episode returns from the current observation. For that to work, the observation must contain information that correlates with future returns.

**With `setSpeed(0)` (Run 011):**
- Hazard fires → target instantly stops → `min_peer_accel` spikes to **-13.9** (normalized by MAX_ACCEL=10)
- This is 30x larger than any normal traffic signal — impossible to miss
- The critic trivially learns: "big accel spike = bad returns"
- explained_variance = 0.956

**With gradual `slowDown` (Runs 014-015):**
- Hazard fires → target decelerates at ~-3 to -5 m/s² over 2-4 seconds
- `min_peer_accel ≈ -0.3 to -0.5` (normalized) — overlaps with normal driving variation
- But `PENALTY_IGNORING_HAZARD = -5.0/step × ~300 steps = -1500` creates a massive return gap
- The critic sees states that look identical but have returns differing by ~2000 points
- It can't learn which states lead to which returns → explained_variance = 0

Initial Run 016 hypothesis: the problem was not the reward scale itself, but the observation lacked a feature that distinguishes hazard from non-hazard states when the peer deceleration is gradual.

### Why `min_peer_accel` alone isn't enough

At normalized values of -0.3 to -0.5, the gradual braking signal sits in the same range as:
- Normal car-following deceleration adjustments
- Formation compression effects
- Emulator sensor noise

The critic would need to learn a precise threshold (accel < -0.25 means hazard) from a very noisy gradient. With the extreme reward penalty riding on this distinction, the value function collapses to predicting the mean.

---

## What changed

### Single feature addition: `braking_received` at ego[5]

**File: `ml/envs/observation_builder.py`**

The `build()` method now accepts `braking_received: bool` and appends a 6th element to the ego vector:

```
ego[0] = speed / 30.0
ego[1] = accel / 10.0
ego[2] = heading / 180.0
ego[3] = peer_count / 8
ego[4] = min_peer_accel / 10.0    (unchanged)
ego[5] = 1.0 if braking_received else 0.0    (NEW, LATCHED)
```

**File: `ml/envs/convoy_env.py`**

The `_step_espnow()` method computes `any_braking_peer_received` from the V2V message loop (any peer with `accel_x <= BRAKING_ACCEL_THRESHOLD` which is -2.5 m/s²). When True, it sets `_braking_received_latched = True` for the rest of the episode. The **latched** value is passed to the obs builder and was intended to track the same hazard phase as the reward's latched `_hazard_source_braking_latched` flag. Run 016 later showed this was only partially true, because the observation latch uses `any_braking_peer_received` while the reward is still gated by `hazard_source_braking_received`.

The observation space Box was updated from shape (5,) to (6,), with braking_received bounded [0.0, 1.0].

**File: `ml/models/deep_set_extractor.py`**

`ego_feature_dim` default changed from 5 to 6. The features_dim automatically becomes 38 (32 embed_dim + 6 ego).

### Why `any_braking_peer_received` and not `hazard_source_braking_received`

Two options were considered:

1. `hazard_source_braking_received` — only True when the specific injected hazard source is braking AND ego received its V2V message. This is training-only; in deployment, we don't know which vehicle is the "hazard source."

2. `any_braking_peer_received` — True when ANY peer's V2V-reported acceleration is below -2.5 m/s². This works identically in training AND deployment.

We chose option 2 because:
- It maps directly to what the ESP32 firmware would compute from real V2V messages
- It creates zero sim-to-real gap for this feature
- In training (formation-fixed base_real with sigma=0, speedDev=0), normal peers never decelerate below -2.5 m/s², so this flag only activates during genuine hazard events
- In deployment, any peer braking hard is a legitimate hazard signal worth reacting to

### Why a binary feature instead of scaling the continuous signal

A binary threshold is more learnable for PPO because:
- It creates a discrete state partition the critic can immediately exploit
- No need to learn a precise decision boundary from gradient descent
- The threshold (-2.5 m/s²) is already calibrated to real braking dynamics
- The continuous `min_peer_accel` at ego[4] is still available for nuance — the binary flag at ego[5] gives the critic a clean anchor

---

## Expected impact on training

### The critic should now work

With `braking_received` in the observation, the critic can trivially partition states:
- `ego[5] = 0.0` → calm state → predict returns ~-200 to +300
- `ego[5] = 1.0` → hazard state → predict returns ~-1500 to -3000

This should produce nonzero explained_variance within the first 100-200k steps. If explained_variance is still 0 at 500k with this feature, there is a different fundamental problem.

### The actor should learn V2V reaction

With a working critic providing accurate advantage estimates, the actor can learn:
- "When ego[5]=1.0 and I brake → REWARD_EARLY_REACTION (+2.0/step) and no PENALTY_IGNORING_HAZARD"
- "When ego[5]=1.0 and I don't brake → PENALTY_IGNORING_HAZARD (-5.0/step) for the rest of the episode"

The reward swing of +7.0 per step is massive and should be very learnable with PPO.

---

## Launch checklist for Run 016

### Completed before EC2 launch

- [x] **Docker integration tests pass** — `cd roadsense-v2v && ./ml/run_docker.sh integration`
  - This validates the full SUMO pipeline with the new 6-dim observation
  - Completed locally on March 12, 2026
  - Critical: the `test_braking_peer_in_observation` test now also checks `ego[5]==1.0`

- [x] **Update training script** — `ml/scripts/cloud/run_training.sh`
  - `RUN_ID="cloud_prod_016"` is set
  - Header comments updated to describe the braking_received feature
  - `HAZARD_PROBABILITY=0.5` retained from Run 015

- [x] **Push to master** — git push so EC2 can pull the latest code
  - Completed before Run 016 launch on March 12, 2026

- [x] **Verify on EC2 after git pull** — SSH in and confirm:
  ```bash
  grep "braking_received" /home/ubuntu/work/ml/envs/observation_builder.py
  grep "ego_feature_dim" /home/ubuntu/work/ml/models/deep_set_extractor.py
  grep "HAZARD_PROBABILITY" /home/ubuntu/work/ml/envs/hazard_injector.py
  ```
  - Verified as part of EC2 launch flow before training start

### Run status

- [x] **Run 016 launched on EC2** — `cloud_prod_016`
  - Training was launched on March 12, 2026
  - S3 target: `s3://saferide-training-results/cloud_prod_016/`
- [x] **Run 016 killed at ~300k steps** — March 12, 2026
  - `explained_variance` stayed at `0` or numerical noise through the latest samples (`-2.38e-07` to `1.79e-07`)
  - This matches the same dead-critic signature seen in Runs 014 and 015
  - The 1M checkpoint was not worth waiting for because the primary run hypothesis had already failed

### Early diagnostic checkpoints during training

**At ~200k steps:**
```bash
grep "explained_variance" /var/log/training-run.log | tail -10
```
- If explained_variance > 0.01: critic is waking up. This alone is a major breakthrough.
- If explained_variance = 0: something else is wrong. Do NOT wait to 1M.

**At ~500k steps:**
```bash
grep "explained_variance" /var/log/training-run.log | tail -10
grep "ep_rew_mean" /var/log/training-run.log | tail -10
grep "std" /var/log/training-run.log | tail -5
```
- explained_variance should be > 0.1 by now
- std should be declining (not stuck at 0.5)

**At 1M steps: V2V reaction check**

Copy checkpoint locally:
```bash
scp -i ~/Downloads/saferide-key.pem \
  ubuntu@<EC2_IP>:/home/ubuntu/work/results/checkpoints/deep_sets_1000000_steps.zip \
  /home/amirkhalifa/RoadSense2/roadsense-v2v/results/cloud_prod_016_checkpoint_1M.zip
```

Run reaction check:
```bash
cd /home/amirkhalifa/RoadSense2/roadsense-v2v/ml

./run_docker.sh python3 -m ml.scripts.check_v2v_reaction \
  --model_path /work/results/cloud_prod_016_checkpoint_1M.zip \
  --dataset_dir /work/ml/scenarios/datasets/dataset_v6_formation_fix \
  --emulator_params /work/ml/espnow_emulator/emulator_params_measured.json \
  --output /work/results/cloud_prod_016_checkpoint_1M_v2v.json
```

### Interpretation matrix

| explained_var at 500k | V2V reaction at 1M | Verdict |
|-----------------------|-------------------|---------|
| > 0.3 | > 0% | Run is alive — keep training to 10M |
| > 0.3 | 0% | Critic works but actor hasn't learned yet — check at 2M |
| 0 | any | Feature didn't help — deeper architectural issue |

---

## Pass criteria for Run 016

1. **explained_variance > 0.5** by 5M steps (Run 011 reached 0.956)
2. **V2V reaction > 90%** in final eval (target 100%, Run 011 baseline)
3. **avg_reward > -200** (Run 011 was -86.62)
4. **0% collision rate**
5. **H5 re-validation**: model action > 0.1 within 3s of peer braking onset in real recordings

## Actual failure analysis at ~300k

### What the 300k checkpoint proved

The core Run 016 hypothesis was falsified.

- Expected result from this document: nonzero `explained_variance` by ~100k-200k, clearly positive by ~500k.
- Actual result at ~300k: the last 20 `explained_variance` samples were all `0` or floating-point noise near zero.
- Interpretation: adding `braking_received` did not wake the critic. This run remained in the same dead-critic regime as Runs 014 and 015.

### Why the critic likely stayed dead

The original diagnosis in this file was directionally useful but incomplete. The remaining issue is not just "missing binary observation feature." There is still value-target aliasing.

1. Hidden future hazard still splits returns while the observation looks identical.
   - `HAZARD_PROBABILITY` is 0.5 and hazard onset still happens later in the episode (`hazard_step` sampled inside 150-350).
   - Before the hazard begins, many states still look identical in the observation but have very different future returns depending on whether a hidden hazard will occur later.
   - This means `ego[5]=0.0` does **not** cleanly mean "calm episode with calm return."

2. Observation and reward still key off different event definitions.
   - Observation latch: `braking_received` is driven by `any_braking_peer_received`.
   - Reward shaping: `PENALTY_IGNORING_HAZARD` and `REWARD_EARLY_REACTION` are driven by `hazard_source_braking_received` for the injected hazard source only.
   - Those signals are correlated, but they are not the same latent variable. The doc's assumption that the new bit "matches the reward's latched flag" was too strong.

3. There is still a one-step timing mismatch on the transition where braking is first observed.
   - Reward is computed after the post-step ESP-NOW update, using newly received braking evidence.
   - The policy action for that transition was chosen from the previous observation, which did not yet contain the new `braking_received` latch.
   - This is probably secondary, but it adds more noise exactly where the large hazard penalty begins.

### Updated conclusion

Run 016 shows that the deeper problem is reward/value alignment under hidden hazard timing, not just observation feature separability. The binary feature helped make the observation cleaner, but it did not make the return function learnable enough for PPO's critic in the current setup.

### Implications for the next run

- Align reward gating with the same observable signal the policy/critic sees, or explicitly give the critic the exact reward-driving latch during training.
- Treat hidden future hazard timing as part of the problem. If a future run is meant to diagnose critic learning, reduce or remove hidden episode-type variance before changing PPO hyperparameters.
- Only investigate reward normalization, critic LR, or value-network capacity **after** observation/reward alignment is fixed. Run 016 did not isolate those yet.

---

## Files changed

| File | Change |
|------|--------|
| `ml/envs/observation_builder.py` | `braking_received` param, ego 5→6 dim |
| `ml/envs/convoy_env.py` | Observation space 5→6, passes braking signal to obs builder |
| `ml/models/deep_set_extractor.py` | `ego_feature_dim` 5→6, features_dim 37→38 |
| `ml/envs/hazard_injector.py` | `HAZARD_PROBABILITY = 0.5` (from Run 015) |
| `ml/tests/unit/test_deep_set_extractor.py` | Shape assertions 5→6, 37→38 |
| `ml/tests/unit/test_observation_builder.py` | Shape assertion 5→6 |
| `ml/tests/unit/test_convoy_env.py` | Shape assertions 5→6 |
| `ml/tests/integration/test_convoy_env_integration.py` | Shape assertions 5→6 |
| `ml/tests/integration/test_run006_fixes.py` | Added ego[5] braking_received assertion |
| `ml/tests/integration/test_scenario_switching.py` | Shape assertion 5→6 |
| `ml/scripts/validate_against_real_data.py` | Compute + pass braking_received to obs builder (was missing) |
