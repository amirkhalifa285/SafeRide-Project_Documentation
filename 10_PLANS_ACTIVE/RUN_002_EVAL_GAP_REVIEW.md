# Run 002 Evaluation Gap - Review Request

**Created:** February 21, 2026
**Status:** EVAL COMPLETE (Feb 21) — 40 episodes, hazards ON, 0 collisions. BUT only n=1,2 covered. Need eval scenarios with n=3,4,5+ to satisfy Phase 6.6.3.
**Audience:** Agent reviewer / implementation agent

## 0) Implementation Progress

### Session 1 (Feb 21, 2026) — Previous agent
- [x] Step A - Metrics schema expanded in both evaluation paths.
- [x] Step B - Peer count extraction from `vehicles.rou.xml` implemented.
- [x] Step C - Coverage control added via `--episodes_per_scenario`.
- [~] Step D - Hazard mode field added to output BUT defaulted to False (opt-in). **BUG.**
- [x] Step E - Unit tests added.

### Session 2 (Feb 21, 2026) — CRITICAL FIX + VALID EVAL RUN
- [x] **Step D FIX** - `hazard_injection` now defaults to **True** in both scripts.
  - `train_convoy.py`: flag renamed `--no_eval_hazard_injection` (opt-out).
  - `evaluate_model.py`: flag renamed `--no_hazard_injection` (opt-out).
  - Previous 40-episode run was **INVALID** (ran with `hazard_injection=False`).
- [x] Step E updated - Two new tests: `test_evaluate_hazard_injection_on_by_default`, `test_evaluate_hazard_injection_disabled_with_flag`. All 12 tests passing.
- [x] CLAUDE.md updated with "Evaluation MUST Use Hazard Injection" rule for all agents.
- [x] **40-episode eval re-run with hazard_injection=True** — COMPLETED. Results below.
- [ ] Run 001 vs Run 002 `n=2` delta not yet computed.
- [ ] **BLOCKING: Eval scenarios only contain n=1 and n=2. Need n=3,4,5+ scenarios to satisfy Phase 6.6.3.**

### Valid Evaluation Results (Feb 21, 2026 — hazard_injection=True)

**Artifact:** `ml/models/runs/cloud_prod_002/results/eval_final_40ep_hazard_on.json`

| Metric | Value |
|---|---|
| Episodes | 40 (K=10 per scenario) |
| Hazard Injection | **True** |
| Collisions | 0/40 |
| Success Rate | **100%** |
| Avg Reward | 516.14 (+/- 65.36) |
| Min/Max Reward | 444.5 / 908.5 |
| Avg Length | 1000.0 steps |

**Per-Scenario Breakdown:**

| Scenario | Peers | Episodes | Collisions | Success Rate | Avg Reward | Std |
|---|---|---|---|---|---|---|
| eval_000 | 2 | 10 | 0 | 100% | 516.7 | 24.1 |
| eval_001 | 1 | 10 | 0 | 100% | 484.6 | 1.7 |
| eval_002 | 1 | 10 | 0 | 100% | 545.4 | 121.0 |
| eval_003 | 2 | 10 | 0 | 100% | 518.0 | 1.1 |

**Per-Peer-Count Breakdown:**

| Peer Count | Episodes | Collisions | Success Rate | Avg Reward |
|---|---|---|---|---|
| n=1 | 20 | 0 | 100% | 515.0 |
| n=2 | 20 | 0 | 100% | 517.3 |

**Interpretation:**
- Model handles hazard injection without any collisions across all tested scenarios.
- High-reward spikes (908.5) indicate successful hazard avoidance events earning bonus reward.
- Low-variance scenarios (eval_001 std=1.7, eval_003 std=1.1) had few/no hazard events.
- High-variance scenarios (eval_002 std=121.0) had frequent hazard events — all avoided.

### REMAINING GAP: Peer Count Coverage

**This evaluation only covers n=1 and n=2 peers.**

Phase 6.6.3 requires: "Model handles variable n (tested on n=1,2,3,4,5)".
The current eval dataset (`dataset_v2/base`) only has eval scenarios with 1 or 2 peers.
This means **we still cannot prove the model works for n=3,4,5.**

The Deep Sets architecture is designed for variable n, and training included variable peer counts,
but the evaluation evidence does not yet cover n>2.

