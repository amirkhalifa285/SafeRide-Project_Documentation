# Run 012 Root Cause Analysis

**Date:** March 10, 2026
**Author:** Codex
**Scope:** Verify why Run 012 failed, using local artifacts, code, and real-data replay
**Runs compared:** `cloud_prod_011` vs `cloud_prod_012`
**Status:** Root cause confirmed, fix implemented in code, all unit and Docker integration tests passing, Run 013 in progress

> Note:
> Sections up to `Fixes Implemented` describe the pre-fix Run 012 failure state.
> The `Fixes Implemented` section reflects the current codebase after Claude's patch.

---

## Executive Verdict

Run 012 did **not** fail because "gradual braking is too realistic" in some vague sense.

It failed because the training objective is still built around **late geometry consequences** rather than **early V2V braking signal usage**:

1. The hazard signal **did reach ego** in Run 012.
2. The observation already contains peer acceleration (`peers[*][4]`, `ego[4]=min_peer_accel`).
3. But the reward function gives **no direct incentive** to brake when that signal appears.
4. The only meaningful penalty for "do nothing" still turns on **late**: when distance drops below 10 m.
5. Gradual `slowDown()` removed the artificial instant-closing-rate shock that previously forced Run 011 to react anyway.
6. PPO therefore converged toward a near-zero policy again: survive if possible, brake rarely, accept unsafe proximity.

So the real root cause is:

**Run 012 changed the hazard physics, but did not change the reward economics to value early V2V response.**

Claude's explanation is directionally close, but incomplete. The failure is not just "less urgency." The deeper issue is that **the environment objective still makes early braking irrational** under gradual hazards.

---

## What the Evidence Says

### 1. Run 012 lost V2V reaction completely in SUMO

From `ml/results/cloud_prod_012/metrics.json`:

- Eval episodes: `276`
- Collision rate: `0.0%`
- Behavioral success rate: `53.6%`
- Avg pct time unsafe: `28.2%`
- Hazard reaction rate: `0 / 276`

Compared to Run 011:

- Reaction rate: `276 / 276`
- Behavioral success rate: `84.1%`
- Avg pct time unsafe: `16.7%`

This is not a mild regression. It is a total collapse of active braking behavior.

### 2. The hazard message path was still working

Run 012 hazard episodes:

- `hazard_message_received_by_ego = 276 / 276`
- `hazard_any_braking_peer_received = 268 / 276`
- `reaction_detected = 0 / 276`

That means:

- Mesh relay was not broken.
- Hazard injection was not failing.
- Ego usually did receive braking-related peer messages.
- The model still chose not to brake.

This rules out "the signal never got there" as the root cause.

### 3. Run 012 survived by riding the edge

Run 012 hazard episodes:

- Mean `min_distance_post_hazard_m`: `2.82 m`
- Episodes under `1.0 m`: `54 / 276`
- Worst case: `0.015 m`
- Episodes under `5.0 m`: `232 / 276`

Run 011 was already too close at times, but Run 012 is materially worse:

- Run 011 mean `min_distance_post_hazard_m`: `4.15 m`
- Run 011 episodes under `1.0 m`: `23 / 276`

So Claude was right about passive survival existing, but understated how severe it is. Run 012 is effectively a "coast and pray" policy.

### 4. Run 012 is also worse on real convoy replay

Using `ml/scripts/validate_against_real_data.py` on local recordings:

**Main convoy recording (`V001_tx_004.csv` / `V001_rx_004.csv`):**

- Run 011: `1 / 25` braking events detected
- Run 012: `0 / 25` braking events detected

**Extra recording (`V001_tx_005.csv` / `V001_rx_005.csv`):**

- Run 011: `3 / 23` braking events detected
- Run 012: `2 / 23` braking events detected

Run 012 did not fix sim-to-real. It slightly worsened the already-bad real-data sensitivity.

### 5. Run 012 policy stayed much more stochastic than Run 011

