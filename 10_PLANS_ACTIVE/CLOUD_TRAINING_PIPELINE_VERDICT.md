# Cloud Training Pipeline Verdict

Date: 2026-01-15
Role: Master Architect (Planner)
Status: APPROVE WITH CONDITIONS

## Decision Summary
- Approve the cloud training direction (EC2 first, SageMaker later) because it aligns with the single-container ML design and avoids TraCI network overhead.
- Block execution until scenario randomization and SUMO reload behavior are hardened to avoid breaking V001 spawn assumptions and determinism.

## Sources Reviewed
- AGENT_ROLES/PLANNER.md
- docs/10_PLANS_ACTIVE/CLOUD_TRAINING_PIPELINE_PROPOSAL.md
- docs/_MAPS/INDEX.md
- docs/20_KNOWLEDGE_BASE/ML_AUGMENTATION_PIPELINE.md
- docs/20_KNOWLEDGE_BASE/ML_DOCKER_ENVIRONMENT.md
- docs/10_PLANS_ACTIVE/N_Element_Implementation/PHASE_3_4_SPEC.md
- docs/10_PLANS_ACTIVE/N_Element_Implementation/PHASE_3_4_REVIEW.md
- docs/00_ARCHITECTURE/DEEP_SETS_N_ELEMENT_ARCHITECTURE.md
- docs/00_ARCHITECTURE/ESPNOW_EMULATOR_DESIGN.md
- Code: roadsense-v2v/ml/envs/convoy_env.py
- Code: roadsense-v2v/ml/envs/sumo_connection.py
- Code: roadsense-v2v/ml/training/train_convoy.py
- Code: roadsense-v2v/ml/scenarios/*

## Alignment Check
- N-element Deep Sets requirement: training must keep variable peer counts and permutation invariance.
- Sim2Real principle: use measured ESP-NOW params for emulator and log their versions with each run.
- Single-container training: scenario generation and training must run inside the same Docker image.

## Blocking Conditions (Must Resolve Before Cloud Execution)
1. V001 spawn invariants
   - Current ConvoyEnv.reset expects V001 to exist at time 0.
   - Scenario generation must guarantee V001 spawn at t=0, or ConvoyEnv must wait for V001.
2. SUMO reload semantics
   - Reload must fully reset TraCI state and preserve step length 0.1.
   - Prevent port leaks and orphaned SUMO processes; stop must precede start or use traci.load safely.
3. Dataset governance
   - Generated scenarios must live outside git (S3 or ignored local path).
   - Every dataset requires a manifest: seed, parameter ranges, base scenario hash,
     git commit, emulator params version, and SUMO version.
4. Train/eval split discipline
   - Training uses randomized scenario selection.
   - Evaluation must use a fixed, versioned scenario list (no randomization).
5. Variable-N coverage
   - Scenario generation must include 0-8+ peers to exercise Deep Sets behavior.
   - Avoid any padding, sorting, or hardcoded peer slots in training logic.

## Non-Blocking Recommendations
- Create a ScenarioManager module (ml/envs/scenario_manager.py) to centralize selection,
  seeding, and manifest loading.
- Add CLI flags to train_convoy.py: --dataset_dir, --eval_dir, --seed, --emulator_params.
- Add a lightweight dataset sanity check (missing files, V001 presence, valid sumocfg).
- Use cost guardrails on AWS (auto-stop, budget alerts, spot for non-critical runs).
- Update ML_AUGMENTATION_PIPELINE.md to clarify the online-simulation path vs offline FCD.

## Cloud Path (Phased)
Phase 0 (Local Docker validation)
- Implement generator and dataset manifest.
- Run short training locally with random scenario selection.

Phase 1 (EC2)
- Use current Docker image from ml/Dockerfile.
- Store datasets on S3; sync to EBS/EFS on instance.
- Run training via ./run_docker.sh train with mounted dataset dir.

Phase 2 (SageMaker)
- Push Docker image to ECR.
- Mount S3 dataset and write model artifacts back to S3.
- Enable hyperparameter sweeps only after Phase 1 shows stable metrics.

## Final Verdict
Proceed with the proposal after resolving the blocking conditions above.
This path is the right accelerator, but only if it preserves the N-element
architecture guarantees and the Sim2Real contract.
