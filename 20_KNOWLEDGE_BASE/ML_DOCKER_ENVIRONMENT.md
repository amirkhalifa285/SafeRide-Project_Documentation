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
├── run_demo_gui.sh      # Fedora/Wayland PoC demo launcher (Run 025)
├── demo_convoy_gui.py   # SUMO GUI demo (loads trained model)
├── demo_replay_plot.py  # Real-data replay visualization (matplotlib)
├── scenarios/demo_poc/  # Dedicated 3-vehicle PoC demo scenario
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

The default `./run_docker.sh demo` path is not sufficient on Fedora Wayland. Use the manual Docker command below instead.

```bash
SCENARIO_DIR=/absolute/path/to/scenario_dir
XAUTH_FILE="${XAUTHORITY:-$(find /run/user/$(id -u) -maxdepth 1 -name '.mutter-Xwaylandauth.*' | head -n1)}"
if [ -z "$XAUTH_FILE" ]; then
  echo "No Xwayland auth file found"
  false
fi

xhost +local:
docker run --rm -it \
  --security-opt label=disable \
  --user "$(id -u):$(id -g)" \
  -e DISPLAY="$DISPLAY" \
  -e QT_X11_NO_MITSHM=1 \
  -e XAUTHORITY=/tmp/.Xauthority \
  -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
  -v "$XAUTH_FILE:/tmp/.Xauthority:ro" \
  -v "$SCENARIO_DIR:/data:Z" \
  -w /data \
  ghcr.io/eclipse-sumo/sumo:main \
  sumo-gui -c scenario.sumocfg
xhost -local:
```

If you see `Could not build output file 'fcd_output.csv' Permission denied`, remove stale root-owned outputs once:

```bash
rm -f "$SCENARIO_DIR/fcd_output.csv" "$SCENARIO_DIR/ssm_output.xml"
```

`./run_docker.sh demo` still uses the older Wayland path, so do not rely on it for Fedora until that script is updated.

### Fedora/Wayland: Dedicated PoC Demo Launcher (Recommended)

A dedicated launcher script `ml/run_demo_gui.sh` handles all Fedora/Wayland/SELinux quirks automatically:

```bash
cd roadsense-v2v

# Default: loads Run 025 500k model, demo_poc scenario
./ml/run_demo_gui.sh

# Custom scenario or delay
./ml/run_demo_gui.sh --scenario base_real --delay 0.2

# Random actions (no model, for testing)
./ml/run_demo_gui.sh --no-model
```

This script:
- Auto-detects Wayland vs X11 and sets up `XAUTHORITY`, `xhost`, SELinux overrides
- Runs as `--user $(id -u):$(id -g)` to avoid root-owned file issues
- Defaults to the Run 025 PoC model (`replay_ft_500000_steps.zip`)
- Passes all extra args through to `demo_convoy_gui.py`
- Cleans up `xhost` permissions on exit

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
| `./ml/run_demo_gui.sh` | PoC demo with Run 025 model (Fedora/Wayland) |

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

See platform-specific instructions above. On Fedora Wayland, use the manual Docker command with `XAUTHORITY`, `--security-opt label=disable`, and `--user "$(id -u):$(id -g)"`.

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
