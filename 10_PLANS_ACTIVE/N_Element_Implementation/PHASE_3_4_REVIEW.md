# Phase 3 & 4 Review: Causality Fix & Training Pipeline

**Date:** 2026-01-15
**Agent Role:** REVIEWER
**Spec:** `docs/10_PLANS_ACTIVE/N_Element_Implementation/PHASE_3_4_SPEC.md`
**Builder Report:** `docs/10_PLANS_ACTIVE/N_Element_Implementation/PHASE_3_4_BUILDER_REPORT.md`

---

## Verdict: APPROVED (after fixes)

---

## Spec Compliance

- [x] `sync_and_get_messages()` added to emulator — returns `Dict[str, ReceivedMessage]`
- [x] `ConvoyEnv._step_espnow` refactored: transmit phase → receive phase → build observation
- [x] `deep_set_policy.py` created with `create_deep_set_policy_kwargs()` helper
- [x] `policies/__init__.py` properly exports the function
- [x] `train_convoy.py` created with SB3 PPO + MultiInputPolicy + DeepSetExtractor config
- [x] Integration test `test_training_pipeline.py` created with SUMO skip marker
- [x] Unit test mocks updated to support `sync_and_get_messages`
- [ ] **Missing:** Explicit causality unit test (spec acceptance criteria item 1)
- [ ] **Missing:** `ml/training/__init__.py` (module not properly packaged)

---

## Technical Review

### Memory: PASS
- No unbounded allocations
- `last_received.copy()` returns shallow copy — acceptable for dict of frozen dataclasses
- Fixed buffer sizes in ObservationBuilder (MAX_PEERS=8)

### Timing: PASS
- Causality logic is correct: messages queued with `arrival_time = current_time_ms + latency`
- `sync_and_get_messages()` correctly drains queue only for messages where `arrival_time <= current_time_ms`

### Safety: PASS (with note)
- Age calculation verified correct:
  ```python
  age_ms = current_time_ms - received.received_at_ms + received.age_ms
  ```
  - At `current_time_ms=1030`, for message sent at T=1000 with latency=20ms:
  - `received_at_ms = 1020`, `received.age_ms = 20`
  - `age_ms = 1030 - 1020 + 20 = 30ms` (correct total age since send)

- Staleness threshold sourcing is correct:
  ```python
  staleness_threshold = params.get("observation", {}).get(
      "staleness_threshold_ms", self.obs_builder.STALENESS_THRESHOLD
  )
  ```
  Falls back to ObservationBuilder default (500ms) if emulator params unavailable.

### Architecture: PASS
- Maintains separation of concerns: emulator handles network effects, env handles formatting
- `sync_and_get_messages()` correctly separates "receive" from "format" per spec design decision
- Deep Sets integration follows SB3 patterns correctly

---

## Issues Found

### Critical (Blocking Merge)

1. **[ml/training/__init__.py:MISSING]** — Module packaging incomplete
   - `ml/training/` directory lacks `__init__.py`
   - Will cause `ModuleNotFoundError` when running `python -m ml.training.train_convoy`
   - **Impact:** Training script may not be importable as module

2. **[test_training_pipeline.py:14]** — Non-Pythonic SUMO availability check
   ```python
   SUMO_AVAILABLE = os.system("sumo --version > /dev/null 2>&1") == 0
   ```
   - `os.system()` spawns shell, less reliable than `shutil.which()`
   - Silent on Windows (no `/dev/null`)
   - **Impact:** Test skip logic may fail on some platforms

### Non-Critical (Should Fix)

3. **[MISSING]** — No explicit causality unit test
   - Spec Acceptance Criteria #1: "ConvoyEnv correctly delays message perception (causality test)"
   - Current unit tests mock `sync_and_get_messages` but don't validate actual delay behavior
   - **Recommendation:** Add test that:
     1. Transmits at T=100 with 50ms latency
     2. Calls `sync_and_get_messages(120)` — should return empty
     3. Calls `sync_and_get_messages(150)` — should return message
   - **Impact:** Causality regression could go undetected

