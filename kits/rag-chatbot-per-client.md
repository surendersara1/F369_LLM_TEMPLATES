<!-- Kit Version: 1.0 | Template library: F369_LLM_TEMPLATES v1.x | Partials: F369_CICD_Template partials v2.0 -->

# Kit — RAG Chatbot (One per Client)

**Client ask:** "Stand up a custom chatbot for this client, grounded in their documents. They upload PDFs; end users ask questions; chatbot answers with citations. One isolated deployment per client."
**Engagement:** 2 weeks · 2 developers · moderate compliance (SOC 2 default; HIPAA-ready via kit parameters).
**Deliverable:** CDK repo (TypeScript or Python) + React chat UI + ingestion pipeline + retrieval-augmented chat Lambda, deployable end-to-end per client.

**Grounding:** S3 Vectors as vector store (verified GA, Nov 2025 + 2026), Bedrock Titan Text Embeddings V2 (1024-dim cosine), Bedrock Claude for generation. Architecture verified against AWS user guide + CloudFormation reference + `awslabs/s3vectors-embed-cli` tool.

---

## 1. Kit-wide parameters (fill ONCE; carried to every template + Claude call)

```
# --- Identity -----------------------------------------------------------
PROJECT_NAME:               rag-chatbot-{client_slug}          e.g. rag-chatbot-acme
AWS_REGION:                 us-east-1  OR us-east-2 / us-west-2 / eu-central-1 / eu-west-1 /
                            eu-west-2 / eu-west-3 / eu-north-1 / ap-southeast-1 / ap-southeast-2 /
                            ap-south-1 / ap-northeast-1 / ap-northeast-2 / ca-central-1
                            (S3 Vectors is in 14 regions as of GA — confirm for client data-residency)
AWS_ACCOUNT_ID:             [12-digit]
ENV:                        dev | stage | prod
CLIENT_SLUG:                acme
TARGET_LANGUAGE:            typescript | python
                            partials are Python-first; for TS add to every Claude prompt:
                            "Translate partial CDK from Python to TypeScript, preserving 5 non-negotiables."

# --- DECISIONS set at kickoff (change → major rework) -------------------
VECTOR_STORE_MODE:          custom_s3_vectors | bedrock_kb_s3_vectors | bedrock_kb_opensearch
                            custom_s3_vectors  → DIY: your chunker + Titan calls + put_vectors.
                                                 MAX CONTROL; matches explicit client ask.
                                                 DEFAULT for this kit.
                            bedrock_kb_s3_vectors → Bedrock KB manages chunking/embedding,
                                                    storage in S3 Vectors. Simpler, ~75% control.
                            bedrock_kb_opensearch → legacy, more expensive, only if hybrid
                                                    aggregation search is required.

DOCUMENT_PARSER:            textract | unstructured | pypdf
                            textract     → OCR-capable, scanned PDFs, best extraction accuracy,
                                           highest cost per page (~$0.0015).
                            unstructured → open-source, many formats, runs in Lambda layer;
                                           good balance, limited scanned-PDF accuracy.
                            pypdf        → native PDFs only, cheapest, no OCR.
                            DEFAULT: textract (enterprise clients); unstructured (startups).

CHUNKING_STRATEGY:          fixed | semantic | by_section
                            fixed       → N tokens + overlap. Cheapest, predictable.
                            semantic    → sentence-similarity aware splitter. Better retrieval
                                          quality, ~2× cost of fixed.
                            by_section  → markdown/heading-aware. Best for structured docs.
                            DEFAULT: fixed (good-enough for most RAG; escalate to semantic
                                     if grounding score <0.7 after first eval pass).

AGENT_ARCHITECTURE:         plain_bedrock_lambda | strands_lambda | strands_agentcore
                            plain_bedrock_lambda → direct InvokeModel from Lambda, simplest,
                                                   matches single-turn RAG well. DEFAULT.
                            strands_lambda       → Strands Agent with @tool retrieve_chunks fn;
                                                   use if kit grows to multi-tool (SQL, web, etc.).
                            strands_agentcore    → AgentCore Runtime container; use if
                                                   sessions >15 min or microVM isolation required.

CONVERSATION_MEMORY:        ddb_session | s3_session_manager | agentcore_memory | none
                            ddb_session         → app-managed rolling window in DDB (simplest).
                                                  DEFAULT.
                            s3_session_manager  → Strands built-in, S3-backed per-session.
                            agentcore_memory    → AWS-managed STM + LTM strategies (summary,
                                                  user preference, semantic). Only when agent
                                                  is deployed to AgentCore Runtime.
                            none                → stateless chat (Q&A only, no multi-turn).

EXPECTED_QPS:               low (<10 per-user rpm) | moderate (10-100 rpm) | high (100+ rpm)
                            S3 Vectors is explicitly positioned for "less frequent queries" —
                            sub-second for infrequent, ~100ms for frequent. For high QPS add
                            scheduled snapshot export to OpenSearch Serverless for hot path.
                            DEFAULT: moderate (chatbot sessions rarely sustained high QPS).

# --- Document ingestion --------------------------------------------------
SUPPORTED_FORMATS:          [pdf, docx, txt, md, html]
MAX_FILE_SIZE_MB:           100
CHUNK_SIZE_TOKENS:          512
CHUNK_OVERLAP_TOKENS:       64

# --- Embeddings (confirmed: AWS Titan V2) --------------------------------
EMBEDDING_MODEL_ID:         amazon.titan-embed-text-v2:0
EMBEDDING_DIMENSION:        1024     # Titan V2 default; 512 or 256 reduced variants also supported
EMBEDDING_NORMALIZE:        true     # Titan V2 is symmetric + normalized

# --- Vector store (confirmed: S3 Vectors) --------------------------------
VECTOR_BUCKET_NAME:         {project_name}-vectors-{env}
VECTOR_INDEX_NAME:          {project_name}-docs
VECTOR_DATA_TYPE:           float32   # only float32 supported currently
VECTOR_DISTANCE_METRIC:     cosine    # cosine | euclidean (lowercase in API)
VECTOR_TOP_K:               5
NON_FILTERABLE_METADATA:    ["source_text"]   # canonical RAG idiom — stores chunk text
FILTERABLE_METADATA:        ["doc_id", "page", "section", "uploaded_at", "access_group"]

# --- Chat LLM ------------------------------------------------------------
MODEL_ID:                   us.anthropic.claude-sonnet-4-7-20260109-v1:0
FALLBACK_MODEL_ID:          us.anthropic.claude-haiku-4-5-20251001-v1:0
MAX_CONTEXT_TOKENS:         8000    # bounded for prompt caching hit-rate
ENABLE_PROMPT_CACHING:      true    # system prompt + retrieved chunks cached per turn
SYSTEM_PROMPT_S3_PATH:      s3://{PROJECT_NAME}-config-{env}/prompts/system.md

# --- Session / memory config --------------------------------------------
CONVERSATION_WINDOW:        10     # last N turns in context (rolling)
SESSION_TTL_HOURS:          24

# --- UI ------------------------------------------------------------------
STREAM_RESPONSES:           true   # WebSocket token streaming
SHOW_CITATIONS:             true   # render retrieved chunks as clickable refs
SUPPORT_FEEDBACK:           true   # thumbs up/down + free-text → eval store

# --- Governance ----------------------------------------------------------
PII_REDACTION_FIELDS:       [name, email, phone, ssn, credit_card]
DENIED_TOPICS:              [competitor_info, legal_advice, medical_advice]
COMPLIANCE_STANDARD:        SOC2 | HIPAA | PCI_DSS | NONE
RETENTION_DAYS:             365     # conversations + citations; 2555 (7y) for HIPAA

# --- Deployment topology -------------------------------------------------
DEPLOY_MODE:                microstack   # 'monolith' for POC; 'microstack' for prod
```

