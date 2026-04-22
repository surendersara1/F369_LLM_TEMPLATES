<!-- Kit Version: 1.1 | Template library: F369_LLM_TEMPLATES v1.x | Partials: F369_CICD_Template partials v2.0 -->

# Kit — HR Interview Analyzer

**Client ask:** "Recruiter uploads interview videos, system extracts audio + video, analyses each question, scores candidates automatically, produces a structured report with evidence."
**Engagement:** 2 weeks · 2 developers · regulated (protected-class bias monitoring required).
**Deliverable:** CDK repo (TypeScript or Python) + React portal + scoring workers, deployable end-to-end.

**Validated against real production build:** emapta-avar (AI-Powered Virtual Automated Recruitment), 9 CDK stacks, 29 Lambdas, 2 Fargate containers, Aurora Postgres Serverless v2 — live at `d122b1xsbt6roz.cloudfront.net`.

---

## 1. Kit-wide parameters (fill ONCE; carried to every template + Claude call)

```
# --- Identity -----------------------------------------------------------
PROJECT_NAME:           hr-interview-{client_slug}          e.g. hr-interview-acme
AWS_REGION:             us-east-1
AWS_ACCOUNT_ID:         [12-digit]
ENV:                    dev | stage | prod
CLIENT_SLUG:            acme

# --- LANGUAGE TARGET (new in v1.1) --------------------------------------
TARGET_LANGUAGE:        typescript | python
                        typescript → CDK with `aws-cdk-lib` (Node.js), npm package mgmt
                        python     → CDK v2 Python (`aws_cdk`), pip / poetry mgmt
                        Partials are Python-first; for typescript, add this line to every
                        Claude prompt: "Translate the partial's CDK code from Python to
                        TypeScript idiomatic style, preserving the 5 non-negotiables."

# --- ARCHITECTURE DECISIONS (new in v1.1 — set at engagement kickoff) ---
MEDIA_PIPELINE:         mediaconvert | fargate_ffmpeg
                        mediaconvert   → simpler, AWS-managed, enough for standard extraction
                        fargate_ffmpeg → custom container (Librosa/OpenSMILE/PRAAT), needed
                                         for voice-emotion / pitch-variance / prosody features
                        DEFAULT: fargate_ffmpeg (matches emapta-avar; richer voice features)

VISUAL_ANALYSIS:        rekognition | fargate_opencv_bedrock
                        rekognition          → built-in faces/expressions/attention
                        fargate_opencv_bedrock → OpenCV frame sampling + Claude vision on
                                                 sampled frames; more control, higher cost
                        DEFAULT: rekognition (unless client wants custom frame logic)

SCORE_STORE:            dynamodb | aurora_serverless_v2
                        dynamodb           → flat keyed lookup, fast, cheap
                        aurora_serverless_v2 → JOINs, window functions, role-based rubrics,
                                               reporting queries across candidate × rubric × dim
                        DEFAULT: aurora_serverless_v2 if SCORING_DIMENSIONS > 3 OR
                                 reporting requirements include cohort / percentile comparisons

AGENT_ARCHITECTURE:     strands_agentcore | strands_lambda | plain_bedrock_lambda
                        strands_agentcore     → multi-agent orchestration, long sessions (8h),
                                                microVM isolation; for complex scoring agents
                        strands_lambda        → single-agent scoring with @tool fns in Lambda
                        plain_bedrock_lambda  → direct InvokeModel per modality; simpler, less
                                                orchestration. Matches emapta-avar reference.
                        DEFAULT: plain_bedrock_lambda (per-modality classifier is single-shot,
                                 agentic coordination overkill for HR scoring)

INTAKE_MODE:            single | batch
                        single → one video per upload
                        batch  → recruiter uploads manifest + N candidate videos (ZIP or
                                 presigned-URL list); system fans out per-candidate
                        DEFAULT: batch (matches emapta-avar; single is a subset capability)

# --- Business logic -----------------------------------------------------
SCORING_DIMENSIONS:     [communication, technical, cultural_fit, red_flags]
ROLE_RUBRIC_S3_PATH:    s3://{PROJECT_NAME}-{AWS_ACCOUNT_ID}-{ENV}-config/rubrics/{role}.json
COMPANY_VALUES:         [client-specific; e.g. "customer_obsession, ownership, dive_deep"]
MAX_VIDEO_MINUTES:      60
CONCURRENT_UPLOADS:     10
MIN_SCORING_CONFIDENCE: 0.70   # below this, flag for human review

# --- Compliance ---------------------------------------------------------
PII_REDACTION_FIELDS:   [candidate_name, prior_employer, dob, protected_class]
RETENTION_DAYS:         90    # after hiring decision
COMPLIANCE_STANDARD:    SOC2  # or HIPAA if healthcare-adjacent client
BIAS_MONITOR_ENABLED:   true

# --- Models -------------------------------------------------------------
MODEL_ID:               us.anthropic.claude-sonnet-4-7-20260109-v1:0   # Sonnet 4.7
FALLBACK_MODEL_ID:      us.anthropic.claude-haiku-4-5-20251001-v1:0    # Haiku 4.5
VISION_MODEL:           us.anthropic.claude-sonnet-4-7-20260109-v1:0

# --- Deployment topology ------------------------------------------------
DEPLOY_MODE:            microstack   # 'monolith' for POC; 'microstack' for prod
```

