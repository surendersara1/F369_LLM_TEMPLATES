<!-- Kit Version: 1.0 | Template library: F369_LLM_TEMPLATES v1.x | Partials: F369_CICD_Template partials v2.0 -->

# Kit — Qualitative Research Audio Analytics (Multi-Persona, Production-Grade, 1K+ audio/day)

**Client ask:** "Analyze thousands of qualitative research audio recordings per month — focus groups, in-depth interviews, telephone surveys — with AI-driven transcription, summaries, sentiment, themes, quotes-with-audio-seek, and cross-study semantic search. Production-grade, multi-persona (Ops admin + Analyst + Moderator + Client-viewer), multi-tenant, governed."
**Engagement:** 2 weeks POC · 2 developers · evolution path to production well-documented
**Deliverable:** CDK Python repo (13 stacks) + React/Vite SPA with Admin + Analyst personas + Step Functions audio workflow + RDS PostgreSQL + DynamoDB ops tracking + S3 Vectors semantic search + mock data + runbook + deploy scripts.

**Scope rationale:** First kit in the library built for a **market-research-operations** domain (vs. HR / legal / BI). Industrial scale (1K+ audio/day), multi-persona UX (Ops vs Analyst vs Client), and "research deliverable" output shape (not scorecard, not BI dashboard) distinguish it from existing kits. See companion design doc at [`kits/_design/qualitative-research-audio-analytics.md`](_design/qualitative-research-audio-analytics.md) for business case, UX rationale, 3 business flows with pricing, and competitive positioning vs. Qualtrics / Dscout.

---

## 1. Kit-wide parameters (fill ONCE; carried to every template + Claude call)

