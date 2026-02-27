# RoadSense V2V Documentation Index

**Last Updated:** February 27, 2026
**Total Documents:** 67

---

## CURRENT PROJECT STATUS

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  ARCHITECTURE CORRECTIONS (Feb 26-27, 2026 — Professor Meeting)             │
├─────────────────────────────────────────────────────────────────────────────┤
│  ✅ Phase A: Cone Filter (Python observation pipeline)                       │
│  ✅ Phase B: Continuous Action Space (Box(1,) replaces Discrete(4))          │
│  ✅ Phase C: Mesh Relay in Emulator (multi-hop simulation)                   │
│  ✅ Phase D: Mesh in ConvoyEnv (simulate_mesh_step integration)              │
│  ✅ Phase E: Firmware Mesh Relay (ConeFilter + MeshRelayPolicy, 18 tests)    │
│  ⏳ Phase F: Recording Strategy Update (docs only) ◄── NEXT                 │
│  ► Run 003: Retrain with mesh + continuous actions + cone filter             │
│                                                                              │
│  See: MESH_AND_ACTION_ARCHITECTURE_CORRECTION.md                             │
│  Run 002 is BASELINE ONLY — trained without mesh/cone/continuous actions.    │
├─────────────────────────────────────────────────────────────────────────────┤
│  PHASE 6: Real Data Pipeline - Production Model Path                        │
├─────────────────────────────────────────────────────────────────────────────┤
│  ✅ Phases 1-4: ML Architecture COMPLETE (Deep Sets, ConvoyEnv, Emulator)   │
│  ✅ Cloud Run 002: 100% success (40ep) BUT only n=1,2 - Feb 21, 2026       │
│  ✅ Emulator calibrated from convoy data (cruising noise) - Feb 24, 2026    │
│                                                                              │
│  ⏳ Phase 6.2: Recording #2 (roof mount, mesh-aware firmware, hard brake)   │
│  ► Phase 6.3: Process convoy → SUMO base scenario                           │
│  ► Phase 6.6: Train Run 003 (mesh + continuous + cone filter)               │
│  ► Phase 6.7: 200-episode eval (n=1-5, hazards ON)                         │
│  ► Phase 7:   Quantization (TFLite INT8 for ESP32)                          │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Start Here:**
- **IMMEDIATE:** [Phase 6 Real Data Pipeline](../10_PLANS_ACTIVE/PHASE_6_REAL_DATA_PIPELINE.md) ► **PRODUCTION MODEL PATH**
- **Architecture:** [Deep Sets Architecture](../00_ARCHITECTURE/DEEP_SETS_N_ELEMENT_ARCHITECTURE.md) - Solves n-element problem
- **Reference:** [Cloud Run 002 Results](../20_KNOWLEDGE_BASE/CLOUD_TRAINING_RUN_002_RESULTS.md) ► 40 episodes with per-n metrics
- **Professor Response:** [Professor Comments Response](../10_PLANS_ACTIVE/PROFESSOR_COMMENTS_RESPONSE_PLAN.md)

---

## 00_ARCHITECTURE (5 files)

System description, contracts, and high-level design documents.

| Document | Display Name |
|----------|--------------|
| [DEEP_SETS_N_ELEMENT_ARCHITECTURE.md](../00_ARCHITECTURE/DEEP_SETS_N_ELEMENT_ARCHITECTURE.md) | **Deep Sets: n-Element Problem** ► CURRENT (Jan 2026) |
| [ARCHITECTURE_V2_MINIMAL_RL.md](../00_ARCHITECTURE/ARCHITECTURE_V2_MINIMAL_RL.md) | Architecture V2: Minimal RL *(obs space SUPERSEDED)* |
| [CLAUDE.md](../00_ARCHITECTURE/CLAUDE.md) | RoadSense V2V Project Context |
| [ESPNOW_EMULATOR_DESIGN.md](../00_ARCHITECTURE/ESPNOW_EMULATOR_DESIGN.md) | ESP-NOW Emulator Design Document |
| [RoadSense2-SRS-2025.md](../00_ARCHITECTURE/RoadSense2-SRS-2025.md) | RoadSense V2V System - Software Requirements Specification |

---

## 10_PLANS_ACTIVE (26 files)

Current ongoing plans, TODOs, and work-in-progress execution plans.

