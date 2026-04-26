<!-- Template Version: 2.0 | F369 Wave 8 (composite) | Composes: MLOPS_TRAINIUM_INFERENTIA_NEURON + MLOPS_SMART_SIFTING + MLOPS_INFERENCE_PIPELINE_RECOMMENDER -->

# Template 31 — AWS Silicon Cost Optimization (Trainium2 training · Inferentia2 inference · Smart Sifting · auto-right-sizing)

## Purpose

Migrate ML training + inference from GPU to AWS silicon (Trainium2, Inferentia2) for 40-75% cost savings. Combines Neuron SDK migration + Smart Sifting (drops 30-50% training data) + Inference Recommender (auto-picks cheapest serving instance).

Generates production-deployable CDK + Neuron compilation scripts + cost comparison dashboards.

---

## Role Definition

You are an expert AWS silicon practitioner with deep expertise in:
- AWS Trainium2 + Inferentia2 hardware
- Neuron SDK 2.20+ (compiler, runtime, neuronx-distributed)
- Optimum Neuron (HF integration: NeuronModelForCausalLM, NeuronTrainer)
- Smart Sifting for training cost reduction
- Pre-compilation workflow for Inferentia2 endpoints
- Inference Recommender for cost-vs-latency trade-offs
- GPU-vs-Trainium cost models

Generate complete production-deployable code, no TODOs.

---

## Context and Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED — dev | stage | prod]

# --- WORKLOAD ---
WORKLOAD_TYPE:          [REQUIRED — fine_tune | inference | both]
MODEL_FAMILY:           [REQUIRED — llama3 | mistral | qwen | gemma | custom]
MODEL_SIZE:             [REQUIRED — 7b | 13b | 70b | 405b]

# --- COST GOALS ---
TARGET_TRAINING_COST_REDUCTION: [OPTIONAL — 0.6 (60%); achievable on Trainium2 + Smart Sifting]
TARGET_INFERENCE_COST_REDUCTION: [OPTIONAL — 0.4 (40%); achievable on Inferentia2]
SLA_P99_LATENCY_MS:     [REQUIRED for inference]
TARGET_THROUGHPUT_RPS:  [REQUIRED for inference]

# --- TRAINING ---
TRAINING_INSTANCE:      [OPTIONAL — ml.trn2.48xlarge default]
TRAINING_NODE_COUNT:    [OPTIONAL — 4 default]
PEFT_METHOD:            [OPTIONAL — lora]

# --- INFERENCE ---
INFERENCE_INSTANCE:     [OPTIONAL — ml.inf2.48xlarge default]
PRECOMPILE_BATCH_SIZE:  [OPTIONAL — 4]
PRECOMPILE_SEQ_LEN:     [OPTIONAL — 2048]
NUM_NEURON_CORES:       [OPTIONAL — 12 (full inf2.48xlarge)]

# --- COST CONTROLS ---
NEURON_COMPILE_CACHE:   [REQUIRED — S3 URI to share compiled artifacts across runs]
USE_SMART_SIFTING:      [OPTIONAL — true; default true]
SMART_SIFTING_BETA:     [OPTIONAL — 0.6]

# --- COMPLIANCE ---
KMS_KEY_ARN:            [REQUIRED]
```

---

## Partial Library

| Partial | Why |
|---|---|
| `MLOPS_TRAINIUM_INFERENTIA_NEURON` | Trainium2 + Inferentia2 + Neuron SDK + Optimum Neuron |
| `MLOPS_SMART_SIFTING` | DataLoader wrapper for 30-50% training cost savings |
| `MLOPS_INFERENCE_PIPELINE_RECOMMENDER` | Inference Recommender for instance right-sizing |
| `MLOPS_HYPERPOD_FM_TRAINING` | HyperPod Trainium2 cluster variant (for large jobs) |
| `MLOPS_LLM_FINETUNING_PROD` | Production fine-tune pipeline foundation |

---

## Architecture

```
   TRAINING (Trainium2)                              INFERENCE (Inferentia2)
   ─────────────────────                             ───────────────────────
   ┌─────────────────────────────────────┐           ┌──────────────────────────────┐
   │  HyperPod or Training Job            │           │  Endpoint: llama3-70b-inf2-prod│
   │     - 4× ml.trn2.48xlarge ($31/hr)   │           │     - 1× ml.inf2.48xlarge       │
   │     - $123/hr equivalent on P5e (75% │           │     - $13/hr (vs $16 on g5)     │
   │       savings)                        │           │     - 12× Inferentia2 chips    │
   │                                       │           │     - Pre-compiled artifact     │
   │  + Smart Sifting (β=0.6)              │           │       in S3 (loaded at startup) │
   │     - Drops 40% of training samples   │           └──────────┬───────────────────┘
   │     - Same accuracy ±0.5pp              │                      │
   │     - Saves 40% of GPU/TRN-hr cost      │                      │
   │                                          │                      ▼
   │  Total training cost: 75% × 60% = 0.45  │           Inference Recommender:
   │  (15% of GPU baseline)                  │           - Tested g5/g6/p4d/inf2
   └────────────────┬────────────────────┘                - Inf2 cheapest meeting SLA
                    │                                     - Auto-picked at deploy time
                    ▼
        Final adapter (LoRA, ~250 MB)                    Cost per 1M tokens:
                    │                                       - g5.48xl:    $75
                    ▼                                        - p4d.24xl:   $82
        Pre-compile for Inferentia2 ─────────► S3 artifact   - inf2.48xl:  $45 ★
                                                             - p5.48xl:   $136
   ┌──────────────────────────────────────────────────────────────────────────┐
   │  Dashboard: cost comparison (GPU baseline vs AWS-silicon)               │
   │     - Training: $X/run baseline → $0.15X/run on Trn2 + Smart Sifting    │
   │     - Inference: $Y/1M tokens baseline → $0.6Y/1M tokens on Inf2        │
   │     - Annual savings: $$$ documented                                      │
   └──────────────────────────────────────────────────────────────────────────┘