```
# --- Identity -----------------------------------------------------------
PROJECT_NAME:                qra-{client_slug}
AWS_REGION:                  us-east-1  (any region with Transcribe + Bedrock + S3 Vectors GA)
AWS_ACCOUNT_ID:              [12-digit]
ENV:                         dev | stage | prod
CLIENT_SLUG:                 research_america
TARGET_LANGUAGE:             python    (CDK default; TypeScript available on request)

# --- DOMAIN PACK — drives schema + prompts + demo data ----------------
DOMAIN_PACK:                 qualitative_research    ← NEW pack (this is the first client)
CLIENT_VERTICALS:            [pharma, retail, finance, healthcare, automotive]
RECORDING_TYPES:             [focus_group, in_depth_interview, telephone_survey, ethnography]

# --- SCALE --------------------------------------------------------------
EXPECTED_DAILY_LOAD:         250          # 7000 recordings / 28 business days
EXPECTED_PEAK_HOURLY_LOAD:   40           # morning drop-of-prior-night pattern
MAX_RECORDING_DURATION_MIN:  180          # 3-hour focus group ceiling
AUDIO_FORMATS:               [mp3, wav, m4a, mp4, aac, flac]
MAX_FILE_SIZE_MB:            2048         # 2 GB — 3h WAV stereo

# --- TRANSCRIPTION ------------------------------------------------------
TRANSCRIBE_LANGUAGES:        [en-US, es-US]   # POC; expand per vertical
SPEAKER_DIARIZATION:         true
MAX_SPEAKERS:                10           # focus group upper bound
CUSTOM_VOCAB_BY_VERTICAL:    true         # pharma drug names etc.
AUTO_LANGUAGE_ID:            true         # for mixed-language sessions

# --- AI ANALYSIS --------------------------------------------------------
SUMMARIZATION_MODEL_ID:      us.anthropic.claude-sonnet-4-7-20260109-v1:0
CODING_MODEL_ID:             us.anthropic.claude-haiku-4-5-20251001-v1:0
EMBEDDING_MODEL_ID:          amazon.titan-embed-text-v2:0
EMBEDDING_DIM:               1024
SENTIMENT_METHOD:            bedrock      # Claude-based (not Amazon Comprehend — poorer on research voice)
QUOTE_EXTRACTION:            enabled
QUOTE_MAX_PER_SESSION:       30
THEME_EXTRACTION:            enabled
PER_QUOTE_SENTIMENT:         true         # sentiment at quote level, not just session
CROSS_SESSION_AGGREGATION:   enabled      # project-level theme roll-up
REPORT_DRAFTING:             enabled      # DOCX/PPTX auto-draft

# --- PII / GOVERNANCE ---------------------------------------------------
PII_REDACTION_ENABLED:       true         # comprehend + Bedrock Guardrails hybrid
PII_REDACTION_TARGETS:       [name, phone, email, ssn, address, dob, medical_record_number]
TRANSCRIBE_CONTENT_REDACTION: true         # Transcribe-level PII masking
BEDROCK_GUARDRAILS_ENABLED:  true
HIPAA_READY_FLAG:            false        # true when pharma client w/ BAA signed

# --- STORAGE STRATEGY ---------------------------------------------------
# DDB for operational / time-series / status
# RDS Aurora Postgres for structured business data
# S3 Vectors for semantic search
RDS_ENGINE:                  aurora_postgres_v16
RDS_SERVERLESS_V2:           true
RDS_MIN_ACU:                 0.5
RDS_MAX_ACU:                 8            # scale with load
DDB_TABLES:                  [jobs, audit_log, cost_ledger, session_state, rate_limit]
S3_VECTOR_INDEXES:           [quotes, transcript_chunks, themes]

# --- MULTI-TENANCY ------------------------------------------------------
TENANT_MODEL:                per_client_isolation
TENANT_PK_STRATEGY:          client_id_prefix_on_every_pk  # e.g. "acme#job123"
CROSS_TENANT_DENY:           strict       # IAM condition keys + runtime verification

# --- RETENTION ----------------------------------------------------------
RETENTION_DAYS_AUDIO:        90           # client-configurable; Glacier after
RETENTION_DAYS_TRANSCRIPT:   365
RETENTION_DAYS_INSIGHTS:     2555         # 7 years (market research IP)
RETENTION_DAYS_AUDIT_LOG:    2555
S3_LIFECYCLE_TO_GLACIER_DAYS: 180

# --- AUTH & ROLES -------------------------------------------------------
AUTH:                        cognito_user_pool
ROLES:                       [Admin, Analyst, Moderator, ClientViewer]
SAML_FEDERATION_READY:       true         # SAML IdP stub for Phase 2
STATIC_CREDS_POC:            true         # seed admin + 3 analyst users for demo
MFA_REQUIRED_ROLES:          [Admin]
SESSION_MAX_MINUTES:         60
REFRESH_TOKEN_DAYS:          30

# --- COST GUARDRAILS ----------------------------------------------------
COST_BUDGET_MONTHLY_USD:     25000        # 25% headroom over SOW $19,708/month projection
COST_PER_JOB_TARGET_USD:     3.00         # $19,708 / 7000 ≈ $2.82 baseline
TRANSCRIBE_DAILY_BUDGET_MIN: 15000        # 250 × 60 min
BEDROCK_DAILY_TOKEN_BUDGET:  5_000_000
COST_ANOMALY_ALARM_PCT:      30           # alarm when daily spend > 30% over 7d avg

# --- UI / UX ------------------------------------------------------------
UI_FRAMEWORK:                react_vite   # SOW says React.js; Next.js SSR not needed
UI_LIBRARY:                  tailwind_radix   # Tailwind CSS + Radix primitives
AUDIO_PLAYER_WAVEFORM:       true         # wavesurfer.js — click quote → seek
ADMIN_DASHBOARD_REFRESH_SEC: 5
ANALYST_DASHBOARD_PAGE_SIZE: 25
EXPORT_FORMATS:              [docx, pptx, pdf, md, csv]
I18N_ENABLED:                false        # Phase 2

# --- FRONTEND HOSTING ---------------------------------------------------
CDN:                         cloudfront
FRONTEND_BUCKET_OAC:         true
WAF_RULESET:                 [aws_managed_core, aws_managed_sqli, rate_limit_1000rpm]
ALLOWED_EMBED_DOMAINS:       [https://app.{client_slug}.com]

# --- INTEGRATIONS (POC = none; Phase 2) --------------------------------
QUALTRICS_INTEGRATION:       disabled
SLACK_NOTIFICATIONS:         disabled
WEBHOOK_ENDPOINTS:           []
CRM_SYNC:                    disabled

# --- COMPLIANCE ---------------------------------------------------------
COMPLIANCE_STANDARD:         SOC2                # HIPAA flag available for pharma-client phase
CLOUDTRAIL_ENABLED:          true
CONFIG_RULES_ENABLED:        true
MACIE_ON_AUDIO_BUCKET:       true
OBJECT_LOCK_AUDIT_BUCKET:    true
AUDIT_LOG_TO_S3:             true
```

