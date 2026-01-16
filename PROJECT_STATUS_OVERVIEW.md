# RoadSense V2V Project Status Overview

**Last Updated:** January 16, 2026
**Purpose:** Single source of truth for current project status and priorities.
**Audience:** AI agents and developers navigating this codebase.

---

## CURRENT PHASE: Phase 5 - Firmware Migration (Deep Sets Complete)

```
================================================================================
                         READ THIS FIRST - PROJECT STATUS
================================================================================

⚠️  CRITICAL ARCHITECTURE CHANGE - January 13, 2026  ⚠️

WHAT CHANGED:
  The fixed 11-dimensional observation space (hardcoded for 2 peers: V002, V003)
  has been REPLACED by a Deep Sets architecture that handles VARIABLE n peers.

  This is per professor's guidance document (professors_research_for_RL.pdf).

THE n-ELEMENT PROBLEM:
  - Ego vehicle receives V2V from n neighbors where n is UNKNOWN and DYNAMIC
  - A fixed observation space CANNOT handle this
  - Solution: Permutation-invariant set encoder (Deep Sets)

WHAT WE ARE DOING NOW:
  1. Firmware migration: port Deep Sets inference to ESP32 (Phase 5)
  2. ESP-NOW LR mode migration after firmware port
  3. Production training runs with longer timesteps + eval coverage

KEY DOCUMENTS:
  - Architecture: 00_ARCHITECTURE/DEEP_SETS_N_ELEMENT_ARCHITECTURE.md
  - Implementation: 10_PLANS_ACTIVE/N_ELEMENT_IMPLEMENTATION_PLAN.md

DO NOT:
  - Hardcode peer slots (V002, V003)
  - Pad to max-n with zeros
  - Sort peers by distance
  - Use fixed Box(11,) observation space

================================================================================
```

---

## Implementation Roadmap

```
COMPLETED                       CURRENT                         PLANNED
─────────────────────────────────────────────────────────────────────────────────

[Phase 1: ConvoyEnv Dict Obs] → [Phase 2: Deep Sets Policy] → [Phase 3: Emulator Fix]
  ✅ COMPLETE                     ✅ COMPLETE                    ✅ COMPLETE

[Phase 4: Training Pipeline]   [Cloud Training: Run 001]
  ✅ COMPLETE                     ✅ COMPLETE (80% success)

                              ► [Phase 5: Firmware Migration]
                                ○ ESP-NOW LR Mode Migration
                                ○ EC2 AMI Creation (infra)
```

---

## Recent Achievements

### Jan 16, 2026 - Cloud Training Run 001 COMPLETE
- **Training completed:** 5M timesteps on c6i.xlarge (il-central-1)
- **Results:**
  - Reward: -478 → +478 (model learned successfully)
  - Evaluation: **80% success rate** (16/20 episodes)
  - 4/5 eval scenarios pass consistently; 1 edge case identified (late-spawn)
- **Model still improving** at 5M steps - consider extended run
- **Artifacts:** `s3://saferide-training-results/cloud_prod_001/`
- **Details:** See `20_KNOWLEDGE_BASE/CLOUD_TRAINING_RUN_001_RESULTS.md`

### Jan 15, 2026
- ConvoyEnv reset now waits for V001 spawn with a timeout to avoid TraCI "Vehicle not known" failures.
- Scenario generator sorts route files by depart time; dataset_v1 regenerated to satisfy SUMO ordering.
- Dataset-based training smoke run completed with evaluation output saved to `ml/models/runs/`.

## Key Files Location

### Emulator Parameters (USE THESE)

| File | Purpose | Status |
|------|---------|--------|
| `ml/espnow_emulator/emulator_params_5m.json` | **CURRENT** - Valid 5m stationary test data | Use for ConvoyEnv |
| `ml/espnow_emulator/emulator_params.json` | Copy of 5m params (or link) | Active |
| `ml/espnow_emulator/emulator_params_LR_final.json` | **FUTURE** - After LR mode + clean drive | Not yet created |

### Active Plans (CHECK THESE)

| Document | Purpose | Priority |
|----------|---------|----------|
| `docs/00_ARCHITECTURE/DEEP_SETS_N_ELEMENT_ARCHITECTURE.md` | n-Element problem solution | **CRITICAL - READ FIRST** |
| `docs/10_PLANS_ACTIVE/N_ELEMENT_IMPLEMENTATION_PLAN.md` | Implementation steps | **HIGH - START HERE** |
| `docs/10_PLANS_ACTIVE/EC2_AMI_CREATION_PLAN.md` | Create reusable training AMI | **HIGH - NEXT SESSION** |
| `docs/10_PLANS_ACTIVE/ESPNOW_LONG_RANGE_MODE_MIGRATION.md` | Next firmware change after Deep Sets | MEDIUM |

