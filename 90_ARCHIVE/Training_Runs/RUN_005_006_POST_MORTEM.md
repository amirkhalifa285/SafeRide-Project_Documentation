# Run 005 & Run 006 Post-Mortem — Root Cause Analysis

**Created:** March 5, 2026
**Status:** ANALYSIS COMPLETE — Strategy B+ fixes implemented for Run 007 prep (March 5, 2026)
**Audience:** All AI agents working on RoadSense ML training

---

## Executive Summary

Run 005 and Run 006 both failed. The failures share a common pattern: **changes that pass unit tests but break real training because the tests never exercise the actual reward/physics dynamics over thousands of steps.**

| Run | Result | Core Failure |
|-----|--------|--------------|
| 005 | avg_reward=+175, 26.4% collisions, 0% V2V reaction | GPS-noise false collisions + reward carrot too weak |
| 006 | avg_reward=-6,700, std exploded to 3.05, policy diverged | PENALTY_IGNORING_HAZARD fires on normal driving, not just hazards |

**Both runs wasted ~$5-8 each on EC2. The lesson is the same: you cannot validate reward shaping with unit tests. You must reason about cumulative reward dynamics over full episodes.**

---

## Run 005 Failure Analysis

### What was changed (from Run 004)
1. 30-step warmup in `reset()` to prevent spawn-time collisions
2. Early reaction reward `_early_reaction_bonus()` → +2.0 for proactive braking
3. Eval matrix flags wired into cloud script
4. `HAZARD_PROBABILITY` 0.3 → 0.5
5. Ego route-end guard
6. `run_docker.sh` array quoting fix

### What went wrong
- **Collision rate went UP (19% → 26.4%)** despite warmup fix. The warmup may have been insufficient, or the GPS-noise false collision problem (which existed in Run 004 too) was exposed more by the longer episodes.
- **V2V reaction stayed at 0%** because the +2.0 early reaction bonus was still too weak versus the comfort penalty. The model's optimal strategy remained "never brake."
- **Reward dropped (+352 → +175)** — the increased hazard probability (50% vs 30%) meant more episodes had hazards the model couldn't react to, dragging down average reward.

### What the agents missed
The agents never calculated the cumulative reward math:
- Comfort penalty for braking: -0.4 to -10.0 per step (depending on decel magnitude)
- Early reaction bonus: +2.0 per step (only during hazard + braking)
- Hazard episodes are ~50% of training. Of those, the hazard window is maybe 20-40 steps out of 1000.
- Net economics: braking costs comfort penalty for ALL steps, bonus only applies during narrow hazard window
- **The model correctly learned that not braking is always better.**

This is a reward design failure that no unit test can catch. The unit tests verified that `_early_reaction_bonus()` returns +2.0 given the right inputs. They never verified that +2.0 is enough to change the policy over 10M steps.

---

## Run 006 Failure Analysis

### What was changed (from Run 005)
1. Ground-truth collision detection (SUMO positions for termination, mesh for reward)
2. Pre-action route-end guard at top of `step()`
3. Warmup feeds emulator + extends if GT distance too close
4. `min_peer_accel` added to ego observation (4→5 dims)
5. `PENALTY_IGNORING_HAZARD = -5.0` per step when not braking during detected hazard
6. Comfort penalty zeroed when braking during detected hazard
7. `emulator.reset()` per episode, hazard injection failure tracking

### What went wrong

**THE KILLER BUG: `any_braking_peer` is true during NORMAL DRIVING, not just hazards.**

The `PENALTY_IGNORING_HAZARD = -5.0` fires every step when:
- `any_braking_peer == True` AND
- ego decel < 0.5 m/s² AND
- distance < 30m

The problem is in how `any_braking_peer` is determined (`convoy_env.py` lines 440-444):

```python
if (
    float(msg.accel_x) <= self.BRAKING_ACCEL_THRESHOLD  # -0.5 m/s²
    or float(msg.speed) <= self.BRAKING_SPEED_THRESHOLD  # 0.5 m/s
):
    any_braking_peer_received = True
```

