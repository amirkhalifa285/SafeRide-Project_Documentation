# SUMO Base Scenario Plan (Dataset v2)

**Status:** ACTIVE  
**Owner:** User (manual scenario creation)  
**Last Updated:** January 22, 2026

---

## Goal (Primary Emphasis)

Create **5 manually built base SUMO scenarios** from real-world map crops (OSM → SUMO) that **physically make sense in the GUI**, then **augment** those bases into **~80–100 total scenarios** using the generator. The output must **cover variable peer counts n ∈ {1,2,3,4,5}** across the dataset, achieved via augmentation (not by varying base vehicle counts).

---

## Constraints & Must-Haves

- **V001 (ego) must depart at t=0** in every base (generator validates this).
- **Base scenarios always use 3 vehicles** (V001 + V002 + V003).  
  - Peer-count variation is handled during augmentation (optional peer dropping).
- **Avoid lane changes** to match model assumptions. Prefer **single-lane** routes/networks.
- **Routes must be physically plausible** (validated in SUMO GUI).
- **Vehicles sorted by depart time** in `vehicles.rou.xml` (SUMO ordering requirement).
- **SSM device enabled** in vehicle type (copy from existing base).

---

## Plan

### 1) Define the 5 Base Scenarios (Manual)

Pick 5 distinct real-map crops with different topology types. Example set:
- Straight arterial
- Curved arterial
- T-intersection / 4-way
- Merge/on-ramp
- Roundabout

All bases use 3 vehicles. Peer-count variation is applied during augmentation.

---

### 2) Build Each Base Scenario (Repeat ×5)

**Files per base:**
- `map.osm` (cropped OSM)
- `network.net.xml` (SUMO network)
- `vehicles.rou.xml` (routes + vehicles)
- `scenario.sumocfg` (simulation config)

**Steps:**
1. Crop OSM to a small area and save as `map.osm`.
2. Convert to SUMO network:  
   `netconvert --osm-files map.osm -o network.net.xml`
3. Create `vehicles.rou.xml`:
   - Include `V001` + `V002..V00N` peers
   - Ensure realistic spawn positions and routes
   - Enable SSM device
   - Sort vehicles by depart time
4. Create `scenario.sumocfg`:
   - 0.1s step length
   - FCD output enabled

### Base Tracking
- [x] Base 01 (single-lane residential chain, GUI validated)
- [ ] Base 02
- [ ] Base 03
- [ ] Base 04
- [ ] Base 05

---

### 3) GUI Validation (Critical)

Run each base in **SUMO GUI** and visually confirm:
- Vehicles spawn and remain on valid lanes
- Directions and paths make sense
- Spacing and speeds are realistic
- No teleports / invalid routes

Only proceed to augmentation once a base passes GUI sanity checks.

---

### 4) Augment Each Base into a Dataset Slice

Use the generator to produce 16–20 scenarios per base with optional route randomization and peer drops:

```
cd roadsense-v2v
./ml/run_docker.sh generate \
  --base_dir ml/scenarios/base_<X> \
  --output_dir ml/scenarios/datasets/dataset_v2/<X> \
  --seed 42 \
  --train_count 16 \
  --eval_count 4 \
  --route_randomize_non_ego \
  --peer_drop_prob 0.3 \
  --min_peers 1
```

Repeat for all 5 bases → **80–100 total scenarios**.

---

### 5) Consolidate + Spot-Check

If required for training, merge base-specific dataset outputs into a single
`dataset_v2` with unified `train/`, `eval/`, and `manifest.json`.  
Spot-check a few augmented scenarios in GUI.

---

## Acceptance Criteria

- [ ] 5 base scenarios created from real map crops
- [ ] Each base visually validated in SUMO GUI
- [ ] Augmentation covers **n ∈ {1,2,3,4,5}** via peer dropping
- [ ] 80–100 total scenarios generated
- [ ] Spot checks confirm augmentations remain physically plausible
