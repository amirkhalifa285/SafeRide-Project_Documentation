# Quantization Dual-Track Plan (PTQ + QAT)

**Status:** Planned (Future Phase)
**Last Updated:** January 22, 2026
**Owner:** Amir + ML Agent

---

## Purpose

When we reach the quantization stage, we will **run Post-Training Quantization (PTQ)** and **Quantization-Aware Training (QAT)** in parallel, then pick the best model based on the same evaluation criteria.

This plan aligns with the professor’s guidance: try both approaches at the same time and select the best-performing INT8 model.

---

## Scope

This plan applies to the **Deep Sets split model**:
- `peer_encoder.tflite` (called n times)
- `policy_head.tflite` (called once after pooling)

---

## Preconditions

- [ ] Float training complete and best PPO checkpoint selected
- [ ] Float inference evaluation metrics recorded (baseline)
- [ ] Model exported to TF SavedModel (or equivalent) for conversion
- [ ] Deep Sets split confirmed (peer encoder + policy head)

---

## Step 1: Build a Shared Quantization Dataset (Required for Both Tracks)

- [ ] Log representative **peer inputs** (6-D) for the encoder
- [ ] Log **pooled latent + ego features** (32 + 4) for the policy head
- [ ] Log **float outputs** for distillation (encoder embeddings + policy logits)
- [ ] Freeze a small, fixed eval set (same seeds/scenarios for both tracks)

**Artifacts:**
- `quant_dataset_peer_inputs.npy`
- `quant_dataset_policy_inputs.npy`
- `quant_dataset_peer_outputs.npy`
- `quant_dataset_policy_outputs.npy`
- `quant_eval_scenarios.json`

---

## Step 2: PTQ Track (Fast Baseline)

- [ ] Use representative dataset for calibration
- [ ] Export INT8 `peer_encoder.tflite`
- [ ] Export INT8 `policy_head.tflite`
- [ ] Run identical eval suite as float baseline

**Artifacts:**
- `peer_encoder_ptq_int8.tflite`
- `policy_head_ptq_int8.tflite`
- `eval_ptq.json`

---

## Step 3: QAT Track (Higher Accuracy Candidate)

- [ ] Rebuild encoder/head in TF/Keras and load float weights
- [ ] Apply QAT wrappers (`tfmot.quantization.keras`)
- [ ] Train on quant dataset with **distillation loss**
- [ ] Export INT8 `peer_encoder.tflite` + `policy_head.tflite`
- [ ] Run identical eval suite as float + PTQ

**Artifacts:**
- `peer_encoder_qat_int8.tflite`
- `policy_head_qat_int8.tflite`
- `eval_qat.json`

---

## Step 4: Head-to-Head Evaluation (Pick Winner)

**Metrics (must use same eval scenarios):**
- [ ] Success rate / collision rate
- [ ] Reward vs float baseline
- [ ] Accuracy drop ≤ 10% vs float
- [ ] Model size ≤ 100KB (each sub-model)
- [ ] ESP32 latency ≤ 50ms (end-to-end)

**Decision Rule:**
Pick the model with the **best eval success rate** that meets size + latency constraints.

---

## Risks / Notes

- QAT is more complex to implement but may reduce accuracy drop.
- PTQ is faster and may already meet the accuracy budget.
- Keep **identical eval seeds** and **same data** for fair comparison.
- Store all conversion logs for reproducibility.

---

## Completion Criteria

- [ ] PTQ and QAT INT8 models exported
- [ ] Same evaluation suite run on both
- [ ] Winner selected + documented
- [ ] ESP32 latency + memory confirmed

