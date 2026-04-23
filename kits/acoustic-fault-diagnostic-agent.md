<!-- Kit Version: 1.0 | Template library: F369_LLM_TEMPLATES v1.x | Partials: F369_CICD_Template partials v2.0 -->

# Kit — Acoustic Fault Diagnostic Agent (Engineering Expert)

**Client ask:** "Engineers upload engine/machine sounds; system classifies the fault, proposes probable root cause with cited evidence, and recommends next steps. Fill the gap left by AWS Lookout for Equipment retirement (Oct 7, 2026)."
**Engagement:** 2 weeks · 2 developers · regulated-ready (SOC 2 default; FMEA RPN escalation built-in).
**Deliverable:** CDK repo (Python or TypeScript) + React diagnostic dashboard + trained AST/Wav2Vec2 classifier on SageMaker + Strands multi-agent reasoning layer on AgentCore Runtime.

**Rationale:** See companion design doc [`kits/_design/acoustic-fault-diagnostic-agent.md`](_design/acoustic-fault-diagnostic-agent.md) — business case, UX walkthrough, 3 business flows (Toyota manufacturing QA / dealer service / fleet maintenance), AWS Lookout-for-Equipment migration story, Strands + AgentCore + SageMaker feature mapping.

---

## 1. Kit-wide parameters (fill ONCE; carried to every template + Claude call)