| Document | Display Name |
|----------|--------------|
| [MESH_AND_ACTION_ARCHITECTURE_CORRECTION.md](../10_PLANS_ACTIVE/MESH_AND_ACTION_ARCHITECTURE_CORRECTION.md) | **Mesh + Continuous Action Correction** ► PHASES A-E COMPLETE, Phase F pending |
| [MESH_AND_ACTION_ARCHITECTURE_CORRECTION_PROGRESS.md](../10_PLANS_ACTIVE/MESH_AND_ACTION_ARCHITECTURE_CORRECTION_PROGRESS.md) | Mesh + Action Correction Progress Tracker |
| [PHASE_6_REAL_DATA_PIPELINE.md](../10_PLANS_ACTIVE/PHASE_6_REAL_DATA_PIPELINE.md) | **Phase 6: Real Data Pipeline** ► IMMEDIATE PRIORITY (Jan 24) |
| [AGENT_HANDOFF_FIELD_READINESS_PLAN.md](../10_PLANS_ACTIVE/AGENT_HANDOFF_FIELD_READINESS_PLAN.md) | **Field Readiness Handoff Plan** ► ACTIVE (Feb 12 implementation pass) |
| [N_ELEMENT_IMPLEMENTATION_PLAN.md](../10_PLANS_ACTIVE/N_ELEMENT_IMPLEMENTATION_PLAN.md) | n-Element Deep Sets Implementation (Phases 1-5) |
| [SUMO_BASE_SCENARIO_PLAN.md](../10_PLANS_ACTIVE/SUMO_BASE_SCENARIO_PLAN.md) | SUMO Base Scenarios (Dataset v2) |
| [DATALOGGER_ACCEL_UPDATE.md](../10_PLANS_ACTIVE/DATALOGGER_ACCEL_UPDATE.md) | DataLogger Accelerometer Update for Network Characterization |
| [ESPNOW_CHARACTERIZATION_IMPLEMENTATION.md](../10_PLANS_ACTIVE/ESPNOW_CHARACTERIZATION_IMPLEMENTATION.md) | ESP-NOW Network Characterization Implementation Guide |
| [ESPNOW_LONG_RANGE_MODE_MIGRATION.md](../10_PLANS_ACTIVE/ESPNOW_LONG_RANGE_MODE_MIGRATION.md) | ESP-NOW LR Mode Migration Plan (extends range) |
| [hw_firmware_migration_plan.md](../10_PLANS_ACTIVE/hw_firmware_migration_plan.md) | Hardware Firmware Migration Plan |
| [QUANTIZATION_DUAL_TRACK_PLAN.md](../10_PLANS_ACTIVE/QUANTIZATION_DUAL_TRACK_PLAN.md) | Quantization Dual-Track Plan (PTQ + QAT) |
| [CLOUD_TRAINING_PIPELINE_PROPOSAL.md](../10_PLANS_ACTIVE/CLOUD_TRAINING_PIPELINE_PROPOSAL.md) | Cloud Training Pipeline Proposal (Draft) |
| [CLOUD_TRAINING_PIPELINE_VERDICT.md](../10_PLANS_ACTIVE/CLOUD_TRAINING_PIPELINE_VERDICT.md) | Cloud Training Pipeline Verdict (Approved with Conditions) |
| [CLOUD_TRAINING_AWS_EXECUTION_PLAN.md](../10_PLANS_ACTIVE/CLOUD_TRAINING_AWS_EXECUTION_PLAN.md) | Cloud Training AWS Execution Plan ✅ RUN 001 COMPLETE, Run 002 READY |
| [EC2_AMI_CREATION_PLAN.md](../10_PLANS_ACTIVE/EC2_AMI_CREATION_PLAN.md) | **EC2 AMI Creation Plan** ✅ COMPLETED (ami-03a3037588b0f34f2) |
| [RUN_002_EVAL_GAP_REVIEW.md](../10_PLANS_ACTIVE/RUN_002_EVAL_GAP_REVIEW.md) | Run 002 Eval Gap Review (n=3,4,5 still needed) |
| [THREE_VEHICLE_CHARACTERIZATION_PLAN.md](../10_PLANS_ACTIVE/THREE_VEHICLE_CHARACTERIZATION_PLAN.md) | Three Vehicle Characterization Plan |

### Agent Prompts
| Document | Display Name |
|----------|--------------|
| [CONVOY_DATA_INTERPRETER.md](../AGENTS/CONVOY_DATA_INTERPRETER.md) | **Convoy Data Interpreter Agent** (Feb 24, 2026) |
| [CONVOY_FIELD_VALIDATOR.md](../AGENTS/CONVOY_FIELD_VALIDATOR.md) | **Convoy Field Validator Agent** ► NEW (Feb 26, 2026) |
| [BOOKKEEPER_PROMPT.md](../AGENT_PROMPTS/BOOKKEEPER_PROMPT.md) | Documentation Bookkeeper Agent |

