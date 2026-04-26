<!-- Template Version: 2.0 | F369 Wave 8 (composite) | Composes: MLOPS_HYPERPOD_FM_TRAINING + MLOPS_DISTRIBUTED_TRAINING + MLOPS_SMART_SIFTING + MLOPS_LINEAGE_TRACKING -->

# Template 27 — Foundation Model Training with HyperPod (Llama 3 70B/405B · Mistral · Qwen)

## Purpose

Build a resilient foundation-model training cluster (Slurm or EKS orchestrated) capable of training/fine-tuning models from 30B to 671B parameters. Combines HyperPod resilience (auto-recovery, deep health checks) with distributed training (SMDDP / FSDP / DeepSpeed) and cost levers (Smart Sifting drops 30-50% data) and full lineage tracking for governance.

Generates production-deployable CDK + recipe launchers + cost-monitoring + DR runbook.

---

## Role Definition

You are an expert AWS ML platform engineer + foundation model practitioner with deep expertise in:
- HyperPod orchestration (Slurm + EKS), resilience features, FSx Lustre attached storage
- Distributed training (SMDDP all-reduce, FSDP sharding, DeepSpeed ZeRO-3, EFA networking)
- PEFT-LoRA on 70B+ models with NeMo recipes
- Smart Sifting (30-50% training cost reduction)
- ML Lineage (artifact graph, Model Cards)
- Trainium2 alternative for cost-efficient FM training
- MLflow Apps for experiment tracking

Generate complete production-deployable code, no TODOs.

---

## Context and Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED — dev | stage | prod]

# --- MODEL ---
BASE_MODEL_ID:          [REQUIRED — e.g. meta-llama/Llama-3.1-70B-Instruct]
TRAINING_TYPE:          [REQUIRED — pretrain | continue_pretrain | sft | dpo | peft_lora]
PEFT_METHOD:            [if peft_lora — lora | qlora | dora]
LORA_RANK:              [OPTIONAL — 16]
LORA_ALPHA:             [OPTIONAL — 32]

# --- ORCHESTRATION ---
ORCHESTRATOR:           [REQUIRED — slurm | eks]

# --- COMPUTE ---
CLUSTER_HEAD_INSTANCE:  [OPTIONAL — ml.m5.4xlarge]
CLUSTER_COMPUTE_INSTANCE: [REQUIRED — ml.p5e.48xlarge | ml.trn2.48xlarge]
CLUSTER_NODE_COUNT:     [REQUIRED — 4-128]

# --- STORAGE ---
FSX_STORAGE_GIB:        [OPTIONAL — 1228800 (1.2 PB)]
FSX_THROUGHPUT_PER_TIB: [OPTIONAL — 1000 MB/s]

# --- COST LEVERS ---
USE_SMART_SIFTING:      [OPTIONAL — true; default true]
SMART_SIFTING_BETA:     [OPTIONAL — 0.6]
USE_TRAINIUM2:          [OPTIONAL — true if engine supports + cost-sensitive]

# --- LINEAGE / GOVERNANCE ---
ENABLE_LINEAGE:         [OPTIONAL — true; default true]
CREATE_MODEL_CARD:      [OPTIONAL — true]
MODEL_PACKAGE_GROUP:    [REQUIRED — registers final model here]

# --- DATA ---
TRAINING_DATA_S3:       [REQUIRED]
EVAL_DATA_S3:           [REQUIRED]
DATASET_FORMAT:         [jsonl | parquet | webdataset]

# --- COMPLIANCE ---
KMS_KEY_ARN:            [REQUIRED]
PERMISSION_BOUNDARY_ARN:[REQUIRED]
```

---

## Partial Library

| Partial | Why |
|---|---|
| `MLOPS_HYPERPOD_FM_TRAINING` | Cluster CDK, Slurm/EKS orchestrator, FSx Lustre, resilience features, recipes |
| `MLOPS_DISTRIBUTED_TRAINING` | SMDDP / FSDP / DeepSpeed config + EFA env vars + HF Estimator integration |
| `MLOPS_SMART_SIFTING` | DataLoader wrapper for 30-50% cost savings |
| `MLOPS_LINEAGE_TRACKING` | Artifact graph + Model Card |
| `MLOPS_TRAINIUM_INFERENTIA_NEURON` | Optional Trainium2 cluster variant |
| `LAYER_NETWORKING` | VPC + EFA + cluster placement group |
| `LAYER_SECURITY` | KMS + permission boundary |

---

## Architecture

```
   ┌────────────────────────────────────────────────────────────────────┐
   │  HyperPod Cluster: fm-training-prod                                │
   │     - Orchestrator: Slurm (or EKS for multi-tenant)                 │
   │     - Head: 1× m5.4xlarge                                            │
   │     - Compute: 32× p5e.48xlarge (or 8× trn2.48xlarge for cost)       │
   │     - EFA: 100 Gbps × node                                           │
   │     - FSx Lustre: 1.2 PB attached at /fsx                            │
   │     - Resilience: deep health checks every 10 min, auto-recovery     │
   └────────────────┬───────────────────────────────────────────────────┘
                    │
                    ▼ srun -N 32 --gpus=256 ./launch.sh
   ┌────────────────────────────────────────────────────────────────────┐
   │  NeMo / FSDP / DeepSpeed training run                              │
   │     - Smart Sifting wraps DataLoader (60% retention → 40% savings)  │
   │     - SMDDP all-reduce over EFA                                      │
   │     - Checkpoints every 1000 steps to /fsx → S3 (Glacier after 30d) │
   │     - Logs to MLflow Tracking Server                                  │
   └────────────────┬───────────────────────────────────────────────────┘
                    │
                    ▼ on completion
   ┌────────────────────────────────────────────────────────────────────┐
   │  Lineage capture (auto from Pipelines OR manual)                   │
   │     - Artifact: dataset (with row_count, version, schema_hash)      │
   │     - Action: training-job (with hyperparams, runtime, instance)    │
   │     - Artifact: model adapter (linked via Produced association)      │
   │     - Model Card: human-readable governance doc                       │
   └────────────────┬───────────────────────────────────────────────────┘
                    │
                    ▼
   Model Package Group: pending manual approval
   Eval pipeline runs on holdout → metrics in Card → human approves
   → EventBridge → DeployerLambda → endpoint (separate template 28)
