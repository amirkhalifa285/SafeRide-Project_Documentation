# Run 010 Eval Recovery Plan

**Date:** March 8, 2026
**Owner:** Amir + execution agent
**Priority:** CRITICAL
**Status:** ACTIVE (`E0-E2 complete`, `E3 smoke audit done — BLOCKED on formation breakup fix`)
**Target Run:** `cloud_prod_010`

---

## 1. Problem Statement

Run 010 is the first run with non-zero V2V reaction, but the current evaluation does **not** provide a trustworthy final accuracy number.

What is currently true:
- `run_010_results/metrics.json` shows **40 reactions out of 46 hazard-received episodes**.
- Collision rate is **0%**.
- Overall behavior appears materially better than Run 009.

What is currently **not** true:
- The evaluation does **not** have full deterministic coverage.
- `coverage_ok=false`.
- `12/15` `(peer_count, source_rank_ahead)` buckets are missing or partial.
- The current reaction rate is therefore **promising but biased**, not final.

### Root Cause

The current pipeline mixes three different assumptions:

1. `ml/scripts/fix_eval_peer_counts.py` enforces **peer count only**.
2. `ml/envs/hazard_injector.py` injects hazards only into peers that are **actually ahead of ego at runtime**.
3. `ml/eval_matrix.py` assumes every scenario with `n` peers can cover all ranks `1..n`.

That assumption is false for the current eval dataset. A scenario can have `n=5` total peers while still failing to present ranks `3,4,5` ahead of ego during the hazard window.

---

## 2. Immediate Objective

Produce a **trustworthy full-coverage evaluation** for the **frozen Run 010 model** so we can answer:

1. What is the real reaction rate across all `n=1..5` peer counts?
2. How does performance scale by source rank ahead?
3. Is the model strong only on nearest-peer hazards, or genuinely robust?
4. Does Run 010 justify sim-to-real validation, or is Run 011 required first?

## 2.1 Progress Update (March 8, 2026)

### Completed so far

1. **Run 010 artifacts frozen inside the repo**
   - Frozen path created:
     `roadsense-v2v/ml/models/frozen/cloud_prod_010/`
   - Copied artifacts:
     - `model_final.zip`
     - `metrics_run_010_raw.json`
     - `training-run.log`

2. **Geometry-aware audit implemented**
   - New script added:
     `ml/scripts/audit_eval_matrix_capability.py`
   - Shared eval dataset helpers added:
     `ml/eval_dataset.py`

3. **Deterministic planner upgraded to use capability metadata**
   - `ml/eval_matrix.py` now accepts per-scenario supported ranks
   - `ml/scripts/evaluate_model.py` and `ml/training/train_convoy.py`
     now consume `eval_capability_audit.json` when present
   - Planner now fails fast instead of silently scheduling impossible buckets

4. **Capability-based curation tooling added for Phase E3**
   - New script added:
     `ml/scripts/curate_eval_by_capability.py`
   - Purpose:
     select eval scenarios based on actual `(n, rank)` runtime support,
     not peer count alone

5. **Targeted unit coverage is green**
   - Relevant planner/audit/curation/eval unit slice:
     **32 tests passing**

6. **Candidate eval dataset generated and rewritten to explicit `n=1..5` coverage**
   - Raw candidate dataset created:
     `ml/scenarios/datasets/dataset_v4_eval_repair/`
   - Working copy created:
     `ml/scenarios/datasets/dataset_v4_eval_repair_fixed40/`
   - Applied `fix_eval_peer_counts.py` to the working copy with a 40-scenario target list:
     - `8x n=1`
     - `8x n=2`
     - `8x n=3`
     - `8x n=4`
     - `8x n=5`
   - Verified rewritten eval split now contains:
     `eval_n1_000..007`, `eval_n2_000..007`, `eval_n3_000..007`,
     `eval_n4_000..007`, `eval_n5_000..007`

