# SUMO Research & Setup Guide for RoadSense V2V

**Document Purpose:** Provide comprehensive context for AI agents assisting with SUMO-based data augmentation pipeline implementation for the RoadSense V2V project.

**Created:** December 17, 2025  
**Author:** Amir Khalifa (with Claude AI research assistance)  
**Target Audience:** AI agents with knowledge of RoadSense architecture, ML team  
**Related Documents:** `ML_AUGMENTATION_PIPELINE.md`, `DATA_AUGMENTATION_WORKFLOW.md`, `CLAUDE.md`

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Key Decisions Made](#key-decisions-made)
3. [Understanding Surrogate Safety Measures (SSM)](#understanding-surrogate-safety-measures-ssm)
4. [Hazard Classification Strategy](#hazard-classification-strategy)
5. [How SUMO Fits Into RoadSense](#how-sumo-fits-into-roadsense)
6. [Complete Data Pipeline Architecture](#complete-data-pipeline-architecture)
7. [SUMO Technical Reference](#sumo-technical-reference)
8. [Development Environment Setup](#development-environment-setup)
9. [Getting Started: Mock Pipeline Exercises](#getting-started-mock-pipeline-exercises)
10. [Questions for Professors](#questions-for-professors)
11. [References and Resources](#references-and-resources)

---

## Executive Summary

### Project Context

RoadSense V2V is a vehicle-to-vehicle hazard detection system using ESP32 microcontrollers with embedded ML. The system requires training data that includes both normal driving and hazard scenarios. Since real hazard scenarios cannot be safely collected, we use SUMO (Simulation of Urban Mobility) to:

1. **Augment** real-world driving data with synthetic variations
2. **Generate** hazard scenarios that would be unsafe to collect in reality
3. **Automatically label** training data using Surrogate Safety Measures (SSM)

### Key Architectural Decision

**SUMO-only pipeline (no Veins/OMNeT++)** — Vehicle dynamics are simulated in SUMO, network effects (message delay, packet loss) are added via Python post-processing. This avoids the complexity of full network simulation while achieving equivalent ML training results.

### Target Output

From 4 real-world driving scenarios, generate 40+ augmented scenarios with automatic hazard labels, suitable for training a Reinforcement Learning agent that detects hazards and warns drivers.

---

## Key Decisions Made

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Hazard labeling method | **SSM-based automatic labeling** | Objective, consistent, scalable. Manual button-press labeling rejected as impractical. |
| Classification type | **Multi-class** (4 levels) | Allows nuanced warnings: safe → caution → warning → critical |
| Simulation approach | **SUMO-only** | Avoids Veins/OMNeT++ complexity; network effects added in Python |
| GPS sampling rate | **10 Hz** | Matches hardware capability, standard for automotive applications |
| Lane change scenarios | **Excluded** | Per professor guidance, focus on convoy and intersection scenarios |
| SUMO version | **1.25.0+** | Official Docker image `ghcr.io/eclipse-sumo/sumo:main` (verified Dec 20, 2025) |

---

## Understanding Surrogate Safety Measures (SSM)

### What is SSM?

Surrogate Safety Measures are mathematical metrics that quantify "how close was that to being a crash?" without requiring an actual collision. They are the industry standard for traffic safety research and provide **automatic ground truth labels** for ML training.

### Core SSM Metrics

| Metric | Full Name | What it Measures | Formula (Conceptual) |
|--------|-----------|------------------|---------------------|
| **TTC** | Time to Collision | Seconds until impact if trajectories unchanged | `Distance ÷ Closing_Speed` |
| **DRAC** | Deceleration Rate to Avoid Crash | Required braking intensity to prevent collision | `(v_follower² - v_leader²) ÷ (2 × gap)` |
| **PET** | Post-Encroachment Time | Time gap between vehicles crossing same point | `t_arrival_second - t_departure_first` |

### Why SSM is Critical for RoadSense

1. **Defines what "hazard" means mathematically** — No subjective human judgment
2. **Enables automatic labeling** — SUMO calculates SSM for every timestep
3. **Provides RL reward signal** — Agent learns from objective safety metrics
4. **Scales to any dataset size** — Works for 4 scenarios or 4000 scenarios

### SSM Thresholds from Research Literature

**TTC Thresholds (FHWA Standard):**
| TTC Value | Classification | Interpretation |
|-----------|----------------|----------------|
| TTC > 4.0s | Safe | Normal driving |
| 2.0s < TTC ≤ 4.0s | Elevated risk | Caution zone |
| 1.5s < TTC ≤ 2.0s | Serious conflict | Warning required |
| TTC ≤ 1.5s | Severe conflict | Emergency/near-crash |

**DRAC Thresholds:**
| DRAC Value | Classification |
|------------|----------------|
| DRAC < 3.4 m/s² | Comfortable deceleration |
| 3.4 ≤ DRAC < 6.0 m/s² | Uncomfortable braking |
| DRAC ≥ 6.0 m/s² | Emergency braking |

---

## Hazard Classification Strategy

### Multi-Class Hazard Labels (Recommended)

Based on research discussion, RoadSense will use a 4-class hazard classification:

| Class | Label | TTC Threshold | DRAC Threshold | System Action |
|-------|-------|---------------|----------------|---------------|
| 0 | `SAFE` | TTC > 4.0s | DRAC < 2.0 m/s² | No alert |
| 1 | `CAUTION` | 2.0s < TTC ≤ 4.0s | 2.0 ≤ DRAC < 3.4 m/s² | Soft advisory (visual) |
| 2 | `WARNING` | 1.5s < TTC ≤ 2.0s | 3.4 ≤ DRAC < 5.0 m/s² | Audible warning |
| 3 | `CRITICAL` | TTC ≤ 1.5s | DRAC ≥ 5.0 m/s² | Emergency alert |

**Note:** Final thresholds should be confirmed with professors. The values above are based on traffic safety research standards.

### How Labels Connect to RL Training

```
┌─────────────────────────────────────────────────────────────────┐
│                    RL REWARD FUNCTION                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  State: [ego_speed, v002_dist, v002_speed, v002_accel, ...]    │
│                                                                 │
│  Action: [NO_ALERT, SOFT_WARNING, HARD_WARNING, EMERGENCY]      │
│                                                                 │
│  Reward:                                                        │
│    +1.0  Correct warning level matches hazard class             │
│    -0.5  Warning too early (false alarm / driver annoyance)     │
│    -2.0  Warning too late or missed (safety failure)            │
│    -0.1  Any warning when SAFE (nuisance penalty)               │
│                                                                 │
│  SSM provides the ground truth hazard class for reward calc     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## How SUMO Fits Into RoadSense

### SUMO's Role in the Project

| Capability | How RoadSense Uses It |
|------------|----------------------|
| **Microscopic traffic simulation** | Simulates individual vehicle behavior with realistic car-following models |
| **FCD (Floating Car Data) output** | Provides position, speed, heading at each timestep → becomes sensor data |
| **SSM Device** | Calculates TTC, DRAC, PET automatically → becomes hazard labels |
| **TraCI (Traffic Control Interface)** | Python API to inject events (emergency braking, communication failures) |
| **OSM Import** | Convert real-world road networks for realistic scenarios |

### What SUMO Provides vs. What We Add

| Data Field | Source |
|------------|--------|
| Position (lat, lon) | SUMO FCD output with `--fcd-output.geo true` |
| Speed (m/s) | SUMO FCD output |
| Heading (degrees) | SUMO FCD output (angle) |
| Acceleration | SUMO FCD output with `--fcd-output.acceleration true` (v1.22.0+) |
| Gyroscope (angular velocity) | **Calculated** from heading changes between timesteps |
| V2V message age | **Python post-processing** (simulated network delay) |
| Packet loss | **Python post-processing** (probabilistic model) |
| Hazard label | SUMO SSM Device output |

### SUMO Does NOT Provide

- ESP-NOW protocol simulation (not needed — application layer is equivalent)
- Real IMU noise characteristics (can be added in post-processing if needed)
- Exact V2V message format (we convert FCD to our CSV format)

---

## Complete Data Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         PHASE 1: REAL DATA COLLECTION                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   3 × ESP32 vehicles driving in convoy (20 min sessions)                │
│   Logging: GPS (10Hz), IMU (10Hz), V2V messages                         │
│   Output: 4 CSV files (convoy scenarios, normal driving)                │
│                                                                         │
│   Note: No hazard scenarios collected (unsafe). Only normal driving.    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      PHASE 2: CSV → SUMO CONVERSION                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Input: Real CSV with GPS traces from all 3 vehicles                   │
│                                                                         │
│   Process:                                                              │
│   1. Extract GPS bounding box → Download OSM for that area              │
│   2. Convert OSM → SUMO network (.net.xml)                              │
│   3. Map GPS traces → SUMO routes (.rou.xml)                            │
│   4. Generate SUMO config (.sumocfg)                                    │
│                                                                         │
│   Output: Base SUMO scenario that replays real driving                  │
│                                                                         │
│   Tools: netconvert, osmWebWizard, custom Python scripts                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    PHASE 3: SUMO AUGMENTATION + SSM                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   For each base scenario, generate 10 variations:                       │
│                                                                         │
│   Augmentation Parameters:                                              │
│   ┌────────────────┬───────────────┬─────────────────────────────────┐  │
│   │ Parameter      │ Range         │ Effect                          │  │
│   ├────────────────┼───────────────┼─────────────────────────────────┤  │
│   │ speed_factor   │ 0.8 - 1.2     │ All vehicles ±20% speed         │  │
│   │ decel_factor   │ 0.7 - 1.3     │ Braking intensity ±30%          │  │
│   │ spacing_offset │ -10m to +10m  │ Initial vehicle gap             │  │
│   │ tau (headway)  │ 0.5s - 1.5s   │ Following distance preference   │  │
│   │ sigma          │ 0.3 - 0.7     │ Driver imperfection             │  │
│   └────────────────┴───────────────┴─────────────────────────────────┘  │
│                                                                         │
│   Each variation runs with SSM Device enabled:                          │
│   - TTC calculated every timestep                                       │
│   - DRAC calculated every timestep                                      │
│   - Conflicts logged when thresholds crossed                            │
│                                                                         │
│   Output: 10 variations × 4 scenarios = 40 augmented FCD + SSM files    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    PHASE 4: EDGE CASE INJECTION (TraCI)                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Use TraCI Python API to create scenarios that don't occur naturally:  │
│                                                                         │
│   Scenario Types:                                                       │
│   1. Emergency brake: Lead vehicle stops suddenly (speed → 0)           │
│   2. Chain reaction: Brake wave propagates through convoy               │
│   3. Intersection conflict: Cross-traffic timing creates near-miss      │
│                                                                         │
│   These create high-TTC/DRAC events for CRITICAL class training data    │
│                                                                         │
│   Output: 20+ additional edge case scenarios with SSM labels            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                  PHASE 5: FCD → TRAINING CSV CONVERSION                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Convert SUMO outputs to RoadSense training format:                    │
│                                                                         │
│   FCD XML → CSV columns:                                                │
│   - timestamp_ms (from SUMO time × 1000)                                │
│   - ego_speed (V001 speed)                                              │
│   - v002_lat, v002_lon, v002_speed, v002_heading                        │
│   - v002_accel_x, v002_accel_y, v002_accel_z (from SUMO accel + heading)│
│   - v002_gyro_x, v002_gyro_y, v002_gyro_z (from heading delta)          │
│   - v003_* (same for third vehicle)                                     │
│                                                                         │
│   SSM XML → Label column:                                               │
│   - hazard_class: 0=SAFE, 1=CAUTION, 2=WARNING, 3=CRITICAL              │
│   - Derived from TTC/DRAC values at each timestep                       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                 PHASE 6: NETWORK EFFECTS (Python Post-Processing)       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Add realistic V2V communication characteristics:                      │
│                                                                         │
│   Message Delay:                                                        │
│   - Base delay: 10-50ms (measured ESP-NOW characteristics)              │
│   - Jitter: Gaussian with σ=5ms                                         │
│   - Applied to v002_age_ms, v003_age_ms fields                          │
│                                                                         │
│   Packet Loss:                                                          │
│   - Base rate: 2% random loss                                           │
│   - Burst probability: 10% if previous packet lost                      │
│   - Effect: Increased age_ms (stale data)                               │
│                                                                         │
│   Output: Final training-ready CSV files                                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      PHASE 7: DATASET ASSEMBLY                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Final Dataset Composition:                                            │
│   - 4 real scenarios (normal driving, sparse hazards)                   │
│   - 40 augmented scenarios (varied parameters)                          │
│   - 20+ edge cases (emergency braking, chain reactions)                 │
│   ─────────────────────────────────────────────────                     │
│   Total: 64+ scenarios, ~384,000 training rows (at 10Hz, 10min each)    │
│                                                                         │
│   Class Distribution (estimated):                                       │
│   - SAFE: ~85%                                                          │
│   - CAUTION: ~10%                                                       │
│   - WARNING: ~4%                                                        │
│   - CRITICAL: ~1% (mostly from edge cases)                              │
│                                                                         │
│   Note: Class imbalance is expected. Handle with weighted loss or       │
│   oversampling during RL training.                                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## SUMO Technical Reference

### Key SUMO Components

| Component | Purpose | File Extension |
|-----------|---------|----------------|
| **Network** | Road layout (edges, lanes, junctions) | `.net.xml` |
| **Routes** | Vehicle definitions and paths | `.rou.xml` |
| **Config** | Simulation parameters | `.sumocfg` |
| **Additional** | Extra elements (detectors, SSM config) | `.add.xml` |
| **FCD Output** | Vehicle positions/speeds per timestep | `.xml` |
| **SSM Output** | Conflict events with TTC/DRAC | `.xml` |

### Essential SUMO Commands

```bash
# Convert OSM to SUMO network
netconvert --osm-files map.osm -o network.net.xml

# Run simulation (headless)
sumo -c scenario.sumocfg

# Run simulation (GUI)
sumo-gui -c scenario.sumocfg

# Run with FCD output
sumo -c scenario.sumocfg --fcd-output fcd.xml --fcd-output.geo true

# Run with FCD + SSM
sumo -c scenario.sumocfg \
    --fcd-output fcd.xml \
    --fcd-output.geo true \
    --fcd-output.acceleration true \
    --device.ssm.probability 1.0 \
    --device.ssm.file ssm_output.xml
```

### SUMO Configuration Template

```xml
<!-- scenario.sumocfg -->
<configuration>
    <input>
        <net-file value="network.net.xml"/>
        <route-files value="vehicles.rou.xml"/>
        <additional-files value="ssm_config.add.xml"/>
    </input>
    <time>
        <begin value="0"/>
        <end value="600"/>           <!-- 10 minutes -->
        <step-length value="0.1"/>   <!-- 10 Hz -->
    </time>
    <output>
        <fcd-output value="fcd_output.xml"/>
        <fcd-output.geo value="true"/>
        <fcd-output.acceleration value="true"/>
    </output>
</configuration>
```

### SSM Device Configuration

```xml
<!-- ssm_config.add.xml -->
<additional>
    <vType id="equipped" accel="2.6" decel="4.5" length="5" maxSpeed="50">
        <param key="has.ssm.device" value="true"/>
        <param key="device.ssm.measures" value="TTC DRAC PET"/>
        <param key="device.ssm.thresholds" value="4.0 3.4 2.0"/>
        <param key="device.ssm.range" value="100.0"/>
        <param key="device.ssm.file" value="ssm_output.xml"/>
    </vType>
</additional>
```

### Vehicle Route Template

```xml
<!-- vehicles.rou.xml -->
<routes>
    <vType id="car" accel="2.6" decel="4.5" sigma="0.5" 
           length="5" maxSpeed="50" tau="1.0"/>
    
    <route id="convoy_route" edges="edge1 edge2 edge3 edge4"/>
    
    <!-- V003: Lead vehicle -->
    <vehicle id="V003" type="car" depart="0" route="convoy_route" 
             departSpeed="15" departPos="0"/>
    
    <!-- V002: Middle vehicle -->
    <vehicle id="V002" type="car" depart="1" route="convoy_route" 
             departSpeed="15" departPos="-30"/>
    
    <!-- V001: Ego vehicle (rear) -->
    <vehicle id="V001" type="car" depart="2" route="convoy_route" 
             departSpeed="15" departPos="-60"/>
</routes>
```

### FCD Output Format (What You Get)

```xml
<fcd-export>
    <timestep time="10.0">
        <vehicle id="V001" x="234.56" y="567.89" lat="32.0853" lon="34.7818"
                 angle="45.2" speed="14.5" pos="100.0" slope="0.0"
                 acceleration="0.3"/>
        <vehicle id="V002" x="264.12" y="598.34" lat="32.0856" lon="34.7821"
                 angle="45.0" speed="14.8" pos="130.0" slope="0.0"
                 acceleration="-0.5"/>
        <vehicle id="V003" x="293.78" y="628.91" lat="32.0859" lon="34.7824"
                 angle="44.8" speed="12.1" pos="160.0" slope="0.0"
                 acceleration="-2.1"/>
    </timestep>
    <!-- ... more timesteps ... -->
</fcd-export>
```

### SSM Output Format (What You Get)

```xml
<SSMLog>
    <conflict begin="52.3" end="54.8" ego="V001" foe="V002">
        <conflictType value="FOLLOWING"/>
        <minTTC time="53.1" position="..." egoPosition="..." foePosition="..."
                value="0.87"/>
        <maxDRAC time="53.1" position="..." value="4.2"/>
    </conflict>
    <!-- ... more conflicts ... -->
</SSMLog>
```

---

## Development Environment Setup

### Current Setup

- **Host OS:** Windows with WSL2 (also tested on Fedora Linux)
- **Docker Image:** `ghcr.io/eclipse-sumo/sumo:main` (official SUMO project image)
- **SUMO Version:** 1.25.0+ (v1_25_0+0604 or newer)
- **GUI Requirement:** X11 forwarding for sumo-gui (WSLg on Windows 11, X server on Linux)
- **Python Dependencies:** `traci`, `sumolib` (standard versions preferred over `libtraci` for Docker compatibility)
- **✅ Verified:** CSV output working, tested December 20, 2025

**⚠️ Critical Note on TraCI vs. LibTraCI:**
For this project, we use the standard `traci` Python package, NOT `libtraci`.
- **`traci`**: Pure Python client. Communicates via TCP/IP sockets. Highly robust for connecting to SUMO running inside Docker containers or remote instances.
- **`libtraci`**: C++ bindings. Faster, but requires strict ABI compatibility with the installed SUMO libraries. Often fails in Docker/cross-platform setups due to version mismatches.
**Action:** Always use `pip install traci sumolib`. Do not mix with `libtraci` to avoid dependency hell.
**Reference:** [TraCI vs LibTraCI Documentation](https://sumo.dlr.de/docs/TraCI/Interfacing_TraCI_from_Python.html) - READ THIS thoroughly before debugging connection issues.

### WSL2 + Docker + GUI Setup

For running SUMO GUI from Docker on Windows/WSL2, the following setup is required:

**Option A: WSLg (Windows 11)**
Windows 11 has built-in WSLg that provides automatic GUI support. If running Windows 11, GUI apps should work automatically.

**Option B: X Server (Windows 10 or manual setup)**
1. Install an X server on Windows (VcXsrv or Xming)
2. Configure DISPLAY environment variable in WSL2
3. Pass DISPLAY to Docker container

**Docker Run Command for GUI:**
```bash
# From WSL2 terminal
docker run -it --rm \
    -e DISPLAY=$DISPLAY \
    -v /tmp/.X11-unix:/tmp/.X11-unix \
    -v $(pwd):/data:Z \
    -w /data \
    ghcr.io/eclipse-sumo/sumo:main \
    sumo-gui
```

**Verify SUMO Installation:**
```bash
docker run --rm ghcr.io/eclipse-sumo/sumo:main sumo --version
# Expected: Eclipse SUMO sumo v1_25_0+0604 or newer
```

---

## Getting Started: Mock Pipeline Exercises

### Exercise 1: Hello World — First SUMO Scenario

**Goal:** Create the simplest possible scenario with 3 vehicles on a straight road.

**Tasks:**
1. Generate a simple straight road network using netgenerate
2. Create a route file with 3 vehicles in convoy
3. Run the simulation and observe in GUI
4. Generate FCD output and examine the XML structure

**Learning Outcomes:**
- Understand SUMO file structure (net, rou, cfg)
- Observe car-following behavior
- See how vehicles maintain gaps automatically

### Exercise 2: Enable SSM and Observe Conflicts

**Goal:** Add SSM device to the convoy scenario and generate a near-miss.

**Tasks:**
1. Add SSM configuration to the scenario
2. Modify lead vehicle to brake suddenly (lower decel parameter or use stops)
3. Run simulation with SSM output enabled
4. Examine SSM output XML — find TTC and DRAC values

**Learning Outcomes:**
- Understand how SSM device works
- See TTC/DRAC values in action
- Connect SSM output to hazard labeling concept

### Exercise 3: Import Real Road Network from OSM

**Goal:** Create a scenario on actual roads (e.g., a section of Tel Aviv).

**Tasks:**
1. Use osmWebWizard to download a small area
2. Examine the generated network in sumo-gui
3. Create a simple route on the real roads
4. Run convoy scenario on real road geometry

**Learning Outcomes:**
- Understand OSM → SUMO workflow
- See how real road complexity affects simulation
- Prepare for using real GPS trace locations

### Exercise 4: Parameter Variation (Manual Augmentation)

**Goal:** Create variations of a base scenario by modifying parameters.

**Tasks:**
1. Start with the Exercise 2 scenario as base
2. Create 3 variations:
   - Variation A: Vehicles 20% faster
   - Variation B: Vehicles start 10m closer
   - Variation C: Lead vehicle brakes harder (lower decel)
3. Run all variations and compare SSM outputs

**Learning Outcomes:**
- Understand how parameters affect conflict severity
- Manual practice before automation
- See range of TTC/DRAC values achievable

### Exercise 5: TraCI Introduction — Dynamic Control

**Goal:** Use Python TraCI to inject an emergency braking event mid-simulation.

**Tasks:**
1. Write a simple TraCI Python script
2. Start simulation, let it run for 30 seconds
3. At t=30s, force lead vehicle to stop suddenly
4. Observe the chain reaction and SSM values

**Learning Outcomes:**
- Understand TraCI API basics
- See how to create controlled hazard scenarios
- Foundation for edge case generation phase

---

## Questions for Professors

The following questions emerged from research and should be confirmed before implementation:

### 1. SSM Threshold Confirmation
> "We propose using TTC < 1.5s as CRITICAL, 1.5-2.0s as WARNING, 2.0-4.0s as CAUTION. Do these thresholds align with your expectations for V2V warning system behavior?"

### 2. Multi-Class vs Binary
> "We're planning 4-class hazard labels (SAFE/CAUTION/WARNING/CRITICAL). Would you prefer binary (HAZARD/NO_HAZARD) for initial implementation, then expand later?"

### 3. Intersection Scenarios
> "Should we include intersection near-miss scenarios in training data, or focus exclusively on convoy/following scenarios for the B.Sc. scope?"

### 4. RL Reward Function
> "For the RL reward function, should the penalty for missing a hazard be significantly higher than the penalty for a false alarm? (Safety-critical systems typically weight misses more heavily)"

### 5. Validation Dataset
> "Should we hold out some real-world data (not augmented) purely for final validation, or use all real data for training/augmentation?"

---

## References and Resources

### SUMO Documentation
- Main Documentation: https://sumo.dlr.de/docs/index.html
- Tutorials Index: https://sumo.dlr.de/docs/Tutorials/index.html
- FCD Output: https://sumo.dlr.de/docs/Simulation/Output/FCDOutput.html
- SSM Device: https://sumo.dlr.de/docs/Simulation/Output/SSM_Device.html
- TraCI Python: https://sumo.dlr.de/docs/TraCI/Interfacing_TraCI_from_Python.html (Use `traci` package, avoid `libtraci` for Docker)
- OSM Import: https://sumo.dlr.de/docs/Networks/Import/OpenStreetMap.html
- Safety Documentation: https://sumo.dlr.de/docs/Simulation/Safety.html

### Traffic Safety Research
- FHWA SSAM (Surrogate Safety Assessment Model): Standard TTC/PET thresholds
- SAE J2735 BSM: V2V message format standard (project compliance target)

### RoadSense Project Documents
- `ML_AUGMENTATION_PIPELINE.md` — Detailed pipeline specification
- `DATA_AUGMENTATION_WORKFLOW.md` — Original workflow document
- `CLAUDE.md` — Project overview and architecture
- `DATALOGGER_MODIFICATION_PROMPT.md` — Ego-perspective logging format

### Docker Image
- Image: `ghcr.io/eclipse-sumo/sumo:main` (official SUMO project)
- Contains: SUMO 1.25.0+, Python 3, sumolib, traci, Parquet support
- Pull: `docker pull ghcr.io/eclipse-sumo/sumo:main`
- Benefits: Latest bug fixes, CSV/Parquet output, official support

---

## Document Changelog

| Date | Change | Author |
|------|--------|--------|
| 2025-12-17 | Initial document created from research session | Amir + Claude |
| 2025-12-20 | Updated to SUMO 1.25.0+, verified CSV output, updated Docker image references | Amir + Claude |

---

**End of Document**
