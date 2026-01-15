# Cloud Training Automation Progress Tracker

Status: Active
Owner: Planner
Goal: One-button training pipeline (generate -> augment -> train -> eval -> save artifacts)

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
- IN PROGRESS
- BLOCKED
- COMPLETE

## Phase Status
1. Phase 0 - Readiness and invariants: NOT STARTED
2. Phase 1 - Scenario generator + manifest: NOT STARTED
3. Phase 2 - Scenario selection + SUMO reload: NOT STARTED
4. Phase 3 - Training CLI + evaluation harness: NOT STARTED
5. Phase 4 - One-button runner + artifact packaging: NOT STARTED
6. Phase 5 - Cloud handoff (optional): NOT STARTED

## Open Risks
- V001 spawn timing consistency across generated scenarios.
- SUMO reload safety (port leaks, stale TraCI).
- Dataset reproducibility without a manifest.
- Evaluation integrity if eval data is randomized.

## Notes
- This tracker is for automation prep only; cloud execution is secondary.
- Do not implement padding or ordering of peers; keep Deep Sets invariants intact.