From training logs:

- Run 011 final std: about `0.043`
- Run 012 final std: about `0.086`

Run 012 did train stably enough to finish, but it did **not** collapse to a sharp confident policy the way Run 011 did. That is consistent with a weak or ambiguous objective.

---

## The Actual Root Cause Chain

## A. The observation already has the braking signal

`ml/envs/observation_builder.py` includes:

- peer accel in `peers[:, 4]`
- minimum peer accel in `ego[4]`

So the model is not blind. The braking signal is present.

## B. The environment detected braking peers, but the reward ignored that information

`ml/envs/convoy_env.py` computes `any_braking_peer_received` from mesh messages:

- `ml/envs/convoy_env.py:430-461`

But in the actual reward call:

- `ml/envs/convoy_env.py:359-364`

only these are passed:

- `distance`
- `action_value`
- `deceleration`
- `closing_rate`

`any_braking_peer_received` was **not passed into the reward**.

Then in `ml/envs/reward_calculator.py:120-155`:

- `any_braking_peer` exists in the function signature
- `early_reaction = 0.0`
- `ignoring_hazard = 0.0`

So the reward had **zero active V2V term**.

This is the most important finding in the whole postmortem.

That was the key pre-fix bug: the codebase talked as if there were an "active V2V incentive," but the reward implementation did not actually reward early response to peer braking.

## C. The only strong "do something" penalty activated late

`ml/envs/reward_calculator.py`:

- safe following reward: `+4` from `20 m` to `35 m`
- missed warning penalty: only when `distance < 10 m` and `closing_rate > 0.5`
- early braking far away can also incur comfort / unnecessary-braking penalties

So under the pre-fix reward:

- At `20-35 m`, doing nothing is often rewarded.
- Below `10 m`, not braking becomes explicitly bad.
- There is no reward for "peer started hard braking, brake early now."

This is exactly the wrong shape for gradual hazards.

## D. `slowDown()` removed the old artificial forcing function

Run 011's instant `setSpeed(0)` hazard made the peer speed collapse immediately.
That created a huge closing-rate spike, so the ego reached the late-danger zone quickly.

Run 012 changed hazard injection to gradual braking:

- `ml/envs/hazard_injector.py:208-215`

For a 50 km/h peer (`13.9 m/s`) slowed to zero over `2-4 s`, the relative closing builds slowly.

Approximate time to reach key reward thresholds if ego does not brake:

From a `25-30 m` gap:

- `distance < 20 m`: about `1.2-2.4 s`
- `distance < 10 m`: about `2.1-3.4 s`
- `distance < 5 m`: about `2.4-3.8 s`

That means the policy can spend roughly the first `1.5-3.0 s` of a hazard in a regime where:

- there is no explicit hazard-signal reward,
- no missed-warning penalty yet,
- and braking is still costly.

Under those economics, PPO relearned the old passive strategy.

## E. CF override still did not create an early-action incentive

CF override is the reason Run 010/011 ever reacted at all, but it is not enough by itself here.

- low action releases to CF before override: `ml/envs/action_applicator.py:49-55`
- override activates only after `CF_OVERRIDE_GRACE_STEPS = 3`: `ml/envs/convoy_env.py:295-303`

Once override is active, action near zero holds speed instead of braking. That stops free-riding on SUMO car-following.

But "hold speed" is still often cheaper than braking until the geometry becomes bad enough.

So the policy does not free-ride on CF anymore. It free-rides on **late reward thresholds**.

That is the new failure mode.

---

## Why Claude's Explanation Is Incomplete

Claude said:

- gradual hazards are gentle enough that the model can survive passively
- therefore the urgency disappeared

That is true but not sufficient.

What Claude missed in the first pass is:

1. The pre-fix reward function had **no direct mechanism** to value braking on V2V signal arrival.
2. `ConvoyEnv` computed braking-peer information but did not use it in reward.
3. The reward still explicitly favored non-braking at safe distances.
4. Therefore Run 012 was almost guaranteed to drift back toward passivity once the instant-stop shock was removed.