7. **Deterministic eval tooling now supports explicit structural bucket exclusion**
   - `ml/eval_matrix.py` now accepts excluded buckets in addition to required peer counts
   - `ml/scripts/evaluate_model.py` adds:
     `--matrix_excluded_buckets`
   - `ml/training/train_convoy.py` adds:
     `--eval_matrix_excluded_buckets`
   - Excluded buckets are removed from:
     - planning
     - expected coverage counts
     - reported eval-matrix metadata

8. **New deterministic-matrix unit slice is green**
   - Updated tests for:
     - excluded bucket parsing
     - planning without excluded buckets
     - capability-aware planning with an excluded structural bucket
     - train/eval metrics carrying excluded bucket metadata
   - Targeted unit slice:
     **28 tests passing**

### Audit result on the current eval dataset (`dataset_v3/base_real`)

Audit artifact created:
`ml/scenarios/datasets/dataset_v3/base_real/eval_capability_audit.json`

Union of supported ranks by peer count:

- `n=1` -> supports **no ranks**
- `n=2` -> supports **rank 1 only**
- `n=3` -> supports **no ranks**
- `n=4` -> supports **ranks 1,2 only**
- `n=5` -> supports **ranks 1,2 only**

Therefore the following buckets are structurally impossible on the current eval set:

- `(1,1)`
- `(2,2)`
- `(3,1)`, `(3,2)`, `(3,3)`
- `(4,3)`, `(4,4)`
- `(5,3)`, `(5,4)`, `(5,5)`

This reproduces the old missing-bucket pattern from Run 010 and confirms the
root cause is **geometry**, not evaluator randomness.

Additional important detail:

- `n4_rank1` and `n4_rank2` are only supported by one of the two `n=4` scenarios
- `n5_rank1` and `n5_rank2` are supported, but `rank3+` is not supported at all
- this explains why the old count-only planner sometimes observed partial
  coverage for near ranks while completely missing far ranks

### Current conclusion

The current eval dataset `ml/scenarios/datasets/dataset_v3/base_real` is
**invalid for final deterministic evaluation**.

With the new capability-aware planner, this dataset now fails fast with:

```text
no eligible scenarios for bucket n1_rank1
```

That is the correct behavior. We should **not** run final Run 010 evaluation on
`dataset_v3/base_real` again.

### New conclusion after `dataset_v4_eval_repair_fixed40` rewrite

The raw 40-scenario candidate dataset was not directly suitable for variable-`n`
evaluation because its eval split remained all-`n=5`. That is now corrected in
`ml/scenarios/datasets/dataset_v4_eval_repair_fixed40/` by explicit peer-count
rewriting on a copy of the candidate set.

Important clarification from code review:

- `hazard_injector.py` targets peers that are **ahead of ego at runtime**
  (ranked by longitudinal distance), not peers inside ego's observation vector
- mesh relay and ego cone filtering are separate mechanisms
- far-source hazards can still propagate to ego via CF cascade + relay even when
  the original source is not a direct ego observation element

Working hypothesis to validate with smoke audit:

- `n5_rank5` may be a **structural bucket exclusion** for the real
  `base_real`-derived geometry rather than a dataset-generation failure
- if confirmed, final deterministic evaluation should use explicit bucket
  exclusion instead of pretending `15/15` coverage is possible on this base
  convoy geometry

---

## 3. Non-Negotiable Rules

1. **Do not retrain yet.** Freeze the Run 010 model and fix evaluation first.
2. **Hazard injection must remain ON.**
3. **Evaluation must cover `n=1,2,3,4,5`.**
4. **Evaluation must cover all source ranks `1..n` for each `n`.**
5. **Use measured emulator params only:** `ml/espnow_emulator/emulator_params_measured.json`
6. **Use `base_real`-derived scenarios only.**
7. **Do not treat any reaction-rate number as final unless `coverage_ok=true`.**

---

## 4. Definition of "True Accuracy"

