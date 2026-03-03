# Run 004 Plan: Hazard Targeting, Reward Realism, and Source-Specific Evaluation

**Date:** March 2, 2026
**Updated:** March 2, 2026 (Progress update after implementation + review fixes)
**Owner:** Amir + Architect Agent + Codex
**Priority:** CRITICAL (Pre-Run-004 Blocker)
**Status:** IN PROGRESS — H0/H1/H2/H4 implemented and unit tested; H3/H5 pending
**TDD:** MANDATORY — Every change follows Red-Green-Refactor

---

## 1) Problem Statement

Current pipeline has mesh relay, continuous actions, and Deep Sets observation handling, but five gaps remain:

1. **Hazard source coverage gap:** hazard injection targets nearest peer only, causing source bias.
2. **Evaluation observability gap:** no pass/fail metrics for "did ego react correctly when source vehicle X braked?"
3. **Reward uses SUMO ground truth, not mesh-visible state:** `_calculate_min_distance_and_closing_rate()` iterates ALL SUMO peers, not just what ego received through the mesh. This means ego gets punished for not reacting to peers it cannot see.
4. **MAX_DECEL is wrong:** set to 5.0 m/s² but real-life convoy braking recorded -8.63 m/s². The model's "100% braking" is only ~58% of actual capability.
5. **No sim-to-real validation:** no mechanism to replay real recording data through the trained model to verify behavior before field deployment.

Without fixing ALL of these, the model cannot be trusted in a real-life test.

---

## 2) Objective

For `n=1..5` peer scenarios, demonstrate that ego (`V001`) can react appropriately to braking hazards originating from different upstream vehicles (not only nearest), using mesh-received information under realistic loss/latency, with reward signals that match what the model can actually observe.

Validate the trained model offline against real convoy recording data before field deployment.

---

## 2.1) Implementation Progress (March 2, 2026)

### Completed in code + tests
- ✅ **H0 complete:** `MAX_DECEL` updated to `8.0` with updated action applicator tests.
- ✅ **H4 complete:** reward distance/closing now use **mesh-visible peers only** from `_step_espnow()`.
- ✅ **H1 complete:** `HazardInjector` now supports:
  - `nearest`
  - `uniform_front_peers`
  - `fixed_vehicle_id`
  - `fixed_rank_ahead`
  - per-episode reset overrides (`target_strategy`, fixed source/rank, forced hazard, forced hazard step).
- ✅ **H2 complete:** source-specific evaluation instrumentation added to both:
  - `ml/training/train_convoy.py`
  - `ml/scripts/evaluate_model.py`
  Includes per-episode hazard/reaction fields and aggregated `source_reaction_summary`.

### Architect-review bug fixes integrated (H2)
- ✅ Fixed reception-gate bug:
  - `hazard_message_received_by_ego` no longer incorrectly gated by `not reaction_detected`.
- ✅ Fixed reaction-start bug:
  - reaction tracking now starts only **after actual hazard injection** (`hazard_injected=True`), not just scheduled `hazard_step`.
- ✅ Fixed schema mismatch:
  - `evaluate_model.py` `source_reaction_summary` now includes `braking_signal_reception_rate` to match training-eval schema.

### Minor cleanup completed
- ✅ Removed dead `hazard_source_received_this_step` field from env `info`.
- ✅ Reduced `HazardInjector.__init__` redundancy (fields no longer pre-set then overwritten by `_reset_state()`).
- ✅ Documented intended semantics: reaction metric tracks **RL braking command**, not passive car-following decel while released.

### Verification status
- ✅ Full unit suite passed in `ml` venv:
  - `pytest tests/unit/ -q` -> `187 passed`
- ✅ New/updated test coverage added for:
  - `evaluate_model` source summary schema
  - `fixed_vehicle_id` behind-ego skip behavior
  - hazard instrumentation loop behavior in `train_convoy` eval (covers reception + reaction timing gates)

---

## 3) Required Architecture Outcomes

### O1) Training hazard source diversity
Hazard events must be distributed across front peers (not nearest-only dominance).

### O2) Deterministic source-specific eval
Evaluation must include controlled episodes where hazard source ID is explicitly selected.

