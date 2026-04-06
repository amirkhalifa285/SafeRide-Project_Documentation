# RoadSense V2V Project Status Overview

**Last Updated:** April 6, 2026 (Blind Curve Mesh Proof Extended — WORKING)
**Purpose:** Single source of truth for current project status and priorities.
**Audience:** AI agents and developers navigating this codebase.

---

## CURRENT PHASE: BLIND CURVE DEMO COMPLETE → NEXT: FINE-TUNING BEFORE QUANTIZATION

### Blind Curve V2V Demo — WORKING (March 31 – April 6, 2026)

**Goal:** Demonstrate the core V2V value proposition — ego vehicle receives a braking
warning relayed around a blind curve where there is no line-of-sight to the hazard vehicle.

**Final Configuration:** 3-vehicle convoy on a hairpin curve (Road 784).
V003 (lead, in/past the curve) emergency brakes. V002 relays to V001 (ego) via mesh.
Ego reacts via V2V before it can physically see V003 around the bend.

**Scenario:** `ml/scenarios/base_curve/` — exported from OSM, hairpin curve at ~1075-1191m.

**Mesh Proof Extension (April 6):** Added `ml/scenarios/base_curve_mesh4/` to show a
longer relay chain `V004 → V003 → V002 → V001` on the same blind curve. The demo now
prints a clean unique forwarding chain to ego (`MESH TRACE`) alongside the ego-visible
received hop counts (`MESH RX`), so relay proof is readable without exposing raw
duplicate propagation rounds.

### Changes Made (March 31 – April 3, 2026)

| File | Change | Why |
|------|--------|-----|
| `ml/scenarios/base_curve/` | New scenario: 3 vehicles, hairpin curve, Road 784, `emergencyDecel="20.0"` | Blind curve demo with hard emergency stop |
| `ml/envs/hazard_injector.py:432-442` | Fallback: inject hazard on `fixed_vehicle_id` even if outside heading-based cone filter | Curved road geometry breaks heading-based front-peer detection |
| `ml/envs/observation_builder.py:58-73` | `filter_observable_peers()` and `build()` accept `half_angle_deg` param | Configurable observation cone width for curves |
| `ml/envs/convoy_env.py` | Store `cone_half_angle_deg`, pass to `filter_observable_peers()`, `build()`, AND `simulate_mesh_step()` | Widen observation cone, build cone, AND relay cone — all three needed |
| `ml/demo_convoy_gui.py` | New CLI args: `--hazard_vehicle`, `--cone_half_angle`, `--hazard_step`, `--max_relay_hops` | Configurable curve demo without code changes |
| `ml/demo_convoy_gui.py` | `BRAKING_DURATION` 1.0s, `HAZARD_DECEL` 25.0, `HAZARD_WINDOW_START` 50 | Abrupt emergency stop; allow earlier injection |
| `ml/demo_convoy_gui.py` | Build emulator with `max_relay_hops` override, pass to ConvoyEnv | Configurable hop count for multi-vehicle scenarios |
| `ml/scenarios/base_curve_mesh4/` | New 4-vehicle blind-curve mesh proof scenario | Show `V004→V003→V002→V001` without disturbing the original 3-car RL demo |
| `ml/espnow_emulator/espnow_emulator.py` + `ml/envs/convoy_env.py` + `ml/demo_convoy_gui.py` | Capture raw relay events and reconstruct unique source→ego chain for printing | Clean professor-facing mesh proof without changing simulator behavior |

### Debugging Journey & Lessons Learned

1. **6-vehicle convoy failed** — model trained on 3-vehicle straight roads couldn't interpret observations from 5-hop relay with alien peer feature vectors (different relative headings/positions on curve)
2. **Three separate cone filters** had to be widened — relay cone (`simulate_mesh_step`), braking latch cone (`filter_observable_peers` in `_step_espnow`), and observation build cone (`build()` in `observation_builder.py`). Missing any one silently drops all peers.
3. **SUMO `decel` vType cap** — `slowDown()` is capped by the vehicle's `decel`/`emergencyDecel` params. Must set `emergencyDecel="20.0"` in vType for truly hard braking.
4. **3-vehicle topology matches training** — reducing to V001/V002/V003 (1-2 hops) produced immediate model reaction. The model generalizes to curve geometry but not to alien relay depths.

### Demo Command (Working)
```bash
cd roadsense-v2v && ./ml/run_demo_gui.sh \
  --scenario base_curve --hazard_vehicle V003 \
  --cone_half_angle 90 --hazard_step 120
```

### Longer Mesh Proof Command (Working)
```bash
cd roadsense-v2v && ./ml/run_demo_gui.sh \
  --scenario base_curve_mesh4 --hazard_vehicle V004 \
  --cone_half_angle 90 --hazard_step 120
```

**Validated terminal proof (April 6):**
- `MESH TRACE: [V004->V003, V003->V002, V002->V001]`
- `MESH RX: [V002(hop=0), V003(hop=1), V004(hop=2)]`

### Key Design Decisions
- **Wider cone is demo-only**: all other callers (training, replay, validator) default to 45°
- **All changes backward-compatible**: new params have defaults matching existing behavior

---

## NEXT STEPS: FINE-TUNING → QUANTIZATION PIPELINE

### Priority 1: Fine-Tune Model (Pre-Quantization)
- **Goal:** Improve detection rate and reduce false positive rate before INT8 quantization
- **Current baseline (Run 025, 500k):** 64.0% detection / 11.0% FP on Recording #2
- **Extra Driving FP:** ~18-20% (systematic, needs work)
- **Approach TBD:** Additional replay fine-tuning, reward adjustments, or training on mixed scenarios

### Priority 2: TFLite INT8 Quantization
- Convert fine-tuned model to TensorFlow Lite for Microcontrollers
- Target: ESP32 inference at 10Hz
- Must validate post-quantization accuracy against pre-quantization baseline

---

## PREVIOUS PHASE: RUN 025 — TEMPORAL EGO STACK — COMPLETED, TARGET MET

