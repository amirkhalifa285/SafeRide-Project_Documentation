# Phase 3 & 4 Specification: Causality Fix & Training Pipeline

**Document:** `PHASE_3_4_SPEC.md`
**Status:** READY FOR IMPLEMENTATION
**Target Agent:** BUILDER
**Prerequisites:** Phase 2 (Deep Sets Policy) - COMPLETE

---

## Phase 3: Emulator Causality & Cleanup

**Objective:** Fix a critical Sim2Real causality violation in `ConvoyEnv` and prepare `ESPNOWEmulator` for variable-N peer support.

### 1. The Causality Bug
Currently, `ConvoyEnv` uses the return value of `emulator.transmit()` immediately.
- **Bug:** `transmit()` returns the message *before* latency is applied (it queues it for future delivery).
- **Result:** The agent sees messages `latency_ms` before they should physically arrive.
- **Fix:** `ConvoyEnv` must transmit all messages first (fire-and-forget), then **poll** the emulator for messages that have actually arrived by `current_time`.

### 2. `ml/espnow_emulator/espnow_emulator.py` Updates

#### A. Add `sync_and_get_messages()` Method
Add a new method to retrieve currently valid messages as objects (not flattened dict).

```python
def sync_and_get_messages(self, current_time_ms: int) -> Dict[str, ReceivedMessage]:
    """
    Process pending queue to deliver arrived messages, then return all valid messages.
    
    This replaces get_observation() for the N-element architecture.
    It separates the 'receive' logic from the 'format' logic.
    
    Args:
        current_time_ms: Current simulation time
        
    Returns:
        Dict mapping vehicle_id -> ReceivedMessage for all received (non-stale) messages.
    """
    # 1. Process pending messages (same logic as get_observation)
    while not self.pending_messages.empty():
        try:
            arrival_time, _seq, vehicle_id, received_msg = self.pending_messages.queue[0]
        except IndexError:
            break
            
        if arrival_time <= current_time_ms:
            self.pending_messages.get()
            self.last_received[vehicle_id] = received_msg
        else:
            break
            
    # 2. Return all valid messages in buffer
    # Note: Logic for staleness/validity happens here or in env?
    # Let's return the raw buffer and let Env filter by age if needed, 
    # OR filter here to be consistent with 'get_observation'.
    # Decision: Return raw buffer (last_received), Env handles staleness filtering.
    return self.last_received.copy()
```

### 3. `ml/envs/convoy_env.py` Updates

#### A. Update `_step_espnow`
Refactor to decouple transmission from reception.

```python
def _step_espnow(self, ego_state: VehicleState, current_time_ms: int) -> ...:
    # 1. TRANSMIT PHASE (Fire and Forget)
    # Loop over all peers and transmit
    # DO NOT collect return values here for observation
    for vehicle_id in traci.vehicle.getIDList():
        if vehicle_id == self.EGO_VEHICLE_ID: continue
        
        peer_state = self.sumo.get_vehicle_state(vehicle_id)
        msg = peer_state.to_v2v_message(timestamp_ms=current_time_ms)
        
        # transmit() queues the message if successful
        self.emulator.transmit(
            sender_msg=msg,
            sender_pos=(peer_state.x, peer_state.y),
            receiver_pos=(ego_state.x, ego_state.y),
            current_time_ms=current_time_ms
        )

    # 2. RECEIVE PHASE (Poll for arrivals)
    # Only get messages that have actually arrived by current_time_ms
    received_map = self.emulator.sync_and_get_messages(current_time_ms)
    
    # 3. BUILD OBSERVATION
    peer_observations = []
    for vid, received in received_map.items():
        # Filter logic (moved from Env or kept?)
        # Calculate age
        age_ms = current_time_ms - received.received_at_ms + received.age_ms
        
        # Append to list for ObservationBuilder
        peer_observations.append({
             # ... map fields ...
             "age_ms": age_ms,
             "valid": True # Validity is handled by ObservationBuilder logic
        })
        
    # ... call obs_builder ...
```

---

## Phase 4: Training Pipeline

**Objective:** Implement the training script and verify the complete N-Element pipeline (Env → Emulator → DeepSetPolicy).

### 1. New File: `ml/training/train_convoy.py`

Create a standard Stable-Baselines3 training script.

#### Requirements:
1. **Imports:**
   - `gymnasium`
   - `stable_baselines3` (PPO)
   - `ml.envs` (to register env)
   - `ml.policies.deep_set_policy` (for `create_deep_set_policy_kwargs`)

2. **Configuration:**
   - `TOTAL_TIMESTEPS = 100_000`
   - `ENTROPY_COEF = 0.01`
   - `LEARNING_RATE = 3e-4`
   - `N_STEPS = 2048`

3. **Main Function:**
   - Initialize `RoadSense-Convoy-v0`.
   - Initialize `PPO` with `MultiInputPolicy`.
   - **Crucial:** Use `policy_kwargs=create_deep_set_policy_kwargs(peer_embed_dim=32)`.
   - Setup `TensorboardCallback` (or just `tensorboard_log` param).
   - Run `model.learn()`.
   - Save model to `ml/models/saved/`.

#### Example Code Structure:

```python
import os
import gymnasium as gym
from stable_baselines3 import PPO
from stable_baselines3.common.callbacks import CheckpointCallback
from ml.policies.deep_set_policy import create_deep_set_policy_kwargs
import ml.envs  # Register envs

def train():
    env = gym.make("RoadSense-Convoy-v0", sumo_cfg="...", render_mode=None)
    
    policy_kwargs = create_deep_set_policy_kwargs(peer_embed_dim=32)
    
    model = PPO(
        "MultiInputPolicy",
        env,
        policy_kwargs=policy_kwargs,
        verbose=1,
        tensorboard_log="./logs/tensorboard/"
    )
    
    checkpoint_callback = CheckpointCallback(
        save_freq=10000,
        save_path="./logs/checkpoints/",
        name_prefix="deep_sets_v1"
    )
    
    print("Starting training...")
    model.learn(total_timesteps=100_000, callback=checkpoint_callback)
    model.save("ml/models/saved/deep_sets_final")

if __name__ == "__main__":
    train()
```

### 2. TDD / Verification Steps

**Test 4.1: Integration Test**
Create `ml/tests/integration/test_training_pipeline.py`.
- Run a short training loop (e.g., 100 steps) inside the test.
- Assert no exceptions.
- Assert model file is created.

---

## Files Touched
- `ml/espnow_emulator/espnow_emulator.py` (Modify)
- `ml/envs/convoy_env.py` (Modify)
- `ml/training/train_convoy.py` (Create)
- `ml/tests/integration/test_training_pipeline.py` (Create)

## Acceptance Criteria
1. `ConvoyEnv` correctly delays message perception (causality test).
2. `train_convoy.py` runs without error using `DeepSetPolicy`.
3. Observations passed to model contain correct N-element Dict structure.