4. **[deep_set_policy.py:7]** — Absolute import path assumption
   ```python
   from ml.models.deep_set_extractor import DeepSetExtractor
   ```
   - Requires `ml` to be in PYTHONPATH
   - Other files use relative imports (`from envs.convoy_env`)
   - **Impact:** Import may fail if run from wrong directory
   - **Recommendation:** Either use relative import or ensure consistent packaging

5. **[train_convoy.py:22]** — No existence check for scenario file
   ```python
   scenario_path = os.path.join(base_dir, "scenarios", "base", "scenario.sumocfg")
   ```
   - If file missing, error occurs deep in gym.make(), hard to diagnose
   - **Recommendation:** Add early check with clear error message

---

## Required Changes (for APPROVED)

- [ ] Create `ml/training/__init__.py` (can be empty or with `__all__`)
- [ ] Replace `os.system()` SUMO check with `shutil.which("sumo") is not None`

---

## Recommended Changes (Optional but Advised)

- [ ] Add explicit causality unit test to `test_emulator.py` or new `test_causality.py`
- [ ] Add scenario file existence check in `train_convoy.py` with helpful error
- [ ] Consider relative import in `deep_set_policy.py`:
  ```python
  from ..models.deep_set_extractor import DeepSetExtractor
  ```

---

## Code Quality Assessment

### Positives
- Clean separation between transmit/receive phases in `_step_espnow`
- Proper use of `getattr()` with fallback for emulator params
- Integration test correctly marked with `@pytest.mark.integration` and `@pytest.mark.slow`
- Mock emulator in unit tests correctly simulates new API

### Minor Style Notes
- `train_convoy.py` cleanly handles resource cleanup with `try/finally`
- Constants properly defined at module level (`TOTAL_TIMESTEPS`, etc.)
- Type hints consistent throughout

---

## Notes for PLANNER

1. **Phase 5 consideration:** When adding real SUMO scenarios, ensure `ml/scenarios/base/scenario.sumocfg` exists or update default path logic.

2. **Future TFLite export:** The `DeepSetExtractor` uses `torch.Tensor` operations (`masked_fill`, `max`). Verify these ops have ONNX/TFLite equivalents before planning model conversion phase.

3. **Observation `get_observation()` vs `sync_and_get_messages()` parity:** The old `get_observation()` method still exists in the emulator. Consider deprecating or removing it to prevent confusion. Both methods now process the pending queue identically.

---

## Verification Commands

After BUILDER fixes:

```bash
# Verify module imports
cd roadsense-v2v/ml && python -c "from training.train_convoy import train; print('OK')"

# Run unit tests
cd roadsense-v2v && pytest ml/tests/unit/test_n_element_env.py ml/tests/unit/test_convoy_env.py -v

# Run integration test (requires SUMO)
cd roadsense-v2v && pytest -m integration ml/tests/integration/test_training_pipeline.py -v
```

---

## Summary

The implementation is architecturally sound and correctly addresses the causality bug. The core logic in `sync_and_get_messages()` and the refactored `_step_espnow()` are correct. However, two packaging/portability issues must be fixed before merge:

1. Missing `__init__.py` in training module
2. Platform-dependent SUMO check

Once these are addressed, the code is ready for commit.

---

---

## Post-Review Fixes (2026-01-15)

BUILDER addressed all required changes:

- [x] **SUMO check:** Replaced `os.system()` with `SUMO_AVAILABLE` environment variable (Docker-compatible)
- [x] **Module packaging:** Added `ml/training/__init__.py` for conventional imports
- [x] **Causality test:** Added `sync_and_get_messages` test case to existing `test_causality_fix.py`

**Final Verdict:** APPROVED — Ready for commit.

---

**Reviewer:** REVIEWER Agent
**Approved:** 2026-01-15
