# RoadSense V2V Project Status Overview

**Last Updated:** March 6, 2026 (Run 008 failed — std exploded; Run 009 "Active V2V + Stability" launched)
**Purpose:** Single source of truth for current project status and priorities.
**Audience:** AI agents and developers navigating this codebase.

---

## CURRENT PHASE: Phase 6.14 — Run 009 Training (Active V2V + Stability Fixes)

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
  ❌ Root cause: 5 fixes were unit-tested but never Docker-validated with real SUMO

  RUN 006 — COMPLETED, FAILED ON EC2 (March 5, 2026)
  ❌ avg_reward collapsed (~ -6,700), policy diverged
  ❌ Root cause: hazard-specific reward terms fired during normal driving
  ✅ Infrastructure fixes from Run 006 remain valid and retained

  RUN 007 — FAILED AT 500K STEPS (March 5, 2026)
  ❌ avg_reward=-7620, explained_variance=0, clip_fraction=0 at 542K steps
  ❌ Root cause: comfort penalty drowns safety reward spread. PPO sees flat reward.

  RUN 007.1 — FAILED AT 3M STEPS (March 5, 2026)
  ❌ avg_reward=-5190 at 3M steps, std=1.59 (policy still random)
  ❌ Root cause: "No Man's Land" Trap — discrete zone boundaries create poverty trap.
     Braking was free at 9.9m (unsafe) but full-price at 10.1m (neutral).
     Neutral zone (10-15m) gave 0.0 reward but cost comfort to cross.
     Agent rationally stayed in unsafe zone at -5/step.

  RUN 008 — FAILED AT 7.6M STEPS (March 6, 2026)
  ❌ Launched: Linear Ramp "Gradient Bridge" (Grok analysis)
  ✅ Early success: std dropped to 0.753 at 1M, reward hit -4120 (ramp worked!)
  ❌ Policy exploded: std went 0.75 → 0.88 → 4.36 by 7.6M steps
  ❌ clip_fraction=0, approx_kl=0.00002, policy_gradient_loss≈0 (completely dead)
  ❌ Root causes (THREE problems):
     1. PPO TRAINING INSTABILITY: LR=3e-4 too high, n_steps=2048 too small,
        log_std unconstrained → drifted upward → policy exploded into random noise.
     2. ECONOMIC MARGINALITY: Even when stable, active V2V braking was economically
        marginal vs "do nothing" (let SUMO CF model drive for free). REWARD_SAFE=3.0
        wasn't enough margin over comfort costs to make active braking dominant.
     3. ENTROPY-DRIVEN STD DRIFT: ent_coef=0.01 dominated weak policy gradient in
        late training. |entropy_loss×ent_coef|≈0.029 vs |policy_gradient_loss|≈1e-5.
        Entropy pressure actively pushed std upward even as PPO updates died.
  ✅ Key insight: The ramp DID provide gradient (proven by -4120 recovery at 1M).
     The reward structure is fundamentally sound; needs stability + stronger incentive.

  RUN 009 — IN PROGRESS ON EC2 (March 6, 2026)
  Hybrid fix: keep ramp structure + stability fixes + stronger active V2V incentive.

  Reward changes (3 constants):
    REWARD_SAFE: 3.0 → 4.0  (stronger pull to safe zone, active braking clearly wins)
    REWARD_FAR:  -1.0 → -2.0 (stronger anti-laziness, punishes passive CF drift)
    RAMP_HIGH:   3.0 → 4.0  (steeper gradient, span 9 instead of 8)

  Stability fixes (4 hyperparameters):
    learning_rate: 3e-4 → 1e-4  (prevents oscillation that killed Run 008)
    n_steps:       2048 → 4096  (smoother gradient estimates)
    log_std_init:  0.0 → -0.5   (starts std≈0.6, prevents std explosion)
    ent_coef:      0.01 → 0.0   (removes entropy pressure that drove std drift in Run 008)

  ✅ 221 unit tests passing
  ✅ Docker integration tests passing
  🔜 Monitoring on EC2: cloud_prod_009, 10M steps, c6i.xlarge

  Expected milestones (if working):
    0-2M: reward climbs past -4000 (like Run 008 early phase)
    2-5M: std drops below 0.5, V2V reaction signal appears
    5-10M: policy convergence, positive reward

  Kill signals:
    - ep_rew_mean flat around -7000 at 3M
    - std stuck above 0.8 or rising
    - clip_fraction near zero consistently

  ⚠️ CRITICAL: Board Y-axis is forward (braking axis, not X).
     Always use --forward-axis y with analyze_convoy_recording.py

