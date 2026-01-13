# CLAUDE.md

**Context**: Guidance for Claude Code (claude.ai/code) working on the RoadSense V2V project.
**Last Updated:** January 1, 2026

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
  - Emulates packet loss, latency, and jitter based on *measured hardware data*.
  - see `00_ARCHITECTURE/ESPNOW_EMULATOR_DESIGN.md`.
- **Machine Learning:**
  - **Approach:** Reinforcement Learning (PPO/DQN).
  - **Agent:** Runs on **V001 (Ego Vehicle)** only.
  - **Input:** Received V2V messages from neighbors + Ego Speed.
  - **Deployment:** TensorFlow Lite for Microcontrollers (Quantized INT8).

---

## 🛑 Critical Constraints & Truths

### The "Single-Agent" Architecture
**Protocol:**
1.  **V001 (Ego)** is the intelligent agent.
2.  **V002 & V003** are "dumb" broadcasters (Lead Vehicles).
3.  **Logging:** Only V001 logs data. It logs what it *sees* (the V2V packets from others).
4.  **Inference:** V001 calculates collision risk for *itself*.

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

