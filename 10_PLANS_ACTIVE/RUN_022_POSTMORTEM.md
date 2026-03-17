# Run 022 Postmortem

**Date:** March 16, 2026
**Model:** `cloud_prod_022` (`roadsense-v2v/ml/models/runs/cloud_prod_022/model_final.zip`)
**Training length:** 2,000,000 PPO timesteps
**Plan doc:** `docs/10_PLANS_ACTIVE/RUN_022_REMOVE_EGO_HEADING.md`
**Final verdict:** **SUMO pass, replay fail.** Run 022 is **not promotable** to a 10M production run and **not deployable** as-is.

---

## 1. Executive Summary

Run 022 removed ego heading from the observation exactly as planned:
- ego observation changed from 6 dims to 5 dims
- the route-geometry shortcut from Run 021 was removed
- the 2M diagnostic completed and cleared the minimum SUMO gate

Inside SUMO, the run remained good enough to keep learning from:
- final eval reward `= 440.59`
- collision rate `= 0.0%`
- behavioral success `= 98.55%`
- weighted V2V reaction `≈ 64.4%` from `source_reaction_summary`
- eval matrix covered peer counts `n=1..5`

However, the real-data replay gate still failed decisively:

| Recording | Sensitivity | False Positive Rate | Target | Verdict |
|-----------|-------------|---------------------|--------|---------|
| Recording #2 | `12.0%` (`3/25`) | `2.55%` | `>60% / <15%` | FAIL |
| Extra Driving | `26.1%` (`6/23`) | `4.58%` | `>75% / <20%` | FAIL |

This means Run 022 fixed the Run 021 false-positive catastrophe, but it did **not** recover real-world sensitivity. The policy fell back toward the same low-FP / low-sensitivity regime already seen in Run 020.

---

## 2. What Run 022 Proved

### 2.1 Ego heading really was the dominant false-positive leak

Run 021 failed because heading acted as a route-position proxy. Removing it sharply reduced false positives:
- Recording #2 FP: `19.16%` → `2.55%`
- Extra Driving FP: `93.83%` → `4.58%`

That is too large to be noise. Heading removal should be treated as a permanent correction, not a temporary experiment.

### 2.2 SUMO trainability survived the observation change

Run 022 did **not** collapse when heading was removed:
- the 2M run completed
- the final eval matrix still produced `0` collisions
- the policy still passed the plan's `>50%` 2M V2V bar
- the eval matrix still covered `n=1..5`

So the remaining problem is no longer "can the heading-free model train at all?" It can. The stronger problem is that it becomes too conservative once the heading shortcut is removed.

### 2.3 The remaining transfer gap is primarily sensitivity

After heading removal, the replay failure mode is now clear:
- the model is mostly calm during real driving
- false positives are low
- but it misses most real braking events even when peers are present

This is the same failure family as Run 020, not Run 021.

---

## 3. What Failed

### 3.1 The mandatory replay gate failed on both recordings

Run 022's plan required replay validation to pass before any 10M promotion.
It did not.

Key replay outcomes:
- Recording #2: `12.0%` sensitivity, `2.55%` FP
- Extra Driving: `26.1%` sensitivity, `4.58%` FP

This is not a borderline miss. Sensitivity is still far below the required thresholds on both datasets.

### 3.2 Heading removal alone did not recover causal reactivity

The post-Run-021 theory was:
- remove the spurious heading shortcut
- keep the useful hazard-decel randomization and resolved-hazard changes
- replay sensitivity should stay high while FP collapses

What actually happened:
- FP collapsed as expected
- SUMO reaction dropped to a middling `~64.4%`
- sensitivity collapsed with it

That means the strong replay sensitivity seen in Run 021 was not being carried by causal V2V features alone. Once heading was removed, the remaining causal signal was still not sufficient for good real-data transfer.

### 3.3 Extra-driving self-RX was not the reason for failure

The extra-driving RX CSV contains `81` self-messages from `V001`, and the replay validator currently does not exclude self-RX.

That was checked explicitly:
- original Extra replay: `26.1%` sensitivity / `4.58%` FP
- self-RX filtered replay: `21.7%` sensitivity / `3.83%` FP

So the no-go decision does **not** depend on that artifact.

---

## 4. Artifacts

### Training artifact

- `roadsense-v2v/ml/models/runs/cloud_prod_022/metrics.json`

### Replay artifacts generated in this analysis

- `roadsense-v2v/ml/data/sim_to_real_validation_run022_recheck_20260316/validation_report_recording_02_decay.json`
- `roadsense-v2v/ml/data/sim_to_real_validation_run022_recheck_20260316/validation_report_extra_driving_decay.json`
- `roadsense-v2v/ml/data/sim_to_real_validation_run022_recheck_20260316/validation_report_extra_driving_decay_no_self.json`

---

## 5. Final Disposition

- **Do not promote Run 022 to 10M.**
- **Do not use Run 022 for deployment or quantization.**
- **Keep ego heading removed in future runs.**
- **Use Run 022 as the clearest proof so far that specificity is fixed, but sensitivity recovery is still unsolved.**

The next run should target **replay sensitivity recovery without reintroducing route-coded features**.