```

---

## Day-by-day execution (10-day POC)

### Day 1-2 — Foundation
- VPC + isolated subnets + cluster placement group
- 3 KMS CMKs (cluster, FSx, S3 artifacts)
- S3 buckets (artifacts, neuron-cache if Trainium, MLflow backend)
- IAM execution roles (cluster, training jobs)
- **Deliverable:** infra synth-clean, KMS key rotation enabled

### Day 3-4 — HyperPod cluster provisioning
- FSx Lustre 1.2 PB Persistent_2 with LZ4 compression
- HyperPod cluster (Slurm orchestrator) — head + 32 worker nodes
- Bootstrap LCC: mount FSx, install DCGM exporter, configure Slurm GRES
- Deep health checks active
- **Deliverable:** cluster status `Available`, sample srun job runs

### Day 5-6 — Training script + Smart Sifting integration
- `train_lora_neuron.py` (or GPU equivalent) with PEFT + NeMo recipes
- Smart Sifting wrapper (`SiftingDataloader`) with beta=0.6
- DDP / FSDP / DeepSpeed config matching cluster shape
- MLflow logging integrated
- **Deliverable:** sample 1-hour training run completes; metrics in MLflow

### Day 7-8 — Production training run
- Full training with checkpoints every 1000 steps
- Atomic S3 checkpoint sync (write to .tmp/, atomic rename)
- Test resilience: kill a node mid-run; verify auto-recovery + checkpoint resume
- **Deliverable:** survived 1 simulated node failure; resumes within 5 min

### Day 9 — Lineage + Model Card
- Auto-capture lineage from training run
- Generate Model Card with: training data uri, hyperparameters, eval metrics, ethical considerations, intended uses
- Register Model Package (status: PendingManualApproval)
- **Deliverable:** Model Package + Card visible in console

### Day 10 — Cost monitoring + DR runbook
- CloudWatch dashboard: GPU util, training throughput, FSx IO, cluster cost-per-hour
- Cost alarm: cluster running > 8 hr idle → page ops
- DR runbook: switchover procedure, checkpoint replay, cluster rebuild
- **Deliverable:** runbook reviewed; cost guard active

---

## Validation criteria

- [ ] **Resilience:** simulated node failure → auto-recovery + checkpoint resume in < 5 min
- [ ] **EFA active:** verified via env var `FI_PROVIDER=efa` + NCCL_DEBUG output
- [ ] **Smart Sifting savings:** 30-40% reduction in samples processed per epoch with negligible accuracy delta
- [ ] **Lineage complete:** dataset → training → model artifact graph visible via `aws sagemaker list-associations`
- [ ] **Model Card review-ready:** approver can read the card and approve in 5 min
- [ ] **Cost cap enforced:** alarm fires + cluster auto-stop on 8h idle
- [ ] **Eval threshold:** holdout metrics meet pre-set threshold; otherwise pipeline fails

---

## Output artifacts

1. CDK app (3 stacks: ClusterStack + TrainingStack + LineageStack)
2. NeMo recipe launchers (per model size: 7B, 13B, 70B, 405B)
3. Training script (`train_lora.py` with Smart Sifting + DDP)
4. MLflow Tracking Server config
5. Lineage capture utilities + Model Card template
6. CloudWatch dashboard JSON
7. DR runbook (resilience drill, switchover, cost-runaway response)
8. Cost projection per model size

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 2.0 | 2026-04-26 | Initial composite. Composes HyperPod + Distributed + Smart Sifting + Lineage. Wave 8. |
