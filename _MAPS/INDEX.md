# RoadSense V2V Documentation Index

**Last Updated:** January 13, 2026
**Total Documents:** 47

---

## CURRENT PROJECT STATUS

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  PHASE: n-Element Deep Sets Architecture Implementation                     │
├─────────────────────────────────────────────────────────────────────────────┤
│  ✅ RTT Characterization: COMPLETE (valid 5m data)                          │
│  ✅ ESP-NOW Emulator: COMPLETE (84/84 tests)                                │
│  ⚠️  ARCHITECTURE CHANGE: Fixed obs space → Deep Sets (variable n peers)    │
│  ► n-Element Implementation: IN PROGRESS (see N_ELEMENT_IMPLEMENTATION_PLAN)│
│  ○ ConvoyEnv Update: BLOCKED (waiting for Deep Sets changes)                │
│  ○ LR Mode Migration: PLANNED (after pipeline validation)                   │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Start Here:**
- **CRITICAL:** [Deep Sets Architecture](../00_ARCHITECTURE/DEEP_SETS_N_ELEMENT_ARCHITECTURE.md) - Solves n-element problem
- Implementation: [n-Element Implementation Plan](../10_PLANS_ACTIVE/N_ELEMENT_IMPLEMENTATION_PLAN.md)
- Next firmware change: [ESP-NOW LR Mode Migration](../10_PLANS_ACTIVE/ESPNOW_LONG_RANGE_MODE_MIGRATION.md)

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

## 10_PLANS_ACTIVE (8 files)

Current ongoing plans, TODOs, and work-in-progress execution plans.

| Document | Display Name |
|----------|--------------|
| [N_ELEMENT_IMPLEMENTATION_PLAN.md](../10_PLANS_ACTIVE/N_ELEMENT_IMPLEMENTATION_PLAN.md) | **n-Element Deep Sets Implementation** ► CURRENT PRIORITY |
| [DATALOGGER_ACCEL_UPDATE.md](../10_PLANS_ACTIVE/DATALOGGER_ACCEL_UPDATE.md) | DataLogger Accelerometer Update for Network Characterization |
| [ESPNOW_CHARACTERIZATION_IMPLEMENTATION.md](../10_PLANS_ACTIVE/ESPNOW_CHARACTERIZATION_IMPLEMENTATION.md) | ESP-NOW Network Characterization Implementation Guide |
| [ESPNOW_LONG_RANGE_MODE_MIGRATION.md](../10_PLANS_ACTIVE/ESPNOW_LONG_RANGE_MODE_MIGRATION.md) | ESP-NOW LR Mode Migration Plan (extends range) |
| [hw_firmware_migration_plan.md](../10_PLANS_ACTIVE/hw_firmware_migration_plan.md) | Hardware Firmware Migration Plan |

### RTT Implementation (Subdirectory)
| Document | Display Name |
|----------|--------------|
| [RTT_FIRMWARE_IMPLEMENTATION_PROMPT.md](../10_PLANS_ACTIVE/RTT_Implementation/RTT_FIRMWARE_IMPLEMENTATION_PROMPT.md) | ESP-NOW RTT Characterization Firmware Implementation Guide |
| [RTT_PROGRESS_TRACKER.md](../10_PLANS_ACTIVE/RTT_Implementation/RTT_PROGRESS_TRACKER.md) | RTT Firmware TDD Progress Tracker ✅ CHARACTERIZATION COMPLETE |
| [RTT_TEST_SUITE_SUMMARY.md](../10_PLANS_ACTIVE/RTT_Implementation/RTT_TEST_SUITE_SUMMARY.md) | RTT Firmware TDD Test Suite Summary |

### ConvoyEnv Implementation (Subdirectory) - CURRENT FOCUS
| Document | Display Name |
|----------|--------------|
| [CONVOY_ENV_IMPLEMENTATION_PLAN.md](../10_PLANS_ACTIVE/ConvoyEnv_Implementation/CONVOY_ENV_IMPLEMENTATION_PLAN.md) | ConvoyEnv Gymnasium Implementation Plan |
| [CONVOY_ENV_PROGRESS_TRACKER.md](../10_PLANS_ACTIVE/ConvoyEnv_Implementation/CONVOY_ENV_PROGRESS_TRACKER.md) | **ConvoyEnv Progress Tracker** ► READY TO BEGIN (using 5m emulator params) |

---

## 20_KNOWLEDGE_BASE (12 files)

How-to guides, troubleshooting, workflows, and reusable reference notes.

NOTE: SUMO FILES HAVE BEEN MOVED TO A SEPARATE FOLDER.

| Document | Display Name |
|----------|--------------|
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
1. **n-Element Deep Sets Architecture** - [Architecture](../00_ARCHITECTURE/DEEP_SETS_N_ELEMENT_ARCHITECTURE.md) | [Implementation Plan](../10_PLANS_ACTIVE/N_ELEMENT_IMPLEMENTATION_PLAN.md) ► CRITICAL
2. **ConvoyEnv Update** - Must implement Dict observation space with variable peers
3. **ESP-NOW LR Mode** - [Migration Plan](../10_PLANS_ACTIVE/ESPNOW_LONG_RANGE_MODE_MIGRATION.md) (After Deep Sets validated)
4. **Clean Drive Data** - Collect with LR mode for production training

### ESP-NOW Emulator
- [Design](../00_ARCHITECTURE/ESPNOW_EMULATOR_DESIGN.md)
- [Progress](../90_ARCHIVE/Emulator/ESPNOW_EMULATOR_PROGRESS.md) ✅ COMPLETE (84/84 tests)
- [Current Params](../../roadsense-v2v/ml/espnow_emulator/emulator_params_5m.json) (5m stationary test)

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

**Document Version:** 1.0
