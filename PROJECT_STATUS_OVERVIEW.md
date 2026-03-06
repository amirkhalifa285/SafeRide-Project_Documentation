# RoadSense V2V Project Status Overview

**Last Updated:** March 5, 2026 (Run 007.1 failed at 3M steps; Run 008 "Gradient Bridge" prep in progress)
**Purpose:** Single source of truth for current project status and priorities.
**Audience:** AI agents and developers navigating this codebase.

---

## CURRENT PHASE: Phase 6.13 — Run 008 Training Prep (Gradient Bridge)

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

  RUN 003 — COMPLETED, POLICY FAILED + POST-MORTEM DONE (March 2, 2026)
  ✅ Run 003: 10M steps, avg_reward=-6511, 6 root causes fixed (169 tests)

  RUN 004 — COMPLETED, FIRST SUCCESSFUL MODEL (March 3, 2026)
  ✅ Run 004: 10M steps, avg_reward=+352, 81% behavioral success
  ⚠️ 19% collision rate (ALL 38 spawn-time, steps 3-32 — not model intelligence)
  ⚠️ 0% V2V hazard reaction (model is passive, lets CF model handle it)
  ⚠️ Eval matrix NOT used (cloud script was missing flags)

  RUN 005 — COMPLETED, REGRESSED FROM RUN 004 (March 4-5, 2026)
  ❌ Run 005: avg_reward=+175 (vs +352 in Run 004 — WORSE)
  ❌ Collision rate: 26.4% (vs 19% in Run 004 — WORSE)
  ❌ V2V hazard reaction: 0% (unchanged from Run 004)
  ❌ Root cause: 5 fixes were unit-tested but never Docker-validated with real SUMO
  ⚠️ Lesson: 201 unit tests with mocked SUMO proved NOTHING about real physics

  RUN 006 — COMPLETED, FAILED ON EC2 (March 5, 2026)
  ❌ avg_reward collapsed (~ -6,700), policy diverged
  ❌ Root cause: hazard-specific reward terms fired during normal driving
  ✅ Infrastructure fixes from Run 006 remain valid and retained

  RUN 007 — FAILED AT 500K STEPS (March 5, 2026)
  ❌ avg_reward=-7620, explained_variance=0, clip_fraction=0 at 542K steps
  ❌ Policy brain-dead: ~-10.02/step, no learning signal detected
  ❌ Root cause: comfort penalty (-7.33 at random-policy mean action 4.0 m/s²)
     drowns safety reward spread (-5 to +1 = 6.0). PPO sees flat negative reward,
     value function predicts constant, advantages are meaningless → no update.

  RUN 007.1 — FAILED AT 3M STEPS (March 5, 2026)
  ❌ avg_reward=-6990, stuck at plateau from 1.3M to 3M steps.
  ❌ Root cause: "No Man's Land" Trap (Discontinuity Cliff).
     Braking was free at 9.9m (unsafe) but full-price at 10.1m (neutral).
     Neutral zone (10-15m) gave 0.0 reward but cost -5.0 comfort to cross.
     Agent rationally decided staying in unsafe was cheaper than crossing to safe.
  ✅ Infrastructure fixes from Run 006/007 remain valid

  RUN 008 — PRE-LAUNCH PREP (March 5, 2026)
  "Gradient Bridge" overhaul based on Grok analysis:
  ✅ Fix 1: Linear Safety Ramp (5m to 20m) — ramps reward from -5.0 to +3.0.
            Eliminates zones; every cm of distance gain is rewarded.
  ✅ Fix 2: Scaled Comfort Suppression — multiplier ramps from 0.0 at 5m to 1.0
            at 20m. Comfort cost fades in smoothly as agent gets safer.
  ✅ Fix 3: Lazy Penalty — reward = -1.0 for distance > 35m.
  ✅ Fix 4: V2V Thresholds — BRAKING_ACCEL_THRESHOLD = -3.5 (prevents noise).
  ✅ 211 unit + 19 integration tests passing (local + Docker)
  🔜 Ready for EC2: cloud_prod_008, 10M steps, c6i.xlarge

  ⚠️ CRITICAL: Board Y-axis is forward (braking axis, not X).
     Always use --forward-axis y with analyze_convoy_recording.py

CURRENT STATUS:
  ✅ Run 008 "Gradient Bridge" fixes implemented and tested
  ✅ Economic cliffs eliminated; reward landscape is now a smooth slope
  ✅ All infrastructure improvements retained
  ✅ Full Docker test suites passed

