# Phase 2 Implementation Plan: Parameter Loading & Configuration

**Document Type:** AI Agent Implementation Prompt
**Target Component:** `ESPNOWEmulator` (Initialization & Config Loading)
**Methodology:** Test-Driven Development (TDD)
**Priority:** High (Foundation for all simulation logic)

---

## 1. Objective
Implement robust parameter loading for the `ESPNOWEmulator` class. The system must support:
1.  Loading configuration from a JSON file.
2.  Falling back to safe defaults if no file is provided.
3.  **Recursive Merging:** Allowing partial JSON files (e.g., overriding only 'latency' settings while keeping other defaults).
4.  **Validation:** Handling corrupted files, invalid types, and logical inconsistencies (e.g., negative latency).

---

## 2. Technical Specification

### 2.1 The "Recursive Merge" Strategy
To ensure the emulator never crashes due to a missing configuration key, we will implement a **Deep Merge** strategy:
1.  Start with `_default_params()` (Complete, valid configuration).
2.  If `params_file` is provided:
    *   Load JSON content.
    *   Recursively update the default dictionary with the JSON content.
    *   *Example:* If JSON is `{"latency": {"base_ms": 50}}`, only `base_ms` is updated; `jitter_std_ms` remains at default.

### 2.2 Validation Logic
After merging, the configuration must be validated:
*   **Types:** Ensure numbers are numbers, lists are lists.
*   **Ranges:** Latency > 0, Probabilities between 0.0 and 1.0.
*   **Logic:** `distance_threshold_1` < `distance_threshold_2`.

---

## 3. TDD Plan: Write These Tests FIRST

Create `ml/tests/test_parameter_loading.py`. Do **NOT** implement the logic until these tests exist and fail (Red-Green-Refactor).

### Test Group A: Basic Initialization
1.  `test_init_defaults()`: Initialize with `params_file=None`. Verify internal `self.params` matches `_default_params`.
2.  `test_init_with_valid_file()`: Initialize with a valid `emulator_params.json` (use fixture). Verify values match file.

### Test Group B: File Handling Edge Cases
3.  `test_init_file_not_found()`: Pass a non-existent path. Expect `FileNotFoundError` (don't silently fall back to defaults, explicit failure is better for debugging).
4.  `test_init_corrupted_json()`: Pass a file containing invalid JSON syntax (e.g., `{ "key": `). Expect `json.JSONDecodeError`.

### Test Group C: Recursive Merging (The "Partial Config" Case)
5.  `test_recursive_merge_logic()`:
    *   Create a temporary JSON file with ONLY `{"latency": {"base_ms": 999}}`.
    *   Initialize emulator.
    *   **Verify:** `params['latency']['base_ms']` is 999.
    *   **Verify:** `params['latency']['jitter_std_ms']` is still the default (e.g., 8).

### Test Group D: Data Validation & Logic
6.  `test_invalid_probability_range()`: JSON with `packet_loss: { base_rate: 1.5 }`. Expect `ValueError`.
7.  `test_negative_latency()`: JSON with `latency: { base_ms: -10 }`. Expect `ValueError`.
8.  `test_inverted_thresholds()`: JSON where `threshold_1` (e.g., 100) > `threshold_2` (e.g., 50). Expect `ValueError`.

---

## 4. Implementation Steps

### Step 4.1: Utility Method `_recursive_update`
Implement a static or helper method to merge dictionaries deep:
```python
def _recursive_update(base: dict, update: dict) -> dict:
    for k, v in update.items():
        if isinstance(v, dict) and k in base:
            _recursive_update(base[k], v)
        else:
            base[k] = v
    return base
```

### Step 4.2: Validation Method `_validate_params`
Implement a method to sanity-check the final dictionary:
*   Check probabilities (0-1).
*   Check positive constraints (time, distance).

### Step 4.3: Update `__init__`
Refactor `__init__` to:
1.  Load defaults.
2.  If file exists, load and `_recursive_update`.
3.  Call `_validate_params`.

---

## 5. Edge Case Checklist (Ultrathinking)

- [ ] **Empty JSON `{}`**: Should result in default parameters (valid).
- [ ] **Wrong Types**: If user provides `"base_ms": "fast"`, Python might crash later during math. Validation step should catch `isinstance(val, (int, float))`.
- [ ] **Extra Keys**: If JSON has `{"future_feature": 1}`, ignore or allow? *Decision: Allow (forward compatibility).*
- [ ] **Permission Error**: If file exists but unreadable. Let standard `IOError` propagate.

---

## 6. Execution Instructions for Agent
1.  **Context**: You are working in `roadsense-v2v/ml` Verify venv is activated.
2.  **Action 1**: Create `tests/test_parameter_loading.py` with the 8 tests defined above.
3.  **Action 2**: Run tests -> Expect Failures (Red).
4.  **Action 3**: Modify `espnow_emulator/espnow_emulator.py` to implement recursive merging and validation.
5.  **Action 4**: Run tests -> Expect Pass (Green).