### O3) Reaction-quality reporting
Metrics must quantify reaction timing and safety quality per hazard source and peer-count, distinguishing "model didn't react" from "model never received the hazard."

### O4) Reward-observation consistency
Reward signal must only penalize/reward ego based on peers it can observe through the mesh, not SUMO omniscient state.

### O5) Physically accurate action limits
MAX_DECEL must match measured real-world vehicle capability (8.0 m/s²).

### O6) Sim-to-real confidence
Trained model must be validated against real recording data before field deployment.

---

## 4) Implementation Plan

### Phase H0 — MAX_DECEL Correction (DO FIRST — trivial)

**Problem:**
`ActionApplicator.MAX_DECEL = 5.0` but real convoy braking measured -8.63 m/s² raw. The model cannot perform a real emergency stop.

**Change:**
```python
# action_applicator.py
MAX_DECEL = 8.0  # m/s² — measured real-world emergency braking
```

**Comfort thresholds remain unchanged** — they are human-comfort limits, not vehicle limits:
- `GENTLE_DECEL_THRESHOLD = 0.5` (no penalty below this)
- `UNCOMFORTABLE_BRAKE_THRESHOLD = 3.0` (gradual penalty)
- `HARSH_BRAKE_THRESHOLD = 4.5` (max penalty -10)
- Above 4.5 m/s²: flat -10 penalty — available for true emergencies but always penalized

This means the model has access to 4.5–8.0 m/s² (the "emergency zone") but pays a steep comfort cost. It should only use this range when the safety reward (+100 avoided collision) outweighs the comfort penalty (-10).

**Files:**
- `roadsense-v2v/ml/envs/action_applicator.py`
- `roadsense-v2v/ml/tests/unit/test_action_applicator.py`

**Tests (update existing):**
```
test_action_applicator_one_means_max_decel  → assert MAX_DECEL == 8.0
test_action_applicator_half_means_half_max_decel → assert 4.0 m/s²
test_action_applicator_decel_at_full_action → verify 8.0 * 0.1 = 0.8 m/s speed drop per step
```

**Exit criteria:**
- `MAX_DECEL == 8.0` in code and all tests
- All existing action applicator tests pass with updated constant
- All reward calculator tests still pass (thresholds unchanged)

---

### Phase H1 — Hazard Injector Source-Selection Upgrade

### Scope
Enhance `HazardInjector` to support source-selection strategies.

### Proposed API additions
- `target_strategy: str` with allowed values:
  - `nearest` (backward-compatible current behavior)
  - `uniform_front_peers` (training default)
  - `fixed_vehicle_id` (deterministic eval)
  - `fixed_rank_ahead` (deterministic by convoy order)
- Optional per-episode override via `reset(options=...)` or env-level options.

### Required outputs in step info
- `hazard_injected` (existing)
- `hazard_source_id` (new)
- `hazard_step` (new)
- `hazard_source_rank_ahead` (new, optional)

### Chain Reaction Behavior (IMPORTANT — no code change needed)

When the hazard injector pins one vehicle to 0 m/s, SUMO's Krauss car-following model automatically cascades braking through all vehicles behind the target. This means:

```
T=0:  V006 (lead) → injector pins to 0 m/s
T+1:  V005 sees V006 stopped → CF model brakes V005
T+2:  V004 sees V005 braking → CF model brakes V004
...
T+4:  V002 brakes naturally → mesh delivers V002's braking data to ego
```

Ego receives **multiple independent braking signals** from intermediate vehicles, not just the original source's message. This makes far-source hazards observable even under packet loss — the chain reaction creates redundant signals.

**No code change needed** — SUMO CF model handles this natively. Peer vehicles are NOT under manual speed control (only ego and the hazard target are).

### Files
- `roadsense-v2v/ml/envs/hazard_injector.py`
- `roadsense-v2v/ml/envs/convoy_env.py`
- `roadsense-v2v/ml/tests/unit/test_hazard_injector.py`
- `roadsense-v2v/ml/tests/unit/test_convoy_env.py`

### Acceptance tests
- `uniform_front_peers` samples across eligible front peers over many episodes.
- `fixed_vehicle_id` always targets requested source when active.
- Invalid fixed source (missing/inactive/behind) handled deterministically (skip or fallback according to policy).

