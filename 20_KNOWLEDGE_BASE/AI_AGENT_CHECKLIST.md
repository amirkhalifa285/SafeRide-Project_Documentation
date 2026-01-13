# AI Agent Pre-Implementation Checklist

**Purpose:** Prevent critical mistakes before writing any code or creating files.

---

## ⚠️ MANDATORY: Before Creating ANY Files

### Step 1: Verify Working Directory
- [ ] Run `pwd` to check current directory
- [ ] Compare against CLAUDE.md structure
- [ ] Confirm you're in `roadsense-v2v/` (Git repo) NOT `RoadSense2/` (personal workspace)

### Step 2: Cross-Reference with CLAUDE.md
- [ ] Read the directory structure section in CLAUDE.md
- [ ] Identify correct subdirectory (hardware/, ml/, simulation/, etc.)
- [ ] Verify path matches: `/home/amirkhalifa/RoadSense2/roadsense-v2v/<subdirectory>/`

### Step 3: Confirm with User if ANY Doubt
- [ ] If path doesn't match documentation, STOP and ask user
- [ ] Don't make assumptions about directory structure
- [ ] Better to ask than to create files in wrong location

### Step 4: Verify Git Repository
- [ ] Check that `.git/` directory exists in parent
- [ ] Run `git status` to confirm we're in a Git repo
- [ ] Files we create SHOULD be tracked by Git (unless explicitly in .gitignore)

---

## 🔍 Red Flags (STOP and Ask User)

- [ ] Creating files in `/home/amirkhalifa/RoadSense2/` root (NOT the Git repo)
- [ ] Creating files in `/home/amirkhalifa/RoadSense2/docs/` (personal docs, not Git)
- [ ] Path doesn't contain `roadsense-v2v/`
- [ ] `git status` returns "not a git repository"

---

## ✅ Correct Paths (Examples)

| Component | CORRECT Path | WRONG Path |
|-----------|--------------|------------|
| ML code | `/home/amirkhalifa/RoadSense2/roadsense-v2v/ml/` | `/home/amirkhalifa/RoadSense2/ml/` |
| Hardware firmware | `/home/amirkhalifa/RoadSense2/roadsense-v2v/hardware/` | `/home/amirkhalifa/RoadSense2/hardware/` |
| Simulation | `/home/amirkhalifa/RoadSense2/roadsense-v2v/simulation/` | `/home/amirkhalifa/RoadSense2/simulation/` |
| Planning docs (personal) | `/home/amirkhalifa/RoadSense2/docs/` | `/home/amirkhalifa/RoadSense2/roadsense-v2v/docs/` |

---

## 🎯 Quick Verification Script

Before starting any implementation, run:

```bash
# Verify we're in the correct location
pwd
# Should print: /home/amirkhalifa/RoadSense2/roadsense-v2v/<something>

# Verify Git repo
git rev-parse --show-toplevel
# Should print: /home/amirkhalifa/RoadSense2/roadsense-v2v

# If either fails, STOP and ask user for correct directory
```

---

## 📋 TDD Workflow (Updated with Verification)

### Phase -1: Location Verification (NEW - MANDATORY)
1. ✅ Check `pwd`
2. ✅ Verify against CLAUDE.md
3. ✅ Confirm Git repo
4. ✅ Get user confirmation if ANY doubt

### Phase 0: Project Setup
5. Create directory structure
6. Set up dependencies
...

---

## 🚨 Cost of Mistakes

**Time wasted:**
- 10-15 minutes to identify mistake
- 5-10 minutes to move files
- Potential for corrupted Git history if committed wrong location

**Risk if undetected:**
- Files not in Git repo → lost work
- Files in wrong Git repo → confusion
- Teammates can't access code
- CI/CD breaks

**Prevention:** 2 minutes of verification saves hours of debugging.

---

## 🚨 CRITICAL: TDD and Architectural Design Rules

**Added:** December 26, 2025 (ESPNOWEmulator refactor)

### Rule 1: NEVER Implement Without Tests (TDD Mandate)

**Violation Example (ESPNOWEmulator):**
- Progress tracker said "Phase 2 Step 2.1 - Writing tests"
- Reality: Full implementation already written (984 lines)
- Test coverage: 0% for Phases 2-8

**Consequence:** Bugs went undetected:
1. Causality violation (messages visible before arrival)
2. Dead code (burst loss never worked)
3. No validation of correctness

**Correct Approach:**
```
FOR EACH PHASE:
  1. Write tests FIRST (define expected behavior)
  2. Run tests (should FAIL - no implementation yet)
  3. Implement ONLY what tests require
  4. Run tests again (should PASS)
  5. Refactor if needed (tests still pass)
  6. Move to next phase
```

