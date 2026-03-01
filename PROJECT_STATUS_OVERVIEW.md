# RoadSense V2V Project Status Overview

**Last Updated:** March 1, 2026
**Purpose:** Single source of truth for current project status and priorities.
**Audience:** AI agents and developers navigating this codebase.

---

## CURRENT PHASE: Phase 6.6 — Training Run 003

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

  PREP WORK — ALL COMPLETE (March 1, 2026)

  ✅ Step 1: Emulator params validated against Rec #2 (burst stats updated)
  ✅ Step 2: Convoy logs processed into ml/scenarios/base_real/
     - V002/V003 trajectories extracted from RX data
     - OSM exported, network generated, single-edge route (-635191227)
     - 6 vehicles: V001 ego + V002-V003 real + V004-V006 synthetic (ahead)
     - Board placement: front windshield (production-realistic)
  ✅ Step 3: dataset_v3 generated (25 train + 10 eval)
     - 100% base_real derived (no synthetic mixing)
     - Train: variable n via peer_drop_prob=0.4
     - Eval: deterministic n=1,2,3,4,5 (exactly 2 each)
     - Smoke train passed (run_20260301_025156, 1000 steps, hazards ON)

  ⚠️ CRITICAL: Board Y-axis is forward (braking axis, not X).
     Always use --forward-axis y with analyze_convoy_recording.py

IMMEDIATE NEXT STEPS:
  1. Training Run 003: 10M steps on EC2 (mesh + continuous + cone + real data)
  2. 200-episode eval (n=1-5, hazards ON, 20 episodes per scenario)
  3. Quantization (TFLite INT8 for ESP32)
  4. Deploy on ESP32

KEY DOCUMENTS:
  - **Prep Work Plan:** 10_PLANS_ACTIVE/PHASE_6_PREP_WORK_PLAN.md (ALL COMPLETE)
  - **Phase 6 Pipeline:** 10_PLANS_ACTIVE/PHASE_6_REAL_DATA_PIPELINE.md
  - **Architecture:** 00_ARCHITECTURE/DEEP_SETS_N_ELEMENT_ARCHITECTURE.md
  - **Arch Correction:** 10_PLANS_ACTIVE/MESH_AND_ACTION_ARCHITECTURE_CORRECTION.md (COMPLETE)

DO NOT:
  - Train without mesh relay in the emulator
  - Train without cone filtering
  - Train with Discrete(4) action space — must be continuous Box(1,)
  - Train on pure synthetic data (professor rejected this)
  - Use uncalibrated emulator params for production
  - Hardcode peer slots or use fixed observation space
  - Run analyzer without --forward-axis y (board Y = forward)
  - Mix synthetic base scenarios into dataset_v3 (100% base_real only)
  - Use route randomization flags with base_real (single route, silently no-ops)

================================================================================
```

---

## Implementation Roadmap

```
COMPLETED (All Prep)                              NOW: RUN 003 TRAINING
─────────────────────────────────────────────────────────────────────────────────

[Phase 1-4: ML Architecture]    [ARCH CORRECTION - COMPLETE]   [PREP - COMPLETE]
  ✅ ConvoyEnv Dict Obs            ✅ A. Cone Filter (Python)     ✅ Emulator validated
  ✅ Deep Sets Policy              ✅ B. Continuous Actions        ✅ base_real/ created
  ✅ Emulator Causality Fix        ✅ C. Emulator Mesh Relay      ✅ dataset_v3 generated
  ✅ Training Pipeline             ✅ D. ConvoyEnv Mesh            ✅ Eval n=1-5 enforced
  ✅ Run 001 (80%, n=2 only)       ✅ E. Firmware Mesh             ✅ Smoke train passed
  ✅ Run 002 (BASELINE ONLY)       ✅ F. Recording Strategy
  ✅ EC2 Training AMI              ✅ G. Real Data Collection    [Run 003: Production]
  ✅ Convoy Recording #1                                          ► Train 10M steps
  ✅ Emulator calibrated                                          ► 200-ep evaluation
  ✅ Recording #2 (GO) + Extra                                    ○ Quantize INT8
                                                                  ○ Deploy on ESP32