> **⚠ Model-ID note:** sub-templates in `mlops/`, `devops/` etc. still ship Claude 3.5 / 4.0 IDs. Override with the `MODEL_ID` / `FALLBACK_MODEL_ID` / `VISION_MODEL` above — library-wide model-ID sweep is a separate uplift pass.

---

## 2. Architecture (swap variants driven by §1 parameters)

```
Recruiter uploads interview video(s)
          │                      ┌─── INTAKE_MODE=batch → PATTERN_BATCH_UPLOAD partial
          │                      │      (manifest → fan-out one SQS msg per candidate)
          │◀─────────────────────┘
          ▼
  CloudFront → S3 (presigned URL, KMS-encrypted)
          │  S3 → EventBridge → IngestLambda
          ▼
  Media pipeline (MEDIA_PIPELINE decision):
     ├── mediaconvert   → MediaConvert job → audio.wav + 480p preview + chapters
     └── fargate_ffmpeg → ECS Fargate (FFmpeg) → audio.wav + frames + waveform
          │
          ▼
  Parallel analysis fan-out (EVENT_DRIVEN_PATTERNS partial):
     ├── Transcribe              → transcript JSON (diarization)
     ├── Audio features          → Fargate + Librosa/OpenSMILE/PRAAT (voice emotion)
     │                            OR just Comprehend sentiment
     ├── Visual analysis (VISUAL_ANALYSIS decision):
     │     ├── rekognition            → Rekognition Video: expressions/attention
     │     └── fargate_opencv_bedrock → OpenCV frames → Claude vision via Bedrock
     └── Segmenter Lambda         → chunk transcript into per-question segments
          │
          ▼
  FAN-IN AGGREGATOR (EVENT_DRIVEN_FAN_IN_AGGREGATOR partial — NEW):
     aggregator Lambda waits for all N stream results keyed by {interview_id, question_id},
     computes weighted final score from SCORING_DIMENSIONS × role rubric.
          │
          ▼
  Scoring logic (AGENT_ARCHITECTURE decision):
     ├── plain_bedrock_lambda → direct InvokeModel per dimension, inline rubric
     ├── strands_lambda       → Strands Agent with @tool score_* fns, Bedrock model
     └── strands_agentcore    → AgentCore Runtime container, multi-tool orchestration
     + Bedrock Guardrail (PII redaction on outputs)
     + Clarify bias monitor on scoring model
          │
          ▼
  Storage (SCORE_STORE decision):
     ├── dynamodb             → flat keyed scores table + S3 curated (transcripts, reports)
     └── aurora_serverless_v2 → DATA_AURORA_SERVERLESS_V2 partial (NEW):
                                 complex scoring JOINs, role rubrics, reporting queries
          │
          ▼
  UI: React + CloudFront + OAC (LAYER_FRONTEND partial)
      API GW REST (report) + API GW WebSocket (live scoring progress)
          │
          ▼
  Governance: CloudTrail → Object-Lock audit bucket; Macie on uploads; Clarify bias;
              WAF on CloudFront + API GW; AgentCore Guardrail on Bedrock outputs.
```

---

## 3. Execution plan

### Week 1 — Foundation + media pipeline + analysis workers