---

## 2. Architecture

```
                            ┌─────────────────────────────────────┐
                            │   Browser UI (React + Vite + CF+OAC)│
                            │                                      │
                            │  /login  /analyst/*  /admin/*        │
                            │  /admin/ops   /admin/cost            │
                            │  /admin/quality  /admin/users        │
                            │  /analyst/projects  /analyst/search  │
                            │  /analyst/library  /analyst/session/:id│
                            └──────────────┬───────────────────────┘
                                           │
                                           │  HTTPS (WAF + API GW REST)
                                           │  Cognito JWT auth
                                           ▼
  ┌──────────────────────────────────────────────────────────────────────────────────┐
  │  API Gateway (REST, regional, with WAF)                                           │
  │                                                                                    │
  │  /upload           → UploadLambda (presigned URL)                                 │
  │  /jobs             → JobsApiLambda (list/filter/get)                              │
  │  /jobs/:id/status  → StatusLambda (DDB read, fast)                                │
  │  /jobs/:id         → InsightsApiLambda (transcript + insights + quotes)           │
  │  /projects         → ProjectsApiLambda                                            │
  │  /quotes/:id/audio → AudioSeekLambda (presigned S3 URL w/ ?t=timestamp)           │
  │  /search           → SemanticSearchLambda                                         │
  │  /library/ask      → RagChatLambda (Ask-My-Library)                               │
  │  /export           → ExportLambda (DOCX/PPTX generation)                          │
  │                                                                                    │
  │  /admin/ops        → AdminOpsLambda (realtime ops stats from DDB)                 │
  │  /admin/cost       → AdminCostLambda (cost ledger)                                │
  │  /admin/quality    → AdminQualityLambda (WER + PII-hit stats)                     │
  │  /admin/users      → AdminUsersLambda (Cognito mgmt)                              │
  │  /admin/tenants    → AdminTenantsLambda (per-client config)                       │
  └──────────────────┬─────────────────────────────────────────────────────────────────┘
                     │
                     │
  ┌──────────────────┼─────────────────────────────────────────────────────────────────┐
  │                  ▼                                                                 │
  │  S3 audio-raw  ──► S3 ObjectCreated event ──► EventBridge ──► Step Functions      │
  │                                                                                    │
  │  ┌──────────────────────────────────────────────────────────────────────┐         │
  │  │ Step Functions "AudioProcessingWorkflow"                              │         │
  │  │                                                                        │         │
  │  │  1. ValidateAudio (Lambda): format, size, virus scan gate             │         │
  │  │  2. CreateJobRow (Lambda): DDB jobs row = QUEUED                      │         │
  │  │  3. StartTranscriptionJob (SDK task): Transcribe async                 │         │
  │  │  4. WaitForTranscription (Wait + Poll pattern)                         │         │
  │  │  5. ParseTranscript (Lambda): normalize JSON, speaker labels          │         │
  │  │  6. BedrockAnalysis (Lambda, parallel branch):                        │         │
  │  │       - summary                                                        │         │
  │  │       - sentiment (overall + per-quote)                                │         │
  │  │       - themes                                                         │         │
  │  │       - quote extraction w/ timestamps                                 │         │
  │  │       - code suggestions (pre-apply traditional market-research codes) │         │
  │  │  7. PiiRedaction (Lambda): Comprehend + Guardrails sweep              │         │
  │  │  8. EmbeddingGeneration (Lambda): Titan v2 on chunks + quotes         │         │
  │  │  9. PersistInsights (Lambda): RDS writes + S3 Vectors PutVectors      │         │
  │  │ 10. UpdateJobRow (Lambda): DDB jobs row = COMPLETE + duration         │         │
  │  │ 11. EmitMetrics (Lambda): CloudWatch cost + quality metrics           │         │
  │  │                                                                        │         │
  │  │ Error handling: per-step Catch → DLQ + audit log + user notification  │         │
  │  └──────────────────────────────────────────────────────────────────────┘         │
  └──────────────────────────────────────────────────────────────────────────────────┘
                     │                                     │
                     │                                     │
                     ▼                                     ▼
  ┌──────────────────────────┐              ┌──────────────────────────────────────┐
  │  DynamoDB (operational)  │              │  RDS Aurora Postgres Serverless v2   │
  │                          │              │                                      │
  │  Table: jobs             │              │  Table: users (Cognito shadow)       │
  │    PK: client_id#job_id  │              │  Table: clients                      │
  │    GSIs: by status,      │              │  Table: projects                     │
  │          by analyst,     │              │  Table: sessions  (recordings)       │
  │          by date         │              │  Table: transcripts (full-text GIN)  │
  │    TTL: null (keep)      │              │  Table: insights  (JSONB)            │
  │                          │              │  Table: quotes    (with ts, speaker) │
  │  Table: audit_log        │              │  Table: themes    (per project)      │
  │    PK: tenant#date       │              │  Table: codes     (coding framework) │
  │    SK: user#timestamp    │              │  Table: exports                      │
  │    TTL: 7 yrs            │              │  Table: tenant_configs               │
  │                          │              │                                      │
  │  Table: cost_ledger      │              │  Read replicas: 1 (dev), 2 (prod)    │
  │    PK: client#date       │              │  Snapshots: 7-day PITR               │
  │    SK: job_id            │              └──────────────────────────────────────┘
  │                          │
  │  Table: session_state    │
  │    (Cognito sessions)    │              ┌──────────────────────────────────────┐
  │    TTL: session max      │              │  S3 Vectors (semantic search)        │
  │                          │              │                                      │
  │  Table: rate_limit       │              │  Bucket: qra-vectors-{env}           │
  │    PK: client#endpoint   │              │  Index: quotes        (1024 cosine)  │
  │    SK: minute_bucket     │              │  Index: transcript_chunks            │
  │    TTL: 5 min            │              │  Index: themes                       │
  └──────────────────────────┘              │  Metadata: tenant_id (filter pushdown)│
                                             └──────────────────────────────────────┘

                        ▲                    ▲                    ▲
                        │                    │                    │
     ┌──────────────────┴─────┐    ┌─────────┴─────────┐    ┌────┴────────────────┐
     │ Admin dashboards read  │    │ Analyst dashboards │    │ Cross-study search  │
     │ from DDB (ops-cockpit) │    │ read from RDS + DDB│    │ queries S3 Vectors  │
     └────────────────────────┘    └────────────────────┘    └─────────────────────┘

  Observability:  CloudWatch dashboards (ops + cost + quality) + SNS alarms
  Compliance:      CloudTrail → Object-Lock audit bucket, Config rules, Macie on audio
  Security:        WAF, KMS (3 CMKs — audio, db, reports), Cognito + MFA for Admin
```

