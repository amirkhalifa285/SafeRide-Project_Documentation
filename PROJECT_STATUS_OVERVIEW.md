# RoadSense V2V Project Status Overview

**Last Updated:** March 13, 2026 (Run 017 postmortem updated; Run 018 A+B follow-up prepared)
**Purpose:** Single source of truth for current project status and priorities.
**Audience:** AI agents and developers navigating this codebase.

---

## CURRENT PHASE: POST-RUN-017 POSTMORTEM / RUN-018 A+B FOLLOW-UP PREPARED

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
  ✅ Run 010 model (10M steps) has V2V reaction — first viable model
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
  ✅ RUN 018 FOLLOW-UP IMPLEMENTED LOCALLY (March 13, 2026):
     - Option A: stronger gradual hazard cue by shortening slowDown duration
       from 2.0-4.0s to 0.5-1.5s
     - Option B: PPO training wrapped in VecNormalize(norm_obs=False,
       norm_reward=True) with vecnormalize.pkl saved after training
     - If Run 018 still fails, re-open investigation into warmup/pre-episode
       signal contamination or another remaining observability bug

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
  1. ✅ Run 014 killed (critic dead at 987k, HAZARD_PROB=1.0)
  2. ✅ Run 015 killed (critic still dead at 300k, HAZARD_PROB=0.5 didn't help)
  3. ✅ Root cause: observation can't distinguish hazard vs calm episodes
  4. ✅ braking_received binary feature implemented (ego 5→6 dim)
  5. ✅ Docker integration tests passed for Run 016
  6. ✅ Cloud training script updated for Run 016 (RUN_ID=cloud_prod_016)
  7. ✅ Run 016 launched on EC2 (`cloud_prod_016`)
  8. ✅ Monitor explained_variance at ~300k (still dead)
  9. ✅ Kill Run 016 at ~300k (primary hypothesis failed)
  10. ✅ Run 017 fix plan drafted
  11. ✅ External review complete — diagnostic plan approved
  12. ✅ Run 017 diagnostic patch set implemented
  13. ✅ Run 017 postmortem updated with warmup-latch contamination bug
  14. ✅ Run 018 A+B follow-up implemented locally
  15. ► Run 018 and evaluate whether stronger gradual braking cue + reward normalization recover critic learning
  16. ► If Run 018 fails: investigate remaining pre-episode contamination / observability bugs before adding more training complexity

KEY DOCUMENTS:
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
COMPLETED (Runs 001-016 + All Architecture)        NOW: POST-RUN-016 ANALYSIS
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

[Run 016: KILLED @300k]
  ✅ braking_received binary feature added (ego obs 5→6 dim)
  ✅ any peer accel <= -2.5 → ego[5]=1.0 (maps to real ESP32 V2V)
  ✅ features_dim 37→38 (DeepSetExtractor updated)
  ✅ Docker integration tests passed
  ❌ explained_variance stayed ~0 through 300k
  ❌ Feature did not solve dead critic
  ✅ Root cause updated: hidden hazard timing + reward/observation mismatch remain

CURRENT FOCUS: Run 016 postmortem → redesign reward/value alignment → relaunch → H5 re-validate → Quantize → Deploy
─────────────────────────────────────────────────────────────────────────────────
  ✅ Run 010: first model with active V2V braking (87% reaction rate)
  ✅ Formation fix: 15/15 eval buckets, zero failures
  ✅ Run 011: 100% V2V reaction in SUMO (276/276)
  ✅ H5: Sim-to-real gap identified and corrected into reward-objective diagnosis
  ✅ Run 012 postmortem: signal reached ego, reward ignored early V2V response
  ✅ Run 013 postmortem: reward signal timing bug (8-16% coverage)
  ✅ Latch fix: hazard_source_braking_latched persists for episode (→99% coverage)
  ✅ base_real scenario fixed (sigma=0, speedDev=0, 25m spacing)
  ✅ All integration tests switched to base_real + ALL PASSING
  ✅ HAZARD_PROBABILITY = 1.0 selected for Run 014
  ✅ Local smoke checks complete (pipeline + dataset-based)
  ✅ V2V signal path validated locally at 100k (40/40 message/braking reception)
  ⚠️ No RL reaction yet at 100k (0/40) — treat as too-early checkpoint, not blocker
  ✅ Run 014: KILLED — critic dead (explained_variance=0), V2V reaction 0% at 1M
  ✅ Root cause: HAZARD_PROBABILITY=1.0 starved critic of contrastive episodes
  ✅ Run 015: KILLED — critic still dead (explained_variance=0 at 300k)
     - HAZARD_PROBABILITY=0.5 did NOT fix critic. Bimodal episodes confirmed but
       observation can't distinguish them (gradual slowDown signal too weak)
  ✅ braking_received binary feature added to ego obs (5→6 dim, index 5)
     - Gives critic/actor explicit 0/1 signal when any peer brakes hard
     - 266 unit tests + Docker integration tests passed before EC2 launch
  ✅ Run 016: KILLED at ~300k
     - explained_variance remained 0 / numerical noise despite braking_received
     - next fix must address reward/value alignment, not just add another obs bit
  ○ Re-validate H5 — must react to peer braking at distance
  ○ Quantize INT8 for ESP32
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

---

## EC2 Training Procedure (PROVEN — Use This Exactly)

... (rest of file unchanged)