| Day | Template (fill with kit params, paste to Claude) | Partials Claude MUST load |
|---|---|---|
| 1 | [`iac/02_cdk_ml_llm_infrastructure`](../iac/02_cdk_ml_llm_infrastructure.md) — scaffolds CDK app with stacks for each decision in §1 | `LAYER_NETWORKING`, `LAYER_SECURITY`, `LAYER_DATA`, `LAYER_BACKEND_LAMBDA`, `LAYER_OBSERVABILITY` |
| 2 | **IF `INTAKE_MODE=batch`:** instruct Claude "generate a `BatchIntakeStack` per `PATTERN_BATCH_UPLOAD` partial — presigned URL + batch manifest handler + per-candidate SQS fan-out + progress DDB." | `PATTERN_BATCH_UPLOAD` (NEW), `EVENT_DRIVEN_PATTERNS`, `LAYER_BACKEND_LAMBDA` |
| 3 | Media pipeline — **IF `MEDIA_PIPELINE=mediaconvert`:** [`mlops/15_multimodal_processing_pipelines`](../mlops/15_multimodal_processing_pipelines.md). **IF `fargate_ffmpeg`:** instruct Claude "generate a `MediaProcessingStack` with ECS Fargate + FFmpeg container per `LAYER_BACKEND_ECS` partial; orchestrated by Step Functions per `WORKFLOW_STEP_FUNCTIONS`." | `WORKFLOW_STEP_FUNCTIONS`, `EVENT_DRIVEN_PATTERNS`, `LAYER_BACKEND_ECS`, `LAYER_BACKEND_LAMBDA` |
| 4 | Analysis workers — **IF `VISUAL_ANALYSIS=rekognition`:** [`mlops/15`] already covers it. **IF `fargate_opencv_bedrock`:** instruct Claude "generate a `VisualAnalysisStack` with Frame Analyzer Lambda + OpenCV + Bedrock vision per `LAYER_BACKEND_LAMBDA` + `LLMOPS_BEDROCK`." For audio-features-heavy clients, add Librosa Fargate container. | `LAYER_BACKEND_LAMBDA`, `LAYER_BACKEND_ECS`, `LLMOPS_BEDROCK`, `EVENT_DRIVEN_PATTERNS` |
| 5 | Fan-in aggregator — instruct Claude "generate an `AggregatorStack` per `EVENT_DRIVEN_FAN_IN_AGGREGATOR` partial — aggregator Lambda waits for transcript + audio + visual streams keyed by {interview_id, question_id}, computes weighted final score." | `EVENT_DRIVEN_FAN_IN_AGGREGATOR` (NEW), `LAYER_BACKEND_LAMBDA`, `LAYER_DATA` |

### Week 2 — Scoring + storage + UI + governance + go-live

| Day | Template | Partials |
|---|---|---|
| 6 | Scoring — branch on `AGENT_ARCHITECTURE`: **`plain_bedrock_lambda`** → instruct Claude "generate `ScoringStack` with Lambda per-dimension scorers per `LLMOPS_BEDROCK`". **`strands_lambda`** → [`mlops/20_strands_agent_lambda_deployment`]. **`strands_agentcore`** → [`mlops/22_strands_agentcore_deployment`]. | `LLMOPS_BEDROCK` (always) · plus `STRANDS_AGENT_CORE`+`STRANDS_TOOLS`+`STRANDS_DEPLOY_LAMBDA` (strands_lambda) · plus `AGENTCORE_RUNTIME`+`STRANDS_DEPLOY_ECS` (strands_agentcore) |
| 7 | Storage — **IF `SCORE_STORE=aurora_serverless_v2`:** instruct Claude "generate `AuroraScoringStack` per `DATA_AURORA_SERVERLESS_V2` partial; include RDS Data API access for serverless workers." **IF `dynamodb`:** already covered by `LAYER_DATA`. | `DATA_AURORA_SERVERLESS_V2` (NEW, for aurora) OR `LAYER_DATA` (for DDB) |
| 8 | Guardrails + bias — [`mlops/12_bedrock_guardrails_agents`](../mlops/12_bedrock_guardrails_agents.md) + [`devops/14_clarify_realtime_bias_monitoring`](../devops/14_clarify_realtime_bias_monitoring.md) | `AGENTCORE_AGENT_CONTROL`, `MLOPS_CLARIFY_EXPLAINABILITY` |
| 9 | UI + observability — instruct Claude "generate `FrontendStack` (React + CloudFront + OAC per `LAYER_FRONTEND`), `ApiStack` (REST + WebSocket per `LAYER_API`), dashboards per `LAYER_OBSERVABILITY`." | `LAYER_FRONTEND`, `LAYER_API`, `LAYER_OBSERVABILITY`, `AGENTCORE_OBSERVABILITY` |
| 10 | Compliance + network — [`devops/02_vpc_networking_ml`](../devops/02_vpc_networking_ml.md) + [`devops/07_macie_pii_training_data`](../devops/07_macie_pii_training_data.md) + instruct Claude "generate `ComplianceStack` per `COMPLIANCE_HIPAA_PCIDSS` with COMPLIANCE_STANDARD={kit param}." | `LAYER_NETWORKING`, `COMPLIANCE_HIPAA_PCIDSS`, `SECURITY_WAF_SHIELD_MACIE` |

