# Formation Stability Fix Plan

**Date:** March 9, 2026
**Owner:** Amir + execution agent
**Priority:** CRITICAL (blocks Run 010 eval recovery and all future training)
**Status:** IN PROGRESS — core code fixes implemented, tests green, dataset regeneration/audit pending
**Parent Plan:** `RUN_010_EVAL_RECOVERY_PLAN.md` (Phase E3, step 6c)

---

## 0. Progress Update (March 9, 2026)

### Implemented

- Added shared scenario-layout helper: `roadsense-v2v/ml/scenario_layout.py`
- `gen_scenarios.py` now redistributes peer `departPos` after dropout and forces generated `scenario.sumocfg` end time to `65s`
- `fix_eval_peer_counts.py` now redistributes peer `departPos` after enforcing `n` and rewrites copied `scenario.sumocfg` to `65s`
- `convoy_env.py` default episode length reduced from `1000` to `500` steps
- `hazard_injector.py` hazard window moved from steps `30-80` to `150-350`
- `ml/scenarios/base_real/scenario.sumocfg` updated from `<end value="120" />` to `<end value="65" />`
- Unit/integration tests updated to match the new timing window and shorter route budget

### Validation Completed

- Targeted unit slice for the touched files passed locally (`77 passed`)
- Docker integration suite passed after updating `test_eval_matrix_coverage_mini` for the new hazard window

### Remaining Before Run 011

- Generate the next fixed dataset (`dataset_v6` or final chosen name) with the new spacing/timing behavior
- Run smoke audit on representative `n=1..5` eval scenarios
- Run full capability audit and verify `15/15` deterministic bucket support
- Visually validate the resulting scenarios in SUMO GUI before training

---

## 1. Problem Statement

Eval scenarios (and training scenarios) produce **broken convoy formations**
where vehicles go bumper-to-bumper or end up behind ego, making hazard
injection into ranked peers impossible. Only rank-1 (nearest peer) works;
ranks 2-5 fail across all peer counts.

Post-jitter-fix smoke audit result (dataset_v5_jitter_fix_fixed40):

| Scenario    | Peers | Supported Ranks | Before Fix |
|-------------|-------|-----------------|------------|
| eval_n1_000 | 1     | rank 1          | none       |
| eval_n2_000 | 2     | rank 1          | rank 1     |
| eval_n3_000 | 3     | rank 1          | rank 1     |
| eval_n4_000 | 4     | rank 1          | rank 1     |
| eval_n5_000 | 5     | rank 1          | ranks 1, 2 |

Bucket coverage: **5 out of 15 (33%)** — same as before jitter fix.

The jitter fix (setting `spawn_jitter_s = (0.0, 0.0)` in `gen_scenarios.py`)
solved the n=1 case but did not address the underlying formation problems.

---

## 2. Root Causes (Three Compounding Issues)

### Issue A: SUMO Safe-Insertion Staggering ("Hidden Jitter")

Even with `depart="0"` for all vehicles, **SUMO refuses to insert them
simultaneously** on a single-lane road when gaps violate safety constraints.

SUMO's safe-insertion check requires:

```
gap >= minGap + vehicle_length + tau * departSpeed
     = 2.5   + 5.0            + 1.0 * 2.13
     = 9.63m  (at base_real defaults)
```

With augmented tau up to 1.5:

```
gap >= 2.5 + 5.0 + 1.5 * 2.13 = 10.7m
```

Actual gaps in `base_real`:

| From  | To    | departPos gap | Safe? (tau=1.0) | Safe? (tau=1.5) |
|-------|-------|---------------|-----------------|-----------------|
| V001  | V002  | 10.921m       | YES (10.9 > 9.6)| BORDERLINE      |
| V002  | V003  | 5.892m        | **NO**          | **NO**          |
| V003  | V004  | 22.000m       | YES             | YES             |
| V004  | V005  | 22.000m       | YES             | YES             |
| V005  | V006  | 22.000m       | YES             | YES             |

Result: SUMO delays V003 insertion to t=4.7s, V004 to t=9.0s, V005 to
t=13.5s, V006 to t=17.9s — even though all have `depart="0"` in the XML.

