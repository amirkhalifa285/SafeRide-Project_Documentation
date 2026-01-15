# Phase 1 SPEC: Scenario Generator + Manifest

**Document Type:** Builder Specification
**Phase:** 1 of 5 (Cloud Training Automation)
**Status:** Ready for Implementation
**Created:** January 15, 2026
**Author:** PLANNER Agent

---

## Objective

Create `gen_scenarios.py` that generates augmented SUMO scenarios from base templates, producing reproducible train/eval splits with a manifest file for tracking lineage.

---

## Prerequisites

- [x] Base scenarios exist under `roadsense-v2v/ml/scenarios/base/`
- [x] SUMO 1.25.0+ available in Docker (`roadsense-v2v/ml/Dockerfile`)
- [x] `sumolib` available in requirements.txt
- [x] Phase 0 invariants understood (V001 must exist at t=0)

---

## Implementation Steps

### Step 1: Create the Generator Script

**File:** `roadsense-v2v/ml/scripts/gen_scenarios.py`

```python
#!/usr/bin/env python3
"""
Scenario Generator for Cloud Training Automation.

Generates augmented SUMO scenarios from base templates with reproducible
randomization and train/eval splits.

Usage:
    python -m ml.scripts.gen_scenarios \
        --base_dir ml/scenarios/base \
        --output_dir ml/scenarios/datasets/dataset_v1 \
        --seed 42 \
        --train_count 20 \
        --eval_count 5
"""
```

**Required Arguments:**
| Argument | Type | Description |
|----------|------|-------------|
| `--base_dir` | Path | Base scenario directory (must contain `scenario.sumocfg`) |
| `--output_dir` | Path | Output dataset directory |
| `--seed` | int | Master random seed for reproducibility |
| `--train_count` | int | Number of training scenarios to generate |
| `--eval_count` | int | Number of evaluation scenarios to generate |

**Optional Arguments:**
| Argument | Type | Default | Description |
|----------|------|---------|-------------|
| `--emulator_params` | Path | `ml/espnow_emulator/emulator_params_5m.json` | Emulator params to reference |

### Step 2: Define Augmentation Parameters

The generator must vary the following parameters within these ranges:

| Parameter | XML Attribute | Range | Distribution |
|-----------|---------------|-------|--------------|
| `speedFactor` | `<vType speedFactor="...">` | [0.8, 1.2] | Uniform |
| `sigma` | `<vType sigma="...">` | [0.0, 0.5] | Uniform |
| `decel` | `<vType decel="...">` | [3.5, 5.5] m/s² | Uniform |
| `tau` | `<vType tau="...">` | [0.5, 1.5] s | Uniform |
| `vehsPerHour` | (spawn timing multiplier) | [0.7, 1.3] | Uniform |
| `spawn_jitter_s` | `<vehicle depart="...">` | [0.0, 2.0] s | Uniform |

**CRITICAL INVARIANT:** V001 `depart` time must remain `0` or the generator must ensure V001 exists before any `traci.vehicle.getIDList()` call in ConvoyEnv. The safest approach: **V001 always departs at t=0, only V002+ get jittered spawn times.**

### Step 3: Implement XML Modification Logic

**Function signatures:**

