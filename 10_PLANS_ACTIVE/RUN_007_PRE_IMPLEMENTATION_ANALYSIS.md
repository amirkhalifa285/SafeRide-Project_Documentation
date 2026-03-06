# Run 007 Pre-Implementation Analysis

**Date:** March 5, 2026  
**Author:** Codex (pre-code analysis only)  
**Status:** Historical pre-implementation draft (superseded by implemented Strategy B+ update in `RUN_007_STRATEGIC_ANALYSIS.md` Section 13)

---

## 1) Why this document exists

`docs/PROJECT_STATUS_OVERVIEW.md` and `docs/10_PLANS_ACTIVE/RUN_005_006_POST_MORTEM.md` conflict on Run 006 outcome:
- `PROJECT_STATUS_OVERVIEW.md` states Run 006 was ready for EC2.
- `RUN_005_006_POST_MORTEM.md` states Run 006 failed with severe reward divergence.

This document reconciles that conflict by analyzing the current code path and SUMO/TraCI behavior before implementing fixes.

---

## 2) Current code path and where the failure likely starts

### 2.1 Signal chain

In [`roadsense-v2v/ml/envs/convoy_env.py`](/home/amirkhalifa/RoadSense2/roadsense-v2v/ml/envs/convoy_env.py):

1. `step()` applies ego action through `ActionApplicator.apply()`.
2. `HazardInjector.maybe_inject()` may force one front peer to speed `0.0`.
3. `sumo.step()` advances SUMO (`traci.simulationStep()`).
4. `_step_espnow()` reads mesh-visible peer messages and sets `any_braking_peer_received` with:
   - `msg.accel_x <= -0.5` OR
   - `msg.speed <= 0.5`
5. `RewardCalculator.calculate()` in [`roadsense-v2v/ml/envs/reward_calculator.py`](/home/amirkhalifa/RoadSense2/roadsense-v2v/ml/envs/reward_calculator.py) applies:
   - `PENALTY_IGNORING_HAZARD = -5.0/step` when `any_braking_peer=True`, `distance <= 30m`, ego decel `< 0.5`

### 2.2 Why this is risky in SUMO

In SUMO Krauss car-following, routine longitudinal control includes frequent mild decel in normal traffic (especially in convoy spacing control). Using `-0.5 m/s^2` as a hazard detector is too sensitive.

Also, `speed <= 0.5 m/s` captures non-hazard states:
- vehicles near depart,
- vehicles at route end,
- temporary creep/queue behavior.

This means hazard logic can trigger without hazard injection.

---

## 3) TraCI behavior that matters (and why unit tests missed it)

### 3.1 Step timing semantics

From [`roadsense-v2v/ml/envs/sumo_connection.py`](/home/amirkhalifa/RoadSense2/roadsense-v2v/ml/envs/sumo_connection.py):
- SUMO runs at `--step-length 0.1` (10 Hz).
- `getAcceleration()` and `getSpeed()` are sampled from SUMO dynamics each step.

From [`roadsense-v2v/ml/envs/hazard_injector.py`](/home/amirkhalifa/RoadSense2/roadsense-v2v/ml/envs/hazard_injector.py):
- Hazard is injected via `traci.vehicle.setSpeed(target, 0.0)` at one scheduled step.

From [`roadsense-v2v/ml/envs/action_applicator.py`](/home/amirkhalifa/RoadSense2/roadsense-v2v/ml/envs/action_applicator.py):
- Ego action `<= 0.02` releases control via `setSpeed(-1)`.

### 3.2 Why tests passed but training failed

Existing integration tests validated:
- penalty can fire during forced hazards,
- plumbing and shape correctness.

They did not validate:
- penalty firing rate in **non-hazard** episodes,
- cumulative contribution of `reward_ignoring_hazard` over full episodes.

So tests passed while reward economics were still broken.

---

## 4) Reward-economics sanity check