For Run 010, "accuracy" is **not** a single scalar. The corrected evaluation must report:

1. **Injection success rate** per bucket
2. **Hazard reception rate** per bucket
3. **Reaction rate conditional on reception** per bucket
4. **End-to-end reaction rate** per bucket
5. **Reaction time** per bucket
6. **Min distance post-hazard** per bucket
7. **Collision rate** per bucket
8. **Unsafe-time %** per bucket

### Final headline metrics should be:

1. **Macro reaction rate** across all 15 buckets
2. **Per-bucket reaction table** for `(n, rank)`
3. **Nearest-peer vs far-peer comparison**
4. **Overall collision rate**
5. **Overall reaction time distribution**

---

## 5. Immediate Execution Plan

## Phase E0 - Freeze and Stage Run 010 Artifacts

**Status:** COMPLETE (March 8, 2026)

### Goal
Make the Run 010 checkpoint available to Docker-based evaluation and establish a fixed artifact path.

### Why this must happen first
`ml/run_docker.sh` mounts only the `roadsense-v2v/` repo into the container. The current Run 010 results live outside the repo in `/home/amirkhalifa/RoadSense2/run_010_results/`, so the model file is **not visible inside Docker** by default.

### Required actions

1. Create a repo-visible frozen model directory.
2. Copy the final Run 010 checkpoint into it.
3. Copy the current `metrics.json` and `training-run.log` for traceability.

### Recommended paths

```bash
mkdir -p /home/amirkhalifa/RoadSense2/roadsense-v2v/ml/models/frozen/cloud_prod_010

cp /home/amirkhalifa/RoadSense2/run_010_results/model_final.zip \
  /home/amirkhalifa/RoadSense2/roadsense-v2v/ml/models/frozen/cloud_prod_010/model_final.zip

cp /home/amirkhalifa/RoadSense2/run_010_results/metrics.json \
  /home/amirkhalifa/RoadSense2/roadsense-v2v/ml/models/frozen/cloud_prod_010/metrics_run_010_raw.json

cp /home/amirkhalifa/RoadSense2/run_010_results/training-run.log \
  /home/amirkhalifa/RoadSense2/roadsense-v2v/ml/models/frozen/cloud_prod_010/training-run.log
```

### Fallback
If `model_final.zip` is missing, use:
- `best_model.zip`, or
- `checkpoints/deep_sets_10000000_steps.zip`

### Exit Criteria

- Frozen model exists under the repo
- Docker can see the model path
- No further evaluation uses the external `run_010_results/` model path directly

### Actual result

- Frozen model path created:
  `ml/models/frozen/cloud_prod_010/`
- Copied successfully:
  - `model_final.zip`
  - `metrics_run_010_raw.json`
  - `training-run.log`

---

## Phase E1 - Build a Geometry-Aware Eval Capability Audit

**Status:** COMPLETE (March 8, 2026)

### Goal
Determine which eval scenarios can **actually** support which `(peer_count, rank_ahead)` buckets during the real hazard window.

### Problem this solves
The current planner knows only total peer count. It does not know whether a peer of rank `k` is actually ahead of ego at the injection step.

### Required implementation

Add a new script:

```text
ml/scripts/audit_eval_matrix_capability.py
```

### What the script must do

For each eval scenario:

1. Infer peer count from the route file
2. Sweep hazard steps across the real injection window (`30..80`)
3. For each rank `1..n`, run the real env with:
   - forced hazard
   - `hazard_target_strategy=fixed_rank_ahead`
   - `hazard_fixed_rank_ahead=<rank>`
   - `hazard_step=<step>`
4. Record whether hazard injection succeeds or fails
5. Record the failure reason if it fails

### Important implementation rule

This audit must use the **real environment and real hazard injector**, not XML-only heuristics. The goal is to mirror runtime truth.

### Required output

Write a JSON artifact like:

```text
ml/scenarios/datasets/<dataset_name>/eval_capability_audit.json
```

### Minimum JSON fields

- `scenario_id`
- `peer_count`
- `supported_ranks_any_step`
- `supported_steps_by_rank`
- `failed_steps_by_rank`
- `failure_reasons_by_rank`

### Exit Criteria

- The audit reproduces the current failure pattern on the existing eval dataset
- For every eval scenario, we know exactly which ranks are truly achievable
- We can answer "why is this bucket impossible?" before running full eval

### Actual result on `dataset_v3/base_real`

- Audit script implemented:
  `ml/scripts/audit_eval_matrix_capability.py`
- Audit executed successfully on the existing eval dataset
- Output written to:
  `ml/scenarios/datasets/dataset_v3/base_real/eval_capability_audit.json`
- The audit reproduced the coverage failure pattern exactly
- We now know that the current dataset cannot support all 15 buckets
- Dominant failure shape:
  - no valid `n=1` hazard source ahead of ego
  - no valid `n=3` buckets at all
  - `n=4` and `n=5` only support near ranks (`1,2`)

---

## Phase E2 - Make the Deterministic Eval Matrix Capability-Aware

**Status:** COMPLETE (March 8, 2026)

### Goal
Prevent the planner from scheduling impossible buckets.

### Required implementation

Upgrade the eval planner path so it can consume scenario capability metadata from Phase E1.

Files likely affected:
- `ml/eval_matrix.py`
- `ml/scripts/evaluate_model.py`
- `ml/training/train_convoy.py`
- `ml/eval_dataset.py`

### New planner behavior

For each required bucket `(n, rank)`:

1. Find scenarios with peer count `n`
2. Filter to only scenarios that can actually support `rank`
3. Build the deterministic plan using only eligible scenarios
4. Fail fast if no eligible scenario exists

### Required behavior on failure

If a required bucket has zero eligible scenarios, the planner must raise a **clear pre-run error** before any expensive evaluation starts.

### Tests required

1. Planner skips scenarios that have correct `n` but unsupported `rank`
2. Planner fails fast if a bucket has zero eligible scenarios
3. Planner still rotates assignments across multiple eligible scenarios
4. Existing count-only behavior remains valid when all ranks are supported

### Exit Criteria

- Planner never schedules a bucket that the audit marks impossible
- Unsupported buckets fail before rollout
- Deterministic matrix logic becomes trustworthy again

### Actual result

- Planner upgrade implemented
- Shared capability metadata loader added
- Unit tests added for:
  - skipping unsupported scenarios
  - failing fast when a bucket has zero eligible scenarios
  - preserving valid count-only behavior when capabilities are fully available
- Current audited dataset now fails immediately with:
  `no eligible scenarios for bucket n1_rank1`
- This prevents another misleading partial-coverage eval run

---

## Phase E3 - Regenerate and Curate a Correct Eval Dataset

**Status:** BLOCKED (`smoke audit completed` — formation breakup root cause identified, fix required)

### Goal
Create an eval dataset that can genuinely support all 15 buckets.

### Why this is necessary
Even with a capability-aware planner, the current 10-scenario eval set is too weak geometrically. We need a larger candidate pool, then a curated eval set chosen for real front-of-ego hazard support.

### Recommended strategy

1. Generate a **larger candidate dataset**
2. Audit all candidate eval scenarios with the Phase E1 tool
3. Curate a new eval subset that supports all 15 buckets
4. Preserve `base_real` and measured emulator params

### Recommended generation command

```bash
cd /home/amirkhalifa/RoadSense2/roadsense-v2v

./ml/run_docker.sh generate \
  --base_dir ml/scenarios/base_real \
  --output_dir ml/scenarios/datasets/dataset_v4_eval_repair \
  --seed 42 \
  --train_count 25 \
  --eval_count 40 \
  --peer_drop_prob 0.4 \
  --eval_peer_drop_prob 0.0 \
  --min_peers 1 \
  --emulator_params ml/espnow_emulator/emulator_params_measured.json
```

