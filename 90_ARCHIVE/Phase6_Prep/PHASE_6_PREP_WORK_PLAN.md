# Phase 6.3+: Prep Work Plan — Validate, Process, Generate

**Created:** March 1, 2026
**Status:** ACTIVE — IMMEDIATE PRIORITY
**Goal:** Complete all prep work (Steps 1-3) needed before Training Run 003.
**Prerequisite:** Recording #2 (GO) + Extra driving data (GO) — COMPLETE.

---

## Executive Summary

Three steps stand between us and Run 003:

| Step | What | Deliverable | New Code Needed? |
|------|------|-------------|------------------|
| **1** | Validate emulator params | Validated `emulator_params_measured.json` (minor updates) | Small validation script |
| **2** | Process convoy logs into SUMO scenario | `ml/scenarios/base_real/` | V002/V003 trajectory extraction + SUMO route generation |
| **3** | Generate dataset_v3 | `ml/scenarios/datasets/dataset_v3/` | Eval peer-count enforcement script |

### Progress Update (March 1, 2026)

- **Step 1 COMPLETE:** `emulator_params_measured.json` validated against Recording #2, burst mean updated to ~1.6, metadata updated, backup created, validation script added.
- **Step 2.2.1 COMPLETE:** `analyze_convoy_recording.py` now extracts `V002`/`V003` trajectories from V001 RX data in ego-only mode.
- **Step 2.2.2 COMPLETE:** `map.osm` exported and `network.net.xml` generated in `ml/scenarios/base_real/`.
- **Step 2.2.3-2.2.5 COMPLETE:** `process_convoy_to_sumo.py` created; `vehicles.rou.xml`, `scenario.sumocfg`, and `recording_metadata.json` generated.
- **Critical correction applied:** route locked to single driven edge `-635191227` and `departLane=\"0\"` for all vehicles to prevent reverse-edge/U-turn behavior.
- **Step 3 COMPLETE:** dataset_v3 generated (25 train + 10 eval). Eval peer counts enforced deterministically (2x each n=1-5). 1000-step smoke train passed. All 25 unit tests pass.
- **Blocker resolved:** eval dropout produced only 1x n=5 with seed=42. Fixed by adding `--eval_peer_drop_prob 0.0` to gen_scenarios.py (eval keeps all peers, fixer trims to exact counts).

**ALL PREP WORK COMPLETE. READY FOR RUN 003.**

---

## Step 1: Emulator Param Validation

### 1.1 Current State

The active `emulator_params_measured.json` was calibrated from **Recording #1 (Feb 21, 2026)**
using 6-file mode (all 3 vehicles logging). That calibration used cross-device clock offset
estimation and separated healthy links from bad links (V001-V002 had ~85% loss due to
board placement near the car door).

**Current params (from Rec #1 healthy links):**
- Latency base: 3.84ms, jitter: 1.04ms
- Direct link loss: 5.8% (0-20m), 11.6% (20-50m)
- Sensor noise (cruising): GPS 2.96m, heading 4.71deg, accel 0.39 m/s2
- Burst loss: mean length 1.245

**Recording #2 (Feb 28)** was ego-only (V001 TX+RX only). Its raw extracted params
(`convoy_analysis_site/convoy_emulator_params.json`) have **inflated values** for GPS
(11.7m) and heading (28.6deg) because ego-only mode can't properly separate sensor noise
from trajectory variance — the "cruising" segment only had 264 samples and was likely on
a curve.

However, Rec #2 provides valuable **per-link validation data:**
- V002->V001 direct link: PDR 0.852, relative latency mean 0.66ms / p95 3.0ms
- V003->V001 (via relay): PDR 0.752 (expected lower due to relay path loss)
- V002->V001 burst stats: mean burst 1.97, max 23

### 1.2 Decision: Keep Rec #1 Params, Validate Against Rec #2

Rec #1's 6-file calibration produced better-quality estimates (proper clock sync,
healthy-link filtering, stationary/cruising noise separation). Rec #2 confirms these
are in the right range.

**What to do:**
1. **Validate** that Rec #1 values are consistent with Rec #2 per-link data
2. **Update burst stats** if Rec #2 provides better data (it has more samples)
3. **Update `_metadata`** to document that both recordings were considered
4. **Optionally widen domain randomization** ranges to encompass both recordings