### RTT Implementation (Subdirectory)
| Document | Display Name |
|----------|--------------|
| [RTT_FIRMWARE_IMPLEMENTATION_PROMPT.md](../10_PLANS_ACTIVE/RTT_Implementation/RTT_FIRMWARE_IMPLEMENTATION_PROMPT.md) | ESP-NOW RTT Characterization Firmware Implementation Guide |
| [RTT_PROGRESS_TRACKER.md](../10_PLANS_ACTIVE/RTT_Implementation/RTT_PROGRESS_TRACKER.md) | RTT Firmware TDD Progress Tracker ✅ CHARACTERIZATION COMPLETE |
| [RTT_TEST_SUITE_SUMMARY.md](../10_PLANS_ACTIVE/RTT_Implementation/RTT_TEST_SUITE_SUMMARY.md) | RTT Firmware TDD Test Suite Summary |

### N-Element Implementation (Subdirectory) - CURRENT FOCUS
| Document | Display Name |
|----------|--------------|
| [N_ELEMENT_PROGRESS_TRACKER.md](../10_PLANS_ACTIVE/N_Element_Implementation/N_ELEMENT_PROGRESS_TRACKER.md) | **N-Element Progress Tracker** ► Phase 4 COMPLETE |
| [PHASE_1_SPEC.md](../10_PLANS_ACTIVE/N_Element_Implementation/PHASE_1_SPEC.md) | Phase 1: ConvoyEnv Dict Observation Space |
| [PHASE_1_BUILDER_REPORT.md](../10_PLANS_ACTIVE/N_Element_Implementation/PHASE_1_BUILDER_REPORT.md) | Phase 1 Builder Report |
| [PHASE_1_REVIEW.md](../10_PLANS_ACTIVE/N_Element_Implementation/PHASE_1_REVIEW.md) | Phase 1 Review - **APPROVED** |
| [PHASE_2_SPEC.md](../10_PLANS_ACTIVE/N_Element_Implementation/PHASE_2_SPEC.md) | Phase 2: Deep Sets Policy Network |
| [PHASE_2_BUILDER_REPORT.md](../10_PLANS_ACTIVE/N_Element_Implementation/PHASE_2_BUILDER_REPORT.md) | Phase 2 Builder Report |
| [PHASE_2_REVIEW.md](../10_PLANS_ACTIVE/N_Element_Implementation/PHASE_2_REVIEW.md) | Phase 2 Review - **APPROVED** |
| [PHASE_3_4_SPEC.md](../10_PLANS_ACTIVE/N_Element_Implementation/PHASE_3_4_SPEC.md) | Phase 3 & 4: Causality Fix & Training Pipeline |
| [PHASE_3_4_BUILDER_REPORT.md](../10_PLANS_ACTIVE/N_Element_Implementation/PHASE_3_4_BUILDER_REPORT.md) | Phase 3 & 4 Builder Report |
| [PHASE_3_4_REVIEW.md](../10_PLANS_ACTIVE/N_Element_Implementation/PHASE_3_4_REVIEW.md) | Phase 3 & 4 Review - **APPROVED** |

### ConvoyEnv Implementation (Subdirectory)
| Document | Display Name |
|----------|--------------|
| [CONVOY_ENV_IMPLEMENTATION_PLAN.md](../10_PLANS_ACTIVE/ConvoyEnv_Implementation/CONVOY_ENV_IMPLEMENTATION_PLAN.md) | ConvoyEnv Gymnasium Implementation Plan |
| [CONVOY_ENV_PROGRESS_TRACKER.md](../10_PLANS_ACTIVE/ConvoyEnv_Implementation/CONVOY_ENV_PROGRESS_TRACKER.md) | ConvoyEnv Progress Tracker (Phase 7 Complete) |

### Cloud Training Automation (Subdirectory)
| Document | Display Name |
|----------|--------------|
| [IMPLEMENTATION_PLAN.md](../10_PLANS_ACTIVE/CLOUD_TRAINING_AUTOMATION/IMPLEMENTATION_PLAN.md) | Cloud Training Automation Implementation Plan |
| [PROGRESS_TRACKER.md](../10_PLANS_ACTIVE/CLOUD_TRAINING_AUTOMATION/PROGRESS_TRACKER.md) | Cloud Training Automation Progress Tracker |