NEXT STEPS:
  1. Push Run 008 code to git (master branch)
  2. Launch Run 008 on EC2 (c6i.xlarge, 10M steps)
  3. Monitor explained_variance + reward curve (expect positive crossing < 1M)
  4. H5: Validate trained model against real convoy recordings (offline replay)
  5. Quantization (TFLite INT8 for ESP32)
  6. Deploy on ESP32

KEY DOCUMENTS:
  - **Run 008 Gradient Plan:** (Grok analysis in GROK_OPINION.txt)
  - **Run 007.1 Post-Mortem:** (Internal analysis of "No Man's Land" trap)
  - **Run 007 Strategic Plan:** `10_PLANS_ACTIVE/RUN_007_STRATEGIC_ANALYSIS.md`
  - **Architecture:** 00_ARCHITECTURE/DEEP_SETS_N_ELEMENT_ARCHITECTURE.md

DO NOT:
  - Train without mesh relay in the emulator
  - Train without cone filtering
  - Train with Discrete(4) action space
  - Train on pure synthetic data
  - Use setSpeed() to "hold" current speed for action=0
  - Set UNSAFE_DIST > 10m
  - Compute reward distance from SUMO ground truth — must use mesh-visible peers
  - Trust unit tests alone — always Docker-validate with real SUMO before EC2

================================================================================
```

---

## Implementation Roadmap

```
COMPLETED (Runs 001-007 + All Architecture)        NOW: RUN 008 READY FOR EC2
─────────────────────────────────────────────────────────────────────────────────

[Phase 1-4: ML Architecture]    [ARCH CORRECTION - COMPLETE]   [PREP - COMPLETE]
  ✅ ConvoyEnv Dict Obs            ✅ A. Cone Filter (Python)     ✅ Emulator validated
  ✅ Deep Sets Policy              ✅ B. Continuous Actions        ✅ base_real/ created
  ✅ Emulator Causality Fix        ✅ C. Emulator Mesh Relay      ✅ dataset_v3 generated
  ✅ Training Pipeline             ✅ D. ConvoyEnv Mesh            ✅ Eval n=1-5 enforced
  ✅ Run 001 (80%, n=2 only)       ✅ E. Firmware Mesh             ✅ Smoke train passed
  ✅ Run 002 (BASELINE ONLY)       ✅ F. Recording Strategy
  ✅ EC2 Training AMI              ✅ G. Real Data Collection    [Run 003: FAILED]
  ✅ Convoy Recording #1                                        [Run 004: FIRST SUCCESS]
  ✅ Emulator calibrated                                          ✅ avg_reward=+352
  ✅ Recording #2 (GO) + Extra                                    ⚠️ spawn collisions
[Run 005: REGRESSED]                                           [Run 006: FAILED]
  ❌ avg_reward=+175                                             ❌ reward economy collapse
[Run 007: FAILED]                                              [Run 007.1: FAILED]
  ❌ avg_reward=-7620                                            ❌ "No Man's Land" Trap
[Run 008: Gradient Bridge Implementation]                      [PRE-LAUNCH - READY]
  ✅ Linear Safety Ramp (5-20m)                                   ✅ GT collision detection
  ✅ Scaled Comfort Suppression                                   ✅ Ego obs 4→5 dims
  ✅ V2V BRAKING_ACCEL_THRESHOLD (-3.5)                           ✅ 211 unit tests pass
  ✅ Lazy Penalty (>35m)                                          ✅ 19 integration tests pass
  ✅ Mandatory Pre-Flight Diagnostics                             ► Push → EC2 training

CURRENT FOCUS: Push to Git → Run 008 on EC2 → Analyze → H5 Sim-to-Real
─────────────────────────────────────────────────────────────────────────────────
  ✅ Run 004 completed: +352 avg reward (first working model!)
  ✅ Run 007.1 post-mortem: Discovered "No Man's Land" discontinuity trap
  ✅ Run 008 fixes: Grok-inspired Gradient Bridge (Smooth slope, no cliffs)
  ► Push to git, launch Run 008 on EC2 (10M steps, dataset_v3)
  ○ H5: Sim-to-real validation against real recordings
  ○ Quantize INT8 for ESP32
  ○ Deploy on ESP32
```

---

## EC2 Training Procedure (PROVEN — Use This Exactly)

... (rest of file unchanged)