---

### Phase H2 — Source-Specific Evaluation Instrumentation

### Scope
Add explicit reaction metrics tied to hazard source.

### Reaction Detection Threshold

```python
REACTION_DECEL_THRESHOLD = 0.5  # m/s² (6.25% of MAX_DECEL)
```

Rationale: below 0.5 m/s² could be noise or CF model drift. Above 0.5 is intentional braking by the model.

### Episode-level metrics (minimum)
- `hazard_source_id`
- `hazard_step`
- `hazard_message_received_by_ego` (bool — **CRITICAL**: separates "didn't react" from "never received")
- `reaction_detected` (bool)
- `reaction_time_s` (from hazard step to first ego decel > REACTION_DECEL_THRESHOLD)
- `min_distance_post_hazard_m`
- `collision_post_hazard` (bool)
- `pct_time_unsafe_post_hazard`

### How `hazard_message_received_by_ego` works

After hazard injection, check whether the hazard source vehicle's ID appears in `received_map` (the emulator's mesh output) at any step between `hazard_step` and ego's first reaction (or episode end if no reaction). If the source was never mesh-visible to ego, the episode is categorized as "no reception" — distinct from "reception but no reaction."

Additionally, check whether **any peer's braking state** was received (chain reaction data). An episode where ego saw intermediate vehicles braking but not the original source is still informative.

### Aggregated summaries
- `source_reaction_summary` keyed by:
  - peer count (`n`)
  - source ID or source rank ahead

Each summary bucket should include:
- episodes
- reception_rate (fraction where hazard source was mesh-visible to ego)
- reaction_rate (fraction where ego reacted, conditional on reception)
- avg reaction time
- collision rate
- avg min-distance-post-hazard

### Files
- `roadsense-v2v/ml/training/train_convoy.py`
- `roadsense-v2v/ml/scripts/evaluate_model.py`
- `roadsense-v2v/ml/tests/unit/test_train_convoy.py`

---

### Phase H3 — Deterministic Eval Matrix (n and source coverage)

### Requirement
Evaluation set must include hazards from multiple source positions for each peer-count.

### Minimum matrix
For each `n in {1,2,3,4,5}`:
- Hazards injected at each valid source rank among front peers
- At least 10 episodes per `(n, source-rank)` bucket (or architect-approved minimum)

### Pass criteria
- No missing `(n, source-rank)` buckets in final report
- Reaction metrics emitted for all buckets
- `hazard_message_received_by_ego` populated for all episodes

---

### Phase H4 — Reward Mesh-Visibility Gating (NEW)

### Problem

`convoy_env.py:_step_espnow()` returns `peer_states` built from ALL SUMO vehicles (line 256-259), NOT from the mesh. The reward calculator then uses this ground-truth distance. This creates a training signal mismatch:

- Model is **penalized** for not reacting to a peer it **cannot see** (dropped by packet loss, out of mesh range)
- Model is **rewarded** based on distances to vehicles that may not be in its observation

In real life, the model only has mesh data. The reward must match.

### Change

Replace the ground-truth `peer_states` in the reward distance calculation with **mesh-visible peer states** derived from `received_map`.

```python
# convoy_env.py — _step_espnow() return value change
# BEFORE (line 256-259):
peer_states = [
    state for vehicle_id, state in vehicle_states.items()
    if vehicle_id != self.EGO_VEHICLE_ID
]

# AFTER:
# Build mesh-visible peer states from received_map
mesh_visible_peers = []
for source_id, received in received_map.items():
    msg = received.message
    age_ms = float(received.age_ms)
    if age_ms < staleness_threshold:
        mesh_visible_peers.append(VehicleState(
            x=msg.lon * meters_per_deg,
            y=msg.lat * meters_per_deg,
            speed=msg.speed,
            heading=msg.heading,
        ))

# Return mesh-visible peers for reward calculation
return observation, mesh_visible_peers
```

When no peers are mesh-visible (total packet loss), distance defaults to 1000.0 (existing fallback in `_calculate_min_distance_and_closing_rate`). This means: no visible threat → no safety penalty → model learns that "no data = maintain speed."

