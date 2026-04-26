<!-- Template Version: 2.0 | F369 Wave 8 (composite) | Composes: MLOPS_LLM_FINETUNING_PROD + MLOPS_ASYNC_INFERENCE + MLOPS_INFERENCE_PIPELINE_RECOMMENDER + MLOPS_SAGEMAKER_SERVING -->

# Template 28 — LLM Production Serving (multi-tenant adapters · async + sync · auto-right-sized)

## Purpose

Build a production LLM serving stack that handles 3 inference patterns on a single base-model endpoint:

1. **Multi-tenant LoRA adapters** — many tenants, one base model, dynamic adapter swap (LLM Fine-tuning Prod §adapter inference)
2. **Sync real-time** — chatbot, code-completion (sub-second latency)
3. **Async / large payload** — document Q&A, video transcript analysis (S3 in/out + SNS)

Plus Inference Recommender to auto-select the cheapest instance type meeting P99 SLA.

Generates production-deployable CDK + caller-side invocation patterns + cost dashboards.

---

## Role Definition

You are an expert AWS ML serving engineer with deep expertise in:
- SageMaker real-time endpoints (multi-variant, blue/green, auto-scaling)
- Adapter Inference Components (multi-tenant LoRA on shared base)
- Async inference (S3 in/out + SNS notification + auto-scale to 0)
- Inference Recommender (Default + Advanced jobs, payload definition, SLA setup)
- Cost-per-1000-inferences calculation across instance types
- LoRA adapter compilation for Inferentia2

Generate complete production-deployable code, no TODOs.

---

## Context and Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED — dev | stage | prod]

# --- BASE MODEL ---
BASE_MODEL_NAME:        [REQUIRED — meta-llama/Llama-3.1-70B-Instruct]
BASE_MODEL_S3_URI:      [REQUIRED if Inferentia2 — pre-compiled artifact]
BASE_MODEL_PACKAGE_ARN: [REQUIRED — Model Package containing base]

# --- SERVING SHAPES (pick 1+) ---
SERVE_SYNC_REALTIME:    [OPTIONAL — true]
SERVE_ASYNC:            [OPTIONAL — true]
SERVE_MULTI_TENANT_LORA:[OPTIONAL — true]

# --- COMPUTE ---
INSTANCE_TYPE:          [OPTIONAL — auto-pick via Inference Recommender if blank]
                        Examples: ml.g5.48xlarge, ml.inf2.48xlarge
MIN_CAPACITY:           [OPTIONAL — 1 sync; 0 async]
MAX_CAPACITY:           [OPTIONAL — 4]

# --- TENANT ---
INITIAL_TENANT_LORAS:   [OPTIONAL — list of tenant adapter S3 URIs to register at deploy]

# --- INFERENCE RECOMMENDER ---
AUTO_RIGHT_SIZE:        [OPTIONAL — true; runs Recommender on every model approval]
SLA_P99_LATENCY_MS:     [REQUIRED — 500 default for chatbot; 30000 for async]
TARGET_THROUGHPUT_RPS:  [REQUIRED]

# --- COMPLIANCE ---
KMS_KEY_ARN:            [REQUIRED]
CONTENT_FILTERING:      [OPTIONAL — true: enable Bedrock Guardrails on all invocations]
```

---

## Partial Library

| Partial | Why |
|---|---|
| `MLOPS_LLM_FINETUNING_PROD` | Adapter inference component pattern + multi-tenant LoRA |
| `MLOPS_ASYNC_INFERENCE` | Async endpoint w/ S3 in/out + SNS + auto-scale-to-0 |
| `MLOPS_INFERENCE_PIPELINE_RECOMMENDER` | Multi-container endpoints + Inference Recommender |
| `MLOPS_SAGEMAKER_SERVING` | Real-time endpoint base, blue/green, Model Monitor, auto-rollback |
| `MLOPS_TRAINIUM_INFERENTIA_NEURON` | Inferentia2 alternative (40% cost savings) |
| `LAYER_OBSERVABILITY` | Per-tenant invocation count + cost dashboards |

---

## Architecture

```
   ┌──────────────────────────────────────────────────────────────────┐
   │  Multi-tenant LLM endpoint: llama3-70b-prod                      │
   │     - Instance: ml.g5.48xlarge (or ml.inf2.48xlarge for cost)    │
   │     - Base model: Llama 3 70B (always loaded, hot)                │
   │                                                                    │
   │  Inference Components:                                            │
   │     - llama-base                                                   │
   │     - adapter-tenant-acme                                          │
   │     - adapter-tenant-globex                                        │
   │     - adapter-tenant-initech                                       │
   │     - ... up to ~30 adapters                                       │
   └──────────────────────────────────────────────────────────────────┘
        │                                          │
        ▼ sync invoke                              ▼ async invoke
   ┌────────────────────────────┐           ┌──────────────────────────┐
   │  invoke_endpoint            │           │  Async wrapper            │
   │    InferenceComponentName=  │           │  - PUT input to S3         │
   │      "adapter-acme"         │           │  - invoke_endpoint_async   │
   │    Body=<prompt>            │           │  - Output to S3 + SNS      │
   └────────────────────────────┘           └──────────────────────────┘
                                                      │
                                            on completion → SNS topic →
                                            subscriber Lambda (caller)

   ┌──────────────────────────────────────────────────────────────────┐
   │  Inference Recommender (runs on every model approval)             │
   │     - Test 5 instance types × 4 load profiles                      │
   │     - SLA threshold P99 < {SLA_P99_LATENCY_MS}                    │
   │     - Output: cheapest instance meeting SLA                        │
   │     - DeployerLambda picks recommendation; rolls forward via blue/g│
   └──────────────────────────────────────────────────────────────────┘