```
# --- Identity -----------------------------------------------------------
PROJECT_NAME:               acoustic-diag-{client_slug}          e.g. acoustic-diag-toyota
AWS_REGION:                 us-east-1  (or any region with AgentCore Runtime + S3 Vectors + SageMaker)
AWS_ACCOUNT_ID:             [12-digit]
ENV:                        dev | stage | prod
CLIENT_SLUG:                toyota
TARGET_LANGUAGE:            python | typescript

# --- DOMAIN_PACK — drives classifier fine-tune corpus + FMEA DB + report template
DOMAIN_PACK:                automotive_engine    (DEFAULT — Toyota / Honda / Ford)
                            | industrial_machine  (NBS IIoT legacy — boilers, mining, conveyors)
                            | hvac_boiler
                            | aerospace_engine    (test-cell diagnostics)
                            | custom

# --- CLASSIFIER (architecture + training mode) --------------------------
CLASSIFIER_MODEL:           ast               (DEFAULT — Audio Spectrogram Transformer; SOTA general)
                            | wav2vec2        (speech-adjacent, voice rattle)
                            | hybrid_cnn_lstm (cheap — MFCC+PCP+STE → CNN+LSTM, the paper's 65-dim)
                            | geco            (GeCo Transformer — unsupervised anomaly)

CLASSIFIER_TRAINING_MODE:   fine_tune_from_pretrained  (DEFAULT)
                            | train_from_scratch
                            | lora_adapter             (fastest; +6% SOTA with LoRA per 2024 paper)

# --- INTAKE MODE --------------------------------------------------------
INTAKE_MODE:                web_upload     (DEFAULT — dashboard form)
                            | mobile_app   (tech's phone mic → API GW)
                            | edge_box     (CloudRail / IoT Greengrass → S3)
                            | batch_csv    (fleet manifest of S3 paths)

PROCESSING_LATENCY:         async   (DEFAULT — SageMaker Async Inference)
                            | sync    (< 5s for dealer-counter UX)
                            | batch   (nightly fleet sweep via Batch Transform)

# --- FEATURE EXTRACTION (DSP pipeline — NEW MLOPS_AUDIO_PIPELINE partial)
PREPROCESS_SAMPLE_RATE:     44100
PREPROCESS_WINDOW_SECONDS:  5
PREPROCESS_OVERLAP:         0.5     (50%)
SILENCE_TRIM_TOP_DB:        20      (20 = aggressive, 40 = gentle, 0 = off)

FEATURES:                   [mel_spectrogram, mfcc]           (DEFAULT for CLASSIFIER_MODEL=ast)
                            [mfcc, pcp, short_term_energy]    (for CLASSIFIER_MODEL=hybrid_cnn_lstm)
                            [mel_spectrogram, wavelet_cwt]    (when transient faults dominate)

N_MELS:                     128   (AST expects 128; 64 for smaller models)
N_MFCC:                     40
N_FFT:                      1024
HOP_LENGTH:                 512

AUGMENTATION_ENABLED:       true
AUGMENTATION_METHODS:       [spec_augment, noise_superimpose, time_stretch, pitch_shift]
NOISE_SNR_LEVELS_DB:        [0, 5, 10, 15, 20]   (ESC-50 background — wind / rain / engine / helicopter)

# --- VECTOR STORAGE (NEW PATTERN_AUDIO_SIMILARITY_SEARCH partial) -------
VECTOR_STRATEGY:            per_window           (DEFAULT — finer-grained, catches transients)
                            | aggregated_sample  (cheaper, loses transient detail)
                            | hybrid_both        (best quality, 2× storage cost)

EMBEDDING_DIM:              768    (AST / Wav2Vec2 base penultimate layer)
VECTOR_DISTANCE_METRIC:     cosine
VECTOR_TOP_K:               10
AUDIO_EMBED_BUCKET_NAME:    {project_name}-audio-vectors-{env}

# --- KNOWLEDGE BASE (repair manuals / TSBs / FMEA — reuses DATA_S3_VECTORS) 
KB_REPAIR_MANUALS_BUCKET:   {project_name}-manuals-{env}
EMBEDDING_MODEL_MANUALS:    amazon.titan-embed-text-v2:0
EMBEDDING_DIM_MANUALS:      1024
VECTOR_TOP_K_MANUALS:       5

# --- FMEA / ESCALATION -------------------------------------------------
FMEA_RPN_ESCALATE:          150   (Severity 1-10 × Occurrence 1-10 × Detection 1-10)
FMEA_SOURCE:                client_provided   (DEFAULT — client supplies the spreadsheet)
                            | agent_generated (agent builds it from past failures; v2)

HUMAN_IN_LOOP:              on_high_rpn   (DEFAULT)
                            | always
                            | never

SAMPLES_PER_DIAGNOSIS:      3     (3 × 5s windows — majority vote)

# --- AGENT ARCHITECTURE (same decision as prior kits) ------------------
AGENT_ARCHITECTURE:         strands_agentcore      (DEFAULT — multi-agent with 8h sessions)
                            | strands_lambda       (single-agent, cheaper)
                            | plain_bedrock_lambda (simplest, no Strands)

SUPERVISOR_MODEL:           us.anthropic.claude-sonnet-4-7-20260109-v1:0
SUB_AGENT_MODEL:            us.anthropic.claude-haiku-4-5-20251001-v1:0
REVIEWER_MODEL:             us.anthropic.claude-sonnet-4-7-20260109-v1:0

MEMORY_MODE:                agentcore_memory  (DEFAULT — STM + LTM strategies)
                            | s3_session_manager
                            | none (stateless)

# --- UI / UX -----------------------------------------------------------
STREAM_RESPONSES:           true
SHOW_SPECTROGRAM:           true
SHOW_WAVEFORM:              true
SHOW_SIMILAR_CASES:         true
SHOW_TREND:                 auto_if_fleet_mode  (only when per-machine history > 5 samples)
SUPPORT_FEEDBACK:           true   (accept / reject / escalate buttons → retraining loop)

# --- GOVERNANCE --------------------------------------------------------
COMPLIANCE_STANDARD:        SOC2   (DEFAULT; HIPAA/PCI for regulated)
RETENTION_DAYS:             365    (audio + diagnoses + citations)
PII_REDACTION_FIELDS:       [vehicle_owner_name, license_plate]   (customer data in dealer mode)
GUARDRAIL_BLOCK_UNSAFE_RECS: true  (e.g. "just ignore it", "it's fine to drive")

# --- COST CONTROLS ------------------------------------------------------
MAX_COST_PER_DIAGNOSIS_USD: 0.20
MAX_DIAGNOSES_PER_DAY:      10000
SAGEMAKER_ENDPOINT_TYPE:    async_inference  (cheapest for bursty load)
                            | serverless     (cold start ~5s)
                            | real_time      (24/7 hot, for sync latency)
```