```
================================================================================
                         READ THIS FIRST - PROJECT STATUS
================================================================================

  ARCHITECTURE CORRECTIONS — ALL COMPLETE (Feb 26-28, 2026)

  ✅ Phase A: Cone Filter (Python observation pipeline)
  ✅ Phase B: Continuous Action Space (Box(1,) replaces Discrete(4))
  ✅ Phase C: Mesh Relay in Emulator (multi-hop simulation)
  ✅ Phase D: Mesh in ConvoyEnv (simulate_mesh_step integration)
  ✅ Phase E: Firmware Mesh Relay (ConeFilter + MeshRelayPolicy, 18 tests)
  ✅ Phase F: Recording Strategy Update (mesh-aware ego-only protocol)
  ✅ Phase G: Real Data Collection (Recording #2 + extra driving, GO verdict)

  TRAINING RUNS 003-008 — ALL FAILED (see history table below)

  RUN 009 — COMPLETED, PARTIAL SUCCESS (March 7, 2026)
  ✅ avg_reward=-3460, collision_rate=0%, behavioral_success=72%
  ✅ Perfect PPO stability: std 0.615→0.118 monotonic, explained_variance=0.97
  ❌ V2V reaction: 0% — model converged to passive "do nothing" optimum
  ❌ Root cause: SUMO CF model brakes for free during hazards. RL braking costs
     comfort penalty but gives same reward. Model rationally lets CF handle it.

  RUN 010 — COMPLETED, FIRST V2V REACTION (March 8, 2026)
  ✅ avg_reward=-3119, collision_rate=0%, behavioral_success=73.6%
  ✅ V2V REACTION: 87% (40/46 hazard episodes) — FIRST NONZERO IN PROJECT HISTORY
  ✅ Rank-1 hazards: 100% reaction across all tested peer counts (n=2,4,5)
  ✅ Avg reaction time: 3.25-4.0s (rank-1), 7-8s (rank-2)
  ✅ Zero collisions even without CF safety net
  ✅ PPO stability confirmed: std 0.608→0.120, clip_fraction active throughout
  ⚠️ Rank-2 hazards: 50-70% reaction (6 missed reactions, all rank-2)
  ⚠️ Training plateaued at 10M — no benefit from extending
  ❌ Eval matrix coverage: 12/15 buckets missing (scenario geometry bug)
     - 6/10 eval scenarios fail hazard injection (no_front_peers)
     - Failing: eval_n1_000, eval_n1_001, eval_n2_001, eval_n3_000, eval_n3_001, eval_n4_001
     - Working: eval_n2_000, eval_n4_000, eval_n5_000, eval_n5_001

  Run 010 change vs Run 009 (ONLY change):
    CF override: SUMO car-following model disabled during hazard events.
    When cf_override=True and RL action below release threshold, ego speed is
    HELD at current value (setSpeed(ego, current_speed)) instead of releasing
    to CF (setSpeed(ego, -1)). Activates 3 steps after hazard injection.
    This forces the RL agent to be the sole source of braking during hazards.

  ⚠️ CRITICAL: Board Y-axis is forward (braking axis, not X).
     Always use --forward-axis y with analyze_convoy_recording.py

CURRENT STATUS:
  ✅ RUN 024 CONCLUDED — ALL VARIANTS HIT DISCRIMINATION CEILING (March 19, 2026)
  ✅ Root cause: 4 bugs in ReplayConvoyEnv (ego shift, pre-cone braking, start bias, telemetry)
  ✅ Exploration fix: log_std reset from -2.7 (std=0.064) to -1.0 (std=0.37)
  ✅ Detection 12% → 72% on Recording #2 (v3 @ 1M steps)
  ❌ FP rate 11.3% → 68.2% in v3 — scales linearly with sensitivity
  ❌ v4 geometry gate FAILED: 8.0% det / 12.4% FP (geometry broken in recorded-ego mode)
  ✅ v5 best: 150k = 32.0% det / 16.6% FP (Pareto stalled, can't go higher w/o FP blowup)
  ❌ v6 shadow geometry FAILED: 150k = 16.0% det / 15.5% FP (too conservative)
  ❌ ROOT CAUSE: single-timestep ego (6-dim) cannot discriminate hazard onset from
     stale braking decay. Reward shaping CANNOT fix this — the observation space
     lacks discriminative power.

  ★ RUN 025 — TEMPORAL EGO STACK — COMPLETED, TARGET MET (March 20, 2026)
  ✅ SUMO TRAINING: avg_reward=+432, 0% collisions, 97.8% behavioral success
  ✅ V2V reaction: rank-1 avg ~91% (0.22-0.44s), 276 episodes, all n=1-5
  ✅ REPLAY FINE-TUNING: 1M steps, ego_stack_frames=3, LR 1e-4→1e-5
  ✅ RECORDING #2 BEST (500k): 64.0% detection (16/25), 11.0% FP — TARGET MET
  ✅ 7/10 checkpoints meet >40% det / <15% FP criterion
  ✅ Detection DOUBLED vs Run 024-v5 (32% → 64%), FP DECREASED (16.6% → 11%)
  ✅ ROOT CAUSE CONFIRMED: 3-frame ego stack provides onset-vs-decay discrimination
     that single-frame observation could not achieve
  ⚠️ Extra Driving FP: ~18-20% across all checkpoints (systematic, not overfitting)
  ✅ PoC MODEL READY: replay_ft_500000_steps.zip for professor demonstration
  ✅ PoC DEMO TOOLING READY (March 26, 2026):
     - ml/demo_convoy_gui.py: SUMO GUI demo with trained model inference
     - ml/demo_replay_plot.py: matplotlib replay visualization (3-panel event plot)
     - ml/run_demo_gui.sh: Fedora/Wayland Docker launcher
     - ml/scenarios/demo_poc/: dedicated 3-vehicle demo scenario
     - Output plot: ml/results/run_025_replay_v1/demo_poc_recording02.png
  See: 10_PLANS_ACTIVE/RUN_025_RESULTS.md for full analysis (Section 8: Demo Tooling)
  ✅ FORMATION FIX COMPLETE (March 9, 2026)
  ✅ Run 011 COMPLETE — 100% V2V reaction in SUMO (276/276), avg_reward=-86.62
  ✅ H5 SIM-TO-REAL VALIDATION COMPLETE (March 10, 2026):
     - Model discrimination intact: raw mean shifts +0.40 for hazard observations
     - Model activates on real data during close-approach (0.65-0.78 for 45s)
     - Model does NOT react to peer braking at distance in real recordings
     - Initial diagnosis (incomplete): setSpeed(0) hazard too abrupt vs real braking
     - Corrected diagnosis after Run 012: reward objective trained late geometry
       reaction, not early V2V signal usage
     - See: 10_PLANS_ACTIVE/H5_SIM_TO_REAL_ANALYSIS.md
  ✅ GRADUAL HAZARD INJECTION IMPLEMENTED (March 10, 2026):
     - hazard_injector.py: slowDown(target, 0, 2-4s) replaces setSpeed(target, 0)
     - sumo_connection.py: slow_down() wrapper added
     - maintain_hazard(): pins target at speed 0 after slowdown completes
     - Domain randomization: braking duration uniform(2.0, 4.0)s per episode
     - 253 unit tests + 19 Docker integration tests passing
  ✅ RUN 012 COMPLETE — FAILED (March 10, 2026):
     - 0% V2V reaction in SUMO (0/276 hazard episodes), 0% collisions
     - Behavioral success fell to 53.6%, avg_pct_time_unsafe rose to 28.2%
     - Real-data replay still failed: 0/25 detections on main convoy recording
     - Root cause: hazard signal reached ego, but reward ignored early V2V usage
     - See: 10_PLANS_ACTIVE/RUN_012_ROOT_CAUSE_ANALYSIS.md
  ✅ RUN 013 FIX IMPLEMENTED + VALIDATED (March 10, 2026):
     - convoy_env.py: hazard_source_braking_received gated to injected hazard source
     - reward_calculator.py: PENALTY_IGNORING_HAZARD + REWARD_EARLY_REACTION
     - BRAKING_ACCEL_THRESHOLD lowered from -3.5 to -2.5 for slowDown(2-4s)
     - 262 unit tests + Docker integration tests passing
  ✅ RUN 013 COMPLETE — FAILED (March 11, 2026):
     - 0% V2V reaction (0/276), avg_reward=-1351.68, behavioral_success=42.4%
     - Root cause: hazard_source_braking_received flag only True during 2-4s
       active slowDown, not entire hazard event. Reward signal covers 8-16%
       of post-hazard steps — too weak for PPO.
     - See: 10_PLANS_ACTIVE/RUN_013_ROOT_CAUSE_ANALYSIS.md
  ✅ LATCH FIX IMPLEMENTED (March 11, 2026):
     - convoy_env.py: _hazard_source_braking_latched persists for rest of episode
       once braking signal arrives — reward coverage 8-16% → ~99%
     - Regression tests added (unit + integration)
  ✅ BASE_REAL SCENARIO FIXED (March 11, 2026):
     - 3 formation issues: departPos gaps too small, sigma=0.5, speedDev=0.1 default
     - Fix: 25m uniform spacing, sigma=0.0, speedFactor=1.0 speedDev=0
  ✅ INTEGRATION TESTS SWITCHED TO BASE_REAL (March 11, 2026):
     - All fixtures in conftest.py + test_run006_fixes.py now use base_real
     - Docker integration tests ALL PASSING
  ✅ HAZARD_PROBABILITY SET TO 1.0 (March 11, 2026):
     - ml/envs/hazard_injector.py now injects a hazard every training episode
     - Decision rationale: latch fix restored duration; 1.0 doubles hazard exposure
  ✅ LOCAL PIPELINE SMOKE TRAIN COMPLETE (March 11, 2026):
     - 5k-timestep Docker run completed successfully and saved model
     - Important: this was only a plumbing check (default single scenario), not a behavioral check
  ✅ LOCAL DATASET SMOKE CHECK COMPLETE (March 12, 2026):
     - 100k-timestep training on dataset_v6_formation_fix completed
     - Sequential forced-hazard reaction check: 40/40 hazards injected
     - 40/40 hazard-source messages received by ego
     - 40/40 braking signals received by ego
     - 0/40 RL reactions across n=1,2,3,4,5 (0% reaction)
     - 0/40 collisions, 500-step episodes, success rate 100%
     - Interpretation: signal path is healthy; 100k timesteps is too early to block EC2
  ⚠️ LOCAL DETERMINISTIC EVAL MATRIX MISMATCH FOUND (March 12, 2026):
     - train_convoy/evaluate_model deterministic matrix path mismatched planned scenario
       vs actual env.reset() scenario during local smoke evaluation
     - Workaround for checkpoint checks: use ml/scripts/check_v2v_reaction.py
       with sequential forced-hazard eval instead of deterministic matrix mode
  ✅ RUN 014 KILLED AT ~987k STEPS (March 12, 2026):
     - 1M checkpoint V2V reaction: 0/40 (0%) — no improvement from 100k smoke
     - explained_variance = 0 for entire run — critic learned nothing
     - ep_rew_mean improved (-3.4k → -2.75k) but likely from non-hazard behavior only
     - Root cause: HAZARD_PROBABILITY=1.0 starved the critic — every episode had
       massive hazard penalty, no contrastive calm episodes for value estimation
     - S3: s3://saferide-training-results/cloud_prod_014/
     - Post-mortem: docs/10_PLANS_ACTIVE/RUN_014_CHECKPOINT_HANDOFF.md
  ✅ RUN 015 KILLED AT ~300k STEPS (March 12, 2026):
     - explained_variance still 0 at 300k — identical to Run 014's dead critic
     - HAZARD_PROBABILITY=0.5 did NOT fix the critic
     - ep_rew_mean was better (~-1500 vs ~-3200) confirming bimodal episodes work
     - Root cause identified: observation can't distinguish hazard from calm episodes.
       Gradual slowDown produces min_peer_accel ≈ -0.3 to -0.5 (normalized), similar to
       normal traffic. But PENALTY_IGNORING_HAZARD creates ~2000 point return gap between
       episode types. Critic sees same-looking states with wildly different returns → can't learn.
     - S3: s3://saferide-training-results/cloud_prod_015/
  ✅ BRAKING_RECEIVED BINARY FEATURE IMPLEMENTED + VALIDATED (March 12, 2026):
     - observation_builder.py: ego obs 5-dim → 6-dim, new ego[5] = braking_received (0/1)
     - Triggered by any_braking_peer_received (any peer accel <= -2.5 m/s²)
     - Latched per episode in convoy_env.py; Run 016 later showed this still did not exactly match the reward-driving hazard-source latch
     - validate_against_real_data.py now computes and passes braking_received
     - deep_set_extractor.py: ego_feature_dim 5→6, features_dim 37→38
     - Targeted local unit tests passed (54/54) and Docker integration tests passed
     - See: docs/10_PLANS_ACTIVE/RUN_016_BRAKING_RECEIVED_FEATURE.md
  ✅ RUN 016 KILLED AT ~300k STEPS (March 12, 2026):
     - explained_variance stayed at 0 / numerical noise (~±1e-7) through 300k
     - braking_received feature did NOT wake the critic
     - Run 016 disproved the "missing binary feature only" hypothesis
     - Remaining issue: value-target aliasing still exists before hazard onset,
       and observation/reward still do not key off the exact same event
     - 1M checkpoint was not taken; run was killed early
     - S3: s3://saferide-training-results/cloud_prod_016/
     - Postmortem: docs/10_PLANS_ACTIVE/RUN_016_BRAKING_RECEIVED_FEATURE.md
  ✅ RUN 017 DIAGNOSTIC FIX PLAN REVIEWED + APPROVED (March 12, 2026):
     - External review verified 4 code-backed claims:
       1) observation and reward use different latches
       2) stopped ego can still take ignoring-hazard penalty
       3) braking_received is currently pre-cone
       4) CF override blocks free-riding on SUMO CF once active
     - Approved Run 017 diagnostic package:
       a) align reward to the observation braking latch
       b) add progress feature (ego 6→7 dim)
       c) diagnostic timing: default hazard_step=200, HAZARD_PROBABILITY=1.0
       d) gate ignoring-hazard penalty on ego_speed only
       e) keep braking_received pre-cone for Run 017 and instrument divergence
     - Important: progress feature is diagnostic only, not final deployment input
     - See: docs/10_PLANS_ACTIVE/RUN_017_FIX_PLAN.md
  ✅ RUN 017 POSTMORTEM UPDATE (March 13, 2026):
     - Latest observed cloud log still showed explained_variance at numerical noise
       through ~569k timesteps, but this alone is NOT a reliable kill threshold:
       Run 011 first exceeded 0.1 only at 1,314,816 steps, Run 013 at 389,120
     - More important: a new code-backed bug was found in convoy_env reset/warmup
     - reset() fed _step_espnow() during warmup, and _step_espnow() could latch
       braking_received before the episode started
     - Impact: Run 016 could start with contaminated ego[5]; Run 017 could start
       with contaminated ego[5] AND contaminated reward gating, because reward was
       aligned to that same latch
     - Local fix added: warmup now populates emulator history without updating the
       episode latch, then explicitly clears braking_received before episode start
     - Targeted unit regression added and passing
  ✅ RUN 017 KILLED AT ~565k STEPS (March 13, 2026):
     - explained_variance stayed at 0 / float32 epsilon noise through ~565k
     - 5 structural fixes (reward alignment, progress feature, fixed hazard timing,
       speed-gated penalty, instrumentation) did NOT solve dead critic
     - Root cause: gradual slowDown(2-4s) produces accel signal ~-0.4 m/s² (normalized),
       indistinguishable from normal CF adjustments (-0.3 to -0.5)
     - Also: warmup contamination bug — braking_received latch could be set during
       pre-episode warmup steps. Fixed with update_braking_latch=False + explicit clear.
     - S3: s3://saferide-training-results/cloud_prod_017/
  ✅ RUN 018 COMPLETE — SUCCESS / BEST DIAGNOSTIC (March 13, 2026):
     - 2M diagnostic run — VALIDATED THE FIX
     - Fix A: BRAKING_DURATION 2-4s → 0.5-1.5s (signal strength fix)
       At 13.9 m/s: 0.5s → -27.8 m/s² (clamped to -10), 1.5s → -9.3 m/s²
       Unmistakable signal vs normal CF adjustments (-0.3 to -0.5)
     - Fix B: VecNormalize(norm_obs=False, norm_reward=True) — raw returns span
       -2500 to +2000, now unit-scale for the value function
     - Fix C: Warmup contamination fix (braking_received latch cleared after warmup)
     - RESULTS:
       ✅ V2V reaction: 100% (276/276) across ALL ranks and peer counts
       ✅ Avg reward: +691.72 (vs Run 011's -86.62 — best ever)
       ✅ Collision rate: 0%, Behavioral success: 98.2%
       ✅ Explained variance: 0.71-0.93 (critic alive from step ~28k)
       ✅ Std: 0.603 → 0.091 (still converging — room for 10M improvement)
       ✅ Reaction times: 0.21-1.49s (10x faster than Run 011's 2.88-11.02s)
       ✅ 15/15 eval matrix buckets covered
     - S3: s3://saferide-training-results/cloud_prod_018/
  ✅ RUN 019 COMPLETE — BEST SUMO MODEL / NOT DEPLOYABLE (March 14, 2026):
     - 10M production run — same config as Run 018 + Monitor wrapper
     - RESULTS:
       ✅ V2V reaction: 100% (276/276) across ALL ranks and peer counts
       ✅ Avg reward: -2.48 (median +81.38), 0% collisions
       ✅ Behavioral success: 85.9%, avg_pct_time_unsafe: 9.4%
       ✅ Explained variance: 0.94 (critic fully converged from step ~12k)
       ✅ Std: 0.602 → 0.052 (tighter than Run 018's 0.091)
       ✅ Reaction times: 0.10-2.50s, avg 0.31s (faster than Run 018)
       ✅ 15/15 eval matrix buckets covered, 0 injection failures
       ✅ Deep Sets validated: n=1 through n=5, zero collisions at every count
     - Reaction by rank (avg time / avg min distance):
       rank_1: 0.18-0.27s / 3.1-14.0m | rank_2: 0.18-0.29s / 4.5-13.3m
       rank_3: 0.25-0.43s / 7.5-10.4m | rank_4: 0.44-0.75s / 8.6-10.8m
       rank_5: 1.08s / 9.3m
     - Training trajectory: peaked at 1-3M (ep_rew_mean +500-600), then
       declined as std collapsed below 0.04. Final ep_rew_mean -436 at 10M.
       Reward regression vs Run 018 (+691→-2.48) is from tighter policy
       committing more strongly in edge-case n=1/n=2 scenarios.
     - Safety metrics UNAFFECTED by reward regression: 0 collisions, 100% reaction
     - S3: s3://saferide-training-results/cloud_prod_019/
     - Local: roadsense-v2v/ml/results/cloud_prod_019/
     - IMPORTANT: This is the best simulation model so far, but later March 14
       sim-to-real revalidation showed it is NOT deployable as-is
  ✅ RUN 019 H5 RE-VALIDATION COMPLETE (March 14, 2026):
     - Original replay semantics (`latched`, `progress=0.4`) still fail badly:
       Recording #2 = 100.0% detection / 85.3% false positives
       Extra driving = 91.3% detection / 92.0% false positives
     - Replay ablations proved sticky `braking_received` is a MAJOR issue, but
       replay fixes alone do NOT make Run 019 deployable:
       `off`, `progress=0.0` → Recording #2 = 44.0% / 6.3%, Extra = 78.3% / 21.8%
       `instant`, `progress=0.0` → Recording #2 = 52.0% / 10.6%, Extra = 78.3% / 22.5%
       `latched`, timeout=5000ms, `progress=0.0`
         → Recording #2 = 60.0% / 28.0%, Extra = 91.3% / 29.8%
     - Updated diagnosis: deployment-incompatible observation semantics
       (`braking_received`, `progress`) + incomplete real-world generalization
     - See: docs/10_PLANS_ACTIVE/RUN_019_SIM_TO_REAL_ANALYSIS.md
  ✅ RUN 020 COMPLETE — SUMO PASS / REPLAY FAIL (March 15, 2026):
     - 2M diagnostic run with deployment-compatible observation design
       (`progress` removed, `braking_received` replaced by decay)
     - Actual cloud dataset used: `dataset_v10_run020_post_cone`
       (different dataset ID from the planned preserved `dataset_v6` baseline,
       but still intended to be `base_real`-derived with canonical params)
     - SUMO results:
       ✅ 100% V2V reaction (276/276) across `n=1..5`
       ✅ 0% collisions, 98.55% behavioral success
       ✅ explained_variance healthy throughout: 0.961 near 500k, 0.925 near end
       ✅ avg_reward = +691.62 in final eval
     - Real-data replay results:
       ❌ Recording #2 = 12.0% sensitivity / 6.07% false positives
       ❌ Extra driving = 17.4% sensitivity / 7.72% false positives
     - Interpretation: the permanent-latch mismatch was removed, but the
       retrained policy became too conservative and now misses most real braking
       events even when peers are visible
     - Decision: do NOT promote to 10M, do NOT deploy, use as a negative result
       for the next sensitivity-recovery run
     - See: docs/10_PLANS_ACTIVE/RUN_020_POSTMORTEM.md
  ✅ RUN 021 COMPLETE — SUMO PASS / REPLAY FAIL (March 15, 2026):
     - 2M diagnostic with hazard decel randomization [3.0, 10.0] m/s²
       + 40% resolved-hazard episodes (release to CF after 2-5s)
     - SUMO results:
       ✅ 95.3% V2V reaction (263/276) — n=2-5 all 100%, n=1 at 77%
       ✅ avg_reward=+368, 0% collisions, 97.8% behavioral success
       ✅ PPO healthy: explained_variance 0.85-0.91, std 0.608→0.044
       ✅ Reaction times: 0.19-1.20s (very fast)
     - Real-data replay results:
       ❌ Recording #2 = 40.0% sensitivity (was 12% Run 020) / 19.16% FP
       ❌ Extra Driving = 91.3% sensitivity (was 17% Run 020) / 93.83% FP
     - Root cause: ego heading (ego[2]) is a SPURIOUS FEATURE. Model learned
       heading as a route-position proxy — heading < 0 → action=1.0 (brake max),
       heading > 0.5 → action=0.0 (calm). Extra Driving drives at heading=-0.33
       (model's "brake zone"), Recording #2 at heading=+0.78 (model's "calm zone").
     - Evidence: probing shows heading alone swings action from 0.0 to 1.0 —
       more influential than braking_received, min_peer_accel, or peer distance.
     - See: docs/10_PLANS_ACTIVE/RUN_021_HAZARD_DISTRIBUTION_FIX.md
     - S3: s3://saferide-training-results/cloud_prod_021/
  ✅ RUN 022 COMPLETE — SUMO PASS / REPLAY FAIL (March 16, 2026):
     - 2M diagnostic with ego heading removed (ego 6→5 dim, features_dim 38→37)
     - SUMO results:
       ✅ Weighted V2V reaction ≈ 64.4% (passes the >50% 2M gate, but well below Runs 020-021)
       ✅ avg_reward=+440.59, 0% collisions, 98.55% behavioral success
       ✅ Eval matrix covered peer counts n=1..5
     - Real-data replay results:
       ❌ Recording #2 = 12.0% sensitivity / 2.55% false positives
       ❌ Extra Driving = 26.1% sensitivity / 4.58% false positives
     - Interpretation: heading was the dominant FP leak in Run 021, but removing it
       alone made the policy conservative again: middling SUMO reaction plus low replay sensitivity
     - Sanity check: filtering 81 `V001` self-RX rows from Extra Driving still fails
       (21.7% sensitivity / 3.83% FP), so the no-go verdict is unchanged
     - Decision: do NOT promote to 10M, keep heading removed, draft Run 023 around
       sensitivity recovery without route-coded features
     - See: docs/10_PLANS_ACTIVE/RUN_022_POSTMORTEM.md
  ✅ RUN 023 COMPLETE — SUMO IMPROVED / REPLAY WORSE (March 17, 2026):
     - 2M diagnostic with state-triggered hazard onset (H1) + max_closing_speed (H2)
     - SUMO results:
       ✅ Weighted V2V reaction ≈ 85% (+20pp over Run 022)
       ✅ avg_reward=+416.94, 0% collisions, 98.55% behavioral success
     - Real-data replay results:
       ❌ Recording #2 = 12.0% sensitivity / 11.28% FP (WORSE FP than Run 022)
       ❌ Extra Driving = 8.7% sensitivity / 21.76% FP (WORSE on all metrics)
     - Interpretation: state-bucket broadened SUMO training diversity but domain gap
       is architectural — SUMO improvements do not transfer to real sensor data
     - Decision: move to Run 024 (real-data fine-tuning via ReplayConvoyEnv)
  ► RUN 024 — REPLAY FINE-TUNING BREAKTHROUGH (March 19, 2026):
     - ReplayConvoyEnv built: replays real peer CSVs with recorded/kinematic ego
     - Shares ObservationBuilder + RewardCalculator with ConvoyEnv (identical obs/action spaces)
     - ROOT CAUSE INVESTIGATION COMPLETE — 4 bugs found and fixed:
       1. Ego distribution shift: ReplayConvoyEnv used kinematic ego, validator used recorded
          ego positions. Added `use_recorded_ego=True` mode (matches validator exactly).
       2. Pre-cone braking detection: braking detected on ALL peers before cone filter.
          ConvoyEnv and validator both detect AFTER cone filter. Fixed to match.
       3. Episode start bias: recordings start with 20+ stationary steps (ego speed=0).
          Added `random_start=True` + stationary-skip logic.
       4. Broken telemetry: MetricsCallback got no episode stats. Added Monitor wrapper.
     - EXPLORATION FIX: SUMO-trained model std=0.064 (P(action>0.1)≈7%). Added
       `--reset_log_std -1.0` option to reset std to 0.37 before fine-tuning.
     - RESULTS:
       | Model              | Detection (Rec02) | FP (Rec02)  |
       | Base Run 023       | 12.0%             | 11.3%       |
       | v2 (fixes, no std) | 8.0%              | 12.5%       |
       | v3 @ 200k steps    | 24.0%             | 16.3%       |
       | v3 @ 400k steps    | 56.0%             | 48.3%       |
       | v3 @ 1M steps      | **72.0%**         | 68.2%       |
     - 38 unit tests (11 new), 389 total passing
     - BLOCKED: FP rate scales with sensitivity. braking_received > 0.01 for 93%
       of steps in real recordings (slow decay 0.95/step). PENALTY_IGNORING_HAZARD
       fires almost always → "always brake" is rational → FP explosion.
     - FIX IMPLEMENTED: replay-only reward gating
       now keeps early-reaction loose but tightens ignoring-hazard to require
       strong/fresh braking evidence (`braking_received >= 0.3` OR any_braking_peer)
       AND dangerous geometry (`closing_rate > 0.5` OR `distance < 20m`).
       SUMO reward defaults remain unchanged.
     - Targeted verification complete: replay reward + env unit suite 100/100 passing,
       plus py_compile on runtime files.
     - REPLAY REWARD-GATE DIAGNOSTIC RESULT (`300k`, `log_std=-1.0`):
       Recording #2 = 8.0% sensitivity / 12.41% FP
       Extra Driving = 13.0% sensitivity / 19.60% FP
     - INTERPRETATION: specificity improved but the gate over-corrected and
       killed the sensitivity breakthrough. Keep split-gate architecture, but
       relax thresholds next (`ignore_threshold≈0.2`, `danger_distance≈25m`).
     - NEXT: threshold sweep around the replay-only ignoring gate
     - See: docs/10_PLANS_ACTIVE/RUN_024_REPLAY_CONVOY_ENV_PLAN.md (Section 12)

COMPLETED BLOCKERS:
  1. ✅ EVAL SCENARIO GEOMETRY — FIXED
     - Root cause was NOT just "peers behind ego" — it was 3 compounding issues:
       a) SUMO safe-insertion stagger (V002-V003 gap 5.9m < 9.6m threshold)
       b) Episode 1000 steps (100s) on 785m route (vehicles exit at ~57s)
       c) CF model compresses formation to bumper-to-bumper
     - Fix: redistribute departPos (25-50m gaps), shorten episode (500 steps),
       move hazard window (steps 150-350), reduce sumocfg end (65s)
     - Audit: 15/15 eval buckets covered, zero failures

NEXT STEPS (in order):
  1. ✅ Runs 014-016: all killed (dead critic, gradual slowDown signal too weak)
  2. ✅ Run 017: killed at ~565k (5 structural fixes did NOT solve signal strength)
  3. ✅ Run 018: SUCCESS — faster slowDown (0.5-1.5s) + VecNormalize recovered critic
     100% V2V reaction (276/276), avg_reward=+691.72, explained_variance 0.71-0.93
  4. ✅ Run 019: 10M production run COMPLETE — 100% V2V reaction (276/276),
     avg_reward=-2.48, std 0.052, explained_variance 0.94, 0% collisions
  5. ✅ Re-validate H5 sim-to-real with Run 019 model
     Result: NOT deployable; sticky latch is major issue, replay fixes alone insufficient
  6. ✅ Design retraining plan that preserves Run 018/019 critic fixes while removing
     deployment-incompatible observation semantics
  7. ✅ Run 020 diagnostic complete
     Result: preserves critic / SUMO reaction, but replay sensitivity collapses
  8. ✅ Run 021 complete (hazard decel randomization + resolved-hazard episodes)
     Result: sensitivity improved (12→40% Rec#2, 17→91% Extra) but ego heading
     is spurious — caused 93.83% FP on Extra Driving
  9. ✅ Run 022 complete (heading removed — ego 6→5 dim, features_dim 38→37)
     Result: false positives fixed, but replay sensitivity still too low for promotion
  10. ✅ Run 023 complete (state-triggered onset + max_closing_speed)
      Result: SUMO reaction +20pp (85%) but replay WORSE on all metrics
  11. ✅ Run 024 ReplayConvoyEnv implemented + first fine-tuning attempts
      Result: code works, PPO trains, but replay sensitivity unchanged at 12%/8.7%
  12. ✅ Run 024 root cause investigation — 4 bugs fixed, sensitivity 12%→72%
      But FP rate 11.3%→68.2% — braking_received persistence causes FP explosion
  13. ✅ Run 024-v4 reward gate diagnostic — FAILED (March 19, 2026)
      Split-gate (brk≥0.3 + geometry) reduced FP but killed sensitivity: 8%/12%
      Root cause: geometry gate structurally broken in recorded-ego mode —
      human driver's reactions mask danger (distance stays safe, closing_rate low)
  14. ✅ Run 024-v5 threshold-only gate + LR decay — 150k = 32%/16.6% (Pareto stalled)
  15. ✅ Run 024-v6 shadow reward geometry — 150k = 16%/15.5% (too conservative)
  16. ✅ Run 024 concluded: discrimination ceiling — single-timestep ego lacks temporal context
  17. ► Run 025 LAUNCHED on EC2 (March 19, 2026): ego_stack_frames=3, 2M SUMO training
      After SUMO: replay fine-tune → validate on Recording #2 → target >40% det / <15% FP
      If ceiling holds: escalate to kinematic ego training (action-dependent observations)
  18. ○ Quantize INT8 for ESP32 (TFLite) only after a new replay-passing model exists
  19. ○ Deploy on ESP32
  20. ○ Professor PoC demo

KEY DOCUMENTS:
  - **Run 025 Temporal Ego Stack:** 10_PLANS_ACTIVE/RUN_025_TEMPORAL_EGO_STACK.md
  - **Run 024 Plan + Progress:** 10_PLANS_ACTIVE/RUN_024_REPLAY_CONVOY_ENV_PLAN.md
  - **Run 022 Postmortem:** 10_PLANS_ACTIVE/RUN_022_POSTMORTEM.md
  - **Run 023 Implementation Plan:** 10_PLANS_ACTIVE/RUN_023_IMPLEMENTATION_PLAN.md
  - **Run 023 Hypothesis Set:** 10_PLANS_ACTIVE/RUN_023_HYPOTHESIS_SET.md
  - **Run 022 Plan:** 10_PLANS_ACTIVE/RUN_022_REMOVE_EGO_HEADING.md
  - **Run 021 Plan + Results:** 10_PLANS_ACTIVE/RUN_021_HAZARD_DISTRIBUTION_FIX.md
  - **Run 020 Postmortem:** 10_PLANS_ACTIVE/RUN_020_POSTMORTEM.md
  - **Run 020 Plan / Design Record:** 10_PLANS_ACTIVE/RUN_020_DEPLOYMENT_COMPATIBLE_OBSERVATION.md
  - **Run 017 Fix Plan:** 10_PLANS_ACTIVE/RUN_017_FIX_PLAN.md
  - **H5 Sim-to-Real Analysis:** 10_PLANS_ACTIVE/H5_SIM_TO_REAL_ANALYSIS.md
  - **Run 013 Root Cause Analysis:** 10_PLANS_ACTIVE/RUN_013_ROOT_CAUSE_ANALYSIS.md
  - **Run 012 Root Cause Analysis:** 10_PLANS_ACTIVE/RUN_012_ROOT_CAUSE_ANALYSIS.md
  - **Run 011 Analysis:** 10_PLANS_ACTIVE/RUN_011_ANALYSIS.md
  - **Run 010 Analysis:** 10_PLANS_ACTIVE/RUN_010_ANALYSIS.md
  - **Architecture:** 00_ARCHITECTURE/DEEP_SETS_N_ELEMENT_ARCHITECTURE.md

DO NOT:
  - Train without mesh relay in the emulator
  - Train without cone filtering
  - Train with Discrete(4) action space
  - Train on pure synthetic data
  - Use setSpeed() to "hold" current speed for action=0
  - Compute reward distance from SUMO ground truth — must use mesh-visible peers
  - Trust unit tests alone — always Docker-validate with real SUMO before EC2
  - Use learning_rate > 1e-4 (caused std explosion in Run 008)
  - Leave log_std unconstrained (must set log_std_init=-0.5 or lower)
  - Use ent_coef > 0 (entropy pressure dominated weak policy gradient in Run 008)
  - Disable CF override in future training runs (it's what enabled V2V reaction)
  - Change hyperparameters mid-run if extending training

================================================================================
```

