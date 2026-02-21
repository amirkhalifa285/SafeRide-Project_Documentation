# RoadSense V2V Project Status Overview

**Last Updated:** February 20, 2026
**Purpose:** Single source of truth for current project status and priorities.
**Audience:** AI agents and developers navigating this codebase.

---

## CURRENT PHASE: Phase 6 - Real Data Pipeline (Production Model Path)

```
================================================================================
                         READ THIS FIRST - PROJECT STATUS
================================================================================

⚠️  CRITICAL: REAL DATA RECORDING REQUIRED - January 24, 2026  ⚠️

WHAT'S HAPPENING NOW:
  Two recordings needed before Training Run 002 (production model):

  1. RTT RECORDING (2 vehicles):
     - Network characterization: latency, packet loss, jitter
     - Sensor noise: GPS, IMU, magnetometer variance
     - Output: emulator_params_measured.json
     - Status: ✅ Completed Feb 20, 2026

  2. 3-CAR CONVOY RECORDING (3 vehicles):
     - Real trajectory data for augmentation base
     - Professor requires real data, not pure synthetic
     - Output: base scenario for dataset_v2
     - Status: ⏳ Next critical step

IMMEDIATE NEXT STEPS:
  1. Acquire 3rd SD card and execute 3-car convoy recording (5-10 min)
  2. Process convoy recording → base scenario for dataset_v2
  3. Generate dataset_v2 using emulator_params_measured.json
  4. Implement cone filtering (V001 front FOV)
  5. Training Run 002: 10M steps, production model
  6. Training Run 002b: Real-data base comparison run

TRAINING STRATEGY (Feb 20 decision):
  - Run 002:  Synthetic base scenarios + measured emulator params
  - Run 002b: Real convoy base scenarios + same emulator params
  - Compare both to evaluate real vs synthetic data impact
  - AMI ready: ami-03a3037588b0f34f2 (roadsense-training-v1)

KEY DOCUMENTS:
  - **NEW:** 10_PLANS_ACTIVE/PHASE_6_REAL_DATA_PIPELINE.md ► IMMEDIATE PRIORITY
  - Architecture: 00_ARCHITECTURE/DEEP_SETS_N_ELEMENT_ARCHITECTURE.md
  - Professor Response: 10_PLANS_ACTIVE/PROFESSOR_COMMENTS_RESPONSE_PLAN.md

DO NOT:
  - Train on pure synthetic data (professor rejected this)
  - Skip the real recording step
  - Use uncalibrated emulator params for production
  - Hardcode peer slots or use fixed observation space

================================================================================
```

---

## Implementation Roadmap

```
COMPLETED                       CURRENT                         PLANNED
─────────────────────────────────────────────────────────────────────────────────

[Phase 1-4: ML Architecture]    [Phase 5: Firmware]            [Phase 6: Real Data Pipeline]
  ✅ ConvoyEnv Dict Obs            ○ LR mode enabled           ✅ 6.1 RTT Recording (enhanced)
  ✅ Deep Sets Policy              ○ Channel 6 set             ► 6.2 3-Car Convoy Recording
  ✅ Emulator Causality Fix        ○ Range tests pending       ► 6.3 Augmentation Update
  ✅ Training Pipeline                                          ► 6.4 Cone Filtering
  ✅ Run 001 (80%, n=2 only)                                    ► 6.5 ML Finalization
  ✅ EC2 Training AMI                                           ► 6.6 Training Run 002 (PROD)
                                                                ► 6.7 Training Run 002b (comparison)

CURRENT FOCUS: Phase 6.2 - 3-Car Convoy Recording (Base Scenario Capture)
─────────────────────────────────────────────────────────────────────────────────
  ✅ GPS + magnetometer logging integrated in RTT + unified firmware
  ✅ Mode 1 TX/RX logging schema upgraded (16 columns: accel+gyro+mag)
  ✅ Test updates completed (user reports 92 tests passing)
  ✅ 2-board Mode 1 button workflow validated (start/stop + TX/RX file integrity)
  ✅ Execute RTT drive and regenerate emulator_params_measured.json
  ○ Execute 3-car convoy base recording and process into base_real scenario
```

---

## Recent Achievements

