# RoadSense V2 Architecture: Minimal RL Observation Approach

**Document Type:** Architecture Decision Record + Implementation Plan
**Created:** December 26, 2025
**Author:** Amir Khalifa
**Status:** PARTIALLY SUPERSEDED
**Supersedes:** Previous sensor-synthesis-dependent approach

---

> ## SUPERSEDED NOTICE (January 13, 2026)
>
> **The observation space design in this document (fixed 11-dim) has been superseded by the Deep Sets architecture.**
>
> **See:** `DEEP_SETS_N_ELEMENT_ARCHITECTURE.md` for the correct approach.
>
> **What Changed:**
> - Fixed observation (11 features for 2 peers) replaced with variable-n peer handling
> - MlpPolicy replaced with custom DeepSetPolicy
> - Hardcoded V002/V003 replaced with dynamic peer list
>
> **What Remains Valid:**
> - ESP-NOW Characterization approach
> - Domain Randomization strategy
> - Reward function design
> - Sim2Real pipeline concept

---

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Big Picture Architecture](#2-big-picture-architecture)
3. [Component Diagram and Dataflow](#3-component-diagram-and-dataflow)
4. [Milestone Plan with Acceptance Tests](#4-milestone-plan-with-acceptance-tests)
5. [Risk and Mitigation Table](#5-risk-and-mitigation-table)
6. [Appendix: Decision Rationale](#appendix-decision-rationale)

---

## 1. Executive Summary

### The Problem We're Solving

RoadSense trains an RL agent to make safe driving decisions (Maintain/Caution/Brake/Emergency) based on V2V messages received from other vehicles. The challenge is **Sim2Real transfer**: SUMO simulation provides perfect data, but real ESP-NOW communication has:

- Latency (10-80ms variable)
- Packet loss (2-10%)
- Jitter (latency variance)
- Message staleness

Training on perfect data → model fails on real hardware.

### The Solution (Updated December 26, 2025)

**Three key decisions:**

1. **Minimal RL Observation Space** - The policy uses ONLY what's essential:
   - Peer vehicle positions, speeds, accelerations
   - Message age/staleness indicators
   - Ego vehicle speed
   - **NOT** gyro/mag (these are optional, not blocking)

2. **Python ESP-NOW Emulator** - Built from REAL measurements, not guesses:
   - Characterize actual ESP-NOW on ESP32 hardware (RTT-based)
   - Build emulator that matches measured distributions
   - Apply domain randomization (train WIDER than reality)

3. **SUMO-Only Simulation** - No Veins/OMNeT++ complexity:
   - SUMO provides vehicle dynamics ground truth
   - ESP-NOW emulator provides communication realism
   - Simpler = faster iteration = better results

### Why This Works

| What ML Sees | What ML Doesn't Need |
|--------------|---------------------|
| Peer position (lat/lon) | RF signal strength |
| Peer speed (m/s) | Gyroscope readings |
| Peer acceleration (m/s²) | Magnetometer readings |
| Message age (ms) | MAC layer details |
| Ego speed (m/s) | Channel contention |

The RL agent learns to make safe decisions under **communication uncertainty** (delayed/lost messages), not sensor-level noise.

---

## 2. Big Picture Architecture

### 2.1 Why We Don't Need Full OMNeT++/Veins ESP-NOW Emulation

**Previous Thinking (Wrong):**
> "We need to perfectly emulate ESP-NOW RF characteristics in Veins to train a realistic model."

**Correct Thinking:**
> "The RL policy only sees application-layer messages with timing metadata. It doesn't care HOW the message arrived, only WHEN and WHETHER it arrived."

The policy treats the network as a **stochastic pipe** with three properties:
1. **Latency** - How long until message arrives?
2. **Loss** - Did message arrive at all?
3. **Staleness** - How old is the data in the message?

A Python ESP-NOW emulator can model these properties based on real measurements. No need for full RF simulation.

### 2.2 How Sim2Real is Handled

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SIM2REAL STRATEGY                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  STEP 1: CHARACTERIZE (This Week)                                           │
│  ─────────────────────────────────                                          │
│  Real ESP32 Hardware → RTT Measurement Drive → emulator_params.json         │
│                                                                              │
│  STEP 2: EMULATE (Next Week)                                                │
│  ────────────────────────────                                               │
│  SUMO (perfect) → ESP-NOW Emulator (adds measured noise) → ConvoyEnv        │
│                                                                              │
│  STEP 3: TRAIN WITH DOMAIN RANDOMIZATION                                    │
│  ──────────────────────────────────────────                                 │
│  Train on WIDER range than measured:                                        │
│  - Measured latency: 10-50ms → Train on: 5-80ms                            │
│  - Measured loss: 2% → Train on: 0-15%                                      │
│  - Result: Model robust to variations                                       │
│                                                                              │
│  STEP 4: VALIDATE ON REAL HARDWARE                                          │
│  ────────────────────────────────────                                       │
│  Deploy on ESP32 → Real convoy drive → Compare sim vs real performance     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.3 Why Gyro/Mag Are NOT Required for Convoy RL Policy

**The Task:** Ego vehicle (V001) follows V002/V003 and must decide when to brake.

**What Matters for This Decision:**
- Where are the other vehicles? (position)
- How fast are they going? (speed)
- Are they braking? (acceleration)
- Is my information stale? (message age)
- How fast am I going? (ego speed)

**What Doesn't Matter:**
- Gyroscope readings - useful for detecting rotation, but convoy following is primarily longitudinal
- Magnetometer readings - useful for absolute heading, but relative position is what matters

**Key Insight:** The convoy RL task is a **1D following problem** (longitudinal dynamics). Gyro/mag add complexity without improving the decision quality for this specific scenario.

**Future Extensibility:** Gyro/mag remain in the V2VMessage schema for:
- Lane-change detection scenarios (future work)
- Intersection scenarios requiring heading (future work)
- Ablation studies comparing minimal vs full observation

---

## 3. Component Diagram and Dataflow

### 3.1 Training Pipeline

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         TRAINING PIPELINE                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────────┐                                                        │
│  │   SUMO 1.25.0+   │  FCD Output (10 Hz)                                   │
│  │  (Docker Image)  │  ─────────────────→                                   │
│  │                  │  - vehicle_id                                         │
│  │  Scenarios:      │  - x, y position                                      │
│  │  - convoy_001    │  - speed (m/s)                                        │
│  │  - convoy_002    │  - angle (degrees)                                    │
│  │  - ...           │  - acceleration                                       │
│  └──────────────────┘                                                        │
│           │                                                                  │
│           ▼                                                                  │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                    ESP-NOW EMULATOR (Python)                          │   │
│  │                                                                       │   │
│  │  Input: Perfect SUMO state for V002, V003                            │   │
│  │                                                                       │   │
│  │  Processing:                                                          │   │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐   │   │
│  │  │ Latency Model   │  │ Packet Loss     │  │ Domain Randomization│   │   │
│  │  │                 │  │                 │  │                     │   │   │
│  │  │ base + distance │  │ Bernoulli(p)    │  │ Per-episode params  │   │   │
│  │  │ + jitter        │  │ + burst model   │  │ from wider ranges   │   │   │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────────┘   │   │
│  │                                                                       │   │
│  │  Output: "Received" messages with age_ms, or None if lost            │   │
│  │                                                                       │   │
│  │  Parameters from: emulator_params.json (measured from real ESP32)    │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│           │                                                                  │
│           ▼                                                                  │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                    CONVOY ENV (Gymnasium)                             │   │
│  │                                                                       │   │
│  │  Observation Space (MINIMAL - 11 features):                          │   │
│  │  ┌───────────────────────────────────────────────────────────────┐   │   │
│  │  │ ego_speed                                    (1 float)        │   │   │
│  │  │ v002: rel_x, rel_y, speed, accel, age_ms    (5 floats)        │   │   │
│  │  │ v003: rel_x, rel_y, speed, accel, age_ms    (5 floats)        │   │   │
│  │  │                                                               │   │   │
│  │  │ Total: 11 features                                            │   │   │
│  │  │ NO gyro, NO mag, NO full accel[3]                             │   │   │
│  │  └───────────────────────────────────────────────────────────────┘   │   │
│  │                                                                       │   │
│  │  Action Space: Discrete(4)                                           │   │
│  │  ┌───────────────────────────────────────────────────────────────┐   │   │
│  │  │ 0 = Maintain  │ 1 = Caution  │ 2 = Brake  │ 3 = Emergency     │   │   │
│  │  └───────────────────────────────────────────────────────────────┘   │   │
│  │                                                                       │   │
│  │  Reward Function (per timestep):                                     │   │
│  │  ┌───────────────────────────────────────────────────────────────┐   │   │
│  │  │ -100: Collision (distance < 5m)                               │   │   │
│  │  │  -5:  Unsafe proximity (distance < 15m)                       │   │   │
│  │  │  +1:  Safe following (20m < distance < 40m)                   │   │   │
│  │  │ -10:  Harsh braking (decel > 4.5 m/s²)                        │   │   │
│  │  │  -2:  Unnecessary alert (distance > 40m, action > 1)          │   │   │
│  │  └───────────────────────────────────────────────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│           │                                                                  │
│           ▼                                                                  │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                    RL TRAINING (stable-baselines3)                    │   │
│  │                                                                       │   │
│  │  Algorithm: PPO (Proximal Policy Optimization)                       │   │
│  │  Network: MLP [64, 64]                                               │   │
│  │  Training: 100,000+ timesteps                                        │   │
│  │                                                                       │   │
│  │  Output: policy.zip (PyTorch model)                                  │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│           │                                                                  │
│           ▼                                                                  │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                    TFLITE CONVERSION                                  │   │
│  │                                                                       │   │
│  │  PyTorch → ONNX → TensorFlow → TFLite (INT8 quantized)              │   │
│  │  Target: <100KB model, <50ms inference on ESP32                      │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Characterization Pipeline

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ESP-NOW CHARACTERIZATION PIPELINE                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────────┐         ┌──────────────────┐                          │
│  │   V001 (Sender)  │ ──────→ │  V002 (Reflector)│                          │
│  │                  │ ←────── │                  │                          │
│  │  - Send packet   │  Echo   │  - Receive       │                          │
│  │  - Record T1     │         │  - Echo ASAP     │                          │
│  │  - Record T2     │         │  - No processing │                          │
│  │  - RTT = T2-T1   │         │                  │                          │
│  │  - Log to SD     │         │                  │                          │
│  │  - Log GPS pos   │         │                  │                          │
│  └──────────────────┘         └──────────────────┘                          │
│           │                                                                  │
│           │ 20-30 minute drive (varying distances)                          │
│           ▼                                                                  │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                    SD CARD: rtt_data.csv                              │   │
│  │                                                                       │   │
│  │  sequence,send_time_ms,recv_time_ms,rtt_ms,lat,lon,lost              │   │
│  │  0,1000,1012,12,32.085,34.781,0                                       │   │
│  │  1,1100,1115,15,32.085,34.781,0                                       │   │
│  │  2,1200,-1,-1,32.086,34.782,1  ← lost packet                         │   │
│  │  ...                                                                  │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│           │                                                                  │
│           ▼                                                                  │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                    ANALYSIS SCRIPT (Python)                           │   │
│  │                                                                       │   │
│  │  1. Load CSV                                                          │   │
│  │  2. Calculate packet loss rate                                        │   │
│  │  3. Analyze RTT distribution (mean, std, percentiles)                │   │
│  │  4. Detect burst loss patterns                                        │   │
│  │  5. Correlate with GPS distance (calculated post-hoc)                │   │
│  │  6. Generate domain randomization ranges (1.5x measured)             │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│           │                                                                  │
│           ▼                                                                  │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                    emulator_params.json                               │   │
│  │                                                                       │   │
│  │  {                                                                    │   │
│  │    "latency": {                                                       │   │
│  │      "base_ms": 12,                                                   │   │
│  │      "distance_factor": 0.15,                                         │   │
│  │      "jitter_std_ms": 8                                               │   │
│  │    },                                                                 │   │
│  │    "packet_loss": {                                                   │   │
│  │      "base_rate": 0.02,                                               │   │
│  │      "distance_threshold_m": 80,                                      │   │
│  │      "high_loss_rate": 0.15                                           │   │
│  │    },                                                                 │   │
│  │    "domain_randomization": {                                          │   │
│  │      "latency_range_ms": [5, 80],                                     │   │
│  │      "loss_rate_range": [0.0, 0.20]                                   │   │
│  │    }                                                                  │   │
│  │  }                                                                    │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.3 Deployment Pipeline

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         DEPLOYMENT ON ESP32                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  V003 (Lead)              V002 (Middle)           V001 (Ego + ML)           │
│  ┌──────────┐             ┌──────────┐            ┌──────────────────┐      │
│  │ GPS      │             │ GPS      │            │ GPS              │      │
│  │ IMU      │             │ IMU      │            │ IMU              │      │
│  │          │             │          │            │ TFLite Model     │      │
│  │ Broadcast│ ──V2V────→  │ Broadcast│ ──V2V───→ │                  │      │
│  │ only     │             │ only     │            │ Receives V2V     │      │
│  │          │             │          │            │ Runs inference   │      │
│  │          │             │          │            │ Outputs action   │      │
│  └──────────┘             └──────────┘            └──────────────────┘      │
│                                                           │                  │
│                                                           ▼                  │
│                                               ┌──────────────────────┐      │
│                                               │ Action → Alert       │      │
│                                               │ 0: LED off           │      │
│                                               │ 1: LED slow blink    │      │
│                                               │ 2: LED fast blink    │      │
│                                               │ 3: LED solid + buzzer│      │
│                                               └──────────────────────┘      │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Milestone Plan with Acceptance Tests

### Phase 1: ESP-NOW Characterization (BLOCKING)

**Duration:** 3-4 days
**Owner:** Amir
**Dependencies:** None

| Milestone | Deliverable | Acceptance Test |
|-----------|-------------|-----------------|
| M1.1 | Sender firmware (`sender_main.cpp`) | Compiles with no errors/warnings. Sends 10 packets/sec visible in Serial monitor. |
| M1.2 | Reflector firmware (`reflector_main.cpp`) | Compiles with no errors/warnings. Echoes packets immediately (<1ms processing). |
| M1.3 | Bench test (1m) | 1000 packets sent, <1% loss, RTT 2-15ms, CSV logged to SD. |
| M1.4 | Drive test (20-30 min) | 15,000+ packets logged. GPS coordinates valid. File retrievable. |
| M1.5 | Analysis script | `emulator_params.json` generated. Latency distribution plotted. Loss rate calculated. |

**Acceptance Criteria for Phase 1 Complete:**
- [ ] `emulator_params.json` exists with measured values
- [ ] Latency mean is 10-50ms range (expected for ESP-NOW)
- [ ] Packet loss rate is 1-10% range
- [ ] Distance correlation shows increasing loss beyond ~80m

### Phase 2: ESP-NOW Emulator (Python)

**Duration:** 2-3 days
**Owner:** Amir
**Dependencies:** Phase 1 complete

| Milestone | Deliverable | Acceptance Test |
|-----------|-------------|-----------------|
| M2.1 | `ESPNOWEmulator` class | Unit tests pass. Latency distribution matches measured (KS test p > 0.05). |
| M2.2 | Domain randomization | Per-episode parameters vary within configured ranges. |
| M2.3 | Packet loss model | Loss rate matches measured overall rate (±2%). Burst detection if applicable. |
| M2.4 | Integration test | 1000 simulated transmissions produce expected loss/latency statistics. |

**Acceptance Criteria for Phase 2 Complete:**
- [ ] `espnow_emulator.py` imports and runs without errors
- [ ] Emulator latency distribution matches measured (visual inspection + KS test)
- [ ] Emulator loss rate matches measured (within ±2%)
- [ ] Domain randomization produces varied episodes

### Phase 3: SUMO ConvoyEnv

**Duration:** 3-4 days
**Owner:** Amir
**Dependencies:** Phase 2 complete

| Milestone | Deliverable | Acceptance Test |
|-----------|-------------|-----------------|
| M3.1 | SUMO scenario files | 3-vehicle convoy scenario runs in SUMO without errors. FCD output generated. |
| M3.2 | `ConvoyEnv` class | `env.reset()` and `env.step(action)` work. Observation shape = (11,). |
| M3.3 | Reward function | Collision episode returns total reward < -50. Safe following returns positive. |
| M3.4 | Episode termination | Episode ends on collision OR max_steps OR vehicle exits. |
| M3.5 | Integration test | 100 random episodes complete without errors. Mean episode length > 50 steps. |

**Acceptance Criteria for Phase 3 Complete:**
- [ ] `gymnasium.make('RoadSense-Convoy-v0')` works
- [ ] Random policy runs 100 episodes without crashes
- [ ] Observations include message age_ms that varies over episode
- [ ] Some episodes show packet loss effects (age_ms jumps)

### Phase 4: RL Training

**Duration:** 3-5 days
**Owner:** Amir
**Dependencies:** Phase 3 complete

| Milestone | Deliverable | Acceptance Test |
|-----------|-------------|-----------------|
| M4.1 | Training script | PPO trains for 10,000 steps without errors. Learning curve shows improvement. |
| M4.2 | Hyperparameter search | 3+ hyperparameter configs tested. Best config documented. |
| M4.3 | Full training run | 100,000+ steps. Mean reward improves from random baseline. |
| M4.4 | Policy evaluation | Trained policy achieves <5% collision rate (vs ~20% random). |
| M4.5 | Model checkpoint | `policy.zip` saved. Loads and runs inference correctly. |

**Acceptance Criteria for Phase 4 Complete:**
- [ ] Trained model exists (`policy.zip`)
- [ ] Collision rate < 10% on evaluation episodes
- [ ] Policy takes sensible actions (brakes when distance decreases)
- [ ] Training logs show monotonic reward improvement

### Phase 5: TFLite Conversion & ESP32 Deployment

**Duration:** 5-7 days (high risk)
**Owner:** Amir
**Dependencies:** Phase 4 complete

| Milestone | Deliverable | Acceptance Test |
|-----------|-------------|-----------------|
| M5.1 | PyTorch → ONNX | ONNX model exports. ONNX runtime inference matches PyTorch. |
| M5.2 | ONNX → TensorFlow | TF SavedModel created. TF inference matches ONNX (within 1%). |
| M5.3 | TF → TFLite (float) | TFLite model <200KB. Inference matches TF (within 1%). |
| M5.4 | TFLite quantized (INT8) | Model <100KB. Accuracy drop <5% vs float. |
| M5.5 | ESP32 integration | TFLite Micro runs on ESP32. Inference <50ms. |
| M5.6 | End-to-end test | ESP32 receives V2V, runs inference, outputs action. LED responds. |

**Acceptance Criteria for Phase 5 Complete:**
- [ ] TFLite model < 100KB
- [ ] Inference time on ESP32 < 50ms
- [ ] Accuracy loss from quantization < 10%
- [ ] End-to-end demo works (V2V → inference → LED)

### Phase 6: Real-World Validation

**Duration:** 3-5 days
**Owner:** Amir + Team
**Dependencies:** Phase 5 complete

| Milestone | Deliverable | Acceptance Test |
|-----------|-------------|-----------------|
| M6.1 | 3-vehicle convoy test | Model runs on V001 during real convoy drive. Actions logged. |
| M6.2 | Collision avoidance test | Lead vehicle hard brakes. V001 model outputs Brake/Emergency in <200ms. |
| M6.3 | False positive test | Normal driving. Model outputs Maintain >80% of time. |
| M6.4 | SSM comparison | SUMO replay with SSM. Model actions correlate with TTC thresholds. |

**Acceptance Criteria for Phase 6 Complete:**
- [ ] Real-world test video showing model in action
- [ ] Collision avoidance demonstrated (lead brakes, V001 responds)
- [ ] False positive rate < 15%
- [ ] Thesis-ready results documented

---

## 5. Risk and Mitigation Table

| Risk | Probability | Impact | Mitigation | Contingency |
|------|-------------|--------|------------|-------------|
| **RTT characterization shows unexpected distribution** (bimodal, heavy tail) | Medium | Medium | Document findings honestly. Adjust emulator to match reality. May need non-Gaussian latency model. | If bimodal: model as mixture distribution. If heavy tail: use empirical distribution sampling. |
| **Packet loss is bursty, not independent** | Medium | Medium | Analysis script detects burst patterns. Emulator includes burst loss model. | If severe bursts: increase domain randomization. Document impact on Sim2Real. |
| **SUMO TraCI integration issues** | Low | High | Start with simple straight-road scenario. Test TraCI connection before full environment. | Fallback: Offline FCD processing (not real-time TraCI). |
| **Training diverges / reward doesn't improve** | Medium | High | Start with simpler reward (binary collision/safe). Tune hyperparameters systematically. | Fallback: Use DQN instead of PPO. Reduce observation space further. |
| **Sim2Real gap > 30%** | Medium | High | Domain randomization (train 1.5x wider than measured). Frequent validation on real hardware. | Fallback: Fine-tune on small real dataset. Document gap in thesis. |
| **TFLite quantization accuracy loss > 15%** | Medium | High | Use quantization-aware training. Try different quantization schemes (INT8 vs dynamic). | Fallback: Dual-ESP32 (more memory for larger model). |
| **TFLite model > 100KB** | Medium | Medium | Prune network (remove neurons). Reduce hidden layers. | Fallback: Dual-ESP32 with dedicated ML processor. |
| **ESP32 inference > 100ms** | Low | Medium | Profile and optimize. Use ESP32 DSP extensions. | Fallback: Reduce model complexity. Accept slower inference. |
| **Stale message handling causes instability** | Low | Medium | Clear staleness policy (drop messages > 500ms). Agent learns to handle missing data. | Add explicit "no data" observation state. |
| **GPS position inaccurate for distance calculation** | Medium | Low | Use relative position (difference), not absolute. Apply low-pass filter. | Accept ±5m accuracy. Focus on speed/acceleration signals. |

### Risk Priority Matrix

```
                    IMPACT
              Low    Medium   High
         ┌─────────┬─────────┬─────────┐
    High │         │         │ Training│
         │         │         │ diverges│
P   ─────┼─────────┼─────────┼─────────┤
R   Med  │ GPS     │ Burst   │ Sim2Real│
O        │ accuracy│ loss    │ gap     │
B   ─────┼─────────┼─────────┼─────────┤
    Low  │         │ Stale   │ TraCI   │
         │         │ messages│ issues  │
         └─────────┴─────────┴─────────┘
```

**Focus Areas (High Impact, Any Probability):**
1. Training divergence → Systematic hyperparameter tuning
2. Sim2Real gap → Aggressive domain randomization
3. TFLite quantization → Plan for quantization-aware training

---

## Appendix: Decision Rationale

### A.1 Why Minimal Observation Space

**Previous Approach (from ESPNOW_EMULATOR_DESIGN.md):**
- Observation included gyro[3], mag[3], accel[3]
- Required SensorSynthesizer to derive these from SUMO kinematics
- Added complexity: measure real sensor noise, model noise in emulator

**New Approach:**
- Observation includes only: peer pos/speed/accel + age_ms + ego_speed
- No sensor synthesis required
- Simpler pipeline, faster iteration

**Why This Is Valid:**
1. Convoy following is primarily a **longitudinal control** problem
2. Key signals are: relative position, relative speed, closing rate
3. Gyro/mag provide rotational/heading info - not critical for following
4. Message age/staleness captures communication uncertainty (the real challenge)

### A.2 What Stays in the Message Schema

The V2VMessage struct (90 bytes) still includes gyro[3] and mag[3] fields:

```cpp
struct V2VMessage {
    char vehicle_id[8];
    float lat, lon;
    float speed, heading;
    float accel[3];
    float gyro[3];   // Still transmitted, not used in RL obs
    float mag[3];    // Still transmitted, not used in RL obs
    uint32_t timestamp_ms;
    // ...
};
```

**Rationale:**
- Future scenarios may need heading (intersections)
- Ablation studies can compare minimal vs full observation
- No cost to transmit (fits in 250B ESP-NOW limit)
- Firmware stays consistent across scenarios

### A.3 Sensor Noise Measurement Status

**Previous Plan:** Measure IMU/mag noise during RTT drive, generate `sensor_noise_params.json`

**Updated Status:** OPTIONAL (stretch goal)

- If time permits, add IMU/mag logging to RTT sender firmware
- Generate noise profiles for future use
- **Not blocking** the RL pipeline

**If Implemented Later:**
- SensorSynthesizer class already designed (see ESPNOW_EMULATOR_DESIGN.md)
- Can add to observation space via config flag
- Enables ablation: "Does adding sensor data improve policy?"

---

## Document Changelog

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-12-26 | Amir Khalifa | Initial document based on PROMPT_2512_2034.md decisions |

---

**END OF DOCUMENT**
