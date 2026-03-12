# RoadSense V2V Documentation Index

**Last Updated:** March 12, 2026
**Total Documents:** 71+

---

## CURRENT PROJECT STATUS

```
 Run 014 IN PROGRESS on EC2 (launched March 12, 2026)
 Latch fix + formation-fixed base_real + HAZARD_PROBABILITY=1.0
 Best model: Run 011 (100% V2V reaction, avg_reward=-86.62)
 Runs 012-013 failed (gradual hazard regressions, now fixed via latch)
 S3: s3://saferide-training-results/cloud_prod_014/

 See: PROJECT_STATUS_OVERVIEW.md for full history (Runs 003-014)
```

**Start Here:**
- **Status:** [PROJECT_STATUS_OVERVIEW.md](../PROJECT_STATUS_OVERVIEW.md) - Single source of truth
- **Run 014 Prep:** [RUN_014_PREP_CHECKLIST.md](../10_PLANS_ACTIVE/RUN_014_PREP_CHECKLIST.md) - Latch fix + launch checklist
- **Pipeline:** [Phase 6 Real Data Pipeline](../10_PLANS_ACTIVE/PHASE_6_REAL_DATA_PIPELINE.md) - Production model path
- **Architecture:** [Deep Sets Architecture](../00_ARCHITECTURE/DEEP_SETS_N_ELEMENT_ARCHITECTURE.md) - Solves n-element problem

---

## 00_ARCHITECTURE (5 files)

System description, contracts, and high-level design documents.

| Document | Description |
|----------|-------------|
| [DEEP_SETS_N_ELEMENT_ARCHITECTURE.md](../00_ARCHITECTURE/DEEP_SETS_N_ELEMENT_ARCHITECTURE.md) | Deep Sets: n-Element Problem (CURRENT) |
| [ARCHITECTURE_V2_MINIMAL_RL.md](../00_ARCHITECTURE/ARCHITECTURE_V2_MINIMAL_RL.md) | Architecture V2: Minimal RL (obs space SUPERSEDED) |
| [CLAUDE.md](../00_ARCHITECTURE/CLAUDE.md) | RoadSense V2V Project Context |
| [ESPNOW_EMULATOR_DESIGN.md](../00_ARCHITECTURE/ESPNOW_EMULATOR_DESIGN.md) | ESP-NOW Emulator Design Document |
| [RoadSense-SRS-Final.md](../00_ARCHITECTURE/RoadSense-SRS-Final.md) | Software Requirements Specification |

---

## 10_PLANS_ACTIVE (5 files)

Active plans with open work items.

| Document | Status |
|----------|--------|
| [PHASE_6_REAL_DATA_PIPELINE.md](../10_PLANS_ACTIVE/PHASE_6_REAL_DATA_PIPELINE.md) | Production path. H5 sim-to-real validation pending post-training |
| [CLOUD_TRAINING_AWS_EXECUTION_PLAN.md](../10_PLANS_ACTIVE/CLOUD_TRAINING_AWS_EXECUTION_PLAN.md) | Active EC2 operations. Run 009 in progress |
| [RUN_007_STRATEGIC_ANALYSIS.md](../10_PLANS_ACTIVE/RUN_007_STRATEGIC_ANALYSIS.md) | Reward ramp + stability strategy (applies to Run 007-009) |
| [RUN_004_HAZARD_TARGETING_AND_SOURCE_REACTION_EVAL_PLAN.md](../10_PLANS_ACTIVE/RUN_004_HAZARD_TARGETING_AND_SOURCE_REACTION_EVAL_PLAN.md) | H0-H4 complete. H5 sim-to-real validation pending |
| [QUANTIZATION_DUAL_TRACK_PLAN.md](../10_PLANS_ACTIVE/QUANTIZATION_DUAL_TRACK_PLAN.md) | FUTURE: TFLite INT8 for ESP32 (after successful training) |

### Agent Prompts
| Document | Description |
|----------|-------------|
| [CONVOY_DATA_INTERPRETER.md](../AGENTS/CONVOY_DATA_INTERPRETER.md) | Convoy Data Interpreter Agent |
| [CONVOY_FIELD_VALIDATOR.md](../AGENTS/CONVOY_FIELD_VALIDATOR.md) | Convoy Field Validator Agent (--forward-axis y) |
| [BOOKKEEPER_PROMPT.md](../AGENT_PROMPTS/BOOKKEEPER_PROMPT.md) | Documentation Bookkeeper Agent |

---

## 20_KNOWLEDGE_BASE (16 files)