---

## 20_KNOWLEDGE_BASE (15 files)

How-to guides, troubleshooting, workflows, and reusable reference notes.

NOTE: SUMO FILES HAVE BEEN MOVED TO A SEPARATE FOLDER.

| Document | Display Name |
|----------|--------------|
| [CLOUD_TRAINING_RUN_002_RESULTS.md](../20_KNOWLEDGE_BASE/CLOUD_TRAINING_RUN_002_RESULTS.md) | **Cloud Training Run 002 Results** ► 40-episode eval + per-n evidence (Feb 21, 2026) |
| [CLOUD_TRAINING_RUN_001_RESULTS.md](../20_KNOWLEDGE_BASE/CLOUD_TRAINING_RUN_001_RESULTS.md) | **Cloud Training Run 001 Results** ► 80% SUCCESS (Jan 16, 2026) |
| [AI_AGENT_CHECKLIST.md](../20_KNOWLEDGE_BASE/AI_AGENT_CHECKLIST.md) | AI Agent Pre-Implementation Checklist |
| [DATALOGGER_MODIFICATION_PROMPT.md](../20_KNOWLEDGE_BASE/DATALOGGER_MODIFICATION_PROMPT.md) | DataLogger Modification Prompt |
| [DATALOGGER_VALIDATION_TEST.md](../20_KNOWLEDGE_BASE/DATALOGGER_VALIDATION_TEST.md) | DataLogger Validation Test Guide (At Home) |
| [HARDWARE_PROGRESS.md](../20_KNOWLEDGE_BASE/HARDWARE_PROGRESS.md) | Hardware Development Progress |
| [LINUX_PIP_LARGE_PACKAGES.md](../20_KNOWLEDGE_BASE/LINUX_PIP_LARGE_PACKAGES.md) | **Installing Large Python Packages (PyTorch) on Linux** |
| [ML_AUGMENTATION_PIPELINE.md](../20_KNOWLEDGE_BASE/ML_AUGMENTATION_PIPELINE.md) | ML Data Augmentation Pipeline (SUMO-Only) |
| [ML_DOCKER_ENVIRONMENT.md](../20_KNOWLEDGE_BASE/ML_DOCKER_ENVIRONMENT.md) | **ML Docker Environment Guide** (NEW - cross-platform setup) |
| [NETWORK_CHARACTERIZATION_GUIDE.md](../20_KNOWLEDGE_BASE/NETWORK_CHARACTERIZATION_GUIDE.md) | ESP-NOW Network Characterization Guide |
| [PLATFORMIO_TESTING_README.md](../20_KNOWLEDGE_BASE/PLATFORMIO_TESTING_README.md) | PlatformIO Unit Testing with Unity |
| [SUMO_EX3_README.md](../20_KNOWLEDGE_BASE/SUMO_EX3_README.md) | SUMO Exercise 3 README |
| [SUMO_EX4_README.md](../20_KNOWLEDGE_BASE/SUMO_EX4_README.md) | SUMO Exercise 4 README |
| [SUMO_EXERCISE_FINDINGS.md](../20_KNOWLEDGE_BASE/SUMO_EXERCISE_FINDINGS.md) | SUMO Exercise Findings |
| [SUMO_RESEARCH_AND_SETUP_GUIDE.md](../20_KNOWLEDGE_BASE/SUMO_RESEARCH_AND_SETUP_GUIDE.md) | SUMO Research and Setup Guide |

SUMO FILES HAVE BEEN MOVED TO A SEPARATE FOLDER.

---

## 90_ARCHIVE (22 files)

Implemented/completed plans, approved code reviews, and historical documents.

### Emulator Implementation Plans (Complete)

