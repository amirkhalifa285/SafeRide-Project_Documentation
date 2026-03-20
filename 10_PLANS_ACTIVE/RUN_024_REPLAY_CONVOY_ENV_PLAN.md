# Run 024 — ReplayConvoyEnv: Real-Data Fine-Tuning

**Status:** ROOT CAUSE INVESTIGATION COMPLETE — 4 BUGS FIXED — SENSITIVITY 12%→72% ACHIEVED — REPLAY REWARD GATING TESTED — FP IMPROVED BUT SENSITIVITY COLLAPSED
**Date:** 2026-03-19 (updated after replay reward-gating diagnostic run)
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

---

## 11. Implementation Progress (March 19, 2026)

### 11.1 Code Implementation — COMPLETE (v2: root cause fixes applied)

All planned components have been implemented and tested. After initial fine-tuning
attempts (v1, 11.3 below) showed zero sensitivity improvement, a root cause
investigation (Section 12) identified 4 bugs. These were fixed and validated in
a second implementation pass (v2).

| Component | File | Status | Tests |
|---|---|---|---|
| `EgoKinematics` | `ml/envs/ego_kinematics.py` | ✅ Complete (~70 lines) | 15/15 passing |
| `ReplayTrajectory` | `ml/envs/replay_trajectory.py` | ✅ Complete (~240 lines) | 20/20 passing |
| `ReplayConvoyEnv` | `ml/envs/replay_convoy_env.py` | ✅ v2 with 4 fixes (~350 lines) | **38/38 passing** |
| Gym registration | `ml/envs/__init__.py` | ✅ `RoadSense-ReplayConvoy-v0` registered | — |
| Fine-tuning script | `ml/scripts/run_replay_fine_tuning.py` | ✅ v2 with Monitor + log_std reset | — |
| Canonical data | `ml/data/recordings/{recording_02,extra_driving}/` | ✅ Copied | — |

**Test summary:** 73 unit tests (38 replay env + 15 ego kin + 20 replay traj),
all passing. 389 total unit tests passing (no regressions).

### 11.2 Smoke Tests — PASSED

1. **Model loads on ReplayConvoyEnv:** Run 023 model (`cloud_prod_023`) predicts
   successfully on `ReplayConvoyEnv` with identical observation/action spaces.
2. **50k smoke fine-tuning:** Completed in ~1 minute. PPO training loop runs
   correctly, critic learns (explained_variance 0.42 → 0.90).
3. **Augmentation works:** Synthetic hazard injection triggers `braking_received_decay=1.0`,
   model produces action up to 0.567 in response.

### 11.3 Fine-Tuning Runs — COMPLETED BUT SENSITIVITY DID NOT IMPROVE

#### Run 024-A: LR=1e-5, 500k steps

| Metric | Value |
|---|---|
| Training time | 9.7 minutes (873 FPS) |
| Explained variance | 0.42 → 0.91 |
| Policy std | 0.064 → 0.065 (stable) |
| Clip fraction | 0.02-0.03 (conservative) |

**Replay validation (LR=1e-5):**

| Recording | Sensitivity | FP Rate | vs Run 023 |
|---|---|---|---|
| Recording #2 | **12.0%** (3/25) | 12.13% | 0pp / +0.9pp |
| Extra Driving | **8.7%** (2/23) | 21.23% | 0pp / -0.5pp |

**No change in sensitivity.** Same 3 detected events in Rec#2 (events 11, 12, 16 — close-approach section). Same 2 detected in Extra (events 21-22 — close-approach). All other events still have `model_max_action=0.000`.

#### Run 024-B: LR=3e-5, 500k steps

| Metric | Value |
|---|---|
| Training time | 9.6 minutes |
| Explained variance | 0.42 → 0.93 |
| Policy std | 0.064 → 0.069 (slight drift up, healthy) |
| Clip fraction | 0.03-0.05 (more active than 1e-5) |

**Replay validation (LR=3e-5):**

| Recording | Sensitivity | FP Rate | vs Run 023 |
|---|---|---|---|
| Recording #2 | **12.0%** (3/25) | 14.12% | 0pp / +2.8pp |
| Extra Driving | **8.7%** (2/23) | 21.70% | 0pp / -0.1pp |

