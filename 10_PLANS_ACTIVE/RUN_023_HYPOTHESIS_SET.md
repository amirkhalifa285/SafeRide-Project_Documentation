# Run 023 Hypothesis Set

**Date:** March 16, 2026
**Predecessor:** Run 022 (`cloud_prod_022`) — SUMO pass, replay fail
**Purpose:** Define the smallest credible hypothesis set for the next sensitivity-recovery run.
**See also:** `docs/10_PLANS_ACTIVE/RUN_023_IMPLEMENTATION_PLAN.md`

---

## 1. Starting Point

Run 022 established three facts:
- removing ego heading fixed the catastrophic false positives from Run 021
- the heading-free policy still trains, but only to a middling SUMO reaction level (`~64.4%`)
- replay sensitivity is still too low (`12.0%` Recording #2, `26.1%` Extra Driving)

So the next run must preserve:
- `base_real` as the road / route grounding
- no ego heading
- no route-position proxies (`progress`, absolute position, absolute heading)
- CF override
- fast hazard signal (`BRAKING_DURATION 0.5-1.5s`)
- decay-based `braking_received`
- reward normalization and current PPO stability settings

Important clarification:
- **Do not replace `base_real` with a different road or route.**
- **Do not break the real-data grounding.**
- “Broaden hazard-onset geometry” means broadening the **relative vehicle scene at the instant braking starts**, not changing the map.

In this document, "hazard-onset geometry" means:
- source rank ahead (`rank_1`, `rank_2`, ...)
- ego-to-source longitudinal gap at onset
- closing speed at onset
- local convoy compression / stretch at onset
- peer count visible to ego at onset

It does **not** mean:
- using a different road
- using a different route family
- adding absolute map position
- adding absolute heading back into the observation

---

## 2. Core Hypotheses

### H1 — Hazard onset geometry is still too narrow

**Claim:** Even on the correct `base_real` road, hazards still start from a narrow range of **relative vehicle states**. The policy may still be learning a convoy-state template that does not match the diversity of real braking onset conditions.

**Why it fits Run 022:**
- Run 022 clears the minimum SUMO gate but is much weaker than Runs 020-021
- removing heading fixed specificity, but sensitivity remained low
- this suggests the remaining decision boundary is still overfit to training geometry

**Candidate intervention:**
- keep `base_real`, the same road, and the same route grounding
- stop treating episode time alone as the effective hazard trigger
- sample a desired onset bucket over:
  - source rank ahead
  - ego-to-source gap
  - closing speed
  - peer-count / local convoy state
- inject only when the chosen source reaches that sampled bucket on the same route
- keep a fallback injection window so the episode cannot stall forever

**Prediction:**
- replay sensitivity rises on both recordings
- false positives stay low because no absolute-route feature is added back

**Non-goal:**
- this is **not** a proposal to change the road, route, or `base_real` source recording
- this is a proposal to make hazards begin under more varied **relative scenes** on that same road

### H2 — The observation lacks an explicit deployment-safe closing-risk signal

**Claim:** Once heading is removed, the current feature set may be too compressed. `min_peer_accel` and decayed `braking_received` tell the model that braking happened, but not clearly enough whether the braking peer is close enough and closing fast enough to justify ego braking in real data.

**Why it fits Run 022:**
- the policy is mostly calm on replay
- SUMO reaction also dropped once heading disappeared
- FP is already low, so the missing behavior is selective activation
- real moderate braking may not be separated cleanly enough from calm traffic

**Candidate intervention:**
- add one causal, deployment-safe scalar such as front-peer time-to-collision, closing speed, or nearest-front-peer gap
- compute it only from mesh-visible peers, never from absolute route state

**Prediction:**
- sensitivity rises without recreating the Run 021 route leak
- the biggest gains should appear on Recording #2, where the model is currently too conservative

### H3 — Reward still favors waiting for late geometry, not early V2V response

**Claim:** The current reward may still allow the policy to wait until the gap becomes obviously unsafe, which works in SUMO but misses early real-world braking events.

**Why it fits Run 022:**
- SUMO no longer looks dominant once heading is removed
- replay misses are frequent
- the model often produces near-zero action even during real braking events with visible peers

**Candidate intervention:**
- strengthen early-reaction reward when `braking_received` is active and a front-peer risk metric is nontrivial
- reduce any reward path that still lets the policy succeed mainly through late geometry braking

**Prediction:**
- action onset in SUMO shifts earlier
- replay sensitivity improves more than FP does

### H4 — Replay hygiene needs one small fix, but it is not the main blocker

**Claim:** The validator should exclude `V001` self-messages in RX replays.

**Why it fits Run 022:**
- Extra Driving RX contains `81` self-messages
- current validator does not filter them

**Important:** this is **not** the root cause of Run 022 failure. Filtering self-RX made Extra Driving slightly worse, not better.

**Action:**
- fix the validator before the next replay round so future comparisons are cleaner

---

## 3. Recommended Order

### Priority 1 — Test H1 first

This is the most conservative next step:
- it preserves `base_real`
- it preserves the same road and route grounding
- it preserves the deployed observation shape
- it does not introduce new model inputs
- it directly targets remaining geometry overfit

### Priority 2 — If H1 is insufficient, test H2 next

If geometry randomization alone does not lift replay sensitivity enough, the next most plausible explanation is missing causal state, not missing generic capacity.

### Priority 3 — Treat H3 as a second-stage adjustment

Reward changes should come after either:
- geometry broadening, or
- one added causal feature

Otherwise it is too easy to re-enter the Runs 012-017 pattern of reward changes without fixing the real signal mismatch.

---

## 4. Recommended Success Gates for the Next Run

The next run should keep the same hard replay gate:

| Recording | Sensitivity | False Positive Rate |
|-----------|-------------|---------------------|
| Recording #2 | `> 60%` | `< 15%` |
| Extra Driving | `> 75%` | `< 20%` |

Additional guidance:
- do not accept a run that improves sensitivity by reintroducing high FP
- do not accept any feature that can encode route identity or absolute orientation
- do not start a 10M extension without replay pass on both recordings

---

## 5. Proposed Working Decision

**Recommended Run 023 direction:** test **H1 first** while keeping Run 022's heading-free observation intact.

In one sentence:
- **Keep `base_real`, but replace effectively fixed-time hazard onset with state-triggered hazard onset on the same route.**

If that fails, the strongest next candidate is:
- **one** added deployment-safe risk feature from H2
- followed by replay validation before any broader redesign