**The jitter fix removed explicit jitter but SUMO creates its own implicit
stagger because the V002-V003 gap (5.9m) is below the safe insertion
threshold.**

### Issue B: Episode Too Long for Route

| Parameter           | Value             |
|---------------------|-------------------|
| Route length        | 785.47m           |
| Speed limit         | 13.89 m/s (50 km/h) |
| Sim step length     | 0.1s              |
| RL max_steps        | 1000 (= 100s)     |
| sumocfg end         | 120s              |

Route traversal times by speedFactor:

| speedFactor | Desired speed | V001 exits at | V006 exits at |
|-------------|---------------|---------------|---------------|
| 0.8         | 11.11 m/s     | ~71s          | ~63s          |
| 1.0         | 13.89 m/s     | ~57s          | ~51s          |
| 1.2         | 16.67 m/s     | ~47s          | ~42s          |

**Vehicles leave the road at 42-71s, but the episode runs for 100s.** The
remaining 30-58% of each episode has ego truncated with `ego_route_ended`,
wasting compute and — critically — any hazard injected after vehicles exit
fires on an empty road.

SUMO GUI confirmed: vehicles reach end of route at ~second 57 but simulation
continues to second 120.

### Issue C: CF Model Compresses Formation to Bumper-to-Bumper

The car-following (Krauss) model's equilibrium following distance is:

```
gap_eq = tau * cruise_speed + minGap
```

| tau | speedFactor | cruise_speed | gap_eq |
|-----|-------------|--------------|--------|
| 0.5 | 0.8        | 11.11 m/s    | 8.1m   |
| 1.0 | 1.0        | 13.89 m/s    | 16.4m  |
| 1.5 | 1.2        | 16.67 m/s    | 27.5m  |

With initial departPos gaps of 10.9m (V001-V002) and 5.9m (V002-V003), the
CF model either compresses (gap > equilibrium at low tau) or cannot maintain
formation (gap < equilibrium at high tau but SUMO staggers insertion).

All vehicles share the **same vType** (same speedFactor, tau, decel), so they
converge to the same cruise speed. The CF model drives them to `gap_eq`
spacing regardless of initial positions. With small initial gaps and a short
route, the formation never stabilizes into properly-spaced ranks.

SUMO GUI observations confirmed:
- **eval_n1_000:** V001 and V002 bumper-to-bumper the entire route.
- **eval_n2_000:** All 3 vehicles bumper-to-bumper, ego "glitching."
- **eval_n3_000:** Vehicles spawn in pairs (2 front, 2 back), then compress
  to bumper-to-bumper.
- **eval_n4_000:** Ego starts far, accelerates to close gap, ends up in pack.
- **eval_n5_000:** Ego spawns at t=0 but last peer (V006) doesn't appear
  until t=17.9s due to SUMO insertion delays. Huge initial gap never closes.

---

## 3. Why These Issues Also Affect Training

All training scenarios use the same `gen_scenarios.py` with the same
`base_real` starting positions. The same three issues apply:

1. V002-V003 gap too small → SUMO staggers insertion in training too
2. Episodes waste 30-50% of steps after vehicles exit the route
3. CF model compresses formations to bumper-to-bumper

**This means all prior training runs (004-010) trained on scenarios where
peers were partially behind ego or bumper-to-bumper.** This likely explains
why the model only learned to handle rank-1 (nearest peer) hazards — it
never experienced properly-spaced multi-rank formations during training.

---

## 4. Fix Plan

### Fix 1: Shorten Episode and Hazard Window to Fit Route

**Rationale:** The route is the real recorded convoy path (785m). Extending it
loses realism. Instead, fit the episode within the route's natural duration.

**Changes required:**

#### a) `convoy_env.py` — Reduce `DEFAULT_MAX_STEPS`

```
DEFAULT_MAX_STEPS = 1000  →  DEFAULT_MAX_STEPS = 500
```

500 steps × 0.1s = 50s of RL time. With ~6s warmup, total sim time = ~56s.
Even at speedFactor=1.2 (V001 exits at ~47s), ego is active for most of the
episode. At speedFactor=0.8 (exits at ~71s), ego is active for the full
episode.