CURRENT STATUS:
  ✅ Run 009 launched on EC2 with all fixes
  ✅ Reward ramp structure preserved (proven to provide gradient)
  ✅ Stability fixes prevent Run 008's std explosion
  ✅ Stronger safe/far rewards make active braking beat passive "do nothing"

NEXT STEPS:
  1. Monitor Run 009 — check at 500K, 2M, 5M steps
  2. If successful: H5 Sim-to-real validation against real convoy recordings
  3. Quantization (TFLite INT8 for ESP32)
  4. Deploy on ESP32
  5. Professor PoC demo (training curve + SUMO-GUI demo)

KEY DOCUMENTS:
  - **Gradient Bridge Analysis:** GROK_OPINION.txt (Grok ramp analysis)
  - **Run 007 Strategic Plan:** 10_PLANS_ACTIVE/RUN_007_STRATEGIC_ANALYSIS.md
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
  - Use ent_coef > 0 (entropy pressure dominated weak policy gradient in Run 008, drove std drift)

================================================================================
```

---

## Implementation Roadmap

```
COMPLETED (Runs 001-008 + All Architecture)        NOW: RUN 009 ON EC2
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
  ✅ Recording #2 (GO) + Extra                                    ⚠️ 0% V2V reaction
[Run 005: REGRESSED]                                           [Run 006: FAILED]
  ❌ avg_reward=+175                                             ❌ reward economy collapse
[Run 007: FAILED]              [Run 007.1: FAILED]            [Run 008: FAILED]
  ❌ comfort drowns safety       ❌ poverty trap (zones)         ❌ std explosion (LR too high)
  ❌ avg_reward=-7620            ❌ avg_reward=-5190             ✅ ramp proved gradient works
[Run 009: IN PROGRESS]
  ► Steeper ramp (+4 safe, -2 far) + stability (LR 1e-4, n_steps 4096, log_std -0.5)
  ► Monitoring on EC2 (cloud_prod_009, 10M steps)

CURRENT FOCUS: Monitor Run 009 → Analyze → H5 Sim-to-Real → Quantize → Deploy
─────────────────────────────────────────────────────────────────────────────────
  ✅ Run 004: only successful model (avg_reward=+352, but passive "do nothing")
  ✅ Run 008: ramp provided gradient (hit -4120) but PPO stability failed
  ► Run 009: ramp + stronger incentive + stability = should get active V2V braking
  ○ H5: Sim-to-real validation against real recordings
  ○ Quantize INT8 for ESP32
  ○ Deploy on ESP32
  ○ Professor PoC demo
```

---

## Training Run History (Quick Reference)

| Run | Reward | Key Metric | Status | Root Cause |
|-----|--------|-----------|--------|------------|
| 003 | -6511 | trapped in penalty loop | FAILED | ActionApplicator pinned speed, PENALTY_MISSED_WARNING too aggressive |
| 004 | **+352** | 81% behavioral success | **SUCCESS** | "Do nothing" strategy — 0% V2V reaction |
| 005 | +175 | regressed from 004 | FAILED | Unit tests mocked SUMO — no Docker validation |
| 006 | -6700 | policy diverged | FAILED | Hazard reward terms fired during normal driving |
| 007 | -7620 | explained_variance=0 | FAILED | Comfort penalty drowns safety signal |
| 007.1 | -5190 | std=1.59 at 3M | FAILED | Discrete zone boundaries create poverty trap |
| 008 | -7340 | std=4.36 at 7.6M | FAILED | LR too high + "do nothing" beats active braking |
| **009** | **TBD** | **monitoring** | **IN PROGRESS** | Ramp + stability + stronger incentive |

---

## EC2 Training Procedure (PROVEN — Use This Exactly)

... (rest of file unchanged)