---

## 2. Architecture

```
Audio ingress (per INTAKE_MODE):
  web_upload    → CloudFront → API GW → S3 raw-audio bucket
  mobile_app    → API GW → S3
  edge_box      → IoT Core → S3
  batch_csv     → admin submits manifest → S3 for bulk score
       │
       ▼  S3 → EventBridge → IngestionLambda
       ▼
   MLOPS_AUDIO_PIPELINE (NEW):
      ├── Validate format + duration + sample rate
      ├── librosa: resample + trim silence + segment (5s / 50% overlap)
      ├── Feature extraction per FEATURES param
      ├── Augmentation (if AUGMENTATION_ENABLED=true)
      └── Save to S3 curated + DDB audio_metadata row
       │
       ▼
   SageMaker inference per CLASSIFIER_MODEL + PROCESSING_LATENCY:
      ├── AST / Wav2Vec2 / Hybrid CNN+LSTM / GeCo
      └── Returns {class_probs, anomaly_score, penultimate_embedding}
       │
       ▼
   PATTERN_AUDIO_SIMILARITY_SEARCH (NEW):
      ├── EmbedderLambda extracts 768-dim vector from penultimate layer
      └── Put vectors (per_window / aggregated / hybrid) into S3 Vectors
       │
       ▼
   Agent (Strands supervisor on AgentCore Runtime):
      ├── ClassifierTool      → SageMaker endpoint results
      ├── SimilarCaseTool     → S3 Vectors audio-similarity query
      ├── ManualKnowledgeTool → S3 Vectors RAG on repair manuals (Titan v2)
      ├── DSPAnalystTool      → AgentCore Code Interpreter (librosa/scipy/pywt on-demand)
      ├── FMEAScoreTool       → compute Severity × Occurrence × Detection = RPN
      ├── Synthesizer         → writes diagnosis card
      └── Reviewer            → grounding validator on every TSB citation
       │
       ▼
   Dashboard (React + CloudFront + OAC + API GW WebSocket):
      ├── Upload / record
      ├── Waveform + spectrogram preview with anomaly overlay
      ├── Diagnosis card + RPN + citations
      ├── Similar cases with audio playback
      ├── Trend (fleet mode)
      └── Accept / Reject / Escalate → feedback loop

   Governance:
      ├── FMEA RPN ≥ threshold → Step Functions waitForTaskToken → human
      ├── Guardrail blocks unsafe recs
      ├── Clarify bias monitor on classifier (per-year / per-region fairness)
      ├── CloudTrail → Object-Lock audit bucket
      └── Retraining loop: Accept/Reject labels → weekly SageMaker Pipeline
```

---

## 3. Execution plan

### Week 1 — Foundation + audio pipeline + classifier