**Alternative:** 450 steps (45s) for more margin at high speedFactor.

**Implementation status:** DONE on March 9, 2026 (`DEFAULT_MAX_STEPS = 500`).

#### b) `hazard_injector.py` — Adjust hazard window

Current: RL steps 30-80 (sim seconds ~9-14 after start).
This is too early — formation hasn't stabilized.

Target: Hazard injection between **sim seconds 20-40** (after sim start).

```
HAZARD_WINDOW_START = 30   →  HAZARD_WINDOW_START = 150
HAZARD_WINDOW_END   = 80   →  HAZARD_WINDOW_END   = 350
```

At 0.1s/step, RL step 150 = 15s after warmup ≈ sim second 21.
RL step 350 = 35s after warmup ≈ sim second 41.

This gives:
- ~15s for formation to stabilize before hazard
- ~10-15s after hazard for the model to react before route ends
- Safe margin even at speedFactor=1.2

**Implementation status:** DONE on March 9, 2026
(`HAZARD_WINDOW_START = 150`, `HAZARD_WINDOW_END = 350`).

#### c) `scenario.sumocfg` (generated) — Reduce sim end time

```
<end value="120" />  →  <end value="65" />
```

65s = 50s episode + 6s warmup + 9s buffer.

**Implementation:** Modify `gen_scenarios.py` to set the sumocfg end time, or
modify the base_real template.

**Implementation status:** DONE on March 9, 2026 in both places:
- base template `ml/scenarios/base_real/scenario.sumocfg`
- generated/rewritten scenarios via `gen_scenarios.py` and `fix_eval_peer_counts.py`

#### d) `train_convoy.py` / `evaluate_model.py` — Update defaults

Both scripts accept `--max_steps`. The default comes from `convoy_env.py`, so
changing the class default is sufficient. But any hardcoded values in cloud
training scripts must also be updated.

**Implementation status:** VERIFIED — no additional code change required.
`train_convoy.py` / `evaluate_model.py` inherit the env default unless
explicitly overridden.

### Fix 2: Redistribute departPos with Wider Spacing

**Rationale:** The real recording positions have a 5.9m V002-V003 gap that
SUMO cannot handle. When `fix_eval_peer_counts.py` drops peers, remaining
vehicles keep their original too-close positions.

**Changes required:**

#### a) `fix_eval_peer_counts.py` — Redistribute positions after dropping peers

After enforcing peer count, redistribute remaining peer departPos values
evenly across a usable span of the route.

Design:
- V001 stays at departPos=0.000m (invariant)
- Available span: 0 to ~600m (leave 185m buffer before route end for travel)
- Minimum gap between consecutive vehicles: **25m** (covers worst-case
  tau=1.5 × departSpeed=2.13 + minGap=2.5 + length=5.0 = 10.7m, with 2.3x
  safety margin)
- Peers distributed evenly starting from 25m ahead of ego

Example redistributed positions:

| n=1 | V001=0m, V002=50m |
| n=2 | V001=0m, V002=40m, V003=80m |
| n=3 | V001=0m, V002=35m, V003=70m, V004=105m |
| n=4 | V001=0m, V002=30m, V003=60m, V004=90m, V005=120m |
| n=5 | V001=0m, V002=25m, V003=50m, V004=75m, V005=100m, V006=125m |

These gaps ensure:
- All vehicles can insert simultaneously (gap >> safe-insertion threshold)
- CF model has room to stabilize without bumper-to-bumper compression
- All ranks 1..n remain ahead of ego during the hazard window
- The farthest vehicle (at 125m for n=5) still has 660m of road to travel

**Implementation status:** DONE on March 9, 2026.

#### b) `gen_scenarios.py` — Apply same redistribution for training scenarios

Training scenarios also need proper spacing. Add a
`redistribute_peer_positions()` function that both `gen_scenarios.py` and
`fix_eval_peer_counts.py` can call after peer dropout.

