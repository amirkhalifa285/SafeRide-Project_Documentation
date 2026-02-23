# Cloud Training Run 002 Results

**Date:** February 21, 2026  
**Run ID:** `cloud_prod_002`  
**Model Path:** `roadsense-v2v/ml/models/runs/cloud_prod_002/results/model_final.zip`

## 1) Outcome Summary
- Training completed to `10,000,000` timesteps.
- Final post-fix evaluation completed with `40` episodes (`10` per eval scenario).
- Aggregate evaluation (`hazard_injection=false`):
  - `collision_rate = 0.0`
  - `success_rate = 1.0`
  - `truncation_rate = 1.0`
  - `avg_reward = 508.06`

## 2) Per-Peer-Count Evidence (from `eval_final_40ep.json`)
- `n=1`: 20 episodes, 0 collisions, avg reward `494.7`
- `n=2`: 20 episodes, 0 collisions, avg reward `521.425`

## 3) Per-Scenario Coverage
- `eval_000`: 10 episodes
- `eval_001`: 10 episodes
- `eval_002`: 10 episodes
- `eval_003`: 10 episodes

Coverage is balanced across all 4 eval scenarios.

## 4) Critical Operational Lesson
`dataset_v2` is not preserved by copying only run artifacts (`/work/results`) or S3 results output.  
The cloud script uploads model/results, but dataset directories must be copied/backed up separately.

Practical impact:
- A terminated EC2 instance can leave you with model + metrics but without `dataset_v2`.
- If base scenario and generation settings are known, `dataset_v2` can be regenerated deterministically.

## 5) Regeneration / Reproduction Commands
From repo root:

```bash
cd /home/amirkhalifa/RoadSense2/roadsense-v2v
./ml/run_docker.sh generate \
  --base_dir ml/scenarios/base \
  --output_dir ml/scenarios/datasets/dataset_v2/base \
  --seed 42 \
  --train_count 16 \
  --eval_count 4 \
  --route_randomize_non_ego \
  --route_include_v001 \
  --peer_drop_prob 0.3 \
  --min_peers 1 \
  --emulator_params ml/espnow_emulator/emulator_params_measured.json
```

```bash
cd /home/amirkhalifa/RoadSense2/roadsense-v2v
./ml/run_docker.sh python -m ml.scripts.evaluate_model \
  --model_path ml/models/runs/cloud_prod_002/results/model_final.zip \
  --dataset_dir ml/scenarios/datasets/dataset_v2/base \
  --emulator_params ml/espnow_emulator/emulator_params_measured.json \
  --episodes_per_scenario 10 \
  --output ml/models/runs/cloud_prod_002/results/eval_final_40ep.json
```

## 6) Remaining Gap
Run 001 comparison artifact is not currently available locally, so direct `n=2` regression delta vs Run 001 is pending.
