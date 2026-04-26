<!-- Template Version: 2.0 | F369 Wave 8 (composite) | Composes: MLOPS_GROUND_TRUTH_PLUS + MLOPS_GROUND_TRUTH + MLOPS_SAGEMAKER_TRAINING -->

# Template 33 — Managed Labeling Pipeline (Ground Truth Plus → automated training pipeline)

## Purpose

Stand up a labeling-to-training pipeline using **Ground Truth Plus (managed labeling service)** for engagements that need expert labelers (medical, legal, geospatial) or large-volume labeling without operating an in-house workforce. Auto-triggers training pipeline when batch labels arrive.

Generates production-deployable CDK + batch-complete trigger Lambda + training pipeline + auditing.

---

## Role Definition

You are an expert AWS ML engineer + project manager with deep expertise in:
- Ground Truth Plus engagement model (4-6 week onboarding, AWS PM, vendor workforce)
- Self-managed Ground Truth alternative (Mechanical Turk, private workforce)
- Output manifest format (JSONL with source-ref, label, metadata, worker-id)
- IAM grants for `ground-truth-labeling.sagemaker.amazonaws.com` service principal
- KMS cross-service encryption
- Trigger-based training kickoff (S3 → EventBridge → Lambda → Pipeline)

Generate complete production-deployable code, no TODOs.

---

## Context and Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED — dev | stage | prod]

# --- LABELING SERVICE ---
LABELING_MODE:          [REQUIRED — gt_plus | self_managed_gt]
TASK_TYPE:              [REQUIRED — image_classification | image_bbox | image_segmentation | text_classification | ner | medical_imaging | legal_review]
EXPECTED_VOLUME:        [REQUIRED — total objects estimated]
WORKFORCE_TYPE:         [if gt_plus — vendor | expert (radiologist, lawyer)]
                        [if self_managed_gt — public (mturk) | private (cognito) ]

# --- INPUT ---
INPUT_BUCKET_NAME:      [REQUIRED — where you upload raw data]
INPUT_PREFIX:           [REQUIRED — e.g. medical-images-batch-1/]

# --- OUTPUT ---
OUTPUT_BUCKET_NAME:     [REQUIRED — where labeled data lands]

# --- DOWNSTREAM TRAINING ---
TRAINING_PIPELINE_NAME: [REQUIRED — name of SageMaker Pipeline triggered by labels]
MIN_BATCH_SIZE:         [OPTIONAL — 5000 default; trigger only fires if N+ labels]

# --- COMPLIANCE ---
KMS_KEY_ARN:            [REQUIRED]
RETAIN_LABELED_DATA:    [OPTIONAL — true; default true (forever)]
PII_HANDLING:           [REQUIRED if medical/legal — must be redacted before labeling]
```

---

## Partial Library

| Partial | Why |
|---|---|
| `MLOPS_GROUND_TRUTH_PLUS` | Managed labeling service supporting infra (input/output buckets + IAM grants) |
| `MLOPS_GROUND_TRUTH` | Self-managed alternative (private workforce + Mechanical Turk) |
| `MLOPS_SAGEMAKER_TRAINING` | Training pipeline that consumes labeled output |
| `MLOPS_LINEAGE_TRACKING` | Track which labels trained which model |
| `LAYER_SECURITY` | KMS + IAM patterns for cross-service grants |

---

## Architecture (GT Plus path)

```
   Customer
        │
        │  1. Engage AWS account team (4-6 week onboarding for GT Plus)
        │
        ▼
   AWS sets up GT Plus project in console
        │
        │  2. Customer uploads raw data
        ▼
   ┌──────────────────────────────────────────────────────────────────┐
   │  Input bucket: s3://qra-gtplus-input/                            │
   │     - KMS-encrypted (AWS PM service role can decrypt)              │
   │     - Block Public + enforce SSL                                   │
   └────────────────┬─────────────────────────────────────────────────┘
                    │
                    │  3. AWS workforce labels objects + AWS PM does QC
                    │
                    ▼
   ┌──────────────────────────────────────────────────────────────────┐
   │  Output bucket: s3://qra-gtplus-output/                          │
   │     - manifest.jsonl per batch                                    │
   │     - QC report                                                    │
   │     - Audit trail (worker-ids per label)                           │
   └────────────────┬─────────────────────────────────────────────────┘
                    │
                    │  4. S3 ObjectCreated event (manifest.jsonl) →
                    │
                    ▼
   ┌──────────────────────────────────────────────────────────────────┐
   │  Trigger Lambda                                                   │
   │     - Counts labels in manifest                                    │
   │     - If count >= MIN_BATCH_SIZE: kick off SageMaker Pipeline      │
   │     - Else: log + wait for next batch                              │
   └────────────────┬─────────────────────────────────────────────────┘
                    │
                    ▼
   ┌──────────────────────────────────────────────────────────────────┐
   │  SageMaker Training Pipeline                                     │
   │     - data prep step                                                │
   │     - training step (uses labels as ground truth)                   │
   │     - eval step                                                     │
   │     - register Model Package (with batch-id metadata for lineage)   │
   └──────────────────────────────────────────────────────────────────┘