---

## 3. Execution plan — 2-week POC, 2 developers

### Week 1 — Foundation + pipeline

| Day | Template | Partials Claude MUST load |
|---|---|---|
| **1** | [`iac/02_cdk_ml_llm_infrastructure`](../iac/02_cdk_ml_llm_infrastructure.md) — Scaffold CDK app with 13 stacks. Client briefing: walkthrough + demo of HR-interview-analyzer as reference shape. | `LAYER_BACKEND_LAMBDA`, `LAYER_NETWORKING`, `LAYER_SECURITY` |
| **2** | [`devops/02_vpc_networking_ml`](../devops/02_vpc_networking_ml.md) — `NetworkStack` with PrivateLink (Transcribe, Bedrock, S3, RDS, DDB). [`devops/08_kms_encryption_ml`](../devops/08_kms_encryption_ml.md) — `SecurityStack` with 3 CMKs. | `LAYER_NETWORKING`, `LAYER_SECURITY` |
| **3** | **Data layer** — `DataStack` (RDS Aurora v2 per `DATA_AURORA_SERVERLESS_V2` + 5 DDB tables per `LAYER_DATA`). `MediaStack` (3 S3 buckets: audio-raw, transcripts, reports per `LAYER_BACKEND_LAMBDA` same-stack-bucket + OAC rule). `VectorStack` per `DATA_S3_VECTORS`. Apply schema migrations via Flyway or Alembic custom resource. | `DATA_AURORA_SERVERLESS_V2`, `LAYER_DATA`, `DATA_S3_VECTORS` |
| **4** | **Ingest + orchestration** — `UploadApiStack` per `LAYER_API` (presigned URL pattern). `OrchestrationStack` with Step Functions per `WORKFLOW_STEP_FUNCTIONS`. Wire S3 → EB → SFN per `EVENT_DRIVEN_PATTERNS`. | `LAYER_API`, `WORKFLOW_STEP_FUNCTIONS`, `EVENT_DRIVEN_PATTERNS` |
| **5** | **Transcription + analysis Lambdas** — `StartTranscriptionJob`, `ParseTranscript`, `BedrockAnalysis` (fan-out: summary + sentiment + themes + quotes), `PiiRedaction`, `EmbeddingGeneration`, `PersistInsights`. PROMPT LIBRARY for Bedrock per vertical. | (See §5 prompt library) |

