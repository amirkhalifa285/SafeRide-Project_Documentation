# ConvoyEnv Gymnasium Implementation Plan

**Document Type:** AI Agent Implementation Prompt + Architecture Design
**Created:** January 5, 2026
**Author:** Amir Khalifa (with Claude analysis)
**Priority:** HIGH - Next step after RTT characterization
**Dependencies:** ESP-NOW Emulator (COMPLETE), SUMO Exercise 4 (COMPLETE)
**Status:** OBSERVATION SPACE SUPERSEDED

---

> ## SUPERSEDED NOTICE (January 13, 2026)
>
> **The observation space design in this document (Box(11,) with hardcoded V002/V003) has been superseded.**
>
> **See:** `10_PLANS_ACTIVE/N_ELEMENT_IMPLEMENTATION_PLAN.md` for updated observation space.
>
> **Key Changes:**
> - Observation: `Box(11,)` --> `Dict` with variable peers
> - Policy: `MlpPolicy` --> `MultiInputPolicy` with `DeepSetFeatureExtractor`
> - Peers: Hardcoded V002/V003 --> Dynamic peer list (0 to MAX_PEERS)
>
> **What Remains Valid:** SUMO integration, reward function, episode management, testing strategy

---

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Architecture Overview](#2-architecture-overview)
3. [Component Design](#3-component-design)
4. [Observation & Action Spaces](#4-observation--action-spaces)
5. [Reward Function Design](#5-reward-function-design)
6. [SUMO Integration Strategy](#6-sumo-integration-strategy)
7. [Implementation Phases](#7-implementation-phases)
8. [Testing Strategy](#8-testing-strategy)
9. [File Structure](#9-file-structure)
10. [Critical Design Decisions](#10-critical-design-decisions)

---

## 1. Executive Summary

### What We're Building

A **Gymnasium-compatible environment** (`ConvoyEnv`) that:
1. Runs SUMO convoy simulations via TraCI
2. Feeds vehicle states through the **existing ESP-NOW Emulator**
3. Provides realistic observations with communication effects (latency, loss, staleness)
4. Exposes a discrete action space for hazard response decisions
5. Computes rewards based on safety metrics (distance, TTC-inspired thresholds)

### Why This Matters

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TRAINING PIPELINE POSITION                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ✅ RTT Firmware (COMPLETE)     → Field drive pending                   │
│  ✅ ESP-NOW Emulator (COMPLETE) → 84/84 tests, production-ready         │
│  ⏳ ConvoyEnv (THIS TASK)       → Gymnasium environment                 │
│  ⏳ RL Training                 → After ConvoyEnv complete              │
│  ⏳ TFLite Conversion           → After training                        │
│  ⏳ Real-World Validation       → Final phase                           │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Key Constraints

| Constraint | Value | Source |
|------------|-------|--------|
| Observation features | 11 floats | ARCHITECTURE_V2_MINIMAL_RL.md |
| Action space | Discrete(4) | Professor requirement |
| Simulation rate | 10 Hz (0.1s step) | Matches hardware |
| Max episode length | 1000 steps (100s) | Training efficiency |
| SUMO version | 1.25.0+ | Docker image verified |

---

## 2. Architecture Overview

### System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         ConvoyEnv ARCHITECTURE                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │                        SUMO Simulation                              │    │
│  │                                                                     │    │
│  │   V003 (Lead)  ────────  V002 (Middle)  ────────  V001 (Ego)       │    │
│  │   "Dumb"                 "Dumb"                   RL Agent          │    │
│  │   Broadcaster            Broadcaster              Receiver          │    │
│  │                                                                     │    │
│  │   TraCI Interface:                                                  │    │
│  │   - getPosition(), getSpeed(), getAcceleration()                   │    │
│  │   - setSpeed() for action application                               │    │
│  │                                                                     │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                              │                                              │
│                              │ Perfect vehicle states (10 Hz)               │
│                              ▼                                              │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │                     ESP-NOW Emulator (Existing)                     │    │
│  │                                                                     │    │
│  │   Input: Perfect SUMO states for V002, V003                        │    │
│  │                                                                     │    │
│  │   Processing:                                                       │    │
│  │   ├── Latency model (base + distance + jitter)                     │    │
│  │   ├── Packet loss model (distance-dependent + burst)               │    │
│  │   ├── Sensor noise injection                                        │    │
│  │   └── Causality enforcement (message queue)                         │    │
│  │                                                                     │    │
│  │   Output: Realistic observations with age_ms, valid flags          │    │
│  │                                                                     │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                              │                                              │
│                              │ Observations with communication effects      │
│                              ▼                                              │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │                       ConvoyEnv (Gymnasium)                         │    │
│  │                                                                     │    │
│  │   Observation Space: Box(11,) float32                              │    │
│  │   ┌─────────────────────────────────────────────────────────────┐  │    │
│  │   │ ego_speed                                    (1)             │  │    │
│  │   │ v002: rel_dist, rel_speed, accel, age_norm, valid (5)       │  │    │
│  │   │ v003: rel_dist, rel_speed, accel, age_norm, valid (5)       │  │    │
│  │   └─────────────────────────────────────────────────────────────┘  │    │
│  │                                                                     │    │
│  │   Action Space: Discrete(4)                                        │    │
│  │   ┌─────────────────────────────────────────────────────────────┐  │    │
│  │   │ 0=Maintain │ 1=Caution │ 2=Brake │ 3=Emergency              │  │    │
│  │   └─────────────────────────────────────────────────────────────┘  │    │
│  │                                                                     │    │
│  │   Reward Function: Distance-based safety metrics                   │    │
│  │                                                                     │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                              │                                              │
│                              │ (obs, reward, terminated, truncated, info)   │
│                              ▼                                              │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │                     RL Training (stable-baselines3)                 │    │
│  │                                                                     │    │
│  │   Algorithm: PPO                                                    │    │
│  │   Network: MLP [64, 64]                                            │    │
│  │   Training: 100,000+ timesteps                                      │    │
│  │                                                                     │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Data Flow Per Step

```
1. ConvoyEnv.step(action)
   │
   ├── 2. Apply action to V001 via TraCI
   │       └── setSpeed() based on action type
   │
   ├── 3. Advance SUMO simulation by 0.1s
   │       └── traci.simulationStep()
   │
   ├── 4. Get perfect vehicle states from SUMO
   │       └── V001, V002, V003: pos, speed, accel, heading
   │
   ├── 5. Transmit V002, V003 states through emulator
   │       └── emulator.transmit(msg, sender_pos, receiver_pos, time)
   │
   ├── 6. Get observation from emulator
   │       └── emulator.get_observation(ego_speed, time)
   │
   ├── 7. Calculate reward
   │       └── Based on distance, action appropriateness
   │
   ├── 8. Check termination conditions
   │       └── Collision, max_steps, vehicle_exited
   │
   └── 9. Return (obs, reward, terminated, truncated, info)
```

---

## 3. Component Design

### 3.1 SUMOConnection Class

**Purpose:** Manage SUMO/TraCI lifecycle and vehicle state extraction.

```python
class SUMOConnection:
    """
    Manages SUMO simulation via TraCI.

    Responsibilities:
    - Start/stop SUMO process
    - Extract vehicle states
    - Apply actions to ego vehicle
    - Handle scenario loading
    """

    def __init__(self, sumo_cfg: str, gui: bool = False,
                 sumo_binary: str = None, port: int = None):
        """
        Args:
            sumo_cfg: Path to .sumocfg file
            gui: If True, launch sumo-gui instead of sumo
            sumo_binary: Override SUMO binary path (for Docker)
            port: TraCI port (None = auto-select)
        """

    def start(self) -> None:
        """Start SUMO simulation."""

    def stop(self) -> None:
        """Stop SUMO simulation."""

    def step(self) -> None:
        """Advance simulation by one timestep."""

    def get_vehicle_state(self, vehicle_id: str) -> VehicleState:
        """Get current state of a vehicle."""

    def set_vehicle_speed(self, vehicle_id: str, speed: float) -> None:
        """Set target speed for a vehicle."""

    def get_simulation_time(self) -> float:
        """Get current simulation time in seconds."""

    def is_vehicle_active(self, vehicle_id: str) -> bool:
        """Check if vehicle is still in simulation."""
```

### 3.2 VehicleState Dataclass

**Purpose:** Structured container for vehicle state from SUMO.

```python
@dataclass(frozen=True)
class VehicleState:
    """Vehicle state extracted from SUMO via TraCI."""
    vehicle_id: str
    x: float              # SUMO x coordinate (meters)
    y: float              # SUMO y coordinate (meters)
    speed: float          # Speed in m/s
    acceleration: float   # Acceleration in m/s²
    heading: float        # Heading in degrees (SUMO angle)
    lane_position: float  # Position along lane

    def to_v2v_message(self, timestamp_ms: int) -> V2VMessage:
        """Convert to V2VMessage for emulator."""
```

### 3.3 ActionApplicator Class

**Purpose:** Translate discrete actions to SUMO vehicle control.

```python
class ActionApplicator:
    """
    Translates discrete RL actions to SUMO vehicle control.

    Actions affect ego vehicle (V001) speed:
    - MAINTAIN (0): No intervention, let SUMO car-following handle
    - CAUTION (1): Reduce speed by small amount
    - BRAKE (2): Moderate braking
    - EMERGENCY (3): Maximum deceleration
    """

    # Action definitions
    MAINTAIN = 0
    CAUTION = 1
    BRAKE = 2
    EMERGENCY = 3

    # Speed reduction per action (m/s per step)
    ACTION_DECEL = {
        0: 0.0,    # Maintain - no change
        1: 0.5,    # Caution - gentle slowdown
        2: 2.0,    # Brake - moderate
        3: 4.5,    # Emergency - max decel
    }

    def apply(self, sumo: SUMOConnection, action: int) -> float:
        """
        Apply action to ego vehicle.

        Returns:
            Actual deceleration applied (for reward calculation)
        """
```

### 3.4 RewardCalculator Class

**Purpose:** Compute step reward based on safety metrics.

```python
class RewardCalculator:
    """
    Calculates reward based on:
    - Distance to lead vehicle
    - Action appropriateness
    - Comfort (harsh braking penalty)

    Reward structure from ARCHITECTURE_V2_MINIMAL_RL.md:
    - Collision: -100
    - Unsafe proximity (<15m): -5
    - Safe following (20-40m): +1
    - Harsh braking (>4.5 m/s²): -10
    - Unnecessary alert (>40m, action>1): -2
    """

    # Distance thresholds (meters)
    COLLISION_DIST = 5.0
    UNSAFE_DIST = 15.0
    SAFE_DIST_MIN = 20.0
    SAFE_DIST_MAX = 40.0

    # Reward values
    REWARD_COLLISION = -100.0
    REWARD_UNSAFE = -5.0
    REWARD_SAFE = +1.0
    REWARD_HARSH_BRAKE = -10.0
    REWARD_UNNECESSARY_ALERT = -2.0

    def calculate(self, distance: float, action: int,
                  deceleration: float) -> Tuple[float, Dict]:
        """
        Calculate reward for current step.

        Returns:
            Tuple of (reward, info_dict)
        """
```

### 3.5 ObservationBuilder Class

**Purpose:** Convert emulator output to normalized observation array.

```python
class ObservationBuilder:
    """
    Builds normalized observation array from emulator output.

    Observation space (11 features):
    [0]    ego_speed (normalized by max_speed)
    [1-5]  v002: rel_dist, rel_speed, accel, age_norm, valid
    [6-10] v003: rel_dist, rel_speed, accel, age_norm, valid

    Normalization:
    - Speed: / 30.0 (max highway speed)
    - Distance: / 100.0 (max relevant distance)
    - Acceleration: / 10.0 (max expected accel)
    - Age: / 500.0 (staleness threshold)
    - Valid: 0.0 or 1.0
    """

    MAX_SPEED = 30.0       # m/s
    MAX_DISTANCE = 100.0   # meters
    MAX_ACCEL = 10.0       # m/s²
    STALENESS_THRESHOLD = 500.0  # ms

    def build(self, ego_state: VehicleState,
              emulator_obs: Dict) -> np.ndarray:
        """
        Build observation array from ego state and emulator output.

        Returns:
            np.ndarray of shape (11,) with dtype float32
        """
```

---

## 4. Observation & Action Spaces

### 4.1 Observation Space (11 Features)

| Index | Feature | Description | Normalization | Range |
|-------|---------|-------------|---------------|-------|
| 0 | `ego_speed` | Ego vehicle speed | / 30.0 | [0, 1+] |
| 1 | `v002_rel_dist` | Distance to V002 | / 100.0 | [-1, 1+] |
| 2 | `v002_rel_speed` | Relative speed (v002 - ego) | / 30.0 | [-1, 1] |
| 3 | `v002_accel` | V002 acceleration | / 10.0 | [-1, 1] |
| 4 | `v002_age_norm` | Message age | / 500.0 | [0, 1+] |
| 5 | `v002_valid` | Is message fresh? | boolean | {0, 1} |
| 6 | `v003_rel_dist` | Distance to V003 | / 100.0 | [-1, 1+] |
| 7 | `v003_rel_speed` | Relative speed (v003 - ego) | / 30.0 | [-1, 1] |
| 8 | `v003_accel` | V003 acceleration | / 10.0 | [-1, 1] |
| 9 | `v003_age_norm` | Message age | / 500.0 | [0, 1+] |
| 10 | `v003_valid` | Is message fresh? | boolean | {0, 1} |

**Design Rationale:**
- **Relative distance/speed**: More useful for following task than absolute values
- **Normalization**: Helps neural network training convergence
- **Valid flag**: Explicit signal for stale/missing data (critical for safety)

### 4.2 Action Space (Discrete 4)

| Action | Name | Effect | When Appropriate |
|--------|------|--------|------------------|
| 0 | MAINTAIN | No intervention | Safe following distance |
| 1 | CAUTION | Gentle slowdown (-0.5 m/s) | Elevated risk (TTC 2-4s) |
| 2 | BRAKE | Moderate braking (-2.0 m/s) | Serious conflict (TTC 1.5-2s) |
| 3 | EMERGENCY | Max deceleration (-4.5 m/s) | Severe conflict (TTC <1.5s) |

**Note:** Actions are *advisory* - they represent warning levels in the real system. The speed reduction values are for simulation only.

---

## 5. Reward Function Design

### 5.1 Reward Components

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         REWARD STRUCTURE                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  R_total = R_safety + R_comfort + R_appropriateness                     │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ R_safety (Distance-based)                                        │    │
│  │                                                                  │    │
│  │   d < 5m  (Collision)     → -100                                │    │
│  │   d < 15m (Unsafe)        → -5                                  │    │
│  │   20m < d < 40m (Safe)    → +1                                  │    │
│  │   d > 40m (Too far)       → +0.5 (still okay, but not optimal)  │    │
│  │   else                    → 0                                    │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ R_comfort (Deceleration-based)                                   │    │
│  │                                                                  │    │
│  │   |decel| > 4.5 m/s²      → -10 (Harsh braking)                 │    │
│  │   |decel| > 3.0 m/s²      → -2  (Uncomfortable)                 │    │
│  │   else                    → 0                                    │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ R_appropriateness (Action vs Situation)                          │    │
│  │                                                                  │    │
│  │   d > 40m AND action > 1  → -2 (Unnecessary alert)              │    │
│  │   d < 15m AND action == 0 → -3 (Missed warning)                 │    │
│  │   else                    → 0                                    │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Reward Shaping Philosophy

1. **Safety is paramount**: Collision penalty (-100) dominates all other rewards
2. **Comfort matters**: Harsh braking is penalized even if safe
3. **False alarms are costly**: Unnecessary alerts annoy drivers
4. **Missed warnings are dangerous**: Failing to warn is worse than false alarm

### 5.3 Episode Termination

| Condition | Type | Reward Applied |
|-----------|------|----------------|
| Distance < 5m | `terminated=True` | -100 (collision) |
| Steps >= 1000 | `truncated=True` | None |
| Vehicle exits simulation | `truncated=True` | None |
| SUMO simulation ends | `truncated=True` | None |

---

## 6. SUMO Integration Strategy

### 6.1 TraCI Connection Options

**Option A: Local SUMO (Recommended for Development)**
```python
# Direct connection to local SUMO installation
sumo_binary = "sumo"  # or "sumo-gui"
traci.start([sumo_binary, "-c", sumo_cfg, "--step-length", "0.1"])
```

**Option B: Docker SUMO (Recommended for CI/Production)**
```python
# Launch Docker container with TraCI port exposed
# docker run -p 8813:8813 -v $(pwd):/data ghcr.io/eclipse-sumo/sumo:main \
#   sumo --remote-port 8813 -c /data/scenario.sumocfg

traci.connect(port=8813)
```

### 6.2 Scenario Management

The environment should support multiple scenarios:

```python
class ScenarioManager:
    """
    Manages SUMO scenario selection and variation.

    Scenarios from exercise4_variations/:
    - base/               → Normal driving
    - var_tight_convoy/   → Close spacing
    - var_speed_fast/     → Higher speeds
    - var_weak_brakes/    → Lower decel capability
    - var_aggressive_follower/ → Aggressive tau
    """

    def __init__(self, scenario_dir: str):
        self.scenarios = self._discover_scenarios(scenario_dir)

    def get_random_scenario(self) -> str:
        """Return path to random .sumocfg"""

    def get_scenario(self, name: str) -> str:
        """Return path to specific scenario"""
```

### 6.3 Emergency Brake Injection (Critical for Training)

SUMO's car-following model is safe by design - it won't naturally create collisions. We need to inject hazard events:

```python
class HazardInjector:
    """
    Injects hazard events mid-episode using TraCI.

    Hazard types:
    - EMERGENCY_BRAKE: Lead vehicle stops suddenly
    - CHAIN_REACTION: Middle vehicle brakes, lead continues
    - SLOWDOWN: Gradual deceleration (tests false positive rate)
    """

    HAZARD_PROBABILITY = 0.3  # 30% of episodes get a hazard
    HAZARD_WINDOW = (30, 80)  # Inject between steps 30-80

    def maybe_inject(self, step: int, sumo: SUMOConnection) -> bool:
        """
        Probabilistically inject a hazard event.

        Returns True if hazard was injected.
        """
```

---

## 7. Implementation Phases

### Phase 0: Project Setup (Day 1 Morning)

| Step | Task | Acceptance Criteria |
|------|------|---------------------|
| 0.1 | Create directory structure | `ml/envs/` exists |
| 0.2 | Create `__init__.py` files | Package importable |
| 0.3 | Add dependencies to requirements.txt | `gymnasium`, `traci`, `sumolib` |
| 0.4 | Verify SUMO Docker connectivity | `docker run ... sumo --version` works |
| 0.5 | Copy exercise4 scenarios to ml/scenarios/ | Scenarios accessible from ml/ |

### Phase 1: Core Data Structures (Day 1 Morning)

| Step | Task | Acceptance Criteria |
|------|------|---------------------|
| 1.1 | Write tests for `VehicleState` | 5 tests defined |
| 1.2 | Implement `VehicleState` dataclass | Tests pass |
| 1.3 | Write tests for `to_v2v_message()` conversion | 3 tests defined |
| 1.4 | Implement conversion method | Tests pass |

### Phase 2: SUMO Integration (Day 1 Afternoon)

| Step | Task | Acceptance Criteria |
|------|------|---------------------|
| 2.1 | Write tests for `SUMOConnection.start/stop` | 4 tests defined |
| 2.2 | Implement SUMO connection lifecycle | Tests pass |
| 2.3 | Write tests for `get_vehicle_state()` | 5 tests defined |
| 2.4 | Implement vehicle state extraction | Tests pass |
| 2.5 | Write tests for `set_vehicle_speed()` | 3 tests defined |
| 2.6 | Implement speed control | Tests pass |
| 2.7 | Integration test: Run 100 steps | No crashes, states extracted |

### Phase 3: Emulator Integration (Day 1 Evening)

| Step | Task | Acceptance Criteria |
|------|------|---------------------|
| 3.1 | Write tests for SUMO→Emulator pipeline | 4 tests defined |
| 3.2 | Implement message transmission in env | Tests pass |
| 3.3 | Write tests for observation generation | 5 tests defined |
| 3.4 | Implement `ObservationBuilder` | Tests pass |
| 3.5 | Verify causality (message not visible before arrival) | Causality test passes |

### Phase 4: Action Application (Day 2 Morning)

| Step | Task | Acceptance Criteria |
|------|------|---------------------|
| 4.1 | Write tests for `ActionApplicator` | 6 tests defined |
| 4.2 | Implement action→speed mapping | Tests pass |
| 4.3 | Write tests for action effects on simulation | 4 tests defined |
| 4.4 | Verify actions affect V001 speed correctly | Integration test passes |

### Phase 5: Reward Function (Day 2 Morning)

| Step | Task | Acceptance Criteria |
|------|------|---------------------|
| 5.1 | Write tests for `RewardCalculator` | 10 tests defined |
| 5.2 | Implement distance-based rewards | Tests pass |
| 5.3 | Implement comfort penalties | Tests pass |
| 5.4 | Implement appropriateness rewards | Tests pass |
| 5.5 | Verify reward ranges are reasonable | Statistical test passes |

### Phase 6: Episode Management (Day 2 Afternoon)

| Step | Task | Acceptance Criteria |
|------|------|---------------------|
| 6.1 | Write tests for `reset()` | 5 tests defined |
| 6.2 | Implement episode reset | Tests pass |
| 6.3 | Write tests for termination conditions | 6 tests defined |
| 6.4 | Implement collision detection | Tests pass |
| 6.5 | Implement truncation logic | Tests pass |
| 6.6 | Write tests for `HazardInjector` | 4 tests defined |
| 6.7 | Implement hazard injection | Tests pass |

### Phase 7: Integration & Gymnasium Registration (Day 2 Afternoon)

| Step | Task | Acceptance Criteria |
|------|------|---------------------|
| 7.1 | Assemble `ConvoyEnv` class | All components integrated |
| 7.2 | Write full episode integration test | 1 episode completes |
| 7.3 | Register with Gymnasium | `gym.make('RoadSense-Convoy-v0')` works |
| 7.4 | Random agent test (100 episodes) | All episodes complete without crash |
| 7.5 | Verify observation/action spaces | Spaces match spec |

### Phase 8: Documentation & Validation (Day 2 Evening)

| Step | Task | Acceptance Criteria |
|------|------|---------------------|
| 8.1 | Add docstrings to all public methods | 100% coverage |
| 8.2 | Create usage example script | `examples/train_convoy.py` |
| 8.3 | Run full test suite | All tests pass |
| 8.4 | Document test coverage | Coverage report generated |
| 8.5 | Update progress tracker | All phases marked complete |

---

## 8. Testing Strategy

### 8.1 Test Categories

| Category | Purpose | Count (Est.) |
|----------|---------|--------------|
| Unit tests | Individual component behavior | ~40 |
| Integration tests | Component interaction | ~15 |
| Statistical tests | Stochastic behavior validation | ~5 |
| End-to-end tests | Full episode runs | ~5 |

### 8.2 Critical Test Cases

**Must-Have Tests:**
1. **Causality**: Message not visible before arrival time
2. **Collision detection**: Episode terminates on collision
3. **Reward correctness**: Collision gives -100, safe gives +1
4. **Observation shape**: Always (11,) float32
5. **Action effects**: Each action produces expected speed change
6. **Reset hygiene**: No state leaks between episodes
7. **SUMO lifecycle**: Start/stop doesn't crash

### 8.3 Test Fixtures

```python
# conftest.py fixtures

@pytest.fixture
def mock_sumo():
    """Mock SUMO connection for unit tests."""

@pytest.fixture
def real_sumo(tmp_path):
    """Real SUMO connection for integration tests."""

@pytest.fixture
def emulator():
    """ESP-NOW emulator with default params."""

@pytest.fixture
def env():
    """Full ConvoyEnv for end-to-end tests."""
```

---

## 9. File Structure

```
roadsense-v2v/ml/
├── envs/
│   ├── __init__.py
│   ├── convoy_env.py           # Main Gymnasium environment
│   ├── sumo_connection.py      # SUMO/TraCI management
│   ├── action_applicator.py    # Action → SUMO translation
│   ├── reward_calculator.py    # Reward computation
│   ├── observation_builder.py  # Observation normalization
│   ├── hazard_injector.py      # Emergency brake injection
│   └── scenario_manager.py     # Scenario selection
├── espnow_emulator/            # Existing (unchanged)
│   ├── __init__.py
│   └── espnow_emulator.py
├── scenarios/                  # SUMO scenario files
│   ├── base/
│   ├── var_tight_convoy/
│   ├── var_speed_fast/
│   └── ...
├── tests/
│   ├── __init__.py
│   ├── conftest.py             # Shared fixtures
│   ├── test_data_structures.py # Existing
│   ├── test_vehicle_state.py   # NEW
│   ├── test_sumo_connection.py # NEW
│   ├── test_action_applicator.py # NEW
│   ├── test_reward_calculator.py # NEW
│   ├── test_observation_builder.py # NEW
│   ├── test_convoy_env.py      # NEW - Integration tests
│   └── test_hazard_injector.py # NEW
├── examples/
│   ├── demo_emulator.py        # Existing
│   └── train_convoy.py         # NEW - Training script
└── requirements.txt            # Updated with gymnasium, traci
```

---

## 10. Critical Design Decisions

### Decision 1: Relative vs Absolute Observations

**Choice:** Relative distances/speeds
**Rationale:**
- More generalizable (works on different road geometries)
- Directly relevant to following task
- Smaller observation space

### Decision 2: Normalized vs Raw Observations

**Choice:** Normalized to roughly [-1, 1] range
**Rationale:**
- Better neural network training convergence
- Prevents feature dominance
- Industry standard practice

### Decision 3: Discrete vs Continuous Actions

**Choice:** Discrete(4)
**Rationale:**
- Matches real system (warning levels, not continuous control)
- Simpler to implement and debug
- Professor requirement

### Decision 4: Episode Length

**Choice:** Max 1000 steps (100 seconds)
**Rationale:**
- Long enough for meaningful convoy behavior
- Short enough for efficient training
- Matches realistic driving segment

### Decision 5: Hazard Injection Strategy

**Choice:** Probabilistic injection (30% of episodes)
**Rationale:**
- Natural scenarios are too safe (SUMO prevents collisions)
- Need CRITICAL class examples for balanced training
- Random timing prevents agent from memorizing patterns

### Decision 6: SUMO Integration Mode

**Choice:** TraCI real-time (not offline FCD processing)
**Rationale:**
- Enables action feedback loop (RL needs this)
- Allows hazard injection mid-episode
- More realistic training dynamics

---

## Appendix A: Gymnasium API Reference

```python
class ConvoyEnv(gymnasium.Env):
    """
    RoadSense V2V Convoy Environment

    A 3-vehicle convoy following scenario with ESP-NOW communication effects.
    The ego vehicle (V001) follows V002 and V003, receiving V2V messages
    with realistic latency, packet loss, and staleness.

    Observation Space:
        Box(11,) - [ego_speed, v002_*, v003_*]

    Action Space:
        Discrete(4) - [Maintain, Caution, Brake, Emergency]

    Reward:
        Distance-based safety rewards with comfort and appropriateness penalties.
    """

    metadata = {"render_modes": ["human", "rgb_array"], "render_fps": 10}

    def __init__(self,
                 scenario_dir: str = None,
                 emulator_params: str = None,
                 domain_randomization: bool = True,
                 hazard_injection: bool = True,
                 render_mode: str = None,
                 max_steps: int = 1000):
        """
        Args:
            scenario_dir: Path to SUMO scenarios directory
            emulator_params: Path to emulator_params.json (optional)
            domain_randomization: Enable per-episode parameter variation
            hazard_injection: Enable random emergency brake events
            render_mode: "human" for GUI, None for headless
            max_steps: Maximum steps per episode
        """

    def reset(self, seed=None, options=None):
        """Reset environment to initial state."""

    def step(self, action: int):
        """Execute one timestep."""

    def render(self):
        """Render current state (if render_mode='human')."""

    def close(self):
        """Clean up resources."""
```

---

## Appendix B: Example Training Script

```python
# examples/train_convoy.py

import gymnasium as gym
from stable_baselines3 import PPO
from stable_baselines3.common.env_checker import check_env

# Register environment
import ml.envs  # This registers 'RoadSense-Convoy-v0'

# Create environment
env = gym.make('RoadSense-Convoy-v0',
               domain_randomization=True,
               hazard_injection=True)

# Verify environment
check_env(env, warn=True)

# Train
model = PPO("MlpPolicy", env, verbose=1,
            learning_rate=3e-4,
            n_steps=2048,
            batch_size=64,
            n_epochs=10,
            gamma=0.99,
            tensorboard_log="./tensorboard/")

model.learn(total_timesteps=100_000)
model.save("convoy_ppo_100k")

# Evaluate
obs, _ = env.reset()
for _ in range(1000):
    action, _ = model.predict(obs, deterministic=True)
    obs, reward, terminated, truncated, info = env.step(action)
    if terminated or truncated:
        obs, _ = env.reset()

env.close()
```

---

**Document Version:** 1.0
**Created:** January 5, 2026
**Status:** Ready for Implementation

---

**END OF DOCUMENT**