**Still no change in sensitivity.** Higher LR produced slightly more FP but zero improvement in detection. The fine-tuning is not modifying the model's behavior on real replay at all.

### 11.4 v1 Fine-Tuning Blocker — ROOT CAUSE INVESTIGATION

See **Section 12** below for the complete root cause investigation. In summary:
4 bugs were identified, all confirmed with code evidence. All 4 were fixed and
tested with 11 new unit tests. The investigation also discovered a 5th issue
(policy std too tight for exploration) and a 6th issue (reward persistence
causing FP explosion).

---

## 12. Root Cause Investigation (March 19, 2026)

### 12.1 External Review (Codex)

The codebase was reviewed by Codex, which identified 4 findings ranked by severity:

1. **HIGH — Ego-state distribution shift.** `ReplayConvoyEnv` uses `EgoKinematics`
   for ego position/speed/accel, while `validate_against_real_data.py` uses the
   recorded TX row directly. The model is fine-tuned on a different observation
   distribution than it sees during validation.
   - Files: `replay_convoy_env.py:153,198`, `ego_kinematics.py:52`,
     `validate_against_real_data.py:119,327`

2. **HIGH — `braking_received` computed before cone filter.** In `ReplayConvoyEnv`,
   braking detection ran on ALL peers before cone filtering. In both `ConvoyEnv`
   (line 561-568) and the validator (line 341-349), braking detection runs on
   cone-filtered peers ONLY. This creates an obs/reward mismatch — braking from
   behind-ego peers triggers the ignoring penalty but those peers are invisible
   in the observation.
   - Files: `replay_convoy_env.py:202` vs `convoy_env.py:561-568`

3. **MEDIUM — Episode slicing biased toward recording start.** `reset()` always
   started from `self._snapshots[0]`, and both recordings start with
   `speed=0.00` (stationary). The fine-tuning over-sampled stationary startup
   instead of mid-convoy hazard contexts.
   - Recording #2: first 67 steps (6.7s) stationary
   - Extra Driving: first 38 steps (3.8s) stationary

4. **LOW — Broken telemetry.** `MetricsCallback` requires `info["episode"]` which
   only exists when the env is wrapped with `Monitor`. The env was NOT wrapped.
   Both training summaries reported `episodes_completed: 0` and
   `final_ep_rew_mean: null`.

### 12.2 Fixes Applied (v2)

| # | Bug | Fix | Test Added |
|---|-----|-----|------------|
| 1 | Ego-state distribution shift | Added `use_recorded_ego=True` mode: ego pos/speed/accel come from TX recording, matching validator exactly. RL action affects reward only (via deceleration), not observations. | `TestRecordedEgoMode` (6 tests) |
| 2 | `braking_received` pre-cone-filter | Moved braking detection AFTER `filter_observable_peers()`. Now matches ConvoyEnv and validator. | `TestBrakingConeFilter` (2 tests) |
| 3 | Episode start bias | Added `random_start=True` and `_pick_start_offset()` — skips stationary startup, randomizes into first half of recording. | `TestStartOffset` (3 tests) |
| 4 | Broken telemetry | Wrapped env with `Monitor()` in `make_env()`. | Verified via `episodes_completed: 1007` in training summary |

**New parameters on `ReplayConvoyEnv.__init__`:**
```python
use_recorded_ego: bool = False   # True → validator-matching obs pipeline
random_start: bool = False       # True → skip stationary, randomize start
```

**New parameter on `run_replay_fine_tuning.py`:**
```python
--reset_log_std FLOAT   # Reset policy log_std before fine-tuning
--use_kinematic_ego     # Use kinematic ego instead of recorded (default: recorded)
```

**Default fine-tuning now uses:** `use_recorded_ego=True`, `random_start=True`.

### 12.3 v2 Fine-Tuning (fixes applied, no exploration reset)

Run 024-v2: LR=3e-5, 500k steps, `use_recorded_ego=True`, `random_start=True`,
Monitor-wrapped, cone-filtered braking.

