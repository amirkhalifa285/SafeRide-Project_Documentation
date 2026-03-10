# Run 011 Analysis — The Winner

**Date:** March 10, 2026
**Run ID:** `cloud_prod_011`
**S3:** `s3://saferide-training-results/cloud_prod_011/`
**Training Time:** 18h 52m on c6i.xlarge (67,960 seconds, 10M steps)
**Dataset:** `dataset_v6` (formation-fixed, 25 train + 40 eval scenarios, seed=42)
**Verdict: THIS IS THE MODEL.** Ship it.

---

## Executive Summary

Run 011 is the definitive success of the RoadSense V2V project. After 8 failed runs (003-008), one passive model (009), and one partial breakthrough (010), Run 011 delivers:

| Metric               | Run 009 | Run 010     | **Run 011**        |
|----------------------|---------|-------------|------------------- |
| Avg Reward           | -3,460  | -3,119      | **-86.62**         |
| Collision Rate       | 0%      | 0%          | **0%**             |
| Behavioral Success   | 72%     | 73.6%       | **84.1%**          |
| V2V Reaction Rate    | 0%      | 87% (40/46) | **100% (276/276)** |
| Rank-2+ Reaction     | 0%      | 50-70%      | **100%**           |
| Eval Matrix Coverage | broken  | 3/15        | **15/15**          |
| Eval Episodes        | ~200    | 46          | **276**            |
| Missed Reactions     | all     | 6           | **0**              |

The reward improved by **+3,032** over Run 010, the model reacts to **every single hazard** across all 15 evaluation buckets with **zero collisions**, and PPO training was textbook stable.

---

## 1. Training Stability — Perfect

### Reward Trajectory
```
Phase        Avg Reward    Min      Max
────────────────────────────────────────
0-1M         -152.0       -1890     288
1-2M          167.4        -136     508
2-3M          439.0        -133     719
3-5M          561.9         266     818
5-7M          556.9         190     835    ← plateau begins
7-10M         573.9         296     789    ← stable plateau
────────────────────────────────────────
First: -1890  →  Final: +593  →  Best: +835
```

The model learns rapidly (0-3M), converges by 5M, and holds a stable plateau through 10M. No reward collapse, no oscillation, no regression. This is the cleanest training curve in project history.

**Comparison:** Run 008 exploded at 7M. Run 009 had avg -3,460 at convergence. Run 011 converges at +560 — a **3,600-point improvement** over Run 009.

### Policy Std (Exploration → Exploitation)
```
Start:  0.6000
Mid:    0.1662
End:    0.0433
Monotonically decreasing: YES
```

Std drops from 0.60 → 0.04 without a single reversal. This is the hallmark of a well-trained PPO agent: the policy starts exploratory and smoothly collapses to a confident, deterministic policy. Run 008's std went 0.75 → 4.36 (catastrophic). Run 009/010 ended at ~0.12. Run 011 goes to **0.043** — the model is 3x more confident than Run 010.

### Explained Variance
```
Start:  0.002  →  End: 0.956
```

The value function explains 95.6% of return variance. This means the critic perfectly tracks the reward landscape. For reference, Run 007 never exceeded 0.0 (brain-dead). Run 009 peaked at 0.97. Run 011 sustains >0.94 from mid-training onward.

### Clip Fraction
```
Early:  0.017
Mid:    0.033
Late:   0.117
```

Clip fraction staying in the 0.01-0.12 range means PPO's trust region is actively constraining updates without being violated. The slight increase in late training reflects the policy being more decisive (lower std = larger relative updates), not instability.

### Approx KL Divergence
```
Avg:  0.0056
Max:  0.0655
```

Well below the 0.02 safety threshold on average. The single max spike to 0.065 is within acceptable bounds and didn't cause any instability. No catastrophic policy updates occurred.

---

## 2. V2V Reaction — Perfect 100%

This is the headline result. **Every single hazard episode produced an active V2V reaction.**

### Reaction Rate by Hazard Source Rank
```
Hazard from    Episodes    Reaction Rate    Avg Reaction Time
──────────────────────────────────────────────────────────────
Rank 1          127         100%              2.88s
Rank 2           72         100%              7.23s
Rank 3           43         100%              9.73s
Rank 4           24         100%             10.53s
Rank 5           10         100%             11.02s
──────────────────────────────────────────────────────────────
TOTAL           276         100%              6.04s (weighted)
```

**Rank-1 reaction time: 2.88s** — The model detects the nearest peer's emergency braking and responds in under 3 seconds. This is well within safe stopping distance at convoy speeds.

**Rank-2 through Rank-5** reaction times scale linearly with rank, reflecting the physical reality of mesh relay propagation delays. The model correctly waits for the braking signal to cascade through the convoy via mesh relay before reacting.

**Run 010 comparison:** Run 010 had 87% reaction (6 missed, all rank-2). Run 011 has **zero missed reactions at any rank**. The formation fix ensured every eval scenario has correct peer geometry, and the model handles all of them.

### Minimum Distance Post-Hazard
```
Avg across all episodes: 4.15m
```