**`BRAKING_ACCEL_THRESHOLD = -0.5 m/s²` is way too sensitive.** In SUMO's Krauss car-following model, vehicles routinely decelerate at -0.5 to -2.0 m/s² during normal following. This is not emergency braking — it's just the CF model adjusting speed to maintain following distance.

**`BRAKING_SPEED_THRESHOLD = 0.5 m/s` triggers on near-stopped vehicles**, which includes vehicles that haven't fully departed yet or are at the end of their route.

### The math that proves the bug

- Episodes are ~725 steps long
- In a normal convoy at 30m distance, peers fluctuate between -2.0 and +1.0 m/s² due to CF dynamics
- `accel_x <= -0.5` is true for roughly 40-60% of steps (just from normal following)
- Most of these steps, ego distance is < 30m (convoy following distance is 10-20m)
- The penalty fires: ~400 steps * -5.0 = **-2,000 per episode** from the ignoring-hazard penalty alone
- Plus REWARD_UNSAFE (-5.0) fires when distance < 10m
- Plus PENALTY_MISSED_WARNING (-3.0) fires during closing
- Total: **-6,700 average reward** — the model is in a penalty trap

### Why `std` exploded from 1.0 to 3.05

The policy tried to escape the constant -5.0/step penalty by increasing exploration (raising std). But no action can fix the problem — braking triggers comfort penalty, not braking triggers ignoring-hazard penalty. The policy diverged because there is no good action in this reward landscape.

### Evidence from training log

```
Step 0:        ep_rew_mean = -10,400    std = 1.02    clip_fraction = 0
Step 1M:       ep_rew_mean = -7,000     std = ~2.0    clip_fraction = ~0
Step 3M:       ep_rew_mean = -7,200     std = ~2.5    clip_fraction = 0
Step 6.4M:     ep_rew_mean = -6,880     std = 3.07    clip_fraction = 0
```

- Reward never went positive — trapped from the start
- `clip_fraction = 0` means PPO's clipping never activates — the policy gradient is dead
- `std` monotonically increasing = policy divergence

### What the agents missed

The agents wrote 8 integration tests in Docker that all passed. But none of those tests checked:
1. **What percentage of steps have `any_braking_peer == True` during normal (non-hazard) episodes?**
2. **What is the cumulative reward over a full episode with the new penalty?**
3. **Does the penalty fire during normal driving or only during actual hazard injection?**

A simple diagnostic would have caught this:
```python
# Run 1 episode with hazard_injection=False
# Count how many steps have any_braking_peer == True
# If > 10% → the penalty will fire during normal driving → bug
```

This was never done. The integration tests verified that the penalty fires when it should (forced hazard + no braking), but never verified that it DOESN'T fire when it shouldn't (normal driving).

---

## Pattern: Why Reward Bugs Keep Recurring (Runs 003, 005, 006)

| Run | Bug | Same Pattern |
|-----|-----|-------------|
| 003 | `PENALTY_MISSED_WARNING` fires every step at distance < 15m | Penalty fires during normal driving |
| 005 | `REWARD_EARLY_REACTION` (+2.0) too weak vs comfort penalty | Reward math not computed over full episode |
| 006 | `PENALTY_IGNORING_HAZARD` fires every step via overly sensitive threshold | Penalty fires during normal driving |

**The recurring failure mode:** A reward component is designed to encourage/discourage a specific behavior during specific conditions, but the trigger condition is too broad and fires during NORMAL DRIVING. Unit tests verify the function returns the right value for the right inputs. Nobody checks whether the inputs occur in realistic scenarios.

---

## How To Properly Analyze Reward Changes Before Training

### Step 1: Cumulative Reward Math (MANDATORY before any run)

For EVERY reward component, calculate:
- How many steps per episode does this fire? (not just "does it fire" — what PERCENTAGE)
- What is the cumulative contribution over a full 1000-step episode?
- Compare the magnitude to other components. Is the new component > 10x any existing one? That's a red flag.