| Metric | Value |
|---|---|
| Training time | 9.4 min (894 FPS) |
| Episodes completed | **1007** (was 0 in v1!) |
| explained_variance | → 0.711 (critic alive) |
| ep_rew_mean | -4530 → **-2120** (improving) |
| Policy std | 0.064 → 0.067 |

**Replay validation (v2):**

| Recording | Sensitivity | FP Rate | vs Base 023 |
|---|---|---|---|
| Recording #2 | **8.0%** (2/25) | 12.53% | -4pp / +1pp |
| Extra Driving | **8.7%** (2/23) | 21.21% | 0pp / -1pp |

**Result: Fixes improved training health (critic alive, episodes counted,
reward improving), but sensitivity did NOT improve.** The policy std was
too tight (0.064) for the model to explore different actions.

### 12.4 Exploration Analysis — The Tight Policy Problem

Deep diagnostic on the v2 validation timeseries for Recording #2:

```
Steps with braking_received > 0.5: 279 / 1956 (14.3%)
Steps with braking_received > 0.0: 1819 / 1956 (93.0%)

Model action during braking_received > 0.5:
  mean = 0.138, max = 0.982, min = 0.000

Steps with action > 0.1: 226 / 1956 (11.6%)
```

**Event-by-event analysis of braking_received during validation windows:**

| Event | real_accel | brk_recv (mean/max) | model_action (mean/max) | Status |
|-------|-----------|---------------------|------------------------|--------|
| 4 | -3.62 | **1.000/1.000** | 0.000/0.000 | MISSED |
| 5 | -3.92 | 0.332/0.358 | 0.000/0.000 | MISSED |
| 9 | -2.58 | 0.501/0.540 | 0.000/0.000 | MISSED |
| 10 | -2.39 | 0.585/0.630 | 0.000/0.000 | MISSED |
| 12 | -3.70 | **0.626/1.000** | 0.005/0.087 | MISSED |
| 13 | -1.90 | 0.718/0.774 | 0.000/0.000 | MISSED |
| 11 | -1.80 | 0.132/0.142 | 0.218/0.369 | DETECTED |
| 16 | -1.81 | 0.015/0.017 | 0.065/0.246 | DETECTED |

**Critical finding:** Events 4, 9, 10, 12, 13 all have `braking_received > 0.5`
but the model outputs `action=0.000`. The SUMO-trained policy has
`std=0.064`, meaning `P(action > 0.1 | mean=0) = P(z > 1.49) ≈ 6.8%`.
The model literally CAN'T explore actions above the validation threshold.
Fine-tuning with a tight policy can't discover that braking during hazard
gives higher reward because it never tries.

### 12.5 v3 Fine-Tuning (log_std reset for exploration)

Run 024-v3: LR=1e-4, 1M steps, `reset_log_std=-1.0` (std: 0.064 → **0.368**),
all v2 fixes applied.

| Metric | Value |
|---|---|
| Training time | 19.3 min |
| Episodes completed | 2007 |
| ep_rew_mean | -4560 → **-1947** |
| Policy std (before) | 0.064 |
| Policy std (reset to) | **0.368** |

**Replay validation (v3 final model):**

| Recording | Sensitivity | FP Rate | vs Base 023 |
|---|---|---|---|
| Recording #2 | **72.0%** (18/25) | **68.20%** | **+60pp** / +57pp |
| Extra Driving | **34.8%** (8/23) | 23.43% | **+26pp** / +2pp |

**BREAKTHROUGH: Sensitivity jumped from 12% to 72% on Recording #2.**

### 12.6 Checkpoint Sweep — Sensitivity/FP Tradeoff

Validated all v3 checkpoints on Recording #2:

| Checkpoint | Detection | FP | Sensitivity Δ vs Base | FP Δ vs Base |
|---|---|---|---|---|
| Base Run 023 | 12.0% | 11.3% | — | — |
| v3 @ 200k | **24.0%** | 16.3% | +12pp | +5pp |
| v3 @ 400k | **56.0%** | 48.3% | +44pp | +37pp |
| v3 @ 600k | **44.0%** | 53.6% | +32pp | +42pp |
| v3 @ 800k | **64.0%** | 59.9% | +52pp | +49pp |
| v3 @ 1M | **72.0%** | 68.2% | +60pp | +57pp |