```python
def load_base_scenario(base_dir: Path) -> Tuple[ET.ElementTree, ET.ElementTree, ET.ElementTree]:
    """
    Load base scenario XML files.

    Args:
        base_dir: Path to base scenario directory

    Returns:
        Tuple of (sumocfg_tree, network_tree, routes_tree)

    Files read:
        - {base_dir}/scenario.sumocfg
        - {base_dir}/network.net.xml
        - {base_dir}/vehicles.rou.xml
    """


def augment_routes(
    routes_tree: ET.ElementTree,
    rng: np.random.Generator,
    params: Dict[str, Tuple[float, float]]
) -> ET.ElementTree:
    """
    Apply augmentation to vehicles.rou.xml.

    Args:
        routes_tree: Parsed XML tree
        rng: NumPy random generator (seeded)
        params: Dict of {param_name: (min, max)} ranges

    Returns:
        Modified XML tree

    Modifications:
        1. Modify <vType> attributes: speedFactor, sigma, decel, tau
        2. Jitter <vehicle depart="..."> times (except V001)

    INVARIANT: V001 depart MUST remain 0.
    """


def write_scenario(
    output_dir: Path,
    scenario_id: str,
    sumocfg_tree: ET.ElementTree,
    network_tree: ET.ElementTree,
    routes_tree: ET.ElementTree
) -> Path:
    """
    Write augmented scenario to output directory.

    Args:
        output_dir: Dataset output directory
        scenario_id: Unique scenario identifier (e.g., "train_001")
        *_tree: XML trees to write

    Returns:
        Path to written scenario.sumocfg

    Directory structure created:
        {output_dir}/{scenario_id}/
        ├── scenario.sumocfg
        ├── network.net.xml
        └── vehicles.rou.xml
    """
```

### Step 4: Implement Dataset Layout

**Directory structure:**

```
ml/scenarios/datasets/{dataset_id}/
├── manifest.json           # Metadata for reproducibility
├── train/
│   ├── train_000/
│   │   ├── scenario.sumocfg
│   │   ├── network.net.xml
│   │   └── vehicles.rou.xml
│   ├── train_001/
│   │   └── ...
│   └── train_019/
│       └── ...
└── eval/
    ├── eval_000/
    │   └── ...
    └── eval_004/
        └── ...
```

### Step 5: Implement Manifest Writer

**Function signature:**

```python
def write_manifest(
    output_dir: Path,
    seed: int,
    base_dir: Path,
    emulator_params_path: Path,
    train_scenarios: List[str],
    eval_scenarios: List[str],
    augmentation_ranges: Dict[str, Tuple[float, float]]
) -> None:
    """
    Write manifest.json with full reproducibility metadata.

    Args:
        output_dir: Dataset output directory
        seed: Master random seed used
        base_dir: Path to base scenario
        emulator_params_path: Path to emulator params JSON
        train_scenarios: List of train scenario IDs
        eval_scenarios: List of eval scenario IDs
        augmentation_ranges: Dict of augmentation parameter ranges

    File written: {output_dir}/manifest.json
    """
```

**Manifest schema (exact format):**

```json
{
  "dataset_id": "dataset_v1_2026_01_15",
  "created_at": "2026-01-15T14:30:00Z",
  "seed": 42,
  "base_scenario": {
    "path": "ml/scenarios/base",
    "sumocfg_hash": "sha256:abc123...",
    "network_hash": "sha256:def456...",
    "routes_hash": "sha256:ghi789..."
  },
  "augmentation_ranges": {
    "speedFactor": [0.8, 1.2],
    "sigma": [0.0, 0.5],
    "decel": [3.5, 5.5],
    "tau": [0.5, 1.5],
    "spawn_jitter_s": [0.0, 2.0]
  },
  "train_scenarios": ["train_000", "train_001", "..."],
  "eval_scenarios": ["eval_000", "eval_001", "..."],
  "emulator_params": {
    "path": "ml/espnow_emulator/emulator_params_5m.json",
    "hash": "sha256:jkl012..."
  },
  "environment": {
    "sumo_version": "1.25.0",
    "git_commit": "abc1234",
    "container_image": "ghcr.io/eclipse-sumo/sumo:main"
  }
}
```

**Hash computation:**

```python
import hashlib

def compute_file_hash(file_path: Path) -> str:
    """Compute SHA256 hash of file contents."""
    sha256 = hashlib.sha256()
    with open(file_path, 'rb') as f:
        for chunk in iter(lambda: f.read(8192), b''):
            sha256.update(chunk)
    return f"sha256:{sha256.hexdigest()[:16]}"
```

