# Run 024 — ReplayConvoyEnv: Real-Data Fine-Tuning

**Status:** READY FOR IMPLEMENTATION (Run 023 gate passed)
**Date:** 2026-03-17 (updated after Run 023 results)
**Author:** Amir + Claude

---

## 0. Run 023 Baseline (Predecessor Results)

Run 023 (`cloud_prod_023`) completed a **2M-step diagnostic** with:
- **State-triggered hazard onset** (H1) — training uses `state_bucket` mode
- **max_closing_speed** added to ego obs (H2) — ego dims 5 -> 6

### Run 023 SUMO Eval (276 episodes, deterministic matrix, 15/15 buckets)

| Metric | Value |
|--------|-------|
| Avg Reward | **+416.94** |
| Collision Rate | **0%** |
| Behavioral Success | **98.55%** |
| Weighted V2V Reaction | **~85%** |

**V2V reaction by rank:**

| Peers | Rank 1 | Rank 2 | Rank 3 | Rank 4 | Rank 5 |
|-------|--------|--------|--------|--------|--------|
| n=1 | 89.3% (1.2s) | - | - | - | - |
| n=2 | 100% (0.2s) | 78.6% (6.5s) | - | - | - |
| n=3 | 100% (0.2s) | 94.7% (2.9s) | 50% (5.7s) | - | - |
| n=4 | 100% (0.2s) | 100% (0.2s) | 92.9% (0.4s) | 92.9% (2.9s) | - |
| n=5 | 100% (0.2s) | 100% (0.2s) | 100% (0.2s) | 100% (0.3s) | 100% (1.9s) |

**Key observations:**
- +20pp V2V reaction over Run 022 (64.4% -> ~85%). H1 state-bucket worked.
- Weak spots: n=1 rank_1 (89%), n=3 rank_3 (50%). Far-peer and single-peer hazards remain harder.
- Run was only 2M (diagnostic). Did NOT extend to 10M.

### Run 023 Replay Validation (March 17, 2026)

Validated with `validate_against_real_data.py`, braking_received_mode=decay:

| Recording | Sensitivity | FP Rate | Detected/Total |
|-----------|------------|---------|----------------|
| Recording #2 | **12.0%** | **11.28%** | 3/25 |
| Extra Driving | **8.7%** | **21.76%** | 2/23 |

**Comparison to Run 022:**

| Metric | Run 022 | Run 023 | Delta |
|--------|---------|---------|-------|
| Rec#2 Sensitivity | 12.0% | 12.0% | 0 |
| Rec#2 FP | 2.55% | 11.28% | **+8.7pp worse** |
| Extra Sensitivity | 26.1% | 8.7% | **-17.4pp worse** |
| Extra FP | 4.58% | 21.76% | **+17.2pp worse** |

**Run 023 replay is WORSE than Run 022 on every metric except Rec#2 sensitivity (tied).**