---

## Implementation Roadmap

```
COMPLETED (Runs 001-024 + All Architecture)        NOW: RUN 025 TEMPORAL EGO STACK
─────────────────────────────────────────────────────────────────────────────────

[Phase 1-4: ML Architecture]    [ARCH CORRECTION - COMPLETE]   [PREP - COMPLETE]
  ✅ ConvoyEnv Dict Obs            ✅ A. Cone Filter (Python)     ✅ Emulator validated
  ✅ Deep Sets Policy              ✅ B. Continuous Actions        ✅ base_real/ created
  ✅ Emulator Causality Fix        ✅ C. Emulator Mesh Relay      ✅ dataset_v6 generated
  ✅ Training Pipeline             ✅ D. ConvoyEnv Mesh            ✅ Eval 15/15 buckets
  ✅ Run 001 (80%, n=2 only)       ✅ E. Firmware Mesh             ✅ Smoke train passed
  ✅ Run 002 (BASELINE ONLY)       ✅ F. Recording Strategy
  ✅ EC2 Training AMI              ✅ G. Real Data Collection
  ✅ Convoy Recording #1
  ✅ Emulator calibrated
  ✅ Recording #2 (GO) + Extra

[Runs 003-008: ALL FAILED]     [Run 009: STABLE BUT PASSIVE]  [Run 010: BREAKTHROUGH]
  ❌ 003: speed pinning           ✅ avg_reward=-3460             ✅ avg_reward=-3119
  ❌ 004: passive (0% V2V)        ✅ PPO stability solved         ✅ V2V reaction: 87%
  ❌ 005: mocked SUMO             ❌ V2V reaction: 0%             ✅ 0% collision rate
  ❌ 006: reward economy          ❌ CF model did all braking     ✅ CF override worked
  ❌ 007: comfort drowns signal                                   ⚠️ Eval coverage: 3/15
  ❌ 007.1: poverty trap
  ❌ 008: std explosion

[FORMATION FIX - COMPLETE]     [Run 011: COMPLETE]            [Run 012: POST-MORTEM]
  ✅ Root cause: 3 issues           ✅ 100% V2V reaction (276/276)   ✅ Offline replay done
     (insertion stagger,            ✅ avg_reward=-86.62             ✅ Model activates on real data
      episode too long,             ✅ 0% collisions                 ⚠️ Doesn't react to V2V signal
      CF compression)               ✅ 15/15 eval buckets              at distance (sim-to-real gap)
  ✅ departPos redistribution       ✅ PPO std 0.60→0.04             ✅ Initial diagnosis: setSpeed(0)
  ✅ Episode 1000→500 steps                                            too abrupt vs real braking
  ✅ Hazard window 30-80→150-350                                     ✅ Final diagnosis: reward
                                                                     objective ignored early
                                                                     V2V response
                                                                     ✅ Fix: hazard-gated reward
                                                                     + threshold -2.5
  ✅ Audit: 15/15 buckets

[Run 013: FAILED]              [Run 014: KILLED]              [Run 015: KILLED]
  ✅ 0% V2V reaction (0/276)       ✅ Latch fix implemented          ✅ HAZARD_PROB 0.5 (was 1.0)
  ✅ avg_reward=-1351.68            ✅ base_real scenario fixed       ❌ explained_variance still 0
  ✅ Root cause: reward signal      ❌ explained_variance=0              at 300k — HAZARD_PROB was
     only 8-16% of hazard steps    ❌ V2V reaction 0% at 1M             NOT the critic problem
  ✅ Latch fix implemented          ❌ Killed at ~987k steps          ✅ Root cause: observation
  ✅ base_real formation fixed      ✅ Root cause: HAZARD_PROB=1.0       can't distinguish hazard vs
                                       (partial — not sole cause)        calm states with gradual
                                                                         slowDown signal

[Run 016: KILLED @300k]         [Run 017: KILLED @565k]
  ✅ braking_received feature        ✅ 5 structural fixes applied
     added (ego obs 5→6 dim)         ✅ Reward aligned to obs gate
  ❌ explained_variance ~0           ✅ Progress feature (ego 6→7)
  ❌ Feature didn't fix critic       ❌ explained_variance still ~0
                                     ❌ slowDown signal still too weak

[Run 018: SUCCESS — DIAGNOSTIC]   [Run 019: BEST SUMO MODEL / NOT DEPLOYABLE]
  ✅ BRAKING_DURATION 0.5-1.5s       ✅ 10M production run COMPLETE
     (was 2-4s → signal now -10)     ✅ 100% V2V reaction (276/276)
  ✅ VecNormalize(norm_reward=True)   ✅ avg_reward=-2.48 (median +81)
  ✅ Warmup contamination fix        ✅ 0% collisions, 85.9% behav success
  ✅ 100% V2V reaction (276/276)     ✅ std 0.602→0.052, expl_var 0.94
  ✅ avg_reward=+691.72 (2M BEST)    ✅ Reaction: 0.10-2.50s (avg 0.31s)
  ✅ explained_variance 0.71-0.93    ✅ 15/15 eval, Deep Sets n=1-5
  ✅ Reaction: 0.21-1.49s (10x↑)    ⚠️ Replay-corrected sim-to-real still fails
  ✅ 15/15 eval, 0% collisions

[Run 020: SUMO PASS / REPLAY FAIL]
  ✅ Deployment-compatible ego obs
     (remove `progress`, binary latch → decay)
  ✅ 2M diagnostic completed, critic healthy
  ✅ 100% V2V reaction (276/276), 0% collisions in SUMO
  ✅ False positives now low on real replay
  ❌ Recording #2 replay: 12.0% sensitivity / 6.07% FP
  ❌ Extra replay: 17.4% sensitivity / 7.72% FP
  ❌ Not promotable to 10M; next focus is sensitivity recovery

CURRENT FOCUS: Run 025 — Temporal Ego Stack (ego_stack_frames=3, 18-dim ego, features_dim=50)
               → SUMO training on EC2 → replay fine-tune → target >40% det / <15% FP on Rec#2
─────────────────────────────────────────────────────────────────────────────────
  ✅ Runs 012-016: gradual slowDown signal too weak for critic (5 consecutive failures)
  ✅ Run 017: 5 structural fixes still failed — warmup contamination found & fixed
  ✅ Run 018: BRAKING_DURATION 0.5-1.5s + VecNormalize RECOVERED CRITIC
     100% V2V reaction (276/276), avg_reward=+691.72, explained_variance 0.71-0.93
     Reaction times 10x faster than Run 011 (0.21-1.49s vs 2.88-11.02s)
  ✅ Run 019: 10M PRODUCTION COMPLETE — 100% V2V reaction (276/276), 0% collisions
     std 0.052, explained_variance 0.94, reaction times 0.10-2.50s (avg 0.31s)
     BUT real-data replay still shows non-deployable sensitivity/specificity tradeoff
  ✅ Run 019 H5 re-validation + replay ablations COMPLETE
     sticky latch is major issue; replay fixes alone still leave either low
     sensitivity or high false positives
  ✅ Run 020 complete
     deployment-compatible observation semantics preserved critic / SUMO reaction,
     but replay sensitivity collapsed to 12-17% despite low false positives
  ✅ Run 021 complete (hazard decel randomization + resolved-hazard episodes)
     SUMO PASS: 95.3% V2V reaction, avg_reward=+368, 0% collisions
     Replay FAIL: sensitivity improved but ego heading caused 93.83% FP on Extra
  ✅ Run 022 complete (heading removed — ego 6→5 dim)
     SUMO PASS: ~64.4% weighted reaction, avg_reward=+440.59, 0% collisions
     Replay FAIL: Rec#2 12.0% / 2.55%, Extra 26.1% / 4.58%
  ✅ Run 023 complete (state-triggered onset + max_closing_speed)
     SUMO +20pp (85%) but replay WORSE — domain gap is architectural
  ✅ Run 024 CONCLUDED (replay fine-tuning: v3-v6 all hit discrimination ceiling)
     v3: 72%/68% | v5: 32%/16.6% | v6: 16%/15.5% — single-timestep ego can't
     discriminate onset vs decay. Reward shaping exhausted.
  ► Run 025 LAUNCHED on EC2 — ego_stack_frames=3 (18-dim ego, features_dim=50)
     Fresh 2M SUMO training, then replay fine-tune, target >40% det / <15% FP
  ○ Quantize INT8 for ESP32 only after a replay-passing model exists
  ○ Deploy on ESP32
  ○ Professor PoC demo
```