### If 40 eval scenarios are not enough

Regenerate with `--eval_count 60`.

### Important note

`fix_eval_peer_counts.py` is **not sufficient** as the final solution because it enforces peer count only. It may still be used as part of dataset preparation, but it cannot be the only gate.

### What changed in execution

For this candidate set, auditing the raw `dataset_v4_eval_repair` split before
peer-count rewriting was misleading because the eval split remained all-`n=5`.
The corrected preparation sequence is now:

1. generate raw candidate dataset
2. copy it to `dataset_v4_eval_repair_fixed40`
3. apply peer-count rewrite on the copy
4. run capability smoke audit on the rewritten eval split
5. run full audit / curation only after the smoke result matches expectations

### Actual result so far

- `dataset_v4_eval_repair_fixed40` exists in the repo
- peer-count rewrite applied successfully
- eval split now has 40 scenarios:
  - `8x n=1`
  - `8x n=2`
  - `8x n=3`
  - `8x n=4`
  - `8x n=5`
- representative route files verified after rewrite:
  - `eval_n1_000` -> 1 peer
  - `eval_n2_000` -> 2 peers
  - `eval_n3_000` -> 3 peers
  - `eval_n4_000` -> 4 peers
  - `eval_n5_000` -> 5 peers
- smoke audit command prepared but not yet completed

### Smoke audit command

```bash
cd /home/amirkhalifa/RoadSense2/roadsense-v2v

./ml/run_docker.sh python3 -m ml.scripts.audit_eval_matrix_capability \
  --dataset_dir ml/scenarios/datasets/dataset_v4_eval_repair_fixed40 \
  --scenario_ids eval_n1_000,eval_n2_000,eval_n3_000,eval_n4_000,eval_n5_000 \
  --emulator_params ml/espnow_emulator/emulator_params_measured.json \
  --seed 42 \
  --output ml/scenarios/datasets/dataset_v4_eval_repair_fixed40/eval_capability_smoke.json
```

### Smoke audit result (March 9, 2026)

**Status:** COMPLETED — result is **BAD**, only 5/15 buckets supported.

Smoke audit artifact:
`ml/scenarios/datasets/dataset_v4_eval_repair_fixed40/eval_capability_smoke.json`

| Scenario | Peers | Supported Ranks | Failed Ranks | Failure Reason |
|----------|-------|-----------------|--------------|----------------|
| `eval_n1_000` | 1 | **none** | rank 1 | `no_front_peers` |
| `eval_n2_000` | 2 | rank 1 | rank 2 | `target_not_found` |
| `eval_n3_000` | 3 | rank 1 | ranks 2, 3 | `target_not_found` |
| `eval_n4_000` | 4 | rank 1 | ranks 2, 3, 4 | `target_not_found` |
| `eval_n5_000` | 5 | ranks 1, 2 | ranks 3, 4, 5 | `target_not_found` |

Bucket coverage: **5 out of 15 (33%)**

Supported: `(2,1)`, `(3,1)`, `(4,1)`, `(5,1)`, `(5,2)`

Missing: ALL of n=1, all rank 2+ for n=2/3/4, and ranks 3-5 for n=5.

### Root cause: formation breakup during simulation (NOT starting geometry)

**The starting geometry is correct.** In `base_real`, ego (V001) is at the rear
(departPos=0.0m) and all peers start ahead:

```
V006: 82.813m  (farthest ahead)
V005: 60.813m
V004: 38.813m
V003: 16.813m
V002: 10.921m  (closest ahead)
V001:  0.000m  ← EGO (rear of convoy)
```

`fix_eval_peer_counts.py` correctly drops farthest peers first
(V006 → V005 → V004), keeping the closest-to-ego originals (V002, V003).

