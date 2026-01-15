# Phase 2 Specification: Deep Sets Policy Network

**Document:** `PHASE_2_SPEC.md`
**Status:** READY FOR IMPLEMENTATION
**Target Agent:** BUILDER
**Architecture Reference:** `00_ARCHITECTURE/DEEP_SETS_N_ELEMENT_ARCHITECTURE.md`
**Depends On:** Phase 1 (ConvoyEnv Dict Observation Space) - COMPLETE

---

## 1. Objective

Implement the **Deep Sets policy network** that processes the variable-n peer observations from Phase 1's Dict observation space. This network will be compatible with stable-baselines3 (SB3) for PPO training.

---

## 2. Background

### Why Deep Sets?

Standard MLPs cannot handle variable-length inputs. The Deep Sets architecture solves this by:
1. **Shared Encoder (φ)**: Encodes each peer independently with the same MLP weights
2. **Permutation-Invariant Pooling**: Aggregates peer embeddings via max-pooling
3. **Fixed Output**: Produces a fixed-size latent vector regardless of peer count

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│  Input: Dict observation from ConvoyEnv                         │
│    - ego: (4,) normalized ego features                          │
│    - peers: (8, 6) peer features (padded)                       │
│    - peer_mask: (8,) validity mask                              │
├─────────────────────────────────────────────────────────────────┤
│  Stage 1: Per-Peer Encoding                                     │
│    peers[i] → φ(peers[i]) → h_i ∈ ℝ³²                          │
│    (shared MLP applied to each of 8 peer slots)                 │
├─────────────────────────────────────────────────────────────────┤
│  Stage 2: Masked Max Pooling                                    │
│    z = max_pool({h_i : peer_mask[i] == 1})                      │
│    If no valid peers: z = zeros(32)                             │
├─────────────────────────────────────────────────────────────────┤
│  Stage 3: Concatenation                                         │
│    combined = concat(z, ego) → ℝ³⁶                             │
├─────────────────────────────────────────────────────────────────┤
│  Output: combined (36,) → SB3 policy/value heads                │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. Technical Specification

### 3.1 File Structure

Create the following new file:
```
ml/
├── models/
│   ├── __init__.py
│   └── deep_set_extractor.py   ← NEW
```

### 3.2 DeepSetExtractor Class

**File:** `ml/models/deep_set_extractor.py`

```python
class DeepSetExtractor(BaseFeaturesExtractor):
    """
    Custom feature extractor for Dict observation spaces with variable peers.

    Implements Deep Sets architecture:
    1. Shared MLP encodes each peer independently
    2. Max pooling aggregates peer embeddings (masked)
    3. Concatenates pooled features with ego state

    Compatible with stable-baselines3 PPO/A2C/SAC.
    """
```

### 3.3 Network Dimensions

| Component | Input | Output | Notes |
|-----------|-------|--------|-------|
| Peer Encoder (φ) | 6 | 32 | Two-layer MLP: 6→32→32 |
| Max Pool | (8, 32) | 32 | Over valid peers only |
| Combined | 32 + 4 | 36 | Peer latent + ego state |
| **features_dim** | - | **36** | Output to SB3 policy head |

### 3.4 Implementation Requirements

#### A. Constructor

```python
def __init__(
    self,
    observation_space: gym.spaces.Dict,
    peer_feature_dim: int = 6,
    ego_feature_dim: int = 4,
    embed_dim: int = 32,
):
```

Must:
- Validate observation_space has keys: `ego`, `peers`, `peer_mask`
- Initialize shared peer encoder MLP
- Set `self._features_dim = embed_dim + ego_feature_dim` (36)

#### B. Peer Encoder Architecture

```python
self.peer_encoder = nn.Sequential(
    nn.Linear(peer_feature_dim, embed_dim),
    nn.ReLU(),
    nn.Linear(embed_dim, embed_dim),
    nn.ReLU(),
)
```

#### C. Forward Method

```python
def forward(self, observations: Dict[str, torch.Tensor]) -> torch.Tensor:
    """
    Args:
        observations: Dict with keys:
            - 'ego': (batch, 4) ego features
            - 'peers': (batch, 8, 6) peer features
            - 'peer_mask': (batch, 8) validity mask (1=valid, 0=padded)

    Returns:
        features: (batch, 36) combined features for policy head
    """
```

Must:
1. Encode all peer slots: `h = peer_encoder(peers)` → (batch, 8, 32)
2. Apply mask: Set invalid peer embeddings to -inf for max pool
3. Max pool: `z = h.max(dim=1)` → (batch, 32)
4. Handle n=0 case: If all peers masked, z = zeros
5. Concatenate: `[z, ego]` → (batch, 36)

