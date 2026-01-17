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

## ⚠️ POST-MORTEM (Jan 17, 2026): Fixed n=2 Training Data

### Issue Discovered

**All 25 scenarios in dataset_v1 contained exactly 3 vehicles (V001 + V002 + V003).**

This means the model was trained with **constant n=2 peers** for the entire 5M timesteps. The Deep Sets architecture is designed to handle variable n, but it never experienced:
- n=0 (isolated driving)
- n=1 (single peer)
- n=3+ (multiple peers)

### Impact Assessment

| Aspect | Status |
|--------|--------|
| Architecture correctness | ✅ Valid - Deep Sets handles variable n |
| Permutation invariance | ✅ Valid - Max pooling works for any n |
| 80% success rate | ⚠️ Only valid for n=2 scenarios |
| n≠2 behavior | ❓ Unknown and untested |

### Root Cause

The `monitored_vehicles` field in `emulator_params_5m.json` was set to `["V002", "V003"]`. While this field is **NOT used by ConvoyEnv** (it uses `sync_and_get_messages()` which handles all peers dynamically), the **SUMO scenarios themselves** only defined 3 vehicles.

### Resolution

1. Removed `monitored_vehicles` from emulator params JSON files (misleading, unused)
2. Added Phase 6 to `N_ELEMENT_IMPLEMENTATION_PLAN.md` with variable-n requirements
3. **Run 002 must use dataset_v2 with n ∈ {1,2,3,4,5} peers**

### Lesson Learned

> **Training data diversity is as important as architecture correctness.**
> Deep Sets can handle variable n, but only if trained with variable n.

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

1. **Generate dataset_v2** with variable peer counts n ∈ {1,2,3,4,5}
2. **Run 002:** 10M timesteps with variable n dataset + evaluation
3. Port Deep Sets inference to ESP32 firmware (Phase 5)
4. Create EC2 AMI to avoid instance setup overhead

**See:** `N_ELEMENT_IMPLEMENTATION_PLAN.md` Phase 6 for detailed requirements.