**The problem is that by the hazard window (steps 30-80), ego has overtaken most
peers.** The hazard injector classifies "ahead" by **current longitudinal position
at injection time**, not starting position. Peers that started ahead but got
overtaken are classified as "behind" → `no_front_peers` / `target_not_found`.

Why ego overtakes peers — **PRECISE ROOT CAUSE (confirmed March 9)**:

**`gen_scenarios.py` lines 151-164:**

```python
spawn_jitter_s = (0.0, 2.0)    # peers get 0-2 seconds delay
for vehicle in root.findall("vehicle"):
    if vehicle_id == "V001":
        continue                 # ego SKIPPED — always depart=0
    jitter = rng.uniform(0.0, 2.0)
    vehicle.set("depart", base_depart + jitter)
```

1. **Ego always departs at t=0.** Peers get 0-2s depart delay (line 154
   explicitly skips V001).
2. **Ego accelerates on an empty road** while peers haven't spawned yet.
   With `accel=2.6 m/s²` and `departSpeed=1.965 m/s`, ego reaches ~6.8 m/s
   and travels ~8.1m in just 1.85 seconds.
3. **SUMO insertion fails** when the peer tries to spawn: V002 wants
   `departPos=10.921m` but ego is at ~8.1m. The remaining gap (~2.8m) is
   less than vehicle length (5m) + tau headway. SUMO either delays
   insertion, inserts behind ego, or fails entirely.
4. **Result:** peers that started ahead of ego in `base_real` (all at
   `depart=0`) end up behind ego after jitter is applied.

Concrete example from `eval_n1_000`:
- V001: `depart=0`, `departPos=0.0m`, `departSpeed=1.965 m/s`
- V002: `depart=1.855`, `departPos=10.921m`, `departSpeed=1.746 m/s`
- At t=1.855 when V002 tries to spawn, V001 is at ~8.1m going ~6.8 m/s
- Gap = 10.921 - 8.1 = ~2.8m. Vehicle length = 5m. **Cannot insert.**

In `base_real`, all vehicles depart at `t=0` simultaneously — no conflict,
formation holds perfectly.

### Why `eval_n5_000` supports rank 1-2 but not 3-5

- V003 departs at t=0.269 (small delay), 16.8m ahead → survives
- V002 departs at t=1.072, only 10.9m ahead → borderline, may fail
- V004 departs at t=1.558, 38.8m ahead → ego traveled ~10m, gap ~28m → may survive
- V005 departs at t=1.285, 60.8m ahead → survives
- V006 departs at t=1.559, 82.8m ahead → survives

But even for far peers that spawn successfully, the speedFactor/CF model
interaction over 30-80 steps may further reshuffle the formation. Only the
2 most robustly-ahead peers remain consistently in front at hazard time.

### Implications

- The old `dataset_v3/base_real` failure had the **same root cause** — the
  `gen_scenarios.py` jitter code, not "geometry". The Phase E1 audit correctly
  identified which buckets fail but attributed it to static geometry when it
  was really a depart-time jitter problem.
- `fix_eval_peer_counts.py` cannot fix this — it controls peer count and
  starting positions, not depart time jitter.
- Generating more scenarios (60, 100) will not help — they all use the same
  jitter code.
- **Training scenarios also have this bug** — all prior training runs used
  scenarios where peers were partially behind ego due to jitter. This may
  explain why the model learned to handle only 1-2 front peers.

### Required fix

**Option A was applied (remove depart time jitter) — PARTIALLY EFFECTIVE.**

Set `spawn_jitter_s = (0.0, 0.0)` so all vehicles depart at `t=0`
simultaneously. This fixed n=1 (from "no ranks" to "rank 1") but did NOT
fix ranks 2+ for any peer count. Post-fix smoke audit still shows only
5/15 buckets.

**Root cause is deeper than jitter — see
`FORMATION_STABILITY_FIX_PLAN.md` for the full three-issue analysis:**