---

## 2. Architecture

```
Admin / recruiter uploads PDFs + docs
         │
         ▼
   CloudFront → S3 raw bucket (KMS-encrypted, block-public, versioned)
         │  S3 → EventBridge → IngestionLambda
         ▼
   Document ingestion pipeline (PATTERN_DOC_INGESTION_RAG partial — NEW):
       ├── Parse (per DOCUMENT_PARSER):
       │     textract | unstructured | pypdf
       ├── Chunk (per CHUNKING_STRATEGY):
       │     fixed (512 tokens + 64 overlap) | semantic | by_section
       ├── Embed (Titan V2 → 1024-dim cosine):
       │     bedrock-runtime.invoke_model(amazon.titan-embed-text-v2:0)
       └── Store (per VECTOR_STORE_MODE):
             custom_s3_vectors    → s3vectors.put_vectors (batched)
             bedrock_kb_s3_vectors → Bedrock KB ingestion job
             bedrock_kb_opensearch → Bedrock KB ingestion job (OS backend)
         │
         ▼
   DDB doc_metadata table: {doc_id, filename, chunk_count, status, uploaded_at, indexed_at}

End user asks question (React chat + CloudFront + OAC)
         │
         ▼
   API GW v2 WebSocket (streaming) → ChatLambda
         ├── Embed question via Titan V2 (same model as ingestion — symmetric)
         ├── s3vectors.query_vectors(topK=5, filter={access_group: ...}, returnMetadata=True)
         ├── Build prompt:
         │     system prompt (cached) + retrieved chunks (cached) + history + user Q
         ├── bedrock-runtime.converse_stream(Sonnet 4.7, guardrailConfig=...)
         ├── Stream tokens back via WebSocket
         └── Log query + citation keys + confidence to DDB for audit + eval
         │
         ▼
   Session state (per CONVERSATION_MEMORY):
       ddb_session | s3_session_manager | agentcore_memory | none
         │
         ▼
   Governance:
       Bedrock Guardrail (PII redact + deny topics) on chat model
       Grounding validator (STRANDS_EVAL) on every response
       CloudTrail → Object-Lock audit bucket (COMPLIANCE_HIPAA_PCIDSS)
       WAF on CloudFront + API GW
       Macie on raw upload bucket
```