### Week 2 — UX + dashboards + quality + delivery

| Day | Template | Partials |
|---|---|---|
| **6** | `AuthStack` per `LAYER_SECURITY` — Cognito user pool with 4 role groups + MFA for Admin + static-creds seed for POC demo. `InsightsApiStack` + `AdminApiStack` per `LAYER_API` — separate API paths for Admin-only ops. | `LAYER_SECURITY`, `LAYER_API` |
| **7** | **Frontend foundation** — React + Vite + Tailwind + Radix + react-router. Auth context, typed API client, role-gated routes. `FrontendStack` per `LAYER_FRONTEND` (S3 + CloudFront + OAC). | `LAYER_FRONTEND` |
| **8** | **Analyst persona UI** — `/analyst/projects` (list), `/analyst/projects/:id` (detail), `/analyst/session/:id` (transcript viewer + audio player with waveform + quote-to-seek), `/analyst/search` (cross-study semantic search). [`mlops/25_react_portal_cloudfront`](../mlops/25_react_portal_cloudfront.md) for hosting. | `LAYER_FRONTEND` |
| **9** | **Admin persona UI** — `/admin/ops` (realtime ops cockpit w/ 5-sec refresh from DDB), `/admin/cost` (cost ledger + anomaly alarms), `/admin/quality` (WER + PII-hit stats), `/admin/users` (Cognito mgmt), `/admin/tenants` (per-client config). [`devops/11_custom_cloudwatch_model_quality`](../devops/11_custom_cloudwatch_model_quality.md) for metric feeder. | `LAYER_OBSERVABILITY`, `LAYER_FRONTEND` |
| **10** | **Observability + Compliance + Go-live** — `ObservabilityStack` (ops dashboards + cost alarms + quality alarms). `ComplianceStack` per `COMPLIANCE_HIPAA_PCIDSS` (Object-Lock audit bucket, Config rules, CloudTrail, Macie on audio) — POC mode SOC2; HIPAA flag ready for Phase 2. UAT checklist execution. System documentation + deployment guide. | `COMPLIANCE_HIPAA_PCIDSS`, `LAYER_OBSERVABILITY`, `SECURITY_WAF_SHIELD_MACIE` |

---

## 4. Complete partial reference

**Core platform (ALWAYS load):**
- `LAYER_BACKEND_LAMBDA` — 5 non-negotiables
- `LAYER_NETWORKING` — VPC + PrivateLink endpoints
- `LAYER_SECURITY` — KMS + IAM + Cognito patterns
- `LAYER_DATA` — DDB patterns (operational tables)
- `LAYER_API` — API GW REST + WebSocket
- `LAYER_FRONTEND` — React + CloudFront + OAC
- `LAYER_OBSERVABILITY` — dashboards + alarms

**Data:**
- `DATA_AURORA_SERVERLESS_V2` — RDS Aurora Postgres v2 (authoritative canonical — see `../../F369_CICD_Template/prompt_templates/partials/README.md` Canonical Registry)
- `DATA_S3_VECTORS` — vector storage (authoritative)
- `DATA_MULTITENANT_DDB` — per-client isolation pattern *(verify exists; if not, flag as Kit 6 partial gap)*

