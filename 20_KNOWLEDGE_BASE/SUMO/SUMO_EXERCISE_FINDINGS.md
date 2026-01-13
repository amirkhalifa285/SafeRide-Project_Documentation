# SUMO Exercise Findings & Data Structure Analysis

**Date:** December 18, 2025 (Updated: December 20, 2025)
**Author:** Amir Khalifa
**Purpose:** Document hands-on SUMO learning results for data engineering pipeline implementation

**🆕 SUMO Version Upgrade (December 20, 2025):**
- **Upgraded to:** SUMO 1.25.0+ (v1_25_0+0604)
- **Docker Image:** `ghcr.io/eclipse-sumo/sumo:main` (official)
- **Verified:** CSV output working on both Fedora and Windows WSL2
- **Key Benefit:** Native CSV output instead of XML parsing

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Exercise 1: Basic Convoy Simulation](#exercise-1-basic-convoy-simulation)
3. [Exercise 2: SSM Emergency Braking](#exercise-2-ssm-emergency-braking)
4. [SSM Output Analysis](#ssm-output-analysis)
5. [FCD Output Analysis](#fcd-output-analysis)
6. [Hazard Classification Mapping](#hazard-classification-mapping)
7. [Data Pipeline Implications](#data-pipeline-implications)
8. [Questions for Professors](#questions-for-professors)

---

## Executive Summary

Successfully validated SUMO's capability to:
- Simulate realistic 3-vehicle convoy behavior
- Automatically detect safety conflicts via SSM device
- Generate ground truth hazard labels (TTC/DRAC values)
- Export training data at 10 Hz (matches ESP32 hardware)

**Key Finding:** SSM provides **automatic, objective hazard classification** without manual labeling - this is the ground truth for RL training.

---

## Exercise 1: Basic Convoy Simulation

### Setup
- **Network:** 4-segment straight road (~800m)
- **Vehicles:**
  - V003 (red) - Lead
  - V002 (green) - Middle
  - V001 (blue) - Ego/follower
- **Speed:** 13 m/s (~47 km/h)
- **Spacing:** 30m initial gaps

### Observations
- SUMO's Krauss car-following model maintained realistic gaps automatically
- Vehicles accelerated/decelerated smoothly to maintain safe distance
- Convoy formation stable throughout 120-second simulation

### Files Generated
- `network.net.xml` - Road geometry
- `vehicles.rou.xml` - Vehicle definitions and routes
- `scenario.sumocfg` - Simulation configuration

---

## Exercise 2: SSM Emergency Braking

### Scenario Design

**Network:**
- 6-segment straight road (~3000m)
- Single lane, 20 m/s speed limit

**Vehicles with SSM Enabled:**
```xml
<vType id="car_with_ssm" accel="2.6" decel="4.5" sigma="0.5"
       length="5" maxSpeed="20" tau="1.0">
    <param key="has.ssm.device" value="true"/>
    <param key="device.ssm.measures" value="TTC DRAC PET"/>
    <param key="device.ssm.thresholds" value="4.0 3.4 2.0"/>
    <param key="device.ssm.range" value="100.0"/>
</vType>
```

**Emergency Event:**
- V003 (lead) programmed to stop at edge C0D0, position 200m
- Stop duration: 5 seconds
- Trigger: Sudden deceleration from 18 m/s to 0 m/s

**Initial Conditions:**
- V003: departPos=100m (front)
- V002: departPos=60m (middle, 40m behind V003)
- V001: departPos=20m (rear, 40m behind V002)
- All vehicles: departSpeed=18 m/s

---

## Exercise 3: Import Real Road Network from OpenStreetMap

**Date Completed:** December 19, 2025

### Setup
- **Network Source:** OpenStreetMap export (real-world road geometry)
- **Location:** Downloaded custom area via OpenStreetMap export
- **Road Type:** 4-lane divided highway with curves (2 lanes per direction)
- **Network Conversion:** `netconvert --osm-files map.osm -o network.net.xml`

### Key Learnings: Edge IDs vs Lane IDs

**Critical Understanding:**
- **Edge** = Complete road segment between two junctions (what routes use)
- **Lane** = Individual lane within an edge (internal SUMO representation)

**Example from network:**
```
Lane ID:  636924784#1_0  (lane 0 of edge #1)
          636924784#1_1  (lane 1 of edge #1)
Edge ID:  636924784#1    ← Use this for routes! ✓
```

**Common mistake:** Using lane IDs (e.g., `636924784#1_0`) in route definitions causes "edge not found" errors.

### Understanding departPos Parameter

**Positive values:** Distance from START of edge
- `departPos="0"` → At start (0m)
- `departPos="60"` → 60m from start

**Negative values:** Distance from END of edge
- `departPos="-30"` → 30m before end
- `departPos="-60"` → 60m before end

**Convoy Formation (Correct):**
```xml
<vehicle id="V003" departPos="60"/>  <!-- Lead: 60m from start -->
<vehicle id="V002" departPos="30"/>  <!-- Middle: 30m from start -->
<vehicle id="V001" departPos="0"/>   <!-- Ego: 0m (at start) -->
```

This creates proper convoy order: V001 (rear) → V002 (middle) → V003 (lead) with 30m gaps.

### Files Generated
- `map.osm` - OpenStreetMap export (real road geometry)
- `network.net.xml` - SUMO network from OSM (160KB)
- `vehicles.rou.xml` - 3-vehicle convoy route
- `scenario.sumocfg` - Simulation configuration

### Observations
- Real-world roads have **curves** (unlike Exercise 1 straight roads)
- **Divided highways** have separate edges for each direction (positive/negative edge IDs)
- **Intersections** create edge discontinuities - must find connected path
- SUMO GUI's **"Select reachable"** feature helps identify valid routes

### Route Configuration Used
```xml
<route id="convoy_route" edges="636924784#1 636924784#0"/>
```

**Edge order matters:** Routes must follow connected edges in correct direction.

### Challenges Encountered & Solutions

**Challenge 1: Wrong file version in container**
- **Issue:** Volume mount showed cached old file
- **Solution:** Exit container, verify file in WSL, restart container
- **Lesson:** Always check file timestamps match between WSL and container

**Challenge 2: Network file not updated**
- **Issue:** Old `network.net.xml` from previous map
- **Solution:** Delete old network, regenerate with `netconvert`
- **Lesson:** Network file must be regenerated after updating map.osm

**Challenge 3: Using lane IDs instead of edge IDs**
- **Issue:** `Error: The edge '636924784#1_0' within the route is not known`
- **Solution:** Remove `_0`, `_1` suffixes - use edge ID only
- **Lesson:** Routes use edges, not lanes

**Challenge 4: Disconnected edges**
- **Issue:** `No connection between edge '636924784#1' and edge '636924784#1'`
- **Solution:** Used "Select reachable" to find actually connected edges
- **Lesson:** Cannot assume edges are connected without verifying

**Challenge 5: Convoy in wrong order**
- **Issue:** Initial config used negative `departPos` incorrectly
- **Solution:** Changed to positive values with V003 furthest, V001 closest to start
- **Lesson:** Understand departPos coordinate system before placing vehicles

### Success Criteria Met
✅ Successfully imported real-world OSM road network
✅ Converted OSM to SUMO network with netconvert
✅ Created 3-vehicle convoy on curved highway
✅ Vehicles followed realistic road geometry
✅ Convoy maintained formation through curves

### Simulation Results
- **Duration:** 120 seconds
- **Vehicles:** V003 (red, lead), V002 (green, middle), V001 (blue, ego)
- **Behavior:** SUMO Krauss car-following model maintained safe gaps on curved road
- **Road complexity:** More realistic than straight-road exercises (curves affect dynamics)

### Next Steps (Exercise 4-5)
- **Exercise 4:** Parameter variation (speed, spacing, decel) for manual augmentation
- **Exercise 5:** TraCI dynamic control to inject emergency braking events

---

## SSM Output Analysis

### Raw SSM Data Extract

```xml
<SSMLog>
    <!-- MOST CRITICAL: V002 (middle) following V003 (lead) -->
    <conflict begin="0.00" end="132.50" ego="V002" foe="V003">
        <minTTC time="61.90" position="1195.00,-1.60" type="2" value="1.91" speed="4.99"/>
        <maxDRAC time="57.70" position="1195.00,-1.60" type="2" value="3.02" speed="18.44"/>
        <PET time="NA" position="NA" type="NA" value="NA" speed="NA"/>
    </conflict>

    <!-- V001 (ego) following V003 (lead) -->
    <conflict begin="56.80" end="122.60" ego="V001" foe="V003">
        <minTTC time="61.90" position="1195.00,-1.60" type="2" value="3.61" speed="7.93"/>
        <maxDRAC time="57.70" position="1195.00,-1.60" type="2" value="2.10" speed="18.65"/>
        <PET time="NA" position="NA" type="NA" value="NA" speed="NA"/>
    </conflict>

    <!-- V001 (ego) following V002 (middle) -->
    <conflict begin="0.00" end="137.60" ego="V001" foe="V002">
        <minTTC time="62.70" position="1183.59,-1.60" type="2" value="3.68" speed="6.34"/>
        <maxDRAC time="62.70" position="1183.59,-1.60" type="2" value="0.43" speed="6.34"/>
        <PET time="NA" position="NA" type="NA" value="NA" speed="NA"/>
    </conflict>
</SSMLog>
```

### Key SSM Metrics Explained

| Metric | Full Name | What It Measures | Critical Value |
|--------|-----------|------------------|----------------|
| **minTTC** | Minimum Time to Collision | Lowest TTC value during conflict (seconds until impact if no action taken) | **1.91s** (V002 vs V003) |
| **maxDRAC** | Maximum Deceleration Rate to Avoid Crash | Hardest braking required to prevent collision (m/s²) | **3.02 m/s²** (V002 vs V003) |
| **PET** | Post-Encroachment Time | Time gap between vehicles crossing same point | Not applicable (same-lane following) |

### Conflict Timeline

**Time 57.7s:** Maximum DRAC event
- V003 begins emergency braking
- V002 traveling at 18.44 m/s, requires 3.02 m/s² deceleration

**Time 61.9s:** Minimum TTC event (MOST CRITICAL MOMENT)
- V002 vs V003: **TTC = 1.91 seconds** (WARNING level!)
- V002 speed reduced to 4.99 m/s (still decelerating)
- V001 vs V003: TTC = 3.61 seconds (CAUTION level)
- V001 speed 7.93 m/s

**Time 62.7s:** V001 vs V002 closest approach
- TTC = 3.68s (CAUTION level)
- DRAC = 0.43 m/s² (very comfortable)

---

## FCD Output Analysis

### Data Structure

```xml
<fcd-export>
    <timestep time="0.00">
        <vehicle id="V001" x="20.00" y="-1.60" angle="90.00" type="car_with_ssm"
                 speed="18.00" pos="20.00" lane="A0B0_0" slope="0.00" acceleration="0.00"/>
        <vehicle id="V002" x="60.00" y="-1.60" angle="90.00" type="car_with_ssm"
                 speed="18.00" pos="60.00" lane="A0B0_0" slope="0.00" acceleration="0.00"/>
        <vehicle id="V003" x="100.00" y="-1.60" angle="90.00" type="car_with_ssm"
                 speed="18.00" pos="100.00" lane="A0B0_0" slope="0.00" acceleration="0.00"/>
    </timestep>

    <timestep time="1.20">
        <vehicle id="V001" x="43.15" y="-1.60" angle="90.00" type="car_with_ssm"
                 speed="19.98" pos="43.15" lane="A0B0_0" slope="0.00" acceleration="0.25"/>
        <vehicle id="V002" x="82.33" y="-1.60" angle="90.00" type="car_with_ssm"
                 speed="18.74" pos="82.33" lane="A0B0_0" slope="0.00" acceleration="0.61"/>
        <vehicle id="V003" x="123.03" y="-1.60" angle="90.00" type="car_with_ssm"
                 speed="19.95" pos="123.03" lane="A0B0_0" slope="0.00" acceleration="-0.10"/>
    </timestep>
</fcd-export>
```

### Available Fields per Timestep

| Field | Description | Unit | RoadSense Equivalent |
|-------|-------------|------|---------------------|
| `time` | Simulation time | seconds | timestamp_ms (× 1000) |
| `id` | Vehicle ID | string | V001/V002/V003 |
| `x`, `y` | Cartesian position | meters | Can convert to lat/lon with `--fcd-output.geo true` |
| `angle` | Heading | degrees | v002_heading, v003_heading |
| `speed` | Velocity | m/s | v002_speed, v003_speed, ego_speed |
| `pos` | Position on current lane | meters | Internal SUMO coordinate |
| `lane` | Current lane ID | string | Lane tracking |
| `slope` | Road gradient | degrees | Not used in flat scenarios |
| `acceleration` | Longitudinal acceleration | m/s² | v002_accel_x, v003_accel_x (projected) |

### Sampling Rate

- **Configuration:** `<step-length value="0.1"/>`
- **Output:** 10 Hz (10 samples/second)
- **Matches:** ESP32 hardware logging rate (10 Hz)

---

## Hazard Classification Mapping

### Proposed Multi-Class Labels (from SUMO_RESEARCH_AND_SETUP_GUIDE.md)

| Class | Label | TTC Threshold | DRAC Threshold | Observed in Exercise 2 |
|-------|-------|---------------|----------------|------------------------|
| 0 | `SAFE` | TTC > 4.0s | DRAC < 2.0 m/s² | Early/late in simulation |
| 1 | `CAUTION` | 2.0s < TTC ≤ 4.0s | 2.0 ≤ DRAC < 3.4 m/s² | **V001 vs V003: TTC=3.61s** |
| 2 | `WARNING` | 1.5s < TTC ≤ 2.0s | 3.4 ≤ DRAC < 5.0 m/s² | **V002 vs V003: TTC=1.91s, DRAC=3.02** |
| 3 | `CRITICAL` | TTC ≤ 1.5s | DRAC ≥ 5.0 m/s² | Not triggered (collision avoided) |

### Actual Classification from Exercise 2

**Time 61.9s (Critical Moment):**

| Vehicle Pair | minTTC | maxDRAC | Proposed Label | Justification |
|--------------|--------|---------|----------------|---------------|
| V002 → V003 | 1.91s | 3.02 m/s² | **WARNING** | TTC in WARNING range (1.5-2.0s), DRAC just below WARNING threshold |
| V001 → V003 | 3.61s | 2.10 m/s² | **CAUTION** | TTC in CAUTION range (2.0-4.0s), DRAC in CAUTION range |
| V001 → V002 | 3.68s | 0.43 m/s² | **CAUTION** | TTC in CAUTION range, DRAC very low (safe) |

### Edge Case: Borderline Classifications

**V002 vs V003 DRAC = 3.02 m/s²**
- Below 3.4 m/s² threshold for WARNING
- But TTC = 1.91s clearly indicates WARNING
- **Recommendation:** Use **TTC as primary classifier**, DRAC as secondary validation

---

## Data Pipeline Implications

### Phase 5: FCD → Training CSV Conversion (Updated)

**Input Files:**
1. `fcd_output.xml` - Vehicle states at 10 Hz
2. `ssm_output.xml` - Conflict events with TTC/DRAC

**Output Format:** RoadSense training CSV

```csv
timestamp_ms,ego_speed,
v002_lat,v002_lon,v002_speed,v002_heading,v002_accel_x,v002_accel_y,v002_accel_z,v002_gyro_x,v002_gyro_y,v002_gyro_z,v002_age_ms,
v003_lat,v003_lon,v003_speed,v003_heading,v003_accel_x,v003_accel_y,v003_accel_z,v003_gyro_x,v003_gyro_y,v003_gyro_z,v003_age_ms,
hazard_class
```

**New Column:** `hazard_class` (0=SAFE, 1=CAUTION, 2=WARNING, 3=CRITICAL)

### Conversion Logic

**For each timestep in FCD:**

1. **Extract V001 (ego) state:**
   - `ego_speed` ← FCD `speed` for V001

2. **Extract V002 state:**
   - `v002_lat`, `v002_lon` ← FCD `x,y` (convert to lat/lon or use `--fcd-output.geo true`)
   - `v002_speed` ← FCD `speed` for V002
   - `v002_heading` ← FCD `angle` for V002
   - `v002_accel_x` ← FCD `acceleration` for V002 (longitudinal)
   - `v002_accel_y`, `v002_accel_z` ← Calculate from heading changes (gyro integration)
   - `v002_gyro_z` ← Δ(heading) / Δt (yaw rate)
   - `v002_age_ms` ← 0 (assuming perfect communication in simulation)

3. **Extract V003 state:** (same as V002)

4. **Determine hazard_class:**
   - Query SSM output for conflicts involving V001 at current timestamp
   - If minTTC found:
     - TTC ≤ 1.5s → `hazard_class = 3` (CRITICAL)
     - 1.5s < TTC ≤ 2.0s → `hazard_class = 2` (WARNING)
     - 2.0s < TTC ≤ 4.0s → `hazard_class = 1` (CAUTION)
     - TTC > 4.0s → `hazard_class = 0` (SAFE)
   - If no conflict at timestamp → `hazard_class = 0` (SAFE)

### Challenges Identified

1. **SSM Conflict Duration:**
   - SSM logs conflicts over time ranges (e.g., `begin="0.00" end="132.50"`)
   - Need to interpolate TTC/DRAC values between logged events
   - Simplification: Label entire conflict duration with worst-case (minTTC) class

2. **Multiple Simultaneous Conflicts:**
   - V001 may have conflicts with both V002 and V003 at same time
   - Resolution: Use **most severe** (lowest TTC) as hazard_class

3. **Gyroscope Simulation:**
   - FCD doesn't provide angular velocity directly
   - Calculate from heading changes: `gyro_z = (heading[t] - heading[t-1]) / 0.1`
   - Lateral acceleration from curvature (not applicable in straight road)

4. **Message Age Simulation:**
   - FCD assumes perfect communication (age_ms = 0)
   - Phase 6 post-processing adds realistic ESP-NOW delays (10-50ms + jitter)

---

## ✅ RESOLVED: Professor Feedback on SSM Role (December 18, 2025)

### Original Question: SSM as Ground Truth for Hazard Classification?

**Our initial understanding (INCORRECT):**
> "SSM metrics (TTC/DRAC) will serve as **training labels** for our RL model - label every timestep as SAFE/CAUTION/WARNING/CRITICAL based on SSM values."

### Professor's Clarification:

**Professor Resh's Response:**
> "SSM seems to be parallel to RL rather than part of it, the idea behind RL is to let the AI model learn what to do by itself rather than by functional calculations. so I think it may be used to **confirm AI results not be part of the learning algorithm**."

**Follow-up Question:** Should we implement our own SSM from vehicle positions/speeds?

**Professor's Answer:**
> "you should concentrate on RL and then inference using the trained model, you can use the sumo SSM module for **verification purposes** do not develop your own I see no need for that"

### CORRECTED APPROACH

**SSM Role:** ✅ **Validation tool ONLY** - NOT training labels

**True RL Training:**
- **Agent learns from:** Reward function based on distance, collision outcomes, comfort
- **NOT from:** Pre-calculated SSM hazard labels
- **Example reward:** -100 for collision, +1 for safe distance, -10 for harsh braking

**SSM Usage (Validation Phase):**
- Run trained model on test scenarios
- Compare model decisions against SUMO SSM calculations
- Verify model learned safe behaviors (e.g., model brakes when TTC < 1.5s)

**Key Insight:** RL agent discovers safety strategies through trial-and-error (maximizing rewards), SSM confirms the learned policy aligns with objective safety metrics.

**2. TTC vs DRAC Priority**

> "In our test scenario, we observed a conflict where TTC=1.91s (WARNING range) but DRAC=3.02 m/s² (below WARNING threshold of 3.4). Should we prioritize **TTC as the primary classifier** with DRAC as validation, or use a combined scoring function?"

**3. Multi-Class vs Binary Classification**

> "We're planning **4-class hazard labels** (SAFE/CAUTION/WARNING/CRITICAL) based on SSM thresholds. Should we start with binary (HAZARD/NO_HAZARD) for initial RL training and expand later, or implement multi-class from the beginning?"

**4. Conflict Duration Handling**

> "SSM logs conflicts over time ranges (e.g., 60 seconds). Should we label the **entire conflict duration** with the worst-case (minTTC) class, or interpolate TTC values frame-by-frame for more granular labels?"

**5. Ego Vehicle Perspective**

> "When V001 (ego) has simultaneous conflicts with both V002 and V003, should we label the timestep with:
> - (A) Most severe conflict (lowest TTC), or
> - (B) Immediate follower only (V001→V002), or
> - (C) Multi-target labeling (separate columns for each conflict)?"

---

## Next Steps

### Immediate (RL Training Environment)
1. ✅ **Exercise 3 COMPLETE** - Real-world OSM road network import successful (December 19, 2025)
2. **Exercise 4 (Next)** - Parameter variation (speed, spacing, decel) for manual augmentation
3. **Exercise 5** - TraCI dynamic control to inject emergency braking events
4. **Design reward function** - distance-based safety + comfort penalties (already defined in CLAUDE.md)
5. **Implement FCD → training CSV converter** - extract vehicle positions/speeds for RL state observations
6. **Set up RL training pipeline** - stable-baselines3 PPO/DQN with SUMO environment

### Validation (After Training)
1. **Collect 3-4 real ESP32 convoy scenarios** - for final model validation
2. **Compare model decisions vs. SSM** - verify trained policy aligns with TTC/DRAC safety metrics
3. **Measure performance** - collision avoidance rate, false alarm rate, comfort metrics

### SUMO Learning Progress
- ✅ Exercise 1: Basic convoy simulation (straight road)
- ✅ Exercise 2: SSM emergency braking scenario
- ✅ Exercise 3: Import real road networks from OpenStreetMap (December 19, 2025)
- ⏳ Exercise 4: Parameter variation (manual augmentation)
- ⏳ Exercise 5: TraCI dynamic control (inject emergency events)

---

## References

- **SUMO SSM Device Docs:** https://sumo.dlr.de/docs/Simulation/Output/SSM_Device.html
- **FHWA SSAM:** Surrogate Safety Assessment Model (TTC/PET thresholds)
- **Project Docs:** `SUMO_RESEARCH_AND_SETUP_GUIDE.md`, `CLAUDE.md`

---

**Document Version:** 1.2
**Last Updated:** December 20, 2025
**Status:** SUMO 1.25.0+ upgrade complete | CSV output verified | Exercise 3 complete - OSM import successful | Proceeding to Exercise 4