| Document | Display Name |
|----------|--------------|
| [EMULATOR_PHASE2_IMPLEMENTATION_PLAN.md](../90_ARCHIVE/Emulator/EMULATOR_PHASE2_IMPLEMENTATION_PLAN.md) | Phase 2: Parameter Loading |
| [EMULATOR_PHASE3_IMPLEMENTATION_PLAN.md](../90_ARCHIVE/Emulator/EMULATOR_PHASE3_IMPLEMENTATION_PLAN.md) | Phase 3: Latency Modeling |
| [EMULATOR_PHASE4_IMPLEMENTATION_PLAN.md](../90_ARCHIVE/Emulator/EMULATOR_PHASE4_IMPLEMENTATION_PLAN.md) | Phase 4: Packet Loss Modeling |
| [EMULATOR_PHASE5_IMPLEMENTATION_PLAN.md](../90_ARCHIVE/Emulator/EMULATOR_PHASE5_IMPLEMENTATION_PLAN.md) | Phase 5: Sensor Noise |
| [EMULATOR_PHASE6_IMPLEMENTATION_PLAN.md](../90_ARCHIVE/Emulator/EMULATOR_PHASE6_IMPLEMENTATION_PLAN.md) | Phase 6: Core Transmission & Observation Logic |
| [PHASE8_IMPLEMENTATION_PLAN.md](../90_ARCHIVE/Emulator/PHASE8_IMPLEMENTATION_PLAN.md) | Phase 8: Domain Randomization & Lifecycle Management |
| [PHASE9_IMPLEMENTATION_PLAN.md](../90_ARCHIVE/Emulator/PHASE9_IMPLEMENTATION_PLAN.md) | Phase 9: Integration & Sim2Real Validation |
| [PHASE10_IMPLEMENTATION_PLAN.md](../90_ARCHIVE/Emulator/PHASE10_IMPLEMENTATION_PLAN.md) | Phase 10: Documentation & Final Polish |

### Approved Code Reviews

| Document | Display Name |
|----------|--------------|
| [PHASE3_CODE_REVIEW.md](../90_ARCHIVE/Emulator/PHASE3_CODE_REVIEW.md) | Phase 3 Code Review - APPROVED |
| [PHASE4_CODE_REVIEW.md](../90_ARCHIVE/Emulator/PHASE4_CODE_REVIEW.md) | Phase 4 Code Review - APPROVED (Production Ready) |
| [PHASE5_CODE_REVIEW.md](../90_ARCHIVE/Emulator/PHASE5_CODE_REVIEW.md) | Phase 5 Code Review - APPROVED |
| [PHASE5_CODE_REVIEW_FIXES_SUMMARY.md](../90_ARCHIVE/Emulator/PHASE5_CODE_REVIEW_FIXES_SUMMARY.md) | Phase 5 Fixes Summary |
| [PHASE6_CODE_REVIEW.md](../90_ARCHIVE/Emulator/PHASE6_CODE_REVIEW.md) | Phase 6 Code Review - APPROVED |
| [PHASE6_CODE_REVIEW_FIXES_SUMMARY.md](../90_ARCHIVE/Emulator/PHASE6_CODE_REVIEW_FIXES_SUMMARY.md) | Phase 6 Fixes Summary |
| [PHASE8_CODE_REVIEW.md](../90_ARCHIVE/Emulator/PHASE8_CODE_REVIEW.md) | Phase 8 Code Review - CONDITIONALLY APPROVED |
| [PHASE8_FIXES_SUMMARY.md](../90_ARCHIVE/Emulator/PHASE8_FIXES_SUMMARY.md) | Phase 8 Fixes Summary |
| [PHASE9_CODE_REVIEW.md](../90_ARCHIVE/Emulator/PHASE9_CODE_REVIEW.md) | Phase 9 Code Review - APPROVED |

### ConvoyEnv Implementation (Complete Phases)

| Document | Display Name |
|----------|--------------|
| [PHASE_7_SPEC.md](../90_ARCHIVE/ConvoyEnv/PHASE_7_SPEC.md) | Phase 7: Integration & Gymnasium Registration |

### Progress Trackers (Complete)

| Document | Display Name |
|----------|--------------|
| [ESPNOW_EMULATOR_PROGRESS.md](../90_ARCHIVE/Emulator/ESPNOW_EMULATOR_PROGRESS.md) | ESP-NOW Emulator Progress - ALL PHASES COMPLETE |
| [TDD_PROGRESS_TRACKER.md](../90_ARCHIVE/TDD_Implementation/TDD_PROGRESS_TRACKER.md) | RTT Firmware TDD Progress - GREEN PHASE COMPLETE |
| [TDD_TEST_SUITE_SUMMARY.md](../90_ARCHIVE/TDD_Implementation/TDD_TEST_SUITE_SUMMARY.md) | RTT Firmware TDD Test Suite Summary |

### Archived Duplicates

| Document | Display Name |
|----------|--------------|
| [ESPNOW_EMULATOR_DESIGN_TDD_copy.md](../90_ARCHIVE/Duplicates/ESPNOW_EMULATOR_DESIGN_TDD_copy.md) | ESP-NOW Emulator Design (TDD Copy) |
| [ESPNOW_CHARACTERIZATION_IMPLEMENTATION_TDD_copy.md](../90_ARCHIVE/Duplicates/ESPNOW_CHARACTERIZATION_IMPLEMENTATION_TDD_copy.md) | ESP-NOW Characterization Implementation (TDD Copy) |

