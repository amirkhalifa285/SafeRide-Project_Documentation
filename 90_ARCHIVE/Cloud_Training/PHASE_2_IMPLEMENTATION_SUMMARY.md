# Phase 2 Implementation Summary: Scenario Selection + SUMO Reload

**Date:** 2026-01-15
**Builder:** Codex CLI (BUILDER role)
**Scope:** Phase 2 of Cloud Training Automation

## Objectives Completed

- Implement dataset-driven scenario selection for ConvoyEnv.
- Add safe SUMO reload flow (stop -> set_config -> start).
- Provide deterministic selection for train/eval modes.
- Maintain backward compatibility with single `sumo_cfg`.
- Add unit + integration tests for scenario switching and leak checks.

## Key Changes

### Scenario Manager

- Added `ml/envs/scenario_manager.py` implementation.
- Loads `manifest.json`, validates required fields, stores scenario IDs.
- Derives scenario paths from IDs with train/eval prefix logic.
- Train mode: random selection with replacement (seeded RNG).
- Eval mode: sequential selection with wrap-around.

### ConvoyEnv Updates

- Constructor supports `dataset_dir`, `scenario_mode`, `scenario_seed`.
- Enforces mutual exclusivity of `sumo_cfg` and `dataset_dir`.
- Defers scenario selection until `reset()` (no selection in `__init__`).
- `reset()` updates SUMO config per selected scenario.
- `reset()` info includes `scenario_id`.
- Added convenience methods: `set_eval_mode()`, `set_train_mode()`, `get_scenario_count()`.

### SUMOConnection Updates

- Added `set_config()` to update scenario path for next `start()`.

### Tests Added

- `ml/tests/unit/test_scenario_manager.py` for ScenarioManager behaviors.
- `ml/tests/unit/test_convoy_env.py` additions for dataset mode.
- `ml/tests/integration/test_scenario_switching.py` integration test for leak checks.
- `ml/tests/integration/conftest.py` dataset fixture for integration tests.

## Tests Executed

### Unit Suite (Docker)

Command:
```
ml/run_docker.sh test
```
Result:
- 101 tests passed.
- Warning: Gymnasium registry override (existing behavior).

### Integration Suite (Docker)

Command:
```
ml/run_docker.sh integration
```
Result:
- 11 tests passed.
- Warnings: Gymnasium registry override, observation shape note from SB3 checker.

## Backward Compatibility

- Existing usage with `sumo_cfg=".../scenario.sumocfg"` remains unchanged.
- Dataset mode is opt-in via `dataset_dir`.

## Notes

- Scenario selection is deferred to `reset()` to avoid skipping eval index 0.
- Integration test uses Linux-only `pgrep` and is skipped on non-Linux hosts.
