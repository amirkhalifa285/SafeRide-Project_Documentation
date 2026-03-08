# Proposal: Automated Cloud Training & Scenario Augmentation Pipeline

**Target Audience:** System Architect / Planner
**Status:** DRAFT (For Review)
**Date:** 2026-01-15
**Goal:** Scale the RL training from a single laptop to AWS infrastructure, incorporating automated scenario augmentation (Domain Randomization) to improve model robustness.

---

## 1. Pipeline Overview

The proposed pipeline decouples **Scenario Generation** (creating variation files) from **Model Training** (consuming them). Both stages utilize the existing Docker container to ensure Simulation-to-Training consistency.

```mermaid
graph LR
    A[Base Scenarios (x5)] -->|Augmentation Script (Docker)| B[Generated Dataset (x100)]
    B -->|Commit/S3| C[AWS Cloud Storage]
    C -->|Mount| D[AWS Compute (EC2/SageMaker)]
    D -->|Train| E[Trained Model (TFLite)]
```

---

## 2. Scenario Augmentation Strategy

We will transform the 5 "Base" scenarios into a robust dataset of 100+ variations to prevent overfitting.

### 2.1 The Generator Script (`ml/scripts/gen_scenarios.py`)
This script will run **inside the Docker container**. It utilizes `sumolib` to parse existing XML files, modify parameters, and write new scenario folders.

**Variations to Apply:**
1.  **Traffic Density:** Vary flow probability (`vehsPerHour`) by ±30%.
2.  **Driver Behavior (Sigma):** Vary driver imperfection (`0.0` perfect to `0.5` erratic).
3.  **Speed Distributions:** Vary `speedFactor` (e.g., fast/aggressive traffic vs. slow/cautious).
4.  **Spawn Timing:** Randomize start times to prevent "memory" of specific vehicle positions.

### 2.2 Directory Structure
The script will output a structured dataset:
```text
ml/scenarios/
├── base/                   # Original manually crafted scenarios
│   ├── highway_fast/
│   └── urban_intersection/
└── dataset_v1/             # Generated (Git-ignored or S3-backed)
    ├── train/
    │   ├── scenario_001_highway_dense_aggressive/
    │   ├── scenario_002_highway_sparse_cautious/
    │   └── ... (80 folders)
    └── eval/
        └── ... (20 folders)
```

---

## 3. AWS Infrastructure Options

We propose two paths. **Recommendation: Start with Option A (EC2), then migrate to Option B (SageMaker) for final tuning.**

### Option A: AWS EC2 (Compute Optimized)
**"The Remote Workstation Approach"**

- **Hardware:** `c6i.8xlarge` (32 vCPU, 64GB RAM).
- **Workflow:**
    1.  Provision Instance (Ubuntu AMI).
    2.  Install Docker.
    3.  Pull `roadsense-ml` image from ECR (or build from git).
    4.  Run `./run_docker.sh train` inside `tmux`.
- **Pros:**
    -   Identical workflow to local development.
    -   Full interactive shell access (htop, logs, debugging).
    -   Easy to mount local volumes for quick iteration.
- **Cons:**
    -   Manual lifecycle management (must remember to stop instance).
    -   Harder to run simultaneous experiments (need to manage multiple instances).

### Option B: AWS SageMaker (Training Jobs)
**"The MLOps Approach"**

- **Hardware:** `ml.c5.9xlarge`.
- **Workflow:**
    1.  Push Docker image to Amazon ECR.
    2.  Upload `dataset_v1` to Amazon S3.
    3.  Trigger generic Estimator job pointing to Image + S3 Data.
- **Pros:**
    -   **Serverless:** Pay only for the exact seconds of training.
    -   **Hyperparameter Tuning:** Can automatically run 50 jobs to find the best Learning Rate.
    -   **Versioning:** Every model is linked to specific data/code in S3.
- **Cons:**
    -   "Black Box" debugging: If it crashes, you must dig through CloudWatch logs.
    -   Slightly higher setup complexity (IAM roles, ECR push scripts).

---

## 4. Required Code Changes

To support this pipeline, we need to modify the ML codebase to handle "Domain Randomization" (loading random scenarios per episode).

### 4.1 `ml/envs/convoy_env.py`
**Current:** Accepts a single `sumo_cfg` path string.
**Proposed Change:**
-   Update `__init__` to accept `scenario_list: List[str]` or `scenario_dir: str`.
-   In `reset()`:
    ```python
    def reset(self, seed=None, options=None):
        # Randomly select a new config for this episode
        selected_cfg = self.rng.choice(self.scenario_list)
        self.sumo_connection.reload(selected_cfg)  # Need to implement reload capability
        ...
    ```

### 4.2 `ml/envs/sumo_connection.py`
**Current:** Starts SUMO once in `__init__`.
**Proposed Change:**
-   Add `reload(config_path)` method.
-   This method must tear down the existing TraCI connection and start a new one with the new config file (SUMO requires a restart to change network/config).

### 4.3 `ml/training/train_convoy.py`
**Current:** Hardcodes path to `scenarios/base/scenario.sumocfg`.
**Proposed Change:**
-   Add Argument Parser.
-   Allow passing `--dataset_dir` path.
-   Glob all `.sumocfg` files in that directory and pass list to `ConvoyEnv`.

---

## 5. Next Steps for Approval

1.  **Approve Architecture:** Confirm preference for EC2 (Dev) vs SageMaker (Prod).
2.  **Approve Code Scope:** Authorize modification of `ConvoyEnv` to support multi-scenario reloading (this is the most complex code change).
3.  **Approve Tooling:** Authorize creation of `ml/scripts/gen_scenarios.py`.
