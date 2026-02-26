# CRITICAL Architecture Correction: Mesh Relay + Continuous Actions

**Created:** February 26, 2026
**Status:** ACTIVE — MUST BE COMPLETED BEFORE NEXT TRAINING RUN
**Priority:** HIGHEST — Blocks all future model training
**Origin:** Professor meeting feedback, February 26, 2026
**TDD:** MANDATORY — Every change must follow Red-Green-Refactor

---

## Executive Summary

Two fundamental architecture flaws were identified during professor review:

1. **MESH IS NOT IMPLEMENTED.** The V2VMessage struct has `hopCount` and `sourceMAC` fields. PackageManager deduplicates by `sourceMAC`. But NO component actually relays messages. Every vehicle only broadcasts its own data. The emulator and ConvoyEnv simulate direct point-to-point, not multi-hop mesh. This must be fixed in firmware, emulator, and simulation.

2. **ACTION SPACE IS WRONG.** The discrete 4-action space (MAINTAIN/CAUTION/BRAKE/EMERGENCY with hardcoded deceleration values) defeats the purpose of RL. The model should LEARN the percentage of deceleration needed to avoid collisions. The action space must be continuous: a single float in [0.0, 1.0] representing the fraction of maximum deceleration to apply. That's the whole point of PPO.

3. **CONE FILTERING MISSING.** Already documented elsewhere but now even more critical: cone filtering determines what each vehicle REBROADCASTS in the mesh, not just what the ego observes.

---

## How Mesh SHOULD Work (Professor's Description)

### Convoy: A (front) → B (middle) → C (ego, rear)

```
Timestep T:

  A broadcasts:
    - A's own state (hop=0, source=A)

  B receives A's broadcast:
    - Cone filter: Is A in front of B? → YES → mark for relay
  B broadcasts:
    - B's own state (hop=0, source=B)
    - A's state RELAYED (hop=1, source=A)

  C (ego) receives from B:
    - B's own state (hop=0) → direct
    - A's state (hop=1) → relayed through B

  C runs inference on: {A (1 hop), B (direct)}
```

### Key Principles

1. **Each vehicle broadcasts its own data AND rebroadcasts data from vehicles in its front cone.**
2. **Cone filtering determines what gets relayed**, not just what gets observed.
3. **Ego receives everything through the mesh** — direct messages from immediate neighbors PLUS relayed messages from further vehicles.
4. **Recording only needs the ego vehicle** — its RX log captures all data (direct + relayed).
5. **Deduplication by sourceMAC + timestamp** prevents duplicate processing when the same message arrives via multiple paths. PackageManager already does this correctly.

### Cone Filter Logic (GPS-Based)

```
is_in_cone(ego_heading, ego_pos, peer_pos, half_angle=45°):
    bearing = atan2(peer.y - ego.y, peer.x - ego.x)
    diff = normalize_angle(bearing - ego_heading)
    return abs(diff) <= half_angle
```

- Uses GPS positions and heading
- Default half-angle: 45° (total cone: 90° forward)
- Vehicles behind are filtered OUT and NOT relayed

---

## How Actions SHOULD Work (Professor's Correction)

### Current (WRONG)

```python
action_space = Discrete(4)
# 0 = MAINTAIN  → 0.0 m/s² decel
# 1 = CAUTION   → 0.5 m/s² decel
# 2 = BRAKE     → 2.0 m/s² decel
# 3 = EMERGENCY → 4.5 m/s² decel
```

Problem: The model picks from 4 preset values. It never learns the nuance of "I need exactly 37% braking here." The whole point of RL/PPO is to learn the continuous control policy.

### Correct

```python
action_space = Box(low=0.0, high=1.0, shape=(1,), dtype=float32)
# 0.0 = no deceleration (maintain speed)
# 0.5 = 50% of max deceleration
# 1.0 = 100% of max deceleration (emergency stop)

MAX_DECEL = 5.0  # m/s² (physical max for the vehicle)
actual_decel = action[0] * MAX_DECEL
new_speed = max(0.0, current_speed - actual_decel * dt)
```

The model learns: "given this traffic situation, I should decelerate by X%." PPO handles continuous action spaces natively.

