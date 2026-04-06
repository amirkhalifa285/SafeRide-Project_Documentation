# Professor PoC Demo Plan

**Created:** April 6, 2026
**Goal:** Deliver the 3 deliverables from `Prof_POC_requirements.md`.
**Status:** IN PROGRESS — mesh proof validated on curve demo

---

## Requirements → Assets

| # | Professor Wants | Asset | Status |
|---|----------------|-------|--------|
| 1a | Cars → intersection → stop + mesh proof | `ml/scenarios/base_intersection/` | **DONE** — 4 vehicles, hazard_step=85 |
| 1b | 3 cars → tight bend → stop | `base_curve` | **DONE** |
| 1b-alt | Longer curve mesh chain proof | `base_curve_mesh4` | **DONE + VALIDATED** — clean terminal proof on Apr 6 |
| 2 | Augmented scenarios: 5+ cars in SUMO, different behaviors | `dataset_v6_formation_fix/` eval scenarios (n=1-5 peers, up to 6 vehicles) | **EXISTS** — just run in SUMO GUI |
| 3 | EGO brakes in augmented scenarios, Deep Sets variable n | `replay_ft_500000_steps.zip` + `demo_convoy_gui.py` | **TODO:** run demos, vary hazard vehicle |

---

## Phase 1: Intersection Scenario — DONE

### `ml/scenarios/base_intersection/`

- **Network:** `base/network.net.xml` (copied), junction `10203729629`
- **Route:** `-636924784#1` (453m) → `-636924784#0` (1617m), lane 0
- **Vehicles:** 4 (V001-V004), 30m gaps, `departSpeed=10`, starting at 100-230m
- **Hazard:** V004 (lead) emergency brakes at intersection, `hazard_step=85`
- **Mesh path:** V004 → V003 → V002 → V001 (up to 3 hops)

**Launch command:**
```bash
cd roadsense-v2v
./ml/run_demo_gui.sh --scenario base_intersection --hazard_vehicle V004 --hazard_step 85 --max_relay_hops 5
```

### Mesh Relay Proof — DONE

Validated in the curve demo with a clean causal chain display:
- `MESH TRACE: [V004->V003, V003->V002, V002->V001]`
- `MESH RX: [V002(hop=0), V003(hop=1), V004(hop=2)]`

Important implementation note:
- The emulator still keeps the raw multi-round forwarding trace for debugging.
- `demo_convoy_gui.py` now reconstructs and prints only the unique source-to-ego chain, so duplicate relay rounds and bounce traffic do not clutter the professor-facing output.

### Optional: Curve Mesh Proof with Longer Chain

Use `ml/scenarios/base_curve_mesh4/` when you want the blind-curve video to show a
3-hop forwarding chain instead of the original 2-hop layout:

```bash
cd roadsense-v2v
./ml/run_demo_gui.sh --scenario base_curve_mesh4 --hazard_vehicle V004 --cone_half_angle 90 --hazard_step 120
```

Expected terminal proof:
- `MESH TRACE: [V004->V003, V003->V002, V002->V001]`
- `MESH RX: [V002(hop=0), V003(hop=1), V004(hop=2)]`

---

## Phase 2: Show Augmented Scenarios in SUMO GUI (No EGO)

Use existing `dataset_v6_formation_fix/eval/` scenarios directly:

| Demo | Scenario | Total Cars | Braking Vehicle |
|------|----------|-----------|----------------|
| A | `eval_n4_000` | 5 (ego + 4 peers) | V005 |
| B | `eval_n5_000` | 6 (ego + 5 peers) | V006 |
| C | `eval_n2_000` | 3 (ego + 2 peers) | V003 |

Each demo has a different vehicle braking to show variety.

---

## Phase 3: EGO Braking in Augmented Scenarios

Same model for ALL demos (`replay_ft_500000_steps.zip`, ego_stack=3):

| Demo | Scenario | Peers (n) | Hazard Vehicle |
|------|----------|-----------|---------------|
| 1 | `eval_n2_000` | 2 | V003 |
| 2 | `eval_n4_000` | 4 | V005 |
| 3 | `eval_n5_000` | 5 | V006 |

```bash
./ml/run_demo_gui.sh \
  --scenario datasets/dataset_v6_formation_fix/eval/eval_n5_000 \
  --hazard_vehicle V006 \
  --model_path ml/results/run_025_replay_v1/checkpoints/replay_ft_500000_steps.zip \
  --ego_stack_frames 3 --max_relay_hops 4
```

**Key proof:** Same `.zip` → model brakes at n=2, n=4, n=5 with no retraining.

---

## Execution Order

1. ~~Create `base_intersection/` → test in Docker~~ ✅
2. ~~Add mesh relay terminal logging → capture proof~~ ✅
3. Run augmented eval scenarios in SUMO GUI → record ← **NEXT**
4. Run EGO demos (n=2, n=4, n=5) → record
5. Compile for professor

---

## Success Criteria

- [x] Video: 4 cars → intersection → stop (hazard_step=85)
- [x] Proof: Mesh relay V004→V003→V002→V001 with clean terminal chain + RX hop counts
- [ ] Video: 5-car augmented scenario, V005 brakes
- [ ] Video: 6-car augmented scenario, V006 brakes
- [ ] Video: EGO braking with n=2 peers
- [ ] Video: EGO braking with n=4 peers
- [ ] Video: EGO braking with n=5 peers
- [ ] Proof: Same model handles all n (Deep Sets)