1. **SUMO safe-insertion staggering:** Even with depart="0", SUMO delays
   V003-V006 by 5-18 seconds because V002-V003 gap (5.9m) is below the
   safe insertion threshold (9.6m). This is "hidden jitter" from SUMO.
2. **Episode too long for route:** 785m route at 13.89 m/s → vehicles exit
   at ~57s, but episode runs 100s. 43% of each episode is wasted.
3. **CF model compresses formation:** Small initial gaps → bumper-to-bumper
   within seconds. Rank ordering breaks down.

### Previous contingency (now superseded)

~~If smoke audit confirms `n5_rank5` remains unreachable, use explicit exclusion:~~

The problem is much larger than `n5_rank5`. 10 of 15 buckets are broken.
Exclusion is not viable — the fix must address formation stability.

### Recommended curation approach

Prefer a new curation step or script that selects scenarios based on:

1. required peer count
2. actual supported ranks from the capability audit
3. diversity across scenarios

### Implementation update

A new curation script now exists:

```text
ml/scripts/curate_eval_by_capability.py
```

It:

1. reads `eval_capability_audit.json`
2. checks bucket coverage for all required `(n, rank)` pairs
3. greedily selects a minimal supporting eval subset
4. writes a curated dataset with:
   - filtered `eval_scenarios`
   - filtered `eval_capability_audit.json`
   - curation metadata recorded in `manifest.json`

### Recommended curation command

```bash
cd /home/amirkhalifa/RoadSense2/roadsense-v2v/ml
source venv/bin/activate
cd ..

python3 -m ml.scripts.curate_eval_by_capability \
  --source_dataset_dir ml/scenarios/datasets/dataset_v4_eval_repair \
  --output_dataset_dir ml/scenarios/datasets/dataset_v4_eval_repair_curated \
  --required_peer_counts 1,2,3,4,5 \
  --min_scenarios_per_bucket 1
```

If this fails for `min_scenarios_per_bucket=1`, the candidate pool is still too
weak and we must regenerate with `--eval_count 60`.

### Desired dataset property

At minimum:
- every bucket has at least **one** valid supporting scenario

Preferred:
- every bucket has **two or more** supporting scenarios

### Exit Criteria

- New eval dataset exists
- Capability audit confirms support for all 15 buckets
- The eval set is selected based on actual runtime capability, not only peer count

---

## Phase E4 - Re-Evaluate the Frozen Run 010 Model

### Goal
Run a corrected, trustworthy evaluation using the frozen model and the repaired eval dataset.

### Command template

```bash
cd /home/amirkhalifa/RoadSense2/roadsense-v2v

./ml/run_docker.sh python3 -m ml.scripts.evaluate_model \
  --model_path ml/models/frozen/cloud_prod_010/model_final.zip \
  --dataset_dir ml/scenarios/datasets/dataset_v4_eval_repair_curated \
  --emulator_params ml/espnow_emulator/emulator_params_measured.json \
  --seed 42 \
  --use_deterministic_matrix \
  --matrix_peer_counts "1,2,3,4,5" \
  --matrix_episodes_per_bucket 10 \
  --output ml/models/frozen/cloud_prod_010/eval_full_coverage.json
```

### Notes

1. In deterministic matrix mode, `evaluate_model.py` already forces hazard injection per planned episode.
2. Do not disable hazard injection.
3. Do not swap in a newer checkpoint.

### Required success gates

The corrected evaluation is only valid if all of the following are true:

1. `coverage_ok=true`
2. `missing_buckets=[]`
3. `injection_attempted > 0`
4. every required bucket has observed episodes
5. no silent fallback to count-only assumptions

### Required outputs

- `eval_full_coverage.json`
- bucket table for all 15 `(n, rank)` combinations
- updated markdown summary comparing:
  - Run 009
  - Run 010 old eval
  - Run 010 corrected eval

---

## Phase E5 - Decision Gate After Corrected Eval