So the correct statement is not:

> "Realistic hazards made the model worse."

It is:

> "Realistic hazards exposed that the objective never actually trained early V2V reaction in the first place."

That distinction matters, because it changes the fix.

---

## What This Means for Run 011

Run 011 is **not** the ship model.

Its 100% SUMO reaction rate is real, but it is tied to an artifact:

- instant-stop hazard physics
- late geometry-driven reward

It learned a useful emergency response in simulation, but not the behavior you actually need on the real convoy: reacting to peer braking at distance.

So your complaint is correct:

**"Run 011 did not react to the real convoy data we recorded, so it's useless"** is the right engineering conclusion for deployment.

It is still useful as a diagnostic milestone, but not as a deployment candidate.

---

## Recommended Fix Direction

Do **not** just revert to `setSpeed(0)`.

Do **not** just keep `slowDown()` and hope longer training fixes it.

The next run needs an objective that explicitly rewards early use of the braking signal without reintroducing the old Run 006/007 poison.

### Minimum required changes

1. Add a hazard-conditioned reward term tied to the **actual injected hazard target**, not generic `any_braking_peer`.
2. Reward braking when:
   - hazard is active,
   - the hazard source message has reached ego,
   - the hazard source is braking hard,
   - and ego begins decelerating before distance falls into the <10 m panic zone.
3. Penalize non-response during that same window.
4. Keep this gated to the true injected hazard source to avoid the old "normal driving triggers penalty every step" failure.

### Good implementation shape

Use environment-side truth during training only:

- `hazard_injected`
- `hazard_source_id`
- whether that source has been received by ego
- whether that source is currently broadcasting hard braking

Then apply a small-to-moderate early-response reward / ignore-hazard penalty only in that window.

That gives the agent a clean causal lesson:

> "When the injected hazard source brakes and I receive that message, I should brake before geometry gets ugly."

### Secondary adjustments worth testing

1. Increase the "too late" threshold above `10 m` for gradual hazards.
2. Consider immediate CF override on hazard injection or on first hazard-message reception.
3. Keep a mixed hazard curriculum:
   - mostly gradual `slowDown()`
   - some harsher cases
   - but no dependency on physics-breaking `setSpeed(0)`
4. Add real-data replay as a required acceptance gate for future runs.

---

## Fixes Implemented (March 10, 2026 — Claude)

All three bugs identified above have been fixed.

- `262` unit tests passing locally
- Docker integration tests passing locally
- Code is ready and being used for Run 013

### Fix 1: Wire `hazard_source_braking_received` into reward (the core bug)

**File:** `ml/envs/convoy_env.py` (lines 371-395)

The reward call now passes a **hazard-gated** flag.  This is NOT generic `any_braking_peer`.
It is True only when ALL of:
- `hazard_injector._hazard_injected` is True (a hazard is active)
- `hazard_source_id` is not None (we know the target)
- The hazard source's V2V message was received by ego this step
- That message shows `accel_x <= BRAKING_ACCEL_THRESHOLD`

```python
hazard_source_braking_received = False
if (
    self.hazard_injector is not None
    and self.hazard_injector._hazard_injected
    and hazard_source_id is not None
):
    for received in received_map.values():
        msg = received.message
        source = msg.source_id or msg.vehicle_id
        if source == hazard_source_id:
            if float(msg.accel_x) <= self.BRAKING_ACCEL_THRESHOLD:
                hazard_source_braking_received = True
            break

reward, reward_info = self.reward_calculator.calculate(
    distance=distance,
    action_value=action_value,
    deceleration=actual_decel,
    closing_rate=closing_rate,
    any_braking_peer=any_braking_peer_received,
    hazard_source_braking_received=hazard_source_braking_received,
)
```

