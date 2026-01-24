# RoadSense V2V Project Status Overview

**Last Updated:** January 23, 2026
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
  2. ESP-NOW LR mode validation (enabled; extended range tests pending)
  3. Production training runs with longer timesteps + eval coverage (Run 001 was a test run)
  4. Professor feedback alignment: real-recording base, **cone filtering inside V001 AI inference**, diagram fixes

KEY DOCUMENTS:
  - Architecture: 00_ARCHITECTURE/DEEP_SETS_N_ELEMENT_ARCHITECTURE.md
  - Implementation: 10_PLANS_ACTIVE/N_ELEMENT_IMPLEMENTATION_PLAN.md
  - Professor Response: 10_PLANS_ACTIVE/PROFESSOR_COMMENTS_RESPONSE_PLAN.md

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
  ✅ COMPLETE                     ✅ COMPLETE (test run; 80% success, n=2 only)

                              ► [Phase 5: Firmware Migration]
                                ○ Phase 5.1 LR mode enabled + max TX power
                                ○ Channel set to 6 for home testing
                                ○ Extended range tests pending (10m/15m/20m)
                              ► [Phase 6: Training Run 002]     ← NEW: Variable n, 10M steps
                                ○ Dataset v2 with n ∈ {1,2,3,4,5}
                                ○ EC2 AMI Creation (infra)