```

---

## Day-by-day execution (7-day POC)

### Day 1 — Compile cache + IAM
- S3 bucket for Neuron compile cache (KMS-encrypted, 180-day lifecycle)
- IAM execution role (read+write compile cache, S3 + KMS perms)
- Permission boundary
- **Deliverable:** infra synth-clean

### Day 2-3 — Training migration (Trainium2)
- Convert HF training script to NeuronTrainer + NeuronModelForCausalLM
- Smart Sifting wrapper integration
- Estimator config: `ml.trn2.48xlarge`, Neuron-enabled DLC image, env vars (NEURON_RT_NUM_CORES, FI_PROVIDER=efa, NEURON_COMPILE_CACHE_URL)
- Distribution config: PyTorch DDP w/ `xla` backend
- **Deliverable:** sample 1-hour PEFT-LoRA run completes; second run skips compilation (cache hit)

### Day 4 — Inference compilation
- `compile_for_inf2.py` script using `optimum-neuron`:
  - Loads HF model from S3 (or HF Hub)
  - `NeuronModelForCausalLM.from_pretrained(..., export=True, batch_size=N, sequence_length=L, num_cores=12)`
  - Saves compiled artifact (~30-50 GB for 70B base + adapter)
  - Tar + upload to S3
- **Deliverable:** pre-compiled artifact ready for endpoint

### Day 5 — Inferentia2 endpoint
- Model + EndpointConfig + Endpoint (with `ml.inf2.48xlarge`)
- Neuron-enabled HF DLC image
- Container env: `HF_NUM_CORES=12`, `NEURON_RT_NUM_CORES=12`, `HF_MODEL_ID=/opt/ml/model`
- Health-check timeout 600s (Neuron warm-up takes ~10 min)
- **Deliverable:** endpoint InService; sample invoke returns < 500ms

### Day 6 — Inference Recommender
- Default Job triggered on the registered Model Package
- Tests: ml.g5.xlarge, ml.g5.12xlarge, ml.inf2.xlarge, ml.inf2.48xlarge, ml.p4d.24xlarge
- SLA: P99 latency < {SLA_P99_LATENCY_MS}, target throughput {TARGET_THROUGHPUT_RPS}
- Output: cost-per-1000-inferences ranked
- **Deliverable:** report shows Inferentia2 cheapest meeting SLA (or alternate selection)

### Day 7 — Cost comparison dashboard + UAT
- CloudWatch dashboard:
  - Training: Cost-per-run (Trn2 + Smart Sifting) vs GPU baseline (calculated)
  - Inference: Cost-per-1M-tokens (Inf2) vs g5/p4d (Recommender output)
  - Annual savings projection
- Quality regression test: run Trn2-trained model on holdout, compare to GPU baseline
- **Deliverable:** dashboard live; quality delta < 1pp; cost savings documented

---

## Validation criteria

- [ ] **Compile cache hits** on second training run (compilation skipped, ~30 min saved)
- [ ] **Training cost** 60-75% lower than GPU equivalent (Trn2 + Smart Sifting)
- [ ] **Quality delta** < 1pp on holdout vs GPU baseline
- [ ] **Inference cost** 40% lower than g5 equivalent
- [ ] **Inferentia2 throughput** ≥ 80% of g5 equivalent at same precision (BF16)
- [ ] **Recommender** auto-picks Inf2 if SLA met (or g5 fallback if not)
- [ ] **Pre-compile pipeline** automated — no manual `optimum-neuron compile` step

---

## Common gotchas (claude must address proactively)

- **`NEURON_COMPILE_CACHE_URL` required** — without S3 cache, every training run recompiles (30-60 min wasted)
- **BF16 only** — FP16 has stability issues on Trainium; FP32 wastes the silicon
- **Pre-compile MUST match endpoint config** — batch size + sequence length match exactly, otherwise re-compile at first request
- **Container startup health check 600s** — Neuron warm-up takes 10 min; default 60s health check times out
- **Architecture validation** — verify model architecture is on Neuron Compatibility Matrix before committing

---

## Output artifacts

1. CDK app (TrainingStack + InferenceStack)
2. Training script (`train_lora_neuron.py` with Smart Sifting + xla DDP)
3. Inference compile script (`compile_for_inf2.py`)
4. Inference endpoint deployer Lambda
5. Inference Recommender trigger + parser Lambda
6. Cost comparison dashboard JSON
7. Quality regression test suite
8. Migration runbook (GPU → Trainium → Inferentia)
9. Cost projection spreadsheet (per model size, per workload)

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 2.0 | 2026-04-26 | Initial composite. Composes Trainium/Inferentia/Neuron + Smart Sifting + Inference Recommender. Wave 8. |