Extra recording at 200k checkpoint: 13.0% detection / 21.0% FP (mild improvement).

### 12.7 New Blocker — `braking_received` Persistence Causes FP Explosion

**Root cause of high FP rate:**

In real convoy recordings, `braking_received > 0.01` for **93% of all steps**.
The slow decay (0.95/step, half-life ~1.35s) never fully drains between
frequent normal traffic braking events. The `PENALTY_IGNORING_HAZARD` fires
at threshold `braking_received > 0.01`, creating a persistent penalty of
`-5.0 × braking_received` for NOT braking on 93% of steps.

**Why this didn't happen in SUMO:**
In SUMO training, hazards are injected in discrete windows (steps 150-350).
Between hazards, no peers brake, so `braking_received` decays to ~0 and the
ignoring penalty is dormant for most of the episode. In real convoy data,
normal traffic produces frequent mild braking (peer accel crossing -2.5
intermittently), keeping `braking_received` perpetually above the 0.01
threshold.

**Quantified impact:**
- mean `braking_received` across recording: 0.168
- Average ignoring penalty per step if NOT braking: -5.0 × 0.168 = **-0.84**
- Over 500 steps per episode: cumulative penalty ≈ **-390** for not braking
- This makes always-braking the rational policy, explaining the 68% FP rate

**This is a reward design issue, NOT a bug.** The reward was designed for
SUMO's clean on/off hazard signal, not real-world persistent braking activity.

### 12.8 Current Status & Recommended Next Steps

**What works:**
- All 4 code bugs fixed and tested (38 replay env tests passing)
- Recorded-ego mode eliminates obs distribution shift
- Log_std reset enables exploration → 72% sensitivity achieved (first time ever)
- Infrastructure is solid: Monitor telemetry, random start, checkpointing

**What needs fixing:**
- FP rate is too high (68% at 72% sensitivity, 16% at 24% sensitivity)
- The 200k checkpoint (24% detection / 16% FP) is the best balanced result
  but still below the Section 9.1 target of >40% detection / <15% FP

**Recommended approaches for the next session (pick one):**

1. **Raise ignoring-penalty threshold for replay mode.** Change the condition from
   `braking_received > 0.01` to `braking_received > 0.3` or `0.5` when training
   on replay data. This ensures the penalty only fires during actual strong braking
   events, not the persistent low-level signal. Could be a `ReplayRewardCalculator`
   subclass or a parameter on `RewardCalculator`.

2. **Faster braking decay for replay mode.** Change `BRAKING_DECAY` from 0.95 to
   0.80 in `ReplayConvoyEnv` (half-life 0.3s instead of 1.35s). The signal drains
   faster between events. Downside: must also match in validation script.

3. **Binary braking signal (no decay).** Use `braking_received_mode="instant"` where
   `braking_received = 1.0` only on steps where a peer is currently below -2.5 m/s²,
   0.0 otherwise. No decay tail. The ignoring penalty only fires on steps with
   active V2V braking signal.

4. **Tune log_std reset + training length more carefully.** The 200k checkpoint
   showed 24% / 16% — try `reset_log_std=-1.5` (std=0.22, less aggressive
   exploration) and 300-400k steps. May find a better tradeoff without reward changes.

**Files to modify for approaches 1-3:**
- `ml/envs/replay_convoy_env.py` — braking detection / decay logic
- `ml/envs/reward_calculator.py` (or subclass) — ignoring penalty threshold
- `ml/scripts/run_replay_fine_tuning.py` — new CLI flags
- `ml/scripts/validate_against_real_data.py` — must match if decay changes

**Results directory:**
```
ml/results/
├── run_024_replay/        # v1 (LR=1e-5, pre-fix)
├── run_024_replay_v2/     # v2 (4 fixes, LR=3e-5, no log_std reset)
│   ├── baseline/          # Run 023 baseline validation for comparison
│   └── validation/        # v2 validation reports
└── run_024_replay_v3/     # v3 (4 fixes + log_std=-1.0, LR=1e-4, 1M)
    ├── checkpoints/       # 50k-1M step checkpoints (20 files)
    └── validation/        # v3 validation + checkpoint sweep reports
```

