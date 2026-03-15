# Run 020 Postmortem

**Date:** March 15, 2026
**Model:** `cloud_prod_020` (`roadsense-v2v/ml/models/runs/cloud_prod_020/model_final.zip`)
**Training length:** 2,000,000 PPO timesteps
**Plan doc:** `docs/10_PLANS_ACTIVE/RUN_020_DEPLOYMENT_COMPATIBLE_OBSERVATION.md`
**Primary goal:** Remove deployment-incompatible observation features (`progress`, sticky `braking_received`) without regressing critic health or V2V reaction.
**Final verdict:** **Partial success in SUMO, failure on the actual deployment-transfer objective.** Run 020 is **not promotable** to a 10M production run and **not deployable** as-is.

---

## 1. Executive Summary

Run 020 proved that the new deployment-compatible observation design can train stably in SUMO:
- critic health stayed strong throughout the 2M diagnostic
- the final eval matrix achieved 0 collisions
- the final eval matrix achieved 100% reaction across peer counts `n=1..5`

However, the real-data replay check failed decisively:
- **Recording #2:** `12.0%` sensitivity, `6.07%` false positives
- **Extra Driving:** `17.4%` sensitivity, `7.72%` false positives

This is the opposite side of the Run 019 failure:
- Run 019 with the sticky latch was far too reactive on real data
- Run 020 removed that catastrophic false-positive behavior
- but the retrained policy became far too conservative and now misses most real braking events

The net result is:
- **the observation fix solved the permanent-latch mismatch**
- **but it did not solve sim-to-real transfer**
- **promotion to 10M is blocked**

---

## 2. What Run 020 Changed

Per the plan, Run 020 made two observation-space changes:

1. Removed `progress` from the ego observation.
2. Replaced sticky binary `braking_received` with a decaying scalar:

```python
BRAKING_DECAY = 0.95

if any(peer.accel <= -2.5 for peer in cone_filtered_peers):
    braking_received = 1.0
else:
    braking_received *= 0.95
```

The design intent was sound:
- same semantics in training and deployment
- bounded temporal memory for a feedforward policy
- no permanent latch
- no episode-timer feature
- reward shaping still aligned with the observation

Run 020 successfully answered one narrow question:
- **Can this observation design still train a healthy SUMO policy?**
- Answer: **yes**

It failed the more important question:
- **Does this observation design transfer better to real data?**
- Answer: **no**

---

## 3. Artifacts Reviewed

### Training and SUMO artifacts

- `roadsense-v2v/ml/models/runs/cloud_prod_020/metrics.json`
- `roadsense-v2v/ml/models/runs/cloud_prod_020/training-run.log`
- `roadsense-v2v/ml/models/runs/cloud_prod_020/model_final.zip`
- `roadsense-v2v/ml/models/runs/cloud_prod_020/smoke_014_v6_100k/v2v_reaction_check.json`

### Real-data replay artifacts generated for this postmortem

- `roadsense-v2v/ml/data/sim_to_real_validation_run020/validation_report_recording_02_decay.json`
- `roadsense-v2v/ml/data/sim_to_real_validation_run020/timeseries_recording_02_decay.npz`
- `roadsense-v2v/ml/data/sim_to_real_validation_run020/validation_report_extra_driving_decay.json`
- `roadsense-v2v/ml/data/sim_to_real_validation_run020/timeseries_extra_driving_decay.npz`

### Prior baseline artifacts used for comparison

- `roadsense-v2v/ml/data/sim_to_real_validation_run019/validation_report_recording_02_latched_p04.json`
- `roadsense-v2v/ml/data/sim_to_real_validation_run019/validation_report_recording_02_off_p0.json`
- `roadsense-v2v/ml/data/sim_to_real_validation_run019/validation_report_extra_driving.json`
- `roadsense-v2v/ml/data/sim_to_real_validation_run019/validation_report_extra_driving_off_p0.json`

---

## 4. What Passed

### 4.1 The 2M diagnostic completed successfully

The training run completed normally and saved all expected artifacts.

