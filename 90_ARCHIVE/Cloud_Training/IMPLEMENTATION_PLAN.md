# Cloud Training Automation Implementation Plan (Push-Button)

Status: Proposed
Scope: Local one-command pipeline with cloud-ready artifacts.

## Phase 0: Readiness and Invariants

### Objective
Lock architecture assumptions and define invariants needed for automation.

### Prerequisites
- docs/00_ARCHITECTURE/DEEP_SETS_N_ELEMENT_ARCHITECTURE.md reviewed.
- docs/10_PLANS_ACTIVE/CLOUD_TRAINING_PIPELINE_VERDICT.md accepted.

### Implementation Steps
1. Document invariants in this plan:
   - V001 must exist at t=0 or env must wait for it.
   - No peer padding, sorting, or slotting.
   - Fixed eval set, randomized train set.
2. Add dataset manifest schema (JSON) in this plan to enforce reproducibility.

### Exit Criteria
- Invariants agreed and referenced by all later phases.
- Manifest schema captured and used by generator and runner.

### Files Touched
- docs/10_PLANS_ACTIVE/CLOUD_TRAINING_AUTOMATION/IMPLEMENTATION_PLAN.md

### Risks / Notes
- If V001 spawn is not guaranteed, training may intermittently fail at reset.

## Phase 1: Scenario Generator + Manifest

### Objective
Generate augmented SUMO scenarios with a reproducible manifest.

### Prerequisites
- Base scenarios exist under `roadsense-v2v/ml/scenarios/base/`.
- SUMO and sumolib available in Docker (see `roadsense-v2v/ml/Dockerfile`).

### Implementation Steps
1. Create `roadsense-v2v/ml/scripts/gen_scenarios.py`:
   - Input: base scenario directory, output dataset directory, seed.
   - Variations: traffic density, driver sigma, speedFactor, spawn timing.
2. Define dataset layout:
   - `ml/scenarios/datasets/<dataset_id>/train/...`
   - `ml/scenarios/datasets/<dataset_id>/eval/...`
3. Write `manifest.json` in dataset root:
   - dataset_id, seed, ranges, base scenario hash
   - git commit, emulator params path + hash
   - SUMO version, container image tag
4. Ensure generated datasets are git-ignored:
   - Update `roadsense-v2v/.gitignore` or `roadsense-v2v/ml/.gitignore`.

### Exit Criteria
- Generator creates train/eval splits with a manifest.
- V001 spawn verified in all generated scenarios (sanity check).

### Files Touched
- roadsense-v2v/ml/scripts/gen_scenarios.py
- roadsense-v2v/.gitignore (or roadsense-v2v/ml/.gitignore)
- roadsense-v2v/ml/scenarios/datasets/ (generated, ignored)

### Risks / Notes
- If `vehicles.rou.xml` generation allows late ego spawn, ConvoyEnv reset will fail.
- Track SUMO version drift between local and cloud runs via manifest.

## Phase 2: Scenario Selection + SUMO Reload

### Objective
Allow ConvoyEnv to select a random scenario per episode from a directory.

### Prerequisites
- Phase 1 manifest format finalized.

### Implementation Steps
1. Extend `roadsense-v2v/ml/envs/scenario_manager.py`:
   - Load manifest and scenario list from dataset dir.
   - Provide deterministic selection via seed.
2. Update `roadsense-v2v/ml/envs/convoy_env.py`:
   - Accept `scenario_list` or `dataset_dir`.
   - On reset, pick a scenario and reload SUMO.
3. Update `roadsense-v2v/ml/envs/sumo_connection.py`:
   - Add safe reload behavior (stop/start or traci.load).
   - Preserve `--step-length 0.1`.

### Exit Criteria
- Episodes switch scenarios without leaking SUMO processes.
- Reset succeeds for all scenarios in a dataset.

### Files Touched
- roadsense-v2v/ml/envs/scenario_manager.py
- roadsense-v2v/ml/envs/convoy_env.py
- roadsense-v2v/ml/envs/sumo_connection.py

### Risks / Notes
- TraCI reload behavior is brittle; prefer full stop/start if instability appears.
- Reuse of ports can cause intermittent failures if stop is skipped.

