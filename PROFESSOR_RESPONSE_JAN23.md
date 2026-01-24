# Response to Professor's Comments - January 23, 2026

**Student:** Amir Khalifa
**Project:** RoadSense V2V Hazard Detection System
**Date:** January 23, 2026

---

## Comment 1: Dataset Gathering & Augmentation

**Your Comment:**
> "You decided to NOT do recordings? Because you want to hallucinate with your imagination, so you have more options????"

**Acknowledged Issue:**
Training is **SUMO-first**. Real RTT logs exist but are **not yet integrated** into the emulator/training pipeline, and real recordings are **not yet used as the augmentation base**.

**Resolution - Hybrid Approach:**

1. **RTT Characterization Experiment** (immediate next step)
   - Measure real ESP-NOW latency, packet loss, and jitter at multiple distances (1m, 10m, 30m, 50m)
   - Already designed in `docs/00_ARCHITECTURE/ESPNOW_EMULATOR_DESIGN.md`
   - Output: `emulator_params.json` with measured values

2. **Short Real Recording (Mandatory Base)**
   - Record 5-minute 3-car convoy scenario with V2V logging enabled
   - Captures real GPS, IMU, and V2V message data
   - Used as the **base** for augmentation and emulator validation

3. **Training with Measured Parameters**
   - SUMO simulation continues (necessary for scale and safety)
   - ESP-NOW emulator uses **measured** parameters instead of guesses
   - Validate against the real recording to measure Sim2Real gap

```
REVISED WORKFLOW:
┌─────────────────────────────────────────────────────────────┐
│  1. RTT Characterization (measure real ESP-NOW params)      │
│       ↓                                                     │
│  2. Short Real Recording (REQUIRED base, 3-car convoy)      │
│       ↓                                                     │
│  3. SUMO Training with MEASURED emulator parameters         │
│       ↓                                                     │
│  4. Validate model against real recording                   │
│       ↓                                                     │
│  5. Iterate if Sim2Real gap > 25%                           │
└─────────────────────────────────────────────────────────────┘
```

**Expanded Training Flow (per your 1–7):**
1. Record a few physical cars (3-car convoy)
2. Generate traffic with augmentations
3. Create multiple scenarios from augmented data
4. Train by applying **ego actions** while non‑ego follows the scenario DB (as‑is)
5. Visualize simulation results (SUMO GUI / episode replays)
6. Select the checkpoint that **converges** and minimizes crashes
7. Quantize and deploy the AI agent on ESP32 (V001)

---

## Comment 2: Use Case Diagram

**Your Feedback:**
- "System" cannot be an actor using itself
- "Other cars" don't "use the system"
- Need proper <<include>> and <<extend>> relationships
- Use ellipses for use cases

**Corrected Design:**

**System Boundary:** RoadSense firmware (sensors are external actors)

**Actors (Who uses the system?):**
- **Driver** (primary) - receives hazard alerts
- **GPS Sensor** (secondary) - external component providing position data
- **IMU Sensor** (secondary) - external component providing motion data