Key facts:
- `training_timesteps = 2000000`
- final model, checkpoints, metrics, and VecNormalize state all exist
- no sign of critic collapse or early abort

### 4.2 Kill criterion for critic health passed cleanly

The plan required:
- `explained_variance > 0.1` by 500k steps

Actual result:
- near 500k steps (`499,712`), `explained_variance = 0.961`
- near the end (`1,998,848`), `explained_variance = 0.925`
- minimum `explained_variance` after 500k was still `0.798`

Conclusion:
- **critic health was not the problem in Run 020**

### 4.3 Final SUMO evaluation passed

Final evaluation from `metrics.json`:

| Metric | Result |
|--------|--------|
| Eval episodes | 276 |
| Collision rate | 0.0% |
| Success rate | 100.0% |
| Behavioral success rate | 98.55% |
| Avg pct time unsafe | 1.65% |
| Hazard injection | ON |
| Eval peer counts | `1,2,3,4,5` |

### 4.4 Reaction generalized across all peer counts

Per-peer-count summary:

| Peer Count | Episodes | Reaction Rate | Collision Rate | Behavioral Success | Avg Pct Time Unsafe |
|------------|----------|---------------|----------------|--------------------|---------------------|
| `n=1` | 56 | 100.0% | 0.0% | 100.0% | 0.41% |
| `n=2` | 56 | 100.0% | 0.0% | 94.64% | 1.07% |
| `n=3` | 56 | 100.0% | 0.0% | 100.0% | 1.89% |
| `n=4` | 56 | 100.0% | 0.0% | 100.0% | 1.45% |
| `n=5` | 52 | 100.0% | 0.0% | 98.08% | 3.54% |

This means the Deep Sets policy still handled variable `n` correctly under the new observation design.

### 4.5 Source-rank reaction scaling still looked sensible

Selected examples from `source_reaction_summary`:
- `n=1`, rank 1: 100% reaction, average reaction time `0.659s`
- `n=5`, rank 5: 100% reaction, average reaction time `1.8s`

The model remained capable of reacting to farther hazards through the relay chain.

---

## 5. Residual SUMO Weaknesses

Run 020 was not perfect even inside SUMO. Three scenarios had behavioral success below 100%:

| Scenario | Behavioral Success | Avg Pct Time Unsafe | Avg Reward |
|----------|--------------------|---------------------|------------|
| `eval_n2_001` | 71.43% | 3.60% | `-267.60` |
| `eval_n2_006` | 85.71% | 3.89% | `-237.97` |
| `eval_n5_003` | 85.71% | 10.63% | `816.06` |

Interpretation:
- these are not collision failures
- they are clearance or late-braking quality issues inside otherwise successful episodes
- they are secondary compared to the replay failure, but they show Run 020 was not a flawless SUMO policy either

---

## 6. Early Smoke Result: Misleading but Worth Recording

The bundled 100k smoke artifact at:
- `roadsense-v2v/ml/models/runs/cloud_prod_020/smoke_014_v6_100k/v2v_reaction_check.json`

showed:
- 40 hazard episodes
- 40 hazard messages received
- 40 braking signals received
- **0 reaction episodes**

That sounds catastrophic, but there are two reasons not to over-interpret it:

1. The final 2M model fully recovered to 100% reaction in the main eval matrix.
2. The smoke run used `dataset_v6_formation_fix`, while the actual cloud run used `dataset_v10_run020_post_cone`.

Takeaway:
- the 100k smoke artifact is useful historical context
- but it is not a clean apples-to-apples predictor of the final 2M run

---

## 7. What Failed

### 7.1 The actual deployment objective failed

The Run 020 plan set these replay targets:

| Recording | Target Sensitivity | Target False Positives |
|-----------|--------------------|------------------------|
| Recording #2 | `> 60%` | `< 15%` |
| Extra Driving | `> 75%` | `< 20%` |

Actual Run 020 replay results:

| Recording | Sensitivity | False Positive Rate | Verdict |
|-----------|-------------|---------------------|---------|
| Recording #2 | `12.0%` (`3/25`) | `6.07%` | FAIL |
| Extra Driving | `17.4%` (`4/23`) | `7.72%` | FAIL |

