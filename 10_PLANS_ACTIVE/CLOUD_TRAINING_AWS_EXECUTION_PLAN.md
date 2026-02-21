# AWS Cloud Training Execution Plan (RoadSense V2V)

**Status:** ACTIVE - Preparing Run 002
**Goal:** Run full RL training on AWS while maintaining a strict $20/month budget.
**Architect:** Master Architect (Interactive CLI)
**Last Updated:** February 20, 2026

---

## 🎯 Run Status

| Run ID | Instance | Started | Timesteps | Status |
|--------|----------|---------|-----------|--------|
| cloud_prod_001 | c6i.xlarge (On-Demand) | Jan 16, 2026 | 5M | **COMPLETED** (80% eval, n=2 only) |
| cloud_prod_002 | c6i.xlarge (AMI-based) | TBD | 10M | **NEXT** |

**Run 002 results location:** `s3://saferide-training-results/cloud_prod_002/`

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
set -euo pipefail
exec > /var/log/user-data.log 2>&1

export DEBIAN_FRONTEND=noninteractive
export AWS_DEFAULT_REGION=il-central-1

# 1. Environment Setup
apt-get update && apt-get install -y docker.io awscli git
systemctl enable --now docker

# 2. Code Acquisition (replace <PAT> locally before pasting)
git clone https://<PAT>@github.com/roadsense-team/roadsense-v2v.git /home/ubuntu/work
cd /home/ubuntu/work/ml

# 3. Dataset Generation
./run_docker.sh generate \
    --base_dir ml/scenarios/base \
    --output_dir ml/scenarios/datasets/dataset_v1 \
    --seed 42 --train_count 80 --eval_count 20 \
    --emulator_params ml/espnow_emulator/emulator_params_5m.json

# 4. Training Execution
# CRITICAL: --output_dir uses CONTAINER path (/work/...), not host path
./run_docker.sh train \
    --dataset_dir ml/scenarios/datasets/dataset_v1 \
    --emulator_params ml/espnow_emulator/emulator_params_5m.json \
    --total_timesteps 5000000 \
    --run_id cloud_prod_001 \
    --output_dir /work/results \
    --skip_eval

# 5. Artifact Archival (uses HOST path - maps to container's /work/results)
aws s3 cp /home/ubuntu/work/results s3://saferide-training-results/cloud_prod_001 --recursive

# 6. Self-Termination
shutdown -h now
```

> **Path Mapping Note:** The `run_docker.sh` script mounts host `/home/ubuntu/work` to container `/work`. Arguments passed to Python scripts run INSIDE the container, so use container paths for `--output_dir`. The S3 upload runs OUTSIDE the container, so use host paths.

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
- **SSH Access Issues:** AWS "My IP" auto-detection often fails.
  - *Mitigation:* Use `curl -4 ifconfig.me` to get actual public IP for security group rules.

---

## 📝 Lessons Learned (Jan 16, 2026 - First Run)

### Docker Path Mapping Bug (CRITICAL)
**Problem:** Initial script used `--output_dir /home/ubuntu/work/results` (host path) for training, but training runs INSIDE the Docker container where that path doesn't exist.

**Root Cause:** `run_docker.sh` mounts `$REPO_ROOT:/work`, so container sees `/work` as root, not `/home/ubuntu/work`.

**Fix:** Use container paths for arguments passed to Python scripts:
- ❌ `--output_dir /home/ubuntu/work/results`
- ✅ `--output_dir /work/results`

### SSH Security Group
**Problem:** AWS Console "My IP" auto-detection returned wrong IP (due to VPN/CGNAT/proxy).

**Fix:** Manually determine IP with `curl -4 ifconfig.me` and use that exact IP/32 in security group.

### Instance Type Decision
**Decision:** Used On-Demand instead of Spot for first run.

**Rationale:** Spot can be interrupted mid-training. For a critical first validation run, On-Demand (~$0.20/hr) is worth the extra cost for reliability. Consider Spot for subsequent runs where interruption is acceptable.

---

## 🚀 Run 002: Production Model (Synthetic Base)

### What Changed from Run 001

| Aspect | Run 001 | Run 002 |
|--------|---------|---------|
| Timesteps | 5M | **10M** |
| Dataset | dataset_v1 (constant n=2) | **dataset_v2 (variable n={1..5})** |
| Emulator params | emulator_params_5m.json | **emulator_params_measured.json** (real RTT drive) |
| Launch method | Raw user-data (15 min setup) | **AMI-based (~2 min setup)** |
| Augmentation | Speed/decel variation only | **+ peer dropout + route randomization** |

### Launch Procedure (AMI-Based)

1. **Pre-flight:** Push latest code to GitHub (SafeRide repo)
2. **Launch:** EC2 console -> Launch from `roadsense-training-v1` AMI
   - Instance: c6i.xlarge, il-central-1
   - IAM role: RoadSense-Trainer-Role
   - User data: paste contents of `ml/scripts/cloud/run_training.sh` (with PAT filled in)
   - Security group: SSH from your IP only
3. **Monitor (optional):** SSH in, `tail -f /var/log/training-run.log`
4. **Results:** Auto-uploaded to `s3://saferide-training-results/cloud_prod_002/`
5. **Instance self-terminates** after training + upload

### Training Config

```
Algorithm:        PPO with Deep Sets
Total Timesteps:  10,000,000 (~48h on c6i.xlarge)
Learning Rate:    3e-4
Batch Size:       2048
Entropy Coeff:    0.01
Dataset:          dataset_v2/base_01 (16 train / 4 eval, variable n)
Emulator:         emulator_params_measured.json (Feb 20 RTT drive)
```

### Cost Estimate

- c6i.xlarge On-Demand: ~$0.20/hr x 48h = **~$10**
- S3 storage: negligible
- AMI storage: ~$0.50/month
- **Total: ~$11** (within $20 budget)