---

## Quick Links by Topic

### CURRENT PRIORITIES (In Order)
1. **Phase F: Recording Strategy docs** - document mesh-aware recording protocol
2. **Convoy Recording #2** - roof-mount boards, mesh-relay firmware, hard braking
3. **Run 003: Retrain** - mesh + continuous actions + cone filter (first correct training)
4. **200-episode eval (n=1-5, hazards ON)** - prove Deep Sets handles variable n
5. **Quantization** - TFLite INT8 for ESP32 deployment

### RECENTLY COMPLETED
- ✅ **Architecture Correction Phases A-E** - Feb 26-27, 2026. Cone filter, continuous action space, mesh relay in emulator/ConvoyEnv/firmware. [Progress](../10_PLANS_ACTIVE/MESH_AND_ACTION_ARCHITECTURE_CORRECTION_PROGRESS.md)
- ✅ **Convoy Recording #1 + Emulator Calibration** - Feb 21-24, 2026. Emulator calibrated from healthy links + cruising sensor noise.
- ✅ **Run 002 Evaluation** - Feb 21, 2026. 100% success (40ep) BUT baseline only (no mesh/cone/continuous).
- ✅ **Phase 3 & 4: Causality Fix + Training Pipeline** - [Review](../10_PLANS_ACTIVE/N_Element_Implementation/PHASE_3_4_REVIEW.md) APPROVED (Jan 15, 2026)

### ESP-NOW Emulator
- [Design](../00_ARCHITECTURE/ESPNOW_EMULATOR_DESIGN.md)
- [Progress](../90_ARCHIVE/Emulator/ESPNOW_EMULATOR_PROGRESS.md) ✅ COMPLETE (84/84 tests)
- [Active Params](../../roadsense-v2v/ml/espnow_emulator/emulator_params_measured.json) (convoy-calibrated, Feb 24)
- [Convoy Calibration Source](../../roadsense-v2v/ml/espnow_emulator/emulator_params_convoy.json)
- [RTT Backup](../../roadsense-v2v/ml/espnow_emulator/emulator_params_measured_rtt_backup.json) (pre-convoy)
- [Original 5m Test](../../roadsense-v2v/ml/espnow_emulator/emulator_params_5m.json)

### Network Characterization
- [Implementation Plan](../10_PLANS_ACTIVE/ESPNOW_CHARACTERIZATION_IMPLEMENTATION.md)
- [Field Guide](../20_KNOWLEDGE_BASE/NETWORK_CHARACTERIZATION_GUIDE.md)
- [RTT Progress](../10_PLANS_ACTIVE/RTT_Implementation/RTT_PROGRESS_TRACKER.md) ✅ COMPLETE
- [LR Mode Migration](../10_PLANS_ACTIVE/ESPNOW_LONG_RANGE_MODE_MIGRATION.md) (PLANNED)

### SUMO Simulation & RL Training
- [Setup Guide](../20_KNOWLEDGE_BASE/SUMO_RESEARCH_AND_SETUP_GUIDE.md)
- [Docker Environment](../20_KNOWLEDGE_BASE/ML_DOCKER_ENVIRONMENT.md) ► NEW (cross-platform)
- [Exercise Findings](../20_KNOWLEDGE_BASE/SUMO_EXERCISE_FINDINGS.md)
- [ML Pipeline](../20_KNOWLEDGE_BASE/ML_AUGMENTATION_PIPELINE.md)
- [ConvoyEnv Implementation Plan](../10_PLANS_ACTIVE/ConvoyEnv_Implementation/CONVOY_ENV_IMPLEMENTATION_PLAN.md) ► CURRENT
- [ConvoyEnv Progress](../10_PLANS_ACTIVE/ConvoyEnv_Implementation/CONVOY_ENV_PROGRESS_TRACKER.md) ► Phase 7 Complete

### Hardware Development
- [Progress Tracking](../20_KNOWLEDGE_BASE/HARDWARE_PROGRESS.md)
- [Migration Plan](../10_PLANS_ACTIVE/hw_firmware_migration_plan.md)
- [Testing Guide](../20_KNOWLEDGE_BASE/PLATFORMIO_TESTING_README.md)

---

**Document Version:** 1.1