### 12.9 Replay Reward-Gating Fix Implemented (March 19, 2026)

Implemented the approved replay-only fix instead of changing observation
semantics again.

**Design now in code:**
- `ReplayConvoyEnv` defaults to a replay reward config where:
  - `early_reaction_threshold = 0.01` (unchanged, stays loose)
  - `ignoring_hazard_threshold = 0.3`
  - `ignoring_require_danger_geometry = True`
  - `ignoring_danger_distance = 20.0m`
  - `ignoring_danger_closing_rate = 0.5 m/s`
  - `ignoring_use_any_braking_peer = True`
- `RewardCalculator` now supports separate gates for:
  - early-reaction bonus
  - ignoring-hazard penalty
- The tightened ignoring penalty now requires:
  - strong/fresh braking evidence: `braking_received >= 0.3` OR `any_braking_peer`
  - dangerous geometry: `closing_rate > 0.5` OR `distance < 20m`
- Fresh active braking uses `any_braking_peer` to force full signal strength
  (`max(braking_received, 1.0)`) so a new hazard is not missed on the first step.
- `ConvoyEnv` / SUMO behavior is unchanged because `RewardCalculator` defaults
  still preserve the old gating when no replay config is supplied.
- `validate_against_real_data.py` was intentionally NOT changed. This is a
  training-objective fix, not an observation-pipeline fix.

**Files changed:**
- `roadsense-v2v/ml/envs/reward_calculator.py`
- `roadsense-v2v/ml/envs/replay_convoy_env.py`
- `roadsense-v2v/ml/scripts/run_replay_fine_tuning.py`
- `roadsense-v2v/ml/tests/unit/test_reward_calculator.py`
- `roadsense-v2v/ml/tests/unit/test_replay_convoy_env.py`

**Targeted verification completed:**
- `pytest tests/unit/test_reward_calculator.py tests/unit/test_replay_convoy_env.py`
  - Result: **100/100 passed**
- `python -m py_compile envs/reward_calculator.py envs/replay_convoy_env.py scripts/run_replay_fine_tuning.py`
  - Result: passed

**Recommended first diagnostic run:**
```bash
cd /home/amirkhalifa/RoadSense2/roadsense-v2v/ml
source venv/bin/activate

python -m scripts.run_replay_fine_tuning \
  --model_path models/runs/cloud_prod_023/model_final.zip \
  --vecnormalize_path models/runs/cloud_prod_023/vecnormalize.pkl \
  --recordings_dir data/recordings \
  --output_dir results/run_024_replay_v4_reward_gate \
  --total_timesteps 300000 \
  --learning_rate 1e-4 \
  --reset_log_std -1.0 \
  --ignore_hazard_threshold 0.3 \
  --ignore_danger_distance 20.0 \
  --ignore_danger_closing_rate 0.5
```

**Success criteria for this diagnostic:**
- Recording #2: beat the old 200k tradeoff (`24% / 16.3%`)
- Target gate: `>40%` detection and `<15%` FP
- Extra Driving: keep FP at or below the low-20s and preferably under `20%`
- SUMO regression check remains required before any promotion decision

### 12.10 Reward-Gating Diagnostic Result (Run 024-v4, March 19, 2026)

Ran the first replay-only reward-gating diagnostic exactly as planned:
- `300k` steps
- `learning_rate=1e-4`
- `reset_log_std=-1.0`
- recorded ego + random start
- replay reward config:
  - `ignoring_hazard_threshold=0.3`
  - `ignoring_danger_distance=20.0`
  - `ignoring_danger_closing_rate=0.5`
  - `ignoring_use_any_braking_peer=True`
  - `early_reaction_threshold=0.01`

**Training summary:**
- episodes completed: `606`
- final `ep_rew_mean`: `-2029.3`
- training time: `5.4 min`

**Replay validation results (decay mode, unchanged validator):**