## Phase 3: Training CLI + Evaluation Harness

### Objective
Expose dataset selection and evaluation via CLI.

### Prerequisites
- Phase 2 scenario selection.

### Implementation Steps
1. Update `roadsense-v2v/ml/training/train_convoy.py`:
   - Add argparse: `--dataset_dir`, `--eval_dir`, `--seed`, `--emulator_params`.
   - Randomized scenario selection for train; fixed list for eval.
2. Add evaluation loop that runs after training:
   - Store metrics: avg reward, collisions, truncations.
3. Extend integration tests:
   - Update `roadsense-v2v/ml/tests/integration/test_training_pipeline.py`
     to support dataset_dir and skip if SUMO missing.

### Exit Criteria
- Training runs on a dataset directory with deterministic seed.
- Evaluation produces metrics stored with the model artifacts.

### Files Touched
- roadsense-v2v/ml/training/train_convoy.py
- roadsense-v2v/ml/tests/integration/test_training_pipeline.py

### Risks / Notes
- Ensure evaluation uses a fixed scenario list to prevent metric drift.

## Phase 4: One-Button Runner + Artifact Packaging

### Objective
Create a single entrypoint that runs generate -> train -> eval -> package.

### Prerequisites
- Phases 1-3 complete.

### Implementation Steps
1. Add `roadsense-v2v/ml/scripts/run_pipeline.py`:
   - Steps: generate dataset, sanity check, train, eval, package.
   - Inputs: run_id, seed, base_scenarios, emulator_params.
2. Define artifact structure:
   - `ml/models/runs/<run_id>/`
   - `model.zip`, `metrics.json`, `manifest.json`, `config.json`.
3. Add a "dry-run" mode to verify data paths without training.

### Exit Criteria
- Single command runs the full pipeline with reproducible artifacts.
- Artifacts contain both model and metadata.

### Files Touched
- roadsense-v2v/ml/scripts/run_pipeline.py
- roadsense-v2v/ml/models/runs/ (generated)

### Risks / Notes
- Ensure artifacts are small enough for upload to S3 later.

## Phase 5: Cloud Handoff (Optional)

### Objective
Make the pipeline portable to EC2/SageMaker without logic changes.

### Prerequisites
- Phase 4 complete.

### Implementation Steps
1. Add a minimal wrapper in `roadsense-v2v/ml/run_docker.sh`:
   - New subcommand: `pipeline` to run `run_pipeline.py`.
2. Document AWS run steps:
   - Use ECR image, S3 dataset, and output artifacts to S3.

### Exit Criteria
- The pipeline can be executed unchanged inside Docker on EC2.

### Files Touched
- roadsense-v2v/ml/run_docker.sh
- docs/20_KNOWLEDGE_BASE/ML_DOCKER_ENVIRONMENT.md (notes)

### Risks / Notes
- Budget guardrails needed if automated training is scheduled in AWS.

## Manifest Schema (Draft)

```
{
  "dataset_id": "dataset_v1_2026_01_15",
  "seed": 1234,
  "base_scenarios": ["base/scenario.sumocfg"],
  "ranges": {
    "speedFactor": [0.8, 1.2],
    "sigma": [0.0, 0.5],
    "vehsPerHour": [0.7, 1.3],
    "spawn_jitter_s": [0.0, 2.0]
  },
  "sumo_version": "1.25.0",
  "git_commit": "<hash>",
  "emulator_params": "ml/espnow_emulator/emulator_params_5m.json",
  "emulator_params_hash": "<hash>"
}
```

## Files To Inspect When Implementing
- roadsense-v2v/ml/envs/convoy_env.py
- roadsense-v2v/ml/envs/sumo_connection.py
- roadsense-v2v/ml/envs/scenario_manager.py
- roadsense-v2v/ml/training/train_convoy.py
- roadsense-v2v/ml/scripts/
- docs/20_KNOWLEDGE_BASE/ML_DOCKER_ENVIRONMENT.md
- docs/20_KNOWLEDGE_BASE/ML_AUGMENTATION_PIPELINE.md
- docs/00_ARCHITECTURE/DEEP_SETS_N_ELEMENT_ARCHITECTURE.md
- docs/_MAPS/INDEX.md