### Feb 20, 2026 - EC2 Training AMI Created + Comparison Strategy Decided
- **AMI created:** `ami-03a3037588b0f34f2` (`roadsense-training-v1`) in il-central-1.
- **Contents:** Ubuntu 22.04 + Docker + `roadsense-ml:latest` image pre-built. Launch-to-training: ~2 min.
- **Cloud scripts added:** `ml/scripts/cloud/setup_ami.sh` (AMI builder) + `run_training.sh` (runtime user-data template).
- **Execution plan updated:** Run 002 config documented (10M steps, measured emulator params, dataset_v2 with variable n).
- **Training comparison strategy decided:** Train on synthetic base (Run 002) AND real convoy base (Run 002b), compare results. Addresses professor's real-data requirement with empirical evidence.
- **Repo clarification:** Cloud scripts corrected to use `roadsense-team/roadsense-v2v` repo, `master` branch.

### Feb 20, 2026 - RTT Drive Characterization Complete (Deir Hanna → Tiberias)
- **Field log captured:** `/home/amirkhalifa/RoadSense2/rtt_log.csv` with 20k+ rows and valid GPS/IMU/mag columns.
- **Measured params generated:** `roadsense-v2v/ml/espnow_emulator/emulator_params_measured.json`.
- **Calibration method:** network stats from drive-core interval; sensor-noise stats from a clean stationary window to avoid mixed-location stop bias.
- **Representative network metrics (drive-core):** RTT mean 8.0ms, p95 12ms; one-way mean 4.0ms.
- **Representative loss metrics (drive-core):** overall 33.4%, with burst-loss behavior captured for emulator robustness.
- **n-element guardrail applied:** removed `observation.monitored_vehicles` from RTT-generated params output to avoid fixed-peer confusion with Deep Sets training.

### Feb 15, 2026 - 2-Board Mode 1 Validation Pass (Hardware Session)
- **Button workflow verified end-to-end:** both boards successfully started/stopped logging from GPIO button presses.
- **Artifacts validated:** `V001_tx_003.csv`, `V001_rx_003.csv`, `V002_tx_001.csv`, `V002_rx_001.csv` in `RXTX_test_20261402/`.
- **CSV integrity checks passed:** 16-column headers/rows, monotonic local timestamps, non-zero magnetometer columns in TX/RX files.
- **Session result:** Mode 1 RX/TX logging reliability confirmed for field workflow.
- **Carry-forward note:** RTT capture + `emulator_params_measured.json` regeneration still pending.

### Feb 12, 2026 - Field Readiness Implementation Pass (Code + Tests)
- **Production mag integration complete:** `main.cpp` now fills `msg.sensors.mag` from QMC5883L with graceful fallback to zeros on failure.
- **Reusable mag driver added:** `hardware/src/sensors/mag/QMC5883LDriver.*` with shared-`Wire` initialization policy.
- **RTT firmware refined:** sender now logs mag data; `RTTPacket` extended with mag and GPS quality fields while preserving 90-byte packet size.
- **Mode 1 logging expanded:** TX/RX CSV upgraded from 10 to 16 columns (accel + gyro + mag).
- **Analysis pipeline updated:** `analyze_rtt_characterization.py` parses mag columns and outputs `sensor_noise.mag_std_ut` + mag-derived `heading_std_deg`.
- **Test updates complete:** related RTT/DataLogger/sensor tests updated, plus new QMC driver test suite added; user reports **92 tests passing**.
- **Remaining work:** regenerated measured emulator params + field-day checklist.

### Jan 24, 2026 - Real Data Pipeline Plan Finalized
- **Two-recording strategy confirmed:** RTT (2 cars) for network params, Convoy (3 cars) for trajectory base
- **RTT enhancement planned:** Add GPS + magnetometer logging for complete sensor noise characterization
- **Clock sync analysis:** GPS NMEA time is ±100-500ms (OK for trajectory alignment, not for latency measurement)
- **Data flow clarified:** SUMO provides ground truth → `to_v2v_message()` converts → Emulator adds noise
- **Key finding:** `accel_y`, `accel_z`, `gyro_*` are hardcoded in sim, BUT observation builder only uses `accel_x` so this is OK
- **Plan created:** `PHASE_6_REAL_DATA_PIPELINE.md` with 6 sub-phases to production model
- **Diagrams:** Use case and data flow diagrams updated (user completed)

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
| `ml/espnow_emulator/emulator_params_measured.json` | **CURRENT FOR RUN 002** - Regenerated from Feb 20 RTT drive (latency/loss + mag-aware sensor noise) | Ready for training |
| `ml/espnow_emulator/emulator_params_LR_final.json` | **FUTURE** - After LR mode + clean drive | Not yet created |