This is not a near miss. Sensitivity collapsed by a factor of roughly `5x` relative to the target on both recordings.

### 7.2 False positives improved, but only because the model mostly stopped braking

Run 020 versus Run 019 replay baselines:

| Recording | Mode | Sensitivity | False Positive Rate |
|-----------|------|-------------|---------------------|
| Recording #2 | Run 019 `latched`, `progress=0.4` | 100.0% | 85.3% |
| Recording #2 | Run 019 `off`, `progress=0.0` | 44.0% | 6.3% |
| Recording #2 | Run 020 `decay` | 12.0% | 6.07% |
| Extra Driving | Run 019 `latched`, `progress=0.4` | 91.3% | 92.0% |
| Extra Driving | Run 019 `off`, `progress=0.0` | 78.3% | 21.8% |
| Extra Driving | Run 020 `decay` | 17.4% | 7.72% |

Interpretation:
- Run 020 clearly fixed the catastrophic false-positive blow-up from the sticky latch.
- But it did **not** land on a good tradeoff.
- On both recordings, Run 020 sensitivity is dramatically worse than even the old Run 019 `off` ablation.

This is the core postmortem finding:
- **Run 020 did not just "trim false positives."**
- **It shifted the policy into severe underreaction.**

### 7.3 The misses were not limited to no-peer windows

It would be easy to dismiss some replay misses as "no peers were visible, so the model had nothing to work with." That is only a small part of the story.

#### Recording #2

- total braking events: `25`
- events with at least one peer visible: `24`
- detected among those peer-visible events: `3`
- peer-visible sensitivity: **12.5%**

Missed peer-visible events with real braking at or below `-2.5 m/s^2`:
- `-2.53` with `1.625` peers visible on average, max model action `0.0`
- `-3.62` with `2.0` peers visible, max model action `0.025`
- `-3.92` with `2.0` peers visible, max model action `0.0`
- `-3.11` with `2.0` peers visible, max model action `0.0`
- `-2.58` with `2.0` peers visible, max model action `0.0`

#### Extra Driving

- total braking events: `23`
- events with at least one peer visible: `21`
- detected among those peer-visible events: `4`
- peer-visible sensitivity: **19.0%**

Missed peer-visible events with real braking at or below `-2.5 m/s^2`:
- `-3.52` with `2.0` peers visible, max model action `0.0`
- `-5.71` with `1.5` peers visible, max model action `0.095`
- `-2.66` with `2.0` peers visible, max model action `0.0`

This matters because it rules out the lazy explanation that replay sensitivity collapsed only because the model frequently had no observations.

### 7.4 The policy output distribution confirms underreaction

Run 020 replay action distribution:

| Recording | Mean Action | Median | Pct Steps Above 0.1 |
|-----------|-------------|--------|---------------------|
| Recording #2 | `0.0141` | `0.0` | `5.9%` |
| Extra Driving | `0.0250` | `0.0` | `7.1%` |

This is exactly what the event-level misses suggest:
- the model is mostly outputting zero or near-zero deceleration
- when it does react, it does so rarely

---

## 8. Plan-vs-Run Mismatches

### 8.1 The cloud run did not preserve the planned dataset

The Run 020 plan says to preserve:
- `dataset_v6 (formation-fixed scenarios)`

But the actual cloud training script used:
- `ml/scenarios/datasets/dataset_v10_run020_post_cone`

This is an experimental caveat, but it should be described precisely:

- both dataset IDs are intended to be generated from the same `base_real`
  formation-fixed source scenario family
- this is **not** evidence that Run 020 switched to a different scenario family
- the real issue is that the exact `dataset_v10_run020_post_cone` artifact is
  not present locally, so byte-for-byte identity with `dataset_v6_formation_fix`
  cannot be confirmed after the fact

Implication:
- Run 020 was **not** a pure dataset-ID-locked A/B against the prior setup
- attribution is still somewhat confounded by regeneration / identity ambiguity
- but the replay failure should not be blamed on a different scenario family

### 8.2 The 100k smoke run and the 2M main run used different datasets