### 1.3 Validation Checklist

| Parameter | Rec #1 Value | Rec #2 V002->V001 | Consistent? |
|-----------|-------------|-------------------|-------------|
| Latency base (ms) | 3.84 | 0.66 mean / 3.0 p95 | Yes (same order of magnitude) |
| Direct link PDR | 0.942 (0-20m) | 0.852 | Yes (Rec #2 slightly worse, wider formation) |
| Accel noise (m/s2) | 0.39 | 0.27-0.30 (Rec #2 accel segment) | Yes |
| Gyro noise (rad/s) | 0.023 | 0.025 | Yes |
| Burst mean length | 1.245 | 1.97 | Update to ~1.6 (average) |

**Conclusion:** Current params are valid. Minor burst stat update + metadata annotation.

### 1.4 Implementation

1. Backup current params:
   ```bash
   cp ml/espnow_emulator/emulator_params_measured.json \
      ml/espnow_emulator/emulator_params_pre_v3_backup.json
   ```
2. Update `burst_loss.mean_burst_length` to weighted average of both recordings
3. Add to `_metadata`:
   - `"validated_against": "convoy_recording_02282026 + convoy_extra_data_02282026"`
   - `"rec2_v002_v001_pdr": 0.852`
   - `"rec2_v002_v001_latency_p95_ms": 3.0`
   - `"validation_date": "2026-03-XX"`
4. Optionally widen domain randomization `loss_rate_range` upper bound
   from 0.23 to 0.25 (Rec #2 showed V003->V001 at 24.8%, though that includes relay)

### 1.5 Files Touched

| File | Action |
|------|--------|
| `ml/espnow_emulator/emulator_params_measured.json` | **UPDATE** — burst stats + metadata |
| `ml/espnow_emulator/emulator_params_pre_v3_backup.json` | **CREATE** — backup |

### 1.6 Verification

- [x] Burst mean length updated (was 1.245, now ~1.6)
- [x] `_metadata` documents validation against Rec #2
- [x] Domain randomization ranges still sensible
- [x] No other values changed without justification

---

## Step 2: Process Convoy Logs into SUMO Base Scenario

### 2.1 Current State

**What exists:**
- `ml/scripts/analyze_convoy_recording.py` — the main processing script (97KB).
  Already run with `--no-plots` to produce:
  - `ml/data/convoy_analysis_site/trajectories/V001_trajectory.csv` (1956 samples, 195.5s)
  - `ml/data/convoy_analysis_site/analysis_summary.json` (formation stats, PDR, noise)
  - `ml/data/convoy_analysis_site/convoy_observations.npz` (ego + peer observations)
- V002/V003 position data exists **inside V001's RX log** (their broadcast lat/lon/speed/heading)
  but has NOT been extracted as separate trajectory CSVs yet
- Recording location: rural area outside Deir Hanna, near the Sonol fuel station by Nahal
  Tzalmon, between Mghar and Eilaboun. GPS start coords: ~32.8596N, 35.4031E.
- Board placement: front windshield of each car (line-of-sight, production-realistic)

**What doesn't exist:**
- (Resolved March 1, 2026) V002/V003 trajectory CSVs extracted from RX data
- (Resolved March 1, 2026) SUMO network generated from exported OSM
- (Resolved March 1, 2026) `ml/scenarios/base_real/` directory populated
- (Resolved March 1, 2026) GPS-to-SUMO processing script implemented

### 2.2 Sub-steps

#### 2.2.1 Extract V002/V003 Trajectories from RX Data

The V001 RX log contains broadcast data from V002 and V003 (lat, lon, speed, heading
at each received packet). We need to extract these into trajectory CSVs matching the
V001 format.

**Approach:** Add a function (or modify `extract_trajectories` in `analyze_convoy_recording.py`)
to also process RX data per sender. The RX data will have gaps (dropped packets) —
interpolate to uniform 10Hz timestamps using the same approach as V001 trajectory
extraction (np.interp).

**Output:**
- `convoy_analysis_site/trajectories/V002_trajectory.csv`
- `convoy_analysis_site/trajectories/V003_trajectory.csv`

Same format: `t_local_ms, t_rel_s, lat, lon, x_m, y_m, speed_ms, heading_deg`

**Note:** V002/V003 trajectories will be noisier than V001's (they went through
ESP-NOW transmission) and will have packet-loss gaps. Linear interpolation handles
the gaps for SUMO route generation purposes.

#### 2.2.2 SUMO Network (Amir — manual)

Same workflow as the existing base/ scenario:
1. Go to OpenStreetMap, export the recording area around the route
2. Save as `ml/scenarios/base_real/map.osm`
3. Convert with netconvert inside Docker:
   ```bash
   cd /home/amirkhalifa/RoadSense2/roadsense-v2v
   ./ml/run_docker.sh bash
   # Inside container:
   netconvert --osm-files ml/scenarios/base_real/map.osm \
     -o ml/scenarios/base_real/network.net.xml \
     --geometry.remove --ramps.guess --junctions.join \
     --tls.guess-signals --tls.discard-simple --tls.join
   ```
4. Inspect the network to identify the edge IDs along the recording route

#### 2.2.3 Generate vehicles.rou.xml

Create `ml/scripts/process_convoy_to_sumo.py` that:

**Inputs:**
- V001 trajectory CSV (from analysis output)
- V002, V003 trajectory CSVs (from step 2.2.1)
- SUMO network file (`base_real/network.net.xml`)

**Processing:**
1. Load all 3 trajectory CSVs
2. Identify the **active driving segment** (skip stationary portions where speed < 0.5 m/s)
3. Use `sumolib` to:
   - Load the network
   - Convert GPS lat/lon to SUMO XY coordinates
   - Find the closest edge for trajectory start/end points
   - Determine the route (sequence of edges) for each vehicle
4. Extract real vehicle parameters from trajectory data:
   - `departSpeed`: speed at start of active segment
   - `departPos`: position along the starting edge
   - Formation spacing from analysis_summary.json (V001-V002: ~11m, V001-V003: ~17m)
   - `decel`: max observed braking ~5.0 m/s2 (for vType)
   - `tau`: estimated from following distance / speed
5. Define multiple `<route>` IDs (for augmentation variety) if the network has
   alternative paths
6. **Sort vehicles by depart time** (SUMO requirement — V001 LAST since it's rear)

**Synthetic peers (V004, V005, V006):**
- Placed AHEAD of V003 in the convoy (further forward)
- Rationale: ego (V001) is at the rear. Cone filter is front-facing. Vehicles behind
  V001 would be invisible to the model. More vehicles ahead = longer relay chains =
  better mesh training. The RL model reacts to braking events propagating backward.
- Use V002/V003 speed profiles as templates with position offsets (20-30m ahead of V003)
- Same route, different departPos values
- Depart BEFORE all other vehicles (further ahead = earlier depart in SUMO)

**Output:** `ml/scenarios/base_real/vehicles.rou.xml`
```xml
<routes>
  <vType id="car" accel="2.6" decel="5.0" sigma="0.5" length="5"
    maxSpeed="50" tau="1.0">
    <param key="has.ssm.device" value="true"/>
    <param key="device.ssm.measures" value="TTC DRAC PET"/>
    <param key="device.ssm.thresholds" value="4.0 3.4 2.0"/>
    <param key="device.ssm.range" value="100.0"/>
    <param key="device.ssm.file" value="ssm_output.xml"/>
  </vType>

  <route id="convoy_route" edges="...real edge IDs..."/>
  <!-- Additional route IDs for augmentation variety if available -->

  <!-- Sorted by depart time: front of convoy first -->
  <vehicle id="V006" type="car" depart="0" route="convoy_route"
    departSpeed="..." departPos="..." color="1,0.5,0"/>
  <vehicle id="V005" type="car" depart="0" route="convoy_route"
    departSpeed="..." departPos="..." color="1,0.5,0"/>
  <vehicle id="V004" type="car" depart="0" route="convoy_route"
    departSpeed="..." departPos="..." color="1,0.5,0"/>
  <vehicle id="V003" type="car" depart="0" route="convoy_route"
    departSpeed="..." departPos="..." color="1,0,0"/>
  <vehicle id="V002" type="car" depart="0" route="convoy_route"
    departSpeed="..." departPos="..." color="0,1,0"/>
  <vehicle id="V001" type="car" depart="0" route="convoy_route"
    departSpeed="..." departPos="..." color="0,0,1"/>
</routes>
```

#### 2.2.4 Generate scenario.sumocfg

```xml
<configuration>
  <input>
    <net-file value="network.net.xml"/>
    <route-files value="vehicles.rou.xml"/>
  </input>
  <time>
    <begin value="0"/>
    <end value="120"/>
    <step-length value="0.1"/>
  </time>
  <output>
    <fcd-output value="fcd_output.csv"/>
    <fcd-output.geo value="false"/>
    <fcd-output.acceleration value="true"/>
  </output>
</configuration>
```

#### 2.2.5 Create recording_metadata.json

```json
{
  "recording_date": "2026-02-28",
  "location": "Rural road outside Deir Hanna, near Sonol station by Nahal Tzalmon, between Mghar and Eilaboun",
  "gps_start": {"lat": 32.8596, "lon": 35.4031},
  "board_placement": "Front windshield of each car (line-of-sight, production-realistic)",
  "duration_s": 195.5,
  "vehicles_recorded": ["V001", "V002", "V003"],
  "vehicles_synthetic": ["V004", "V005", "V006"],
  "formation_avg_spacing_m": 13.9,
  "hard_braking_peak_ms2": -8.63,
  "source_files": {
    "tx": "Convoy_recording_02282026/V001_tx_004.csv",
    "rx": "Convoy_recording_02282026/V001_rx_004.csv"
  },
  "analysis_dir": "ml/data/convoy_analysis_site/",
  "notes": "Ego-only mesh recording. V002/V003 trajectories extracted from V001 RX log. V004-V006 are synthetic peers placed ahead of V003."
}
```

### 2.3 Verification

- [x] V002/V003 trajectory CSVs extracted and look reasonable (speed, heading, no huge gaps)
- [x] OSM exported and netconvert succeeds
- [x] Edge IDs identified for the recording route
- [x] `vehicles.rou.xml` has all 6 vehicles sorted by depart time
- [x] V004-V006 placed ahead of V003 with realistic spacing
- [x] SUMO simulation runs without errors (smoke test in Docker)
- [x] V001-V003 speeds/spacing approximately match recording
- [x] `recording_metadata.json` has complete provenance

### 2.4 Files Touched

| File | Action |
|------|--------|
| `ml/scripts/analyze_convoy_recording.py` | **UPDATE** — add RX-based trajectory extraction |
| `ml/scripts/process_convoy_to_sumo.py` | **CREATE** — GPS trajectories to SUMO route file |
| `ml/scenarios/base_real/map.osm` | **CREATE** — Amir exports from OSM |
| `ml/scenarios/base_real/network.net.xml` | **CREATE** — from netconvert |
| `ml/scenarios/base_real/vehicles.rou.xml` | **CREATE** — from processing script |
| `ml/scenarios/base_real/scenario.sumocfg` | **CREATE** |
| `ml/scenarios/base_real/recording_metadata.json` | **CREATE** |

---

## Step 3: Generate dataset_v3

### 3.0 Architect Review — APPROVED WITH CHANGES (March 1, 2026)

Architect validated the dataset strategy. Key decisions finalized:

- [x] **No synthetic merge.** dataset_v3 is 100% base_real-derived. The old synthetic
  base (base/) only supports n=1,2 and has a different road network / speed regime.
  Mixing it in would bias training toward low peer counts and confuse the model with
  incompatible initial conditions. Behavioral diversity comes from gen_scenarios.py
  augmentation ranges (decel, tau, sigma, speedFactor) — no need for a second base.
- [x] **No route randomization flags.** base_real has a single route (`convoy_route`
  on edge `-635191227`). The `--route_randomize_non_ego` flag requires 2+ routes to
  do anything — it silently no-ops with 1 route. Drop the flags to avoid confusion.
- [x] **Eval protocol approved.** Hazard injection ON (default), deterministic n=1..5
  coverage (2 scenarios per n), 200 total eval episodes.
- [x] **scenario.sumocfg end=120 is correct.** `<end value="120"/>` means 120 simulation
  seconds (not 120 steps). ConvoyEnv DEFAULT_MAX_STEPS=1000 at step_length=0.1 = 100
  sim seconds. SUMO's 120s limit is safely above the episode length. No change needed.

### 3.1 Dataset Specification

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Base scenario | `ml/scenarios/base_real/` | Real-data grounded (professor's requirement) |
| Emulator params | `emulator_params_measured.json` (validated) | From Step 1 |
| Training scenarios | 25 | Enough for good peer-count distribution across n=1-5 |
| Eval scenarios | 10 | Exactly 2 per peer count (n=1,2,3,4,5) |
| Seed | 42 | Reproducibility |
| Peer drop prob | 0.4 | With 5 base peers, gives good spread across n=1-5 |
| Min peers | 1 | Allow n=1 scenarios |
| Route randomization | **NO** | Single route in base_real, flags would silently no-op |
| Data source | **100% base_real** | No synthetic mixing — all real-grounded |

**Expected peer count distribution (5 base peers, drop_prob=0.4, 25 scenarios):**

| Peers (n) | Approx. probability | Expected count |
|-----------|-------------------|----------------|
| n=5 | 7.8% | ~2 |
| n=4 | 25.9% | ~6 |
| n=3 | 34.6% | ~9 |
| n=2 | 23.0% | ~6 |
| n=1 | (min_peers catches) | ~2 |

### 3.2 Scenarios vs Episodes (Important Distinction)

- **Scenarios** = SUMO files (road network + vehicle routes). These are static.
- **Episodes** = RL rollouts on those scenarios. Each episode has different hazard
  injection timing, domain randomization seed, and emulator noise.

The evaluation script runs **multiple episodes per scenario**. Target: **200 episodes**.

| Eval setup | Episodes per scenario | Total episodes | Episodes per n-value |
|------------|----------------------|----------------|---------------------|
| 10 scenarios (2 per n) | 20 each | **200** | **40 per n** |

40 episodes per n-value gives solid statistical power for comparing model performance
across different peer counts. The road geometry is the same for all scenarios (single
edge from the real recording), so more scenarios with fewer episodes wouldn't help —
the variation comes from the RL episode randomization, not from different roads.

### 3.3 Eval Scenario Strategy

Eval scenarios MUST have **deterministic peer counts** — exactly 2 scenarios per n value.
Random dropout won't reliably produce exactly 2 of each.

**Approach:** Generate eval scenarios normally, then post-process to enforce exact counts.

**PREREQUISITE:** `fix_eval_peer_counts.py` MUST be written and tested BEFORE running
the generation command. No point generating 10 eval scenarios if we can't guarantee
the exact n coverage afterward.

Create `ml/scripts/fix_eval_peer_counts.py` that:
1. Reads the 10 generated eval scenario route files
2. For each target count (1,1,2,2,3,3,4,4,5,5):
   - Keeps V001 (ego, always present)
   - Keeps exactly n non-V001 vehicles
   - Removes extras from the XML (drop furthest synthetic peers first: V006, V005, V004)
   - Preserves augmented vehicle parameters (decel, tau, sigma, speedFactor)
3. Renames scenarios: `eval_n1_000`, `eval_n1_001`, `eval_n2_000`, etc.
4. Validates V001 still has depart=0 in each rewritten file
5. Updates manifest with:
   - New eval scenario list
   - `"eval_peer_counts_enforced": true`
   - Corrected `peer_count_distribution` (the original will be stale after rewrite)

### 3.4 Execution Steps (In Order)

**Step 3a — Write and test `fix_eval_peer_counts.py` FIRST:**
```bash
cd /home/amirkhalifa/RoadSense2/roadsense-v2v/ml
source venv/bin/activate
# Write the script, then dry-run test against base_real/vehicles.rou.xml
# to verify it correctly removes vehicles and preserves XML structure
```

**Step 3b — Generate dataset:**
```bash
cd /home/amirkhalifa/RoadSense2/roadsense-v2v
./ml/run_docker.sh generate \
  --base_dir ml/scenarios/base_real \
  --output_dir ml/scenarios/datasets/dataset_v3/base_real \
  --seed 42 \
  --train_count 25 \
  --eval_count 10 \
  --peer_drop_prob 0.4 \
  --min_peers 1 \
  --emulator_params ml/espnow_emulator/emulator_params_measured.json
```

**Note:** No `--route_randomize_non_ego` or `--route_include_v001` flags.
Single route in base_real means these would silently no-op.

**Step 3c — Enforce eval peer counts:**
```bash
cd /home/amirkhalifa/RoadSense2/roadsense-v2v/ml
source venv/bin/activate
python scripts/fix_eval_peer_counts.py \
  --dataset_dir scenarios/datasets/dataset_v3/base_real \
  --target_counts 1,1,2,2,3,3,4,4,5,5
```

**Step 3d — Smoke test (1000 training steps):**
```bash
cd /home/amirkhalifa/RoadSense2/roadsense-v2v
./ml/run_docker.sh train --timesteps 1000 \
  --dataset ml/scenarios/datasets/dataset_v3/base_real \
  --emulator_params ml/espnow_emulator/emulator_params_measured.json
```

### 3.5 Verification

- [ ] `fix_eval_peer_counts.py` written and passes dry-run test
- [ ] 25 training scenarios exist and load in SUMO without errors
- [ ] 10 eval scenarios with exact peer counts: 2x n=1, 2x n=2, 2x n=3, 2x n=4, 2x n=5
- [ ] Manifest JSON includes:
  - `peer_count_distribution` (updated after eval fix)
  - `augmentation_config`
  - `eval_peer_counts_enforced: true`
  - Emulator params hash pointing to validated file
- [ ] Quick training smoke test: 1000 steps complete without crash
- [ ] Verify peer count distribution in training set covers n=1 through n=5
- [ ] Document in manifest: `"initial_speed_regime": "low_speed_start_from_standing"`
  (differs from prior runs which started at 15 m/s — important for comparison)

### 3.6 Files Touched

| File | Action |
|------|--------|
| `ml/scenarios/datasets/dataset_v3/` | **CREATE** — full dataset |
| `ml/scripts/fix_eval_peer_counts.py` | **CREATE** — eval peer count enforcer (write FIRST) |

---

## Dependency Graph

```
              ┌──────────────────────────────────────┐
              │  Recording #2 + Extra Data (DONE)    │
              └──────────┬───────────────┬───────────┘
                         │               │
                ┌────────▼──────┐  ┌─────▼─────────────────┐
                │  STEP 1       │  │  STEP 2.2.1           │
                │  Validate     │  │  Extract V002/V003    │
                │  Emulator     │  │  trajectories from RX │
                │  Params       │  └─────┬─────────────────┘
                └────────┬──────┘        │
                         │         ┌─────▼─────────────────┐
                         │         │  STEP 2.2.2           │
                         │         │  OSM Export + Network  │
                         │         │  (Amir manual)        │
                         │         └─────┬─────────────────┘
                         │               │
                         │         ┌─────▼─────────────────┐
                         │         │  STEP 2.2.3-5         │
                         │         │  Generate routes +    │
                         │         │  scenario + metadata  │
                         │         └─────┬─────────────────┘
                         │               │
                ┌────────▼───────────────▼──────────┐
                │          STEP 3                    │
                │   Generate dataset_v3              │
                │   (needs both Step 1 + Step 2)     │
                └────────────────┬──────────────────┘
                                 │
                        ┌────────▼────────┐
                        │  Run 003 Ready  │
                        └─────────────────┘
```

**Parallelism:** Step 1 and Step 2.2.1 can happen in parallel.
Step 2.2.2 (OSM export) can happen anytime. Step 3 needs everything above done.

---

## Execution Order

### Session 1: Step 1 + Step 2.2.1 + Step 2.2.2 (parallel)

**Task A — Validate emulator params (~15 min):**
1. Backup current params
2. Update burst stats + metadata
3. Commit

**Task B — Extract V002/V003 trajectories (~30 min):**
1. Add RX-based trajectory extraction to analyze_convoy_recording.py
   (or write a small standalone extractor)
2. Re-run analysis (or just the extraction) to produce V002/V003 CSVs
3. Verify trajectories look reasonable

**Task C — OSM export (Amir, ~15 min):**
1. Export recording area from OpenStreetMap
2. Save to `ml/scenarios/base_real/map.osm`
3. Run netconvert in Docker

### Session 2: Step 2.2.3-5

**Task D — Create base_real scenario (~1-2 hours):**
1. Write `process_convoy_to_sumo.py`
2. Identify edge IDs in the network
3. Generate vehicles.rou.xml with all 6 vehicles
4. Create scenario.sumocfg + recording_metadata.json
5. Smoke test in SUMO Docker
6. Commit

### Session 3: Step 3

**Task E — Generate dataset_v3 (~45 min):**
1. Write `fix_eval_peer_counts.py` and dry-run test it FIRST
2. Run gen_scenarios.py with base_real (NO route randomization flags)
3. Run fix_eval_peer_counts.py to enforce deterministic n=1-5 eval coverage
4. Verify all scenarios + peer count distribution in manifest
5. Training smoke test (1000 steps in Docker)
6. Commit

---

## Risk Register

| Risk | Impact | Mitigation |
|------|--------|------------|
| V002/V003 RX data too gappy for trajectory extraction | Can't get usable peer trajectories | V002 PDR is 0.852 (good), V003 is 0.752 (acceptable). Linear interpolation fills gaps. |
| OSM road doesn't match GPS path | Vehicles placed off-road in SUMO | Use sumolib edge matching; inspect network in netedit if needed |
| gen_scenarios.py dropout doesn't yield all n values | Missing some n in training | train_count=25 with 5 base peers makes this statistically unlikely; verify in manifest |
| SUMO car-following model doesn't replicate hard braking | Training scenarios too calm | ConvoyEnv has hazard_injection=True which triggers braking events during training regardless |
| Synthetic peers (V004-V006) behave unrealistically | Training on bad data | They use same vType as real peers; augmentation jitters params anyway; SUMO car-following handles dynamics |

---

## DO NOT

- DO NOT use raw ego-only convoy_emulator_params.json values directly (GPS 11m, heading 28deg are inflated)
- DO NOT create base_real with only 2 peers — need 5 for n=1-5 dropout coverage
- DO NOT place synthetic peers behind V001 — cone filter is front-facing, they'd be invisible
- DO NOT forget `--forward-axis y` if re-running any analysis scripts
- DO NOT generate eval scenarios with random peer counts — force exact n=1,2,3,4,5
- DO NOT forget to sort vehicles by depart time in vehicles.rou.xml
- DO NOT mix synthetic base scenarios into dataset_v3 — 100% base_real only
- DO NOT use `--route_randomize_non_ego` or `--route_include_v001` with base_real (single route, would silently no-op)
- DO NOT run gen_scenarios.py before `fix_eval_peer_counts.py` is written and tested

---

## Success Criteria (Gate to Run 003)

All of the following must be true before starting Training Run 003:

- [x] `emulator_params_measured.json` validated + burst stats updated
- [x] V002/V003 trajectory CSVs extracted from RX data
- [x] `ml/scenarios/base_real/` exists with network + routes + config + metadata
- [x] SUMO runs base_real scenario without errors
- [x] `ml/scenarios/datasets/dataset_v3/` generated with 25 train + 10 eval scenarios
- [x] Eval scenarios cover n=1,2,3,4,5 (exactly 2 each, deterministic)
- [x] Quick training smoke test passes (1000 steps, run_20260301_025156, no crash)
- [ ] All files committed to git

---

## Document History

- **March 1, 2026:** Created. Comprehensive plan for Steps 1-3.
- **March 1, 2026 (Update):** Step 1 and Step 2 completed in workspace; route constrained to single driven edge/one-lane behavior to remove U-turn artifact.
- **March 1, 2026 (Update 2):** Added proposed mixed-dataset strategy for Step 3 and marked architect validation as a required gate before dataset generation.
- **March 1, 2026 (Update 3):** Architect review APPROVED WITH CHANGES. Finalized: 100% base_real (no synthetic merge), no route randomization flags (single route), scenario.sumocfg end=120 confirmed correct (120 sim seconds, not steps), fix_eval_peer_counts.py must be written before generation, 200-episode eval strategy (10 scenarios x 20 episodes = 40 per n-value).
- **March 1, 2026 (Update 4):** ALL STEPS COMPLETE. dataset_v3 generated and validated. Eval dropout blocker resolved (added `--eval_peer_drop_prob 0.0`). Smoke train passed (run_20260301_025156). Ready for Run 003.
