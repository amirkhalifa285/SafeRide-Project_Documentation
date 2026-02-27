# CLAUDE.md

**Context**: Guidance for Claude Code (claude.ai/code) working on the RoadSense V2V project.
**Last Updated:** February 27, 2026

---

## 🚀 Project Overview

**RoadSense V2V System** is a Vehicle-to-Vehicle (V2V) hazard detection system using 3 ESP32 units. It combines real-time sensor data (MPU6500 IMU + QMC5883L Mag + GPS) with ESP-NOW mesh networking to enable infrastructure-free collision avoidance.

**Key Features:**
- **RL-based Collision Avoidance:** Single-agent Reinforcement Learning model running on ESP32.
- **Simulation-First:** Training in SUMO with a custom Python ESP-NOW Emulator.
- **Hardware:** ESP32 DevKits, Custom PCB/Breadboard, SD Card logging.

---

## 📁 Directory Structure & Workspace

**⚠️ NOTE:** This file (`CLAUDE.md`) and the `docs/` folder are in Amir's personal workspace, NOT the shared git repo.

### File System Layout
```
/home/amirkhalifa/RoadSense2/              ← PERSONAL WORKSPACE
├── CLAUDE.md                               ← This file
├── docs/                                   ← Documentation Root
│   ├── 00_ARCHITECTURE/                    ← System design & constraints
│   ├── 10_PLANS_ACTIVE/                    ← Current TODOs
│   ├── 20_KNOWLEDGE_BASE/                  ← How-to guides
│   ├── 90_ARCHIVE/                         ← Completed work
│   └── AGENTS/                             ← Custom Agent Prompts
└── roadsense-v2v/                         ← GIT REPOSITORY (Active Work)
    ├── hardware/                           ← ESP32 Firmware (PlatformIO)
    ├── simulation/                         ← SUMO scenarios & Python Tools
    ├── ml/                                 ← TFLite Training & Emulator
    └── common/                             ← Shared C headers (V2VMessage.h)
```

**Workflow Rule:**
- **Always `cd roadsense-v2v/`** before running git commands or build tasks.
- **Documentation:** Refer to `docs/_MAPS/INDEX.md` to find specific files.

---

## ⚡ Current System Architecture

### 1. Hardware Stack (Vehicle Units)
- **MCU:** ESP32 DevKit V1 (3 units: V001, V002, V003)
- **IMU:** **MPU6500** (Accel + Gyro) @ 0x68
  - *Note:* Originally planned MPU9250. MPU6500 lacks internal mag.
- **Magnetometer:** **QMC5883L** @ 0x0D
  - Status: ✅ **Arrived & Validated** (Dec 22, 2025). Provides true heading.
- **GPS:** **NEO-6M** (UART2)
  - ⚠️ **Critical:** Must be powered by **5V** (USB pin), not 3.3V.
- **Storage:** MicroSD Module (SPI)
  - Used for high-speed logging (10Hz) of *received* V2V messages.
- **Network:** **ESP-NOW**
  - Latency: 10-50ms (Stochastic)
  - Payload: 250 bytes max (SAE J2735 BSM format)

### 2. Simulation & ML Stack
- **Simulation Engine:** **SUMO 1.25.0+** (Dockerized)
  - *Deprecated:* Veins & OMNeT++ are **REMOVED** from scope.
- **Sim2Real Bridge:** **ESP-NOW Emulator** (Python)
  - Emulates packet loss, latency, jitter, and **multi-hop mesh relay** based on *measured hardware data*.
  - Mesh simulation via `simulate_mesh_step()` (replaces legacy per-peer `transmit()`).
  - See `00_ARCHITECTURE/ESPNOW_EMULATOR_DESIGN.md`.