| Day | Template (fill with kit params, paste to Claude) | Partials Claude MUST load |
|---|---|---|
| 1 | [`iac/02_cdk_ml_llm_infrastructure`](../iac/02_cdk_ml_llm_infrastructure.md) — scaffold CDK app with stacks: `NetworkStack`, `SecurityStack`, `DataLakeStack` (raw + curated audio buckets), `AudioPipelineStack`, `ClassifierStack`, `VectorStoreStack` (2 indexes), `ManualIngestionStack`, `RuntimeStack` (5 Strands agents), `ApiStack`, `FrontendStack`, `ObservabilityStack`, `ComplianceStack` | `LAYER_NETWORKING`, `LAYER_SECURITY`, `LAYER_DATA`, `LAYER_BACKEND_LAMBDA`, `LAYER_OBSERVABILITY` |
| 2 | Audio pipeline — instruct Claude "Generate `AudioPipelineStack` per **`MLOPS_AUDIO_PIPELINE`** partial with FEATURES={kit param}, PREPROCESS_SAMPLE_RATE={kit}, AUGMENTATION_ENABLED={kit}. Build the librosa+torchaudio+pywavelets+scipy container. Lambda for low-volume + SageMaker Processing job for batch." | **`MLOPS_AUDIO_PIPELINE`** (NEW), `EVENT_DRIVEN_PATTERNS`, `LAYER_BACKEND_LAMBDA` |
| 3 | Classifier training — [`mlops/01_sagemaker_training_pipeline`](../mlops/01_sagemaker_training_pipeline.md) OR [`mlops/02_llm_finetuning_pipeline`](../mlops/02_llm_finetuning_pipeline.md) (for LoRA). Instruct Claude "Fine-tune CLASSIFIER_MODEL={kit} on client's labeled audio + augmented public datasets (MIMII / DCASE)." | `MLOPS_SAGEMAKER_TRAINING`, `MLOPS_CLARIFY_EXPLAINABILITY` (bias monitor) |
| 4 | Classifier serving — [`mlops/03_llm_inference_deployment`](../mlops/03_llm_inference_deployment.md) OR [`mlops/05_model_monitoring_drift`](../mlops/05_model_monitoring_drift.md). Instruct Claude "Deploy classifier as SageMaker Async Inference for default; swap to sync or serverless per PROCESSING_LATENCY={kit}." | `MLOPS_SAGEMAKER_SERVING`, `MLOPS_BATCH_TRANSFORM` (for `PROCESSING_LATENCY=batch`) |
| 5 | Vector stores — instruct Claude "Generate `VectorStoreStack` per `DATA_S3_VECTORS` partial with TWO indexes: audio-embeddings (768-dim cosine per **`PATTERN_AUDIO_SIMILARITY_SEARCH`**) and manual-embeddings (1024-dim cosine). Implement EmbedderLambda per PATTERN_AUDIO_SIMILARITY_SEARCH storage strategy={kit}." | `DATA_S3_VECTORS`, **`PATTERN_AUDIO_SIMILARITY_SEARCH`** (NEW) |

### Week 2 — Agent + dashboard + governance + go-live

