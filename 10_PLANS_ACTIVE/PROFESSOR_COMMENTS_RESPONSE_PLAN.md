# Professor Comments Response Plan - Jan 23, 2026

## Executive Summary

The professor's comments identify **4 critical issues** that require architectural changes and diagram corrections. This document provides:
1. Impact assessment for each issue
2. Corrected ASCII diagrams
3. Specific action items for resolution

## User Decisions
- **Data Strategy**: Hybrid Approach - RTT characterization + **mandatory** short real recording as base + SUMO training with measured params
- **Scope**: Documentation only (no code changes in this session)
- **Output Files**:
  - `docs/10_PLANS_ACTIVE/PROFESSOR_COMMENTS_RESPONSE_PLAN.md` (this file)
  - `docs/PROFESSOR_RESPONSE_JAN23.md` (direct professor feedback file)

## Session Documentation Updates (Jan 23)
- [x] Make real recording **mandatory base** for augmentation
- [x] Clarify cone filtering placement and **V001-only** scope
- [x] Clarify system boundary for use case actors
- [x] Clarify staleness uses local RX time (not sender time)
- [x] Replace "guessed params" wording with "uncalibrated / not integrated"

---

## Comment Analysis & Impact Assessment

### Comment 1: Dataset Gathering & Augmentation (CRITICAL)

**Professor's Concern:**
> "You decided to NOT do recordings? Because you want to hallucinate with your imagination, so you have more options????"

**Current System Reality:**
Training is **SUMO-first**. Real RTT logs exist but are **not yet integrated** into the emulator/training pipeline, and real recordings are **not yet used as the augmentation base**.

```
CURRENT WORKFLOW (What we have):
┌─────────────────────────────────────────────────────────────┐
│  SUMO Synthetic Scenarios                                   │
│       ↓                                                     │
│  Scenario Augmentation (vType params, routes, peer drops)   │
│       ↓                                                     │
│  ESP-NOW Emulator (uncalibrated params; measured logs not yet integrated) │
│       ↓                                                     │
│  RL Training (PPO + Deep Sets)                              │
│       ↓                                                     │
│  TFLite Model for ESP32                                     │
└─────────────────────────────────────────────────────────────┘

PROFESSOR'S EXPECTATION (What he wants):
┌─────────────────────────────────────────────────────────────┐
│  Record 3 Physical Cars (REAL DATA)                         │
│       ↓                                                     │
│  Augment Real Recordings (slight manipulations)             │
│       ↓                                                     │
│  Generate Virtual Cars from Real Basis                      │
│       ↓                                                     │
│  Train on Augmented Real Data                               │
│       ↓                                                     │
│  Validate on Real Hardware                                  │
└─────────────────────────────────────────────────────────────┘
```

**Impact Level: CRITICAL (Architectural)**

**Gap Analysis:**
| Aspect | Professor Expects | Current System |
|--------|------------------|----------------|
| Data Source | Real 3-car recordings | SUMO synthetic |
| Augmentation Basis | Real data + manipulations | Synthetic + parameter jitter |
| Network Model | Measured from hardware | Measured logs not yet integrated |
| Validation | Real-world test | Partial (no real-scenario baseline yet) |

**Resolution: Hybrid Approach**
1. Execute RTT characterization experiment (already designed in `ESPNOW_EMULATOR_DESIGN.md`)
2. **Record a 5-minute 3-car convoy scenario** with V2V logging (mandatory base)
3. Use recorded data to:
   - Validate emulator parameters
   - Generate "real-grounded" augmentations (base scenario derived from real logs)
4. Continue SUMO training at scale but with **measured** network parameters

**Files Requiring Changes (Future):**
- `ml/espnow_emulator/espnow_emulator.py` - Load measured params
- `ml/scripts/gen_scenarios.py` - Accept real data input
- NEW: `hardware/src/logging/CharacterizationLogger.h` - RTT logging
- NEW: `ml/scripts/parse_real_recordings.py` - Convert real logs

---

