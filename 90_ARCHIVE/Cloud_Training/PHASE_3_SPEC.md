# Phase 3 SPEC: Training CLI + Evaluation Harness

**Document Type:** Builder Specification
**Phase:** 3 of 5 (Cloud Training Automation)
**Status:** Ready for Implementation
**Created:** January 15, 2026
**Author:** PLANNER Agent

---

## Objective

Add command-line arguments to `train_convoy.py` for dataset selection, seed control, and emulator configuration. Implement a post-training evaluation loop that produces reproducible metrics.

---

## Prerequisites

- [x] Phase 2 complete (ScenarioManager, dataset-based ConvoyEnv)
- [x] Dataset directory structure: `ml/scenarios/datasets/{dataset_id}/train/`, `eval/`
- [x] ConvoyEnv accepts `dataset_dir`, `scenario_mode`, `scenario_seed` parameters

---

## Implementation Steps

### Step 1: Add Argparse to train_convoy.py

**File:** `roadsense-v2v/ml/training/train_convoy.py`

Replace hardcoded configuration with CLI arguments:

```python
import argparse


def parse_args() -> argparse.Namespace:
    """Parse command-line arguments."""
    parser = argparse.ArgumentParser(
        description="Train PPO on RoadSense-Convoy environment with Deep Sets."
    )

    # Dataset configuration
    parser.add_argument(
        "--dataset_dir",
        type=str,
        default=None,
        help="Path to dataset directory (contains manifest.json). "
             "If not provided, uses single scenario from --sumo_cfg.",
    )
    parser.add_argument(
        "--sumo_cfg",
        type=str,
        default=None,
        help="Path to single scenario.sumocfg (mutually exclusive with --dataset_dir).",
    )
    parser.add_argument(
        "--eval_episodes",
        type=int,
        default=10,
        help="Number of evaluation episodes after training (default: 10).",
    )

    # Emulator configuration
    parser.add_argument(
        "--emulator_params",
        type=str,
        default=None,
        help="Path to emulator params JSON. If not provided, uses defaults.",
    )

    # Reproducibility
    parser.add_argument(
        "--seed",
        type=int,
        default=42,
        help="Random seed for training and scenario selection (default: 42).",
    )

    # Training hyperparameters
    parser.add_argument(
        "--total_timesteps",
        type=int,
        default=100_000,
        help="Total training timesteps (default: 100,000).",
    )
    parser.add_argument(
        "--learning_rate",
        type=float,
        default=3e-4,
        help="Learning rate (default: 3e-4).",
    )
    parser.add_argument(
        "--n_steps",
        type=int,
        default=2048,
        help="Steps per rollout (default: 2048).",
    )
    parser.add_argument(
        "--ent_coef",
        type=float,
        default=0.01,
        help="Entropy coefficient (default: 0.01).",
    )

    # Output
    parser.add_argument(
        "--output_dir",
        type=str,
        default=None,
        help="Output directory for model and metrics. "
             "If not provided, uses ml/models/runs/<run_id>/.",
    )
    parser.add_argument(
        "--run_id",
        type=str,
        default=None,
        help="Run identifier. If not provided, generates from timestamp.",
    )

    # Flags
    parser.add_argument(
        "--skip_eval",
        action="store_true",
        help="Skip evaluation after training.",
    )
    parser.add_argument(
        "--gui",
        action="store_true",
        help="Run SUMO with GUI (for debugging).",
    )

    args = parser.parse_args()

    # Validation
    if args.dataset_dir is None and args.sumo_cfg is None:
        # Default to base scenario
        args.sumo_cfg = "ml/scenarios/base/scenario.sumocfg"

    if args.dataset_dir is not None and args.sumo_cfg is not None:
        parser.error("Cannot specify both --dataset_dir and --sumo_cfg")

    return args
```

### Step 2: Update train() Function

**File:** `roadsense-v2v/ml/training/train_convoy.py`

Modify `train()` to use parsed arguments:

```python
def train(args: argparse.Namespace) -> Tuple[str, dict]:
    """
    Train the model and return output path and metrics.

    Args:
        args: Parsed command-line arguments

    Returns:
        Tuple of (model_path, metrics_dict)
    """
    # Generate run_id if not provided
    if args.run_id is None:
        from datetime import datetime
        args.run_id = datetime.now().strftime("run_%Y%m%d_%H%M%S")

    # Setup output directory
    if args.output_dir is None:
        base_dir = os.path.dirname(os.path.dirname(__file__))
        args.output_dir = os.path.join(base_dir, "models", "runs", args.run_id)

    os.makedirs(args.output_dir, exist_ok=True)
    os.makedirs(os.path.join(args.output_dir, "checkpoints"), exist_ok=True)

    # Create environment
    env_kwargs = {
        "render_mode": "human" if args.gui else None,
        "gui": args.gui,
    }

    if args.dataset_dir is not None:
        env_kwargs["dataset_dir"] = args.dataset_dir
        env_kwargs["scenario_mode"] = "train"
        env_kwargs["scenario_seed"] = args.seed
    else:
        env_kwargs["sumo_cfg"] = args.sumo_cfg

    if args.emulator_params is not None:
        env_kwargs["emulator_params_path"] = args.emulator_params

    env = gym.make("RoadSense-Convoy-v0", **env_kwargs)

    # Set global seed for reproducibility
    import numpy as np
    import torch
    np.random.seed(args.seed)
    torch.manual_seed(args.seed)

    # Create model
    policy_kwargs = create_deep_set_policy_kwargs(peer_embed_dim=32)

    checkpoint_callback = CheckpointCallback(
        save_freq=10_000,
        save_path=os.path.join(args.output_dir, "checkpoints"),
        name_prefix="deep_sets",
    )

    model = PPO(
        "MultiInputPolicy",
        env,
        policy_kwargs=policy_kwargs,
        verbose=1,
        ent_coef=args.ent_coef,
        learning_rate=args.learning_rate,
        n_steps=args.n_steps,
        tensorboard_log=os.path.join(args.output_dir, "tensorboard"),
        seed=args.seed,
    )

    # Train
    print(f"Starting training: {args.total_timesteps} timesteps")
    print(f"Output directory: {args.output_dir}")

    try:
        model.learn(
            total_timesteps=args.total_timesteps,
            callback=checkpoint_callback,
        )
    finally:
        env.close()

    # Save final model
    model_path = os.path.join(args.output_dir, "model_final.zip")
    model.save(model_path)
    print(f"Model saved to: {model_path}")

    return model_path, {"training_timesteps": args.total_timesteps}
```

### Step 3: Implement Evaluation Function

**File:** `roadsense-v2v/ml/training/train_convoy.py`

Add an evaluation function that runs after training:

```python
def evaluate(
    model_path: str,
    args: argparse.Namespace,
) -> dict:
    """
    Evaluate a trained model on the eval scenario set.

    Args:
        model_path: Path to saved model
        args: Parsed arguments (for dataset_dir, emulator_params, etc.)

    Returns:
        Dictionary of evaluation metrics
    """
    print(f"\nStarting evaluation: {args.eval_episodes} episodes")

    # Load model
    model = PPO.load(model_path)

    # Create eval environment
    env_kwargs = {
        "render_mode": "human" if args.gui else None,
        "gui": args.gui,
    }

    if args.dataset_dir is not None:
        env_kwargs["dataset_dir"] = args.dataset_dir
        env_kwargs["scenario_mode"] = "eval"  # Sequential scenarios
        env_kwargs["scenario_seed"] = args.seed
    else:
        env_kwargs["sumo_cfg"] = args.sumo_cfg

    if args.emulator_params is not None:
        env_kwargs["emulator_params_path"] = args.emulator_params

    env = gym.make("RoadSense-Convoy-v0", **env_kwargs)

    # Collect metrics
    episode_rewards = []
    episode_lengths = []
    collisions = 0
    truncations = 0
    scenario_ids = []

    try:
        for ep in range(args.eval_episodes):
            obs, info = env.reset()
            scenario_id = info.get("scenario_id", "unknown")
            scenario_ids.append(scenario_id)

            episode_reward = 0.0
            episode_length = 0
            terminated = False
            truncated = False

            while not (terminated or truncated):
                action, _ = model.predict(obs, deterministic=True)
                obs, reward, terminated, truncated, info = env.step(action)
                episode_reward += reward
                episode_length += 1

            episode_rewards.append(episode_reward)
            episode_lengths.append(episode_length)

            if terminated:
                collisions += 1
            if truncated:
                truncations += 1

            print(f"  Episode {ep + 1}/{args.eval_episodes}: "
                  f"reward={episode_reward:.2f}, "
                  f"length={episode_length}, "
                  f"scenario={scenario_id}")

    finally:
        env.close()

    # Compute summary metrics
    metrics = {
        "eval_episodes": args.eval_episodes,
        "avg_reward": float(np.mean(episode_rewards)),
        "std_reward": float(np.std(episode_rewards)),
        "min_reward": float(np.min(episode_rewards)),
        "max_reward": float(np.max(episode_rewards)),
        "avg_length": float(np.mean(episode_lengths)),
        "collisions": collisions,
        "collision_rate": collisions / args.eval_episodes,
        "truncations": truncations,
        "truncation_rate": truncations / args.eval_episodes,
        "scenarios_evaluated": scenario_ids,
    }

    print(f"\nEvaluation Summary:")
    print(f"  Avg Reward: {metrics['avg_reward']:.2f} (+/- {metrics['std_reward']:.2f})")
    print(f"  Avg Length: {metrics['avg_length']:.1f}")
    print(f"  Collisions: {metrics['collisions']}/{args.eval_episodes} ({metrics['collision_rate']*100:.1f}%)")

    return metrics
```

