4. Design
4.1 Machine Learning Design
This section describes the Reinforcement Learning (RL) subsystem that enables V001 (the Ego vehicle) to make real-time collision avoidance decisions based on V2V messages received from peer vehicles.
Core Architecture Principle: Only one vehicle runs the ML model. Other nodes in the network are "dumb broadcasters" that transmit their state but perform no inference.

4.1.1 The Sim2Real Challenge
Training an RL agent in simulation and deploying it on real hardware introduces a critical problem: the Simulation to Reality gap. SUMO provides perfect vehicle state data, but real ESP-NOW communication exhibits:

Effect
Measured Range
Impact on Agent
Latency
10-80 ms
Delayed state information
Packet Loss
2-15%
Missing updates
Jitter
±30 ms variance
Inconsistent timing
Message Staleness
0-500ms
Outdated Decisions


Solution: An ESP-NOW Emulator built from real hardware measurements injects communication impairments during training. The agent learns to make safe decisions despite imperfect information.
4.1.2 Architecture Overview
The ego vehicle receives V2V broadcasts from n neighboring vehicles, where n is unknown and changes dynamically. The policy uses a permutation-invariant set encoder (Deep Sets) to handle this variable input.

4.1.3 Observation Space
Ego State (4 Dimension):

Feature
Description
Range
speed
Ego velocity (m/s)
Current speed from GPS/IMU
accel
Ego acceleration (m/s^2)
Current longitudinal acceleration
heading
Ego heading (rad)
Current heading from mag
peer_count
Number of visible peers
0-n


Per-Peer Features (6 dimensions each, n peers)

Feature
Description
Range
rel_x, rel_y
Relative position (ego-centric)
Forward/left = positive
rel_speed
Relative velocity 
Approaching = negative
rel_heading
Heading difference
Radians
accel
Peer's acceleration
From V2V message
age_ms
Message staleness
Critical for Sim2Real


The age_ms feature encodes communication uncertainty—the agent learns to discount stale information.



4.1.4 Reward Function

Condition
Reward
Rationale
Collision
-100
Terminal failure
TTC < 2.0 s
−10
Dangerous proximity 
Safe following (headway 1.5–3.0 s)
+1
Target operating zone
Harsh braking (> 4.5 m/s²)
-5
Passenger discomfort
False positive (alert when safe)
-2
Reduce nuisance alerts


4.1.5 ESP-NOW Emulator
The emulator injects realistic communication impairments based on measured ESP32 hardware data:

  {
    "latency": { "base_ms": 12, "jitter_std_ms": 8 },
    "packet_loss": { "base_rate": 0.02, "high_loss_rate": 0.15 },
    "domain_randomization": {
      "latency_range_ms": [5, 80],
      "loss_rate_range": [0.0, 0.20]
    }
  }

Domain Randomization: Training uses parameter ranges 1.5× wider than measured to ensure policy robustness.


** Emulator in system architecture** its fine no comments about it

4.1.6 Deployment on ESP32
The model is split into two TFLite models for efficient inference:
Peer_encoder.tflite - Called once per peer (n times)
Policy_head.tflite - Called once after max-pooling

Memory Budget:
Peer Encoder = ~15 KB
Policy head = ~10 KB
Tensor arena = 8 KB
Total = <40 KB

Inference Loop (Pseudocode):
  // 1. Encode each peer, track max embedding
  float h_max[32] = {-INF};
  for (int i = 0; i < peer_count; i++) {
      float h[32];
      peer_encoder_forward(peer[i], h);
      for (int j = 0; j < 32; j++)
          h_max[j] = max(h_max[j], h[j]);
  }

  // 2. Concatenate ego state + pooled embedding, run policy
  float input[36] = concat(ego_state, h_max);
  int action = policy_head_forward(input);

4.1.7 Safety Overrides
Firmware-level rules that supersede ML output:
Condition: All peers stale (age > 500ms ) -> Force Caution
Condition: TTC < 1.0 s (rule-based) -> Force Emergency
Condition: ML inference timeout -> Fallback heuristic