### Step 6: Add V001 Spawn Validation

**Function signature:**

```python
def validate_v001_spawn(scenario_dir: Path) -> bool:
    """
    Validate that V001 departs at t=0 in the generated scenario.

    Args:
        scenario_dir: Path to scenario directory

    Returns:
        True if V001 departs at t=0, False otherwise

    Raises:
        ValueError: If V001 not found in vehicles.rou.xml

    Implementation:
        1. Parse vehicles.rou.xml
        2. Find <vehicle id="V001" ...>
        3. Assert depart="0" or depart="0.0"
    """
```

**Call this after each scenario generation to enforce the invariant.**

### Step 7: Update .gitignore

**File:** `roadsense-v2v/ml/.gitignore` (create if doesn't exist, otherwise append)

Add the following lines:

```gitignore
# Generated datasets (Phase 1 - Cloud Training Automation)
scenarios/datasets/
```

### Step 8: Add Entry Point to run_docker.sh

**File:** `roadsense-v2v/ml/run_docker.sh`

Add a new subcommand `generate`:

```bash
generate)
    echo "Generating scenarios..."
    shift
    docker run --rm \
        -v "$(pwd)":/work:Z \
        -w /work \
        "$IMAGE_NAME" \
        python -m ml.scripts.gen_scenarios "$@"
    ;;
```

**Usage:**
```bash
./run_docker.sh generate --base_dir ml/scenarios/base --output_dir ml/scenarios/datasets/v1 --seed 42 --train_count 20 --eval_count 5
```

---

## Exit Criteria

- [ ] `gen_scenarios.py` exists and is executable
- [ ] Running `./run_docker.sh generate --seed 42 --train_count 2 --eval_count 1` produces:
  - `ml/scenarios/datasets/*/train/train_000/scenario.sumocfg`
  - `ml/scenarios/datasets/*/train/train_001/scenario.sumocfg`
  - `ml/scenarios/datasets/*/eval/eval_000/scenario.sumocfg`
  - `ml/scenarios/datasets/*/manifest.json`
- [ ] `manifest.json` contains all required fields per schema
- [ ] V001 `depart="0"` verified in ALL generated `vehicles.rou.xml` files
- [ ] Generated datasets are git-ignored (not tracked)
- [ ] Unit tests pass: `pytest ml/tests/unit/test_gen_scenarios.py -v`

---

## Files Touched

| File | Action | Description |
|------|--------|-------------|
| `roadsense-v2v/ml/scripts/gen_scenarios.py` | CREATE | Main generator script |
| `roadsense-v2v/ml/scripts/__init__.py` | CREATE (if missing) | Make scripts a package |
| `roadsense-v2v/ml/.gitignore` | EDIT/CREATE | Add `scenarios/datasets/` |
| `roadsense-v2v/ml/run_docker.sh` | EDIT | Add `generate` subcommand |
| `roadsense-v2v/ml/tests/unit/test_gen_scenarios.py` | CREATE | Unit tests |
| `roadsense-v2v/ml/scenarios/datasets/` | GENERATED | Output directory (ignored) |

---

## Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         gen_scenarios.py Data Flow                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  INPUT                           PROCESS                        OUTPUT      │
│  ─────                           ───────                        ──────      │
│                                                                             │
│  ml/scenarios/base/              ┌─────────────────┐                        │
│  ├── scenario.sumocfg    ───────►│ load_base_      │                        │
│  ├── network.net.xml             │ scenario()      │                        │
│  └── vehicles.rou.xml            └────────┬────────┘                        │
│                                           │                                 │
│  --seed 42               ───────►┌────────▼────────┐                        │
│                                  │ np.random.      │                        │
│                                  │ default_rng()   │                        │
│                                  └────────┬────────┘                        │
│                                           │                                 │
│  augmentation_ranges     ───────►┌────────▼────────┐                        │
│  {speedFactor: [0.8,1.2]         │ augment_routes()│                        │
│   sigma: [0.0,0.5], ...}         │ (N iterations)  │                        │
│                                  └────────┬────────┘                        │
│                                           │                                 │
│                                  ┌────────▼────────┐    ml/scenarios/       │
│                                  │ write_scenario()│───►datasets/{id}/      │
│                                  │ (per scenario)  │    ├── train/          │
│                                  └────────┬────────┘    │   └── train_*/    │
│                                           │             └── eval/           │
│                                  ┌────────▼────────┐        └── eval_*/     │
│                                  │ validate_v001_  │                        │
│                                  │ spawn()         │                        │
│                                  └────────┬────────┘                        │
│                                           │                                 │
│  emulator_params_5m.json ───────►┌────────▼────────┐    manifest.json       │
│  git commit hash                 │ write_manifest()│───►(reproducibility)   │
│  SUMO version                    └─────────────────┘                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Risks / Notes

### Risk 1: V001 Late Spawn
**Problem:** If spawn_jitter is applied to V001, `ConvoyEnv.reset()` will fail when it calls `traci.vehicle.getSubscriptionResults("V001")` before V001 exists.

**Mitigation:** The `augment_routes()` function MUST skip V001 when applying spawn jitter. Add explicit check:
```python
if vehicle.get('id') == 'V001':
    continue  # Never jitter V001 spawn
```

### Risk 2: XML Namespace Issues
**Problem:** SUMO XML files may have namespaces that `xml.etree.ElementTree` doesn't handle cleanly.

**Mitigation:** Use the following parsing approach:
```python
# Remove namespace prefixes for easier XPath
for elem in tree.iter():
    if '}' in elem.tag:
        elem.tag = elem.tag.split('}', 1)[1]
```

### Risk 3: Network File Size
**Problem:** `network.net.xml` can be large (31KB for base scenario). Copying it N times wastes space.

**Mitigation:** Use relative paths in `scenario.sumocfg` to reference a shared network file:
```xml
<net-file value="../../shared/network.net.xml"/>
```

Create a `shared/` directory in the dataset root containing the base network.

### Risk 4: SUMO Version Drift
**Problem:** Scenarios generated with SUMO 1.25.0 may not work with 1.26.0.

**Mitigation:** Record SUMO version in manifest. The generator should call:
```python
import subprocess
result = subprocess.run(['sumo', '--version'], capture_output=True, text=True)
# Parse version from output
```

### Risk 5: Determinism
**Problem:** Running the same seed twice must produce identical scenarios.

**Mitigation:**
1. Use `np.random.default_rng(seed)` (not global `np.random.seed()`)
2. Process scenarios in sorted order
3. Avoid any system-dependent randomness

---

## Test Plan

### Unit Tests (`test_gen_scenarios.py`)

```python
class TestGenScenarios:
    """Unit tests for scenario generator."""

    def test_load_base_scenario_success(self, tmp_path):
        """Verify base scenario loading works."""

    def test_load_base_scenario_missing_file(self, tmp_path):
        """Verify appropriate error when files missing."""

    def test_augment_routes_v001_depart_unchanged(self):
        """CRITICAL: V001 depart must remain 0 after augmentation."""

    def test_augment_routes_parameters_in_range(self):
        """Verify augmented parameters stay within specified ranges."""

    def test_augment_routes_deterministic(self):
        """Same seed produces identical output."""

    def test_write_manifest_schema_valid(self, tmp_path):
        """Verify manifest contains all required fields."""

    def test_validate_v001_spawn_pass(self, tmp_path):
        """Verify validation passes for correct V001 spawn."""

    def test_validate_v001_spawn_fail(self, tmp_path):
        """Verify validation fails for incorrect V001 spawn."""

    def test_compute_file_hash_deterministic(self, tmp_path):
        """Same file produces same hash."""
```

---

## Reference: Existing Base Scenario Structure

**Location:** `roadsense-v2v/ml/scenarios/base/`

**vehicles.rou.xml (relevant excerpt):**
```xml
<routes>
    <vType id="car" accel="2.6" decel="4.5" sigma="0.5"
           length="5" maxSpeed="50" tau="1.0">
        <param key="has.ssm.device" value="true"/>
    </vType>

    <vehicle id="V003" type="car" depart="0" departSpeed="15" departPos="60"/>
    <vehicle id="V002" type="car" depart="0" departSpeed="15" departPos="30"/>
    <vehicle id="V001" type="car" depart="0" departSpeed="15" departPos="0"/>
</routes>
```

**scenario.sumocfg (relevant excerpt):**
```xml
<configuration>
    <input>
        <net-file value="network.net.xml"/>
        <route-files value="vehicles.rou.xml"/>
    </input>
    <time>
        <begin value="0"/>
        <end value="120"/>
        <step-length value="0.1"/>
    </time>
</configuration>
```

---

## Dependencies

| Dependency | Version | Purpose |
|------------|---------|---------|
| `numpy` | >=1.24.0 | Random number generation |
| `sumolib` | >=1.25.0 | SUMO XML utilities (optional, for validation) |
| Standard library | - | `xml.etree.ElementTree`, `hashlib`, `json`, `argparse`, `pathlib` |

---

## Acceptance Checklist (For REVIEWER)

- [ ] `gen_scenarios.py` follows project code style
- [ ] All function signatures match this spec
- [ ] V001 invariant is explicitly enforced (not assumed)
- [ ] Manifest schema exactly matches specification
- [ ] Determinism verified: same seed → identical output
- [ ] Unit tests cover all critical paths
- [ ] `.gitignore` updated to exclude generated datasets
- [ ] `run_docker.sh generate` subcommand works

---

---

## PLANNER Clarifications (January 15, 2026)

### Q1: vehsPerHour in augmentation ranges

**Decision: DROP vehsPerHour entirely.**

The base scenario uses discrete `<vehicle>` entries, not `<flow>`. The IMPLEMENTATION_PLAN.md reference was speculative. Remove `vehsPerHour` from:
- Augmentation ranges table (Step 2)
- `augment_routes()` function
- Manifest `augmentation_ranges` schema

**Updated augmentation parameters:**

| Parameter | XML Attribute | Range | Distribution |
|-----------|---------------|-------|--------------|
| `speedFactor` | `<vType speedFactor="...">` | [0.8, 1.2] | Uniform |
| `sigma` | `<vType sigma="...">` | [0.0, 0.5] | Uniform |
| `decel` | `<vType decel="...">` | [3.5, 5.5] m/s² | Uniform |
| `tau` | `<vType tau="...">` | [0.5, 1.5] s | Uniform |
| `spawn_jitter_s` | `<vehicle depart="...">` | [0.0, 2.0] s | Uniform (V002+ only) |

### Q2: Network file handling (copy vs shared)

**Decision: Use COPY approach for Phase 1 simplicity.**

Copy `network.net.xml` into each scenario directory. The shared approach (Risk 3) is an optimization that can be implemented in a future iteration if disk space becomes a concern.

**Rationale:** The network file is 31KB. With 25 scenarios (20 train + 5 eval), that's ~775KB total—negligible. Keeping scenarios self-contained simplifies debugging and portability.

### Q3: Manifest dataset_id and created_at

**Decision: Use explicit derivation rules.**

```python
dataset_id = output_dir.name  # e.g., "dataset_v1" from --output_dir ml/scenarios/datasets/dataset_v1

from datetime import datetime, timezone
created_at = datetime.now(timezone.utc).isoformat()  # e.g., "2026-01-15T14:30:00+00:00"
```

### Q4: Manifest environment fields inside Docker

**Decision: Graceful fallback with environment variable override.**

```python
def get_git_commit() -> str:
    """Get git commit hash, with fallback."""
    # Check environment variable first (can be passed via Docker)
    if os.environ.get('GIT_COMMIT'):
        return os.environ['GIT_COMMIT']

    try:
        result = subprocess.run(
            ['git', 'rev-parse', '--short', 'HEAD'],
            capture_output=True, text=True, timeout=5
        )
        if result.returncode == 0:
            return result.stdout.strip()
    except (FileNotFoundError, subprocess.TimeoutExpired):
        pass

    return "unknown"


def get_container_image() -> str:
    """Get container image name, with fallback."""
    # Check environment variable (set by run_docker.sh)
    return os.environ.get('CONTAINER_IMAGE', 'unknown')
```

**Update run_docker.sh** to pass these:
```bash
docker run --rm \
    -e GIT_COMMIT="$(git rev-parse --short HEAD 2>/dev/null || echo unknown)" \
    -e CONTAINER_IMAGE="$IMAGE_NAME" \
    ...
```

### Q5: SUMO version handling

**Decision: Require SUMO availability; fail with clear error if missing.**

The generator should ONLY run inside Docker where SUMO is guaranteed. If someone runs it locally without SUMO, fail early with a helpful message.

```python
def get_sumo_version() -> str:
    """Get SUMO version. Fails if SUMO not available."""
    try:
        result = subprocess.run(
            ['sumo', '--version'],
            capture_output=True, text=True, timeout=5
        )
        # Parse "SUMO Version 1.25.0" from output
        for line in result.stdout.split('\n'):
            if 'Version' in line:
                return line.split()[-1]  # e.g., "1.25.0"
    except FileNotFoundError:
        raise RuntimeError(
            "SUMO not found. Run this script inside Docker:\n"
            "  ./run_docker.sh generate --base_dir ml/scenarios/base ..."
        )

    return "unknown"
```

**Why track SUMO version?** Reproducibility. SUMO 1.25 → 1.26 could have subtle behavioral differences (vehicle dynamics, network parsing). The manifest lets us detect if a dataset was generated with a different SUMO version than training uses.

### Q6: Exercise 3/4 scenarios

**Decision: Use existing `ml/scenarios/base/` as the primary base scenario.**

The Exercise 3/4 scenarios referenced in `docs/20_KNOWLEDGE_BASE/` were exploratory work. The current `ml/scenarios/base/` is the production-ready scenario with:
- Validated 3-vehicle convoy (V001, V002, V003)
- SSM device configuration
- 120-second simulation duration
- 10Hz step length

If you want to add Exercise 3/4 as additional base scenarios later, they would need to be migrated into `ml/scenarios/` with the same structure.

---

## Updated Manifest Schema

```json
{
  "dataset_id": "dataset_v1",
  "created_at": "2026-01-15T14:30:00+00:00",
  "seed": 42,
  "base_scenario": {
    "path": "ml/scenarios/base",
    "sumocfg_hash": "sha256:abc123...",
    "network_hash": "sha256:def456...",
    "routes_hash": "sha256:ghi789..."
  },
  "augmentation_ranges": {
    "speedFactor": [0.8, 1.2],
    "sigma": [0.0, 0.5],
    "decel": [3.5, 5.5],
    "tau": [0.5, 1.5],
    "spawn_jitter_s": [0.0, 2.0]
  },
  "train_scenarios": ["train_000", "train_001"],
  "eval_scenarios": ["eval_000"],
  "emulator_params": {
    "path": "ml/espnow_emulator/emulator_params_5m.json",
    "hash": "sha256:jkl012..."
  },
  "environment": {
    "sumo_version": "1.25.0",
    "git_commit": "abc1234",
    "container_image": "ghcr.io/eclipse-sumo/sumo:main"
  }
}
```

**Note:** `vehsPerHour` removed from `augmentation_ranges`.

---

**END OF PHASE 1 SPEC**