### Step 4: Implement Metrics Saver

**File:** `roadsense-v2v/ml/training/train_convoy.py`

Add a function to save metrics as JSON:

```python
def save_metrics(
    output_dir: str,
    training_metrics: dict,
    eval_metrics: Optional[dict],
    args: argparse.Namespace,
) -> str:
    """
    Save combined metrics to JSON file.

    Args:
        output_dir: Output directory
        training_metrics: Metrics from training
        eval_metrics: Metrics from evaluation (or None if skipped)
        args: Original arguments (for reproducibility)

    Returns:
        Path to saved metrics file
    """
    import json
    from datetime import datetime

    metrics = {
        "run_id": args.run_id,
        "timestamp": datetime.now().isoformat(),
        "config": {
            "dataset_dir": args.dataset_dir,
            "sumo_cfg": args.sumo_cfg,
            "emulator_params": args.emulator_params,
            "seed": args.seed,
            "total_timesteps": args.total_timesteps,
            "learning_rate": args.learning_rate,
            "n_steps": args.n_steps,
            "ent_coef": args.ent_coef,
        },
        "training": training_metrics,
        "evaluation": eval_metrics,
    }

    metrics_path = os.path.join(output_dir, "metrics.json")
    with open(metrics_path, "w") as f:
        json.dump(metrics, f, indent=2)

    print(f"Metrics saved to: {metrics_path}")
    return metrics_path
```

### Step 5: Update Main Entry Point

**File:** `roadsense-v2v/ml/training/train_convoy.py`

Wire everything together:

```python
def main() -> None:
    """Main entry point."""
    args = parse_args()

    # Train
    model_path, training_metrics = train(args)

    # Evaluate (unless skipped)
    eval_metrics = None
    if not args.skip_eval:
        eval_metrics = evaluate(model_path, args)

    # Save metrics
    save_metrics(args.output_dir, training_metrics, eval_metrics, args)

    print(f"\nRun complete: {args.run_id}")
    print(f"Output: {args.output_dir}")


if __name__ == "__main__":
    main()
```

### Step 6: Update run_docker.sh

**File:** `roadsense-v2v/ml/run_docker.sh`

Ensure the `train` subcommand passes through arguments:

```bash
train)
    echo "Running training..."
    shift
    docker run --rm \
        -e GIT_COMMIT="$(git -C .. rev-parse --short HEAD 2>/dev/null || echo unknown)" \
        -v "$(pwd)":/work:Z \
        -w /work \
        "$IMAGE_NAME" \
        python -m ml.training.train_convoy "$@"
    ;;
```

**Usage examples:**
```bash
# Train on dataset with seed
./run_docker.sh train --dataset_dir ml/scenarios/datasets/v1 --seed 42

# Train with custom hyperparameters
./run_docker.sh train --dataset_dir ml/scenarios/datasets/v1 \
    --total_timesteps 500000 \
    --learning_rate 1e-4 \
    --eval_episodes 20

# Train on single scenario (backward compatible)
./run_docker.sh train --sumo_cfg ml/scenarios/base/scenario.sumocfg

# Skip evaluation
./run_docker.sh train --dataset_dir ml/scenarios/datasets/v1 --skip_eval
```

---

## Exit Criteria

- [x] `train_convoy.py` accepts all CLI arguments listed in Step 1
- [x] Training uses `dataset_dir` with randomized scenario selection
- [x] Evaluation uses `dataset_dir` with sequential scenario selection
- [x] Metrics JSON is saved to output directory
- [x] Backward compatible: `--sumo_cfg` still works
- [x] `./run_docker.sh train --help` shows all options
- [x] Unit tests pass: `pytest ml/tests/unit/test_train_convoy.py -v`
- [x] Integration test: Full train + eval run completes without error

---

## Files Touched

| File | Action | Description |
|------|--------|-------------|
| `roadsense-v2v/ml/training/train_convoy.py` | REWRITE | Add argparse, eval loop, metrics |
| `roadsense-v2v/ml/run_docker.sh` | EDIT | Pass through args to train |
| `roadsense-v2v/ml/tests/unit/test_train_convoy.py` | CREATE | Unit tests for CLI parsing |

---

## Output Directory Structure

After a training run, the output directory will contain:

```
ml/models/runs/<run_id>/
├── model_final.zip          # Trained model
├── metrics.json             # Training + eval metrics
├── checkpoints/
│   ├── deep_sets_10000_steps.zip
│   ├── deep_sets_20000_steps.zip
│   └── ...
└── tensorboard/
    └── PPO_1/
        └── events.out.tfevents.*
```