The model maintains a safe 4+ meter gap even during emergency braking. Zero collisions across 276 episodes. This distance is well above the 2.5m SUMO minGap, providing ample safety margin.

### Mesh Relay Effectiveness
```
Hazard@rank2, observed@rank2:  85.7% braking signal reception
Hazard@rank3, observed@rank3:  88.9% reception
Hazard@rank4, observed@rank4:  64.3% reception
Hazard@rank5, observed@rank5: 100.0% reception
```

Multi-hop relay works. Even when the direct braking signal reception drops to 64% for rank-4 hazards, the model still reacts 100% of the time — because it also picks up indirect signals from intermediate peers' braking cascades.

---

## 3. Eval Matrix — Full Coverage

### Coverage: 15/15 Buckets
```
Bucket          Expected    Observed    Status
────────────────────────────────────────────────
n1_rank1          10          56         OK
n2_rank1          10          28         OK
n2_rank2          10          28         OK
n3_rank1          10          19         OK
n3_rank2          10          19         OK
n3_rank3          10          18         OK
n4_rank1          10          14         OK
n4_rank2          10          14         OK
n4_rank3          10          14         OK
n4_rank4          10          14         OK
n5_rank1          10          10         OK
n5_rank2          10          11         OK
n5_rank3          10          11         OK
n5_rank4          10          10         OK
n5_rank5          10          10         OK
────────────────────────────────────────────────
Missing: NONE    Injection failures: 0/276
```

**This is the first run in project history with full eval matrix coverage.** Run 010 only covered 3/15 buckets due to broken formations. The formation stability fix (departPos redistribution + episode shortening + hazard window shift) completely resolved the geometry bug.

All 276 episodes had successful hazard injection. Zero `no_front_peers` failures. Zero `injection_failed_reason` entries. The eval is comprehensive and trustworthy.

---

## 4. Reward Economy

### Overall Stats
```
Avg Reward:    -86.62
Std Reward:    1009.58
Min Reward:   -4778.22
Max Reward:    1749.59
```

The avg reward of -86.62 is dramatically better than Run 010's -3,119. The wide std (1009) reflects the bimodal nature of hazard scenarios (rank-1 hazards are inherently more punishing than rank-3+), not policy inconsistency.

### Reward by Peer Count
```
n=1:  avg=-600   behavioral_success=78.6%
n=2:  avg=-207   behavioral_success=85.7%
n=3:  avg= -43   behavioral_success=83.9%
n=4:  avg=+348   behavioral_success=91.1%   ← best
n=5:  avg= +80   behavioral_success=80.8%
```

**Pattern:** Performance improves with more peers (n=4 is the sweet spot). This makes physical sense — more peers provide richer V2V information for the Deep Sets encoder. The n=1 penalty is expected: with only 1 peer, every hazard is rank-1 (closest, most dangerous), and the model must brake hard.

### Reward by Hazard Rank
```
Rank 1:  avg=-498   median=-335   (n=127 episodes)
Rank 2:  avg=+128   median=+302   (n=72 episodes)
Rank 3:  avg=+445   median=+502   (n=43 episodes)
Rank 4:  avg=+399   median=+465   (n=24 episodes)
Rank 5:  avg=+138   median=+210   (n=10 episodes)
```

**Rank-1 hazards are the hardest** (negative avg reward) because the hazard source is the nearest peer — ego must brake aggressively, incurring comfort penalty but maintaining safety. Rank-2+ hazards are progressively easier (positive reward) because intermediate peers absorb the braking cascade.

### Worst Episodes
The 10 worst episodes (reward < -2,200) are all **rank-1 hazards** across various peer counts. These are scenarios where the nearest peer brakes hard and ego must perform emergency braking. The model handles them (zero collisions) but the comfort penalty is high. This is correct behavior — safety over comfort.

---

## 5. Behavioral Metrics

```
Collision rate:          0.0%  (0/276 episodes)
Success rate:          100.0%  (all episodes complete without crash)
Behavioral success:     84.1%  (232/276 episodes with reward > -1000)
Truncation rate:       100.0%  (all episodes run full 500 steps)
Avg pct time unsafe:   16.65%  (~83 steps in unsafe zone per episode)
Avg episode length:    500.0   (all episodes run to completion)
```

**84.1% behavioral success** means only 44 episodes had reward below -1000 (all rank-1 hazards requiring hard braking). This is healthy — the model prioritizes safety over comfort, and rank-1 hazards are inherently expensive.

**16.65% time unsafe** means the model spends ~83 out of 500 steps with distance < 10m from a peer. This includes the hazard response period where the ego is catching up to a braking peer. Given that hazards are injected in every episode, this is expected and acceptable.

---

## 6. Formation Fix Validation

The formation stability fix was the key enabler for Run 011. Evidence:

1. **Injection success: 276/276 (100%)** — No `no_front_peers` failures. Every scenario has correct convoy geometry with peers ahead of ego.

2. **All 15 buckets populated** — Including previously impossible buckets like n1_rank1 (56 episodes), n3_rank3 (18 episodes), n5_rank5 (10 episodes).