Key findings from event-level analysis:
- Most missed events have `model_max_action=0.000` — the model outputs exactly zero.
- The 3 detected events in Rec#2 (events 11, 12, 16) are near the convoy close-approach section, same as prior runs.
- Extra Driving detections (events 21-22) are also close-approach, with model going to action=1.0 (full brake).
- FP regression: avg calm action rose from ~0.01 to 0.039 (Rec#2) / 0.139 (Extra). The `max_closing_speed` feature or state-bucket diversity may be producing spurious activations that don't correlate with real hazards.

**Interpretation:** State-bucket onset broadened SUMO training diversity (+20pp SUMO reaction) but did NOT help real-data transfer. The domain gap is architectural, not diversity-limited. H1 is insufficient alone.

### Decision: Move to Run 024

The replay results confirm that further SUMO-only training (extending to 10M, more H1/H3 tuning) will not close the domain gap. The model must see real sensor dynamics during training.

Run 024 fine-tunes the Run 023 checkpoint on real recorded data via `ReplayConvoyEnv`.

---

## 1. Problem Statement

SUMO-trained models consistently fail on real-data replay:
- Run 020: 12% / 17% sensitivity, 0% / 0% FP
- Run 021: 40% / 91% sensitivity but 19% / 94% FP (heading leak)
- Run 022: 12% / 26% sensitivity, 2.6% / 4.6% FP (heading removed)
- **Run 023: 12% / 8.7% sensitivity, 11.3% / 21.8% FP (state-bucket + max_closing_speed)**

Run 023 proves that SUMO-side training diversity improvements (state-bucket onset,
+20pp SUMO V2V reaction) do NOT transfer to real data. The domain gap is
architectural: the model learned SUMO's `slowDown()` dynamics, not real-world
braking signatures observed through noisy ESP-NOW. The only way to close this
gap is to expose the model to **actual real-world sensor data** during training.

## 2. Approach: Hybrid Replay Environment

Build a new Gymnasium environment — `ReplayConvoyEnv` — that:

1. **Replays real peer trajectories** from recorded CSVs (positions, speeds,
   accels from V002/V003 as logged by V001's RX radio).
2. **Simulates the ego vehicle** with simple 2D kinematics (no SUMO needed).
3. **Uses the identical observation pipeline** (`ObservationBuilder`) and
   reward function (`RewardCalculator`) as `ConvoyEnv`.
4. **Augments** recordings to create diverse training scenarios from limited
   real data.

Because the observation and action spaces are identical, a model trained in
SUMO can be loaded and fine-tuned directly on `ReplayConvoyEnv`.

### 2.1 Fine-Tuning Protocol (from Run 023 checkpoint)

```python
from stable_baselines3 import PPO
from stable_baselines3.common.vec_env import DummyVecEnv, VecNormalize

# Load Run 023 (2M SUMO) model — ~85% V2V reaction baseline
model = PPO.load("ml/models/runs/cloud_prod_023/model_final.zip")

# Load reward normalization stats from SUMO training
vec_env = DummyVecEnv([lambda: ReplayConvoyEnv(...)])
vec_env = VecNormalize.load("ml/models/runs/cloud_prod_023/vecnormalize.pkl", vec_env)
vec_env.training = True       # keep updating stats
vec_env.norm_reward = True

# Lower LR to avoid forgetting SUMO-learned behavior
model.learning_rate = 1e-5
model.set_env(vec_env)
model.learn(total_timesteps=500_000)
```

Key points:
- **Same observation space** (ego 6-dim, peers 8x6, peer_mask 8) — weights
  transfer without modification. Ego dims: speed, accel, peer_count,
  min_peer_accel, braking_received_decay, max_closing_speed (Run 023 added
  the 6th dim).
- **Same action space** (Box(1,)) — policy head unchanged.
- **VecNormalize loaded** — reward scaling stays consistent.
- **Lower LR (1e-5)** — prevents catastrophic forgetting of SUMO-learned
  convoy behavior.  The model already knows "brake when peer brakes"; we just
  need to recalibrate what real braking looks like.
- **Shorter training (500k-1M)** — fine-tuning, not training from scratch.
- **Run 023 was 2M steps** (diagnostic), not a full 10M production run.
  This is intentional — we're transferring the base policy, not a
  fully-converged one.

---

## 3. Directory Structure

All new code lives under the existing `ml/` tree.  No new top-level
directories.

```
ml/
├── envs/
│   ├── convoy_env.py              # EXISTING — SUMO-based env (unchanged)
│   ├── replay_convoy_env.py       # NEW — real-data replay env
│   ├── replay_trajectory.py       # NEW — trajectory loading + augmentation
│   ├── ego_kinematics.py          # NEW — simple 2D kinematic ego model
│   ├── observation_builder.py     # EXISTING — shared by both envs (unchanged)
│   ├── reward_calculator.py       # EXISTING — shared by both envs (unchanged)
│   ├── hazard_injector.py         # EXISTING — SUMO-only (not used by replay)
│   ├── action_applicator.py       # EXISTING — SUMO-only (not used by replay)
│   └── __init__.py                # MODIFIED — register ReplayConvoy env
├── data/
│   └── recordings/                # NEW — canonical copy of real CSVs
│       ├── recording_02/          # Recording #2 (195.5s, has braking)
│       │   ├── V001_tx.csv
│       │   └── V001_rx.csv
│       └── extra_driving/         # Extra driving (581.6s, calm)
│           ├── V001_tx.csv
│           └── V001_rx.csv
├── scripts/
│   └── build_replay_dataset.py    # NEW — augmentation CLI
├── tests/
│   ├── unit/
│   │   ├── test_replay_convoy_env.py     # NEW
│   │   ├── test_replay_trajectory.py     # NEW
│   │   └── test_ego_kinematics.py        # NEW
│   └── integration/
│       └── test_replay_fine_tuning.py    # NEW — end-to-end PPO.load → learn
```

### 3.1 Naming Conventions

| Concept | SUMO Environment | Replay Environment |
|---|---|---|
| Env class | `ConvoyEnv` | `ReplayConvoyEnv` |
| Gym ID | `RoadSense-Convoy-v0` | `RoadSense-ReplayConvoy-v0` |
| Test file | `test_convoy_env.py` | `test_replay_convoy_env.py` |
| Hazard source | `HazardInjector` (SUMO TraCI) | Trajectory augmentation (synthetic braking injected into CSV) |
| Ego dynamics | SUMO car-following + `ActionApplicator` | `EgoKinematics` (simple 2D model) |
| Peers | SUMO vehicles + ESP-NOW Emulator | CSV replay (real sensor data) |

---

## 4. Available Real Data

| Recording | Duration | TX Rows | RX Rows | Braking Events | Notes |
|---|---|---|---|---|---|
| Recording #2 | 195.5s | 4,984 | 3,768 | 1 hard brake (-8.63 m/s^2) | Primary hazard source |
| Extra Driving | 581.6s | 14,899 | 12,092 | 0 (calm driving) | FP calibration + synthetic hazard injection |
| **Total** | **777s** | **19,883** | **15,860** | **1** | ~13 minutes |

CSV schema (TX): `timestamp_local_ms, msg_timestamp, vehicle_id, lat, lon,
speed, heading, accel_x, accel_y, ..., hop_count, source_mac`

CSV schema (RX): same but with `from_vehicle_id` instead of `vehicle_id`.

Forward axis = `accel_y` (board mounted Y-forward).

---

## 5. Component Design

### 5.1 `replay_trajectory.py` — Trajectory Loader + Augmentor

Responsible for:
1. Parsing TX/RX CSVs into time-indexed trajectory data.
2. Converting GPS (lat/lon) → meters (x/y) using the project's standard
   `METERS_PER_DEG_LAT = 111_000.0`.
3. Building per-timestep snapshots: ego state + list of peer observations.
4. Applying augmentations to create varied training episodes.

```python
@dataclass
class TrajectorySnapshot:
    """One timestep of a recorded episode."""
    time_ms: int
    ego_x: float          # meters
    ego_y: float          # meters
    ego_speed: float      # m/s
    ego_heading: float    # degrees, SUMO convention (0=North, clockwise)
    ego_accel: float      # m/s^2, forward axis
    peers: List[PeerSnapshot]  # visible peers at this timestep

@dataclass
class PeerSnapshot:
    """One peer observation at one timestep."""
    vehicle_id: str
    x: float
    y: float
    speed: float
    heading: float
    accel: float
    age_ms: float

class ReplayTrajectory:
    """Loads and augments a single recording."""

    def __init__(self, tx_path, rx_path, forward_axis="y"):
        ...

    def load(self) -> List[TrajectorySnapshot]:
        """Parse CSVs into time-indexed snapshots at 10 Hz."""
        ...

    def augment(self, rng, config: AugmentConfig) -> List[TrajectorySnapshot]:
        """Return an augmented copy of the trajectory."""
        ...
```

#### Augmentation strategies

| Strategy | Parameter | Effect |
|---|---|---|
| **Time shift** | `onset_shift_ms: [-2000, +2000]` | Shifts when braking event starts relative to ego |
| **Accel scale** | `accel_scale: [0.6, 1.4]` | Multiplies peer accel by factor (weaker/stronger braking) |
| **Speed scale** | `speed_scale: [0.8, 1.2]` | Scales all speeds (faster/slower convoy) |
| **GPS noise** | `gps_noise_m: [0.5, 3.0]` | Adds Gaussian noise to peer positions |
| **Packet drop** | `drop_rate: [0.0, 0.3]` | Randomly drops RX messages |
| **Initial offset** | `ego_offset_m: [-15, +15]` | Shifts ego start position along heading |
| **Synthetic hazard** | `inject_brake: True/False` | Inserts a braking event into calm recordings |
| **Staleness jitter** | `age_jitter_ms: [0, 100]` | Adds jitter to message age |

**Synthetic hazard injection** (critical for the Extra Driving recording):
- Pick a random peer, random timestep in [30%, 70%] of episode.
- Replace that peer's accel with a braking profile: ramp from 0 to
  `uniform(-4, -9)` m/s^2 over `uniform(0.5, 2.0)` seconds.
- Decelerate that peer's speed accordingly.
- This creates realistic-looking braking events using real convoy geometry.

### 5.2 `ego_kinematics.py` — Ego Vehicle Model

Simple 2D point-mass model.  No SUMO, no TraCI.

```python
class EgoKinematics:
    """
    Kinematic ego model for replay training.

    The ego follows the recorded heading (road direction doesn't change)
    but its speed is controlled by the RL agent.
    """

    MAX_DECEL = 8.0       # m/s^2, matches ActionApplicator
    MAX_SPEED = 30.0      # m/s
    DT = 0.1              # 10 Hz, matches SUMO step

    def __init__(self, initial_x, initial_y, initial_speed, initial_heading):
        self.x = initial_x
        self.y = initial_y
        self.speed = initial_speed
        self.heading = initial_heading  # SUMO degrees (0=North, clockwise)
        self.acceleration = 0.0

    def step(self, action_value: float, heading_deg: float) -> float:
        """
        Apply RL action and advance one timestep.

        Args:
            action_value: RL output in [0, 1]. 0=coast, 1=full brake.
            heading_deg: Road heading at this timestep (from TX recording).

        Returns:
            Actual deceleration applied (m/s^2).
        """
        clamped = max(0.0, min(1.0, float(action_value)))
        requested_decel = clamped * self.MAX_DECEL

        old_speed = self.speed
        self.speed = max(0.0, self.speed - requested_decel * self.DT)
        self.acceleration = (self.speed - old_speed) / self.DT

        # Advance position along road heading
        self.heading = heading_deg
        heading_rad = math.radians(90.0 - heading_deg)  # SUMO→Cartesian
        self.x += self.speed * math.cos(heading_rad) * self.DT
        self.y += self.speed * math.sin(heading_rad) * self.DT

        actual_decel = (old_speed - self.speed) / self.DT
        return actual_decel

    def to_vehicle_state(self) -> VehicleState:
        """Convert to VehicleState for ObservationBuilder compatibility."""
        return VehicleState(
            vehicle_id="V001",
            x=self.x, y=self.y,
            speed=self.speed,
            acceleration=self.acceleration,
            heading=self.heading,
            lane_position=0.0,
        )
```

Why heading comes from the TX recording (not computed from ego motion):
- The RL agent changes ego speed, which changes ego position.
- But the ego is still on the same road — its heading follows the road.
- The TX recording's heading IS the road heading at each timestep.
- If ego is slightly ahead/behind the recording, we interpolate heading
  from the nearest TX timestamp.

### 5.3 `replay_convoy_env.py` — The Gymnasium Environment

```python
class ReplayConvoyEnv(gym.Env):
    """
    Real-data replay environment for sim-to-real fine-tuning.

    Replays recorded peer trajectories while simulating ego with kinematics.
    Observation and action spaces are IDENTICAL to ConvoyEnv, allowing
    direct weight transfer from SUMO-trained models.
    """

    metadata = {"render_modes": []}

    # Identical to ConvoyEnv
    BRAKING_ACCEL_THRESHOLD = -2.5
    BRAKING_DECAY = 0.95

    def __init__(
        self,
        recordings_dir: str,          # Path to recordings/ directory
        augment: bool = True,          # Enable augmentation
        augment_config: dict = None,   # Override augmentation params
        max_steps: int = 500,
        seed: int = 42,
    ):
        super().__init__()

        # Same spaces as ConvoyEnv
        self.observation_space = spaces.Dict({
            "ego": spaces.Box(low=..., high=..., shape=(6,), dtype=np.float32),
            "peers": spaces.Box(low=-1, high=1, shape=(8, 6), dtype=np.float32),
            "peer_mask": spaces.Box(low=0, high=1, shape=(8,), dtype=np.float32),
        })
        self.action_space = spaces.Box(low=0.0, high=1.0, shape=(1,), dtype=np.float32)

        # Shared components (same instances as ConvoyEnv uses)
        self.obs_builder = ObservationBuilder()
        self.reward_calculator = RewardCalculator()
        self.ego_kinematics = None  # Created per episode

        # Load all recordings
        self.trajectories = [...]  # List[ReplayTrajectory]

    def reset(self, *, seed=None, options=None):
        # 1. Pick random recording
        # 2. Augment it (if enabled)
        # 3. Initialize ego kinematics from first snapshot
        # 4. Reset braking_received_decay
        # 5. Return initial observation
        ...

    def step(self, action):
        # 1. Advance timestep index
        # 2. Get peer snapshot for this timestep
        # 3. Apply action to ego kinematics
        # 4. Build peer observation dicts from snapshot
        # 5. Detect braking peers, update braking_received_decay
        # 6. Build observation via ObservationBuilder
        # 7. Compute distance (ego to nearest peer)
        # 8. Compute reward via RewardCalculator
        # 9. Check termination (collision) / truncation (end of recording)
        # 10. Return (obs, reward, terminated, truncated, info)
        ...
```

Key design decisions:
- **No collision termination from ground truth** — there is no ground truth in
  replay.  Collision = ego-to-nearest-peer distance < 5m (same threshold, but
  computed from ego kinematics + peer CSV positions).
- **No CF override** — there is no car-following model.  The ego's only source
  of braking is the RL action.  This is actually more realistic than SUMO
  training (mirrors real deployment where the ESP32 controls braking).
- **Braking received decay** — identical logic to `ConvoyEnv`.  Scans
  cone-filtered peers for accel < -2.5 m/s^2.
- **Episode length** — min(max_steps, recording_length / step_ms).  Short
  recordings produce short episodes; augmented time shifts can vary this.

### 5.4 Gymnasium Registration

In `ml/envs/__init__.py`, add:
```python
register(
    id="RoadSense-ReplayConvoy-v0",
    entry_point="ml.envs.replay_convoy_env:ReplayConvoyEnv",
    kwargs={
        "recordings_dir": os.path.join(
            os.path.dirname(__file__), "..", "data", "recordings"
        ),
    },
)
```

---

## 6. Implementation Plan — TDD Phases

Each phase is one PR-sized chunk.  Tests are written FIRST, then
implementation to make them pass.

### Phase 1: EgoKinematics (tests first)

**Tests** (`tests/unit/test_ego_kinematics.py`):
```
test_initial_state_matches_constructor_args
test_step_zero_action_maintains_speed          # coast, no decel
test_step_full_action_decelerates_by_max_decel # action=1.0 → 8.0 m/s^2
test_step_clamps_speed_to_zero                 # can't go negative
test_step_updates_position_along_heading       # heading=0 → y increases
test_step_updates_position_heading_east        # heading=90 → x increases
test_step_returns_actual_deceleration
test_to_vehicle_state_produces_valid_object
test_acceleration_tracks_speed_change          # state.acceleration computed correctly
```

**Implementation**: `ml/envs/ego_kinematics.py` (~60 lines)

### Phase 2: ReplayTrajectory — Loading (tests first)

**Tests** (`tests/unit/test_replay_trajectory.py`):
```
test_load_parses_tx_csv_into_snapshots
test_load_parses_rx_csv_into_peer_snapshots
test_load_uses_forward_axis_y_by_default
test_load_converts_gps_to_meters
test_load_excludes_self_rx_messages            # V001 in RX filtered out
test_load_snaps_to_10hz_grid
test_load_handles_missing_rx_rows              # timestamps with no peers
test_snapshot_peer_count_matches_rx_data
test_snapshot_peer_accel_matches_csv
test_snapshot_timestamps_are_monotonic
```

**Implementation**: `ml/envs/replay_trajectory.py` — loading only (~120 lines)

### Phase 3: ReplayTrajectory — Augmentation (tests first)

**Tests** (append to `test_replay_trajectory.py`):
```
test_augment_accel_scale_multiplies_peer_accel
test_augment_accel_scale_preserves_ego
test_augment_speed_scale_affects_all_vehicles
test_augment_gps_noise_adds_gaussian_to_positions
test_augment_packet_drop_removes_peers_randomly
test_augment_ego_offset_shifts_initial_position
test_augment_time_shift_moves_braking_onset
test_augment_synthetic_hazard_creates_braking_event
test_augment_synthetic_hazard_decelerates_peer_speed
test_augment_seed_is_reproducible
test_augment_default_config_is_identity         # no changes with all-zero config
```

**Implementation**: augmentation methods in `replay_trajectory.py` (~150 lines)

### Phase 4: ReplayConvoyEnv — Core (tests first)

**Tests** (`tests/unit/test_replay_convoy_env.py`):
```
# Space checks
test_observation_space_matches_convoy_env       # ego(6,), peers(8,6), mask(8,)
test_action_space_matches_convoy_env            # Box(1,) [0,1]

# Reset
test_reset_returns_obs_info_tuple
test_reset_obs_ego_shape_is_6
test_reset_picks_recording_from_directory
test_reset_initializes_ego_at_recording_start
test_reset_clears_braking_received_decay

# Step
test_step_advances_timestep
test_step_returns_five_tuple
test_step_obs_reflects_real_peer_positions
test_step_ego_position_changes_with_action
test_step_zero_action_ego_coasts
test_step_full_brake_ego_decelerates
test_step_braking_peer_triggers_decay_signal
test_step_behind_peer_does_not_trigger_decay
test_step_reward_uses_reward_calculator
test_step_distance_is_ego_to_nearest_peer

# Termination
test_collision_when_ego_reaches_peer            # distance < 5m → terminated
test_end_of_recording_truncates                 # ran out of snapshots
test_max_steps_truncates

# Info dict
test_info_contains_distance_and_reward_breakdown
test_info_contains_braking_received_decay
test_info_contains_deceleration
```

**Implementation**: `ml/envs/replay_convoy_env.py` (~250 lines)

### Phase 5: ReplayConvoyEnv — Augmentation Integration (tests first)

**Tests** (append to `test_replay_convoy_env.py`):
```
test_augment_enabled_produces_varied_episodes   # two resets differ
test_augment_disabled_produces_same_episode     # deterministic replay
test_synthetic_hazard_produces_braking_signal   # braking_received > 0
test_episode_with_synthetic_hazard_has_reward_ignoring_hazard
```

**Implementation**: wire augmentation into `reset()` (~30 lines)

### Phase 6: Fine-Tuning Integration Test

**Tests** (`tests/integration/test_replay_fine_tuning.py`):
```
test_ppo_load_and_predict_on_replay_env         # model from SUMO works
test_ppo_learn_on_replay_env_does_not_crash     # 1000 steps, no errors
test_vecnormalize_loads_from_sumo_training      # reward stats transfer
test_observation_spaces_are_compatible          # ConvoyEnv == ReplayConvoyEnv
```

Note: this test uses a tiny model (not a real trained checkpoint) to verify
the wiring works.  It does NOT require SUMO.

### Phase 7: Dataset Builder Script + Cloud Script

**`ml/scripts/build_replay_dataset.py`**:
- Takes raw recording CSVs + augmentation config.
- Produces a directory of augmented trajectory files (or pre-computed
  snapshot sequences as `.npz`).
- Reports: N episodes, duration distribution, hazard event counts.

**`ml/scripts/cloud/run_fine_tuning.sh`**:
- Loads Run 023 (or latest SUMO) model from S3.
- Runs fine-tuning on replay data.
- Uploads results to S3.

---

## 7. What ReplayConvoyEnv Does NOT Have

These SUMO-specific components are intentionally excluded:

| Component | Why excluded |
|---|---|
| `SUMOConnection` | No SUMO process needed |
| `HazardInjector` | Hazards come from real data or synthetic augmentation |
| `ActionApplicator` | Replaced by `EgoKinematics` |
| `ScenarioManager` | No `.sumocfg` files |
| CF override | No car-following model to override |
| Ground-truth distance | No SUMO positions — use kinematic ego + CSV peers |

What IS shared:
- `ObservationBuilder` — identical observation construction
- `RewardCalculator` — identical reward function
- Braking received decay logic — identical constants and update rule
- Observation/action spaces — identical shapes and ranges

---

## 8. Risk Assessment

| Risk | Likelihood | Mitigation |
|---|---|---|
| Too little real data (13 min, 1 braking event) | High | Aggressive augmentation; synthetic hazard injection into calm recording gives ~100+ hazard episodes |
| Ego kinematics diverge unrealistically from real trajectory | Medium | Heading follows real recording; only speed differs. Clamp ego position to stay within +-50m of recorded position |
| Overfitting to 2 recordings | High | Augmentation diversity; validate on held-out time segments (first 80% train, last 20% eval) |
| Catastrophic forgetting of SUMO-learned behavior | Medium | Low LR (1e-5); short fine-tuning (500k); evaluate on SUMO eval after fine-tuning to confirm no regression |
| Reward scale mismatch between SUMO and replay | Low | Same RewardCalculator; load VecNormalize stats from SUMO training |

---

## 9. Success Criteria

### 9.0 Run 023 Replay Baseline (Established)

Replay validation completed March 17, 2026:

| Recording | Sensitivity | FP Rate |
|-----------|------------|---------|
| Recording #2 | 12.0% (3/25) | 11.28% |
| Extra Driving | 8.7% (2/23) | 21.76% |

Results stored: `ml/data/sim_to_real_validation_run023/`

### 9.1 Run 024 Diagnostic (500k fine-tuning steps)

**Must-pass (before extending):**
- Recording #2 sensitivity > 40% (up from 12.0%)
- Recording #2 FP < 15% (currently 11.28% — must not regress significantly)
- Extra Driving FP < 25% (currently 21.76% — must not get worse)
- SUMO eval still > 75% V2V reaction (no catastrophic forgetting;
  Run 023 baseline is ~85%, allow 10pp regression)

**Target (for production promotion):**
- Recording #2 sensitivity > 60%, FP < 15%
- Extra Driving sensitivity > 75%, FP < 20%
- SUMO eval > 80% V2V reaction, 0% collisions

### 9.2 Validation Protocol

1. Fine-tune on replay data.
2. Run `validate_against_real_data.py` on fine-tuned model (both recordings).
3. Run SUMO eval with `check_v2v_reaction.py` (Docker) to check for regression.
4. Compare sensitivity/FP against Run 023 replay baseline (Section 9.0).
5. If sensitivity improved but SUMO regressed: iterate on LR / training length.
6. If sensitivity did not improve: investigate observation pipeline mismatch
   between `ReplayConvoyEnv` and `validate_against_real_data.py`.

---

## 10. Open Questions

1. **Train/eval split for recordings**: Split each recording 80/20 by time?
   Or use Recording #2 for training and Extra Driving for eval only?
   Recommendation: use both for training (augmented), hold out last 20% of
   each for validation.

2. **How many augmented episodes?**: With 8 augmentation knobs and 2 base
   recordings, we can generate thousands.  Start with 200 (100 per recording)
   and scale up if needed.

3. **Should we also inject SUMO-like hazards?**: The synthetic hazard
   augmentation uses realistic braking profiles.  We could also inject
   SUMO-style instant stops to maintain backward compatibility.
   Recommendation: NO — the whole point is to move away from SUMO dynamics.

4. **Ego position clamping**: If the RL agent brakes hard, ego falls far
   behind the recorded trajectory.  Should we clamp?  Recommendation: soft
   penalty if ego drifts > 50m from recorded position, but don't hard-clamp
   (let the reward handle it via the "far" penalty at > 35m).