---

## 3. Execution plan

### Week 1 — Foundation + ingestion pipeline

| Day | Template (fill with kit params, paste to Claude) | Partials Claude MUST load |
|---|---|---|
| 1 | [`iac/02_cdk_ml_llm_infrastructure`](../iac/02_cdk_ml_llm_infrastructure.md) — scaffold CDK app with `NetworkStack`, `SecurityStack`, `DataStack`, `StorageStack` (raw docs), `VectorStoreStack`, `IngestionStack`, `ChatStack`, `ApiStack`, `FrontendStack`, `ObservabilityStack`, `ComplianceStack` | `LAYER_NETWORKING`, `LAYER_SECURITY`, `LAYER_DATA`, `LAYER_BACKEND_LAMBDA`, `LAYER_OBSERVABILITY` |
| 2 | Vector store — instruct Claude "Generate `VectorStoreStack` per `DATA_S3_VECTORS` partial: CfnVectorBucket + CfnIndex with DIMENSION={kit param}, DISTANCE_METRIC={kit param}, NON_FILTERABLE_METADATA_KEYS={kit param}. Publish vectors/{bucket_arn, bucket_name, index_arn, index_name, kms_arn} to SSM." | `DATA_S3_VECTORS` (NEW) |
| 3 | Document storage + intake — instruct Claude "Generate `StorageStack` for raw doc uploads: S3 bucket with EventBridge notifications + KMS + block-public. Optionally presigner Lambda per `PATTERN_BATCH_UPLOAD` if admin bulk-upload is required." | `PATTERN_BATCH_UPLOAD` *(optional)*, `EVENT_DRIVEN_PATTERNS`, `LAYER_BACKEND_LAMBDA` |
| 4 | Ingestion pipeline — instruct Claude "Generate `IngestionStack` per `PATTERN_DOC_INGESTION_RAG` partial with DOCUMENT_PARSER={kit}, CHUNKING_STRATEGY={kit}, EMBEDDING_MODEL_ID={kit}. Consume raw-bucket ARN via SSM; put_vectors into S3 Vectors index via SSM. Write `doc_metadata` DDB row per doc with status transitions." | `PATTERN_DOC_INGESTION_RAG` (NEW), `DATA_S3_VECTORS` (NEW), `LLMOPS_BEDROCK`, `LAYER_BACKEND_LAMBDA` |
| 5 | First end-to-end smoke — upload a sample PDF, verify chunks in S3 Vectors, verify DDB row. Dev loop tip: use [`awslabs/s3vectors-embed-cli`](https://github.com/awslabs/s3vectors-embed-cli) for ad-hoc embed+put before the Lambda pipeline is ready. | (smoke test, no new partials) |

### Week 2 — Chat path + UI + governance + go-live

| Day | Template | Partials |
|---|---|---|
| 6 | Chat agent — branch on `AGENT_ARCHITECTURE`: **`plain_bedrock_lambda`** → instruct Claude "Generate `ChatStack` Lambda that embeds user question via Titan V2, calls s3vectors.query_vectors(topK=5), builds prompt with cached system+chunks, streams via bedrock-runtime.converse_stream." **`strands_lambda`** → [`mlops/20_strands_agent_lambda_deployment`](../mlops/20_strands_agent_lambda_deployment.md) with `@tool retrieve_chunks`. **`strands_agentcore`** → [`mlops/22_strands_agentcore_deployment`](../mlops/22_strands_agentcore_deployment.md). | `LLMOPS_BEDROCK` (always) · for Strands: `STRANDS_AGENT_CORE`+`STRANDS_TOOLS`+`STRANDS_MODEL_PROVIDERS`+`STRANDS_DEPLOY_LAMBDA` · for AgentCore: + `AGENTCORE_RUNTIME`+`AGENTCORE_IDENTITY` |
| 7 | Prompt caching + conversation — [`mlops/18_prompt_caching_patterns`](../mlops/18_prompt_caching_patterns.md). Per `CONVERSATION_MEMORY`: ddb_session is simplest (custom DDB table), s3_session_manager uses Strands built-in, agentcore_memory requires AgentCore Runtime. | `LLMOPS_BEDROCK`, `AGENTCORE_MEMORY` *(if agentcore_memory)* |
| 8 | Guardrails + grounding eval — [`mlops/12_bedrock_guardrails_agents`](../mlops/12_bedrock_guardrails_agents.md) (PII_REDACTION_FIELDS + DENIED_TOPICS) + [`mlops/17_llm_evaluation_pipeline`](../mlops/17_llm_evaluation_pipeline.md) with `STRANDS_EVAL` grounding-validator gating hallucinated citations. | `AGENTCORE_AGENT_CONTROL`, `STRANDS_EVAL` |
| 9 | UI + API + observability — instruct Claude "Generate `ApiStack` (REST for session config + WebSocket for streaming per `LAYER_API`), `FrontendStack` (React + CloudFront + OAC per `LAYER_FRONTEND`), dashboards per `LAYER_OBSERVABILITY` showing ingestion latency, query p95, grounding score, Titan token cost, feedback sentiment." | `LAYER_API`, `LAYER_FRONTEND`, `LAYER_OBSERVABILITY`, `AGENTCORE_OBSERVABILITY` |
| 10 | Compliance + go-live — [`devops/02_vpc_networking_ml`](../devops/02_vpc_networking_ml.md) + [`devops/07_macie_pii_training_data`](../devops/07_macie_pii_training_data.md) + instruct Claude "Generate `ComplianceStack` per `COMPLIANCE_HIPAA_PCIDSS` with COMPLIANCE_STANDARD={kit}. Wire CloudTrail to audit bucket, WAF to CloudFront + API GW." | `LAYER_NETWORKING`, `COMPLIANCE_HIPAA_PCIDSS`, `SECURITY_WAF_SHIELD_MACIE` |

---

## 4. Complete partial reference (load each into Claude along with the template)

> **URL template:** `https://github.com/surendersara1/F369_CICD_Template/blob/main/prompt_templates/partials/<NAME>.md`

**Core platform (ALWAYS load):**
- `LAYER_BACKEND_LAMBDA` — 5 non-negotiables + grant helpers
- `LAYER_NETWORKING` — VPC + PrivateLink (Bedrock / Textract / S3 Vectors all have VPCE or run serverless)
- `LAYER_SECURITY` — KMS + IAM + permission boundary
- `LAYER_DATA` — DDB for doc metadata + conversation state
- `LAYER_OBSERVABILITY` — dashboards, alarms
- `EVENT_DRIVEN_PATTERNS` — S3 → EventBridge (ingestion trigger)

**Vector store + ingestion (NEW v2.0 partials this kit ships with):**
- `DATA_S3_VECTORS` **(NEW)** — vector bucket + index CDK (`CfnVectorBucket` + `CfnIndex`), boto3 `put_vectors`/`query_vectors`, batching, non-filterable metadata idiom
- `PATTERN_DOC_INGESTION_RAG` **(NEW)** — parse → chunk → embed → store; decision branches for parser/chunker/store; Titan V2 wiring; DLQ + idempotency

**Bedrock + LLM:**
- `LLMOPS_BEDROCK` — model invocation, guardrail config, cross-region inference profiles

**Agent path (pick ONE branch based on `AGENT_ARCHITECTURE`):**
- Branch A (`plain_bedrock_lambda`): just `LLMOPS_BEDROCK`
- Branch B (`strands_lambda`): + `STRANDS_AGENT_CORE` + `STRANDS_TOOLS` + `STRANDS_MODEL_PROVIDERS` + `STRANDS_DEPLOY_LAMBDA`
- Branch C (`strands_agentcore`): Branch B + `AGENTCORE_RUNTIME` + `AGENTCORE_IDENTITY` + `STRANDS_DEPLOY_ECS`

**Memory (pick ONE based on `CONVERSATION_MEMORY`):**
- `ddb_session`: just `LAYER_DATA` (already loaded)
- `s3_session_manager`: `STRANDS_AGENT_CORE §3.3`
- `agentcore_memory`: `AGENTCORE_MEMORY`
- `none`: skip

**UI:**
- `LAYER_API` — REST + WebSocket streaming
- `LAYER_FRONTEND` — React + CloudFront + OAC
- `STRANDS_FRONTEND` *(if Strands path)* — callback streaming handler

**Governance (ALWAYS for this kit):**
- `AGENTCORE_AGENT_CONTROL` — Guardrail (PII + deny topics) + optional Cedar policy
- `STRANDS_EVAL` — grounding validator catches hallucinated citations before return
- `COMPLIANCE_HIPAA_PCIDSS` — audit bucket + Backup Vault Lock + Config rules
- `SECURITY_WAF_SHIELD_MACIE` — WAF on CloudFront + API GW, Macie on raw bucket
- `AGENTCORE_OBSERVABILITY` — token usage + drift alarms on scoring of retrieval quality

---

## 5. Architecture non-negotiables (repeat in every Claude prompt)

> **Rules from `LAYER_BACKEND_LAMBDA §4.1` — all generated code MUST pass these, regardless of `TARGET_LANGUAGE`:**
>
> 1. Lambda asset paths use `Path(__file__).resolve().parents[N] / "..."` (Python) or `path.join(__dirname, ...)` (TypeScript). Never CWD-relative.
> 2. Cross-stack resource access is **identity-side only** — attach `PolicyStatement` to the consumer's role. Never `bucket.grantReadWrite(crossStackRole)` or `key.grantDecrypt(crossStackRole)`.
> 3. Cross-stack EventBridge → Lambda uses L1 `events.CfnRule` with a static-ARN target.
> 4. Bucket + CloudFront OAC live in the **same stack** — never split.
> 5. Never `encryption_key=ext_key` where the key came from another stack. Pass KMS ARN as a **string** via SSM.
> 6. Every `iam:PassRole` on a Bedrock-invoker / Textract / Lambda execution role includes `Condition: StringEquals iam:PassedToService <service>.amazonaws.com`.
> 7. **S3 Vectors-specific:** scope `s3vectors:*` IAM on the specific index ARN, not bucket wildcard. Index ARN format: `arn:aws:s3vectors:{region}:{account}:bucket/{bucket}/index/{index}`.

---

## 6. Deliverables checklist (end of Week 2)

- [ ] CDK repo (in `TARGET_LANGUAGE`) with 9–11 stacks depending on decisions
- [ ] S3 vector bucket + 1 index (`{project_name}-docs`) provisioned with `non_filterable_metadata_keys=["source_text"]`
- [ ] Ingestion Lambda: parse + chunk + Titan V2 embed + batched `put_vectors`
- [ ] `doc_metadata` DDB table with status transitions (`pending → parsing → chunking → embedding → indexed | failed`)
- [ ] Chat Lambda (or Strands agent) with:
    - Titan V2 question embedding
    - `s3vectors.query_vectors(topK=5, returnMetadata=True)` with optional `filter` on filterable metadata
    - Prompt assembly with system + chunks + history + question
    - `bedrock-runtime.converse_stream` with guardrail + prompt caching
    - Streaming output via WebSocket
- [ ] Bedrock Guardrail attached (PII redaction + denied topics)
- [ ] Grounding validator on every response — flag if score < 0.7
- [ ] React chat UI with streaming + citations + feedback buttons
- [ ] CloudWatch dashboards:
    - Ingestion: chunks/sec, Titan token cost per doc, ingestion p95
    - Retrieval: query p95, top-K relevance, grounding score distribution
    - Safety: guardrail blocks per day, feedback ratio
- [ ] Compliance: Object-Lock audit bucket, Backup Vault Lock, Config rules, CloudTrail multi-region
- [ ] WAF on CloudFront + API GW
- [ ] Runbook + test fixtures + sample `system.md` prompt + per-client rubric

---

## 7. Known gaps and assumptions

**Addressed by v2.0 partials shipped with this kit:**
| Gap | Status | File |
|---|---|---|
| No partial for S3 Vectors | ✅ ADDED | `DATA_S3_VECTORS.md` (865 lines) |
| No partial for document ingestion RAG pipeline | ✅ ADDED | `PATTERN_DOC_INGESTION_RAG.md` (1054 lines) |

**Still open (same gaps as HR kit):**
| Gap | Severity | Suggested file |
|---|---|---|
| No frontend / portal thin-router template | LOW — use `LAYER_FRONTEND` partial directly | future `mlops/25_react_portal_cloudfront.md` |
| No SFN orchestration thin-router template | LOW — not needed for this kit but reusable | future `devops/17_step_functions_orchestration.md` |
| No compliance-blueprint thin-router template | MED — consultants compose from partial directly | future `enterprise/06_compliance_blueprint.md` |
| Library-wide: stale Claude 3.x / 4.0 model IDs | MED — override via `MODEL_ID` kit param | all `mlops/*` template OPTIONAL defaults |

**Design assumptions (consultant should validate with client):**
- **One deployment per client = one CDK app with one set of stacks.** No cross-tenant data paths. If the client has multiple business units, use per-BU access groups in the filterable metadata (`access_group`) rather than per-BU indexes (simpler; S3 Vectors index creation is async).
- **Titan V2 symmetric embedder** means the same model embeds documents and questions — no separate query embedder. If you need asymmetric (Cohere Rerank or similar), it's a post-retrieval reranking layer, not a replacement for Titan.
- **Prompt caching on system+chunks** works because chunks are stable within a conversation turn — but stops working if the retrieved chunks change mid-conversation. Cache only the system prompt block for multi-turn if chunks vary.
- **S3 Vectors targets less-frequent queries** (sub-second infrequent, ~100ms frequent). If sustained QPS > 100 rpm per chatbot instance, add `EXPECTED_QPS=high` handling: schedule a snapshot export from S3 Vectors into OpenSearch Serverless for hot path.

---

## 8. Changelog

| Version | Date | Change |
|---|---|---|
| 1.0 | 2026-04-22 | Initial kit. Validated S3 Vectors facts against AWS user guide + CFN reference + boto3 docs. Ships alongside 2 new v2.0 partials (`DATA_S3_VECTORS`, `PATTERN_DOC_INGESTION_RAG`) in F369_CICD_Template. Five decision branches (`VECTOR_STORE_MODE`, `DOCUMENT_PARSER`, `CHUNKING_STRATEGY`, `AGENT_ARCHITECTURE`, `CONVERSATION_MEMORY`) plus `TARGET_LANGUAGE` + `EXPECTED_QPS` set at engagement kickoff. Default stack: `custom_s3_vectors` + `textract` + `fixed` chunking + `plain_bedrock_lambda` + `ddb_session` — matches the user's explicit Titan + S3 Vectors ask with simplest production-viable shape. |