The bundled smoke artifact used:
- `dataset_v6_formation_fix`

The main cloud run used:
- `dataset_v10_run020_post_cone`

Implication:
- the smoke result and the final 2M result are not directly comparable
- future runs should avoid mixing smoke datasets and production datasets under the same run narrative

### 8.3 Real-data replay was not in the original artifact bundle

When the `cloud_prod_020` folder was first inspected, it contained:
- training logs
- metrics
- checkpoints
- smoke artifact

It did **not** contain the planned replay-validation output directory:
- `ml/data/sim_to_real_validation_run020`

This postmortem adds that missing gate.

Process lesson:
- a run cannot be declared successful on a deployment objective until the replay reports exist

---

## 9. Interpretation

### 9.1 What Run 020 proved

Run 020 proved that:
- deployment-compatible observation semantics can coexist with a healthy critic
- deployment-compatible observation semantics can still produce a strong SUMO policy
- the sticky latch really was responsible for most of the catastrophic real-data false positives

### 9.2 What Run 020 disproved

Run 020 disproved the working assumption that:
- fixing the latch and removing progress would be enough to recover deployable real-data behavior

The model now behaves like this:
- low false positives
- very low braking frequency
- severe miss rate on real braking events

That means the transfer problem is deeper than just the permanent latch.

### 9.3 Most likely reading of the failure

The evidence supports this interpretation:

1. The new decay feature is trainable in SUMO.
2. But the resulting policy learned a decision boundary that is too conservative for real replay.
3. The policy appears to require a stronger or more sustained hazard signal than real data usually provides.
4. This failure persists even when peers are visible and even on harder braking events.

This is an inference from the replay artifacts, not a directly measured hidden-state explanation, but it fits the observed behavior much better than "the latch fix just needed a little tuning."

---

## 10. Decision

### Final disposition of Run 020

- **Do not promote Run 020 to a 10M production run.**
- **Do not use Run 020 for TFLite export or ESP32 deployment.**
- **Do not describe Run 020 as deployment-compatible in any final sense.**

### What Run 020 should be remembered as

Run 020 is a useful negative result:
- it removed the worst replay mismatch
- it preserved critic health
- it preserved SUMO reaction performance
- but it showed that the current training setup can still produce a policy that underreacts badly on real data

---

## 11. Recommended Next Steps

### 11.1 Do not optimize further for false positives first

Run 020 is already low-FP:
- `6.07%` on Recording #2
- `7.72%` on Extra Driving

The primary failure is now **sensitivity**, not specificity.

That means the next run should **not** start with:
- lowering `BRAKING_DECAY` to `0.90`
- further suppressing post-event caution

Those changes would likely make sensitivity even worse.

### 11.2 Recover temporal hazard salience without reintroducing a permanent latch

The approved Run 021 follow-up is now more concrete than the original generic
candidate list. The plan is to keep Run 020's deployment-compatible observation
and critic-stability fixes, but change hazard generation in two orthogonal ways.

#### Fix 1: Randomize desired hazard deceleration, not target speed

Change `hazard_injector.py` so hazards cover a broader braking-intensity range:

```python
HAZARD_DECEL_MIN = 3.0
HAZARD_DECEL_MAX = 10.0

desired_decel = rng.uniform(HAZARD_DECEL_MIN, HAZARD_DECEL_MAX)
braking_duration = rng.uniform(BRAKING_DURATION_MIN, BRAKING_DURATION_MAX)

current_speed = target_state.speed
target_speed = max(0.0, current_speed - desired_decel * braking_duration)
sumo.slow_down(target, target_speed, braking_duration)
self._hazard_target_speed = target_speed
```

Why this was chosen:
- it directly controls the training decel distribution
- it avoids the "random target speed depends on current speed" coupling
- it broadens training beyond the current near-`-10 m/s²` hazard extreme

#### Fix 2: Add resolved-hazard episodes

Change `hazard_injector.py` so some hazards clear after the braking pulse:

```python
HAZARD_RESOLVE_PROB = 0.4
RESOLVE_DELAY_MIN = 20
RESOLVE_DELAY_MAX = 50
```