---

## 4. Complete partial reference (load into each Claude conversation as context)

> **URL template:** `https://github.com/surendersara1/F369_CICD_Template/blob/main/prompt_templates/partials/<NAME>.md`

**Core platform (ALWAYS load):**
- `LAYER_BACKEND_LAMBDA` — 5 non-negotiables, grant helpers
- `LAYER_NETWORKING` — VPC + PrivateLink
- `LAYER_SECURITY` — KMS + IAM + permission boundary
- `LAYER_DATA` — DDB + S3 curated
- `LAYER_OBSERVABILITY` — dashboards, alarms

**Ingestion / orchestration:**
- `PATTERN_BATCH_UPLOAD` *(NEW v2.0 — if `INTAKE_MODE=batch`)*
- `WORKFLOW_STEP_FUNCTIONS`
- `EVENT_DRIVEN_PATTERNS`
- `EVENT_DRIVEN_FAN_IN_AGGREGATOR` *(NEW v2.0 — always for this kit)*

**Media + analysis:**
- `LAYER_BACKEND_ECS` *(if `MEDIA_PIPELINE=fargate_ffmpeg` OR audio Fargate)*
- `LLMOPS_BEDROCK`

**Agent (pick ONE branch based on `AGENT_ARCHITECTURE`):**
- Branch A (`plain_bedrock_lambda`): just `LLMOPS_BEDROCK`
- Branch B (`strands_lambda`): `STRANDS_AGENT_CORE` + `STRANDS_TOOLS` + `STRANDS_MODEL_PROVIDERS` + `STRANDS_DEPLOY_LAMBDA`
- Branch C (`strands_agentcore`): branch B partials + `AGENTCORE_RUNTIME` + `AGENTCORE_IDENTITY` + `STRANDS_DEPLOY_ECS`

**Storage (pick ONE based on `SCORE_STORE`):**
- `dynamodb`: `LAYER_DATA` (already loaded)
- `aurora_serverless_v2`: `DATA_AURORA_SERVERLESS_V2` *(NEW v2.0)*

**UI:**
- `LAYER_API` — REST + WebSocket
- `LAYER_FRONTEND` — React + CloudFront + OAC

**Governance (ALWAYS for HR; load for this kit unconditionally):**
- `AGENTCORE_AGENT_CONTROL` — Guardrail + Cedar policy
- `MLOPS_CLARIFY_EXPLAINABILITY` — bias + SHAP
- `COMPLIANCE_HIPAA_PCIDSS` — audit bucket + Backup Vault Lock
- `SECURITY_WAF_SHIELD_MACIE` — WAF + Macie
- `AGENTCORE_OBSERVABILITY` — token usage + drift alarms
- `STRANDS_EVAL` *(if `AGENT_ARCHITECTURE` uses Strands)* — online eval + grounding

---

## 5. Architecture non-negotiables (repeat in every Claude prompt)

> **Rules from `LAYER_BACKEND_LAMBDA §4.1` — all generated code MUST pass these, regardless of `TARGET_LANGUAGE`:**
>
> 1. Lambda / container asset paths use `Path(__file__).resolve().parents[N] / "..."` (Python) or equivalent `path.join(__dirname, ...)` (TypeScript). Never CWD-relative.
> 2. Cross-stack resource access is **identity-side only** — attach `PolicyStatement` to the consumer's role. Never `bucket.grantReadWrite(crossStackRole)` or `key.grantDecrypt(crossStackRole)`.
> 3. Cross-stack EventBridge → Lambda uses **L1 `events.CfnRule`** with a static-ARN target, NOT `targets.SqsQueue(q)` or `targets.LambdaFunction(fn)` on cross-stack consumers.
> 4. Bucket + CloudFront OAC live in the **same stack** — never split.
> 5. Never `encryption_key=ext_key` (Python) / `encryptionKey: extKey` (TS) where the key came from another stack. Pass KMS ARN as a **string** via SSM.
> 6. Every `iam:PassRole` on a SageMaker / Glue / ECS task role includes `Condition: StringEquals iam:PassedToService <service>.amazonaws.com`.