#### D. Masked Max Pooling (Critical)

```python
# Expand mask: (batch, 8) → (batch, 8, 32)
mask_expanded = peer_mask.unsqueeze(-1).expand_as(peer_embeddings)

# Set padded peers to -inf (excluded from max)
peer_embeddings = peer_embeddings.masked_fill(~mask_expanded.bool(), float('-inf'))

# Max pool over peers dimension
z, _ = peer_embeddings.max(dim=1)  # (batch, 32)

# Handle n=0: If all -inf, replace with zeros
all_masked = ~peer_mask.any(dim=1, keepdim=True)  # (batch, 1)
z = torch.where(all_masked, torch.zeros_like(z), z)
```

---

## 4. Integration with SB3

### 4.1 Policy Kwargs

When creating the PPO model:

```python
from stable_baselines3 import PPO
from models.deep_set_extractor import DeepSetExtractor

policy_kwargs = dict(
    features_extractor_class=DeepSetExtractor,
    features_extractor_kwargs=dict(
        embed_dim=32,
    ),
)

model = PPO(
    "MultiInputPolicy",
    env,
    policy_kwargs=policy_kwargs,
    verbose=1,
)
```

### 4.2 MultiInputPolicy Requirement

SB3's `MultiInputPolicy` is required for Dict observation spaces. The custom `DeepSetExtractor` replaces the default feature extractor.

---

## 5. TDD Requirements

### Step 1: Create `ml/tests/unit/test_deep_set_extractor.py`

Write tests that:

1. **test_extractor_output_shape**:
   - Input: batch of observations with varying valid peer counts
   - Assert output shape is `(batch_size, 36)`

2. **test_extractor_handles_zero_peers**:
   - Input: observation with `peer_mask` all zeros
   - Assert output is valid (no NaN/Inf)
   - Assert peer latent portion is zeros

3. **test_extractor_handles_max_peers**:
   - Input: observation with all 8 peers valid
   - Assert output shape is correct

4. **test_extractor_permutation_invariant**:
   - Input: Two observations with same peers in different order
   - Assert outputs are identical (within tolerance)

5. **test_extractor_features_dim_is_36**:
   - Assert `extractor.features_dim == 36`

6. **test_extractor_works_with_sb3_ppo**:
   - Create PPO with DeepSetExtractor
   - Call `model.predict()` with sample observation
   - Assert no errors

### Step 2: Implementation

Implement `DeepSetExtractor` until all tests pass.

---

## 6. Files Touched

| File | Action | Notes |
|------|--------|-------|
| `ml/models/__init__.py` | Create | Export DeepSetExtractor |
| `ml/models/deep_set_extractor.py` | Create | Main implementation |
| `ml/tests/unit/test_deep_set_extractor.py` | Create | TDD tests |

---

## 7. Acceptance Criteria

1. All TDD tests pass
2. `DeepSetExtractor.features_dim` returns 36
3. Forward pass handles batch sizes 1 to 256
4. Forward pass handles 0 to 8 valid peers
5. Output is permutation-invariant to peer ordering
6. PPO model can be instantiated with the extractor
7. `model.predict()` returns valid actions

---

## 8. Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| -inf propagation in max pool | Explicit n=0 handling with zeros |
| Device mismatch (CPU/GPU) | Use `torch.zeros_like()` for device-safe tensors |
| SB3 version incompatibility | Test with SB3 2.0+ (Dict observation support) |
| Numerical instability | Use ReLU (no vanishing gradients) |

---

## 9. Example Usage

```python
import gymnasium as gym
from stable_baselines3 import PPO
import ml.envs
from models.deep_set_extractor import DeepSetExtractor

# Create environment
env = gym.make("RoadSense-Convoy-v0", sumo_cfg="scenarios/base/scenario.sumocfg")

# Create model with Deep Sets extractor
model = PPO(
    "MultiInputPolicy",
    env,
    policy_kwargs=dict(
        features_extractor_class=DeepSetExtractor,
        features_extractor_kwargs=dict(embed_dim=32),
    ),
    verbose=1,
)

# Train
model.learn(total_timesteps=100_000)

# Save
model.save("deep_sets_convoy_model")
```

---

## 10. References

- Architecture: `docs/00_ARCHITECTURE/DEEP_SETS_N_ELEMENT_ARCHITECTURE.md`
- SB3 Custom Features: https://stable-baselines3.readthedocs.io/en/master/guide/custom_policy.html
- Deep Sets Paper: Zaheer et al., 2017

---

**END OF SPECIFICATION**
