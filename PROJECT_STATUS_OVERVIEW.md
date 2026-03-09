# RoadSense V2V Project Status Overview

**Last Updated:** March 8, 2026 (Run 010 completed — FIRST V2V REACTION: 87%)
**Purpose:** Single source of truth for current project status and priorities.
**Audience:** AI agents and developers navigating this codebase.

---

## CURRENT PHASE: Post-Run 010 — Fix Eval Scenarios, Then Validate + Deploy

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
  ❌ Eval matrix coverage blocks proper characterization of rank-3,4,5 behavior

BEFORE NEXT TRAINING RUN — FIX THESE:
  1. FIX EVAL SCENARIO GEOMETRY (BLOCKER)
     - 6/10 scenarios fail hazard injection with "no_front_peers"
     - Root cause: at injection time, peers are behind ego or outside cone filter
     - Must ensure all eval scenarios have >= 1 peer positioned ahead of ego
     - After fix, re-run eval with the existing Run 010 model (no retraining needed)
     - Target: all 15 (n, rank) buckets covered with 10 episodes each

  2. CHARACTERIZE RANK-2+ BEHAVIOR
     - Rank-1: 100% reaction (confirmed). Rank-2: 50-70% (incomplete data)
     - Rank-3, 4, 5: UNTESTED (0 episodes in those buckets)
     - Fix #1 above unblocks this automatically

  3. OPTIONAL — REACTION SPEED TUNING (Run 011 if needed)
     - Current: 3.25-4.0s reaction time for rank-1 hazards
     - Target: <2s for real deployment
     - Options: explicit reaction-speed bonus in reward, or steeper close-distance penalty
     - Do NOT change hyperparameters — only reward structure

NEXT STEPS (in order):
  1. Fix eval scenario geometry (all 10 scenarios)
  2. Re-evaluate Run 010 model with fixed scenarios (full 15-bucket coverage)
  3. Decide: is Run 010 model sufficient, or does rank-2+ need Run 011?
  4. H5 Sim-to-real validation against real convoy recordings
  5. TFLite INT8 quantization for ESP32
  6. Deploy on ESP32
  7. Professor PoC demo (training curve + SUMO-GUI demo)

KEY DOCUMENTS:
  - **Run 010 Analysis:** 10_PLANS_ACTIVE/RUN_010_ANALYSIS.md
  - **Run 009 Analysis:** 10_PLANS_ACTIVE/RUN_009_ANALYSIS.md
  - **Architecture:** 00_ARCHITECTURE/DEEP_SETS_N_ELEMENT_ARCHITECTURE.md
  - **Run 006 Fix Plan:** 10_PLANS_ACTIVE/RUN_006_FIX_PLAN.md

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
COMPLETED (Runs 001-010 + All Architecture)        NOW: FIX EVAL + VALIDATE
─────────────────────────────────────────────────────────────────────────────────

[Phase 1-4: ML Architecture]    [ARCH CORRECTION - COMPLETE]   [PREP - COMPLETE]
  ✅ ConvoyEnv Dict Obs            ✅ A. Cone Filter (Python)     ✅ Emulator validated
  ✅ Deep Sets Policy              ✅ B. Continuous Actions        ✅ base_real/ created
  ✅ Emulator Causality Fix        ✅ C. Emulator Mesh Relay      ✅ dataset_v3 generated
  ✅ Training Pipeline             ✅ D. ConvoyEnv Mesh            ✅ Eval n=1-5 enforced
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

CURRENT FOCUS: Fix eval scenarios → Re-evaluate → H5 Sim-to-Real → Quantize → Deploy
─────────────────────────────────────────────────────────────────────────────────
  ✅ Run 010: first model with active V2V braking (87% reaction rate)
  ► Fix eval scenario geometry (6/10 scenarios have peers behind ego)
  ► Re-evaluate Run 010 model with all 15 (n, rank) buckets covered
  ○ Run 011 (only if rank-2+ reaction or reaction speed needs improvement)
  ○ H5: Sim-to-real validation against real recordings
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

---

## EC2 Training Procedure (PROVEN — Use This Exactly)

... (rest of file unchanged)