---

## 6. Deliverables checklist (end of Week 2)

- [ ] CDK repo (`TARGET_LANGUAGE`) with 8–10 stacks depending on decisions made in §1
- [ ] Lambda handlers for: ingest, segmenter, per-modality scorers, aggregator, batch-upload-handler (if batch), report API
- [ ] Fargate containers: media-processor (FFmpeg) + audio-analyzer (Librosa et al.) — if `MEDIA_PIPELINE=fargate_ffmpeg`
- [ ] Bedrock Guardrail wired to all scoring calls
- [ ] Clarify bias-monitor baseline + monthly schedule
- [ ] Compliance: Object-Lock audit bucket, Backup Vault Lock, CloudTrail multi-region
- [ ] WAF on CloudFront + API GW
- [ ] React portal deployed to CloudFront + OAC'd S3
- [ ] Role rubrics JSON schema + 3 sample rubrics
- [ ] Runbook (markdown) with rollback steps + sample scoring report
- [ ] `tests/sop/test_*.py` (Python) or `test/*.test.ts` (TS) pytest/Jest harnesses from the partials' §6 worked examples — all synthesize offline

---

## 7. Known gaps flagged to the library maintainer

Running this kit v1.0 (HR Interview Analyzer) against the emapta-avar reference revealed and filled these gaps. **v1.1 updates in this document address them.**

| Gap | Status | File |
|---|---|---|
| No SQS fan-in aggregator partial | ✅ ADDED v2.0 | `EVENT_DRIVEN_FAN_IN_AGGREGATOR.md` |
| No Aurora Serverless v2 deep-dive partial | ✅ ADDED v2.0 | `DATA_AURORA_SERVERLESS_V2.md` |
| No batch-upload / many-to-one intake partial | ✅ ADDED v2.0 | `PATTERN_BATCH_UPLOAD.md` |
| No frontend/portal thin-router template | ⏳ STILL OPEN — consultants use `LAYER_FRONTEND` partial directly | future `mlops/25_react_portal_cloudfront.md` |
| No SFN orchestration thin-router template | ⏳ STILL OPEN — consultants use `WORKFLOW_STEP_FUNCTIONS` partial directly | future `devops/17_step_functions_orchestration.md` |
| No compliance-blueprint thin-router template | ⏳ STILL OPEN — consultants use `COMPLIANCE_HIPAA_PCIDSS` partial directly | future `enterprise/06_compliance_blueprint.md` |
| No Fargate ML custom-container template | ⏳ STILL OPEN — for Librosa / OpenSMILE / custom audio workloads | future `mlops/26_fargate_ml_container.md` |
| No Aurora ML scoring template | ⏳ STILL OPEN — `DATA_AURORA_SERVERLESS_V2` partial covers patterns but no thin-router | future `data/06_aurora_ml_scoring.md` |
| Library-wide: stale model IDs (Claude 3.5 / 4.0 in OPTIONAL defaults) | ⏳ STILL OPEN — one-shot sweep recommended | all `mlops/`, `devops/` templates |

---

## 8. Changelog

| Version | Date | Change |
|---|---|---|
| 1.1 | 2026-04-22 | Validated against emapta-avar reference implementation (9 CDK stacks, 29 Lambdas, 2 Fargate containers, Aurora Postgres). Added: `TARGET_LANGUAGE` parameter (Python/TypeScript support via partial translation prompt); per-modality **decision branches** (`MEDIA_PIPELINE`, `VISUAL_ANALYSIS`, `SCORE_STORE`, `AGENT_ARCHITECTURE`, `INTAKE_MODE`) — consultant picks at engagement kickoff. Partials `PATTERN_BATCH_UPLOAD`, `DATA_AURORA_SERVERLESS_V2`, `EVENT_DRIVEN_FAN_IN_AGGREGATOR` added to F369_CICD_Template; kit now references them explicitly. Retained all §5 non-negotiables. Updated deliverables to reflect language-agnostic output. Retained gap list for library-wide follow-up (4 missing thin-router templates + model-ID sweep). |
| 1.0 | 2026-04-22 | Initial kit. Chains 10 existing templates + 2 partial-only steps across 2 weeks. References partials on public GitHub (F369_CICD_Template @ main). Flagged 3 missing partials that v1.1 fills. |
