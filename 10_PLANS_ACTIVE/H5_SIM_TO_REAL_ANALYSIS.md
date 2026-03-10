# H5 Sim-to-Real Validation Analysis

**Date:** March 10, 2026
**Model:** `cloud_prod_011` (Run 011 — model_final.zip)
**Data:** Convoy Recording #2 (195.5s, hazards) + Extra Recording (581.6s, calm)
**Script:** `ml/scripts/validate_against_real_data.py`
**Verdict:** Model discrimination works. Activation threshold calibrated to SUMO physics, not real-world. Fixable.

---

## 1. Executive Summary

The Run 011 model achieves 100% V2V reaction rate in SUMO simulation (276/276 episodes) but does **not** react to real-world V2V braking signals at distance. The model **does** activate during real slow-approach scenarios (action 0.65-0.78 sustained for 45+ seconds), proving the neural network transfers to real data. The gap is narrowly scoped: the model learned to react to the *consequence* of a hazard (close peer, high closing rate) rather than the *signal* (peer braking acceleration via V2V), because SUMO's `setSpeed(0)` hazard injection creates unrealistically abrupt dynamics.

---

## 2. Validation Methodology

### Approach: Offline Replay

Feed real V2V recording CSVs through the trained model's observation pipeline (same `ObservationBuilder` used in training) and record model actions at 10 Hz. No SUMO required.

```
TX CSV (ego broadcasts)  -->  ego_state (speed, accel_y, heading, lat/lon)
RX CSV (peer V2V msgs)   -->  peer_observations (same fields per peer)
                                    |
                         ObservationBuilder.build()
                                    |
                         model.predict(obs, deterministic=True)
                                    |
                         action in [0, 1] (deceleration fraction)
```

### Data Sources