### Goal
Use the corrected eval to decide whether Run 010 is deployment-candidate material or whether Run 011 is needed first.

### Decision outcomes

#### Outcome A - Strong Full-Coverage Result

Conditions:
- high reaction rate across most buckets
- zero collisions
- acceptable reaction time
- no catastrophic collapse on far-peer hazards

Next step:
- move to H5 sim-to-real validation
- then quantization / deployment prep

#### Outcome B - Rank-1 Strong, Rank-2+ Weak

Conditions:
- nearest-peer hazards are robust
- far-peer hazards degrade sharply

Next step:
- Run 011 should target reward shaping or hazard-signal sensitivity
- do **not** change hyperparameters yet

#### Outcome C - Corrected Eval Falls Materially Below 87%

Conditions:
- corrected coverage reveals earlier result was optimistic

Next step:
- treat old 87% as a partial-coverage artifact
- diagnose by bucket before retraining

#### Outcome D - Coverage Still Fails

Conditions:
- planner or curated dataset still cannot realize all buckets

Next step:
- stop interpretation immediately
- continue fixing eval pipeline until coverage succeeds

---

## 6. Implementation Order For the Agent

The agent should execute in this exact order:

1. Freeze and copy the Run 010 checkpoint into a repo-visible path
2. Build the capability audit script
3. Prove the audit reproduces the current coverage failure
4. Upgrade the deterministic planner to use capability data
5. Add unit tests for the planner changes
6. Generate a larger candidate eval dataset
7. Run the capability audit on the candidate dataset
8. Curate a repaired eval dataset with full bucket support
9. Re-run evaluation on the frozen Run 010 model
10. Write a corrected summary only after `coverage_ok=true`

### Progress against order

1. Freeze and copy the Run 010 checkpoint into a repo-visible path -> ✅ COMPLETE
2. Build the capability audit script -> ✅ COMPLETE
3. Prove the audit reproduces the current coverage failure -> ✅ COMPLETE
4. Upgrade the deterministic planner to use capability data -> ✅ COMPLETE
5. Add unit tests for the planner changes -> ✅ COMPLETE
6. Generate a larger candidate eval dataset -> ✅ COMPLETE (dataset_v4_eval_repair + fixed40 rewrite)
6b. Smoke audit on representative scenarios -> ✅ COMPLETE — **FAILED: 5/15 buckets**
6c. Fix formation breakup (ego overtakes peers) -> **BLOCKED — jitter fix partial, deeper issues found (see FORMATION_STABILITY_FIX_PLAN.md)**
7. Run the capability audit on the candidate dataset -> PENDING (blocked on 6c)
8. Curate a repaired eval dataset with full bucket support -> PENDING
9. Re-run evaluation on the frozen Run 010 model -> PENDING
10. Write a corrected summary only after `coverage_ok=true` -> PENDING

---

## 7. Verification Checklist

Before calling the corrected evaluation "final", verify all boxes below:

- [x] Run 010 model is frozen and repo-visible
- [x] Capability audit exists and is reproducible
- [x] Planner is capability-aware
- [x] Planner has unit tests
- [x] Candidate eval dataset regenerated from `base_real`
- [x] Smoke audit completed on fixed40 dataset — **5/15 buckets, formation breakup**
- [ ] Formation breakup fix applied (ego must not overtake peers during hazard window)
- [ ] Repaired eval dataset supports all 15 buckets
- [ ] Deterministic eval runs with hazard injection ON
- [ ] `coverage_ok=true`
- [ ] `missing_buckets=[]`
- [ ] Per-bucket reaction table generated
- [ ] Final decision made from corrected eval, not partial-coverage eval

---

## 8. Bottom Line

The immediate next move is **not** another training run.

The immediate next move is to:

1. fix evaluation geometry awareness
2. rebuild the eval set so all 15 buckets are truly possible
3. re-evaluate the frozen Run 010 model

Only after that will we know how good Run 010 actually is.