How-to guides, troubleshooting, and reusable reference notes.

| Document | Description |
|----------|-------------|
| [CLOUD_TRAINING_RUN_002_RESULTS.md](../20_KNOWLEDGE_BASE/CLOUD_TRAINING_RUN_002_RESULTS.md) | Run 002 Results: 40-episode eval + per-n metrics |
| [CLOUD_TRAINING_RUN_001_RESULTS.md](../20_KNOWLEDGE_BASE/CLOUD_TRAINING_RUN_001_RESULTS.md) | Run 001 Results: 80% success |
| [AI_AGENT_CHECKLIST.md](../20_KNOWLEDGE_BASE/AI_AGENT_CHECKLIST.md) | AI Agent Pre-Implementation Checklist |
| [DATALOGGER_MODIFICATION_PROMPT.md](../20_KNOWLEDGE_BASE/DATALOGGER_MODIFICATION_PROMPT.md) | DataLogger Modification Prompt |
| [DATALOGGER_VALIDATION_TEST.md](../20_KNOWLEDGE_BASE/DATALOGGER_VALIDATION_TEST.md) | DataLogger Validation Test Guide |
| [DATA_RECORDING_STRATEGY.md](../20_KNOWLEDGE_BASE/DATA_RECORDING_STRATEGY.md) | Recording Strategy + Axis Mapping (--forward-axis y) |
| [HARDWARE_PROGRESS.md](../20_KNOWLEDGE_BASE/HARDWARE_PROGRESS.md) | Hardware Development Progress |
| [LINUX_PIP_LARGE_PACKAGES.md](../20_KNOWLEDGE_BASE/LINUX_PIP_LARGE_PACKAGES.md) | Installing Large Python Packages on Linux |
| [ML_AUGMENTATION_PIPELINE.md](../20_KNOWLEDGE_BASE/ML_AUGMENTATION_PIPELINE.md) | ML Data Augmentation Pipeline (SUMO-Only) |
| [ML_DOCKER_ENVIRONMENT.md](../20_KNOWLEDGE_BASE/ML_DOCKER_ENVIRONMENT.md) | ML Docker Environment Guide |
| [TEAM_CLOUD_TRAINING_ONBOARDING.md](../20_KNOWLEDGE_BASE/TEAM_CLOUD_TRAINING_ONBOARDING.md) | Shared AMI + branch-isolated teammate onboarding runbook |
| [NETWORK_CHARACTERIZATION_GUIDE.md](../20_KNOWLEDGE_BASE/NETWORK_CHARACTERIZATION_GUIDE.md) | ESP-NOW Network Characterization Guide |
| [PLATFORMIO_TESTING_README.md](../20_KNOWLEDGE_BASE/PLATFORMIO_TESTING_README.md) | PlatformIO Unit Testing with Unity |

### SUMO Guides (subfolder)
| Document | Description |
|----------|-------------|
| [SUMO_RESEARCH_AND_SETUP_GUIDE.md](../20_KNOWLEDGE_BASE/SUMO/SUMO_RESEARCH_AND_SETUP_GUIDE.md) | SUMO Research and Setup Guide |
| [SUMO_EXERCISE_FINDINGS.md](../20_KNOWLEDGE_BASE/SUMO/SUMO_EXERCISE_FINDINGS.md) | SUMO Exercise Findings |
| [SUMO_EX3_README.md](../20_KNOWLEDGE_BASE/SUMO/SUMO_EX3_README.md) | SUMO Exercise 3 |
| [SUMO_EX4_README.md](../20_KNOWLEDGE_BASE/SUMO/SUMO_EX4_README.md) | SUMO Exercise 4 |

---

## 90_ARCHIVE (60+ files)

Completed plans, approved reviews, and historical documents.

### Architecture Correction (Complete Feb 26-28, 2026)
| Document | Description |
|----------|-------------|
| [MESH_AND_ACTION_ARCHITECTURE_CORRECTION.md](../90_ARCHIVE/Architecture_Correction/MESH_AND_ACTION_ARCHITECTURE_CORRECTION.md) | Master plan: Phases A-G |
| [MESH_AND_ACTION_ARCHITECTURE_CORRECTION_PROGRESS.md](../90_ARCHIVE/Architecture_Correction/MESH_AND_ACTION_ARCHITECTURE_CORRECTION_PROGRESS.md) | Progress tracker |