| Day | Template | Partials |
|---|---|---|
| 6 | Manual ingestion — instruct Claude "Generate `ManualIngestionStack` per `PATTERN_DOC_INGESTION_RAG` for client's TSB / repair manual corpus. Textract parser + fixed-512 chunker + Titan v2 embedder writing to the manuals index." | `PATTERN_DOC_INGESTION_RAG`, `DATA_S3_VECTORS` |
| 7 | Agent — branch on `AGENT_ARCHITECTURE`: **`strands_agentcore`** → [`mlops/22_strands_agentcore_deployment`](../mlops/22_strands_agentcore_deployment.md). Instruct Claude "Supervisor + 5 sub-agents: Classifier, SimilarCase, ManualKnowledge, DSPAnalyst (Code Interpreter), Synthesizer, Reviewer. Supervisor=Sonnet 4.7, sub-agents=Haiku 4.5, Reviewer=Sonnet 4.7. Every @tool sub-agent wrapped per `STRANDS_TOOLS`." | `STRANDS_AGENT_CORE`, `STRANDS_TOOLS`, `STRANDS_MODEL_PROVIDERS`, `STRANDS_MULTI_AGENT`, `STRANDS_DEPLOY_ECS`, `AGENTCORE_RUNTIME`, `AGENTCORE_IDENTITY`, `AGENTCORE_MEMORY`, `AGENTCORE_CODE_INTERPRETER` |
| 8 | Guardrail + grounding — [`mlops/12_bedrock_guardrails_agents`](../mlops/12_bedrock_guardrails_agents.md). Instruct Claude "Bedrock Guardrail blocks unsafe recommendations (`DENIED_RECS = ['just ignore it', 'fine to drive', 'no need to check']`). Cedar policy enforces 'never diagnose without citation'. Reviewer sub-agent uses `STRANDS_EVAL` grounding validator — every TSB citation checked against source." | `AGENTCORE_AGENT_CONTROL`, `STRANDS_EVAL` |
| 9 | Dashboard + API + observability — instruct Claude "Generate `ApiStack` (REST for upload + WebSocket for streaming progress per `LAYER_API`) + `FrontendStack` (React + CloudFront + OAC per `LAYER_FRONTEND`). Dashboard features: upload form, waveform/spectrogram preview, diagnosis card with RPN, similar-cases playback, trend chart (fleet mode), accept/reject/escalate buttons. Observability dashboards: classifier drift, grounding score, RPN distribution, tech-override rate, cost-per-diagnosis." | `LAYER_API`, `LAYER_FRONTEND`, `STRANDS_FRONTEND`, `LAYER_OBSERVABILITY`, `AGENTCORE_OBSERVABILITY` |
| 10 | Compliance + retraining loop + go-live — instruct Claude "Generate `ComplianceStack` per `COMPLIANCE_HIPAA_PCIDSS` with COMPLIANCE_STANDARD={kit}. Macie on raw audio bucket. WAF on CloudFront + API GW. Implement retraining loop: every Accept/Reject feedback → DDB → weekly SageMaker Pipeline retrain trigger via [`mlops/13_continuous_training_data_versioning`](../mlops/13_continuous_training_data_versioning.md)." | `LAYER_NETWORKING`, `COMPLIANCE_HIPAA_PCIDSS`, `SECURITY_WAF_SHIELD_MACIE`, `MLOPS_SAGEMAKER_TRAINING` |

---

## 4. Complete partial reference (load each into Claude along with the template)

> **URL template:** `https://github.com/surendersara1/F369_CICD_Template/blob/main/prompt_templates/partials/<NAME>.md`

**Core platform (ALWAYS load):**
- `LAYER_BACKEND_LAMBDA` — 5 non-negotiables + grant helpers
- `LAYER_NETWORKING` — VPC + PrivateLink
- `LAYER_SECURITY` — KMS + IAM + permission boundary
- `LAYER_DATA` — DDB for audio_metadata + feedback + machine history
- `LAYER_OBSERVABILITY` — dashboards, alarms
- `EVENT_DRIVEN_PATTERNS` — S3 → EventBridge (ingestion trigger)

**Audio-specific (NEW v2.0 partials this kit ships with):**
- **`MLOPS_AUDIO_PIPELINE`** (NEW) — audio ingestion + preprocessing + feature extraction (librosa, MFCC, PCP, CWT wavelet)
- **`PATTERN_AUDIO_SIMILARITY_SEARCH`** (NEW) — per-window vs aggregated vector strategy, temporal-context idioms

**ML platform:**
- `MLOPS_SAGEMAKER_TRAINING` — classifier fine-tune / LoRA
- `MLOPS_SAGEMAKER_SERVING` — Async / sync / serverless endpoint
- `MLOPS_BATCH_TRANSFORM` — fleet nightly sweep
- `MLOPS_CLARIFY_EXPLAINABILITY` — classifier fairness monitor

**Vector store + RAG:**
- `DATA_S3_VECTORS` — two indexes (audio 768-dim, manuals 1024-dim)
- `PATTERN_DOC_INGESTION_RAG` — TSB / manual corpus ingestion

**Bedrock + LLM:**
- `LLMOPS_BEDROCK` — model invocation, guardrail, cross-region inference