Parameters:
- `ego_pos`: 0.0 (fixed)
- `min_gap`: 25.0m
- `max_convoy_span`: 600.0m (route_length - safety_buffer)

**Implementation status:** DONE on March 9, 2026 via shared helper
`ml/scenario_layout.py`.

### Fix 3: Tighten speedFactor Range (Optional but Recommended)

Current range: (0.8, 1.2) — creates 50% speed variation.

At speedFactor=1.2, vehicles exit the 785m route in ~47s. This cuts the
usable episode short. At speedFactor=0.8, vehicles crawl at 11 m/s and the
CF equilibrium gap drops to 8m, causing compression.

Recommended: **(0.9, 1.1)** — 22% variation, still provides diversity.

This is optional if Fix 1 and Fix 2 work, but reduces edge cases.

**Implementation status:** NOT IMPLEMENTED. Deferred unless the new spacing +
timing fixes still leave instability at the audit stage.

---

## 5. Implementation Order

1. [x] **Fix 2a:** Modify `fix_eval_peer_counts.py` to redistribute departPos
2. [x] **Fix 2b:** Add same redistribution to `gen_scenarios.py`
3. [x] **Fix 1a:** Change `DEFAULT_MAX_STEPS` to 500
4. [x] **Fix 1b:** Change hazard window to steps 150-350
5. [x] **Fix 1c:** Update sumocfg end time in generation
6. [x] **Unit tests:** Update existing tests for new defaults
7. [ ] **Generate dataset_v6** with all fixes
8. [ ] **Smoke audit** on representative scenarios
9. [ ] **Full capability audit** if smoke passes
10. [ ] **Resume E3→E4** from the recovery plan

### Impact on existing training

- All prior training scenarios (dataset_v3 through v5) are invalid
- Run 010 model was trained on broken formations
- After fixes, a new training run (Run 011) is needed for fair comparison
- However, the frozen Run 010 model should still be evaluated on fixed eval
  scenarios first to establish a baseline

---

## 6. Verification Criteria

After all fixes:

- [ ] SUMO GUI shows all vehicles spawning simultaneously (no insertion delay)
- [ ] Inter-vehicle spacing remains > 15m throughout the hazard window
- [ ] No vehicles exit the route before RL step 350 (hazard window end)
- [ ] Smoke audit: all 5 representative scenarios support rank 1
- [ ] Full audit: all 15 buckets (n=1..5, rank=1..n) have ≥1 supporting scenario
- [ ] Ego never overtakes any peer during the hazard window (steps 150-350)
- [x] Targeted unit slice for the fix passes
- [x] Docker integration suite passes with the updated hazard window

---

## 7. Files Modified / Verified

| File | Change |
|------|--------|
| `ml/scenario_layout.py` | New shared helper for peer spacing + sumocfg end time |
| `ml/scripts/fix_eval_peer_counts.py` | Redistribute peer `departPos` after peer-count enforcement; rewrite sumocfg end time to `65s` |
| `ml/scripts/gen_scenarios.py` | Redistribute peer `departPos` after dropout; set generated sumocfg end time to `65s` |
| `ml/envs/convoy_env.py` | `DEFAULT_MAX_STEPS = 500` |
| `ml/envs/hazard_injector.py` | `HAZARD_WINDOW_START = 150`, `HAZARD_WINDOW_END = 350` |
| `ml/scenarios/base_real/scenario.sumocfg` | Base template end time changed to `65s` |
| `ml/tests/unit/test_fix_eval_peer_counts.py` | Added assertions for redistributed positions + rewritten sumocfg end time |
| `ml/tests/unit/test_gen_scenarios.py` | Added assertions for redistributed positions + generated sumocfg end time |
| `ml/tests/unit/test_hazard_injector.py` | Updated hazard-window expectations |
| `ml/tests/unit/test_convoy_env.py` | Added default max-steps assertion |
| `ml/tests/integration/test_run006_fixes.py` | Updated hazard-step/max-step assumptions for the new `150-350` window |
| `ml/training/train_convoy.py` / `ml/scripts/evaluate_model.py` | Verified no hardcoded `max_steps=1000` dependency requiring patch |
