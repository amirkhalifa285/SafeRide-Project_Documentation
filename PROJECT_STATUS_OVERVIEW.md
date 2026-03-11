# RoadSense V2V Project Status Overview

**Last Updated:** March 10, 2026 (Run 013 in progress — hazard-gated reward fix)
**Purpose:** Single source of truth for current project status and priorities.
**Audience:** AI agents and developers navigating this codebase.

---

## CURRENT PHASE: Run 013 In Progress — Gradual Hazard + Hazard-Gated Reward Fix

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
  ► RUN 013 IN PROGRESS (March 10, 2026):
     - Gradual hazard injection retained (slowDown 2-4s)
     - New hazard-gated reward objective teaches early V2V response
     - Goal: react to peer braking signal at distance without reviving Run 006/007 poison

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
  1. ► IN PROGRESS: Run 013 training + evaluation (hazard-gated reward fix)
  2. Re-run H5 validation — must react to peer braking at distance
  3. TFLite INT8 quantization for ESP32
  4. Deploy on ESP32
  5. Professor PoC demo (training curve + SUMO-GUI demo + sim-to-real)

KEY DOCUMENTS:
  - **H5 Sim-to-Real Analysis:** 10_PLANS_ACTIVE/H5_SIM_TO_REAL_ANALYSIS.md
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
COMPLETED (Runs 001-012 + All Architecture)        NOW: RUN 013 TRAINING
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

[Run 013: IN PROGRESS]
  ✅ Run 012 postmortem complete
  ✅ Hazard-gated reward fix implemented
  ✅ 262 unit + Docker integration tests passing
  ► Training in progress with gradual hazard retained

CURRENT FOCUS: Run 013 training → Re-validate H5 → Quantize → Deploy
─────────────────────────────────────────────────────────────────────────────────
  ✅ Run 010: first model with active V2V braking (87% reaction rate)
  ✅ Formation fix: 15/15 eval buckets, zero failures
  ✅ Run 011: 100% V2V reaction in SUMO (276/276)
  ✅ H5: Sim-to-real gap identified and corrected into reward-objective diagnosis
  ✅ Run 012 postmortem: signal reached ego, reward ignored early V2V response
  ✅ Hazard-gated reward fix implemented + validated (262 unit + Docker integration)
  ► Run 013: IN PROGRESS
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
| 013 | *pending* | *pending* | IN PROGRESS | Gradual hazard retained + hazard-gated reward fix for early V2V response |

---

## EC2 Training Procedure (PROVEN — Use This Exactly)

... (rest of file unchanged)
