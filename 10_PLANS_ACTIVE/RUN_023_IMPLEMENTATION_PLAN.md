# Run 023 Implementation Plan — State-Triggered Hazard Onset on `base_real`

**Date:** March 16, 2026
**Status:** Ready to implement
**Predecessor:** `cloud_prod_022` (Run 022 postmortem + Run 023 hypothesis set)
**Goal:** Recover replay sensitivity **without** reintroducing route-coded features by broadening the **relative vehicle state at hazard onset** on the same `base_real` road and route.

---

## 1. Problem Statement

Run 022 proved two things at once:
- removing ego heading was correct
- removing ego heading was not sufficient

What Run 022 fixed:
- Recording #2 false positives: `19.16%` → `2.55%`
- Extra Driving false positives: `93.83%` → `4.58%`

What still failed:
- Recording #2 sensitivity: `12.0%`
- Extra Driving sensitivity: `26.1%`
- weighted SUMO V2V reaction only `~64.4%`

Interpretation:
- Run 021 was over-reactive because it learned a route-position shortcut through heading
- Run 022 removed that shortcut and the model became conservative again
- the remaining causal signal is still not diverse enough at the **moment braking begins**

This is now the primary Run 023 hypothesis:

> On the correct `base_real` road, hazards still begin from too narrow a set of
> **relative convoy states**, so the policy does not learn a robust early-braking
> rule from causal V2V cues alone.

---

## 2. Scope and Non-Goals

### 2.1 What Run 023 WILL preserve

- `base_real` as the road / route grounding
- heading-free ego observation from Run 022
- Deep Sets architecture
- decay-based `braking_received`
- current reward structure
- current PPO hyperparameters
- CF override
- hazard decel randomization `[3.0, 10.0] m/s²`
- resolved-hazard episodes
- deterministic eval matrix flow

### 2.2 What Run 023 will NOT change

- no new road or route family
- no absolute position features
- no ego heading reintroduction
- no `progress` feature
- no reward redesign in the first pass
- no generator-level augmentation redesign in the first pass

Important:
- this is **not** a proposal to replace `base_real`
- this is **not** a proposal to change the map
- this is **not** a proposal to “just randomize time harder”

Run 023 changes **hazard onset state**, not map geometry.

---

## 3. Concrete Solution

## 3.1 Preserve deterministic eval; change training onset logic

Run 023 should split hazard triggering into two modes:

1. `fixed_step`
   - preserves current behavior
   - used by deterministic eval matrix and any explicit `hazard_step` override

2. `state_bucket`
   - new Run 023 training mode
   - used by training episodes by default

This keeps eval reproducible while letting training see broader onset states.

---

## 3.2 State-triggered onset on the same route

### Core idea

Instead of:
- “at step 200, pick a front peer and brake it”

Run 023 should do:
- “within the training search window, brake a front peer **when the chosen source reaches a sampled onset bucket**”

The road stays the same.
The route stays the same.
The hazard target still comes from front peers only.

Only the **relative scene at onset** becomes more diverse.

### Training-only onset search window

For `state_bucket` mode:
- search start: step `150`
- search end: step `300`
- hard fallback injection: step `300`

Rationale:
- preserves the current late-episode geometry discipline
- leaves enough post-hazard runway in a 500-step episode
- avoids silent “no hazard happened” episodes

### Target selection logic

At the beginning of the search window:
- compute current front peers with rank ahead
- sample `desired_rank_ahead` uniformly from the ranks actually available in that episode

This preserves the current “uniform front peer” philosophy while making the chosen source explicit before onset.

### Onset bucket logic

Run 023A should gate onset on **rank + longitudinal gap** only.

Do **not** gate onset on closing speed in the first pass.
Closing speed should be logged for diagnostics, not used as a hard trigger yet.

#### Gap buckets

Use longitudinal ego-to-source gap on the same route, not Euclidean distance.

Initial bucket set:
- `close`: `[12m, 22m)`
- `medium`: `[22m, 35m)`
- `far`: `[35m, 55m)`
- `very_far`: `[55m, 85m)`

These ranges were chosen to:
- include the recorded convoy’s average spacing scale (`~13.9m`)
- include the current redistributed `base_real` spacing scale (`25m`, `50m`, `75m`, ...)
- stay inside the observation horizon (`100m`)

