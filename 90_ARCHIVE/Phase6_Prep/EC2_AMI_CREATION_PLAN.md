# EC2 AMI Creation Plan

**Status:** COMPLETED
**Priority:** HIGH - Current Session
**Created:** January 16, 2026
**Updated:** February 20, 2026

---

## Problem

Each cloud training run requires ~10-15 min of setup (install Docker, clone repo, build Docker image) before training starts. With multiple runs planned (Run 002 synthetic + real-data comparison), this overhead adds up.

## Solution

Create a pre-configured AMI with Docker + `roadsense-ml:latest` image pre-built. Future runs launch in ~2 min (just git pull + go).

### What's Baked Into the AMI

| Baked (slow, rarely changes)        | Runtime (fast, changes per run)       |
|-------------------------------------|---------------------------------------|
| Docker installed + enabled          | `git pull` latest code                |
| AWS CLI                             | Dataset generation (params vary)      |
| `roadsense-ml:latest` Docker image  | Training execution (config varies)    |
| Git installed                       | S3 upload + self-terminate            |
| Repo cloned to `/home/ubuntu/work`  |                                       |

## Steps

- [x] Launch c6i.xlarge from Ubuntu 22.04 in il-central-1
- [x] SSH in and run setup manually (setup_ami.sh had branch name issue, ran commands directly)
- [x] Verify Docker image + SUMO version + ML stack
- [x] Create AMI from the instance
- [x] Document AMI ID below
- [ ] Test: launch new instance from AMI, verify `run_docker.sh test` passes
- [x] Terminate builder instance

## Scripts

| Script | Purpose |
|--------|---------|
| `ml/scripts/cloud/setup_ami.sh` | One-time AMI setup (run via SSH) |
| `ml/scripts/cloud/run_training.sh` | Runtime user-data template for training runs |

### AMI Setup (SSH)
```bash
# SCP or paste, then run:
bash setup_ami.sh <YOUR_GITHUB_PAT>
```

### Launch Training Run (EC2 User Data)
```bash
# Edit run_training.sh: set RUN_ID, GITHUB_PAT, TOTAL_STEPS
# Paste into EC2 User Data when launching from the AMI
```

## AMI Details (Fill after creation)

| Field | Value |
|-------|-------|
| AMI ID | ami-03a3037588b0f34f2 |
| AMI Name | roadsense-training-v1 |
| Region | il-central-1 |
| Base AMI | Ubuntu 22.04 LTS |
| Instance type used | c6i.xlarge |
| EBS | 50GB gp3 |
| Created | February 20, 2026 |

## Security

- `setup_ami.sh` automatically cleans the GitHub PAT from the git remote before snapshot
- `run_training.sh` sets the PAT at runtime, uses it for pull, then immediately clears it
- The AMI itself contains NO credentials

## Benefits

- Launch-to-training: ~2 min (vs ~15 min from scratch)
- Consistent environment across runs
- Reduced setup errors
- Lower cost (less idle time during setup)
- Enables quick iteration for comparison runs (synthetic vs real-data)