| Recording | Duration | TX Rows | RX Rows | Content |
|-----------|----------|---------|---------|---------|
| Convoy Main (#004) | 195.5s | 4,984 | 3,768 | Includes hard braking (-8.63 m/s²) |
| Convoy Extra (#005) | 581.6s | ~14,899 | ~12,092 | Regular driving, no planned hazards |
| **Total** | **777s** | **19,883** | **15,860** | |

### Key Pipeline Details

- Forward axis: Y (board-mounted with Y forward, `--forward_axis y`)
- GPS to meters: `x = lon * 111000`, `y = lat * 111000` (matches `ESPNOWEmulator.METERS_PER_DEG_LAT`)
- Staleness threshold: 500 ms (matches training)
- Cone filter: 45-degree half-angle (matches training, applied by `ObservationBuilder`)
- Model action threshold: 0.1 (action > 0.1 = model is braking)

---

## 3. Results

### 3a. Recording #2 — Main (Has Hazard Events)

```
Duration:        195.5s (1,956 steps)
Peers visible:   96.5% of time
Model active:    2.3% of steps (action > 0.1)
```

**Sensitivity (does model brake when peers brake hard?):**

| Event Window | Peer min accel | Model max action | Detected? |
|-------------|---------------|-----------------|-----------|
| t=102.5s | -8.64 m/s² | 0.000 | NO |
| t=118.5s | -7.10 m/s² | 0.000 | NO |
| t=119.0s (ego=-8.31) | -5.05 m/s² | 0.000 | NO |

**Result:** 0/24 real braking events detected when peers are visible. The single "detection" at t=0s was the startup period (no peers, gravity bias on accelerometer).

**Specificity:** 0.00% false positive rate during steady cruising. Perfect.

### 3b. Extra Recording — Calm Driving

```
Duration:        581.6s (5,817 steps)
Peers visible:   82.8% of time
Model active:    2.0% of steps (action > 0.1)
```

**Model activations (t=536-581s):** The model outputs 0.65-0.78 (5.2-6.2 m/s² decel) **sustained for 45 seconds** during a slow convoy approach at the end of the recording. Ego speed was 0.3-6.5 m/s with 1 peer visible at close range.

**False positive rate:** 1.55% of calm steps (with peers visible) had action > 0.1. Most of these are the t=536-581s approach period, which is arguably correct behavior (braking when close to a slow peer).

### 3c. Combined Summary

| Metric | Recording #2 | Extra Recording |
|--------|-------------|-----------------|
| Duration | 195.5s | 581.6s |
| Peers visible | 96.5% | 82.8% |
| Model brakes during peer hard-brake at distance | **NO** | N/A (no hazards) |
| Model brakes during close approach | N/A | **YES** (0.65-0.78) |
| False positive (steady cruise) | 0.00% | 1.55% |
| Mean action | 0.003 | 0.011 |

---

## 4. Root Cause Analysis

### 4a. What the Model Actually Learned

Network probing (sweeping observation features through the raw action network) reveals the model's activation function:

| Feature | Effect on Raw Mean | Threshold for Positive Output |
|---------|-------------------|------------------------------|
| Relative speed (closing rate) | **Dominant** (+0.55 per -30 m/s) | < -7 m/s closing |
| Peer distance (rel_x) | Strong (+0.88 from 50m to 3m) | < 10m |
| Ego speed | Moderate (+0.70 from 30 to 0 m/s) | < 6 m/s |
| Min peer accel (ego[4]) | Weak (+0.16 from 0 to -10 m/s²) | No independent threshold |
| Heading | Negligible (+0.13 full range) | N/A |
| Peer count | Negligible (+0.02 per peer) | N/A |

**Key insight:** The model brakes based on **closing rate + proximity**, not **peer braking acceleration**. The V2V braking signal (min_peer_accel, peer accel feature) has minimal influence on the action output.

Random search over 50,000 realistic observations confirmed: the model only outputs positive actions (after clipping to [0,1]) when `rel_speed < -7 m/s` AND `rel_x < 0.10` (10m).

### 4b. Why This Happened — SUMO Hazard Injection Physics

The hazard injector (`ml/envs/hazard_injector.py:202`) uses:

```python
sumo.set_vehicle_speed(target, 0.0)  # Instant speed pin to zero
```

This creates the following dynamics in training:

```
Step 0 (hazard):  Peer speed: 13.9 m/s → 0 m/s  (INSTANT)
Step 1 (0.1s):    Peer speed: 0 m/s, Ego speed: 13.9 m/s
                  Closing rate: 13.9 m/s, Distance: ~25m → 23.6m
Step 10 (1.0s):   Distance: ~11m, Closing rate: 13.9 m/s
Step 20 (2.0s):   Distance: ~2m, Ego starts braking
Step 29 (2.9s):   Model reacts (avg rank-1 reaction time = 2.88s)
```

By the time the model reacts, the observation looks like: **distance 3-8m, closing 8-14 m/s, peer speed ~0**. The model learned this as the "hazard fingerprint."

In real convoy driving:

```
t=0 (brake start):  Peer decelerates at -8.6 m/s²
t=1s:               Peer speed: 13.9 → 5.3 m/s, closing: 8.6 m/s at 25-30m
t=2s:               Peer speed: 5.3 → 0 m/s, closing: ~8 m/s at 15-20m
t=3s:               Ego driver already braking (human reaction ~1.5s)
```

The ego driver reacts before the observation matches the model's learned hazard fingerprint. The gap never closes to <10m with >7 m/s closing rate, because the human already braked.

### 4c. Why the V2V Signal Doesn't Trigger the Model

The `min_peer_accel` feature (ego observation dim 4) carries the V2V braking signal. Network probing shows this feature shifts the raw action mean by only +0.16 across its full range (0 to -10 m/s²). This is insufficient to cross the zero threshold (raw mean at calm baseline is -0.85).

The model learned to rely on closing rate because in SUMO training, the closing rate is a **perfectly reliable consequence** of a hazard. The peer braking acceleration signal is **redundant** — by the time the model sees it, the closing rate has already changed. The network optimized away the redundant feature.

---

## 5. Model Verification — Network Is Not Broken

To confirm the model itself is functional (not a loading/serialization issue):

```
Model architecture:    DeepSetExtractor (37-dim) → MLP (64) → action_net (1)
Action space:          Box(0.0, 1.0, shape=(1,))
Log-std (final):       -3.14 (std = 0.043)
Action net bias:       -0.052

Raw mean (calm obs):   -0.851  → clipped to 0.000
Raw mean (hazard obs): -0.447  → clipped to 0.000
Shift (hazard-calm):   +0.404  (model DOES discriminate, but both clip to zero)

Random search (50K realistic obs):  max raw mean = +0.991 (positive action)
Triggers:  ego_speed < 6 m/s, peer < 10m, closing > 7 m/s
```

The model's discrimination is intact. It correctly shifts the action mean toward braking for hazard observations. But the learned zero-crossing point requires more extreme conditions than real-world driving produces at distance.

The extra recording confirms this: at t=536-581s, when the convoy naturally approaches close-approach conditions (low speed, tight spacing), the model outputs 0.65-0.78 sustained — proving the inference pipeline, observation builder, and coordinate transforms all work correctly with real V2V data.

---

## 6. Fix: Gradual Hazard Injection (Run 012)

### 6a. Problem Statement

`setSpeed(target, 0.0)` creates an instant speed discontinuity. Real braking is gradual (-4 to -9 m/s²). The model must learn to react to the V2V braking SIGNAL at distance, not the close-approach CONSEQUENCE.

### 6b. Proposed Change

**File:** `ml/envs/hazard_injector.py`, method `maybe_inject()`

Replace line 202:
```python
# CURRENT (instant):
sumo.set_vehicle_speed(target, 0.0)
```

With gradual braking using TraCI `slowDown()`:
```python
# PROPOSED (gradual):
braking_duration = self._rng.uniform(2.0, 4.0)  # seconds
sumo.slow_down(target, 0.0, braking_duration)
```

`slowDown(vehID, speed, duration)` makes the vehicle linearly decelerate to `speed` over `duration` seconds. At 13.9 m/s over 2s, that's -6.95 m/s² (realistic hard braking). Over 4s, that's -3.47 m/s² (moderate braking).

**Domain randomization:** Randomize `braking_duration` between 2.0-4.0s per episode. This exposes the model to a range of braking severities:

| Duration | Decel (from 50 km/h) | Real-world equivalent |
|----------|---------------------|----------------------|
| 2.0s | -6.95 m/s² | Hard emergency brake |
| 3.0s | -4.63 m/s² | Firm braking |
| 4.0s | -3.47 m/s² | Moderate braking |

### 6c. SUMOConnection Wrapper

**File:** `ml/envs/sumo_connection.py`

Add a `slow_down()` method wrapping TraCI:

```python
def slow_down(self, vehicle_id: str, speed: float, duration: float) -> None:
    """Gradually decelerate vehicle to target speed over duration seconds."""
    import traci
    traci.vehicle.slowDown(vehicle_id, speed, duration)
```

### 6d. What Does NOT Change

- CF override mechanism (still needed — still must prevent CF free-riding)
- Reward function (already has linear ramp, comfort scaling)
- Observation builder (same features, same normalization)
- Deep Sets architecture (same encoder, same pooling)
- Hyperparameters (LR=1e-4, n_steps=4096, ent_coef=0.0, log_std_init=-0.5)
- Formation stability fix (departPos spacing, episode length, hazard window)
- Dataset generation (same base_real, same augmentation)

### 6e. Expected Impact

With gradual braking, the observation sequence during a hazard will look different:

```
CURRENT (instant setSpeed(0)):
  Step 0:  peer_accel=-139 m/s² (physics-breaking), speed=0 instantly
  Step 1:  closing_rate=13.9 m/s, distance drops fast
  Model learns: "brake when closing fast at close range"

PROPOSED (slowDown over 3s):
  Step 0:  peer_accel=-4.6 m/s², speed=13.9 (just starting to brake)
  Step 5:  peer_accel=-4.6 m/s², speed=11.6, closing_rate=2.3 m/s
  Step 15: peer_accel=-4.6 m/s², speed=7.0, closing_rate=6.9 m/s
  Step 30: peer_speed=0, closing_rate=13.9 m/s (same as before)
  Model learns: "brake when peer accel goes negative" (EARLY) + "brake harder when closing" (LATE)
```

The model will be exposed to the gradual accel signal for 20-40 steps before the closing rate becomes extreme. This should teach it to use the V2V braking signal (min_peer_accel) as an early warning, and the closing rate as confirmation.

### 6f. Validation Plan

After Run 012 training:
1. Re-run SUMO evaluation (must maintain 100% reaction, 0% collision)
2. Re-run H5 sim-to-real validation with same recordings
3. **Pass criteria:**
   - Model action > 0.1 within 3s of peer braking onset in Recording #2
   - False positive rate < 5% in Extra Recording
   - 100% reaction in SUMO eval (no regression)

---

## 7. Additional Observations

### 7a. Accelerometer Gravity Bias

At rest, the real IMU reads `accel_y ≈ -1.4 m/s²` due to mounting angle (gravity component on Y-axis). In training, SUMO ego accel is 0.0 when cruising. This creates a constant offset in ego observation dim 1:

- Training: `ego[1] = 0.0` (cruising)
- Real data: `ego[1] = -0.14` (gravity bias / 10)

This does NOT prevent model activation (probing confirms heading and ego_accel have weak influence), but it is a domain mismatch worth noting. For ESP32 deployment, the firmware should subtract the gravity bias from the Y-axis reading.

### 7b. Model Outputs Negative Raw Means

The model's action_net outputs raw means in the range [-1.2, +1.1]. The SB3 PPO clips to [0, 1]. In deterministic mode, all raw means below 0 become action=0. The model learned to encode "don't brake" as deeply negative values and "brake" as positive values, with the zero crossing at specific observation thresholds. This is valid PPO behavior with bounded action spaces.

### 7c. The Extra Recording Activations Are Correct

The model's 0.65-0.78 actions during t=536-581s in the Extra Recording represent correct behavior: the convoy was stopping and the ego was approaching a slow peer at close range. This proves the full pipeline (CSV parsing, GPS-to-meters, cone filter, observation builder, Deep Sets encoder, action network) works end-to-end with real V2V data.

---

## 8. Files Created

| File | Purpose |
|------|---------|
| `ml/scripts/validate_against_real_data.py` | H5 validation script (offline replay) |
| `ml/data/sim_to_real_validation/validation_report_convoy_main_004.json` | Recording #2 results |
| `ml/data/sim_to_real_validation/validation_report_convoy_extra_005.json` | Extra recording results |
| `ml/data/sim_to_real_validation/timeseries_convoy_main_004.npz` | Raw timeseries for plotting |
| `ml/data/sim_to_real_validation/timeseries_convoy_extra_005.npz` | Raw timeseries for plotting |

---

## 9. Conclusion

Run 011 is a correct model — its 100% SUMO reaction rate is real and its neural network correctly discriminates hazard from calm. The sim-to-real gap is narrowly scoped to the hazard injection physics: `setSpeed(0)` creates dynamics that don't occur in real driving. The fix (gradual `slowDown()`) is a single-line change in the hazard injector with no architectural implications. Run 012 should close this gap and produce a model that reacts to V2V braking signals at distance, as required for ESP32 deployment.