### Training Runs (Runs 002-007, all concluded)
| Document | Description |
|----------|-------------|
| [RUN_002_EVAL_GAP_REVIEW.md](../90_ARCHIVE/Training_Runs/RUN_002_EVAL_GAP_REVIEW.md) | Run 002 eval gap (n=3,4,5 needed) |
| [RUN_003_FINDINGS_FOR_ARCHITECT_REVIEW.md](../90_ARCHIVE/Training_Runs/RUN_003_FINDINGS_FOR_ARCHITECT_REVIEW.md) | Run 003 root-cause analysis |
| [RUN_003_SESSION_FIXES_FOR_ARCHITECT_REVIEW.md](../90_ARCHIVE/Training_Runs/RUN_003_SESSION_FIXES_FOR_ARCHITECT_REVIEW.md) | Run 003 follow-up fixes |
| [RUN_005_006_POST_MORTEM.md](../90_ARCHIVE/Training_Runs/RUN_005_006_POST_MORTEM.md) | Run 005/006 post-mortem |
| [RUN_006_FIX_PLAN.md](../90_ARCHIVE/Training_Runs/RUN_006_FIX_PLAN.md) | Run 006: 7 root causes fixed (canonical) |
| [RUN_007_PRE_IMPLEMENTATION_ANALYSIS.md](../90_ARCHIVE/Training_Runs/RUN_007_PRE_IMPLEMENTATION_ANALYSIS.md) | Run 007 pre-implementation analysis |

### N-Element Implementation (Complete Jan 2026)
| Document | Description |
|----------|-------------|
| [N_ELEMENT_IMPLEMENTATION_PLAN.md](../90_ARCHIVE/N_Element_Implementation/N_ELEMENT_IMPLEMENTATION_PLAN.md) | Master plan |
| [N_ELEMENT_PROGRESS_TRACKER.md](../90_ARCHIVE/N_Element_Implementation/N_ELEMENT_PROGRESS_TRACKER.md) | Progress tracker |
| [PHASE_1_SPEC.md](../90_ARCHIVE/N_Element_Implementation/PHASE_1_SPEC.md) | Phase 1: Dict Obs Space |
| [PHASE_2_SPEC.md](../90_ARCHIVE/N_Element_Implementation/PHASE_2_SPEC.md) | Phase 2: Deep Sets Policy |
| [PHASE_3_4_SPEC.md](../90_ARCHIVE/N_Element_Implementation/PHASE_3_4_SPEC.md) | Phase 3-4: Causality + Training |
| [PHASE_5_1_LR_MODE_SPEC.md](../90_ARCHIVE/N_Element_Implementation/PHASE_5_1_LR_MODE_SPEC.md) | Phase 5.1: LR Mode (future) |
| + 6 builder reports and reviews | APPROVED |

### ConvoyEnv Implementation (Complete Jan 2026)
| Document | Description |
|----------|-------------|
| [CONVOY_ENV_IMPLEMENTATION_PLAN.md](../90_ARCHIVE/ConvoyEnv/CONVOY_ENV_IMPLEMENTATION_PLAN.md) | Master plan |
| [CONVOY_ENV_PROGRESS_TRACKER.md](../90_ARCHIVE/ConvoyEnv/CONVOY_ENV_PROGRESS_TRACKER.md) | Phase 7 complete |
| [PHASE_5_SPEC.md](../90_ARCHIVE/ConvoyEnv/PHASE_5_SPEC.md) | Phase 5 spec |
| [PHASE_6_SPEC.md](../90_ARCHIVE/ConvoyEnv/PHASE_6_SPEC.md) | Phase 6 spec |
| [PHASE_7_SPEC.md](../90_ARCHIVE/ConvoyEnv/PHASE_7_SPEC.md) | Phase 7 spec |

### Cloud Training Automation (Complete Jan 2026)
| Document | Description |
|----------|-------------|
| [CLOUD_TRAINING_PIPELINE_PROPOSAL.md](../90_ARCHIVE/Cloud_Training/CLOUD_TRAINING_PIPELINE_PROPOSAL.md) | Original proposal |
| [CLOUD_TRAINING_PIPELINE_VERDICT.md](../90_ARCHIVE/Cloud_Training/CLOUD_TRAINING_PIPELINE_VERDICT.md) | Approved with conditions |
| [IMPLEMENTATION_PLAN.md](../90_ARCHIVE/Cloud_Training/IMPLEMENTATION_PLAN.md) | Implementation plan |
| [PROGRESS_TRACKER.md](../90_ARCHIVE/Cloud_Training/PROGRESS_TRACKER.md) | All phases complete |
| + 5 phase specs and reviews | Phases 1-3 APPROVED |

