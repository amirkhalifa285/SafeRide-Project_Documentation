# ML Docker Environment Guide

**Created:** January 13, 2026
**Last Updated:** January 15, 2026
**Purpose:** Document the Docker-based ML training environment for ConvoyEnv.

---

## Overview

The ML training environment runs in a **single Docker container** that includes:
- SUMO 1.25.0+ (traffic simulation)
- Python 3.10 with virtual environment
- Gymnasium, stable-baselines3, PyTorch
- ESP-NOW Emulator

### Why Single Container?

TraCI (SUMO's Python API) uses **synchronous TCP communication**. In a two-container setup, every `step()` call adds ~2ms network overhead. For RL training with millions of steps, this compounds to hours of wasted time.

Single container = zero network overhead = faster training.

---

## Files

```
ml/
├── Dockerfile           # Image definition
├── run_docker.sh        # Cross-platform helper script
├── demo_convoy_gui.py   # GUI visualization demo
├── .dockerignore        # Build optimization
└── requirements.txt     # Python dependencies
```

---

## Quick Start

```bash
cd roadsense-v2v/ml

# Build (first time, ~4 minutes)
docker build -t roadsense-ml:latest .

# Run unit tests (headless, works everywhere)
./run_docker.sh test

# Run GUI demo (requires display)
./run_docker.sh demo
```

### Smoke Training (Fast)

For quick verification runs, set a small rollout length so PPO doesn't
buffer a full 2048-step rollout:

```bash
./run_docker.sh train --sumo_cfg ml/scenarios/base/scenario.sumocfg \
    --total_timesteps 100 --n_steps 32 --eval_episodes 1
```

---

## Platform-Specific GUI Setup

### Windows 11 + WSL2 (Recommended)

WSLg provides automatic X11/Wayland support. Just run:

```bash
./run_docker.sh demo
```

### Windows 10 + WSL2

Requires VcXsrv:

1. Install VcXsrv from https://sourceforge.net/projects/vcxsrv/
2. Run XLaunch → check "Disable access control"
3. In WSL2:
   ```bash
   export DISPLAY=$(cat /etc/resolv.conf | grep nameserver | awk '{print $2}'):0
   ./run_docker.sh demo
   ```

### Linux + X11

```bash
xhost +local:docker
./run_docker.sh demo
```

### Linux + Wayland (Fedora, etc.)

**Known Issue:** SUMO-GUI does not work from Docker on Wayland.

Error: `FXApp::openDisplay: unable to open display :0`

Attempted fixes (none worked):
- `xhost +local:docker` / `xhost +local:` / `xhost +`
- Passing XAUTHORITY
- `--network=host`

**Workaround Options:**
1. Use headless mode for training (works fine)
2. Run visualization on Windows WSL2 instead
3. Future: Add VNC server inside container

### macOS

Requires XQuartz:

1. `brew install --cask xquartz`
2. Open XQuartz → Preferences → Security → Allow network clients
3. Restart, then: `xhost +localhost`
4. `./run_docker.sh demo`

---

## run_docker.sh Commands

| Command | Description |
|---------|-------------|
| `./run_docker.sh` | Run unit tests (default) |
| `./run_docker.sh test` | Run unit tests |
| `./run_docker.sh integration` | Run integration tests with SUMO |
| `./run_docker.sh demo` | GUI visualization demo |
| `./run_docker.sh gui` | Interactive shell with GUI support |
| `./run_docker.sh bash` | Interactive shell (headless) |
| `./run_docker.sh train` | Run RL training |
| `./run_docker.sh help` | Show all options |

---

## Docker Image Details

**Base Image:** `ghcr.io/eclipse-sumo/sumo:main`
- Debian-based
- SUMO 1.25.0+ pre-installed
- ~4.7GB

**Added by Dockerfile:**
- Python virtual environment at `/opt/venv`
- gymnasium, stable-baselines3, PyTorch (~800MB)
- tensorboard (required for training logs)
- numpy, scipy, matplotlib, pytest

**Final Image Size:** ~6-7GB

---

## CI/CD Compatibility

This setup works with:
- GitHub Actions
- GitLab CI
- AWS SageMaker / ECR
- GCP Vertex AI / GCR
- Azure ML / ACR
- Kubernetes
- Any container-based CI system

Typical pipeline:
```
Push → Build Image → Push to Registry → Run Training → Save Model
```

---

## Troubleshooting

### Build fails with "Cannot uninstall sympy"

The Dockerfile uses a Python virtual environment to avoid conflicts with system packages. If you see this error, the Dockerfile was not updated. Ensure it creates `/opt/venv`.

### GUI shows "unable to open display"

See platform-specific instructions above. On Wayland (Fedora), GUI doesn't work - use headless mode or Windows WSL2.

### Tests fail with "SUMO not installed"

Integration tests require the Docker container. Run via:
```bash
./run_docker.sh integration
```

### Training fails with "tensorboard is not installed"

`train_convoy.py` logs to TensorBoard by default. Ensure `tensorboard` is included
in `ml/requirements.txt` and rebuild the Docker image.

### Training fails with "Vehicle 'V001' is not known"

SUMO ignores unsorted route files and can drop early-departing vehicles (often V001).
If you see a warning like `Route file should be sorted by departure time`, regenerate
the dataset with sorted routes (the generator now enforces this) and retry training.

---

## Architecture Decision Record

**Decision:** Single container with SUMO + Python ML stack
**Date:** January 13, 2026
**Status:** Accepted

**Context:** Need to run ConvoyEnv for RL training. Options were:
1. Single container (all-in-one)
2. Two containers (SUMO separate from Python)
3. SUMO on host, Python in container

**Decision:** Single container

**Rationale:**
- Zero latency for TraCI calls (critical for RL)
- Simpler deployment (one image)
- Works with any CI/CD system
- GPU can be added later by changing base image

**Consequences:**
- Larger image (~7GB)
- Must rebuild if SUMO version changes
- GUI requires X11 forwarding setup