#### Rank-conditioned bucket sampling

To avoid sampling obviously unreachable combinations, sample the gap bucket conditioned on rank:

- `rank_1`:
  - `close`, `medium`, or `far`
- `rank_2`:
  - `medium`, `far`, or `very_far`
- `rank_3+`:
  - `far` or `very_far`

This is intentionally coarse. Run 023 is about broadening onset states, not perfectly modeling every rank-specific distribution.

### Injection condition

During steps `150..300`:
- track the chosen rank-ahead peer
- compute its current longitudinal gap to ego each step
- inject when that gap enters the sampled bucket

### Fallback behavior

If no match occurs by step `300`:
- inject at step `300`
- keep the pre-selected rank if still available
- otherwise fall back to current `uniform_front_peers` behavior

Every fallback must be logged explicitly.

---

## 3.3 New onset metadata and instrumentation

Run 023 must produce auditable onset telemetry.

Add the following metadata to `HazardInjector` and propagate through `ConvoyEnv.info`:
- `hazard_trigger_mode`: `state_bucket` or `fixed_step`
- `hazard_trigger_result`: `bucket_match` or `fallback_step`
- `hazard_onset_gap_bucket`
- `hazard_onset_gap_m`
- `hazard_onset_closing_speed_mps`
- `hazard_onset_peer_count`
- `hazard_onset_search_start_step`
- `hazard_onset_trigger_step`
- `hazard_onset_desired_rank_ahead`

### Metrics aggregation

`metrics.json` for Run 023 must include:
- `onset_trigger_summary`
- `fallback_trigger_rate`
- counts by `hazard_onset_gap_bucket`
- counts by `hazard_onset_desired_rank_ahead`
- reaction rate by onset bucket

Without this, Run 023 is not interpretable enough to justify another cloud cycle.

---

## 3.4 Local feasibility audit before cloud launch

Add a new script:
- `ml/scripts/audit_hazard_onset_coverage.py`

Purpose:
- run the environment in training mode with forced hazard episodes
- record how often each onset bucket actually triggers
- measure fallback rate before any PPO training

This script must answer:
- Are the chosen buckets actually reachable on `base_real`-derived scenarios?
- Is state-bucket mode broadening onset, or just falling back most of the time?

### Minimum required audit outputs

- total episodes
- bucket-match episodes
- fallback episodes
- fallback rate
- counts by rank
- counts by gap bucket
- onset gap histogram
- onset closing-speed histogram

---

## 3.5 Replay hygiene fix

Run 023 should also fix one replay-analysis hygiene issue:
- exclude `V001` self-messages from RX processing in `validate_against_real_data.py`

This is not the root cause of Run 022, but it should be cleaned up before the next replay comparison.

---

## 4. Files to Change

| File | Planned change |
|------|----------------|
| `roadsense-v2v/ml/envs/hazard_injector.py` | Add `trigger_mode`, onset-bucket sampling, longitudinal gap helper, state-bucket search, fallback injection, onset metadata properties |
| `roadsense-v2v/ml/envs/convoy_env.py` | Expose new onset metadata in `info` |
| `roadsense-v2v/ml/training/train_convoy.py` | Aggregate onset-trigger summaries into `metrics.json` |
| `roadsense-v2v/ml/scripts/check_v2v_reaction.py` | Include onset-trigger summaries in verdict output |
| `roadsense-v2v/ml/scripts/audit_hazard_onset_coverage.py` | New preflight audit script |
| `roadsense-v2v/ml/scripts/validate_against_real_data.py` | Filter `V001` self-RX rows |
| `roadsense-v2v/ml/tests/unit/test_hazard_injector.py` | Add tests for state-bucket triggering, bucket sampling, fallback behavior, onset metadata |
| `roadsense-v2v/ml/tests/unit/test_convoy_env.py` | Add assertions for new `info` fields |
| `roadsense-v2v/ml/tests/unit/test_validate_against_real_data.py` | Add self-RX exclusion regression |
| `roadsense-v2v/ml/scripts/cloud/run_training.sh` | Update comments, dataset ID, run ID, and acceptance text for Run 023 |

---

## 5. Implementation Order

