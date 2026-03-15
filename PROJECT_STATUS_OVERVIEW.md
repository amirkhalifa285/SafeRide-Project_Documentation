# RoadSense V2V Project Status Overview

**Last Updated:** March 15, 2026 (Run 020 postmortem complete — SUMO passed, replay failed)
**Purpose:** Single source of truth for current project status and priorities.
**Audience:** AI agents and developers navigating this codebase.

---

## CURRENT PHASE: POST-RUN 020 — SENSITIVITY RECOVERY / SIM-TO-REAL CORRECTION

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
  ⚠️ TRAINING COMPLETE — Run 020 concluded, but deployment is still BLOCKED
  ✅ Run 020 proved deployment-compatible observation semantics can preserve
     critic health and 100% V2V reaction in SUMO
  ❌ Run 020 failed the real-data replay gate: false positives are now low, but
     sensitivity collapsed badly on both recordings
  ✅ CF override validated as the correct architectural fix
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
  8. ► Run 021 approved for implementation:
     orthogonal fixes in `hazard_injector.py` only
     a) randomize desired hazard decel in `3.0-10.0 m/s²`
     b) add resolved-hazard episodes (40% release to CF after 2-5s)
  9. ► Use `dataset_v6_formation_fix` as the explicit control dataset ID for
     the next A/B and require real-data replay to pass before any 10M extension
  10. ○ Quantize INT8 for ESP32 (TFLite) only after a new replay-passing model exists
  11. ○ Deploy on ESP32
  12. ○ Professor PoC demo

KEY DOCUMENTS:
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
COMPLETED (Runs 001-020 + All Architecture)        NOW: SENSITIVITY RECOVERY
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

CURRENT FOCUS: sensitivity-recovery retraining → preserve critic / SUMO behavior
               while recovering real-data braking sensitivity → replay gate before 10M
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
  ► Run 021 approved:
     randomize desired hazard decel in `3.0-10.0 m/s²`
     + add resolved-hazard episodes (40% release to CF after 2-5s)
     in `hazard_injector.py`, while keeping observation / reward / PPO stable
  ► Use `dataset_v6_formation_fix` as the explicit control dataset ID for the
     next A/B and require replay pass before any 10M extension
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

---

## EC2 Training Procedure (PROVEN — Use This Exactly)

... (rest of file unchanged)