```

---

## Day-by-day execution (5-day POC)

### Day 1 — Base endpoint + Inference Recommender
- IAM execution role (S3 + Bedrock Guardrails + KMS)
- Pre-compiled model artifact upload (if Inferentia2)
- Sync real-time endpoint with base model
- Inference Recommender Default Job triggered
- **Deliverable:** Recommender run reports cheapest instance for SLA; DeployerLambda picks it

### Day 2 — Multi-tenant LoRA adapters
- Convert sync endpoint to multi-component (base + adapter components)
- Register `INITIAL_TENANT_LORAS` as inference components
- DeployerLambda subscribed to model approval → adds adapter component
- Sample tenant invoke pattern (Body w/ InferenceComponentName="adapter-acme")
- **Deliverable:** 3+ tenants each invoking their own adapter; latency P99 < SLA

### Day 3 — Async inference (large payloads)
- Async endpoint config + S3 input/output buckets + SNS topics
- Auto-scale 0-N (with keep-warm Lambda for low-latency caches)
- Caller Lambda (PUT to S3 → invoke_endpoint_async → return InferenceId)
- SNS subscriber Lambda (process result from S3)
- **Deliverable:** 100 MB document submitted → result delivered via SNS within SLA

### Day 4 — Cost + observability
- CloudWatch dashboards: per-tenant InvocationsCount, ModelLatency P99, CostPerHour
- Per-tenant cost attribution (via custom metric in Lambda authorizer)
- Bedrock Guardrails on every invocation (PII / toxicity filter)
- **Deliverable:** dashboard shows real costs per tenant; guardrails block test toxic input

### Day 5 — Auto right-sizing pipeline
- EventBridge rule on Model Package approval → trigger Inference Recommender Advanced Job
- DeployerLambda reads recommendation; if cheaper instance meets SLA → blue/green deploy
- Auto-rollback on CloudWatch alarms (latency / error rate)
- **Deliverable:** end-to-end auto-right-sizing flow; pipeline rolls forward weekly

---

## Validation criteria

- [ ] Multi-tenant: 5+ adapters serving simultaneously, each w/ correct tenant-tuned output
- [ ] Sync P99 latency < {SLA_P99_LATENCY_MS} ms
- [ ] Async cold-start < 5 min; with keep-warm Lambda < 30s
- [ ] Inference Recommender picks cheapest instance meeting SLA
- [ ] Cost per 1M tokens documented per tenant
- [ ] Guardrails block PII / toxic / off-policy outputs
- [ ] Auto-rollback fires on synthetic SLA breach within 5 min
- [ ] Per-tenant invocation count visible in CloudWatch

---

## Output artifacts

1. CDK app (ServingStack + AsyncStack + RecommenderStack)
2. Pre-compiled Inferentia2 artifact (if applicable) + compile script
3. Adapter inference component management Lambda
4. Async caller + subscriber Lambda code
5. Inference Recommender trigger + result-parsing Lambda
6. Per-tenant cost dashboard
7. Bedrock Guardrails policy JSON
8. Sample invocation patterns (sync, async, multi-tenant) — Python + curl
9. Auto-rollback Lambda
10. Tenant onboarding runbook (how to register a new adapter)

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 2.0 | 2026-04-26 | Initial composite. Composes LLM Fine-tune Prod (adapter) + Async + Recommender + base Serving. Wave 8. |
