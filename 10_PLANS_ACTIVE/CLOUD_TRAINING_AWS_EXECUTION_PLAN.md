# AWS Cloud Training Execution Plan (RoadSense V2V)

**Status:** PLANNED
**Goal:** Run full RL training (5M+ steps) on AWS while maintaining a strict $20/month budget.
**Architect:** Master Architect (Interactive CLI)
**Last Updated:** January 15, 2026

---

## 🚀 Strategy Overview
We will utilize **AWS EC2 Spot Instances** to access high-performance compute at ~70-90% discount. To ensure cost-control, we use a "Single-Job" lifecycle: the instance spins up, clones the code, trains the model, uploads results to S3, and immediately terminates itself.

### Budget Guardrails
- **Total Limit:** $20.00 / Month.
- **Projected Cost:** ~$11.00 (100 hours of `c6i.xlarge` Spot @ $0.06/hr + storage).
- **Safety Alert:** AWS Budget set to $15.00 with email notification.
- **Auto-Stop:** The training script triggers `shutdown -h now` upon completion or fatal error.

---

## 🛠️ Infrastructure Choices
| Component | Choice | Rationale |
|-----------|--------|-----------|
| **Compute** | `c6i.xlarge` (Spot) | 4 vCPU, 8GB RAM. Optimized for SUMO + PyTorch. |
| **Storage** | 50GB gp3 EBS | Sufficient for Docker image and local datasets. |
| **Authentication**| Private GitHub (Personal) | Bypass Org-level SSH complexities using PAT. |
| **Data Storage** | Amazon S3 | Durable storage for models and `metrics.json`. |

---

## 📋 Step-by-Step Workflow

### Phase 1: Local Preparation & Validation
Before spending cloud budget, we must ensure the dataset and training logic are sound.
1.  **Generate Dataset:**
    - Run `./run_docker.sh generate` to create 100 variations (80 train / 20 eval).
    - Seed: 42.
2.  **Visual Validation:**
    - Manually open 3-5 random generated scenarios in `sumo-gui`.
    - Verify V001 spawns at t=0 and traffic flow is logical.
3.  **Migration:**
    - Initialize a private repository on personal GitHub.
    - Push the entire `roadsense-v2v` folder (to preserve Docker paths and dependencies).

### Phase 2: AWS Provisioning
1.  **S3 Bucket:** Create `roadsense-training-results`.
2.  **IAM Role:** Create `RoadSense-Trainer-Role` with `S3FullAccess`.
3.  **GitHub PAT:** Generate a Personal Access Token with `repo` scope.

### Phase 3: The Training Run
Launch an EC2 Spot Instance with the following **User Data** script:

```bash
#!/bin/bash
# 1. Environment Setup
apt-get update && apt-get install -y docker.io awscli
systemctl start docker

# 2. Code Acquisition
# Replace <PAT> with your GitHub token
git clone https://<PAT>@github.com/amirkhalifa/roadsense-v2v.git /home/ubuntu/work
cd /home/ubuntu/work/ml

# 3. Training Execution
./run_docker.sh train \
    --dataset_dir ml/scenarios/datasets/dataset_v1 \
    --emulator_params ml/espnow_emulator/emulator_params_5m.json \
    --total_timesteps 5000000 \
    --run_id cloud_prod_001 \
    --output_dir /home/ubuntu/work/results

# 4. Artifact Archival
aws s3 cp /home/ubuntu/work/results s3://roadsense-training-results/cloud_prod_001 --recursive

# 5. Self-Termination
shutdown -h now
```

---

## 📈 Training Specifications
- **Timesteps:** 5,000,000 (Expected duration: 18-24 hours).
- **Architecture:** Deep Sets n-element (Permutation Invariant).
- **Emulator:** 5m RTT Characterization Params (`emulator_params_5m.json`).
- **Metrics:** `metrics.json` must include collision rate, reward trend, and scenario list.

---

## ⚠️ Risks & Mitigations
- **Spot Interruption:** AWS may reclaim the instance.
  - *Mitigation:* PPO saves checkpoints to `ml/models/runs/<run_id>/checkpoints` every 10k steps. These are synced to S3 if the script handles signals. (Phase 2 enhancement).
- **Budget Overrun:** Forgetting to terminate.
  - *Mitigation:* `shutdown -h now` is hardcoded in the script. AWS Budgets will alert at 80% of $15.
- **Git Auth Failure:** Token expires.
  - *Mitigation:* Test clone command locally with the PAT before cloud launch.