### Active Plans (CHECK THESE)

| Document | Purpose | Priority |
|----------|---------|----------|
| `docs/10_PLANS_ACTIVE/PHASE_6_REAL_DATA_PIPELINE.md` | **Production model path: recordings → training** | **CRITICAL - START HERE** |
| `docs/00_ARCHITECTURE/DEEP_SETS_N_ELEMENT_ARCHITECTURE.md` | n-Element problem solution | **CRITICAL - READ FIRST** |
| `docs/10_PLANS_ACTIVE/PROFESSOR_COMMENTS_RESPONSE_PLAN.md` | Professor feedback response plan | **HIGH - ACTIVE** |
| `docs/10_PLANS_ACTIVE/N_ELEMENT_IMPLEMENTATION_PLAN.md` | Implementation steps (Phases 1-5) | HIGH - REFERENCE |
| `docs/10_PLANS_ACTIVE/EC2_AMI_CREATION_PLAN.md` | Reusable training AMI | **COMPLETED** (ami-03a3037588b0f34f2) |
| `docs/10_PLANS_ACTIVE/ESPNOW_LONG_RANGE_MODE_MIGRATION.md` | LR mode validation + range tests | MEDIUM - PARALLEL |

### Completed Work (REFERENCE)

| Document | Purpose |
|----------|---------|
| `docs/90_ARCHIVE/Emulator/ESPNOW_EMULATOR_PROGRESS.md` | ESP-NOW emulator implementation (84/84 tests) |
| `docs/20_KNOWLEDGE_BASE/NETWORK_CHARACTERIZATION_GUIDE.md` | Field test lessons learned |

---

## Decision Log

### February 20, 2026: Synthetic vs Real-Data Comparison Training Strategy

**Decision:** Train two models (Run 002 on synthetic base, Run 002b on real convoy base) with identical augmentation and emulator params, then compare results.

**Rationale:**
1. Professor requires real-world data grounding, but practical ML impact is debatable
2. A controlled comparison provides empirical evidence rather than argument
3. Both models use the same measured emulator params (real RTT drive) and augmentation flags
4. The only variable is the base scenario source (synthetic SUMO vs real GPS recording)
5. Either outcome is a publishable thesis finding

**What this means for AI agents:**
- Run 002 uses existing synthetic base scenarios (`ml/scenarios/base_01`)
- Run 002b uses real convoy recording (after 3-car drive is completed and processed)
- SAME augmentation flags, emulator params, hyperparameters for both
- Compare on: success rate, reward, collision rate, generalization across peer counts

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

### January 24, 2026: Two-Recording Strategy for Real Data Collection

**Decision:** Use separate recordings for network characterization (RTT) and trajectory data (3-car convoy).

**Rationale:**
1. RTT firmware measures precise round-trip time (same device timestamps both send and receive)
2. 3-car convoy recording provides trajectory data but clock sync is ±100-500ms (GPS NMEA)
3. Mixing both in one recording would compromise precision of network measurements
4. RTT recording will be enhanced with GPS + magnetometer to capture sensor noise alongside network params

**What this means for AI agents:**
- DO use RTT recording output for `emulator_params_measured.json` (latency, loss, jitter, sensor noise)
- DO use 3-car convoy recording for base augmentation scenario (trajectories only)
- DO NOT try to measure latency from convoy recording (clock sync insufficient)

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
1. **Read:** `docs/10_PLANS_ACTIVE/PHASE_6_REAL_DATA_PIPELINE.md` ► **IMMEDIATE PRIORITY**
2. **Read:** `docs/00_ARCHITECTURE/DEEP_SETS_N_ELEMENT_ARCHITECTURE.md` (Deep Sets + V001-only cone filter)
3. **Current work:** Follow Phase 6 sub-phases in order:
   - 6.1: Enhanced RTT recording ✅ complete (capture + params regeneration)
   - 6.2: 3-car convoy recording ◄ active next step
   - 6.3-6.6: Processing, cone filter, training

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

**Document Version:** 1.4
**Maintained By:** Bookkeeper Agent