**Orchestration / Events:**
- `WORKFLOW_STEP_FUNCTIONS` — SFN state machine
- `EVENT_DRIVEN_PATTERNS` — S3 → EB → SFN wiring
- `EVENT_DRIVEN_FAN_IN_AGGREGATOR` — project-level aggregation across sessions

**Governance:**
- `COMPLIANCE_HIPAA_PCIDSS` — baseline (SOC2 mode; HIPAA toggle Phase 2)
- `SECURITY_WAF_SHIELD_MACIE` — WAF + Macie on audio bucket

**NEW partials this kit surfaces** (authored inline here in the POC; extracted to `F369_CICD_Template/prompt_templates/partials/` post-engagement per Canonical-Copy Rule's maintenance workflow):

1. **`PATTERN_AUDIO_TRANSCRIPTION_PIPELINE.md`** — Amazon Transcribe + SFN orchestration + speaker diarization + language auto-detect + retry/DLQ. (`MLOPS_AUDIO_PIPELINE` covers acoustic ML, not Transcribe-qualitative.)
2. **`PATTERN_QUOTE_MINING_WITH_TIMESTAMPS.md`** — Bedrock prompts for quote + timestamp + speaker-role + per-quote-sentiment extraction; canonical DB + vector schema.
3. **`PATTERN_AI_REPORT_DRAFTING.md`** — Bedrock-driven DOCX/PPTX auto-draft with citation insertion + editable draft persistence.
4. **`PATTERN_OPERATIONAL_COST_TRACKING.md`** — DDB-based cost ledger (job → client → project), CloudWatch metric feeders, cost anomaly alarm.
5. **`PATTERN_AUDIO_SEEK_INTEGRATION.md`** — client-side wavesurfer.js integration with timestamp URL params + range-get S3 presigned URLs for jump-to-moment playback.

---

## 5. Bedrock prompt library

Prompt templates live at `cdk/lambda/bedrock_analysis/prompts/v1/` and are versioned per `mlops/24_bedrock_prompt_management.md` pattern.

- **`summary.txt`** — 3-paragraph executive summary tuned for market research deliverable voice
- **`sentiment_overall.txt`** — transcript-level sentiment (-1 to +1) + confidence
- **`sentiment_per_quote.txt`** — quote-level sentiment with justification
- **`themes.txt`** — 5-10 themes with respondent frequency + representative quotes
- **`quotes.txt`** — 15-30 best quotes with: speaker label, timestamp, theme tag, sentiment, relevance score
- **`codes.txt`** — traditional market-research coding (preset codebook per vertical, AI pre-applies)
- **`report_draft.txt`** — 3-section DOCX draft: Executive Summary / Key Findings / Verbatim Themes
- **`report_draft.pptx.txt`** — 8-slide PPTX deck structure

Per-vertical overrides: `prompts/v1/pharma/quotes.txt` (knows about drug names, patient journey framing), `prompts/v1/retail/themes.txt` (knows shopper journey vocabulary), etc.

---

## 6. Architecture non-negotiables (repeat in every Claude prompt)

> **Rules from `LAYER_BACKEND_LAMBDA §4.1` — all generated code MUST pass:**
>
> 1. Lambda / container asset paths use `Path(__file__).resolve().parents[N] / "..."` (Python). Never CWD-relative.
> 2. Cross-stack resource access is **identity-side only**. Never `bucket.grant_*(crossStackRole)`.
> 3. Cross-stack EventBridge → Lambda uses L1 `events.CfnRule` with static-ARN target.
> 4. Bucket + CloudFront OAC live in the **same stack**. Never split.
> 5. KMS ARNs cross-stack are **strings via SSM**. Not imported L2 `kms.Key`.
>
> **Kit 6-specific additions:**
>
> 6. **Strict tenant isolation** — every DDB/RDS/S3/S3-Vectors query MUST include the `tenant_id` (client_id) as a partition key or filter. IAM conditions MUST enforce `aws:ResourceTag/tenant_id == request tenant_id` on all data-plane reads. No tenant-inferred-from-cookie; tenant comes from Cognito custom claim.
> 7. **Audio presigned URLs** use **Range-GET** and **per-clip-scoped path restriction** — a quote's "seek to 04:22" URL does NOT authorise reading the whole file; it authorises the byte range of that clip only. Built via S3 `GetObject` with `Range` header + scoped presigned URL.
> 8. **Cost attribution on every Bedrock call** — every InvokeModel wraps a decorator that writes to DDB `cost_ledger` table with (client_id, project_id, job_id, model_id, input_tokens, output_tokens, usd_estimate) before returning.
> 9. **PII redaction runs before embedding generation** — embedding a transcript with respondent names bakes PII into vectors forever (vectors aren't easily redactable after the fact).
> 10. **Admin routes isolated to `/admin/*`** on API GW + Cognito group check in the Lambda authorizer. Frontend role-gates client-side for UX, but server-side authorization is the security boundary.

---

## 7. Deliverables checklist (end of Week 2)

- [ ] CDK Python app with 13 stacks, all synth-clean
- [ ] Step Functions workflow deployed + executes end-to-end on sample audio
- [ ] All 11 Lambda functions deployed + IAM-scoped
- [ ] RDS Aurora Postgres v2 with 11-table schema + migrations applied
- [ ] 5 DDB tables (jobs, audit_log, cost_ledger, session_state, rate_limit)
- [ ] 3 S3 Vectors indexes (quotes, transcript_chunks, themes) with seed data
- [ ] Cognito user pool with 4 role groups + MFA for Admin + 4 POC test users seeded
- [ ] React SPA deployed to CloudFront + OAC with Admin + Analyst routes
- [ ] Admin ops cockpit live (5-sec refresh from DDB)
- [ ] Analyst session detail page with click-quote → audio-seek UX
- [ ] Cross-study semantic search returning quotes + studies
- [ ] DOCX + PPTX export working on a sample project
- [ ] CloudWatch observability dashboards (ops + cost + quality) operational
- [ ] WAF + Macie + CloudTrail + Object-Lock audit bucket in place
- [ ] 20+ mock audio files processed end-to-end for demo
- [ ] UAT script + outcomes doc
- [ ] System documentation + deployment guide + runbook
- [ ] Handover session with Research America team + AWS Public Use Case draft

---

## 8. Golden-set tests (nightly regression after go-live)

20 sample audio files across verticals. Each run asserts:

| Assertion | Metric | Pass threshold |
|---|---|---|
| Upload → insight end-to-end latency (60 min recording) | Total SFN duration | p95 < 15 min |
| Transcription word error rate on sampled reference | WER | < 5% (en-US clean audio) |
| Quote extraction recall vs human-labeled golden set | Recall@30 | > 80% |
| Sentiment accuracy vs human-labeled | F1 | > 0.75 |
| PII redaction recall on synthetic-PII audio | Recall | > 99% (zero-leak target) |
| Cross-study search @5 recall | Recall@5 | > 85% |
| Admin ops cockpit refresh latency | p95 DDB read | < 200 ms |
| Audio-seek click → audio playing delay | p95 | < 500 ms |

Failing any assertion = alert to pager + deployment freeze for that promotion window.

---

## 9. Design companion

See [`kits/_design/qualitative-research-audio-analytics.md`](_design/qualitative-research-audio-analytics.md) for:
- Full business case + Research America's strategic context
- Persona UX design (Admin cockpit + Analyst workspace)
- 3 business flow examples (pharma focus group / retail consumer insights / telephone-survey quant-qual hybrid)
- Pricing + payback math (~$275K pitch for production build)
- Competitive positioning vs. Qualtrics / Dscout / SPSS
- Phase 2 + Phase 3 roadmap
- Known risks + mitigations

---

## 10. Changelog

| Version | Date | Change |
|---|---|---|
| 1.0 | 2026-04-23 | Initial kit. First DOMAIN_PACK=qualitative_research. Multi-persona (Admin / Analyst / Moderator / ClientViewer), 1K+ audio/day scale, per-client tenant isolation, DDB+RDS+S3-Vectors tri-store, click-quote-audio-seek UX, DOCX/PPTX auto-drafting, cross-study semantic search, cost attribution per Bedrock call. Surfaces 5 new partials to be extracted post-engagement. Companion design doc at `kits/_design/qualitative-research-audio-analytics.md`. |