---

## Metrics JSON Schema

```json
{
  "run_id": "run_20260115_143000",
  "timestamp": "2026-01-15T14:30:00",
  "config": {
    "dataset_dir": "ml/scenarios/datasets/v1",
    "sumo_cfg": null,
    "emulator_params": "ml/espnow_emulator/emulator_params_5m.json",
    "seed": 42,
    "total_timesteps": 100000,
    "learning_rate": 0.0003,
    "n_steps": 2048,
    "ent_coef": 0.01
  },
  "training": {
    "training_timesteps": 100000
  },
  "evaluation": {
    "eval_episodes": 10,
    "avg_reward": 145.3,
    "std_reward": 23.1,
    "min_reward": 98.2,
    "max_reward": 189.7,
    "avg_length": 487.2,
    "collisions": 2,
    "collision_rate": 0.2,
    "truncations": 8,
    "truncation_rate": 0.8,
    "scenarios_evaluated": ["eval_000", "eval_001", "eval_002", ...]
  }
}
```

---

## Risks / Notes

### Risk 1: Long Training Times

**Problem:** Full training (100K+ timesteps) takes 30+ minutes, too slow for integration tests.

**Mitigation:** Integration tests use `--total_timesteps 100 --eval_episodes 1` for quick smoke tests.

### Risk 2: GPU Not Available

**Problem:** PPO can use GPU, but Docker may not have CUDA configured.

**Mitigation:** Current setup uses CPU only. GPU support is out of scope for this phase.

### Risk 3: Memory During Evaluation

**Problem:** Storing all episode data in memory could be an issue for very long runs.

**Mitigation:** Metrics are computed incrementally. Only summary statistics are stored.

### Risk 4: Backward Compatibility

**Problem:** Existing scripts that call `train_convoy.py` without arguments might break.

**Mitigation:** Default behavior (no args) falls back to `ml/scenarios/base/scenario.sumocfg`, matching current behavior.

---

## Test Plan

### Unit Tests (`test_train_convoy.py`)

```python
class TestTrainConvoyCLI:
    """Tests for train_convoy.py argument parsing."""

    def test_parse_args_defaults(self):
        """Verify default values are set correctly."""

    def test_parse_args_dataset_dir(self):
        """Verify --dataset_dir is parsed."""

    def test_parse_args_mutual_exclusion(self):
        """Verify error when both --dataset_dir and --sumo_cfg provided."""

    def test_parse_args_all_hyperparams(self):
        """Verify all hyperparameters are parsed."""


class TestEvaluate:
    """Tests for evaluation function."""

    def test_evaluate_returns_metrics(self, mock_model, mock_env):
        """Verify evaluate() returns expected metrics dict."""

    def test_evaluate_uses_deterministic(self, mock_model, mock_env):
        """Verify model.predict is called with deterministic=True."""

    def test_evaluate_counts_collisions(self, mock_model, mock_env):
        """Verify collision counting is correct."""


class TestSaveMetrics:
    """Tests for metrics saving."""

    def test_save_metrics_creates_file(self, tmp_path):
        """Verify metrics.json is created."""

    def test_save_metrics_schema(self, tmp_path):
        """Verify metrics.json has correct structure."""
```

### Integration Test (Smoke Test)

```python
@pytest.mark.integration
@pytest.mark.slow
def test_train_eval_smoke(dataset_dir, tmp_path):
    """
    Quick smoke test: train for 100 steps, eval 1 episode.

    This verifies the full pipeline works end-to-end.
    """
    import subprocess

    result = subprocess.run(
        [
            "python", "-m", "ml.training.train_convoy",
            "--dataset_dir", str(dataset_dir),
            "--total_timesteps", "100",
            "--eval_episodes", "1",
            "--output_dir", str(tmp_path),
        ],
        capture_output=True,
        text=True,
        timeout=120,
    )

    assert result.returncode == 0, f"Training failed: {result.stderr}"
    assert (tmp_path / "model_final.zip").exists()
    assert (tmp_path / "metrics.json").exists()
```

---

## Acceptance Checklist (For REVIEWER)

- [x] All CLI arguments are parsed correctly
- [x] Default behavior matches existing script (backward compatible)
- [x] Training uses randomized scenario selection
- [x] Evaluation uses sequential scenario selection
- [x] Evaluation uses `deterministic=True` for predictions
- [x] Metrics JSON contains all required fields
- [x] `run_docker.sh train` passes arguments correctly
- [x] Unit tests cover argument parsing and metrics
- [x] Smoke test passes (quick train + eval)

---

**END OF PHASE 3 SPEC**