**NOT Actors:**
- System (cannot be an actor using itself)
- Other cars / V2V Peers (they broadcast into air, don't "use" our system)
- ESP-NOW (communication medium, not a user)
- SD Card (development only)

**Use Cases with Proper Relationships:**

```
                           ╔═══════════════════════════════════════════╗
                           ║        RoadSense Vehicle Unit             ║
                           ║                                           ║
    ┌───────┐              ║                                           ║
    │ GPS   │──────────────╫───►(  Get Position Data  )               ║
    │Sensor │              ║           │                               ║
    └───────┘              ║           │ <<include>>                   ║
                           ║           ▼                               ║
    ┌───────┐              ║     (  Collect Sensor Data  )◄──[Timer]  ║
    │ IMU   │──────────────╫───►       │              (internal)       ║
    │Sensor │              ║           │                               ║
    └───────┘              ║           │ <<include>>                   ║
                           ║           ▼                               ║
                           ║     (  Broadcast Own State  )             ║
                           ║                                           ║
                           ║     (  Receive V2V Broadcast  )◄──[Event] ║
                           ║           │                  (async)      ║
                           ║           │ <<include>>                   ║
                           ║           ▼                               ║
                           ║     (  Process Peer Data  )               ║
                           ║           │                               ║
                           ║           │ <<include>>                   ║
                           ║           ▼                               ║
                           ║     (  Detect Collision Risk  )           ║
                           ║           │                               ║
                           ║           │ <<extend>> [if risk > 0]      ║
                           ║           ▼                               ║
    ┌───────┐              ║     (  Alert Driver  )                    ║
    │Driver │◄─────────────╫───────────┘                               ║
    └───────┘              ║                                           ║
                           ╚═══════════════════════════════════════════╝
```

**Key Points:**
- All use cases shown in **ellipses** (parentheses = ellipse in ASCII)
- **<<include>>** for sequential dependencies (Collect → Broadcast)
- **<<extend>>** for conditional behavior (Alert only if risk detected)
- Internal triggers (Timer, V2V RX Event) shown as notes, not actors

---

## Comment 3: Timestamp vs SequenceNumber

**Your Comment:**
> "so ... what is the change you want to make? If you want to make a change .. change ... don't talk ..."

**The Change:**

| Before | After |
|--------|-------|
| `uint32_t timestamp` | `uint32_t sequenceNumber` |

**Rationale:**
- Current name "timestamp" implies wall-clock time (requires clock sync)
- Actually used as monotonic counter for deduplication
- New name `sequenceNumber` reflects actual usage
- Staleness is computed from **local receive time**, not sender time

**Files to Change:**
1. `V2VMessage.h` - rename field
2. `PackageManager.cpp` - update dedup logic
3. `PackageManager.h` - update comments
4. `VehicleState.h` - rename field
5. `espnow_emulator.py` - update field name
6. `observation_builder.py` - update age calculation

**Status:** Documented. Will implement in next code session.

---

## Comment 4: Data Flow Diagram

**Your Questions:**
1. Where is MESH maintained?
2. Where is "cone-filtering"?
3. What is "feature builder"?
4. Where is AI-Agent clearly?
5. What is "state manager"?

**Answers:**

| Question | Answer |
|----------|--------|
| Where is MESH? | **PackageManager** - handles deduplication, per-peer storage, 60s cleanup |
| Where is cone-filtering? | **V001-only AI inference step** before Observation Builder (front FOV) |
| What is feature builder? | **REMOVED** - was simulation-only concept, wrongly shown on vehicle |
| Where is AI-Agent? | **AI Inference Subsystem** - now shown prominently with Deep Sets components |
| What is state manager? | **Renamed to "Peer State Storage"** - stores Map<MAC, VehicleState> |

**Key Fix:** Separated embedded system diagram from training pipeline diagram. They were incorrectly mixed before.

**Corrected Embedded Data Flow:**

```
┌────────────────────────────────────────────────────────────────────┐
│                 VEHICLE UNIT NODE (ESP32)                          │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ SENSOR SUBSYSTEM                                            │   │
│  │  MPU6500 (I2C) ──┐                                          │   │
│  │  QMC5883L (I2C) ─┼──► Sample @10Hz ──► Sanitize Data       │   │
│  │  NEO-6M (UART) ──┘                                          │   │
│  └────────────────────────────┬───────────────────────────────┘   │
│                               │                                    │
│                               ▼                                    │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ TX SUBSYSTEM                                                │   │
│  │  Build BSM V2V Message ──► ESP-NOW TX ──────► BROADCAST    │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
│                    ◄──────────────────────────── FROM AIR (V2V)   │
│                               │                                    │
│  ┌────────────────────────────┼───────────────────────────────┐   │
│  │ RX SUBSYSTEM               ▼                                │   │
│  │  ESP-NOW RX Callback ──► RX Queue (ISR-safe)               │   │
│  │                               │                             │   │
│  │                               ▼                             │   │
│  │  ┌─────────────────────────────────────────────────────┐   │   │
│  │  │ PACKAGE MANAGER (MESH)                              │   │   │
│  │  │  • Deduplication by MAC + seqNum                    │   │   │
│  │  │  • Staleness check: discard > 500ms                 │   │   │
│  │  │  • Max 3 msgs per peer, 20 peers max                │   │   │
│  │  └────────────────────────────┬────────────────────────┘   │   │
│  └───────────────────────────────┼────────────────────────────┘   │
│                                  │                                 │
│  ┌───────────────────────────────┼────────────────────────────┐   │
│  │ STATE SUBSYSTEM               ▼                             │   │
│  │  ┌─────────────────────────────────────────────────────┐   │   │
│  │  │ PEER STATE STORAGE                                  │   │   │
│  │  │  Map<MAC, VehicleState>                             │   │   │
│  │  │  Stores latest state per peer                       │   │   │
│  │  │  Provides: n peers, their states                    │   │   │
│  │  └────────────────────────────┬────────────────────────┘   │   │
│  │                    ┌──────────┴──────────┐                  │   │
│  │                    │                     │                  │   │
│  │               EGO STATE            PEER STATES              │   │
│  │               (own data)           (variable n)             │   │
│  └────────────────────┼─────────────────────┼─────────────────┘   │
│                       └──────────┬──────────┘                      │
│                                  │                                 │
│  ┌───────────────────────────────┼────────────────────────────┐   │
│  │ AI INFERENCE SUBSYSTEM (V001 ONLY)                          │   │
│  │                               ▼                             │   │
│  │  Cone Filter (front FOV, V001 only) ──► Observation Builder │   │
│  │  ──► Per-Peer Encoder (φ) ──►                                 │   │
│  │  ──► MAX POOLING ──► Policy Head (π) ──► ACTION            │   │
│  │                                                             │   │
│  │  Actions: 0=Maintain, 1=Caution, 2=Brake, 3=Emergency      │   │
│  └───────────────────────────────┬────────────────────────────┘   │
│                                  │                                 │
│  ┌───────────────────────────────┼────────────────────────────┐   │
│  │ OUTPUT SUBSYSTEM              ▼                             │   │
│  │  Alert (V2V TX)  │  Buzzer/LED  │  SD Card Log             │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘

NOTES:
 • V002/V003 skip AI INFERENCE (they are "dumb broadcasters")
 • Only V001 runs inference (the "intelligent ego vehicle")
 • Cone filtering is **V001-only** inside AI Inference (front FOV)
 • Training pipeline is a SEPARATE diagram (not shown here)
```

---

## Summary of Changes

| Issue | Status | Action |
|-------|--------|--------|
| Dataset (Comment 1) | **Plan Ready** | Execute RTT characterization + real recording as augmentation base |
| Use Case Diagram (Comment 2) | **Design Ready** | Rebuild in draw.io with ellipses, correct actors |
| Timestamp rename (Comment 3) | **Documented** | Will implement: ~40 lines across 6 files |
| Data Flow Diagram (Comment 4) | **Design Ready** | Fix existing diagram: remove Feature Builder, rename State Manager, add MESH + Cone Filter |

---

## Next Steps

1. **Immediate (This Week)**
   - Execute RTT characterization experiment with 2 ESP32 units
   - Record measured parameters to `emulator_params.json`

2. **Short Term (Next Week)**
   - Record 5-minute 3-car convoy scenario
   - Implement cone filtering in Observation Builder and mirror in simulation
   - Update draw.io diagrams based on ASCII designs above

3. **Medium Term**
   - Implement timestamp → sequenceNumber rename
   - Retrain model with measured emulator parameters
   - Validate against real recording
