# Cloud Training Automation Progress Tracker

Status: Active
Owner: Planner
Goal: One-button training pipeline (generate -> augment -> train -> eval -> save artifacts)
Last Updated: January 16, 2026

## Scope
- Local single-command pipeline that can be triggered on demand.
- Cloud-compatible artifacts and metadata for later AWS runs.
- No changes to core Deep Sets architecture assumptions.

## Key References
- docs/10_PLANS_ACTIVE/CLOUD_TRAINING_PIPELINE_PROPOSAL.md
- docs/10_PLANS_ACTIVE/CLOUD_TRAINING_PIPELINE_VERDICT.md
- docs/00_ARCHITECTURE/DEEP_SETS_N_ELEMENT_ARCHITECTURE.md
- docs/00_ARCHITECTURE/ESPNOW_EMULATOR_DESIGN.md
- docs/20_KNOWLEDGE_BASE/ML_DOCKER_ENVIRONMENT.md
- docs/20_KNOWLEDGE_BASE/ML_AUGMENTATION_PIPELINE.md

## Status Key
- NOT STARTED
- SPEC READY (awaiting Builder)
- IN PROGRESS
- BLOCKED
- COMPLETE

## Phase Status

| Phase | Name | Status | Spec | Builder Report | Review |
|-------|------|--------|------|----------------|--------|
| 0 | Readiness and invariants | COMPLETE | IMPLEMENTATION_PLAN.md | N/A | N/A |
| 1 | Scenario generator + manifest | **COMPLETE** | [PHASE_1_SPEC.md](PHASE_1_SPEC.md) | - | [APPROVED](PHASE_1_REVIEW.md) |
| 2 | Scenario selection + SUMO reload | **COMPLETE** | [PHASE_2_SPEC.md](PHASE_2_SPEC.md) | - | [APPROVED](PHASE_2_REVIEW.md) |
| 3 | Training CLI + evaluation harness | **COMPLETE** | [PHASE_3_SPEC.md](PHASE_3_SPEC.md) | - | - |
| 4 | One-button runner + artifact packaging | **COMPLETE** | [AWS_EXECUTION_PLAN](../CLOUD_TRAINING_AWS_EXECUTION_PLAN.md) | - | - |
| 5 | Cloud handoff | **IN PROGRESS** | [aws_training_agent_prompt.md](../../../aws_training_agent_prompt.md) | - | - |

### Phase 5: First Cloud Run (Jan 16, 2026)
- Instance: c6i.xlarge (On-Demand) in il-central-1
- Run ID: `cloud_prod_001`
- Status: **TRAINING** (5M timesteps, ~24-48 hours expected)
- Results: `s3://saferide-training-results/cloud_prod_001/`

## Phase 0 Invariants (Locked)

These invariants are captured in IMPLEMENTATION_PLAN.md and enforced in all subsequent phases:

1. **V001 Spawn:** V001 must exist at t=0. ConvoyEnv waits for V001 on reset.
2. **No Peer Manipulation:** No padding, sorting, or slotting of peers. Deep Sets handles variable-n.
3. **Fixed Eval Set:** Evaluation scenarios are deterministic (fixed list, not randomized per run).
4. **Manifest Tracking:** Every generated dataset includes a manifest.json for reproducibility.

## Phase 1 Summary (COMPLETE)

**Objective:** Create `gen_scenarios.py` that generates augmented SUMO scenarios with train/eval splits and manifest.

**Result:** APPROVED - All tests pass, V001 invariant enforced.
**Update (2026-01-15):** Route files are now sorted by depart time to satisfy SUMO ordering and prevent V001 from being ignored.

---

## Phase 2 Summary

**Objective:** Enable ConvoyEnv to select scenarios from a dataset directory on each reset.

**Files to Create:**
- `roadsense-v2v/ml/tests/unit/test_scenario_manager.py`

**Files to Modify:**
- `roadsense-v2v/ml/envs/scenario_manager.py` (replace placeholder)
- `roadsense-v2v/ml/envs/sumo_connection.py` (add `set_config()`)
- `roadsense-v2v/ml/envs/convoy_env.py` (add `dataset_dir` parameter)
- `roadsense-v2v/ml/envs/__init__.py` (export ScenarioManager)
- `roadsense-v2v/ml/tests/unit/test_convoy_env.py` (add dataset tests)

**Exit Criteria:**
- ScenarioManager loads manifest and selects scenarios
- Train mode: random selection, Eval mode: sequential
- No SUMO process leaks after 10+ resets
- Backward compatible with existing `sumo_cfg` usage

**Result:** APPROVED - Process leak test confirmed clean shutdown.

---

## Phase 3 Summary

**Objective:** Add CLI arguments to `train_convoy.py` for dataset selection and implement post-training evaluation.

**Files to Modify:**
- `roadsense-v2v/ml/training/train_convoy.py` (major rewrite)
- `roadsense-v2v/ml/run_docker.sh` (pass args to train)

**Files to Create:**
- `roadsense-v2v/ml/tests/unit/test_train_convoy.py`

**Key CLI Arguments:**
- `--dataset_dir` - Path to dataset (train uses random, eval uses sequential)
- `--eval_episodes` - Number of evaluation episodes (default: 10)
- `--seed` - Random seed for reproducibility
- `--total_timesteps` - Training duration
- `--output_dir` / `--run_id` - Output location

**Exit Criteria:**
- CLI parsing works for all arguments
- Training uses randomized scenario selection
- Evaluation uses sequential scenario selection
- Metrics JSON saved with training + eval results
- Backward compatible with existing usage

## Open Risks

| Risk | Mitigation | Status |
|------|------------|--------|
| V001 spawn timing inconsistency | Validate V001 depart=0, sort routes by depart time, and wait for spawn in ConvoyEnv reset | RESOLVED (Phase 1) |
| SUMO reload safety (port leaks) | stop/start pattern, not traci.load | RESOLVED (Phase 2) |
| Dataset reproducibility | Manifest schema with hashes | RESOLVED (Phase 1) |
| Evaluation randomization | Fixed eval scenario list | RESOLVED (Phase 1) |
| SUMO stop/start overhead | ~5-10% episode time; can optimize later | Monitor in Phase 4 |

## Notes
- This tracker is for automation prep only; cloud execution is secondary.
- Do not implement padding or ordering of peers; keep Deep Sets invariants intact.
- Phase 1 SPEC is ready for Builder implementation.