---

## Implementation Plan

### Phase A: Cone Filter (SMALLEST, DO FIRST)

**Files to modify:**
- `ml/envs/observation_builder.py` — add `is_in_cone()` method and filter in `build()`
- `ml/tests/unit/test_cone_filter.py` — NEW file

**Tests (write FIRST):**
```
test_cone_filter_peer_directly_ahead_is_in_cone
test_cone_filter_peer_directly_behind_is_out_of_cone
test_cone_filter_peer_at_edge_45_deg_is_in_cone
test_cone_filter_peer_at_46_deg_is_out_of_cone
test_cone_filter_peer_count_reduced_after_filtering
test_cone_filter_no_peers_returns_empty
test_cone_filter_all_peers_behind_returns_empty
test_cone_filter_heading_wraps_around_360
```

**Implementation (~15 lines):**
```python
def is_in_cone(self, ego_heading_rad, ego_pos, peer_x, peer_y, half_angle_deg=45.0):
    dx = peer_x - ego_pos[0]
    dy = peer_y - ego_pos[1]
    bearing = math.atan2(dy, dx)
    diff = (bearing - ego_heading_rad + math.pi) % (2 * math.pi) - math.pi
    return abs(diff) <= math.radians(half_angle_deg)
```

Filter peers in `build()` before the processing loop — skip any peer where `is_in_cone()` returns False.

**Exit criteria:**
- All new tests pass
- All existing observation_builder tests still pass
- Peers behind ego are excluded from observation

---

### Phase B: Continuous Action Space

**Files to modify:**
- `ml/envs/action_applicator.py` — replace discrete with continuous
- `ml/envs/convoy_env.py` — change `action_space` from `Discrete(4)` to `Box(1,)`
- `ml/envs/reward_calculator.py` — update reward to work with continuous decel value
- `ml/models/deep_set_extractor.py` — verify compatible (should be fine)
- `ml/training/train_convoy.py` — PPO already supports continuous; may need hyperparameter tweaks

**Tests to UPDATE (existing tests will break — this is expected):**
```
# action_applicator tests — rewrite for continuous
test_action_applicator_zero_means_no_decel
test_action_applicator_one_means_max_decel
test_action_applicator_half_means_half_max_decel
test_action_applicator_clips_below_zero
test_action_applicator_clips_above_one
test_action_applicator_speed_floor_is_zero
test_action_applicator_returns_actual_decel_applied

# convoy_env tests — update action space checks
test_action_space_is_box_1
test_step_accepts_float_action
test_step_accepts_numpy_array_action

# reward_calculator tests — update for continuous decel input
test_reward_comfort_penalty_scales_with_decel_magnitude
test_reward_no_penalty_for_gentle_decel
test_reward_high_penalty_for_max_decel_when_unnecessary
```

**Implementation:**
```python
# action_applicator.py
MAX_DECEL = 5.0  # m/s² — physical vehicle limit

def apply(self, sumo, action_value: float) -> float:
    # Clip to [0, 1]
    fraction = max(0.0, min(1.0, float(action_value)))
    decel = fraction * self.MAX_DECEL
    current_speed = sumo.get_vehicle_state(self.EGO_VEHICLE_ID).speed
    new_speed = max(0.0, current_speed - decel * 0.1)  # 0.1s timestep
    sumo.set_vehicle_speed(self.EGO_VEHICLE_ID, new_speed)
    return current_speed - new_speed
```

```python
# convoy_env.py
self.action_space = spaces.Box(
    low=np.array([0.0], dtype=np.float32),
    high=np.array([1.0], dtype=np.float32),
    dtype=np.float32,
)
```

**Exit criteria:**
- `action_space` is `Box(1,)` not `Discrete(4)`
- PPO training starts without error
- `check_env()` passes
- Reward function handles continuous decel values
- All updated tests pass

---

### Phase C: Mesh Relay in Emulator

**Files to modify:**
- `ml/espnow_emulator/espnow_emulator.py` — add multi-hop relay simulation
- `ml/tests/unit/test_espnow_emulator.py` — add mesh relay tests