```

---

## Recent Achievements

### Jan 23, 2026 - Professor Feedback Alignment (Docs + Plan)
- **Hybrid data strategy documented:** real 3-car recording is the **mandatory base** for augmentation
- **Cone filtering clarified:** V001-only step inside AI inference before Observation Builder (front FOV)
- **Use-case boundary clarified:** sensors treated as external actors (firmware boundary)
- **Next work:** update draw.io diagrams + implement cone filter in embedded + sim

### Jan 22, 2026 - Scenario Generator Enhanced for Variable Peer Counts
- **gen_scenarios.py enhanced:** Added 4 new CLI arguments for peer variation:
  - `--peer_drop_prob`: Probability of dropping each non-V001 vehicle (0.0-1.0)
  - `--min_peers`: Minimum peers to keep after dropout
  - `--route_randomize_non_ego`: Randomize route assignment for non-V001 vehicles
  - `--route_include_v001`: Include V001 in route randomization
- **New features:** `apply_peer_dropout()`, `randomize_routes()`, `AugmentationConfig` dataclass
- **Manifest enhanced:** Now includes `augmentation_config` and `peer_count_distribution`
- **Tests:** 17 new unit tests added; all tests pass
- **SCENARIO_BUILDING.txt updated:** Spec corrected for n ∈ {1,2,3,4,5} peer counts
- **Impact:** Unblocks dataset_v2 generation for Training Run 002

### Jan 19, 2026 - LR Mode Enabled + Channel 6 Standardized
- **LR mode enabled:** ESP-NOW LR mode + max TX power applied in transport + RTT firmware
- **Channel updated:** Default channel set to 6 to avoid home WiFi interference (routers often on 1)
- **Validation:** Builds + unit tests pass; 5m baseline RTT test passes
- **Pending:** Extended range tests (10m/15m/20m) before updating emulator params

### Jan 17, 2026 - CRITICAL FINDING: Fixed n=2 Training Data
- **Issue Discovered:** All 25 scenarios in `dataset_v1` had exactly 3 vehicles (V001 + 2 peers)
- **Impact:** Model trained on constant n=2 for entire 5M timesteps; behavior for n≠2 is untested
- **Resolution:**
  - Removed `monitored_vehicles` from emulator params (was misleading, not used by ConvoyEnv)
  - Added Phase 6 to N_ELEMENT_IMPLEMENTATION_PLAN.md with variable-n requirements
  - Next training run (Run 002) MUST use variable peer counts n ∈ {1,2,3,4,5}
- **NOT a catastrophic failure:** Deep Sets architecture is correct and should generalize
- **Details:** See `N_ELEMENT_IMPLEMENTATION_PLAN.md` Phase 6

### Jan 16, 2026 - Cloud Training Run 001 COMPLETE
- **Training completed:** 5M timesteps on c6i.xlarge (il-central-1)
- **Results:**
  - Reward: -478 → +478 (model learned successfully)
  - Evaluation: **80% success rate** (16/20 episodes)
  - 4/5 eval scenarios pass consistently; 1 edge case identified (late-spawn)
- **⚠️ CAVEAT:** Model only trained/evaluated with n=2 peers (see Jan 17 finding)
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
| `docs/10_PLANS_ACTIVE/PROFESSOR_COMMENTS_RESPONSE_PLAN.md` | Professor feedback response plan | **HIGH - ACTIVE** |
| `docs/10_PLANS_ACTIVE/EC2_AMI_CREATION_PLAN.md` | Create reusable training AMI | **HIGH - NEXT SESSION** |
| `docs/10_PLANS_ACTIVE/ESPNOW_LONG_RANGE_MODE_MIGRATION.md` | LR mode validation + range tests | HIGH - VALIDATION |

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

### January 19, 2026: ESP-NOW LR Mode Enabled + Channel 6 Default

**Decision:** Enable WIFI_PROTOCOL_LR + max TX power and standardize on channel 6 for home testing.

**Rationale:**
1. Current PCB antenna only works at ~5m (15.7% loss)
2. LR mode expected to extend range to 10-15m
3. Software-only fix (no hardware changes)
4. Must patch all three files: EspNowTransport.cpp, sender_main.cpp, reflector_main.cpp

**What this means for AI agents:**
- DO flash LR-enabled firmware on all devices (sender/reflector/main)
- DO keep all ESP-NOW devices on the same channel (default: 6 for home tests)
- DO complete extended range tests before updating emulator params

### January 23, 2026: Hybrid Data Strategy + Cone Filtering Clarification

**Decision:** Ground augmentation in a real 3-car recording and apply cone filtering only on V001 **inside AI inference** before observation building.

**Rationale:**
1. Professor requires a real-data basis before augmentation
2. Cone filtering is a receiver-side perception filter; broadcasters remain dumb
3. Training and inference must match observation filtering

**What this means for AI agents:**
- DO treat the real recording as the **mandatory base** for augmentation
- DO implement cone filtering **only on V001** (front FOV) and mirror it in simulation

---

## For AI Agents: What To Do

### If returning after a break (start here)
1. **Read:** `docs/10_PLANS_ACTIVE/PROFESSOR_COMMENTS_RESPONSE_PLAN.md` (professor alignment plan)
2. **Read:** `docs/00_ARCHITECTURE/DEEP_SETS_N_ELEMENT_ARCHITECTURE.md` (Deep Sets + V001-only cone filter)
3. **Read:** `docs/20_KNOWLEDGE_BASE/ML_AUGMENTATION_PIPELINE.md` (hybrid real-base pipeline)
4. **First task:** Implement **V001-only cone filtering** in embedded inference and mirror in simulation observation builder

### If asked to work on ML/Training:
1. **READ FIRST:** `DEEP_SETS_N_ELEMENT_ARCHITECTURE.md` (critical architecture)
2. **IMPLEMENT:** `N_ELEMENT_IMPLEMENTATION_PLAN.md` (step-by-step, see Phase 6 for next run)
3. **KEY RULES:**
   - Observation MUST be Dict with variable peers
   - Use DeepSetPolicy, NOT MlpPolicy
   - DO NOT hardcode V002/V003 - handle n peers dynamically
   - Use max pooling for permutation invariance
   - Ensure route files are sorted by depart time (SUMO ignores unsorted entries)
   - **Training datasets MUST vary peer count** (n ∈ {1,2,3,4,5}), not just vehicle params
   - **Next run (Run 002):** 10M timesteps with evaluation, variable n dataset

### If asked to work on ConvoyEnv:
1. Observation space is already `Dict`; do not revert to `Box(11,)`
2. EmulatorESPNOW supports n peers; avoid hardcoded V002/V003 logic
3. Reset now waits for V001 spawn; keep the startup wait/timeout
4. See `N_ELEMENT_IMPLEMENTATION_PLAN.md` Phase 1 and 2

### If asked to create training scenarios/datasets:
1. **User manually:** Crop real-world map from OpenStreetMap, import to SUMO (NetConvert)
2. **User manually:** Create base scenario with route file and sumocfg
3. **Script:** Use `generate_dataset.py` to augment the base scenario
4. **CRITICAL:** The generator script MUST vary the number of vehicles per scenario
   - Generate scenarios with n ∈ {1, 2, 3, 4, 5} peers (plus V001 ego)
   - Do NOT generate all scenarios with the same vehicle count
   - Vary vehicle params (decel, tau, sigma, speedFactor) AND vehicle count
5. See `N_ELEMENT_IMPLEMENTATION_PLAN.md` Phase 6 for requirements

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