| Model | Recording #2 | Extra Driving |
|---|---|---|
| Base Run 023 | 12.0% det / 11.3% FP | 8.7% det / 21.8% FP |
| Run 024-v3 @ 200k | 24.0% det / 16.3% FP | 13.0% det / 21.0% FP |
| Run 024-v3 @ 1M | 72.0% det / 68.2% FP | 34.8% det / 23.4% FP |
| **Run 024-v4 reward gate @ 300k** | **8.0% det / 12.41% FP** | **13.0% det / 19.60% FP** |

**Interpretation:**
- The replay-only reward gate did what it was designed to do on specificity:
  - Recording #2 FP dropped from the v3 frontier (`16.3%-68.2%`) down to `12.41%`
  - Extra Driving FP improved to `19.60%`, the first sub-20% result since this
    replay fine-tuning line began
- But it over-corrected. Recording #2 sensitivity collapsed to `8.0%`, worse than
  the base Run 023 replay (`12.0%`) and far below the v3 200k checkpoint (`24.0%`)
- Extra Driving sensitivity was only `13.0%`, also far below any promotable target

**Conclusion:**
- This exact reward gate is **too strict**. It removes the FP explosion, but it
  also removes the incentive that unlocked the 72% sensitivity breakthrough.
- The fix should be treated as a negative result, not the new default frontier.

**Best next iteration from this result:**
1. Keep the split-gate architecture.
2. Relax the ignoring penalty instead of reverting to the old reward:
   - try `ignoring_hazard_threshold=0.2`
   - try `ignoring_danger_distance=25.0`
   - keep `ignoring_use_any_braking_peer=True`
3. Keep `early_reaction_threshold=0.01` unchanged.
4. Run short sweeps (`200k-300k`) before any longer fine-tune.

### 12.11 Why Geometry Gating Is Structurally Broken — v4 Root Cause (March 19, 2026)

Post-hoc analysis of v4's failure revealed that the problem is **not threshold
strictness**. The geometry gate (`distance < 20 OR closing_rate > 0.5`) is
fundamentally unreliable in `use_recorded_ego=True` mode.

#### The structural issue

In recorded-ego mode, `distance` and `closing_rate` passed to
`RewardCalculator.calculate()` are computed from the **recorded human
trajectory** (see `replay_convoy_env.py:293-296`). During real braking events
in the recording, the human driver braked and maintained safe following distance.
This means:

- `distance` during actual hazards is typically still >20m (human kept distance)
- `closing_rate` during hazards is often <0.5 m/s (human was already decelerating)

The geometry gate `(distance < 20 OR closing_rate > 0.5)` therefore rarely fires
during hazard windows — because **the human's reaction masks the danger
geometry**. The `any_braking_peer` OR-path (6.5% of steps) partially
compensates, but is too sparse alone.

The v4 gate effectively reduced the ignoring penalty's firing rate from ~28%
(brk > 0.3 alone) to perhaps ~5-8% (after geometry filtering), which was too
little training signal.

#### Evidence from timeseries cross-comparison

All measured on Recording #2 (1956 steps, braking_received identical across
models since it derives from the CSV data, not model actions):

| Model | brk>0.3: act>0.1 | calm: act>0.1 | Sensitivity | FP |
|---|---|---|---|---|
| V3 200k (best balanced) | **46.1%** | 2.8% | 24.0% | 16.3% |
| V3 1M (sensitivity peak) | 79.5% | **65.6%** | 72.0% | 68.2% |
| V4 300k (geometry gate) | 34.7% | 2.5% | 8.0% | 12.4% |

Key observations:

1. **V3 200k and V4 have nearly identical calm behavior** (2.8% vs 2.5% action
   rate when `braking_received < 0.01`). Specificity is not the problem.