- **Machine Learning:**
  - **Approach:** Reinforcement Learning (**PPO**) with **Deep Sets** architecture.
  - **Architecture:** Permutation-invariant set encoder solves the n-element problem.
  - **Action Space:** **Continuous `Box(1,)`** — model outputs deceleration fraction [0.0, 1.0].
  - **Observation:** Dict space with ego state (4-dim) + variable peer set (up to 8, 6-dim each).
  - **Agent:** Runs on **V001 (Ego Vehicle)** only.
  - **Input:** V2V messages from mesh network (direct + relayed) + Ego state.
  - **Deployment:** TensorFlow Lite for Microcontrollers (Quantized INT8).
  - See `00_ARCHITECTURE/DEEP_SETS_N_ELEMENT_ARCHITECTURE.md`.

### 3. Mesh Networking (Feb 26, 2026 — Professor Correction)
- **Protocol:** Each vehicle broadcasts its own state AND rebroadcasts data from vehicles in its **front cone**.
- **Cone Filter:** GPS-based bearing check (45 half-angle). Determines what gets relayed.
- **Relay Chain:** A (front) broadcasts → B (middle) receives + relays A's data → C (ego) receives both A (hop=1) and B (hop=0).
- **Deduplication:** By `sourceMAC + timestamp`. PackageManager handles this.
- **Max Hops:** 3 (`MAX_HOP_COUNT` in config.h).
- **Implementation Status:**
  - Firmware (`main.cpp`): ✅ Relay logic + ConeFilter + MeshRelayPolicy (18 tests passing)
  - Emulator (`espnow_emulator.py`): ✅ `simulate_mesh_step()` with multi-hop + cone filter
  - ConvoyEnv (`convoy_env.py`): ✅ Uses mesh simulation instead of per-peer transmit
  - See `docs/10_PLANS_ACTIVE/MESH_AND_ACTION_ARCHITECTURE_CORRECTION.md`.

---

## 🛑 Critical Constraints & Truths

### The "Single-Agent" Architecture
**Protocol:**
1.  **V001 (Ego)** is the intelligent agent (runs inference).
2.  **V002, V003, ... VN** are broadcasters AND mesh relayers. They broadcast their own state and rebroadcast data from vehicles in their front cone.
3.  **Logging:** Only V001 logs data. Through mesh relay, V001's RX log captures data from all reachable vehicles (direct + relayed).
4.  **Inference:** V001 runs the RL model and calculates its own deceleration fraction.

### Action Space (CRITICAL — Feb 26, 2026 Correction)
- **Continuous `Box(low=0.0, high=1.0, shape=(1,))`** — NOT Discrete(4).
- Output: fraction of maximum deceleration. `actual_decel = action * MAX_DECEL` where `MAX_DECEL = 5.0 m/s²`.
- PPO handles continuous actions natively. The model learns the percentage of braking needed.

### Documentation Authority
- **Architecture:** `docs/00_ARCHITECTURE/` is the Source of Truth.
- **Progress:** `docs/20_KNOWLEDGE_BASE/HARDWARE_PROGRESS.md` tracks daily hardware status.

### Hardware Quirks
- **I2C Bus:** SDA=21, SCL=22. Shared by MPU6500 and QMC5883L.
- **SPI Bus:** CS=5, MOSI=23, MISO=19, SCK=18 (SD Card).
- **GPS:** TX=16, RX=17.

---

## 🛠️ Common Commands

### Firmware (PlatformIO)
```bash
# Upload to specific vehicle (Check ID in main.cpp first!)
pio run --target upload --upload-port /dev/ttyUSB0

# Monitor logs
pio device monitor --baud 115200
```

### Simulation (SUMO)
```bash
# Pull Image (GitHub Registry for v1.25+)
docker pull ghcr.io/eclipse-sumo/sumo:main

# Run Headless
docker run --rm -v $(pwd):/data:Z -w /data ghcr.io/eclipse-sumo/sumo:main sumo -c scenario.sumocfg

# Run GUI (Linux/WSLg)
docker run --rm \
  -e DISPLAY=$DISPLAY \
  -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
  -v $(pwd):/data:Z \
  -w /data \
  ghcr.io/eclipse-sumo/sumo:main \
  sumo-gui -c scenario.sumocfg
```

### Documentation Maintenance
Update docs after each session using the docs/AGENT_PROMPTS/BOOKKEPEER_PROMPT

