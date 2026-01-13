# Phase 3 Implementation Plan: Latency Modeling

**Document Type:** AI Agent Implementation Prompt
**Target Component:** `ESPNOWEmulator` (Latency Calculation)
**Methodology:** Test-Driven Development (TDD) with Statistical Verification
**Priority:** High (Critical for Sim2Real fidelity)

---

## 1. Objective
Verify and robustify the `_get_latency` method in `ESPNOWEmulator`. The latency model must accurately simulate:
1.  **Base Latency:** Fixed processing overhead.
2.  **Distance Factor:** Propagation delay or increased retransmissions at range.
3.  **Jitter:** Stochastic variance (Gaussian distribution).
4.  **Clamping:** Latency must never be less than 1 ms.
5.  **Domain Randomization:** When enabled, the base latency must vary per episode.

---

## 2. Technical Specification

### 2.1 The Mathematical Model
The latency $L$ for a message at distance $d$ is calculated as:

$$ L = \max(1.0, L_{base} + (d \times F_{dist}) + \mathcal{N}(0, \sigma_{jitter})) $$

Where:
*   $L_{base}$: Base latency (from params or randomized per episode).
*   $d$: Euclidean distance in meters.
*   $F_{dist}$: Distance factor (ms/meter).
*   $\mathcal{N}(0, \sigma_{jitter})$: Gaussian noise with mean 0 and standard deviation `jitter_std_ms`.

### 2.2 Domain Randomization Interaction
*   **If DR is OFF:** $L_{base} = \text{params['latency']['base_ms']}$
*   **If DR is ON:** $L_{base}$ is sampled from `latency_range_ms` at `reset()` and stored in `self.episode_latency_base`.

---

## 3. TDD Plan: Write These Tests FIRST

Create `ml/tests/test_latency_modeling.py`.

### Test Group A: Deterministic Logic (Mocking Randomness)
*Use `unittest.mock.patch` or `mocker` to force `random.gauss` to return 0.*

1.  `test_latency_deterministic_base()`:
    *   Set distance = 0, mock jitter = 0.
    *   Verify result exactly equals `base_ms`.
2.  `test_latency_distance_scaling()`:
    *   Set distance = 100m, mock jitter = 0.
    *   Verify result equals `base_ms + (100 * distance_factor)`.
3.  `test_latency_minimum_clamp()`:
    *   Set `base_ms` = 5.
    *   Mock jitter = -20 (forcing theoretical result < 0).
    *   Verify result is exactly 1.0.

### Test Group B: Statistical Validation
*Run the function $N=1000$ times and analyze the distribution.*

4.  `test_latency_distribution_statistics()`:
    *   Config: Base=20, Dist=0, JitterStd=5.
    *   Run 1000 times.
    *   **Verify Mean:** approx 20 (± 0.5).
    *   **Verify StdDev:** approx 5 (± 0.5).
    *   *Note: Use a fixed random seed for reproducibility.*

### Test Group C: Domain Randomization
5.  `test_latency_domain_randomization()`:
    *   Enable domain randomization.
    *   Call `reset()` multiple times.
    *   Verify that `_get_latency(dist=0)` returns different "center" values (different base latencies) across episodes, but stays within the configured range.

---

## 4. Implementation Steps

### Step 4.1: Review Existing Code
Check the current `_get_latency` implementation in `espnow_emulator.py`. It likely already exists from the initial draft but needs to be verified against the specs above.

### Step 4.2: Refine `_get_latency` (if needed)
Ensure it uses `self.episode_latency_base` correctly.

### Step 4.3: Refine `reset()` (if needed)
Ensure `_randomize_episode_params` is called and correctly populates `self.episode_latency_base`.

---

## 5. Execution Instructions for Agent

1.  **Context**: You are working in `roadsense-v2v/ml`.
2.  **Action 1**: Create `tests/test_latency_modeling.py` with the 5 tests defined above.
3.  **Action 2**: Run tests. If they fail (e.g. if the current implementation doesn't handle DR correctly), fix the implementation.
4.  **Action 3**: Ensure `pytest` passes with high confidence.