2. **V4 barely responds to braking signals** (34.7% action rate when
   `braking_received > 0.3`, vs V3 200k's 46.1%). The geometry gate suppressed
   the training signal during actual hazard windows.
3. **V3's specificity collapse (2.8% → 65.6%) happens between 200k and 1M**,
   driven by the ignoring penalty firing on 48.5% of steps. The model gradually
   learns "always brake" to avoid the cumulative penalty.

#### Why a threshold sweep alone won't fix this

Relaxing to `distance < 25` or `closing_rate > 0.3` marginally increases the
geometry gate's firing rate but does not fix the structural mismatch. The human
trajectory still masks danger geometry at moderate thresholds. The gate will
still suppress most hazard-window steps, producing a similar sensitivity collapse.

#### `braking_received` frequency at various thresholds

| Threshold | Steps firing | % of 1956 |
|-----------|-------------|-----------|
| > 0.0 (any nonzero) | 1819 | 93.0% |
| > 0.01 (v3 gate) | 949 | **48.5%** |
| > 0.1 | 564 | 28.8% |
| > 0.15 | ~500 | ~25-28%* |
| > 0.2 | 447 | 22.9% |
| > 0.3 (v4 gate) | 375 | 19.2% |
| > 0.5 | 279 | 14.3% |

\*interpolated

`any_braking_peer` fires on 128/1956 steps (6.5%) — this is the instantaneous
signal before decay.

### 12.12 Run 024-v5 Plan: Threshold-Only Gate + LR Decay (March 19, 2026)

**Approach:** Drop the geometry gate entirely. Use braking_received threshold
only, with LR annealing to prevent the specificity collapse observed in v3.

**Reward config:**
```python
reward_config = {
    "early_reaction_threshold": 0.01,           # unchanged — keep loose for bonus
    "ignoring_hazard_threshold": 0.15,           # fires on ~28% of steps (was 0.01→48.5%, was 0.3→19.2%)
    "ignoring_require_danger_geometry": False,    # DROP — broken in recorded-ego mode
    "ignoring_use_any_braking_peer": True,        # keep — adds 6.5% instantaneous signal
    "ignoring_danger_distance": 20.0,             # unused when geometry=False
    "ignoring_danger_closing_rate": 0.5,          # unused when geometry=False
}
```

**Training config:**
```bash
python -m scripts.run_replay_fine_tuning \
  --model_path models/runs/cloud_prod_023/model_final.zip \
  --vecnormalize_path models/runs/cloud_prod_023/vecnormalize.pkl \
  --recordings_dir data/recordings \
  --output_dir results/run_024_replay_v5_threshold_only \
  --total_timesteps 1000000 \
  --learning_rate 1e-4 \
  --lr_schedule linear \
  --reset_log_std -1.0 \
  --ignore_hazard_threshold 0.15 \
  --no_ignore_danger_geometry
```

**Why this should work:**

1. **Threshold 0.15 reduces penalty frequency by ~40%** (from 48.5% at 0.01 to
   ~28% at 0.15). The 0.01–0.15 range is pure decay tail from normal traffic —
   removing it from the penalty eliminates the strongest driver of
   "always-brake" convergence.

2. **No geometry gate avoids the recorded-ego masking problem.** The penalty
   fires based on the braking signal alone, which is reliable in both
   training and validation.

3. **LR annealing (linear decay 1e-4 → 1e-5)** stabilizes the policy after
   initial learning. V3 200k had 16:1 selectivity (46% action during braking,
   2.8% during calm). The specificity collapsed by 1M at constant LR. Decaying
   LR should preserve discrimination while allowing continued sensitivity gain.

4. **1M steps with checkpoints every 50k** enables a full sweep to find the
   optimal sensitivity/FP tradeoff point.

**Expected outcome:**
- The sensitivity/FP curve should be flatter than v3's (slower FP growth per
  unit of sensitivity) because the penalty stops firing on the low-decay-tail
  steps (0.01–0.15).
- Combined with LR annealing, the sweet spot should shift toward higher
  sensitivity at similar FP — plausibly 35-45% sensitivity at 12-18% FP.
- The 50k checkpoint granularity will reveal the exact tradeoff curve.

**Success criteria:**
- Beat v3 200k (24% det / 16% FP) on Pareto dominance
- Target: >40% detection / <15% FP on Recording #2
- Extra Driving FP should stay below 25%

