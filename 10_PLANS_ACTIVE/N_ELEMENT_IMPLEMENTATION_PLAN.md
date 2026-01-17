# n-Element Deep Sets Implementation Plan

**Document Type:** Implementation Plan & Migration Guide
**Created:** January 13, 2026
**Author:** Amir Khalifa
**Status:** ACTIVE - Ready for Implementation
**Priority:** CRITICAL PATH - Blocks RL training
**Reference:** `00_ARCHITECTURE/DEEP_SETS_N_ELEMENT_ARCHITECTURE.md`

---

## Table of Contents

1. [Overview](#1-overview)
2. [Phase 1: ConvoyEnv Modifications](#2-phase-1-convoyenv-modifications)
3. [Phase 2: ESP-NOW Emulator Updates](#3-phase-2-esp-now-emulator-updates)
4. [Phase 3: Custom Policy Network](#4-phase-3-custom-policy-network)
5. [Phase 4: Training Pipeline](#5-phase-4-training-pipeline)
6. [Phase 5: Firmware Changes](#6-phase-5-firmware-changes)
7. [Phase 6: Production Training Run](#7-phase-6-production-training-run-next) ► **NEXT**
8. [Testing Checklist](#8-testing-checklist)

---

## 1. Overview

### What Changes

| Component | Before | After |
|-----------|--------|-------|
| Observation | `Box(11,)` fixed | `Dict` with variable peers |
| Peers handled | Hardcoded V002, V003 | Dynamic list, 0 to MAX_PEERS |
| Network | MlpPolicy (SB3 default) | Custom DeepSetPolicy |
| Emulator | 2 peer tracking | n peer tracking |
| Firmware | Fixed feature extraction | Loop-based encoding |

### Files to Create/Modify

```
roadsense-v2v/ml/
├── envs/
│   ├── convoy_env.py           # MODIFY: Dict obs, variable peers
│   └── observation_builder.py  # MODIFY: Handle n peers
├── espnow_emulator/
│   └── espnow_emulator.py      # MODIFY: n peer support
├── policies/                   # NEW DIRECTORY
│   ├── __init__.py             # NEW
│   ├── deep_set_policy.py      # NEW: Custom policy network
│   └── feature_extractor.py    # NEW: SB3 integration
├── training/
│   └── train_convoy.py         # MODIFY: Use custom policy
└── tests/
    ├── test_deep_set_policy.py # NEW
    └── test_variable_peers.py  # NEW
```

---

## 2. Phase 1: ConvoyEnv Modifications

### 2.1 Observation Space Change

**File:** `roadsense-v2v/ml/envs/convoy_env.py`

**Before:**
```python
self.observation_space = gym.spaces.Box(
    low=-np.inf, high=np.inf,
    shape=(11,),
    dtype=np.float32
)
```

**After:**
```python
MAX_PEERS = 8  # Maximum peers to handle

self.observation_space = gym.spaces.Dict({
    "ego": gym.spaces.Box(
        low=np.array([0.0, -1.0, -1.0, 0.0], dtype=np.float32),
        high=np.array([1.0, 1.0, 1.0, 1.0], dtype=np.float32),
        dtype=np.float32
    ),
    "peers": gym.spaces.Box(
        low=-np.inf,
        high=np.inf,
        shape=(MAX_PEERS, 6),
        dtype=np.float32
    ),
    "peer_mask": gym.spaces.Box(
        low=0.0,
        high=1.0,
        shape=(MAX_PEERS,),
        dtype=np.float32
    )
})
```

### 2.2 Observation Builder Update

**File:** `roadsense-v2v/ml/envs/observation_builder.py`

**New Implementation:**
```python
class ObservationBuilder:
    """
    Builds observations for variable-n peer environments.
    """

    MAX_PEERS = 8

    # Normalization constants
    MAX_SPEED = 30.0
    MAX_DISTANCE = 100.0
    MAX_ACCEL = 10.0
    STALENESS_THRESHOLD = 500.0

    def build(self, ego_state: VehicleState,
              peer_observations: List[Dict],
              ego_pos: Tuple[float, float]) -> Dict[str, np.ndarray]:
        """
        Build observation dict from ego state and variable peer list.

        Args:
            ego_state: Ego vehicle state
            peer_observations: List of peer observation dicts from emulator
            ego_pos: Ego position for relative calculations

        Returns:
            Dict with 'ego', 'peers', 'peer_mask' arrays
        """
        # Ego features (4 dims)
        ego = np.array([
            ego_state.speed / self.MAX_SPEED,
            ego_state.acceleration / self.MAX_ACCEL,
            ego_state.heading / np.pi,
            len(peer_observations) / self.MAX_PEERS
        ], dtype=np.float32)

        # Peer features (MAX_PEERS x 6)
        peers = np.zeros((self.MAX_PEERS, 6), dtype=np.float32)
        peer_mask = np.zeros(self.MAX_PEERS, dtype=np.float32)

        for i, peer in enumerate(peer_observations[:self.MAX_PEERS]):
            if peer.get('valid', False):
                peers[i] = self._extract_peer_features(peer, ego_state, ego_pos)
                peer_mask[i] = 1.0

        return {
            "ego": ego,
            "peers": peers,
            "peer_mask": peer_mask
        }

    def _extract_peer_features(self, peer: Dict,
                                ego_state: VehicleState,
                                ego_pos: Tuple[float, float]) -> np.ndarray:
        """Extract and normalize 6 features from a peer observation."""
        # Calculate relative position in ego-centric frame
        dx = peer['x'] - ego_pos[0]
        dy = peer['y'] - ego_pos[1]

        # Rotate to ego heading frame
        cos_h = np.cos(-ego_state.heading)
        sin_h = np.sin(-ego_state.heading)
        rel_x = dx * cos_h - dy * sin_h
        rel_y = dx * sin_h + dy * cos_h

        return np.array([
            rel_x / self.MAX_DISTANCE,
            rel_y / self.MAX_DISTANCE,
            (peer['speed'] - ego_state.speed) / self.MAX_SPEED,
            (peer['heading'] - ego_state.heading) / np.pi,
            peer['accel'] / self.MAX_ACCEL,
            peer['age_ms'] / self.STALENESS_THRESHOLD
        ], dtype=np.float32)
```

### 2.3 Step Function Update

**In `convoy_env.py`:**

```python
def _step_espnow(self) -> Dict[str, np.ndarray]:
    """Process ESP-NOW communication and build observation."""
    current_time_ms = int(traci.simulation.getTime() * 1000)
    ego_pos = traci.vehicle.getPosition(self.ego_id)
    ego_state = self._get_vehicle_state(self.ego_id)

    # Get all peer vehicles (not just V002, V003)
    peer_observations = []
    for vehicle_id in traci.vehicle.getIDList():
        if vehicle_id == self.ego_id:
            continue

        peer_msg = self._get_vehicle_state(vehicle_id)
        if peer_msg:
            peer_pos = traci.vehicle.getPosition(vehicle_id)

            # Transmit through emulator
            received = self.espnow.transmit(
                sender_msg=peer_msg,
                sender_pos=peer_pos,
                receiver_pos=ego_pos,
                current_time_ms=current_time_ms
            )

            if received:
                peer_observations.append({
                    'vehicle_id': vehicle_id,
                    'x': received.message.lon,
                    'y': received.message.lat,
                    'speed': received.message.speed,
                    'heading': received.message.heading,
                    'accel': received.message.accel_x,
                    'age_ms': received.age_ms,
                    'valid': True
                })

    # Build observation using ObservationBuilder
    return self.obs_builder.build(ego_state, peer_observations, ego_pos)
```

---

## 3. Phase 2: ESP-NOW Emulator Updates

### 3.1 Remove Hardcoded Vehicle IDs

**File:** `roadsense-v2v/ml/espnow_emulator/espnow_emulator.py`

**Before:**
```python
def get_observation(self, ego_speed: float, current_time_ms: int) -> Dict:
    obs = {'ego_speed': ego_speed}

    for vehicle_id in ['V002', 'V003']:  # HARDCODED
        if vehicle_id in self.last_received:
            # ...
```

**After:**
```python
def get_all_peer_observations(self, current_time_ms: int) -> List[Dict]:
    """
    Get observations for ALL peers with active messages.

    Returns:
        List of peer observation dicts (variable length)
    """
    observations = []

    for vehicle_id, recv_msg in self.last_received.items():
        age_ms = current_time_ms - recv_msg.received_at_ms + recv_msg.age_ms

        if age_ms < self.STALENESS_THRESHOLD:
            observations.append({
                'vehicle_id': vehicle_id,
                'x': recv_msg.message.lon,
                'y': recv_msg.message.lat,
                'speed': recv_msg.message.speed,
                'heading': recv_msg.message.heading,
                'accel': recv_msg.message.accel_x,
                'age_ms': age_ms,
                'valid': True
            })

    return observations
```

### 3.2 Update Internal Tracking

```python
def __init__(self, params_file: str = None, domain_randomization: bool = True):
    # ...

    # Change from Dict[str, ReceivedMessage] to support any vehicle ID
    self.last_received: Dict[str, ReceivedMessage] = {}
    self.STALENESS_THRESHOLD = 500  # ms - messages older than this are invalid
```

---

## 4. Phase 3: Custom Policy Network

### 4.1 DeepSetPolicy Implementation

**File:** `roadsense-v2v/ml/policies/deep_set_policy.py`

```python
import torch
import torch.nn as nn
from stable_baselines3.common.torch_layers import BaseFeaturesExtractor
from gymnasium import spaces
import numpy as np


class DeepSetFeatureExtractor(BaseFeaturesExtractor):
    """
    Custom feature extractor for Dict observation with variable peers.

    Implements Deep Sets: φ(peers) -> max pool -> concat with ego -> z
    """

    def __init__(self, observation_space: spaces.Dict,
                 peer_embed_dim: int = 32,
                 features_dim: int = 36):
        super().__init__(observation_space, features_dim)

        # Dimensions from observation space
        self.ego_dim = observation_space["ego"].shape[0]  # 4
        self.peer_feature_dim = observation_space["peers"].shape[1]  # 6
        self.max_peers = observation_space["peers"].shape[0]  # 8

        # Shared peer encoder (φ)
        self.peer_encoder = nn.Sequential(
            nn.Linear(self.peer_feature_dim, peer_embed_dim),
            nn.ReLU(),
            nn.Linear(peer_embed_dim, peer_embed_dim),
            nn.ReLU()
        )

        self._features_dim = peer_embed_dim + self.ego_dim  # 32 + 4 = 36

    def forward(self, observations: Dict[str, torch.Tensor]) -> torch.Tensor:
        """
        Args:
            observations: Dict with:
                - "ego": (batch, 4)
                - "peers": (batch, max_peers, 6)
                - "peer_mask": (batch, max_peers)

        Returns:
            features: (batch, 36)
        """
        ego = observations["ego"]
        peers = observations["peers"]
        mask = observations["peer_mask"]

        batch_size = ego.shape[0]

        # Encode all peers: (batch, max_peers, 6) -> (batch, max_peers, 32)
        peer_embeddings = self.peer_encoder(peers)

        # Mask invalid peers with -inf for max pooling
        mask_expanded = mask.unsqueeze(-1).expand_as(peer_embeddings)
        peer_embeddings = peer_embeddings.masked_fill(mask_expanded == 0, float('-inf'))

        # Max pooling: (batch, max_peers, 32) -> (batch, 32)
        z, _ = peer_embeddings.max(dim=1)

        # Handle all-masked case (n=0): replace -inf with zeros
        all_masked = (mask.sum(dim=1) == 0).unsqueeze(-1)
        z = torch.where(all_masked, torch.zeros_like(z), z)

        # Concatenate with ego: (batch, 32) + (batch, 4) -> (batch, 36)
        return torch.cat([z, ego], dim=1)


def create_deep_set_policy_kwargs(peer_embed_dim: int = 32) -> dict:
    """
    Create policy_kwargs for stable-baselines3 PPO with DeepSetFeatureExtractor.

    Usage:
        model = PPO("MultiInputPolicy", env,
                    policy_kwargs=create_deep_set_policy_kwargs())
    """
    return {
        "features_extractor_class": DeepSetFeatureExtractor,
        "features_extractor_kwargs": {
            "peer_embed_dim": peer_embed_dim,
        },
        "net_arch": [64, 64],  # Policy head architecture
    }
```

### 4.2 Unit Tests

**File:** `roadsense-v2v/ml/tests/test_deep_set_policy.py`

```python
import pytest
import torch
import numpy as np
from gymnasium import spaces
from policies.deep_set_policy import DeepSetFeatureExtractor


@pytest.fixture
def observation_space():
    return spaces.Dict({
        "ego": spaces.Box(low=-1, high=1, shape=(4,), dtype=np.float32),
        "peers": spaces.Box(low=-np.inf, high=np.inf, shape=(8, 6), dtype=np.float32),
        "peer_mask": spaces.Box(low=0, high=1, shape=(8,), dtype=np.float32)
    })


def test_feature_extractor_shape(observation_space):
    """Test output shape is correct."""
    extractor = DeepSetFeatureExtractor(observation_space)

    batch_size = 4
    obs = {
        "ego": torch.randn(batch_size, 4),
        "peers": torch.randn(batch_size, 8, 6),
        "peer_mask": torch.ones(batch_size, 8)
    }

    features = extractor(obs)
    assert features.shape == (batch_size, 36)


def test_permutation_invariance(observation_space):
    """Test that shuffling peers doesn't change output."""
    extractor = DeepSetFeatureExtractor(observation_space)

    ego = torch.randn(1, 4)
    peers = torch.randn(1, 8, 6)
    mask = torch.tensor([[1, 1, 1, 0, 0, 0, 0, 0]], dtype=torch.float32)

    # Original order
    obs1 = {"ego": ego, "peers": peers, "peer_mask": mask}
    z1 = extractor(obs1)

    # Shuffle first 3 peers (the valid ones)
    perm = torch.tensor([2, 0, 1, 3, 4, 5, 6, 7])
    peers_shuffled = peers[:, perm, :]
    mask_shuffled = mask[:, perm]

    obs2 = {"ego": ego, "peers": peers_shuffled, "peer_mask": mask_shuffled}
    z2 = extractor(obs2)

    # Should be identical
    assert torch.allclose(z1, z2, atol=1e-5)


def test_zero_peers(observation_space):
    """Test handling of n=0 case."""
    extractor = DeepSetFeatureExtractor(observation_space)

    obs = {
        "ego": torch.randn(1, 4),
        "peers": torch.randn(1, 8, 6),
        "peer_mask": torch.zeros(1, 8)  # All peers masked
    }

    features = extractor(obs)

    # Should not have NaN or inf
    assert not torch.isnan(features).any()
    assert not torch.isinf(features).any()

    # Peer part should be zeros
    assert torch.allclose(features[:, :32], torch.zeros(1, 32))


def test_variable_peer_count(observation_space):
    """Test with different peer counts in same batch."""
    extractor = DeepSetFeatureExtractor(observation_space)

    obs = {
        "ego": torch.randn(3, 4),
        "peers": torch.randn(3, 8, 6),
        "peer_mask": torch.tensor([
            [1, 0, 0, 0, 0, 0, 0, 0],  # 1 peer
            [1, 1, 1, 1, 1, 0, 0, 0],  # 5 peers
            [1, 1, 1, 1, 1, 1, 1, 1],  # 8 peers
        ], dtype=torch.float32)
    }

    features = extractor(obs)
    assert features.shape == (3, 36)
```

---

## 5. Phase 4: Training Pipeline

### 5.1 Updated Training Script

**File:** `roadsense-v2v/ml/training/train_convoy.py`

```python
import gymnasium as gym
from stable_baselines3 import PPO
from stable_baselines3.common.env_checker import check_env
from stable_baselines3.common.callbacks import EvalCallback

from policies.deep_set_policy import create_deep_set_policy_kwargs
import ml.envs  # Register environments


def main():
    # Create environment
    env = gym.make('RoadSense-Convoy-v0',
                   domain_randomization=True,
                   hazard_injection=True)

    # Verify environment
    check_env(env, warn=True)
    print(f"Observation space: {env.observation_space}")
    print(f"Action space: {env.action_space}")

    # Create model with custom policy
    model = PPO(
        "MultiInputPolicy",  # Required for Dict observation space
        env,
        policy_kwargs=create_deep_set_policy_kwargs(peer_embed_dim=32),
        learning_rate=3e-4,
        n_steps=2048,
        batch_size=64,
        n_epochs=10,
        gamma=0.99,
        verbose=1,
        tensorboard_log="./tensorboard/"
    )

    # Train
    model.learn(
        total_timesteps=100_000,
        callback=EvalCallback(
            env,
            eval_freq=10000,
            n_eval_episodes=10,
            best_model_save_path="./best_model/"
        )
    )

    model.save("convoy_deep_set_100k")


if __name__ == "__main__":
    main()
```

---

## 6. Phase 5: Firmware Changes

### 6.1 Peer Data Structure

**File:** `roadsense-v2v/hardware/include/ml_types.h`

```cpp
#ifndef ML_TYPES_H
#define ML_TYPES_H

#define MAX_PEERS 8
#define PEER_FEATURE_DIM 6
#define EGO_FEATURE_DIM 4
#define PEER_EMBED_DIM 32

struct PeerFeatures {
    float rel_x;
    float rel_y;
    float rel_speed;
    float rel_heading;
    float accel;
    float age_ms;
};

struct EgoFeatures {
    float speed;
    float accel;
    float heading;
    float peer_count;
};

struct MLObservation {
    EgoFeatures ego;
    PeerFeatures peers[MAX_PEERS];
    int peer_count;
};

#endif // ML_TYPES_H
```

### 6.2 Inference Loop

**File:** `roadsense-v2v/hardware/src/ml_inference.cpp`

```cpp
#include "ml_inference.h"
#include "ml_types.h"
#include <tensorflow/lite/micro/micro_interpreter.h>
#include <algorithm>
#include <cmath>

// TFLite model buffers (linked from model_data.h)
extern const unsigned char peer_encoder_model[];
extern const unsigned char policy_head_model[];

// Tensor arena
constexpr int kTensorArenaSize = 8 * 1024;
alignas(16) uint8_t tensor_arena[kTensorArenaSize];

int runInference(const MLObservation& obs) {
    // Initialize interpreters (in real code, do this once in setup)
    // ... TFLite setup code ...

    // === Stage 1: Encode each peer and max-pool ===
    float h_max[PEER_EMBED_DIM];
    std::fill(h_max, h_max + PEER_EMBED_DIM, -INFINITY);

    for (int i = 0; i < obs.peer_count; i++) {
        // Prepare peer input
        float peer_input[PEER_FEATURE_DIM] = {
            obs.peers[i].rel_x,
            obs.peers[i].rel_y,
            obs.peers[i].rel_speed,
            obs.peers[i].rel_heading,
            obs.peers[i].accel,
            obs.peers[i].age_ms / 500.0f  // Normalize
        };

        // Run peer encoder
        float h[PEER_EMBED_DIM];
        runPeerEncoder(peer_input, h);

        // Max pooling
        for (int j = 0; j < PEER_EMBED_DIM; j++) {
            h_max[j] = std::max(h_max[j], h[j]);
        }
    }

    // === Stage 2: Handle n=0 case ===
    if (obs.peer_count == 0) {
        std::fill(h_max, h_max + PEER_EMBED_DIM, 0.0f);
    }

    // === Stage 3: Concatenate and run policy head ===
    float policy_input[PEER_EMBED_DIM + EGO_FEATURE_DIM];

    // Copy peer latent
    std::copy(h_max, h_max + PEER_EMBED_DIM, policy_input);

    // Copy ego features
    policy_input[PEER_EMBED_DIM + 0] = obs.ego.speed / 30.0f;
    policy_input[PEER_EMBED_DIM + 1] = obs.ego.accel / 10.0f;
    policy_input[PEER_EMBED_DIM + 2] = obs.ego.heading / M_PI;
    policy_input[PEER_EMBED_DIM + 3] = obs.ego.peer_count / MAX_PEERS;

    // Run policy head
    float action_logits[4];
    runPolicyHead(policy_input, action_logits);

    // Return argmax
    return std::distance(action_logits,
                         std::max_element(action_logits, action_logits + 4));
}
```

### 6.3 V2V Message Processing

**File:** `roadsense-v2v/hardware/src/v2v_receiver.cpp`

```cpp
void processV2VMessages(MLObservation& obs) {
    obs.peer_count = 0;

    // Get ego state
    obs.ego.speed = gps.getSpeed();
    obs.ego.accel = imu.getAccelX();
    obs.ego.heading = mag.getHeading();

    // Process all received V2V messages
    uint32_t now = millis();

    for (auto& [vehicle_id, msg] : receivedMessages) {
        uint32_t age_ms = now - msg.receive_time;

        // Skip stale messages
        if (age_ms > 500) continue;

        // Skip if already at max peers
        if (obs.peer_count >= MAX_PEERS) break;

        // Calculate relative position
        float ego_x = gps.getX();
        float ego_y = gps.getY();
        float dx = msg.x - ego_x;
        float dy = msg.y - ego_y;

        // Rotate to ego-centric frame
        float cos_h = cos(-obs.ego.heading);
        float sin_h = sin(-obs.ego.heading);

        PeerFeatures& peer = obs.peers[obs.peer_count];
        peer.rel_x = dx * cos_h - dy * sin_h;
        peer.rel_y = dx * sin_h + dy * cos_h;
        peer.rel_speed = msg.speed - obs.ego.speed;
        peer.rel_heading = msg.heading - obs.ego.heading;
        peer.accel = msg.accel;
        peer.age_ms = age_ms;

        obs.peer_count++;
    }

    obs.ego.peer_count = obs.peer_count;
}
```

---

## 7. Phase 6: Production Training Run (NEXT)

**Status:** PLANNED - After Firmware Migration
**Discovered:** January 17, 2026 - Cloud Run 001 post-mortem

### 7.1 Critical Issue: Run 001 Used Fixed n=2

**Problem Identified:**
- Cloud Training Run 001 (5M timesteps) achieved 80% success
- **BUT** all 25 scenarios in `dataset_v1` had exactly 3 vehicles (V001, V002, V003)
- The model was trained with **constant n=2 peers** for the entire run
- The Deep Sets architecture can handle variable n, but never saw n≠2 during training

**Impact:**
- Model behavior for n=0 (isolated), n=1, n=3+ is **untested and unpredictable**
- The 80% success rate is valid but only for n=2 scenarios
- Not a catastrophic failure (architecture is correct), but training data was homogeneous

### 7.2 Dataset V2 Requirements

**Scenario Generation Workflow:**
1. **User manually:** Crop real-world map from OpenStreetMap
2. **User manually:** Import to SUMO NetConvert, create base scenario
3. **Script:** Augment base scenario with `generate_dataset.py`
4. **Script MUST vary:**
   - Number of vehicles per scenario: n ∈ {1, 2, 3, 4, 5} peers (plus ego)
   - Vehicle parameters (decel, tau, sigma, speedFactor) - already done
   - Spawn timing jitter - already done

**Generator Changes Required:**
```python
# In scenario generator, add vehicle count variation:
peer_counts = [1, 2, 3, 4, 5]  # Variable number of peers
for scenario_id, peer_count in enumerate(scenarios):
    # Generate V001 (ego) + V002..V00{peer_count+1} peers
    generate_scenario_with_n_peers(peer_count)
```

### 7.3 Training Run 002 Requirements

| Parameter | Value | Notes |
|-----------|-------|-------|
| Timesteps | **10,000,000** (10M) | 2x previous run |
| Evaluation | **Required** | Run eval episodes after training |
| Dataset | `dataset_v2` | Variable peer counts |
| Peer counts | n ∈ {1,2,3,4,5} | Diverse training |
| Checkpoints | Every 500K steps | For recovery |
| Instance | c6i.xlarge or larger | Same as Run 001 |

**Eval Protocol:**
- Run 20+ episodes on eval set
- Report success rate per peer count (n=1, n=2, etc.)
- Identify any n-specific failure modes

### 7.4 Acceptance Criteria

- [ ] Dataset v2 generated with variable peer counts
- [ ] At least 4 scenarios per peer count (n=1,2,3,4,5)
- [ ] Training completes 10M timesteps
- [ ] Evaluation runs after training
- [ ] Success rate >70% across ALL peer counts
- [ ] No catastrophic failure for any specific n

---

## 8. Testing Checklist

### Phase 1: ConvoyEnv
- [ ] `env.observation_space` is `Dict` type
- [ ] `env.reset()` returns valid Dict observation
- [ ] `env.step()` works with 0 peers
- [ ] `env.step()` works with 8 peers
- [ ] Peer count varies across episode
- [ ] `check_env(env)` passes

### Phase 2: ESP-NOW Emulator
- [ ] `transmit()` works with any vehicle_id
- [ ] `get_all_peer_observations()` returns variable-length list
- [ ] Stale messages are excluded
- [ ] Domain randomization affects all peers

### Phase 3: Custom Policy
- [ ] `DeepSetFeatureExtractor` outputs (batch, 36)
- [ ] Permutation invariance test passes
- [ ] n=0 case handled correctly
- [ ] Variable peer count in batch works

### Phase 4: Training
- [ ] PPO trains without errors
- [ ] Learning curve shows improvement
- [ ] Model checkpoint saves correctly
- [ ] Eval callback works

### Phase 5: Firmware
- [ ] `MLObservation` struct compiles
- [ ] Peer encoder TFLite runs
- [ ] Max pooling produces correct output
- [ ] Policy head TFLite runs
- [ ] End-to-end inference < 50ms

---

## Document Changelog

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-01-13 | Amir Khalifa | Initial implementation plan |
| 1.1 | 2026-01-17 | Claude | Added Phase 6: Production Training Run requirements (variable n) |

---

**END OF DOCUMENT**