## 1) Problem Summary
Training Run 002 appears strong, but the evaluation evidence is not yet sufficient for a final-model claim.

Current artifact (`roadsense-v2v/ml/models/runs/cloud_prod_002/results/metrics.json`) shows:
- `training_timesteps`: 10,000,000
- `eval_episodes`: 10
- `avg_reward`: 507.25
- `collisions`: 0
- `collision_rate`: 0.0
- `truncations`: 10
- `truncation_rate`: 1.0
- Scenarios sampled: `eval_000..eval_003` repeated

## 2) Why This Is a Gap
The project requirements call for stronger evaluation evidence than what Run 002 currently logs.

### Evidence of mismatch
- Required eval protocol expects 20+ episodes and per-peer-count reporting.
  - `docs/10_PLANS_ACTIVE/N_ELEMENT_IMPLEMENTATION_PLAN.md` (Eval Protocol section)
- Phase 6 success criteria requires demonstrating variable-n handling and no regression.
  - `docs/10_PLANS_ACTIVE/PHASE_6_REAL_DATA_PIPELINE.md` (Phase 6.6.3)
- Current training eval code records only aggregate collisions/reward and scenario IDs.
  - `roadsense-v2v/ml/training/train_convoy.py` (`evaluate()`)

### Practical risk
- 10 episodes is too small to claim robust generalization.
- No per-n breakdown means we cannot prove performance across `n in {1,2,3,4,5}`.
- All episodes truncated (`truncation_rate=1.0`): safe (no collision) but lacks richer outcome stratification.

## 3) Objective for Fix
Upgrade evaluation from a single aggregate snapshot to a reproducible evidence package suitable for final-model validation and thesis defense.

## 4) Proposed Fix Plan

### Step A - Expand metrics schema
Update evaluation outputs to include:
- `success_rate` (in addition to `collision_rate`)
- `scenario_summary`:
  - episodes
  - collisions / collision_rate
  - truncations / truncation_rate
  - avg_reward
  - avg_length
- `peer_count_summary` (same metrics grouped by declared scenario peer count)
- Optional `episode_details` (scenario_id, peer_count, reward, length, outcome)

Target files to modify:
- `roadsense-v2v/ml/training/train_convoy.py` (primary training-time eval)
- `roadsense-v2v/ml/scripts/evaluate_model.py` (standalone post-hoc eval)

Reviewer note:
- `evaluate_model.py` already computes `success_rate`; it still needs per-scenario and per-peer-count breakdown.
- `train_convoy.py` needs full schema expansion.

### Step B - Derive peer count per scenario
Compute peer count from each scenario route file:
- `dataset_dir/{train|eval}/{scenario_id}/vehicles.rou.xml`
- Count `<vehicle>` entries excluding `V001`

Reason: current manifest has only global `peer_count_distribution`, not guaranteed per-scenario mapping.

### Step C - Raise evaluation coverage
Support explicit coverage mode for final validation:
- Either `--eval_episodes >= 40` minimum
- Or `--episodes_per_scenario K` to force balanced repetition over all eval scenarios

Recommended baseline for final check:
- `K=10` per eval scenario (if 4 eval scenarios -> 40 episodes)
- Produce both aggregate and per-scenario/per-n metrics

### Step D - Add hazard-mode clarity ✅ FIXED (Session 2)
**CRITICAL:** Hazard injection MUST default to True for all evaluation.
- `train_convoy.py`: `--no_eval_hazard_injection` flag to opt-out (default: hazards ON)
- `evaluate_model.py`: `--no_hazard_injection` flag to opt-out (default: hazards ON)
- Hazard mode recorded in output JSON as `"hazard_injection": true/false`
- Optionally run a second baseline pass with `--no_hazard_injection` for comparison

### Step E - Test updates
Add/adjust unit tests for:
- New metric keys presence
- Correct success/collision math
- Correct per-scenario aggregation
- Correct per-peer-count aggregation

Likely test file:
- `roadsense-v2v/ml/tests/unit/test_train_convoy.py`

## 5) Acceptance Criteria
A fix is considered complete only when all are true:
1. Evaluation JSON includes aggregate + per-scenario + per-peer-count stats.
2. Final eval run uses at least 20 episodes (preferably 40+).
3. Results explicitly report performance by peer count (`n`).
4. **Evaluation runs with `hazard_injection=True` (the default).** Results without hazards are not valid.
5. Run 002 evidence can answer:
   - Did we regress vs Run 001 on n=2?
   - How does model perform on each n bucket?