**Agent path (pick ONE branch based on `AGENT_ARCHITECTURE`):**
- Branch A (`plain_bedrock_lambda`): just `LLMOPS_BEDROCK`
- Branch B (`strands_lambda`): + `STRANDS_AGENT_CORE` + `STRANDS_TOOLS` + `STRANDS_MODEL_PROVIDERS` + `STRANDS_DEPLOY_LAMBDA`
- Branch C (`strands_agentcore`, DEFAULT): Branch B + `AGENTCORE_RUNTIME` + `AGENTCORE_IDENTITY` + `AGENTCORE_MEMORY` + `STRANDS_DEPLOY_ECS`

**Tools (for the agent's toolbox):**
- `AGENTCORE_GATEWAY` — MCP endpoint for all tools
- `AGENTCORE_CODE_INTERPRETER` — **key** — DSP analyst sub-agent runs librosa/scipy/pywt
- `AGENTCORE_BROWSER_TOOL` — optional — fetch public TSBs / recall bulletins
- `STRANDS_MCP_TOOLS` — client-side MCP SigV4 transport

**UI:**
- `LAYER_API` — REST + WebSocket
- `LAYER_FRONTEND` — React + CloudFront + OAC
- `STRANDS_FRONTEND` — streaming callback handler

**Governance (ALWAYS for this kit):**
- `AGENTCORE_AGENT_CONTROL` — Guardrail (PII + denied recs) + Cedar (never-without-citation)
- `STRANDS_EVAL` — grounding validator (every TSB citation checked)
- `COMPLIANCE_HIPAA_PCIDSS` — audit bucket + Backup Vault Lock
- `SECURITY_WAF_SHIELD_MACIE` — WAF + Macie on raw audio bucket
- `AGENTCORE_OBSERVABILITY` — drift alarms, per-diagnosis cost

---

## 5. Architecture non-negotiables (repeat in every Claude prompt)

> **Rules from `LAYER_BACKEND_LAMBDA §4.1` — all generated code MUST pass these:**
>
> 1. Lambda / container asset paths use `Path(__file__).resolve().parents[N] / "..."` (Python) or `path.join(__dirname, ...)` (TypeScript). Never CWD-relative.
> 2. Cross-stack resource access is **identity-side only** — attach `PolicyStatement` to the consumer's role. Never `bucket.grantReadWrite(crossStackRole)` or `endpoint.grant_invoke(crossStackRole)`.
> 3. Cross-stack EventBridge → Lambda uses L1 `events.CfnRule` with a static-ARN target.
> 4. Bucket + CloudFront OAC live in the **same stack** — never split.
> 5. Never `encryption_key=ext_key` where the key came from another stack. Pass KMS ARN as a **string** via SSM.
> 6. Every `iam:PassRole` on a SageMaker / AgentCore role includes `Condition: StringEquals iam:PassedToService <service>.amazonaws.com`.
> 7. **Audio-specific:** scope `s3vectors:*` IAM on the specific index ARN, not bucket wildcard. Scope `sagemaker:InvokeEndpoint` on the specific endpoint name. Batch `put_vectors` per AWS best practice.

---

## 6. Deliverables checklist (end of Week 2)

- [ ] CDK repo (`TARGET_LANGUAGE`) with ~12 stacks: Network, Security, DataLake, AudioPipeline, Classifier (training+serving), VectorStore (2 indexes), ManualIngestion, Runtime (5 Strands agents), Api, Frontend, Observability, Compliance
- [ ] Audio preprocessing pipeline: librosa container + Lambda + SageMaker Processing job, with all `FEATURES` wired
- [ ] Trained classifier (`CLASSIFIER_MODEL`) fine-tuned on client's labeled samples + public augmentation (MIMII / DCASE)
- [ ] SageMaker endpoint deployed per `PROCESSING_LATENCY` (Async default)
- [ ] Vector stores: `{project_name}-audio-vectors` index populated with penultimate embeddings; `{project_name}-manuals` index populated with TSB/manual RAG
- [ ] 5 Strands agents on AgentCore Runtime: supervisor + classifier + similarity + manual-knowledge + DSP analyst + synthesizer + reviewer
- [ ] Bedrock Guardrail + Cedar policy per domain pack (denied recs, never-without-citation)
- [ ] Reviewer grounding validator enforces citation-source matching
- [ ] React dashboard with:
    - upload form + live waveform + spectrogram render with anomaly overlay
    - diagnosis card (fault + confidence + RPN + citations)
    - similar-cases list with audio playback
    - trend chart (fleet mode)
    - accept/reject/escalate buttons → retraining loop
- [ ] Observability dashboards: classifier drift, grounding score, RPN distribution, tech-override rate, cost-per-diagnosis
- [ ] Compliance: Object-Lock audit bucket, Backup Vault Lock, CloudTrail, Config rules, Macie on raw bucket, WAF on CloudFront + API GW
- [ ] FMEA database populated (client-provided spreadsheet ingested + RPN computation module)
- [ ] Retraining loop: weekly SageMaker Pipeline triggered by feedback DDB
- [ ] Runbook — 3 sample scenarios per domain pack, expected outputs, rollback steps
- [ ] Pilot test — 20 labeled audio files per fault class, confusion matrix, before/after accuracy report

---

## 7. Known partial gaps — 2 NEW partials shipped with this kit

Running this kit design against the existing 61 partials surfaced exactly 2 gaps. Both shipped alongside this kit v1.0.

| File | Status | Scope |
|---|---|---|
| **`MLOPS_AUDIO_PIPELINE.md`** | ✅ NEW v2.0 | Audio ingestion + preprocessing (librosa, pywavelets, scipy) + feature extraction (mel-spec / MFCC / PCP / CWT / STE) + SageMaker Processing job + augmentation (SpecAugment / noise / time-stretch / pitch-shift) |
| **`PATTERN_AUDIO_SIMILARITY_SEARCH.md`** | ✅ NEW v2.0 | Audio-embedding similarity on S3 Vectors — per-window vs aggregated vs hybrid storage; temporal-context aggregation; query strategies (find-similar / machine-history / training-data-mining) |

**Still-open gaps (same as prior kits, unchanged):**
- No frontend / portal thin-router template — use `LAYER_FRONTEND` partial directly
- No SFN orchestration thin-router template — use `WORKFLOW_STEP_FUNCTIONS` directly
- No compliance-blueprint thin-router template — use `COMPLIANCE_HIPAA_PCIDSS` directly
- Library-wide: Claude 3.x / 4.0 IDs in template defaults — override via kit params

---

## 8. Design companion

The "why" behind this kit's shape — business case, UX rationale, 3 business-flow examples with sell prices and hour-savings, AWS Lookout-for-Equipment retirement context — lives at:

[`kits/_design/acoustic-fault-diagnostic-agent.md`](_design/acoustic-fault-diagnostic-agent.md)

Partners and sales leads read the design doc before pitching. Engineers delivering a 2-week engagement read this kit.

---

## 9. Changelog

| Version | Date | Change |
|---|---|---|
| 1.0 | 2026-04-22 | Initial kit. Grounded in NBS Audio team's 4 prior experiments (GeCo, YAMNet, PANNs, FaceNet-for-audio) + Nature 2021 paper (MFCC+PCP+STE→CNN+LSTM, 98%) + 2026 SOTA (AST, Wav2Vec2, LoRA fine-tune at 77.75% DCASE 2023 Task 2). Five decision branches (DOMAIN_PACK / CLASSIFIER_MODEL / INTAKE_MODE / PROCESSING_LATENCY / AGENT_ARCHITECTURE) + VECTOR_STRATEGY + FMEA_RPN escalation. Positioned as a replacement for AWS Lookout for Equipment (EOL 2026-10-07). Ships alongside 2 new v2.0 partials (`MLOPS_AUDIO_PIPELINE`, `PATTERN_AUDIO_SIMILARITY_SEARCH`) in F369_CICD_Template. Companion design doc at `kits/_design/acoustic-fault-diagnostic-agent.md`. |