Behavior:
- 60% of hazards remain persistent
- 40% of hazards resolve after a 2-5s delay

`maintain_hazard()` should then:
- pin persistent hazards at `self._hazard_target_speed`
- release resolved hazards back to CF with `setSpeed(-1)` once `step >= resolve_step`

Why this was chosen:
- it teaches the model to react to transient braking, not only persistent stops
- it teaches the model to release braking when the scene normalizes
- it preserves the existing decay signal, which naturally fades after the event

#### Scope of the approved Run 021 change set

The intended Run 021 implementation is deliberately narrow:

| File | Change |
|------|--------|
| `ml/envs/hazard_injector.py` | Randomize desired decel in `[3.0, 10.0] m/s²`, compute `target_speed`, store `_hazard_target_speed`, add resolve logic |
| `ml/envs/hazard_injector.py` (`maintain_hazard`) | Pin at `_hazard_target_speed` instead of always `0.0`; release to CF for resolved hazards |

That is the whole approved change set for the next diagnostic.

### 11.3 What Run 021 Should Not Change

The following pieces remain load-bearing and are intentionally unchanged:

- `BRAKING_DECAY = 0.95`
- `BRAKING_ACCEL_THRESHOLD = -2.5 m/s²`
- decay-scaled reward structure
- PPO hyperparameters
- Deep Sets architecture
- episode length / hazard window

For dataset control, the next diagnostic should use the preserved
`dataset_v6_formation_fix` dataset ID as the explicit comparison baseline, even
though later datasets share the same `base_real` lineage.

### 11.4 Treat real-data replay as a first-class acceptance test

For future runs, the acceptance sequence should be:

1. 2M cloud diagnostic
2. SUMO eval matrix check
3. real-data replay validation
4. only then consider 10M promotion

This is the main process lesson from Run 020.

#### Approved Run 021 acceptance gates

| Gate | Target | Kill / Fail Condition |
|------|--------|-----------------------|
| Critic health @ 500k | `explained_variance > 0.1` | Kill if `0` at 500k |
| SUMO reaction @ 2M | `> 90%` | Kill if `< 50%` |
| Recording #2 replay | sensitivity `> 60%`, FP `< 15%` | Fail if sensitivity `< 40%` |
| Extra Driving replay | sensitivity `> 75%`, FP `< 20%` | Fail if sensitivity `< 50%` |

The replay gate remains mandatory before any 10M promotion.

---

## 12. Commands Used For The Missing Replay Gate

These commands were run locally in the project venv to produce the Run 020 replay reports:

```bash
cd /home/amirkhalifa/RoadSense2/roadsense-v2v
source ml/venv/bin/activate

python -m ml.scripts.validate_against_real_data \
    --tx_csv /home/amirkhalifa/RoadSense2/Convoy_recording_02282026/V001_tx_004.csv \
    --rx_csv /home/amirkhalifa/RoadSense2/Convoy_recording_02282026/V001_rx_004.csv \
    --model_path ml/models/runs/cloud_prod_020/model_final.zip \
    --output_dir ml/data/sim_to_real_validation_run020 \
    --forward_axis y \
    --recording_name recording_02_decay

python -m ml.scripts.validate_against_real_data \
    --tx_csv /home/amirkhalifa/RoadSense2/Convoy_extra_data_02282026/V001_tx_005.csv \
    --rx_csv /home/amirkhalifa/RoadSense2/Convoy_extra_data_02282026/V001_rx_005.csv \
    --model_path ml/models/runs/cloud_prod_020/model_final.zip \
    --output_dir ml/data/sim_to_real_validation_run020 \
    --forward_axis y \
    --recording_name extra_driving_decay
```

The script defaulted to:
- `braking_received_mode = decay`

No extra mode flags were needed.

---

## 13. Bottom Line

Run 020 answered the wrong question successfully.

It successfully showed:
- "Can the decay observation train a strong SUMO policy?"

It failed the question that actually matters:
- "Did this make the model deploy better on real data?"

The honest summary is:
- **SUMO success**
- **replay failure**
- **deployment blocked**