1. Add `trigger_mode` support to `HazardInjector`
2. Add longitudinal gap computation for a specific target
3. Add rank-conditioned bucket sampling
4. Implement search-window trigger logic
5. Implement step-300 fallback path
6. Expose onset metadata via properties and `ConvoyEnv.info`
7. Add onset aggregation to training/eval reporting
8. Add `audit_hazard_onset_coverage.py`
9. Fix self-RX filtering in replay validator
10. Add unit tests
11. Run targeted local tests
12. Run Docker integration suite
13. Run local onset audit
14. Run a short local smoke train
15. Launch the 2M cloud diagnostic only if the preflight gates pass

---

## 6. Preflight Gates

### 6.1 Code gates

Must pass before any training:
- targeted unit tests for injector / env / replay validator
- Docker integration tests

### 6.2 Onset coverage audit gates

Must pass before any cloud launch:
- fallback rate `<= 25%`
- at least `3` gap buckets observed in audit output
- state-bucket triggers observed for peer counts `n=1..5`
- no silent hazard-drop episodes

If these fail:
- do **not** launch cloud
- fix the trigger design first

### 6.3 Local smoke-train gates

Run a short local smoke train after the audit passes.

Minimum expectations:
- training completes normally
- non-zero V2V reaction in smoke check
- no obvious critic collapse

If smoke stays at zero reaction:
- do **not** launch cloud

---

## 7. Training Plan

### Phase 1: Local onset audit

Goal:
- prove the new onset-trigger design actually broadens hazard onset states on `base_real`

### Phase 2: Local smoke train

Goal:
- confirm the modified trigger path still produces trainable hazards

Suggested size:
- `100k` to `300k` timesteps

### Phase 3: 2M cloud diagnostic

Goal:
- test whether broader onset states recover sensitivity without breaking specificity

Hard kill criteria:
- `explained_variance <= 0.1` at `500k`
- weighted SUMO V2V reaction `< 50%` at final diagnostic eval
- fallback trigger rate `> 35%`

Diagnostic success target:
- weighted SUMO V2V reaction `> 75%`
- `0%` collisions
- fallback trigger rate `<= 20%`

### Phase 4: Real-data replay validation

MANDATORY before any 10M extension.

Acceptance targets stay unchanged:

| Recording | Sensitivity | False Positive Rate |
|-----------|-------------|---------------------|
| Recording #2 | `> 60%` | `< 15%` |
| Extra Driving | `> 75%` | `< 20%` |

### Phase 5: 10M production run

Only if:
- preflight passes
- 2M diagnostic clears the SUMO bar
- both replay recordings pass

---

## 8. Risks and Decision Rules

| Risk | Why it matters | Mitigation |
|------|----------------|------------|
| Buckets are not actually reachable | State-trigger mode degenerates into fallback-step mode | Local onset audit before cloud launch |
| Trigger broadens onset too weakly | Replay sensitivity will not move | Log bucket coverage and fallback rate explicitly |
| Trigger broadens onset too aggressively | SUMO reaction may collapse | Keep reward and observation unchanged in first pass |
| No improvement after H1 | Missing causal state may still be the blocker | Move to H2 from the Run 023 hypothesis set: add one deployment-safe risk feature |
| Another ambiguous run | More time burned without learning anything | Require onset-trigger metrics in `metrics.json` |

Decision rule:
- If Run 023 fails replay but shows clearly broader onset coverage and better SUMO reaction than Run 022, move to **H2**
- If Run 023 fails because fallback dominates, fix trigger design before any new training run
- Do **not** bundle reward redesign into this first implementation pass

---

## 9. Predecessor Comparison

| Aspect | Run 022 | Run 023 |
|--------|---------|---------|
| Road / route grounding | `base_real` | Same |
| Ego heading | Removed | Still removed |
| Hazard onset | Effectively fixed-time (`step 200` default) | **State-triggered on same route** |
| Trigger state diversity | Narrow | **Broader rank/gap onset states** |
| Reward | Unchanged | Unchanged |
| Observation | 5-dim ego | Same |
| New telemetry | Limited | **Onset bucket / fallback summaries** |

---

## 10. Concrete Working Summary

Run 023 should be implemented as:

> Keep `base_real`, keep the same road, keep the heading-free observation, and
> replace effectively fixed-time training hazards with **state-triggered hazards**
> that begin when a chosen front peer reaches a sampled rank/gap onset bucket.

This is the narrowest plausible change set that directly tests the current best hypothesis.
