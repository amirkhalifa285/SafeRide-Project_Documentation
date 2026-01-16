# Cloud Training Run 001 - Results Summary

**Date:** January 16, 2026
**Run ID:** `cloud_prod_001`
**Instance:** AWS c6i.xlarge (il-central-1)

---

## Training Configuration

| Parameter | Value |
|-----------|-------|
| Total Timesteps | 5,000,000 |
| Algorithm | PPO with Deep Sets |
| Learning Rate | 3e-4 |
| Batch Size (n_steps) | 2048 |
| Entropy Coefficient | 0.01 |
| Dataset | dataset_v1 (50 train / 5 eval scenarios) |

---

## Results

### Learning Curve

| Checkpoint | Mean Episode Reward |
|------------|---------------------|
| 2K steps | -478.75 (collision) |
| 1.25M steps | +287.48 |
| 2.5M steps | +283.70 |
| 3.75M steps | +389.45 |
| **5M steps** | **+478.26** |

**Observation:** Model was still improving at 5M steps (last 10 readings averaged 480.6).

### Evaluation (20 Episodes)

| Metric | Value |
|--------|-------|
| Success Rate | **80%** (16/20) |
| Avg Reward | 391.48 (+/- 235.0) |
| Avg Episode Length | 818 steps |
| Collisions | 4/20 |

### Failure Analysis

All 4 failures occurred on `eval_001` - a scenario with late vehicle spawn (V002 at 1.85s) creating a close-following situation. The model consistently fails at step 90. Other 4 eval scenarios: 100% success.

---

## Key Findings

1. **Deep Sets architecture works** - Model handles variable peer count correctly
2. **Training converged well** - Loss: 4.93 → -0.03
3. **Policy stabilized** - Entropy decreased, actions became deterministic
4. **Edge case identified** - Late-spawn scenarios need more training coverage

---

## Artifacts

| File | Location |
|------|----------|
| Final Model | `ml/models/runs/cloud_prod_001/model_final.zip` |
| Checkpoints | `ml/models/runs/cloud_prod_001/checkpoints/` (500 files) |
| TensorBoard | `ml/models/runs/cloud_prod_001/tensorboard/` |
| S3 Backup | `s3://saferide-training-results/cloud_prod_001/` |

---

## Next Steps

1. Consider 10M step run to improve edge case handling
2. Port Deep Sets inference to ESP32 firmware (Phase 5)
3. Create EC2 AMI to avoid instance setup overhead