### Safety argument

This is CORRECT behavior for real-world deployment. In real life, if the mesh delivers no data, ego has no information to act on. Penalizing it for not reacting to invisible threats teaches the model to brake randomly, which is worse than maintaining speed.

### Tests

```
test_reward_distance_uses_mesh_visible_peers_not_ground_truth
test_reward_no_penalty_when_peer_not_mesh_visible
test_reward_penalty_when_mesh_visible_peer_is_close
test_reward_distance_defaults_to_far_when_no_mesh_reception
```

### Files
- `roadsense-v2v/ml/envs/convoy_env.py`
- `roadsense-v2v/ml/tests/unit/test_convoy_env.py`

### Exit criteria
- Reward distance is computed from mesh-visible peers only
- No penalty when all peers are mesh-invisible
- Penalty correctly applied when mesh-visible peer is close
- All existing tests updated and passing

---

### Phase H5 — Sim-to-Real Validation (NEW — post-training, pre-deployment)

### Problem

The model is trained entirely in simulation (SUMO + emulator). Before field deployment, we need confidence that it generalizes to real data.

### Available real data

| Recording | Duration | TX rows | RX rows |
|-----------|----------|---------|---------|
| Convoy main (004) | 195.5s | 4,984 | 3,768 |
| Convoy extra (005) | 581.6s | 14,899 | 12,092 |
| **Total** | **777s** | **19,883** | **15,860** |

### Validation approach

**Offline replay:** Feed real RX data through the trained model's observation pipeline and record what actions it would have taken. Compare against what a safe driver should have done.

Steps:
1. Parse real RX CSV into time-ordered V2V messages.
2. At each timestep, build observation from received messages (using the same ObservationBuilder).
3. Run model inference → get action (decel fraction).
4. Compare against ego's actual recorded sensor data (TX CSV has ego's real accel/speed).
5. Report:
   - Does the model brake when real driver braked?
   - Does the model brake when it shouldn't (false positive)?
   - Reaction timing vs real driver reaction timing.

### Key metrics
- `correlation_model_vs_real_braking` — do they brake at the same times?
- `false_positive_rate` — model brakes when real driver didn't (and no hazard existed)
- `false_negative_rate` — model doesn't brake when real driver did
- `reaction_time_delta` — model reaction vs real driver reaction

### Implementation note

This is a NEW script, not a modification of existing training code. It can be implemented after training completes but MUST run before field deployment.

### Files
- `roadsense-v2v/ml/scripts/validate_against_real_data.py` (NEW)
- `roadsense-v2v/ml/tests/unit/test_real_data_validator.py` (NEW)

### Exit criteria
- Script runs without error on both recordings
- Metrics report generated
- No systematic false negatives (model must brake when real driver braked)
- False positive rate documented (some conservatism is acceptable)

---

## 5) Technical Notes

### T1: Cone Filter Architecture (verified correct — no change needed)

Two cone filters exist in the pipeline. Both are correct and serve different purposes:

| Filter | Location | Purpose |
|--------|----------|---------|
| Emulator `simulate_mesh_step()` | Network layer | Each intermediate vehicle only relays data from peers in its forward cone |
| ObservationBuilder `build()` | Perception layer | Ego only includes peers in its forward cone in the observation vector |

These are NOT redundant. Example: a vehicle directly behind ego broadcasts its data. Ego receives it (within ESP-NOW range), but the observation builder correctly excludes it from the state vector. The emulator filter would separately prevent intermediate nodes from relaying that vehicle's data forward.

Both use 45° half-angle (90° total forward cone). Confirmed correct per professor.

### T2: Chain Reaction in SUMO (verified correct — no code change needed)

When hazard injector pins one vehicle to 0 m/s via `setSpeed(0.0)`:
- All vehicles behind it operate under SUMO's Krauss CF model
- CF model automatically cascades braking backward through the convoy
- Each braking vehicle broadcasts its state via mesh
- Ego receives multiple independent braking confirmations

This means far-source hazards are observable through the chain reaction even under packet loss on individual links.

### T3: Hazard injection creates instant stop (known limitation)

