# Run 003 Session Fixes for Architect Review

**Date:** March 2, 2026
**Prepared by:** Codex
**Scope:** Post-`RUN_003_FINDINGS_FOR_ARCHITECT_REVIEW.md` follow-up fixes completed in this session
**Status:** IMPLEMENTED + UNIT TESTED

---

## 1) Why This Addendum Exists

After the initial Run 003 findings, two inconsistencies remained:

1. Top-level evaluation metrics did not export behavioral quality fields already computed internally.
2. Emulator mesh behavior still extrapolated direct links beyond measured coverage and did not enforce firmware-equivalent relay-hop limits.

This addendum records the exact fixes applied in this session.

---

## 2) Fixes Implemented in This Session

### F-1: Export behavioral metrics at top-level evaluation output (FIXED)

**Problem:**
`train_convoy.py` computed `behavioral_success_rate` and `avg_pct_time_unsafe` in internal summary logic but did not include them in top-level `evaluation` metrics JSON.

**Change:**
Added both fields to top-level `metrics` payload returned by `evaluate()`.

**Files:**
- `roadsense-v2v/ml/training/train_convoy.py`
- `roadsense-v2v/ml/tests/unit/test_train_convoy.py`

**Validation:**
- `python -m pytest tests/unit/test_train_convoy.py -q` -> `12 passed`

---

### F-2: Enforce firmware-aligned relay hop cap in emulator (FIXED)

**Problem:**
Python emulator mesh relay previously propagated messages across up to `N-1` rounds without an explicit hop cap, while firmware enforces `MAX_HOP_COUNT = 3`.

**Change:**
Added emulator mesh config default:
- `mesh.max_relay_hops = 3`

Added runtime gate in relay path:
- Do not relay messages whose current hop count is already at/above max relay hops.

**Files:**
- `roadsense-v2v/ml/espnow_emulator/espnow_emulator.py`
- `roadsense-v2v/ml/tests/unit/test_espnow_emulator.py`

---

### F-3: Prevent direct-link extrapolation beyond measured distance coverage (FIXED)

**Problem:**
Mesh direct-link gating used `packet_loss.distance_threshold_2` (100m in measured params) as hard range fallback, while measured metadata indicated no sampled coverage above 50m.

**Change:**
`_mesh_max_range_m()` now prefers measured coverage inferred from `_metadata.distance_bins` when available.

Behavior:
1. If explicit `packet_loss.max_range_m` exists -> use it.
2. Else infer max observed range from non-zero-count distance bins.
3. Else fallback to `distance_threshold_2`.

This avoids simulating direct links in regions with no measured evidence.

**Files:**
- `roadsense-v2v/ml/espnow_emulator/espnow_emulator.py`
- `roadsense-v2v/ml/tests/unit/test_espnow_emulator.py`

---

### F-4: Use measured distance bins for packet-loss-vs-distance, not only generic tiers (FIXED)

**Problem:**
Even with measured params present, packet loss was still computed mostly from legacy tier interpolation and did not faithfully apply measured bin loss rates.

**Change:**
`_get_loss_probability(distance_m)` now:
1. Uses measured `_metadata.distance_bins` loss rates when valid bins are present.
2. Falls back to legacy tier formula when bins are unavailable.
3. Preserves domain-randomization effect by scaling measured bin curve by episode loss-base ratio.

This preserves monotonic loss-vs-distance behavior from measured data (e.g., 0-20m vs 20-50m).

**Files:**
- `roadsense-v2v/ml/espnow_emulator/espnow_emulator.py`
- `roadsense-v2v/ml/tests/unit/test_espnow_emulator.py`

---

## 3) New/Updated Unit-Test Coverage

Added/updated tests in `ml/tests/unit/test_espnow_emulator.py`:

- `test_mesh_uses_measured_distance_bins_to_limit_direct_links`
- `test_mesh_relay_hop_limit_blocks_far_source_in_long_chain`
- `test_measured_distance_bins_increase_loss_with_distance`
- `test_measured_distance_bins_preserve_shape_under_domain_randomization`

Also updated train-metrics test assertions:

- `test_evaluate_returns_metrics` now asserts:
  - `behavioral_success_rate`
  - `avg_pct_time_unsafe`

---

## 4) Verification Results

### Targeted tests

- `python -m pytest tests/unit/test_train_convoy.py -q` -> `12 passed`
- `python -m pytest tests/unit/test_espnow_emulator.py tests/test_packet_loss_modeling.py -q` -> `24 passed`

### Full unit regression

- `python -m pytest tests/unit/ -q` -> `173 passed`

---

## 5) Remaining Run-004 Blockers (Not Yet Implemented)

1. Hazard injection coverage is still nearest-peer dominated (insufficient source diversity for training).
2. Source-specific reaction evaluation is not yet implemented (no per-hazard-source reaction-time/quality reporting).

These are documented in:
- `docs/10_PLANS_ACTIVE/RUN_004_HAZARD_TARGETING_AND_SOURCE_REACTION_EVAL_PLAN.md`

---

## 6) Bottom Line

Session objectives completed:
- Behavioral metrics export inconsistency fixed.
- Emulator-firmware mesh alignment significantly improved (hop-cap + measured-range + measured loss bins).

Training run 004 should **not** start until hazard targeting coverage and source-specific reaction evaluation are implemented and validated.
