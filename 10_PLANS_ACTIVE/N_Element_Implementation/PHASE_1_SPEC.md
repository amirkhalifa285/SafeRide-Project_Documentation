# Phase 1 Specification: ConvoyEnv & N-Element Observations

**Document:** `PHASE_1_SPEC.md`
**Status:** 🟢 READY FOR IMPLEMENTATION
**Target Agent:** BUILDER
**Architecture Reference:** `00_ARCHITECTURE/DEEP_SETS_N_ELEMENT_ARCHITECTURE.md`

---

## 1. Objective
Transform `ConvoyEnv` from a fixed 11-dimensional observation space (hardcoded for 2 peers) to a `Dict` observation space supporting 0 to 8 peers using the Deep Sets data format.

---

## 2. Technical Specification

### 2.1 Observation Space Definition
The new observation space MUST be a `gym.spaces.Dict` with the following keys:

- **`ego`**: `Box(low=[0, -1, -1, 0], high=[1, 1, 1, 1], shape=(4,), dtype=np.float32)`
  - `[speed/30, accel/10, heading/pi, peer_count/8]`
- **`peers`**: `Box(low=-inf, high=inf, shape=(8, 6), dtype=np.float32)`
  - Each of the 8 slots contains: `[rel_x/100, rel_y/100, rel_speed/30, rel_heading/pi, accel/10, age_ms/500]`
- **`peer_mask`**: `Box(low=0, high=1, shape=(8,), dtype=np.float32)`
  - `1.0` if peer slot is valid, `0.0` if padded.

### 2.2 Component Modifications

#### A. `ml/envs/observation_builder.py`
- **Method `build()`**: Should now accept `(ego_state, peer_observations, ego_pos)`.
- **Logic**:
  1. Initialize `peers` array with zeros `(8, 6)`.
  2. Initialize `peer_mask` with zeros `(8,)`.
  3. Loop through `peer_observations` (up to 8).
  4. For each: Calculate relative coordinates, normalize, and set mask to `1.0`.
  5. Return the Dict.

#### B. `ml/envs/convoy_env.py`
- **`__init__`**: Update `self.observation_space` to the Dict structure.
- **`_step_espnow()`**:
  - Replace hardcoded `V002`, `V003` logic.
  - Use `traci.vehicle.getIDList()` to find all vehicles.
  - Exclude `self.ego_id`.
  - For each peer, call `self.espnow.transmit()` to get the emulated message.
  - Collect all successfully received messages into a list and pass to `obs_builder`.

---

## 3. TDD Requirements (BUILDER Must Follow)

### Step 1: Create `ml/tests/unit/test_n_element_env.py`
Write tests that:
1. Instantiate `RoadSense-Convoy-v0`.
2. Assert `env.observation_space` is a `Dict`.
3. Call `env.reset()` and assert it returns a dict with `ego`, `peers`, and `peer_mask`.
4. Mock `traci` to return 0 peers -> Assert `peer_mask` is all zeros.
5. Mock `traci` to return 3 peers -> Assert `peer_mask` has exactly three `1.0` values.

### Step 2: Implementation
Implement changes in `observation_builder.py` and `convoy_env.py` until tests pass.

---

## 4. Files Touched
- `roadsense-v2v/ml/envs/convoy_env.py`
- `roadsense-v2v/ml/envs/observation_builder.py`
- `roadsense-v2v/ml/tests/unit/test_n_element_env.py` (New)

---

## 5. Risks
- **Normalizing Zeros**: Ensure `peer_count/8` doesn't crash if `MAX_PEERS` changes.
- **Coordinate Frames**: Double-check rotation logic in `ObservationBuilder` to ensure it is ego-centric.