`setSpeed(target, 0.0)` pins the vehicle to 0 m/s within one SUMO step (0.1s). Real emergency braking at 8.0 m/s² from 50 km/h takes ~1.7s. This makes training scenarios more extreme than reality. Acceptable: the model trains against worst-case, which should make it conservative in deployment.

### T4: Comfort thresholds are human-centric, independent of MAX_DECEL

The reward comfort penalty thresholds (0.5, 3.0, 4.5 m/s²) represent human comfort levels, not vehicle capability fractions. They remain unchanged when MAX_DECEL increases from 5.0 to 8.0. The new 4.5–8.0 m/s² range is available for true emergencies but always incurs maximum comfort penalty (-10).

---

## 6) Dependency Graph

```
Phase H0: MAX_DECEL correction ──────── (trivial, do first)
    │
    ▼
Phase H4: Reward mesh-visibility ────── (must be done before training)
    │
    ▼
Phase H1: Hazard injector upgrade ───── (source diversity for training)
    │
    ▼
Phase H2: Source-specific eval ──────── (instrumentation)
    │
    ▼
Phase H3: Eval matrix ──────────────── (validation coverage)
    │
    ▼
═══════════════════════════════════════
    TRAIN RUN 004
═══════════════════════════════════════
    │
    ▼
Phase H5: Sim-to-real validation ────── (post-training, pre-deployment)
```

**H0 → H4 → H1 → H2 → H3 → TRAIN → H5**

H0 and H4 are prerequisites for training (wrong constants and wrong reward signal corrupt the model).
H1/H2/H3 improve training quality and eval coverage.
H5 is a post-training gate before real-world deployment.

---

## 7) Pre-Run-004 Gate Checklist

- [x] H0: MAX_DECEL = 8.0, tests pass
- [x] H4: Reward uses mesh-visible peer distances, tests pass
- [x] H1: Hazard injector supports uniform_front_peers + fixed strategies, tests pass
- [x] H2: Source-specific eval metrics with reaction threshold = 0.5 m/s², tests pass
- [x] H2: `hazard_message_received_by_ego` field populated in all eval episodes
- [ ] H3: Deterministic eval matrix generated for n=1..5
- [x] `metrics.json` includes `source_reaction_summary` with reception_rate per bucket
- [x] Full unit suite passes (`pytest tests/unit/ -q`)
- [ ] Dry-run eval confirms non-empty buckets for all required `(n, source-rank)` combinations

If ANY item is unchecked, do not launch Run 004.

### Post-Training Gate (before field deployment)

- [ ] H5: Real data validation script runs on both recordings
- [ ] No systematic false negatives
- [ ] Results documented in run artifacts

---

## 8) Deliverables

1. Code changes: MAX_DECEL, reward gating, hazard injector, eval instrumentation.
2. Updated tests for all phases (TDD — tests written first).
3. Eval artifact with source-specific reaction summaries including reception rates.
4. Sim-to-real validation report against real convoy data.
5. Architect sign-off note referencing this plan.

---

## 9) Decisions Log

| Decision | Value | Rationale | Date |
|----------|-------|-----------|------|
| MAX_DECEL | 8.0 m/s² | Real convoy recording measured -8.63 m/s² raw braking | Mar 2, 2026 |
| Cone half-angle | 45° (no change) | Confirmed acceptable by Amir | Mar 2, 2026 |
| Reaction threshold | 0.5 m/s² | Below this could be noise/CF drift; 6.25% of MAX_DECEL | Mar 2, 2026 |
| Comfort thresholds | Unchanged (0.5/3.0/4.5) | Human-centric, independent of vehicle max | Mar 2, 2026 |
| Double cone filter | Keep both | Network-layer relay filter + perception-layer observation filter serve different purposes | Mar 2, 2026 |
| Reward distance source | Mesh-visible peers only | Must match what model can observe; ground truth creates training mismatch | Mar 2, 2026 |

---

**Document Version:** 2.1
**Previous Version:** 2.0 (March 2, 2026 — initial H0..H5 plan release)
**Authors:** Amir Khalifa + Claude
**Next Review:** After H3 deterministic eval-matrix implementation + dry-run coverage validation
