# Deep Sets Architecture: Solving the n-Element Problem

**Document Type:** Architecture Decision Record (ADR)
**Created:** January 13, 2026
**Author:** Amir Khalifa
**Status:** APPROVED - Supersedes ARCHITECTURE_V2_MINIMAL_RL.md observation space design
**Source:** Professor's guidance document (professors_research_for_RL.pdf)

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [The n-Element Problem](#2-the-n-element-problem)
3. [Deep Sets Architecture](#3-deep-sets-architecture)
4. [Observation Space Design](#4-observation-space-design)
5. [Network Architecture](#5-network-architecture)
6. [ESP32 Deployment Constraints](#6-esp32-deployment-constraints)
7. [Affected Components](#7-affected-components)
8. [Migration Path](#8-migration-path)

---

## 1. Executive Summary

### The Problem

The ego vehicle receives V2V broadcasts from **n neighboring vehicles**, where **n is unknown and changes every timestep**. A fixed-size observation space (e.g., "11 features for 2 peers") fundamentally cannot handle this.

### The Solution

Use a **permutation-invariant set encoder** (Deep Sets architecture) that:
1. Encodes each peer independently with a shared MLP
2. Aggregates all peer embeddings via max-pooling
3. Produces a **fixed-size latent** regardless of n

### Key Insight (From Professor)

> "What you are describing is not a 'variable-size MLP' problem. It is a **set-to-policy** problem, and it has a well-established solution class: **permutation-invariant set encoders with pooling or attention**."

---

## 2. The n-Element Problem

### Why Fixed Observation Fails

| Approach | Problem |
|----------|---------|
| Pad to max-n (e.g., 8 slots) | Model learns "slot positions" - catastrophic overfitting |
| Sort by distance | Destroys permutation invariance |
| Flatten into vector | Observation size changes; breaks Gymnasium API |
| Hardcode 2 peers | Cannot scale to 8+ vehicle scenarios |

### Real-World Scenarios

| Scenario | n (peers) | Fixed-11 Handling | Deep Sets Handling |
|----------|-----------|-------------------|-------------------|
| Empty road | 0 | Half features wasted | z = zero vector |
| Convoy (3 vehicles) | 2 | Works (designed for this) | Works |
| Dense intersection | 6 | Ignores 4 vehicles | All 6 processed |
| Highway merge | 8+ | Fails completely | Scales naturally |

### Formalization

At each timestep, the agent observes:
- **Ego state**: `s = [speed, accel, heading, peer_count]` (fixed 4 dims)
- **Peer set**: `E = {e_1, e_2, ..., e_n}` where each `e_i = [rel_x, rel_y, speed, accel, heading, age_ms]` (6 dims per peer)

Key properties:
- Elements in E are **unordered** (no natural "peer 1" vs "peer 2")
- n **changes every timestep** (vehicles enter/leave range)
- Policy must be **invariant to permutation** of E

---

## 3. Deep Sets Architecture

### Canonical Form

```
z = POOL(φ(e_1), φ(e_2), ..., φ(e_n))
a = π([z, s])
```

Where:
- `φ` is a shared MLP applied to each peer independently
- `POOL` is permutation-invariant (mean, max, or attention)
- `z` is a **fixed-size vector** regardless of n
- `π` is the policy head

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     SET-TO-POLICY NETWORK                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  STAGE 1: Per-Peer Encoding (Shared φ)                                     │
│  ─────────────────────────────────────                                     │
│                                                                             │
│    Peer 1: [rel_x, rel_y, speed, heading, accel, age_ms]  ─┐               │
│    Peer 2: [rel_x, rel_y, speed, heading, accel, age_ms]   │               │
│    Peer 3: [rel_x, rel_y, speed, heading, accel, age_ms]   ├──►  φ(e_i)   │
│      ...                                                   │     (shared) │
│    Peer n: [rel_x, rel_y, speed, heading, accel, age_ms]  ─┘               │
│                                                                             │
│              φ = MLP(6 → 32 → 32)                                          │
│              Produces 32-dim embedding per peer                            │
│                                                                             │
│                                                                             │
│  STAGE 2: Permutation-Invariant Aggregation                                │
│  ──────────────────────────────────────────                                │
│                                                                             │
│    {h_1, h_2, ..., h_n}  →  z = MAX_POOL(h_1, ..., h_n)  →  z ∈ ℝ³²      │
│                                                                             │
│    Properties:                                                              │
│    • Output z is FIXED SIZE (32 dims) regardless of n                      │
│    • Permutation invariant: shuffling peers doesn't change z               │
│    • MAX captures "most threatening" peer signal                           │
│                                                                             │
│                                                                             │
│  STAGE 3: Policy Head                                                       │
│  ────────────────────                                                       │
│                                                                             │
│    ┌─────────────┐     ┌─────────────┐                                     │
│    │ Ego State   │     │ Peer Latent │                                     │
│    │ s ∈ ℝ⁴     │     │ z ∈ ℝ³²    │                                     │
│    └──────┬──────┘     └──────┬──────┘                                     │
│           │                   │                                             │
│           └───────┬───────────┘                                             │
│                   ▼                                                         │
│           ┌─────────────────┐                                              │
│           │ π = MLP(36→64→4)│                                              │
│           └────────┬────────┘                                              │
│                    ▼                                                        │
│           Action ∈ {0: Maintain, 1: Caution, 2: Brake, 3: Emergency}       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Why Max Pooling?

| Pooling | Use Case | RoadSense Fit |
|---------|----------|---------------|
| Mean | Homogeneous crowds | Averages away threats |
| Sum | Counting/density | Scales with n |
| **Max** | Detect closest/strongest | **Best for collision avoidance** |
| Attention | Learned importance | Too heavy for ESP32 |

**Rationale:** For collision avoidance, the single most threatening vehicle dominates. If one car is 5m away braking hard, it doesn't matter if 7 others are 100m away.

---

## 4. Observation Space Design

### Ego State (Fixed, 4 dimensions)

| Feature | Description | Range | Normalization |
|---------|-------------|-------|---------------|
| `speed` | Ego velocity | 0–30 m/s | / 30.0 |
| `accel` | Ego acceleration | -6 to +4 m/s² | / 10.0 |
| `heading` | Ego heading | -π to +π | / π |
| `peer_count` | Number of visible peers | 0–8 | / 8.0 |

### Per-Peer Features (Variable n, 6 dimensions each)

| Feature | Description | Source | Notes |
|---------|-------------|--------|-------|
| `rel_x` | Relative X (ego-centric) | V2V message | Forward = positive |
| `rel_y` | Relative Y (ego-centric) | V2V message | Left = positive |
| `rel_speed` | Relative velocity | V2V message | Approaching = negative |
| `rel_heading` | Heading difference | V2V message | Radians |
| `accel` | Peer's acceleration | V2V message | m/s² |
| `age_ms` | **Message staleness** | Emulator/HW | **Critical for Sim2Real** |

### Why `age_ms` is Critical

This feature encodes **communication uncertainty**:
- Fresh message (age < 50ms): Trust it
- Stale message (age > 200ms): Be cautious
- Very stale (age > 500ms): Possibly lost, fallback heuristics

The agent learns to discount information based on staleness.

---

## 5. Network Architecture

### PyTorch Implementation

```python
class DeepSetPolicy(nn.Module):
    """
    Permutation-invariant policy for variable-n peer observations.
    """

    def __init__(self,
                 peer_feature_dim: int = 6,
                 ego_feature_dim: int = 4,
                 embed_dim: int = 32,
                 hidden_dim: int = 64,
                 n_actions: int = 4):
        super().__init__()

        # Shared peer encoder (φ)
        self.peer_encoder = nn.Sequential(
            nn.Linear(peer_feature_dim, embed_dim),
            nn.ReLU(),
            nn.Linear(embed_dim, embed_dim),
            nn.ReLU()
        )

        # Policy head (π)
        self.policy_head = nn.Sequential(
            nn.Linear(embed_dim + ego_feature_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, n_actions)
        )

    def forward(self, ego_state: torch.Tensor, peer_features: torch.Tensor,
                peer_mask: torch.Tensor) -> torch.Tensor:
        """
        Args:
            ego_state: (batch, 4) - ego features
            peer_features: (batch, max_peers, 6) - peer features (padded)
            peer_mask: (batch, max_peers) - 1 for valid peers, 0 for padding

        Returns:
            action_logits: (batch, 4)
        """
        batch_size = ego_state.shape[0]

        # Encode all peers with shared MLP
        # Shape: (batch, max_peers, 6) -> (batch, max_peers, 32)
        peer_embeddings = self.peer_encoder(peer_features)

        # Apply mask (set padding embeddings to -inf for max pool)
        mask_expanded = peer_mask.unsqueeze(-1).expand_as(peer_embeddings)
        peer_embeddings = peer_embeddings.masked_fill(~mask_expanded.bool(), float('-inf'))

        # Max pooling over peers
        # Shape: (batch, max_peers, 32) -> (batch, 32)
        z, _ = peer_embeddings.max(dim=1)

        # Handle case where all peers are masked (n=0)
        z = torch.where(
            peer_mask.any(dim=1, keepdim=True),
            z,
            torch.zeros_like(z)
        )

        # Concatenate with ego state
        # Shape: (batch, 32) + (batch, 4) -> (batch, 36)
        combined = torch.cat([z, ego_state], dim=1)

        # Policy head
        return self.policy_head(combined)
```

### Gymnasium Observation Space

```python
from gymnasium import spaces

observation_space = spaces.Dict({
    "ego": spaces.Box(
        low=np.array([0, -1, -1, 0]),
        high=np.array([1, 1, 1, 1]),
        dtype=np.float32
    ),
    "peers": spaces.Box(
        low=-np.inf,
        high=np.inf,
        shape=(MAX_PEERS, 6),  # Padded to MAX_PEERS
        dtype=np.float32
    ),
    "peer_mask": spaces.Box(
        low=0,
        high=1,
        shape=(MAX_PEERS,),
        dtype=np.float32
    )
})
```

---

## 6. ESP32 Deployment Constraints

### Memory Budget

| Component | Estimated Size | Notes |
|-----------|----------------|-------|
| Peer encoder (φ) | ~15 KB | 6→32→32 MLP |
| Policy head (π) | ~10 KB | 36→64→4 MLP |
| Tensor arena | 8 KB | TFLite Micro |
| Peer embeddings buffer | 2 KB | 8 peers × 32 floats × 4 bytes |
| **Total** | **~35 KB** | Fits in ESP32 |

### Inference Loop (C++)

```cpp
void runInference(float ego_state[4], PeerData peers[], int peer_count) {
    // 1. Encode each peer with shared weights
    float h_max[32];
    std::fill(h_max, h_max + 32, -INFINITY);

    for (int i = 0; i < peer_count; i++) {
        float peer_features[6] = {
            peers[i].rel_x,
            peers[i].rel_y,
            peers[i].rel_speed,
            peers[i].rel_heading,
            peers[i].accel,
            peers[i].age_ms / 500.0f  // Normalize
        };

        float h[32];
        runPeerEncoder(peer_features, h);  // TFLite inference

        // Max pooling
        for (int j = 0; j < 32; j++) {
            if (h[j] > h_max[j]) h_max[j] = h[j];
        }
    }

    // 2. Handle n=0 case
    if (peer_count == 0) {
        std::fill(h_max, h_max + 32, 0.0f);
    }

    // 3. Concatenate and run policy head
    float input[36];
    std::copy(h_max, h_max + 32, input);
    std::copy(ego_state, ego_state + 4, input + 32);

    int action = runPolicyHead(input);  // TFLite inference
    executeAction(action);
}
```

### TFLite Model Structure

The model is split into two TFLite models for ESP32:
1. `peer_encoder.tflite` - Called n times (once per peer)
2. `policy_head.tflite` - Called once after aggregation

This split avoids dynamic shapes in TFLite Micro.

---

## 7. Affected Components

### Files to Update

| File | Change Required | Priority |
|------|-----------------|----------|
| `ml/envs/convoy_env.py` | Dict observation space, variable peers | HIGH |
| `ml/espnow_emulator/espnow_emulator.py` | Support n peers (not hardcoded V002/V003) | HIGH |
| `hardware/src/ml_inference.cpp` | Loop-based peer encoding | HIGH |
| `docs/SDD/SafeRide_SDD_2025_2.md` | Update ML Design section | HIGH |
| `ARCHITECTURE_V2_MINIMAL_RL.md` | Mark as superseded | MEDIUM |
| `CONVOY_ENV_IMPLEMENTATION_PLAN.md` | Update observation spec | MEDIUM |

### Breaking Changes

1. **Observation space shape changes**: From `Box(11,)` to `Dict`
2. **Training code**: Must use custom policy network (not MlpPolicy)
3. **Firmware**: ML inference code restructured

---

## 8. Migration Path

### Phase 1: Training Infrastructure (Priority)

1. Implement `DeepSetPolicy` in PyTorch
2. Update `ConvoyEnv` to emit Dict observations
3. Update ESP-NOW emulator to handle n peers
4. Create custom feature extractor for stable-baselines3

### Phase 2: Firmware Integration

1. Split model into peer_encoder + policy_head
2. Implement max-pooling in C
3. Update inference loop
4. Validate on ESP32

### Phase 3: Validation

1. Train with 3-vehicle scenarios (baseline)
2. Test with 6-8 vehicle scenarios (scalability)
3. Validate Sim2Real gap

---

## References

- **Professor's Document:** `docs/professors_research_for_RL.pdf`
- **Deep Sets Paper:** Zaheer et al., 2017 - "Deep Sets"
- **Set Transformer:** Lee et al., 2019 - "Set Transformer"
- **RL for Autonomous Driving:** Same pattern used in Waymo, Tesla simulation

---

## Document Changelog

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-01-13 | Amir Khalifa | Initial document based on professor's guidance |

---

**END OF DOCUMENT**