**Enforcement:**
- [ ] Before writing implementation, ask: "Did I write tests for this?"
- [ ] If answer is NO, STOP and write tests first
- [ ] Never batch implementation ahead of tests

---

### Rule 2: Design for Time/Causality in Simulations

**Violation Example (ESPNOWEmulator):**
```python
# BUG: Message added immediately to last_received
def transmit(self, ..., current_time_ms):
    latency_ms = self._get_latency(distance)  # e.g., 50ms
    received = ReceivedMessage(
        received_at_ms=current_time_ms + latency_ms  # Should arrive at t+50
    )
    self.last_received[vehicle_id] = received  # ❌ Visible at t=0!
```

**Consequence:** Agent learns from "future" information that won't exist on real hardware → Sim2Real failure.

**Correct Approach:**
```python
# FIXED: Use event queue (priority queue by arrival time)
def transmit(self, ..., current_time_ms):
    arrival_time = current_time_ms + latency_ms
    self.pending_messages.put((arrival_time, vehicle_id, received))  # Queue it

def get_observation(self, current_time_ms):
    # Process queue: deliver messages that have arrived
    while not self.pending_messages.empty():
        arrival_time, vehicle_id, msg = self.pending_messages.queue[0]
        if arrival_time <= current_time_ms:
            self.last_received[vehicle_id] = msg  # NOW visible
        else:
            break  # Future message
```

**Check Before Implementing Time-Sensitive Logic:**
- [ ] Does this system model delays/latency?
- [ ] Can future information leak to the present?
- [ ] Would this work identically on real hardware?
- [ ] Do I need an event queue or delay buffer?

---

### Rule 3: Validate State Tracking Logic

**Violation Example (Burst Loss):**
```python
def __init__(self):
    self.message_buffer = {}  # Initialized but...

def _check_burst_loss(self, vehicle_id):
    if vehicle_id in self.message_buffer:  # NEVER True!
        return True
    return False
```

**Consequence:** Feature completely non-functional. Burst loss never triggered.

**Correct Approach:**
```python
def __init__(self):
    self.last_loss_state: Dict[str, bool] = {}  # Track loss per vehicle

def transmit(self, ...):
    was_lost = random.random() < loss_prob
    self.last_loss_state[vehicle_id] = was_lost  # ACTUALLY update state

def _check_burst_loss(self, vehicle_id):
    if self.last_loss_state.get(vehicle_id, False):  # Check tracked state
        return random.random() < 0.5
    return False
```

**Check Before Implementing Stateful Logic:**
- [ ] Is state actually populated/updated?
- [ ] Can I trace state flow from init → update → read?
- [ ] Does the state reset properly?
- [ ] Would a test catch missing state updates?

---

### Rule 4: Architectural Review for Critical Systems

**When to Request Architecture Review:**
- Implementing any system with time/causality (simulations, networks, delays)
- Implementing stochastic processes (packet loss, noise, randomization)
- Integrating simulation with real hardware (Sim2Real)
- Building training pipelines for ML

**Required Questions Before Implementation:**
1. "What are the invariants this system must maintain?" (e.g., causality)
2. "How will I test this invariant?" (e.g., test that future messages aren't visible)
3. "What happens if this is wrong?" (e.g., trained model fails on hardware)
4. "Have I designed the architecture on paper first?"

**Red Flags:**
- [ ] Implementing complex logic without a design document
- [ ] "It will probably work" instead of "I can prove it works"
- [ ] State variables initialized but never updated
- [ ] No clear data flow diagram

---

## 📝 Lessons Learned Log

### December 26, 2025 - ESPNOWEmulator Causality Bug

**Mistake:**
1. Implemented full emulator (984 lines) without tests (TDD violation)
2. Messages visible immediately despite future arrival time (causality bug)
3. Burst loss state never tracked (dead code)

**Impact:**
- Would cause Sim2Real failure (agent learns from impossible information)
- 2-3 hours to refactor
- Could have been caught by 10 minutes of test-first development

**Fix Applied:**
- Refactored with PriorityQueue for pending messages
- Fixed burst loss state tracking
- Added architectural design checks to this document

**Prevention:**
- Always write tests first
- Design time-sensitive systems on paper before coding
- Review state tracking logic before implementation

---

**Prevention:** 2 minutes of verification saves hours of debugging.

---

**Last Updated:** December 26, 2025
**Updated By:** ESPNOWEmulator refactor lessons - TDD violations and causality bugs
