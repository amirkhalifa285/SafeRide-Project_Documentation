# Rename Map - RoadSense V2V Documentation Reorganization

**Created:** January 1, 2026
**Total Renames:** 2 files

---

## Renaming Policy Applied

Per the reorganization guidelines, filenames were preserved unless:
1. The filename was clearly misleading relative to document content
2. There were collisions/duplicates that would break the structure
3. The file had a clear H1 title that could be deterministically converted

**Result:** Only 2 files required renaming - both were duplicates that needed disambiguation.

---

## Renames Applied

### 1. ESPNOW_EMULATOR_DESIGN.md (TDD Copy)

| Field | Value |
|-------|-------|
| **Old Path** | `TDD_Implementation/ESPNOW_EMULATOR_DESIGN.md` |
| **New Path** | `90_ARCHIVE/Duplicates/ESPNOW_EMULATOR_DESIGN_TDD_copy.md` |
| **Reason** | Duplicate of `Emulator/ESPNOW_EMULATOR_DESIGN.md` - collision avoidance |
| **Evidence** | Identical H1: "# ESP-NOW Emulator Design Document" in both files |

### 2. ESPNOW_CHARACTERIZATION_IMPLEMENTATION.md (TDD Copy)

| Field | Value |
|-------|-------|
| **Old Path** | `TDD_Implementation/ESPNOW_CHARACTERIZATION_IMPLEMENTATION.md` |
| **New Path** | `90_ARCHIVE/Duplicates/ESPNOW_CHARACTERIZATION_IMPLEMENTATION_TDD_copy.md` |
| **Reason** | Duplicate of root `ESPNOW_CHARACTERIZATION_IMPLEMENTATION.md` - collision avoidance |
| **Evidence** | Identical H1: "# ESP-NOW Network Characterization Implementation Guide" in both files |

---

## Files Considered But Not Renamed

The following files were evaluated for potential renaming but kept their original names:

| File | H1 Title | Decision |
|------|----------|----------|
| `RoadSense2-SRS-2025.md` | "# RoadSense V2V System - Software Requirements Specification" | Kept - name adequately reflects content |
| `CLAUDE.md` | Multiple sections, no single H1 | Kept - standard project context filename |
| `hw_firmware_migration_plan.md` | "# Hardware Firmware Migration Plan" | Kept - matches H1 already |
| `AI_AGENT_CHECKLIST.md` | "# AI Agent Pre-Implementation Checklist" | Kept - filename is clear |
| `ML_AUGMENTATION_PIPELINE.md` | "# ML Data Augmentation Pipeline" | Kept - filename is clear |

---

## Renaming Conventions Used

1. **Duplicate suffix:** `_TDD_copy` added to disambiguate files from TDD_Implementation folder
2. **No case changes:** Original casing preserved (e.g., UPPERCASE_WITH_UNDERSCORES)
3. **No slug conversion:** Did not convert H1 titles to slugs as original names were acceptable

---

**Document Version:** 1.0
**Classification Date:** January 1, 2026