This satisfies Codex's gating requirement: the flag is tied to the **actual injected
hazard target**, not generic peer braking.  Normal driving never triggers it.

### Fix 2: Implement `PENALTY_IGNORING_HAZARD` and `REWARD_EARLY_REACTION`

**File:** `ml/envs/reward_calculator.py`

New constants:
```python
PENALTY_IGNORING_HAZARD = -5.0
REWARD_EARLY_REACTION = 2.0
EARLY_REACTION_DIST = 15.0
```

Logic in `calculate()`:
```python
if hazard_source_braking_received:
    decel_abs = abs(deceleration)
    is_braking = decel_abs > self.GENTLE_DECEL_THRESHOLD  # 0.5 m/s²

    if not is_braking:
        ignoring_hazard = self.PENALTY_IGNORING_HAZARD   # -5.0/step
    elif distance > self.EARLY_REACTION_DIST:             # > 15m
        early_reaction = self.REWARD_EARLY_REACTION       # +2.0/step

total = safety + comfort + appropriateness + early_reaction + ignoring_hazard
```

**Economics verified** — braking beats ignoring at every distance:

| Distance | Ignore (no brake) | Brake (2.4 m/s²) | Delta | Winner |
|----------|-------------------|-------------------|-------|--------|
| 8m       | -8.20             | -3.50             | +4.70 | Brake  |
| 12m      | -5.80             | -1.51             | +4.29 | Brake  |
| 18m      | -2.20             | +3.48             | +5.68 | Brake  |
| 25m      | -1.00             | +4.48             | +5.48 | Brake  |

Normal driving unaffected: braking at 25m costs -1.52 vs +4.00 for cruising.  No false incentive to brake.

### Fix 3: Lower `BRAKING_ACCEL_THRESHOLD` from -3.5 to -2.5

**File:** `ml/envs/convoy_env.py` (line 33)

With `slowDown(0, 4s)` from 13.9 m/s, deceleration = -3.47 m/s² — which was ABOVE
the old -3.5 threshold.  That meant ~25-33% of episodes with 4s braking duration
never flagged a braking peer at all.

New threshold -2.5 catches all `slowDown` durations (2s: -6.95, 3s: -4.63, 4s: -3.47).
Still well below normal driving deceleration (gentle braking = -1 to -2 m/s²).

### Test coverage

- 262 unit tests passing (was 253 before; +9 new reward calculator tests)
- New tests cover: ignoring penalty fires, early reaction bonus fires,
  boundary conditions, normal driving unaffected, braking always beats
  ignoring during hazard
- Integration test `test_run006_fixes` updated: assertions relaxed to
  accept non-zero `reward_ignoring_hazard` during forced hazard episodes

### Why this won't repeat Run 006/007

Run 006/007 failed because a generic braking-peer penalty fired during **normal driving**
every time any peer braked.  The fix here gates the penalty to:
1. An active injected hazard (training-only construct)
2. The specific hazard target's V2V message reaching ego
3. That target showing hard braking (accel < -2.5)

During normal driving, condition 1 is false.  During hazard but before the braking
signal propagates via mesh, condition 3 is false.  This is the "one more safety guard"
that Codex identified as missing from the raw `-5.0 when any_braking_peer` approach.

---

## Bottom Line

Run 012 did not fail because realism is bad.

Run 012 failed because:

- the old instant-stop artifact was removed,
- but the reward still does not teach early V2V response,
- so PPO fell back to the cheapest surviving policy: near-zero braking until danger is obvious.

The root cause is therefore an **objective mismatch**, not a mesh bug, not an observation bug, and not simply "gradual braking was too mild."

The fix (implemented above) adds hazard-gated reward terms that explicitly train the
behavior we need: brake when the V2V signal says the hazard source is braking, even
before the closing rate gets dangerous.

**Current status:** all unit and Docker integration tests are passing, and Run 013 is
in progress with the same gradual `slowDown()` hazard physics plus the hazard-gated
reward fix.