---

## Training Run History (Quick Reference)

| Run | Reward | V2V Reaction | Status | Root Cause / Key Result |
|-----|--------|-------------|--------|------------------------|
| 003 | -6511 | 0% | FAILED | ActionApplicator pinned speed, PENALTY_MISSED_WARNING too aggressive |
| 004 | +352 | 0% | Partial | First working model, but passive — CF did all braking |
| 005 | +175 | 0% | FAILED | Unit tests mocked SUMO — no Docker validation |
| 006 | -6700 | 0% | FAILED | Hazard reward terms fired during normal driving |
| 007 | -7620 | 0% | FAILED | Comfort penalty drowns safety signal |
| 007.1 | -5190 | 0% | FAILED | Discrete zone boundaries create poverty trap |
| 008 | -7340 | 0% | FAILED | LR too high + entropy-driven std drift |
| 009 | -3460 | 0% | Partial | Perfect stability, but passive — CF free-ride problem |
| **010** | **-3119** | **87%** | **BREAKTHROUGH** | **CF override forced RL to brake — first V2V reaction** |
| **011** | **-86.62** | **100%** | **BEST** | **Formation fix + 15/15 eval buckets — 276/276 reactions** |
| 012 | -1135.22 | 0% | FAILED | Gradual hazard exposed reward-objective gap; signal reached ego but reward ignored early V2V response |
| 013 | -1351.68 | 0% | FAILED | Reward signal only fires 8-16% of hazard steps; latch fix implemented for Run 014 |
| 014 | -2750* | 0% | KILLED @987k | Latch fix worked but HAZARD_PROB=1.0 starved critic (explained_variance=0 entire run) |
| 015 | -1500* | 0%* | KILLED @300k | HAZARD_PROB=0.5 fixed bimodal reward but critic still dead — observation can't distinguish episode types |
| 016 | TBD | TBD | KILLED @300k | braking_received binary feature did not wake critic; remaining issue is hidden hazard timing + reward/observation mismatch |
| 017 | TBD | TBD | KILLED @565k | 5 structural fixes (reward alignment, progress, timing, speed-gate, instrumentation) still failed — gradual slowDown signal indistinguishable from normal CF |
| **018** | **+691.72** | **100%** | **SUCCESS (2M diagnostic)** | **BRAKING_DURATION 0.5-1.5s + VecNormalize recovered critic — 276/276 reactions, 0.21-1.49s reaction times, explained_variance 0.71-0.93** |
| **019** | **-2.48** | **100%** | **COMPLETE — BEST SUMO MODEL / NOT DEPLOYABLE** | **10M production: 276/276 reactions, 0.10-2.50s reaction times (avg 0.31s), std 0.052, explained_variance 0.94, 0% collisions. Real-data replay failed: original semantics gave 85-92% false positives, and replay ablations still showed inadequate sensitivity/specificity. Use as retraining baseline, not for deployment.** |
| **020** | **+691.62** | **100%** | **COMPLETE — SUMO PASS / REPLAY FAIL** | **2M diagnostic with deployment-compatible observation semantics: critic healthy (`explained_variance` 0.961 near 500k, 0.925 near end), 276/276 SUMO reactions, 0% collisions, 98.55% behavioral success. Real-data replay removed the catastrophic false positives but collapsed sensitivity: Recording #2 = 12.0% / 6.07%, Extra = 17.4% / 7.72%. Do not promote to 10M or deploy.** |
| **021** | **+368.00** | **95.3%** | **COMPLETE — SUMO PASS / REPLAY FAIL** | **Hazard decel randomization + resolved-hazard episodes improved replay sensitivity, but ego heading became a spurious route-position proxy. Recording #2 = 40.0% / 19.16%, Extra = 91.3% / 93.83%. Do not promote.** |
| **022** | **+440.59** | **64.4%*** | **COMPLETE — SUMO PASS / REPLAY FAIL** | **Heading removed (ego 6→5 dim, features_dim 38→37). Final eval still cleared the 2M SUMO gate: ~64.4% weighted reaction from `source_reaction_summary`, 0% collisions, 98.55% behavioral success, n=1..5 covered. Replay fixed specificity but not sensitivity: Recording #2 = 12.0% / 2.55%, Extra = 26.1% / 4.58%. Keep heading removed, but do not promote to 10M.** |
| **023** | **+416.94** | **~85%** | **COMPLETE — SUMO IMPROVED / REPLAY WORSE** | **State-triggered onset (H1) + max_closing_speed (H2). SUMO +20pp over Run 022 (85% vs 64.4%), 0% collisions, 98.55% behavioral success. Replay WORSE: Rec#2 = 12.0% / 11.28% FP, Extra = 8.7% / 21.76% FP. Domain gap is architectural, not diversity-limited.** |
| **024** | N/A | N/A | **CONCLUDED — DISCRIMINATION CEILING** | **Replay fine-tuning: 4 bugs fixed + log_std reset → v3: 72.0% det / 68.2% FP. v4: geometry gate broken in recorded-ego mode (8%/12%). v5: threshold-only best 150k = 32%/16.6% (Pareto stalled). v6: shadow geometry 150k = 16%/15.5% (too conservative). Root cause: single-timestep ego (6-dim) cannot distinguish hazard onset from stale braking decay. Reward shaping exhausted.** |
| **025** | TBD | TBD | **► LAUNCHED ON EC2** | **Temporal ego stack: ego_stack_frames=3, ego obs 6→18 dim, features_dim 38→50. Stacks [ego_t, ego_{t-1}, ego_{t-2}] so network can see onset jumps vs decay tails. Fresh 2M SUMO training (no weight transfer). 426 unit + 20 integration tests pass. After SUMO: replay fine-tune with --ego_stack_frames 3. Target: >40% det / <15% FP on Rec#2. Escalation: kinematic ego training.** |

\* Run 022's `metrics.json` does not store one top-level overall reaction field; `64.4%` is the weighted aggregate reconstructed from `source_reaction_summary`.

---

## EC2 Training Procedure (PROVEN — Use This Exactly)

... (rest of file unchanged)
