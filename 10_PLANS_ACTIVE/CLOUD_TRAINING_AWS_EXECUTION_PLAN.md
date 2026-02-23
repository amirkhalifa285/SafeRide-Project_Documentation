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
| cloud_prod_002 | c6i.xlarge (AMI-based, On-Demand) | Feb 21, 2026 | 10M | **TRAINING** |

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

## 📝 Lessons Learned (Feb 20-21, 2026 - AMI Creation + Run 002 Launch)

### Wrong GitHub Repository URL (CRITICAL)
**Problem:** Scripts and docs referenced `amirkhalifa285/SafeRide.git` (old personal repo from Run 001) instead of the actual team repo.

**Root Cause:** Copied URL from stale Run 001 docs without verifying the local git remote.

**Fix:**
- ✅ Correct repo: `roadsense-team/roadsense-v2v.git`
- ❌ Wrong repo: `amirkhalifa285/SafeRide.git`
- **ALWAYS** check `git remote -v` in the local repo before writing cloud scripts.

### Wrong Default Branch Name
**Problem:** Scripts used `git pull origin main` but the repo's default branch is `master`.

**Root Cause:** Assumed `main` without checking. The repo uses `master`.

**Fix:**
- ✅ Correct: `git pull origin master`
- ❌ Wrong: `git pull origin main`
- **ALWAYS** check `git branch -a` before writing scripts that reference branch names.

### Git "Dubious Ownership" Error on AMI Instance
**Problem:** User-data script (runs as root) failed on `git pull` because the repo was cloned by the `ubuntu` user during AMI setup. Git's safe directory check blocked root from operating on it.

**Root Cause:** AMI setup ran as `ubuntu` user, but EC2 user-data scripts run as `root`.

**Fix:** Add to the user-data script BEFORE any git commands:
```bash
git config --global --add safe.directory /home/ubuntu/work
```
**Prevention:** Bake this into the AMI setup script so future AMIs don't have this issue.

### run_docker.sh Permission Denied
**Problem:** `./ml/run_docker.sh` failed with "Permission denied" when run from user-data.

**Root Cause:** Git doesn't always preserve execute permissions across clone/pull, especially when root runs a script owned by another user.

**Fix:** Add to user-data script before calling run_docker.sh:
```bash
chmod +x /home/ubuntu/work/ml/run_docker.sh
```
**Prevention:** Bake `chmod +x` into the user-data template.

### S3 Upload + Shutdown Never Ran (CRITICAL)
**Problem:** Training completed successfully but S3 upload and `shutdown -h now` never executed. Instance kept running (wasting money), results stuck on disk.

**Root Cause:** `set -euo pipefail` at the top of the script. When `run_docker.sh train` exited with a non-zero code (even though training succeeded), bash killed the entire script. Every command after training was skipped.

**Fix:** Two changes to `run_training.sh`:
1. Use a `trap cleanup EXIT` handler that ALWAYS runs S3 upload and shutdown, regardless of how the script exits.
2. Disable `set -e` before the training step (`set +e`), capture the exit code, then re-enable.

### IAM Credentials Not Available
**Problem:** Manual `aws s3 cp` also failed with "Unable to locate credentials".

**Root Cause:** Either the IAM role (RoadSense-Trainer-Role) was not attached at launch, or instance metadata was not accessible.

**Fix:**
- **ALWAYS** verify IAM role is attached: check "IAM instance profile" in the launch wizard.
- Add to pre-launch checklist: verify credentials work with `curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials/`
- As a fallback, `scp` results directly to your laptop if S3 fails.

### Base Scenario Path Mismatch
**Problem:** Script referenced `ml/scenarios/base_01` but the actual directory is `ml/scenarios/base`.

**Root Cause:** CLAUDE.md canonical command used a planned name (`base_01`) that was never created.

**Fix:** Always verify paths exist with `ls` before writing them into scripts.

### AMI Setup Script vs Manual Commands
**Lesson:** The `setup_ami.sh` script had multiple issues (wrong repo, wrong branch) that would have failed silently as user-data. Running setup via SSH manually was the right call - it let us catch and fix errors interactively.

**Recommendation for future AMI rebuilds:**
1. Always run setup interactively via SSH first
2. Only automate via user-data once all commands are proven

---

## ⚠️ Pre-Launch Checklist (USE THIS EVERY TIME)

Before launching any training run, verify ALL of the following:

```
[ ] Code pushed to GitHub (roadsense-team/roadsense-v2v, master branch)
[ ] PAT is valid: test with `git ls-remote https://<PAT>@github.com/roadsense-team/roadsense-v2v.git`
[ ] Base scenario dir exists in repo (check with: ls ml/scenarios/base/)
[ ] Emulator params file exists in repo (check with: ls ml/espnow_emulator/emulator_params_measured.json)
[ ] User-data script includes: git config --global --add safe.directory /home/ubuntu/work
[ ] User-data script includes: chmod +x ml/run_docker.sh (before calling it)
[ ] User-data uses correct repo: roadsense-team/roadsense-v2v.git
[ ] User-data uses correct branch: master (NOT main)
[ ] User-data uses correct base path: ml/scenarios/base (NOT base_01)
[ ] --output_dir uses CONTAINER path: /work/results (NOT /home/ubuntu/work/results)
[ ] Security group SSH rule uses IP from: curl -4 ifconfig.me
[ ] IAM role attached: RoadSense-Trainer-Role
[ ] After launch, verify IAM creds: curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials/
[ ] Script uses trap cleanup EXIT (NOT set -euo pipefail for the whole script)
[ ] Training step uses set +e to avoid killing S3 upload and shutdown
```

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