**Escalation if v5 fails:**
If threshold-only gating still can't break the sensitivity/FP coupling, the next
approach is **shadow kinematic ego for reward geometry**:
- Keep recorded ego for observations (matching validator exactly)
- Run `EgoKinematics` in parallel to compute counterfactual `distance` and
  `closing_rate` — reflecting what WOULD happen if the model's action were
  applied
- Use counterfactual geometry for the reward gate

This fixes the structural issue: the shadow ego's geometry reflects the model's
actions (not the human's), so dangerous geometry appears when the model fails to
brake. Implementation requires running both ego models in `step()` and passing
shadow geometry to `RewardCalculator`.

### 12.13 Run 024-v6 Plan: Shadow Reward Geometry (March 19, 2026)

**Decision:** After reviewing the v5 results against the production goal
(Recording #2 hard-brake response), we are taking **Option B next**, not Option A.
The reason is pragmatic: v5 already showed the threshold-only reward still hits
the same sensitivity/FP coupling, while the earlier H5 analysis showed the
policy naturally responds to geometry once it becomes causally meaningful.

**Implementation completed locally:**
- `ReplayConvoyEnv` now accepts `use_shadow_reward_geometry=True`
- When enabled with `use_recorded_ego=True`:
  - observations still use the recorded ego state exactly
  - reward `distance`, `closing_rate`, and `ego_speed` come from a shadow
    `EgoKinematics` rollout driven by the policy action
  - info telemetry now includes both recorded and reward geometry:
    `recorded_distance`, `reward_distance`, `recorded_closing_rate`,
    `reward_closing_rate`, `reward_geometry_source`
- `run_replay_fine_tuning.py` now exposes `--use_shadow_reward_geometry`
- Regression tests verify:
  - recorded ego observations stay unchanged under shadow mode
  - the ignoring penalty changes when the shadow ego brakes vs does not brake

**300k probe config (selected):**
```bash
cd /home/amirkhalifa/RoadSense2/roadsense-v2v
source ml/venv/bin/activate
python -m ml.scripts.run_replay_fine_tuning \
  --model_path ml/models/runs/cloud_prod_023/model_final.zip \
  --vecnormalize_path ml/models/runs/cloud_prod_023/vecnormalize.pkl \
  --recordings_dir ml/data/recordings \
  --output_dir ml/results/run_024_replay_v6_shadow_geometry_300k \
  --total_timesteps 300000 \
  --learning_rate 1e-4 \
  --final_learning_rate 1e-5 \
  --reset_log_std -1.0 \
  --ignore_hazard_threshold 0.15 \
  --ignore_require_danger_geometry \
  --use_shadow_reward_geometry
```

**Why this v6 config is the right next probe:**
1. Keeps the deployed observation contract unchanged.
2. Restores geometry as a causal reward signal instead of a human-masked one.
3. Retains v5's best FP-control tool (`LR` decay).
4. Uses a short 300k run as a cheap proof test before another long sweep.

**Results directory:**
```
ml/results/
├── run_024_replay/                    # v1 (LR=1e-5, pre-fix)
├── run_024_replay_v2/                 # v2 (4 fixes, LR=3e-5, no log_std reset)
├── run_024_replay_v3/                 # v3 (4 fixes + log_std=-1.0, LR=1e-4, 1M)
├── run_024_replay_v4_reward_gate/     # v4 (split-gate: brk≥0.3 + geometry — TOO STRICT)
├── run_024_replay_v5_threshold_only/  # v5 (threshold 0.15, no geometry, LR decay)
└── run_024_replay_v6_shadow_geometry_300k/  # v6 probe (shadow reward geometry)
```

**Outcome note (same day):**
- The v6 300k probe was run after implementation.
- Result: **FAILED** — best Recording #2 checkpoint was only `16.0% / 15.5%`
  at `150k`; final `300k` regressed to `0.0% / 4.9%`.
- The strongest real brake in Recording #2 (`-8.63 m/s²`) still did not produce
  a meaningful model response: best checkpoint `150k` peaked at action `0.147`
  (`~1.17 m/s²`), final `300k` was `0.0`.
- Full details captured in:
  `docs/10_PLANS_ACTIVE/RUN_024_V6_SHADOW_GEOMETRY_300K_RESULTS.md`