CURRENT FOCUS: Training Run 003
─────────────────────────────────────────────────────────────────────────────────
  ► Launch Run 003 on EC2 (10M steps, dataset_v3, all corrections active)
  ► 200-episode eval (n=1-5, 20 episodes per scenario, hazard injection ON)
  ► Compare Run 003 vs Run 002 (baseline) on n=2

  SEE: docs/10_PLANS_ACTIVE/PHASE_6_PREP_WORK_PLAN.md (ALL COMPLETE)
  SEE: docs/10_PLANS_ACTIVE/PHASE_6_REAL_DATA_PIPELINE.md
```

---

## Recent Achievements

### March 1, 2026 - Prep Work Complete — READY FOR RUN 003
- **Step 1 (Emulator validation):** `emulator_params_measured.json` validated against Recording #2. Burst mean updated from 1.245 to ~1.6. Metadata annotated with Rec #2 cross-validation. Backup created.
- **Step 2 (Convoy → SUMO):** Real convoy data processed into `ml/scenarios/base_real/`.
  - V002/V003 trajectories extracted from V001 RX log (new feature in `analyze_convoy_recording.py`).
  - OSM exported for recording area (rural road outside Deir Hanna, near Sonol station by Nahal Tzalmon).
  - Network generated via NetConvert. Route locked to single driven edge `-635191227` (single-lane, matches recording).
  - 6 vehicles: V001 (ego, rear) + V002-V003 (real trajectories) + V004-V006 (synthetic, ahead of V003).
  - Board placement: front windshield of each car (production-realistic, good line-of-sight).
- **Step 3 (dataset_v3):** 25 train + 10 eval scenarios generated from 100% base_real (no synthetic mixing).
  - Train: `peer_drop_prob=0.4` gives natural n=1-5 distribution.
  - Eval: deterministic n=1,2,3,4,5 (exactly 2 each) via `fix_eval_peer_counts.py`.
  - Blocker resolved: eval dropout with seed=42 only produced 1x n=5. Fixed by adding `--eval_peer_drop_prob 0.0` to `gen_scenarios.py` (eval keeps all peers, fixer trims to exact counts).
  - 1000-step smoke train passed (run_20260301_025156, hazard injection ON).
  - 25 unit tests pass (gen_scenarios + fix_eval_peer_counts).
- **New scripts:** `ml/scripts/process_convoy_to_sumo.py`, `ml/scripts/fix_eval_peer_counts.py`, `ml/scripts/validate_emulator_params.py`.
- **Architect review:** dataset_v3 strategy approved. Key decisions: no synthetic merge, no route randomization (single route), 200-episode eval (10 scenarios x 20 episodes = 40 per n-value).
- **Plan:** `docs/10_PLANS_ACTIVE/PHASE_6_PREP_WORK_PLAN.md` — ALL STEPS COMPLETE.

### Feb 28, 2026 - Convoy Recording #2 + Extra Data — GO VERDICT
- **Recording #2 (hazard protocol):** 195.5s ego-only mesh recording. Hard braking -8.63 m/s² raw (0.88g). Mesh relay confirmed (953 relayed packets, max hop=2). V001 as ego.
- **Extra regular driving:** 581.6s (~10 min) regular driving without protocol. Adds distance diversity (21m vs 14m spacing).
- **Combined dataset:** ~777s, 19,883 TX + 15,860 RX rows.
- **Critical discovery: Accelerometer axis mapping.** Board Y-axis is forward (braking direction), not X. Analyzer was checking wrong axis and initially reported false NO-GO. Added `--forward-axis y` flag to `analyze_convoy_recording.py`.
- **Mesh relay validated in the field:** V003 data reaches V001 through V002 relay (hop=2). V003→V001 direct PDR is 0.752 but relay compensates.
- **All architecture correction phases (A-G) now complete.** Data collection milestone achieved.
- **Data locations:** `Convoy_recording_02282026/` (main), `Convoy_extra_data_02282026/` (extra).
- **Analysis:** `ml/data/convoy_analysis_site/` and `ml/data/convoy_analysis_extra/`.

### Feb 26-27, 2026 - Architecture Correction Phases A-F Complete
- **All six phases implemented with TDD:**
  - Phase A: Cone filter in Python observation pipeline (13 tests)
  - Phase B: Continuous action space Box(1,) (36 tests)
  - Phase C: Mesh relay in ESP-NOW emulator (9 tests)
  - Phase D: Mesh in ConvoyEnv (34 tests)
  - Phase E: Firmware mesh relay (18 tests)
  - Phase F: Recording strategy updated (ego-only mesh protocol)

### Feb 26, 2026 - CRITICAL Architecture Correction (Professor Meeting)
- **THREE fundamental issues identified:**
  1. **Mesh not implemented** — vehicles only broadcast own data, no relay. PackageManager is dead code. Emulator is point-to-point.
  2. **Action space wrong** — Discrete(4) with hardcoded values defeats purpose of RL. Must be continuous Box(1,) so model learns deceleration percentage.
  3. **Cone filtering missing** — not just observation filter but determines what each vehicle RELAYS in the mesh.
- **Run 002 reclassified as BASELINE ONLY** — trained without mesh, without cone filter, with wrong action space.
- **Implementation plan created:** `MESH_AND_ACTION_ARCHITECTURE_CORRECTION.md` with 6 phases (A-F), full TDD, dependency graph.
- **Recording strategy corrected:** Professor says record only from ego — mesh delivers everything.
- **Augmentation clarified:** Take 3-vehicle recording, add virtual ego behind rear vehicle, ego gets rear vehicle's relayed data.

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
| `ml/espnow_emulator/emulator_params_measured.json` | **CURRENT** - Convoy-calibrated (Feb 24) + validated against Rec #2 (March 1) | **Active — validated, burst stats updated** |
| `ml/espnow_emulator/emulator_params_convoy.json` | Convoy-specific params (Feb 21 recording source) | Reference |
| `ml/espnow_emulator/emulator_params_measured_rtt_backup.json` | Pre-convoy RTT-only params (backup) | Archive |
| `ml/espnow_emulator/emulator_params_5m.json` | Original 5m stationary test data | Archive |
| `ml/espnow_emulator/emulator_params_pre_v3_backup.json` | Pre-validation backup (Feb 24 original) | Archive |
| `ml/data/convoy_analysis_site/convoy_emulator_params.json` | Recording #2 extracted params (Feb 28) | Reference (ego-only, inflated noise) |
| `ml/data/convoy_analysis_extra/convoy_emulator_params.json` | Extra driving extracted params (Feb 28) | Reference (ego-only, inflated noise) |

### Active Plans (CHECK THESE)

| Document | Purpose | Priority |
|----------|---------|----------|
| `docs/10_PLANS_ACTIVE/PHASE_6_PREP_WORK_PLAN.md` | **Prep work: validate + process + generate** | **COMPLETED** (Steps 1-3, March 1) |
| `docs/10_PLANS_ACTIVE/PHASE_6_REAL_DATA_PIPELINE.md` | **Production model path: recordings → training** | **CRITICAL - Run 003 next** |
| `docs/00_ARCHITECTURE/DEEP_SETS_N_ELEMENT_ARCHITECTURE.md` | n-Element problem solution | **CRITICAL - READ FIRST** |
| `docs/10_PLANS_ACTIVE/MESH_AND_ACTION_ARCHITECTURE_CORRECTION.md` | Mesh relay + continuous actions + cone filter | **COMPLETED** (Phases A-G) |
| `docs/10_PLANS_ACTIVE/N_ELEMENT_IMPLEMENTATION_PLAN.md` | Implementation steps (Phases 1-5) | REFERENCE |
| `docs/20_KNOWLEDGE_BASE/DATA_RECORDING_STRATEGY.md` | Recording protocol + axis mapping | REFERENCE |
| `docs/10_PLANS_ACTIVE/EC2_AMI_CREATION_PLAN.md` | Reusable training AMI | **COMPLETED** (ami-03a3037588b0f34f2) |

### Completed Work (REFERENCE)

| Document | Purpose |
|----------|---------|
| `docs/90_ARCHIVE/Emulator/ESPNOW_EMULATOR_PROGRESS.md` | ESP-NOW emulator implementation (84/84 tests) |
| `docs/20_KNOWLEDGE_BASE/NETWORK_CHARACTERIZATION_GUIDE.md` | Field test lessons learned |

---

## Decision Log

### February 26, 2026: Mesh Relay + Continuous Actions + Cone Filter (Professor Correction)

**Decision:** Three architectural corrections required before any further training.

**What was wrong:**
1. "Mesh" was labeled in code (V2VMessage has hopCount/sourceMAC, PackageManager deduplicates by sourceMAC) but NO component actually relays messages. Every vehicle only broadcasts its own state. The emulator and ConvoyEnv are purely point-to-point.
2. Discrete 4-action space (MAINTAIN/CAUTION/BRAKE/EMERGENCY) with hardcoded decel values defeats the purpose of RL. The model should learn the percentage of deceleration needed — that's what PPO is for.
3. Cone filtering was never implemented. It's not just an observation filter — it determines what each vehicle RELAYS in the mesh.

**What this means for AI agents:**
- Run 002 is a BASELINE only — trained without mesh, without cone filter, with wrong action space
- DO NOT start any training run until ALL three corrections are implemented and tested
- Follow `MESH_AND_ACTION_ARCHITECTURE_CORRECTION.md` phases A through F
- Every change MUST have tests written FIRST (TDD)
- Action space is now `Box(low=0.0, high=1.0, shape=(1,))` — model outputs decel fraction
- Emulator must simulate multi-hop relay, not point-to-point
- Recording strategy: only record from ego (mesh delivers everything)

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
1. **READ FIRST:** `docs/10_PLANS_ACTIVE/PHASE_6_PREP_WORK_PLAN.md` ► **ALL PREP COMPLETE**
2. **Read:** `docs/00_ARCHITECTURE/DEEP_SETS_N_ELEMENT_ARCHITECTURE.md` (Deep Sets architecture)
3. **Reference:** `docs/10_PLANS_ACTIVE/PHASE_6_REAL_DATA_PIPELINE.md` (production model path)
4. **Current work:** Run 003 — Launch training on EC2
   - Dataset: `ml/scenarios/datasets/dataset_v3/base_real/` (25 train + 10 eval, real-grounded)
   - Emulator: `ml/espnow_emulator/emulator_params_measured.json` (validated)
   - Architecture: mesh relay + continuous actions + cone filter (all active)
   - Target: 10M steps, 200-episode eval (n=1-5, hazards ON)

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
   - **Next run (Run 003):** 10M timesteps, dataset_v3 (real-grounded), 200-ep eval

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
| Scenario Gen + Eval Fix | 25/25 | ✅ Complete |
| **Total** | **231/231** | **100% Complete** |

---

## Navigation

- **Full Index:** `docs/_MAPS/INDEX.md`
- **Architecture:** `docs/00_ARCHITECTURE/`
- **Active Plans:** `docs/10_PLANS_ACTIVE/`
- **Knowledge Base:** `docs/20_KNOWLEDGE_BASE/`
- **Archive:** `docs/90_ARCHIVE/`

---

**Document Version:** 1.5
**Maintained By:** Bookkeeper Agent