6. Documentation updated with exact post-fix metrics and date.

## 6) Suggested Execution Commands (for implementer)
From repo root:

```bash
cd /home/amirkhalifa/RoadSense2/roadsense-v2v
# After code changes:
./ml/run_docker.sh pytest ml/tests/unit/test_train_convoy.py -q
```

Dataset locality note (critical):
- `dataset_v2` is not available on this local machine (Fedora workspace) as of February 21, 2026.
- `metrics.json` references `ml/scenarios/datasets/dataset_v2/base`, but only `dataset_v1` exists locally.
- Final 40-episode evidence run must be done either:
  - after pulling `dataset_v2` from EC2/S3 into local workspace, or
  - directly on EC2 where `dataset_v2` already exists.

Run stronger standalone evaluation (hazard injection is ON by default — no flag needed):

```bash
cd /home/amirkhalifa/RoadSense2/roadsense-v2v
./ml/run_docker.sh python -m ml.scripts.evaluate_model \
  --model_path ml/models/runs/cloud_prod_002/results/model_final.zip \
  --dataset_dir ml/scenarios/datasets/dataset_v2/base \
  --emulator_params ml/espnow_emulator/emulator_params_measured.json \
  --episodes 40 \
  --output ml/models/runs/cloud_prod_002/results/eval_final_40ep_hazard.json
```

Optional baseline comparison (hazards off — only for side-by-side, NOT for final gate):

```bash
./ml/run_docker.sh python -m ml.scripts.evaluate_model \
  --model_path ml/models/runs/cloud_prod_002/results/model_final.zip \
  --dataset_dir ml/scenarios/datasets/dataset_v2/base \
  --emulator_params ml/espnow_emulator/emulator_params_measured.json \
  --episodes 40 \
  --no_hazard_injection \
  --output ml/models/runs/cloud_prod_002/results/eval_final_40ep_baseline.json
```

## 7) Reviewer Questions
1. Is per-scenario peer-count extraction from `vehicles.rou.xml` acceptable as source of truth?
2. Should final gate require one or two evaluation modes (`hazard_injection` on/off)?
3. What is the minimum episode count for final acceptance: 20 or 40?
4. Do we need confidence intervals/bootstrapping in addition to point estimates?

## 8) Next Steps — BLOCKING

### Step F - Create eval scenarios with n=3, 4, 5 peers (REQUIRED)

The entire point of the Deep Sets architecture is handling variable n.
**The eval dataset must include scenarios with more than 2 peers.**

Action required:
1. Create new base scenarios with 3, 4, and 5 peer vehicles (V002 through V006).
2. Run augmentation to generate eval variants.
3. Re-run the 40+ episode evaluation covering all n buckets.
4. The final eval JSON must have `peer_count_summary` entries for n=1, n=2, n=3, n=4, n=5.

Without this, Phase 6.6.3 ("Model handles variable n, tested on n=1,2,3,4,5") is NOT satisfied.

**DO NOT mark Run 002 evaluation as complete until n=3,4,5 peer count results exist.**

## 9) Notes
- **CORRECTED (Session 2):** Final gate mode MUST use `hazard_injection=True` (now the default). Without hazards, evaluation only proves the model doesn't crash in calm traffic, which is useless.
- Minimum final acceptance coverage: 40 episodes (recommended `K=10` per eval scenario).
- Code implementation completed in:
  - `roadsense-v2v/ml/training/train_convoy.py`
  - `roadsense-v2v/ml/scripts/evaluate_model.py`
  - `roadsense-v2v/ml/tests/unit/test_train_convoy.py`
- Session 1 verification (Feb 21): `pytest` -> 10 passed.
- **Session 2 fix (Feb 21):** Flipped hazard default to True, added 2 new tests -> 12 passed.
- **Session 2:** CLAUDE.md updated with permanent "Evaluation MUST Use Hazard Injection" rule.
- **eval_final_40ep.json is INVALID** — ran with hazard_injection=False.
- **eval_final_40ep_hazard_on.json is VALID** — 40 episodes, hazard_injection=True, 0 collisions, 100% success. But only covers n=1 and n=2.