### Completed Work (REFERENCE)

| Document | Purpose |
|----------|---------|
| `docs/90_ARCHIVE/Emulator/ESPNOW_EMULATOR_PROGRESS.md` | ESP-NOW emulator implementation (84/84 tests) |
| `docs/20_KNOWLEDGE_BASE/NETWORK_CHARACTERIZATION_GUIDE.md` | Field test lessons learned |

---

## Decision Log

### January 10, 2026: Pipeline-First Approach

**Decision:** Use 5m emulator params for pipeline validation before collecting clean drive data.

**Rationale:**
1. The 5m data is valid (correct RTT, GPS, IMU characteristics)
2. Building ConvoyEnv doesn't require production-perfect data
3. Validates pipeline works before investing in field data collection
4. If architecture has bugs, we'll find them with 5m data
5. Can always retrain later with better data

**What this means for AI agents:**
- DO use `emulator_params_5m.json` for ConvoyEnv implementation
- DO NOT wait for "better" data before building the gym environment
- DO focus on getting 80/80 ConvoyEnv tests passing

### January 10, 2026: ESP-NOW LR Mode Planned

**Decision:** Plan firmware change to enable WIFI_PROTOCOL_LR after pipeline validated.

**Rationale:**
1. Current PCB antenna only works at ~5m (15.7% loss)
2. LR mode expected to extend range to 10-15m
3. Software-only fix (no hardware changes)
4. Must patch all three files: EspNowTransport.cpp, sender_main.cpp, reflector_main.cpp

**What this means for AI agents:**
- DO NOT apply LR mode patches yet (wait for firmware migration completion)
- DO reference `ESPNOW_LONG_RANGE_MODE_MIGRATION.md` when firmware changes are needed

---

## For AI Agents: What To Do

### If asked to work on ML/Training:
1. **READ FIRST:** `DEEP_SETS_N_ELEMENT_ARCHITECTURE.md` (critical architecture)
2. **IMPLEMENT:** `N_ELEMENT_IMPLEMENTATION_PLAN.md` (step-by-step)
3. **KEY RULES:**
   - Observation MUST be Dict with variable peers
   - Use DeepSetPolicy, NOT MlpPolicy
   - DO NOT hardcode V002/V003 - handle n peers dynamically
   - Use max pooling for permutation invariance
   - Ensure route files are sorted by depart time (SUMO ignores unsorted entries)

### If asked to work on ConvoyEnv:
1. Observation space is already `Dict`; do not revert to `Box(11,)`
2. EmulatorESPNOW supports n peers; avoid hardcoded V002/V003 logic
3. Reset now waits for V001 spawn; keep the startup wait/timeout
4. See `N_ELEMENT_IMPLEMENTATION_PLAN.md` Phase 1 and 2

### If asked to work on firmware:
1. ML inference must loop over n peers with shared encoder
2. Max pooling must be done in C
3. See `N_ELEMENT_IMPLEMENTATION_PLAN.md` Phase 5

### If asked about the n-element problem:
1. It's solved by Deep Sets architecture
2. φ (shared encoder) applied to each peer
3. Max pooling aggregates to fixed-size latent
4. Professor's reference: `professors_research_for_RL.pdf`

---

## Quick Reference: Test Counts

| Component | Tests | Status |
|-----------|-------|--------|
| RTT Firmware | 42/42 | ✅ Complete |
| ESP-NOW Emulator | 84/84 | ✅ Complete |
| ConvoyEnv (Dict Obs) | 74/74 | ✅ Complete |
| Deep Sets Policy | 6/6 | ✅ Complete |
| **Total** | **206/206** | **100% Complete** |

---

## Navigation

- **Full Index:** `docs/_MAPS/INDEX.md`
- **Architecture:** `docs/00_ARCHITECTURE/`
- **Active Plans:** `docs/10_PLANS_ACTIVE/`
- **Knowledge Base:** `docs/20_KNOWLEDGE_BASE/`
- **Archive:** `docs/90_ARCHIVE/`

---

**Document Version:** 1.2
**Maintained By:** Bookkeeper Agent