**New Python V2VMessage fields:**
```python
@dataclass(frozen=True)
class V2VMessage:
    # ... existing fields ...
    hop_count: int = 0
    source_id: str = ""  # original sender (empty = self)
```

**New emulator method:**
```python
def simulate_mesh_step(
    self,
    vehicle_states: Dict[str, VehicleState],
    ego_id: str,
    current_time_ms: int,
    cone_half_angle_deg: float = 45.0,
) -> Dict[str, ReceivedMessage]:
    """
    Simulate one full mesh broadcast cycle for all vehicles.

    1. Each vehicle broadcasts its own state
    2. Each vehicle receives from others in ESP-NOW range
    3. Each vehicle applies cone filter on received messages
    4. Each vehicle rebroadcasts front-cone messages (hop+1)
    5. Return what ego received (direct + relayed)
    """
```

**Tests (write FIRST):**
```
test_mesh_direct_message_has_hop_count_zero
test_mesh_relayed_message_has_hop_count_one
test_mesh_ego_receives_relayed_message_from_out_of_range_vehicle
test_mesh_behind_vehicle_not_relayed (cone filter blocks relay)
test_mesh_duplicate_message_deduplicated_by_source_id
test_mesh_relay_adds_latency_per_hop
test_mesh_three_vehicle_convoy_ego_gets_both_peers
test_mesh_relay_respects_esp_now_range_limit
test_mesh_no_relay_when_all_peers_behind
```

**Exit criteria:**
- Ego receives messages from vehicles beyond direct range (via relay)
- Relayed messages have correct hop count
- Cone filter prevents backward relay
- Duplicate messages from multiple paths are deduplicated
- Per-hop latency is additive
- All existing emulator tests still pass

---

### Phase D: Mesh in ConvoyEnv

**Files to modify:**
- `ml/envs/convoy_env.py` — replace `_step_espnow()` with mesh-aware version

**Current `_step_espnow()` (WRONG):**
```python
for vehicle_id in traci.vehicle.getIDList():
    if vehicle_id == self.EGO_VEHICLE_ID:
        continue
    # transmit directly to ego — NO RELAY
```

**New `_step_espnow()`:**
```python
# Get ALL vehicle states from SUMO
all_states = {vid: self.sumo.get_vehicle_state(vid)
              for vid in traci.vehicle.getIDList()}

# Delegate to emulator's mesh simulation
received = self.emulator.simulate_mesh_step(
    vehicle_states=all_states,
    ego_id=self.EGO_VEHICLE_ID,
    current_time_ms=current_time_ms,
)

# Build observation from received messages (already cone-filtered by emulator)
```

**Tests (write FIRST):**
```
test_convoy_env_ego_receives_relayed_messages
test_convoy_env_mesh_step_returns_correct_peer_count
test_convoy_env_observation_includes_relayed_peers
test_convoy_env_hop_count_reflected_in_message_age
```

**Exit criteria:**
- ConvoyEnv uses `simulate_mesh_step()` instead of per-vehicle `transmit()`
- Observation includes both direct and relayed peers
- Full episode completes without error
- `check_env()` passes
- All existing ConvoyEnv tests pass or are updated

---

### Phase E: Firmware Mesh Relay

**Files to modify:**
- `hardware/src/main.cpp` — wire PackageManager, add rebroadcast logic
- `hardware/src/network/mesh/PackageManager.cpp` — already mesh-ready (no changes expected)
- `hardware/src/inference/ConeFilter.h` — NEW: C++ cone filter
- `hardware/src/inference/ConeFilter.cpp` — NEW
- `hardware/test/test_cone_filter/` — NEW: native C++ tests

**Changes to `main.cpp`:**
1. Add `PackageManager packageManager;` as global
2. In `onDataReceived()`: call `packageManager.addPackage(mac, msg, msg.hopCount)`
3. In broadcast loop: after sending own message, iterate PackageManager for front-cone peers and rebroadcast their latest message with `hopCount++`
4. Add `isInCone()` function using GPS heading and position