### RTT Implementation (Complete Jan 2026)
| Document | Description |
|----------|-------------|
| [RTT_FIRMWARE_IMPLEMENTATION_PROMPT.md](../90_ARCHIVE/RTT_Implementation/RTT_FIRMWARE_IMPLEMENTATION_PROMPT.md) | Firmware implementation guide |
| [RTT_PROGRESS_TRACKER.md](../90_ARCHIVE/RTT_Implementation/RTT_PROGRESS_TRACKER.md) | Characterization COMPLETE |
| [RTT_TEST_SUITE_SUMMARY.md](../90_ARCHIVE/RTT_Implementation/RTT_TEST_SUITE_SUMMARY.md) | Test suite summary |
| [RTT_ARCH_REVIEW_CRITIC.md](../90_ARCHIVE/RTT_Implementation/RTT_ARCH_REVIEW_CRITIC.md) | Architecture review |

### Hardware & Firmware (Complete)
| Document | Description |
|----------|-------------|
| [hw_firmware_migration_plan.md](../90_ARCHIVE/Hardware_Firmware/hw_firmware_migration_plan.md) | Firmware migration plan |
| [ESPNOW_CHARACTERIZATION_IMPLEMENTATION.md](../90_ARCHIVE/Hardware_Firmware/ESPNOW_CHARACTERIZATION_IMPLEMENTATION.md) | ESP-NOW characterization |
| [ESPNOW_LONG_RANGE_MODE_MIGRATION.md](../90_ARCHIVE/Hardware_Firmware/ESPNOW_LONG_RANGE_MODE_MIGRATION.md) | LR mode migration (future) |
| [DATALOGGER_ACCEL_UPDATE.md](../90_ARCHIVE/Hardware_Firmware/DATALOGGER_ACCEL_UPDATE.md) | Accel logging update |

### Phase 6 Prep (Complete Mar 2026)
| Document | Description |
|----------|-------------|
| [PHASE_6_PREP_WORK_PLAN.md](../90_ARCHIVE/Phase6_Prep/PHASE_6_PREP_WORK_PLAN.md) | Emulator + dataset_v3 |
| [AGENT_HANDOFF_FIELD_READINESS_PLAN.md](../90_ARCHIVE/Phase6_Prep/AGENT_HANDOFF_FIELD_READINESS_PLAN.md) | Field readiness checklist |
| [EC2_AMI_CREATION_PLAN.md](../90_ARCHIVE/Phase6_Prep/EC2_AMI_CREATION_PLAN.md) | AMI creation (ami-03a3037588b0f34f2) |
| [SUMO_BASE_SCENARIO_PLAN.md](../90_ARCHIVE/Phase6_Prep/SUMO_BASE_SCENARIO_PLAN.md) | Base scenario plan |
| [THREE_VEHICLE_CHARACTERIZATION_PLAN.md](../90_ARCHIVE/Phase6_Prep/THREE_VEHICLE_CHARACTERIZATION_PLAN.md) | 3-vehicle characterization |
| [SCENARIO_BUILDING.txt](../90_ARCHIVE/Phase6_Prep/SCENARIO_BUILDING.txt) | Scenario generation spec |

### Emulator (Complete Dec 2025 - Jan 2026)
- 8 implementation plans (Phases 2-10)
- 9 code reviews (all APPROVED)
- 1 progress tracker (84/84 tests)

### Other
| [Duplicates/](../90_ARCHIVE/Duplicates/) | Archived duplicate files |

---

## Quick Links by Topic

### CURRENT PRIORITIES
1. **Monitor Run 014** on EC2 (S3: cloud_prod_014, ~18-19h)
2. **Post-run:** V2V reaction check with `check_v2v_reaction.py`
3. **H5: Sim-to-real re-validation** — must react to peer braking at distance
4. **Quantization** - TFLite INT8 for ESP32
5. **Deploy on ESP32** + Professor PoC demo

### ESP-NOW Emulator
- [Design](../00_ARCHITECTURE/ESPNOW_EMULATOR_DESIGN.md)
- [Progress](../90_ARCHIVE/Emulator/ESPNOW_EMULATOR_PROGRESS.md) - ALL PHASES COMPLETE
- [Active Params](../../roadsense-v2v/ml/espnow_emulator/emulator_params_measured.json) (convoy-calibrated)

### Hardware
- [Progress](../20_KNOWLEDGE_BASE/HARDWARE_PROGRESS.md)
- [Testing Guide](../20_KNOWLEDGE_BASE/PLATFORMIO_TESTING_README.md)

---

**Document Version:** 2.0