### Comment 2: Use Case Diagram (MEDIUM)

**Professor's Concerns:**
1. "System" as an actor using itself is wrong
2. "Other cars" don't "use the system"
3. "Collect Sensor Data" trigger relationship is incorrect
4. Need <<include>> and <<extend>> properly

**Current Diagram Issues:**
Looking at `NEEDS_FIXING_Use_Case_Diagram.drawio.xml`:
- Has "System Timer (10Hz loop)" as an actor - this is INTERNAL, not external
- Has "V2V Peers" as actor - they don't USE the system, they're external broadcasters
- "Collect sensor data" → "Broadcast Message" with <<include>> - incorrect trigger

**Corrected Use Case Diagram (ASCII):**

The professor's guidance: *"Who uses the system? Those are the actors. How does each actor use the system: those are the direct use cases."*

For RoadSense V2V (system boundary = **RoadSense firmware**):
- **Primary Actor**: The **Driver** - they benefit from hazard alerts
- **Secondary Actors**: External sensors the system uses (GPS, IMU hardware)
- **NOT actors**: "Other cars" (they just broadcast into air), ESP-NOW (it's a medium, not a user), SD Card (development only)
- **Internally triggered use cases** are valid (timer-based)

```
                                ╔════════════════════════════════════════════════════════╗
                                ║            RoadSense Vehicle Unit                       ║
                                ║                                                         ║
        ┌───────┐               ║                                                         ║
        │ GPS   │───────────────╫────────►(  Get Position Data  )                        ║
        │Sensor │               ║               │                                         ║
        └───────┘               ║               │ <<include>>                             ║
                                ║               ▼                                         ║
        ┌───────┐               ║         (  Collect Sensor Data  )◄─────[Timer: 100ms]  ║
        │  IMU  │───────────────╫────────►        │                    (internal trigger) ║
        │Sensor │               ║                 │                                       ║
        └───────┘               ║                 │ <<include>>                           ║
                                ║                 ▼                                       ║
                                ║         (  Broadcast Own State  )                       ║
                                ║                                                         ║
                                ║                                                         ║
                                ║         (  Receive V2V Broadcast  )◄───[V2V RX Event]  ║
                                ║                 │                    (async callback)   ║
                                ║                 │                                       ║
                                ║                 │ <<include>>                           ║
                                ║                 ▼                                       ║
                                ║         (  Process Peer Data  )                         ║
                                ║                 │                                       ║
                                ║                 │ <<include>>                           ║
                                ║                 ▼                                       ║
                                ║         (  Detect Collision Risk  )                     ║
                                ║                 │                                       ║
                                ║                 │ <<extend>> [if risk > threshold]      ║
                                ║                 ▼                                       ║
        ┌───────┐               ║         (  Alert Driver  )                              ║
        │Driver │◄──────────────╫─────────────────┘                                       ║
        │       │               ║                                                         ║
        └───────┘               ║                                                         ║
                                ╚════════════════════════════════════════════════════════╝

ACTORS:
  ┌───────┐
  │ GPS   │  Secondary actor: External sensor system provides position
  └───────┘

  ┌───────┐
  │ IMU   │  Secondary actor: External sensor system provides motion data
  └───────┘

  ┌───────┐
  │Driver │  Primary actor: Human who receives hazard alerts (passive beneficiary)
  └───────┘

INTERNAL TRIGGERS (not actors, just timing mechanisms):
  [Timer: 100ms]     - System timer triggers sensor collection + broadcast
  [V2V RX Event]     - Async callback when packet arrives from air

NOT INCLUDED (per professor guidance):
  ✗ "System" as actor         - System can't be an actor using itself
  ✗ "Other cars" as actors    - They broadcast into air, don't "use" our system
  ✗ "V2V Peers" as actors     - Same reason
  ✗ ESP-NOW as actor          - It's a communication medium, not a user
  ✗ SD Card                   - Development/debug only, not final product
```

**Key Professor Requirements Addressed:**
1. ✓ Use ellipses for use cases (parentheses in ASCII = ellipse placeholder)
2. ✓ No table duplication
3. ✓ System is NOT an actor
4. ✓ Other cars are NOT actors (they just broadcast into air)
5. ✓ Proper <<include>> for sequential dependencies
6. ✓ Proper <<extend>> for conditional behavior (Alert only if risk detected)
7. ✓ Internal timer-triggered use cases shown correctly
8. ✓ External sensors as secondary actors (system uses them)

**Impact Level: MEDIUM (Documentation)**

**Files Requiring Changes:**
- `docs/NEEDS_FIXING_Use_Case_Diagram.drawio.xml` - Complete redesign

---

### Comment 3: Timestamp vs SequenceNumber (LOW-MEDIUM)

**Professor's Concern:**
> "so ... what is the change you want to make? If you want to make a change .. change ... don't talk ..."

**Current Code Reality:**
The field `timestamp` in V2VMessage is used for:
1. **Deduplication** - `if (pkg.message.timestamp == msg.timestamp)` in PackageManager.cpp:79
2. **Staleness checking** - `isStale()` checks `(millis() - timestamp) > maxAge`
3. **Age calculation** - `age()` returns `millis() - timestamp`

**The Issue:**
- "timestamp" implies wall-clock time, which requires clock sync between vehicles
- Actually used as a **sequence/freshness identifier** for deduplication
- Causes confusion about cross-vehicle time synchronization

**Resolution (Documented for Future Implementation):**
Rename `timestamp` → `sequenceNumber` and clarify semantics:

```c
// BEFORE (current):
struct V2VMessage {
    uint32_t timestamp;  // millis() - confusing name
    ...
};

// AFTER (proposed):
struct V2VMessage {
    uint32_t sequenceNumber;  // Monotonic counter from sender's boot
    ...

    // Methods updated:
    bool isNewerThan(uint32_t other) const { return sequenceNumber > other; }
    uint32_t localAge() const { return millis() - receiveTime; }  // staleness uses local RX time
};
```

**Impact Level: LOW-MEDIUM (Code refactor)**

**Files Requiring Changes (Future):**
1. `hardware/src/network/protocol/V2VMessage.h` - Rename field
2. `hardware/src/network/mesh/PackageManager.cpp` - Update dedup logic
3. `hardware/src/network/mesh/PackageManager.h` - Update comments
4. `hardware/include/VehicleState.h` - Rename field
5. `ml/espnow_emulator/espnow_emulator.py` - Update field name
6. `ml/envs/observation_builder.py` - Update age_ms calculation

---

### Comment 4: Data Flow Diagram (CRITICAL)

**Professor's Concerns:**
1. Mixes ML training (simulator) with AI Agent (inference on car)
2. Where is MESH maintained?
3. Where is "cone-filtering"?
4. What is "feature builder"?
5. Where is AI-Agent clearly?
6. What is "state manager"?

**Current Diagram Issues:**
Looking at `NEEDS_FIXING_Data_Flow_Diagram.drawio.xml`:
- Mixed embedded + training concepts (professor wants them separate)
- "State Manager" is unclear (what does it do?)
- "Feature Builder" is simulation-only concept wrongly shown on vehicle
- AI Agent not clearly visible

**Resolution - Professor's Questions Answered:**

| Question | Answer | Diagram Change |
|----------|--------|----------------|
| Where is MESH maintained? | **PackageManager** - handles dedup, per-peer storage, cleanup | Label clearly in diagram |
| Where is cone-filtering? | **V001-only AI inference step** (before Observation Builder, mirrored in sim) | Add explicit "Cone Filter" block in AI Inference |
| What is feature builder? | **REMOVE** - it's a simulation-only concept | Delete from embedded diagram |
| Where is AI-Agent? | **AI Inference block** - runs TFLite model | Make larger, more prominent |
| What is state manager? | **Peer State Storage** - stores Map<MAC, VehicleState> | Rename and clarify purpose |

---

## Corrected Data Flow Diagram (ASCII)

### Vehicle Unit Data Flow (Embedded System - Shows Whole System)

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                     VEHICLE UNIT NODE i (ESP32)                                  │
│                     ════════════════════════════                                 │
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────────┐│
│  │                        SENSOR SUBSYSTEM                                      ││
│  │  ┌──────────┐   ┌──────────┐   ┌──────────┐                                 ││
│  │  │  MPU6500 │   │ QMC5883L │   │  NEO-6M  │                                 ││
│  │  │   IMU    │   │   MAG    │   │   GPS    │                                 ││
│  │  └────┬─────┘   └────┬─────┘   └────┬─────┘                                 ││
│  │       │ I2C          │ I2C          │ UART                                   ││
│  │       └──────────────┴──────────────┘                                        ││
│  │                      │                                                        ││
│  │                      ▼                                                        ││
│  │              ┌───────────────┐                                               ││
│  │              │ Sample @10Hz  │   [System Timer triggers every 100ms]         ││
│  │              │ Sanitize Data │                                               ││
│  │              └───────┬───────┘                                               ││
│  └──────────────────────┼───────────────────────────────────────────────────────┘│
│                         │                                                        │
│                         ▼                                                        │
│  ┌──────────────────────────────────────────────────────────────────────────────┐│
│  │                      TX SUBSYSTEM                                            ││
│  │              ┌───────────────┐                                               ││
│  │              │ Build BSM V2V │                                               ││
│  │              │    Message    │  [90 bytes: header + BSM + sensors + alert]   ││
│  │              └───────┬───────┘                                               ││
│  │                      │                                                        ││
│  │                      ▼                                                        ││
│  │              ┌───────────────┐                                               ││
│  │              │  ESP-NOW TX   │────────────────────────────────► BROADCAST    ││
│  │              │  (Broadcast)  │                                   TO AIR      ││
│  │              └───────────────┘                                               ││
│  └──────────────────────────────────────────────────────────────────────────────┘│
│                                                                                  │
│                         ◄───────────────────────────────────────── FROM AIR     │
│                         │                                          (V2V RX)     │
│  ┌──────────────────────┼───────────────────────────────────────────────────────┐│
│  │                      ▼           RX SUBSYSTEM                                ││
│  │              ┌───────────────┐                                               ││
│  │              │  ESP-NOW RX   │   [Async callback on packet arrival]          ││
│  │              │   Callback    │                                               ││
│  │              └───────┬───────┘                                               ││
│  │                      │                                                        ││
│  │                      ▼                                                        ││
│  │              ┌───────────────┐                                               ││
│  │              │  RX Queue     │   [Thread-safe queue, max 20 msgs]            ││
│  │              │  (ISR-safe)   │                                               ││
│  │              └───────┬───────┘                                               ││
│  │                      │                                                        ││
│  │                      ▼                                                        ││
│  │              ┌───────────────┐                                               ││
│  │              │ PACKAGE       │   [Deduplication by MAC + seqNum]             ││
│  │              │ MANAGER       │   [Staleness check: discard > 500ms]          ││
│  │              │ (MESH)        │   [Max 3 msgs per peer, 20 peers max]         ││
│  │              └───────┬───────┘                                               ││
│  │                      │                                                        ││
│  │                      │ Valid, fresh messages                                  ││
│  │                      ▼                                                        ││
│  └──────────────────────────────────────────────────────────────────────────────┘│
│                         │                                                        │
│  ┌──────────────────────┼───────────────────────────────────────────────────────┐│
│  │                      ▼         STATE SUBSYSTEM                               ││
│  │              ┌───────────────┐                                               ││
│  │              │ PEER STATE    │   [Map<MAC, VehicleState>]                    ││
│  │              │ STORAGE       │   [Stores latest state per peer]              ││
│  │              │               │   [Provides: n peers, their states]           ││
│  │              └───────┬───────┘                                               ││
│  │                      │                                                        ││
│  │        ┌─────────────┴─────────────┐                                         ││
│  │        │                           │                                         ││
│  │        ▼                           ▼                                         ││
│  │  ┌───────────┐             ┌───────────────┐                                 ││
│  │  │ EGO STATE │             │ PEER STATES   │   [Variable n: 0 to 8+]        ││
│  │  │ (own data)│             │ (from V2V RX) │                                 ││
│  │  └─────┬─────┘             └───────┬───────┘                                 ││
│  │        │                           │                                         ││
│  │        └───────────┬───────────────┘                                         ││
│  │                    │                                                          ││
│  └────────────────────┼─────────────────────────────────────────────────────────┘│
│                       │                                                          │
│  ┌────────────────────┼─────────────────────────────────────────────────────────┐│
│  │                    ▼      AI INFERENCE SUBSYSTEM (V001 ONLY)                 ││
│  │            ┌───────────────┐                                                 ││
│  │            │ CONE FILTER   │   [Front FOV only; V001 receiver-side]          ││
│  │            └───────┬───────┘                                                 ││
│  │                    │                                                          ││
│  │                    ▼                                                          ││
│  │            ┌───────────────┐                                                 ││
│  │            │ OBSERVATION   │   [Builds Deep Sets input from filtered peers] ││
│  │            │ BUILDER       │   [Ego: 4 dims, Per-peer: 6 dims]               ││
│  │            └───────┬───────┘                                                 ││
│  │                    │                                                          ││
│  │                    ▼                                                          ││
│  │            ┌───────────────┐                                                 ││
│  │            │ PER-PEER      │   [TFLite: 6 → 32 dims, shared φ]               ││
│  │            │ ENCODER (φ)   │   [Run for each of n peers]                     ││
│  │            └───────┬───────┘                                                 ││
│  │                    │                                                          ││
│  │                    ▼                                                          ││
│  │            ┌───────────────┐                                                 ││
│  │            │ MAX POOLING   │   [Aggregate n embeddings → 32 dims]            ││
│  │            │ (in C code)   │                                                 ││
│  │            └───────┬───────┘                                                 ││
│  │                    │                                                          ││
│  │                    ▼                                                          ││
│  │            ┌───────────────┐                                                 ││
│  │            │ POLICY HEAD   │   [TFLite: 36 → 4 action logits]                ││
│  │            │ (π network)   │                                                 ││
│  │            └───────┬───────┘                                                 ││
│  │                    │                                                          ││
│  │                    ▼                                                          ││
│  │         ┌─────────────────────┐                                              ││
│  │         │ ACTION SELECTION    │                                              ││
│  │         │ 0: Maintain         │                                              ││
│  │         │ 1: Caution (-1 m/s) │                                              ││
│  │         │ 2: Brake (-3 m/s)   │                                              ││
│  │         │ 3: Emergency (-5m/s)│                                              ││
│  │         └─────────┬───────────┘                                              ││
│  │                   │                                                           ││
│  └───────────────────┼──────────────────────────────────────────────────────────┘│
│                      │                                                           │
│                      ▼                                                           │
│  ┌───────────────────────────────────────────────────────────────────────────────┐
│  │                    OUTPUT SUBSYSTEM                                           │
│  │  ┌───────────────┐   ┌───────────────┐   ┌───────────────┐                   │
│  │  │ ALERT (V2V TX)│   │ BUZZER / LED  │   │  SD CARD LOG  │                   │
│  │  │ (risk in msg) │   │ (local alert) │   │ (10Hz logging)│                   │
│  │  └───────────────┘   └───────────────┘   └───────────────┘                   │
│  └───────────────────────────────────────────────────────────────────────────────┘
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘

NOTES (Professor's Questions Addressed):
  • MESH maintained: PackageManager (dedup by MAC+seqNum, per-peer storage, 60s cleanup)
  • Cone filtering: V001-only AI inference step before Observation Builder (front FOV)
  • Feature Builder: REMOVED (simulation-only concept, not on embedded)
  • AI Agent: Clearly shown as "AI INFERENCE SUBSYSTEM" with 3 components
  • State Manager: Renamed to "PEER STATE STORAGE" - stores Map<MAC, VehicleState>

ARCHITECTURE:
  • V002/V003 skip the "AI INFERENCE SUBSYSTEM" entirely (dumb broadcasters)
  • Only V001 runs inference - it's the "intelligent ego vehicle"
  • State storage provides variable-n peer access for Deep Sets
```

**Key Changes from Current Diagram:**
1. ✓ REMOVED "Feature Builder" (was simulation-only, wrongly shown on vehicle)
2. ✓ RENAMED "State Manager" → "PEER STATE STORAGE" (clearer purpose)
3. ✓ ADDED "PackageManager (MESH)" label with explicit mesh annotation
4. ✓ ADDED explicit Cone Filter block inside AI Inference (front FOV only)
5. ✓ MADE AI Agent more prominent (entire subsystem block)
6. ✓ SEPARATED from training flow (this is embedded ONLY)

---

## Simulation & Training Flow (SEPARATE DIAGRAM - Reference Only)

**NOTE:** This is a SEPARATE diagram showing the offline training pipeline. It is NOT part of the "Data Flow Diagram" that shows the embedded vehicle system. The professor asked for these to be clearly separated.

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│              SIMULATION & ML TRAINING PIPELINE (Offline - Big Computer)          │
│              ═══════════════════════════════════════════════════════            │
│                                                                                  │
│  STEP 1: PHYSICAL CHARACTERIZATION                                              │
│  ─────────────────────────────────                                              │
│  ┌──────────────┐                  ┌──────────────┐                             │
│  │ ESP32 V001   │◄─── RTT ping ───►│ ESP32 V002   │   [Measure real latency]    │
│  │ (Sender)     │                  │ (Reflector)  │   [Measure real packet loss]│
│  └──────┬───────┘                  └──────────────┘                             │
│         │                                                                        │
│         ▼                                                                        │
│  ┌──────────────────┐                                                           │
│  │ RTT Logs (CSV)   │   [seq, send_ms, recv_ms, rtt_ms, distance]               │
│  │ @ 1m,10m,30m,50m │                                                           │
│  └────────┬─────────┘                                                           │
│           │                                                                      │
│           ▼                                                                      │
│  ┌──────────────────┐                                                           │
│  │ Analyze & Fit    │   [Calculate: mean_latency, std, loss_rate per distance]  │
│  │ Parameters       │                                                           │
│  └────────┬─────────┘                                                           │
│           │                                                                      │
│           ▼                                                                      │
│  ┌──────────────────┐                                                           │
│  │ emulator_params  │   [JSON: measured latency, loss, jitter models]           │
│  │ .json            │                                                           │
│  └────────┬─────────┘                                                           │
│           │                                                                      │
│           ════════════════════════════════════════════════                       │
│                                                                                  │
│  STEP 2: SCENARIO CREATION                                                       │
│  ─────────────────────────                                                       │
│       ┌──────────────────┐                                                      │
│       │ REAL RECORDING   │   [REQUIRED: 3-car convoy, 5 minutes]                │
│       │ (3 physical cars)│   [Logs: GPS, IMU, V2V messages]                     │
│       └────────┬─────────┘                                                      │
│                │                                                                 │
│                ▼                                                                 │
│       ┌──────────────────┐         ┌──────────────────┐                         │
│       │ Parse Real Logs  │────────►│ Base Scenario    │                         │
│       │ to SUMO Format   │         │ (network + routes│                         │
│       └──────────────────┘         │ + vehicle types) │                         │
│                                    └────────┬─────────┘                         │
│                OR                           │                                    │
│       ┌──────────────────┐                  │                                    │
│       │ SUMO Synthetic   │─────────────────►│                                    │
│       │ Base Scenario    │                  │                                    │
│       └──────────────────┘                  │                                    │
│                                             ▼                                    │
│  ┌──────────────────────────────────────────────────────────────────────────────┐
│  │                    SCENARIO AUGMENTATION                                      │
│  │  ┌──────────────────┐                                                        │
│  │  │ Scenario         │   [Input: base scenario]                               │
│  │  │ Generator        │   [Augments:]                                          │
│  │  │ (gen_scenarios.py│     • Peer dropout (variable n)                        │
│  │  └────────┬─────────┘     • Route randomization                              │
│  │           │               • vType param jitter (speed, decel, tau)           │
│  │           │               • Spawn timing variation                           │
│  │           ▼                                                                   │
│  │  ┌──────────────────┐                                                        │
│  │  │ Augmented Dataset│   [20+ train, 4+ eval scenarios]                       │
│  │  │ (scenarios/)     │   [Each: .sumocfg, .rou.xml, .net.xml]                 │
│  │  └──────────────────┘                                                        │
│  └──────────────────────────────────────────────────────────────────────────────┘
│                                             │                                    │
│           ════════════════════════════════════════════════                       │
│                                             │                                    │
│  STEP 3: TRAINING LOOP (Ego actions + scenario DB as-is)     ▼                   │
│  ───────────────────────────────────────────────────────────                      │
│  ┌──────────────────────────────────────────────────────────────────────────────┐
│  │                                                                               │
│  │  FOR each episode:                                                           │
│  │  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │  │                                                                         │  │
│  │  │  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐              │  │
│  │  │  │  SUMO        │───►│ FCD Output   │───►│ ESP-NOW      │              │  │
│  │  │  │  Simulation  │    │ (10Hz state) │    │ EMULATOR     │              │  │
│  │  │  │  (TraCI)     │    │              │    │ (add latency,│              │  │
│  │  │  └──────────────┘    └──────────────┘    │ loss, jitter)│              │  │
│  │  │                                          └──────┬───────┘              │  │
│  │  │                                                 │                       │  │
│  │  │                                                 ▼                       │  │
│  │  │                                          ┌──────────────┐              │  │
│  │  │                                          │ ConvoyEnv    │              │  │
│  │  │                                          │ (Gymnasium)  │              │  │
│  │  │                                          │              │              │  │
│  │  │                                          │ obs: {ego,   │              │  │
│  │  │                                          │  peers, mask}│              │  │
│  │  │                                          └──────┬───────┘              │  │
│  │  │                                                 │                       │  │
│  │  │         ┌───────────────────────────────────────┘                       │  │
│  │  │         │                                                               │  │
│  │  │         ▼                                                               │  │
│  │  │  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐              │  │
│  │  │  │ Deep Sets    │───►│ PPO Update   │───►│ Policy       │              │  │
│  │  │  │ Policy       │    │ (gradient    │    │ Checkpoint   │              │  │
│  │  │  │ (π network)  │◄───│ descent)     │    │ (.pt file)   │              │  │
│  │  │  └──────────────┘    └──────────────┘    └──────────────┘              │  │
│  │  │                                                                         │  │
│  │  │  Repeat for 100,000 - 10,000,000 timesteps                             │  │
│  │  │  [Ego actions applied each step; non-ego follows scenario DB as-is]     │  │
│  │  └────────────────────────────────────────────────────────────────────────┘  │
│  │                                                                               │
│  └──────────────────────────────────────────────────────────────────────────────┘
│                                             │                                    │
│           ════════════════════════════════════════════════                       │
│                                             │                                    │
│  STEP 4: VISUALIZATION & REVIEW            ▼                                    │
│  ───────────────────────────                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────┐
│  │  ┌──────────────┐   ┌───────────────────────────────┐                       │
│  │  │ SUMO GUI /   │──►│ Review trajectories + crashes │                       │
│  │  │ Episode Replay│  │ (TTC, collisions, near-miss)   │                       │
│  │  └──────────────┘   └───────────────────────────────┘                       │
│  └──────────────────────────────────────────────────────────────────────────────┘
│                                             │                                    │
│           ════════════════════════════════════════════════                       │
│                                             │                                    │
│  STEP 5: EVALUATION & DEPLOYMENT            ▼                                    │
│  ───────────────────────────                                                    │
│  ┌──────────────────────────────────────────────────────────────────────────────┐
│  │  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐                   │
│  │  │ Run on Eval  │───►│ Select Best │───►│ TFLite INT8  │                   │
│  │  │ Scenarios    │    │ (min crashes)│  │ Quantization │                   │
│  │  └──────────────┘    └──────────────┘    └──────┬───────┘                   │
│  │                                                  │                            │
│  │                                                  ▼                            │
│  │                                          ┌──────────────┐                    │
│  │                                          │ Flash to     │                    │
│  │                                          │ ESP32 V001   │                    │
│  │                                          └──────────────┘                    │
│  └──────────────────────────────────────────────────────────────────────────────┘
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Impact Level: CRITICAL (Documentation)**

---

## Summary: Action Items by Priority

### CRITICAL (Must Fix)

| # | Action | Files | Effort |
|---|--------|-------|--------|
| 1 | Execute RTT characterization experiment | NEW: CharacterizationLogger.h | 2-3 days |
| 2 | Record real 3-car scenario (mandatory base) | Hardware + scripts | 1 day |
| 3 | Update emulator with measured params | espnow_emulator.py | 1 day |
| 4 | Fix Data Flow Diagram (remove Feature Builder, clarify Mesh/State/AI) | NEEDS_FIXING_Data_Flow_Diagram.drawio.xml | 2-3 hours |

### MEDIUM (Should Fix)

| # | Action | Files | Effort |
|---|--------|-------|--------|
| 5 | Redesign Use Case Diagram (ellipses, proper actors, remove System/Peers) | NEEDS_FIXING_Use_Case_Diagram.drawio.xml | 2-3 hours |
| 6 | Rename timestamp → sequenceNumber (documentation only for now) | 6 files documented | N/A |
| 7 | Implement cone filtering (embedded + sim) | AI inference filter + sim filter | 0.5-1 day |

### LOW (Nice to Have / Future)

| # | Action | Files | Effort |
|---|--------|-------|--------|
| 8 | Create real-data parsing pipeline | NEW: parse_real_recordings.py | Future |

---

## Draw.io Diagram Guidance

After reviewing the ASCII diagrams above:

### Use Case Diagram - Rebuild in draw.io with:
- **Ellipses** for use cases (not rectangles)
- **Actors**: Driver (primary), GPS sensor, IMU sensor (secondary)
- **NOT actors**: System, Other cars, V2V Peers, ESP-NOW, SD Card
- **Internal triggers**: Timer (100ms), V2V RX Event - shown as notes, not actors
- Use **<<include>>** for sequential dependencies
- Use **<<extend>>** for conditional behaviors (Alert only if risk > 0)

### Data Flow Diagram - FIX the existing diagram:
- Keep the same name (`NEEDS_FIXING_Data_Flow_Diagram.drawio.xml`)
- **REMOVE**: "Feature Builder" (simulation-only concept)
- **RENAME**: "State Manager" → "Peer State Storage" (clearer purpose)
- **ADD LABEL**: "PackageManager (MESH)" to show where mesh is maintained
- **ADD BLOCK**: "Cone Filter (front FOV)" inside AI Inference before Observation Builder
- **CLARIFY**: AI Agent block - make more prominent, show Deep Sets components
- This diagram shows the **whole embedded vehicle system** only
- Training pipeline is a **SEPARATE** diagram (not part of this task)

---

## Code Changes Reference (For Future Implementation)

When ready to implement code changes, here's the summary:

| File | Change | Lines |
|------|--------|-------|
| `V2VMessage.h` | Rename `timestamp` → `sequenceNumber` | ~5 |
| `PackageManager.cpp` | Update dedup logic | ~10 |
| `PackageManager.h` | Update comments | ~5 |
| `VehicleState.h` | Rename field | ~5 |
| `espnow_emulator.py` | Update field name | ~10 |
| `observation_builder.py` | Update age calculation | ~5 |

**Estimated total**: ~40 lines across 6 files