**Tests:**
```
test_cone_filter_cpp_peer_ahead_returns_true
test_cone_filter_cpp_peer_behind_returns_false
test_cone_filter_cpp_matches_python_implementation
test_rebroadcast_increments_hop_count
test_rebroadcast_only_front_cone_peers
test_package_manager_called_on_receive
```

**Exit criteria:**
- PackageManager is actively used in main loop
- Received front-cone messages are rebroadcasted
- Hop count incremented on relay
- C++ cone filter matches Python implementation exactly
- All firmware tests pass

---

### Phase F: Recording Strategy Update

**Professor's instruction:** Only record from the ego vehicle. Through mesh, ego's RX log contains everything.

**Augmentation strategy:**
1. Record real convoy (A, B, C) where C is ego
2. C's RX data contains: B's state (direct) + A's state (relayed via B)
3. To augment: add virtual vehicle D behind C
4. D's input = C's relayed data (C's own state + what C relayed from A and B)
5. This creates a 4-vehicle scenario from a 3-vehicle recording

**No code changes needed** — this is a recording protocol and data processing change. Document in `DATA_RECORDING_STRATEGY.md`.

---

## Dependency Graph

```
Phase A: Cone Filter ──────────────────────────┐
    (no dependencies, do first)                 │
                                                ▼
Phase B: Continuous Actions ──────► Phase D: ConvoyEnv Mesh
    (independent of mesh)                │
                                         │
Phase C: Emulator Mesh ────────────────►─┘
    (depends on Phase A for cone filter)
                                         │
                                         ▼
                                  RETRAIN (Run 003)
                                         │
Phase E: Firmware Mesh ──────────────────┤
    (can happen in parallel               │
     with simulation work)                ▼
                                  Phase F: Recording #2
                                  (use mesh-aware firmware)
```

**Critical path:** A → C → D → Retrain
**Parallel path:** B (can happen alongside A/C)
**Parallel path:** E (firmware, can happen alongside simulation work)

---

## TDD Protocol (NON-NEGOTIABLE)

Every phase follows Red-Green-Refactor:

1. **RED:** Write failing test FIRST. Run it. It MUST fail. It must fail for the RIGHT reason.
2. **GREEN:** Write MINIMUM code to pass. No extras.
3. **REFACTOR:** Clean up. Run tests. If they break, revert.
4. **COMMIT:** Only when ALL tests green.

### Rules

- **No code without a test.** If there's no test for it, it doesn't get written.
- **Never modify a test to make it pass.** Fix the implementation.
- **Existing tests that break are expected** (action space change). Update them to match new spec, then make the implementation pass the new tests.
- **Run the full test suite before every commit.** No regressions.

### Test Locations

| Component | Test File | Type |
|-----------|-----------|------|
| Cone filter (Python) | `ml/tests/unit/test_cone_filter.py` | Unit |
| Action applicator | `ml/tests/unit/test_action_applicator.py` | Unit (update existing) |
| Emulator mesh | `ml/tests/unit/test_espnow_emulator.py` | Unit (add mesh tests) |
| ConvoyEnv mesh | `ml/tests/unit/test_convoy_env.py` | Unit (update) |
| ConvoyEnv integration | `ml/tests/integration/test_convoy_env_mesh.py` | Integration (new) |
| Cone filter (C++) | `hardware/test/test_cone_filter/` | Native (new) |

---

## What This Means for Run 002

Run 002 is **not the production model.** It was trained with:
- Direct broadcast (no mesh relay)
- No cone filtering
- Discrete 4-action space (hardcoded deceleration values)

Run 002 remains a **baseline** for comparison only. The next training run (Run 003) must include all corrections from this document.

---

## Open Questions for Professors (Next Meeting)

1. **Cone half-angle:** Is ±45° (90° total) correct, or should it be wider/narrower?
2. **Max hop count:** Should we limit relay depth (e.g., max 3 hops) to prevent stale data flooding?
3. **MAX_DECEL value:** Is 5.0 m/s² the right physical limit for the vehicle model?
4. **Action output:** Single float [0,1] for decel fraction, or should we also include a steering component?

---

**Document Version:** 1.0
**Author:** Amir Khalifa + Claude
**Next Review:** After Phase A and Phase B implementation