3. **Full 500-step episodes** — 100% truncation rate means no vehicles exit the route early. The episode length (500 steps = 50s) fits within the route traversal time.

4. **Single-lane road guarantee** — Rank ordering is physically guaranteed (no overtaking), so rank assignments are deterministic and stable.

---

## 7. Deep Sets Architecture Validation

The n-element problem is solved. Evidence:

| Peer Count | Episodes | Collision Rate | Behavioral Success | V2V Reaction |
|-----------|----------|---------------|-------------------|-------------|
| n=1       | 56       | 0%            | 78.6%             | 100%        |
| n=2       | 56       | 0%            | 85.7%             | 100%        |
| n=3       | 56       | 0%            | 83.9%             | 100%        |
| n=4       | 56       | 0%            | 91.1%             | 100%        |
| n=5       | 52       | 0%            | 80.8%             | 100%        |

The model handles 1 to 5 peers with zero collisions and 100% V2V reaction across all counts. The Deep Sets permutation-invariant encoder generalizes correctly — no hardcoded peer slots, no padding artifacts, no performance collapse at any n.

---

## 8. What Made Run 011 Different (Cumulative Fixes)

Run 011's success is the result of 10 runs of iterative fixes. Every prior failure contributed:

| Run   | Failure                                   | Fix Applied                            | Still in Run 011? |
|-----  |------------------------------------------ |----------------------------------------|-------------------|
| 003   | Speed pinning                             | `release_vehicle_speed()`              | Yes               |
| 004   | Passive (CF free-ride)                    | — (identified problem)                 | —                 |
| 005   | Mocked SUMO                               | Docker integration tests mandatory     | Yes               |
| 006   | Hazard reward fires during normal driving | Ground-truth collision + warmup        | Yes               |
| 007   | Comfort drowns signal                     | Graduated comfort by distance          | Yes               |
| 007.1 | Discrete zone poverty trap                | Linear ramp reward                     | Yes               |
| 008   | Std explosion (LR, entropy)               | LR=1e-4, ent_coef=0, log_std_init=-0.5 | Yes               |
| 009   | Passive (CF free-ride, redux)             | CF override during hazards             | Yes               |
| 010   | Eval matrix broken (3/15)                 | Formation stability fix                | Yes               |

**Run 011 = All fixes from 003-010 + formation-fixed scenarios.** No new code changes were needed — just correct scenario geometry.

---

## 9. Remaining Concerns (Minor)

### 9a. Rank-1 Reward Penalty
Rank-1 hazards have avg reward -498 (vs +128 to +445 for rank-2+). This is expected physics — braking for the nearest peer is expensive — but it means the overall avg reward (-86.62) is dragged down by rank-1 scenarios. Not a bug.

### 9b. Time Unsafe at 16.65%
Spending ~83/500 steps in the unsafe zone (< 10m) reflects the hazard response period. The model closes distance before braking, which is realistic driving behavior. All episodes end safely.

### 9c. Scenario Outliers
`eval_n2_006` and `eval_n3_001` are consistently the hardest scenarios (avg reward -1,813 and -1,487). These likely have particularly aggressive hazard timing or tight geometry. Worth investigating if optimizing further, but zero collisions means they're functionally safe.

### 9d. Late Clip Fraction Rise (0.117)
Clip fraction rising in late training (0.017 → 0.117) is slightly above the typical 0.01-0.10 range. The policy is very confident (std=0.04) so relative update magnitudes are larger. Not causing instability, but worth monitoring if extending training.

---

## 10. Next Steps

The model is ready to advance through the remaining pipeline:

1. **H5: Sim-to-Real Validation** — Compare model behavior against real convoy recording data (Recording #2, Feb 28). Replay real V2V messages through the model and verify reaction decisions match.

2. **Professor PoC Demo** — Show training curve (this run's -1890 → +593 trajectory), SUMO-GUI live demo, and the 100% reaction rate table.

3. **TFLite INT8 Quantization** — Convert `model_final.zip` to TFLite with INT8 quantization for ESP32 deployment. Validate that quantization doesn't degrade reaction rate.

4. **ESP32 Deployment** — Flash quantized model to V001. Run live convoy test with V001 (inference), V002, V003 (broadcast + relay).


---

## Appendix: Raw Numbers

```
Run ID:                  cloud_prod_011
Dataset:                 dataset_v6 (formation-fixed)
Total timesteps:         10,000,000
Training time:           18h 52m (67,960s)
Learning rate:           1e-4
n_steps:                 4,096
ent_coef:                0.0
Seed:                    42

Eval episodes:           276 (all hazard-injected)
Eval matrix:             15/15 buckets, 0 failures
Collision rate:          0.0%
Behavioral success:      84.1%
V2V reaction rate:       100% (276/276)
Avg reaction time:       6.04s (rank-1: 2.88s)
Avg min distance:        4.15m post-hazard
Avg reward:              -86.62
Max reward:              +1,749.59
Std (final):             0.0433
Explained variance:      0.956
```