Given current constants:
- `PENALTY_IGNORING_HAZARD = -5.0/step`
- trigger includes distance `< 30m` (usually true in convoy)
- trigger can be active on normal decel via `accel <= -0.5`

Even a moderate false trigger rate becomes dominant:
- `300` false-positive steps in an episode => `-1500` reward from this term alone.

That is enough to overpower other components and destabilize PPO updates.

---

## 5) Root-cause statement

Primary root cause is likely **hazard trigger overreach**, not PPO hyperparameters:
- `any_braking_peer` currently mixes hazard and normal CF behavior,
- then a large penalty (`-5`) is applied as if every trigger were a true hazard.

This aligns with post-mortem symptoms:
- strongly negative reward floor,
- persistent divergence,
- poor learnability despite tests passing.

---

## 6) Fix options (pre-implementation)

### Option A: Threshold retune only

- Change braking threshold from `-0.5` to around `-3.0 m/s^2`.
- Remove speed threshold or reduce to near-zero (`<= 0.1`) and avoid OR-triggering on speed alone.

Pros:
- Minimal code change.
- Keeps hazard logic observation-driven.

Cons:
- Still purely threshold-based; may remain sensitive across scenarios.

### Option B: Threshold retune + hazard-context gating (recommended)

- Use hard-brake threshold (`<= -3.0`) for generic detection.
- Do not treat low speed alone as hazard.
- Optionally allow low-speed hazard signal only when it is the known injected hazard target after `hazard_injected=True`.

Pros:
- Strong suppression of normal-driving false positives.
- Preserves hazard detection after injected event.

Cons:
- Slightly more coupling to hazard-injector state during training.

### Option C: Explicit two-phase hazard detector

- Separate “hazard onset detection” and “hazard persistence” with step counters/state machine.
- Penalize only during validated hazard windows.

Pros:
- Most robust long-term architecture.

Cons:
- Bigger design + test scope; not ideal for immediate Run 007 turnaround.

---

## 7) Recommended plan for Run 007

1. Apply Option B (smallest robust patch).
2. Add mandatory diagnostic instrumentation before EC2:
   - per-episode `% steps any_braking_peer=True`
   - total `reward_ignoring_hazard`
   - hazard vs non-hazard split.
3. Add regression tests that fail if non-hazard firing is too high.
4. Run Docker integration validation before training launch.

---

## 8) Mandatory validation gates before spending EC2 budget

A run should be blocked unless all pass:

1. Non-hazard diagnostic episodes (`hazard_injection=False`):
   - `reward_ignoring_hazard` fires on `< 10%` of steps.
2. Forced-hazard episodes:
   - `reward_ignoring_hazard` fires reliably after hazard event.
3. Existing integration suite remains green.
4. Short smoke train (e.g., 5k steps) shows no immediate reward collapse.

---

## 9) Proposed file-level implementation scope (for reviewer)

Planned changes (not yet applied):
- [`roadsense-v2v/ml/envs/convoy_env.py`](/home/amirkhalifa/RoadSense2/roadsense-v2v/ml/envs/convoy_env.py): braking-peer detection logic and thresholds.
- [`roadsense-v2v/ml/tests/unit/test_convoy_env.py`](/home/amirkhalifa/RoadSense2/roadsense-v2v/ml/tests/unit/test_convoy_env.py): false-positive/true-positive signal tests.
- [`roadsense-v2v/ml/tests/integration/test_run006_fixes.py`](/home/amirkhalifa/RoadSense2/roadsense-v2v/ml/tests/integration/test_run006_fixes.py): non-hazard penalty-rate regression guard.

Optional (if reviewer approves):
- add episode-level debug counters to eval outputs in `train_convoy.py`.

---

## 10) Decision request to reviewer

Please review and approve one of:

1. **Approve Option B as-is** (recommended, fastest safe path).
2. **Approve Option A only** (faster but higher residual risk).
3. **Escalate to Option C** (strongest but slower).

No code changes were made while preparing this document.