Example for Run 006 (which SHOULD have been done):
```
PENALTY_IGNORING_HAZARD = -5.0/step
Fires when: any_braking_peer AND distance < 30m AND ego not braking

Q: How often is any_braking_peer true in normal driving?
A: SUMO CF model → peers decel at -0.5 to -2.0 routinely → TRUE ~50% of steps
A: Threshold is -0.5 → way too sensitive for "hazard detection"

Q: How often is distance < 30m?
A: Convoy following distance is 10-20m → TRUE ~90% of steps

Q: Cumulative impact?
A: 500 steps * -5.0 = -2,500/episode → DOMINATES all other reward components

VERDICT: This penalty is broken. It fires during normal driving.
```

### Step 2: Diagnostic Episode Run (MANDATORY before any run)

Run 3-5 episodes in Docker with logging enabled:
1. **Non-hazard episode** (hazard_injection=False): Log `any_braking_peer`, `distance`, and each reward component per step. Count how many steps each penalty fires.
2. **Hazard episode**: Same logging. Compare penalty firing rate between hazard and non-hazard.
3. If any penalty fires > 10% of steps during NON-HAZARD episodes, it's too sensitive.

### Step 3: Threshold Validation

For any threshold used in reward (e.g., `BRAKING_ACCEL_THRESHOLD`):
- What is the NORMAL range of this value during calm driving?
- What is the range during actual hazard events?
- Is there clear separation? If normal range overlaps the threshold, the reward component is broken.

For `BRAKING_ACCEL_THRESHOLD = -0.5`:
- Normal CF deceleration: -0.5 to -2.0 m/s² (routine)
- Hazard braking: -4.0 to -8.0 m/s² (emergency)
- The threshold is IN the normal range → broken

A threshold of -3.0 m/s² would separate normal from hazard braking.

### Step 4: Don't Trust Integration Tests for Reward Validation

Integration tests verify:
- The code doesn't crash
- The observation shapes are correct
- Specific conditions trigger specific outputs

Integration tests CANNOT verify:
- Whether the reward landscape is learnable
- Whether a penalty fires too often
- Whether the cumulative reward makes the desired behavior optimal

**Reward validation requires episode-level analysis, not function-level testing.**

---

## Concrete Fixes Needed for Run 007

### Fix 1: `BRAKING_ACCEL_THRESHOLD` is too sensitive
- Current: `-0.5 m/s²` (fires on normal CF deceleration)
- Proposed: `-3.0 m/s²` (only fires on actual hard braking)
- Rationale: Normal following decel is -0.5 to -2.0. Emergency braking is -4.0 to -8.0. Threshold of -3.0 cleanly separates the two.

### Fix 2: `BRAKING_SPEED_THRESHOLD` is wrong
- Current: `0.5 m/s` (fires on near-stopped vehicles including pre-departure)
- Proposed: Remove entirely, or set to 0.1 m/s
- Rationale: A vehicle going 0.4 m/s is not a hazard signal — it might just be departing.

### Fix 3: Consider tying `any_braking_peer` to hazard injection
- Alternative: Only count `any_braking_peer` when the peer's deceleration changed significantly from the previous step (delta-based detection, not absolute threshold)
- Or: Only enable the ignoring-hazard penalty after the hazard injector has actually fired

### Fix 4: Add diagnostic logging to training
- Log per-episode: how many steps had `any_braking_peer=True`, total ignoring-hazard penalty, total comfort penalty
- This would have caught the Run 006 bug within the first 100 episodes

### Fix 5: MANDATORY pre-training diagnostic
- Before ANY future training run, run the diagnostic episode analysis from Step 2 above
- If any penalty fires > 10% of steps in non-hazard episodes, DO NOT proceed to EC2

---

## Files Referenced

| File | Relevant Lines |
|------|---------------|
| `ml/envs/convoy_env.py` | 33-34 (thresholds), 440-444 (any_braking_peer logic) |
| `ml/envs/reward_calculator.py` | 48-50 (PENALTY_IGNORING_HAZARD), 118-128 (_ignoring_hazard_penalty) |
| `ml/training/train_convoy.py` | Training entry point |
| `~/RoadSense2/run_006_results/training-run.log` | Full training log (reward progression) |
