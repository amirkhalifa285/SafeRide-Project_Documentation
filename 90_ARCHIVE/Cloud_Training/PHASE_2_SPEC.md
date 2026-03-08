# Phase 2 SPEC: Scenario Selection + SUMO Reload

**Document Type:** Builder Specification
**Phase:** 2 of 5 (Cloud Training Automation)
**Status:** Ready for Implementation
**Created:** January 15, 2026
**Author:** PLANNER Agent

---

## Objective

Enable `ConvoyEnv` to select a random scenario from a dataset directory on each episode reset, with safe SUMO process management to prevent resource leaks.

---

## Prerequisites

- [x] Phase 1 complete (scenario generator + manifest)
- [x] Dataset directory structure exists: `ml/scenarios/datasets/{dataset_id}/train/`, `eval/`
- [x] Manifest schema finalized with `train_scenarios` and `eval_scenarios` lists

---

## Implementation Steps

### Step 1: Implement ScenarioManager

**File:** `roadsense-v2v/ml/envs/scenario_manager.py`

Replace the placeholder with a full implementation:

```python
"""
Scenario management for dataset-based training.

Loads manifest from a dataset directory and provides deterministic
scenario selection for training and evaluation.
"""

import json
from pathlib import Path
from typing import List, Optional, Tuple

import numpy as np


class ScenarioManager:
    """
    Manages scenario selection from a dataset directory.

    Attributes:
        dataset_dir: Path to the dataset root directory
        manifest: Loaded manifest dictionary
        train_scenarios: List of training scenario paths
        eval_scenarios: List of evaluation scenario paths
        rng: NumPy random generator for deterministic selection
    """

    def __init__(
        self,
        dataset_dir: str,
        seed: Optional[int] = None,
        mode: str = "train",
    ) -> None:
        """
        Initialize the scenario manager.

        Args:
            dataset_dir: Path to dataset root (contains manifest.json)
            seed: Random seed for reproducible scenario selection
            mode: "train" for randomized selection, "eval" for sequential

        Raises:
            FileNotFoundError: If manifest.json not found
            ValueError: If mode is invalid
        """

    def load_manifest(self, dataset_dir: Path) -> dict:
        """
        Load and validate manifest.json.

        Args:
            dataset_dir: Path to dataset root

        Returns:
            Parsed manifest dictionary

        Raises:
            FileNotFoundError: If manifest.json missing
            json.JSONDecodeError: If manifest is invalid JSON
            KeyError: If required fields missing
        """

    def get_scenario_paths(self, mode: str) -> List[Path]:
        """
        Get list of scenario paths for the given mode.

        Args:
            mode: "train" or "eval"

        Returns:
            List of absolute paths to scenario.sumocfg files
        """

    def select_scenario(self) -> Tuple[Path, str]:
        """
        Select the next scenario based on mode.

        For "train" mode: Random selection (with replacement)
        For "eval" mode: Sequential iteration (cycles after exhaustion)

        Returns:
            Tuple of (path_to_sumocfg, scenario_id)
        """

    def reset_eval_index(self) -> None:
        """Reset the evaluation scenario index to 0."""

    def seed(self, seed: int) -> None:
        """Re-seed the random generator."""
```

**Key behaviors:**

| Mode | Selection Strategy | Use Case |
|------|-------------------|----------|
| `train` | Random with replacement | Training episodes |
| `eval` | Sequential, cyclic | Deterministic evaluation |

### Step 2: Update SUMOConnection for Safe Reload

**File:** `roadsense-v2v/ml/envs/sumo_connection.py`

Add a method to change the scenario configuration:

```python
def set_config(self, sumo_cfg: str) -> None:
    """
    Update the scenario configuration for the next start().

    Args:
        sumo_cfg: Path to the new scenario.sumocfg file

    Note:
        This does NOT restart SUMO. Call stop() then start() to apply.
    """
    self.sumo_cfg = sumo_cfg
```

**IMPORTANT:** Do NOT use `traci.load()`. It is brittle and can cause port conflicts. The safe pattern is:
1. `stop()` - Close TraCI connection
2. `set_config(new_cfg)` - Update config path
3. `start()` - Start fresh SUMO process

### Step 3: Update ConvoyEnv Constructor

**File:** `roadsense-v2v/ml/envs/convoy_env.py`

Modify `__init__` to accept dataset-based configuration:

```python
def __init__(
    self,
    sumo_cfg: Optional[str] = None,           # Single scenario (existing)
    dataset_dir: Optional[str] = None,        # NEW: Dataset directory
    scenario_mode: str = "train",             # NEW: "train" or "eval"
    scenario_seed: Optional[int] = None,      # NEW: Seed for scenario selection
    emulator: Optional[ESPNOWEmulator] = None,
    emulator_params_path: Optional[str] = None,
    max_steps: int = DEFAULT_MAX_STEPS,
    hazard_injection: bool = True,
    render_mode: Optional[str] = None,
    gui: bool = False,
) -> None:
    """
    Initialize ConvoyEnv.

    Args:
        sumo_cfg: Path to single scenario.sumocfg (mutually exclusive with dataset_dir)
        dataset_dir: Path to dataset directory with manifest.json
        scenario_mode: "train" for random selection, "eval" for sequential
        scenario_seed: Seed for reproducible scenario selection
        ... (existing args)

    Raises:
        ValueError: If neither sumo_cfg nor dataset_dir provided
        ValueError: If both sumo_cfg and dataset_dir provided
    """
```

**Validation logic:**

```python
if sumo_cfg is None and dataset_dir is None:
    raise ValueError("Must provide either sumo_cfg or dataset_dir")
if sumo_cfg is not None and dataset_dir is not None:
    raise ValueError("Cannot provide both sumo_cfg and dataset_dir")

if dataset_dir is not None:
    self.scenario_manager = ScenarioManager(
        dataset_dir=dataset_dir,
        seed=scenario_seed,
        mode=scenario_mode,
    )
    # Select initial scenario
    initial_cfg, _ = self.scenario_manager.select_scenario()
    sumo_cfg = str(initial_cfg)
else:
    self.scenario_manager = None
```

### Step 4: Update ConvoyEnv.reset() for Scenario Switching

**File:** `roadsense-v2v/ml/envs/convoy_env.py`

Modify `reset()` to switch scenarios when using a dataset:

```python
def reset(
    self,
    seed: Optional[int] = None,
    options: Optional[Dict[str, Any]] = None,
) -> Tuple[Dict[str, np.ndarray], Dict[str, Any]]:
    super().reset(seed=seed)

    self.emulator.clear()
    self._step_count = 0

    # NEW: Select next scenario if using dataset
    scenario_id = None
    if self.scenario_manager is not None:
        new_cfg, scenario_id = self.scenario_manager.select_scenario()
        self.sumo.set_config(str(new_cfg))

    # Stop previous SUMO instance (if running)
    if self._sumo_started:
        self.sumo.stop()

    # Start SUMO with (potentially new) config
    self.sumo.start()
    self._sumo_started = True

    # ... rest of reset logic unchanged ...

    info = {
        "step": 0,
        "simulation_time": self.sumo.get_simulation_time(),
        "scenario_id": scenario_id,  # NEW: Include in info
    }

    return observation, info
```

### Step 5: Add Convenience Methods

**File:** `roadsense-v2v/ml/envs/convoy_env.py`

Add methods for evaluation control:

```python
def set_eval_mode(self) -> None:
    """
    Switch to evaluation mode (sequential scenario iteration).

    Raises:
        RuntimeError: If not using dataset-based configuration
    """
    if self.scenario_manager is None:
        raise RuntimeError("set_eval_mode() requires dataset_dir configuration")
    self.scenario_manager = ScenarioManager(
        dataset_dir=self._dataset_dir,
        seed=self._scenario_seed,
        mode="eval",
    )


def set_train_mode(self) -> None:
    """
    Switch to training mode (random scenario selection).

    Raises:
        RuntimeError: If not using dataset-based configuration
    """
    if self.scenario_manager is None:
        raise RuntimeError("set_train_mode() requires dataset_dir configuration")
    self.scenario_manager = ScenarioManager(
        dataset_dir=self._dataset_dir,
        seed=self._scenario_seed,
        mode="train",
    )


def get_scenario_count(self) -> Tuple[int, int]:
    """
    Get the number of scenarios in train and eval sets.

    Returns:
        Tuple of (train_count, eval_count)

    Raises:
        RuntimeError: If not using dataset-based configuration
    """
    if self.scenario_manager is None:
        raise RuntimeError("get_scenario_count() requires dataset_dir configuration")
    return (
        len(self.scenario_manager.manifest["train_scenarios"]),
        len(self.scenario_manager.manifest["eval_scenarios"]),
    )
```

### Step 6: Update __init__.py for Registration

**File:** `roadsense-v2v/ml/envs/__init__.py`

Ensure the updated ConvoyEnv is properly exported:

```python
from envs.convoy_env import ConvoyEnv
from envs.scenario_manager import ScenarioManager

__all__ = ["ConvoyEnv", "ScenarioManager"]
```

---

## Exit Criteria

- [ ] `ScenarioManager` loads manifest and selects scenarios
- [ ] `ConvoyEnv` accepts `dataset_dir` parameter
- [ ] Episodes switch scenarios without SUMO process leaks
- [ ] Training mode uses random selection
- [ ] Eval mode uses sequential selection
- [ ] `reset()` returns `scenario_id` in info dict
- [ ] All existing tests pass (backward compatible with `sumo_cfg`)
- [ ] New unit tests pass: `pytest ml/tests/unit/test_scenario_manager.py -v`
- [ ] Integration test: 10 resets without port conflicts or zombie processes

---

## Files Touched

| File | Action | Description |
|------|--------|-------------|
| `roadsense-v2v/ml/envs/scenario_manager.py` | REPLACE | Full implementation (was placeholder) |
| `roadsense-v2v/ml/envs/sumo_connection.py` | EDIT | Add `set_config()` method |
| `roadsense-v2v/ml/envs/convoy_env.py` | EDIT | Add dataset support, scenario switching |
| `roadsense-v2v/ml/envs/__init__.py` | EDIT | Export ScenarioManager |
| `roadsense-v2v/ml/tests/unit/test_scenario_manager.py` | CREATE | Unit tests |
| `roadsense-v2v/ml/tests/unit/test_convoy_env.py` | EDIT | Add dataset mode tests |

---

## Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     ConvoyEnv Reset with Dataset Mode                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ConvoyEnv.__init__(dataset_dir="ml/scenarios/datasets/v1")                │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────┐                                                       │
│  │ ScenarioManager │◄─── manifest.json                                     │
│  │   mode="train"  │     ├── train_scenarios: [train_000, train_001, ...]  │
│  │   seed=42       │     └── eval_scenarios: [eval_000, eval_001, ...]     │
│  └────────┬────────┘                                                       │
│           │                                                                 │
│           ▼                                                                 │
│  ConvoyEnv.reset()                                                         │
│       │                                                                     │
│       ├──► ScenarioManager.select_scenario()                               │
│       │         │                                                           │
│       │         ▼                                                           │
│       │    ┌──────────────────────────────────────────┐                    │
│       │    │ mode="train": rng.choice(train_scenarios)│                    │
│       │    │ mode="eval":  eval_scenarios[index++]    │                    │
│       │    └──────────────────────────────────────────┘                    │
│       │         │                                                           │
│       │         ▼                                                           │
│       │    Returns: ("datasets/v1/train/train_003/scenario.sumocfg",       │
│       │              "train_003")                                           │
│       │                                                                     │
│       ├──► SUMOConnection.set_config(new_cfg)                              │
│       │                                                                     │
│       ├──► SUMOConnection.stop()    ◄─── Close previous TraCI              │
│       │                                                                     │
│       └──► SUMOConnection.start()   ◄─── Fresh SUMO process                │
│                                                                             │
│  info = {"scenario_id": "train_003", ...}                                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Backward Compatibility

The existing `sumo_cfg` parameter remains supported. These two usages are equivalent:

**Old (single scenario):**
```python
env = ConvoyEnv(sumo_cfg="ml/scenarios/base/scenario.sumocfg")
```

**New (dataset-based):**
```python
env = ConvoyEnv(
    dataset_dir="ml/scenarios/datasets/v1",
    scenario_mode="train",
    scenario_seed=42,
)
```

All existing tests using `sumo_cfg` must continue to pass unchanged.

---

## Risks / Notes

### Risk 1: Port Conflicts on Rapid Reset

**Problem:** If SUMO doesn't release its port before the next `start()`, TraCI will fail.

**Mitigation:** The `stop()` → `start()` pattern (not `traci.load()`) ensures clean process termination. TraCI's default behavior releases ports on `close()`.

**Fallback:** If issues persist, add a small delay or implement port pooling:
```python
import time

def stop(self) -> None:
    traci.close()
    time.sleep(0.1)  # Allow OS to release port
```

### Risk 2: Zombie SUMO Processes

**Problem:** If `reset()` raises an exception after `stop()` but before `start()`, no cleanup occurs.

**Mitigation:** Use try/finally in `reset()`:
```python
try:
    if self._sumo_started:
        self.sumo.stop()
        self._sumo_started = False
    self.sumo.start()
    self._sumo_started = True
except Exception:
    self._sumo_started = False
    raise
```