```

### Self-managed GT path (if `LABELING_MODE=self_managed_gt`)

For sub-10K-object projects or when you have your own workforce:

```
   Customer uploads data → Labeling Job (CreateLabelingJob API) →
   Mechanical Turk OR private workforce labels →
   Output to same S3 bucket → same downstream trigger
```

Cheaper (~$0.012-$0.10/task) but you handle:
- Labeling instructions
- Quality consensus (majority vote, etc.)
- Active learning loop
- PM overhead

See `MLOPS_GROUND_TRUTH` partial for self-managed CDK.

---

## Day-by-day execution (5-day POC, plus 4-week GT Plus onboarding parallel)

### Pre-week 0 to week 4 — GT Plus AWS engagement (parallel to POC)
- Engage AWS account team
- Sign GT Plus SOW
- Define labeling guidelines + sample labels
- Pilot batch (100-1000 objects) before going live

### Day 1 — Input bucket + IAM grants
- Input bucket (KMS, Block Public, SSL-enforce)
- Bucket policy grants `ground-truth-labeling.sagemaker.amazonaws.com` Read+List
- KMS grant decrypt to GT Plus service principal
- **Deliverable:** GT Plus team can submit test job successfully

### Day 2 — Output bucket + downstream trigger
- Output bucket (KMS, versioned, lifecycle to Glacier after 30 days)
- Bucket policy grants GT Plus service principal Write+ACL
- KMS grant encrypt
- Trigger Lambda: counts labels in manifest, conditionally starts Pipeline
- S3 ObjectCreated event subscription (suffix: `manifest.jsonl`)
- **Deliverable:** sample manifest upload triggers Pipeline (mock, low MIN_BATCH_SIZE)

### Day 3 — Training pipeline integration
- SageMaker Pipeline (data prep → training → eval → register)
- Reads manifest.jsonl from output bucket
- Hyperparameters: batch_id, manifest_uri
- Lineage capture: links Model Package to batch-id (audit trail of which labels trained which model)
- **Deliverable:** end-to-end test — manifest → trigger → pipeline → registered Model Package

### Day 4 — Quality + audit
- QC report parsing (worker-id, time-per-task, agreement rate)
- CloudWatch metric: per-batch label count + quality score
- Audit log: every label preserved with worker-id (regulator requirement)
- **Deliverable:** audit query "which labels did worker X produce?" returns < 60s

### Day 5 — UAT + cost projection
- Run pilot batch end-to-end (100 sample objects through GT Plus or self-managed)
- Cost projection per object type + total program cost
- Onboarding doc for ML team: how to submit a new batch
- **Deliverable:** cost projection signed-off; onboarding doc reviewed

---

## Validation criteria

- [ ] GT Plus / self-managed labeling submission works
- [ ] Output manifest matches expected schema
- [ ] Trigger Lambda fires on `manifest.jsonl` arrival
- [ ] Training Pipeline launches when batch ≥ MIN_BATCH_SIZE
- [ ] Lineage links Model Package back to specific batch + worker-ids
- [ ] Quality metrics published to CloudWatch
- [ ] Cost per object matches AWS quote ± 10%

---

## Output artifacts

1. CDK app (LabelingStack — input + output buckets + trigger Lambda)
2. Training Pipeline Python DSL + JSON
3. Trigger Lambda code (counts labels, kicks off pipeline)
4. QC report parser Lambda
5. Per-batch lineage capture script
6. Cost projection spreadsheet
7. Onboarding runbook (how to submit a new batch)
8. AWS account team contact info + GT Plus SOW template

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 2.0 | 2026-04-26 | Initial composite. Composes GT Plus + GT + Training Pipeline. Wave 8. |
