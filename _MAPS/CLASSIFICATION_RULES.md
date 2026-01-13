# Classification Rules for RoadSense V2V Documentation

**Created:** January 1, 2026
**Purpose:** Define heuristics used to classify documentation into the 4-bucket structure

---

## Bucket Definitions

### 00_ARCHITECTURE
**Purpose:** System description, contracts, high-level design, ADRs

**Keywords/Indicators:**
- "architecture", "overview", "components", "data flow"
- "protocol", "interfaces", "schema", "message format"
- "design", "ADR", "decision", "specification"
- "SRS" (Software Requirements Specification)
- System-level diagrams or descriptions
- Contract definitions between components

**Classification Criteria:**
- Describes WHAT the system IS (not how to use it)
- Defines interfaces between components
- Contains design decisions or rationale
- Provides high-level system overview

---

### 10_PLANS_ACTIVE
**Purpose:** Current ongoing plans, TODO, WIP execution plans

**Keywords/Indicators:**
- "plan", "next steps", "TODO", "roadmap"
- "milestone (upcoming)", "in progress", "work plan"
- "tasks", "implementation prompt"
- Dates in the future or open checklists
- Status: "Ready for implementation", "Planned", "In Progress"

**Classification Criteria:**
- Contains actionable items NOT yet completed
- Has future-facing language ("will implement", "next steps")
- Represents work that is planned but not done
- Contains open checklists or milestones

---

### 20_KNOWLEDGE_BASE
**Purpose:** How-to, troubleshooting, workflows, reusable notes

**Keywords/Indicators:**
- "how to", "setup", "build", "flash", "debug"
- "troubleshooting", "commands", "workflow"
- "FAQ", "tips", "common errors", "guide"
- "README", tutorial-style content
- Step-by-step instructions
- Reusable reference material

**Classification Criteria:**
- Explains HOW to do something (not what something is)
- Contains troubleshooting information
- Provides workflow documentation
- Serves as reference material for repeated tasks

---

### 90_ARCHIVE
**Purpose:** Implemented/completed plans, past experiments, old decisions (frozen)

**Keywords/Indicators:**
- "done", "implemented", "final results"
- "retrospective", "lessons learned", "old", "deprecated"
- "v1", "previous", "complete", "approved"
- Code reviews marked as "APPROVED"
- Implementation plans marked as "COMPLETE"
- Dates clearly in the past with outcomes

**Classification Criteria:**
- Represents work that IS completed
- Has historical value but is not actively maintained
- Contains completed code reviews or implementation plans
- Documents past decisions or experiments

---

## Ambiguity Resolution Rules

1. **Progress Trackers:** If marked "COMPLETE" or all items checked → 90_ARCHIVE; if ongoing → 10_PLANS_ACTIVE

2. **Implementation Plans + Code Reviews:** If the code review is "APPROVED" → both plan and review go to 90_ARCHIVE

3. **Guides that include future plans:** If primarily instructional → 20_KNOWLEDGE_BASE; if primarily planning → 10_PLANS_ACTIVE

4. **Default Bucket:** If truly ambiguous after applying rules → 20_KNOWLEDGE_BASE with note "Needs human review"

---

## Duplicate Handling

When identical files exist in multiple locations:
1. Keep the file in the most relevant bucket
2. Archive the duplicate in 90_ARCHIVE with suffix `_duplicate_archived`
3. Document in MOVE_MAP.md under "Duplicates Resolved"

---

## Files Excluded from Organization

- `PROMPT_FOR_AGENT.md` - Agent instructions only (not project documentation)

---

**Document Version:** 1.0
**Classification Date:** January 1, 2026