### Risk 3: Eval Mode Exhaustion

**Problem:** In eval mode, iterating past the last scenario could cause issues.

**Mitigation:** Cyclic iteration (wrap around to index 0):
```python
def select_scenario(self) -> Tuple[Path, str]:
    if self.mode == "eval":
        scenario_id = self.eval_scenarios[self._eval_index]
        self._eval_index = (self._eval_index + 1) % len(self.eval_scenarios)
        return self._get_scenario_path(scenario_id), scenario_id
```

### Risk 4: Dataset Not Found

**Problem:** If dataset_dir is invalid, ConvoyEnv construction fails silently.

**Mitigation:** Fail fast with clear error:
```python
def __init__(self, dataset_dir: str, ...):
    manifest_path = Path(dataset_dir) / "manifest.json"
    if not manifest_path.exists():
        raise FileNotFoundError(
            f"manifest.json not found in {dataset_dir}. "
            f"Generate a dataset first: ./run_docker.sh generate ..."
        )
```

---

## Test Plan

### Unit Tests (`test_scenario_manager.py`)

```python
class TestScenarioManager:
    """Unit tests for ScenarioManager."""

    def test_load_manifest_success(self, tmp_path):
        """Verify manifest loading works."""

    def test_load_manifest_missing(self, tmp_path):
        """Verify FileNotFoundError when manifest missing."""

    def test_train_mode_random_selection(self, tmp_path):
        """Verify train mode uses random selection."""

    def test_train_mode_deterministic_with_seed(self, tmp_path):
        """Same seed produces same sequence."""

    def test_eval_mode_sequential(self, tmp_path):
        """Verify eval mode iterates sequentially."""

    def test_eval_mode_cycles(self, tmp_path):
        """Verify eval mode wraps around after exhaustion."""

    def test_get_scenario_paths(self, tmp_path):
        """Verify correct paths are returned."""
```

### Unit Tests (`test_convoy_env.py` additions)

```python
class TestConvoyEnvDataset:
    """Tests for dataset-based ConvoyEnv."""

    def test_init_with_dataset_dir(self, mock_dataset):
        """Verify construction with dataset_dir."""

    def test_init_mutual_exclusion(self):
        """Verify error when both sumo_cfg and dataset_dir provided."""

    def test_reset_switches_scenario(self, mock_dataset, mock_sumo):
        """Verify reset() changes scenario."""

    def test_reset_returns_scenario_id(self, mock_dataset, mock_sumo):
        """Verify info dict contains scenario_id."""

    def test_backward_compatible_sumo_cfg(self, mock_sumo):
        """Verify existing sumo_cfg usage still works."""
```

### Integration Test

```python
@pytest.mark.integration
def test_multiple_resets_no_leaks(dataset_dir):
    """
    Verify 10 consecutive resets don't leak SUMO processes.

    Checks:
    - No zombie sumo/sumo-gui processes
    - No port conflicts
    - Each reset gets a valid observation
    """
    env = ConvoyEnv(dataset_dir=dataset_dir, scenario_mode="train")

    for i in range(10):
        obs, info = env.reset()
        assert "scenario_id" in info
        assert obs["ego"].shape == (4,)

    env.close()

    # Verify no zombie processes
    import subprocess
    result = subprocess.run(["pgrep", "-c", "sumo"], capture_output=True)
    assert result.returncode != 0  # No sumo processes running
```

---

## Reference: Manifest Schema (from Phase 1)

```json
{
  "dataset_id": "dataset_v1",
  "train_scenarios": ["train_000", "train_001", "train_002", ...],
  "eval_scenarios": ["eval_000", "eval_001", ...],
  ...
}
```

The `ScenarioManager` reads `train_scenarios` and `eval_scenarios` to build the scenario lists.

---

## Acceptance Checklist (For REVIEWER)

- [ ] `ScenarioManager` correctly loads manifest
- [ ] Train mode is random, eval mode is sequential
- [ ] Determinism verified: same seed produces same scenario sequence
- [ ] `SUMOConnection.set_config()` implemented
- [ ] `ConvoyEnv` validates mutually exclusive parameters
- [ ] No SUMO process leaks after 10+ resets
- [ ] Backward compatibility: existing `sumo_cfg` usage unchanged
- [ ] All existing tests pass
- [ ] New unit tests cover ScenarioManager
- [ ] Integration test verifies no resource leaks

---

**END OF PHASE 2 SPEC**
