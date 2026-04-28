<!-- Template Version: 2.0 | F369 Wave 15 (composite) | Composes: BEDROCK_KNOWLEDGE_BASES + DATA_OPENSEARCH_SERVERLESS + LAYER_API + SERVERLESS_LAMBDA_POWERTOOLS + LLMOPS_BEDROCK -->

# Template 35 — Advanced RAG with Bedrock Knowledge Bases (hierarchical chunking · hybrid + reranking · multi-tenant · Guardrails · custom UI)

## Purpose

Build a **production-grade custom RAG system** in 4-6 days using Bedrock Knowledge Bases as the retrieval engine. Output: KB with hierarchical chunking, hybrid search, reranking, multi-tenant filtering, Bedrock Guardrails, custom REST API, and React UI integration.

This is the **canonical "deeper RAG control" engagement** — for cases where Q Business UX doesn't fit (custom branding, custom flows, embedded SaaS, fine-grained tenant isolation, specialized domain).

---

## Role Definition

You are an expert AWS RAG architect with deep expertise in:
- Bedrock Knowledge Bases (chunking strategies, vector store options, embedding models)
- Hybrid search (BM25 + vector) tuning + reranking with Cohere/AWS rerank
- Multi-tenant patterns: shared KB with metadata filter vs per-tenant KBs
- Bedrock Guardrails for PII/content filtering
- Custom Lambda chunking for complex source data
- Citation rendering + grounding verification
- Lambda + API Gateway as RAG inference endpoint
- Cost optimization (chunking model + embedding + retrieval LLM)

Generate complete, production-deployable code. No TODOs.

---

## Context and Inputs

```
PROJECT_NAME:                [REQUIRED]
AWS_REGION:                  [REQUIRED]
ENV:                         [REQUIRED]

# --- KB CONFIGURATION ---
SOURCE_TYPE:                 [REQUIRED — s3 | confluence | salesforce | sharepoint | web_crawler]
SOURCE_LOCATION:             [REQUIRED — bucket ARN OR URL]
DOCUMENT_TYPES:              [REQUIRED — pdf,docx,html,md,txt]

CHUNKING_STRATEGY:           [REQUIRED — default | fixed | hierarchical | semantic | none | custom_lambda]
EMBEDDING_MODEL:             [titan-v2-1024 default — also: cohere-embed-english-v3]
EMBEDDING_DIMENSIONS:        [1024 default — 256/512/1024 for Titan v2]

VECTOR_STORE:                [oss_serverless default — also: aurora_pgvector | pinecone | redis]

# --- MULTI-TENANT ---
MULTI_TENANT:                [true/false]
TENANT_ID_FIELD:             [tenant_id default if multi-tenant]
ISOLATION_LEVEL:             [filter default — also: per_kb (regulated)]

# --- RETRIEVAL ---
USE_HYBRID_SEARCH:           [true default for keyword-heavy domain]
USE_RERANKING:               [true default for prod]
RERANK_MODEL:                [cohere-rerank-v3-5 default]
NUM_RESULTS_RETRIEVAL:       [20 default — over-retrieve for rerank]
NUM_RESULTS_FINAL:           [5 default — top-N after rerank]

# --- GENERATION ---
GENERATION_MODEL:            [REQUIRED — claude-haiku-4-5 (cheap) | claude-sonnet-4-6 (balanced) | claude-opus-4-7 (best)]
TEMPERATURE:                 [0.1 default — low for factual]
MAX_TOKENS:                  [1024 default]

# --- GUARDRAILS ---
ENABLE_GUARDRAILS:           [true required for prod]
PII_HANDLING:                [anonymize default — also: block]

# --- API ---
API_DOMAIN:                  [REQUIRED — e.g. rag.example.com]
COGNITO_USER_POOL_ID:        [REQUIRED for auth]

# --- COMPLIANCE ---
KMS_KEY_ARN:                 [REQUIRED]
LOG_RETENTION_DAYS:          [365 for regulated]

# --- OBSERVABILITY ---
SNS_ALARM_TOPIC_ARN:         [REQUIRED]
ENABLE_GROUNDING_EVAL:       [true default — track citation accuracy]
```

---

## Partial Library (Claude MUST load)

| Partial | Why |
|---|---|
| `BEDROCK_KNOWLEDGE_BASES` | KB + chunking + vector stores + hybrid + multi-tenant + Guardrails |
| `DATA_OPENSEARCH_SERVERLESS` | Vector store backend |
| `LAYER_API` | API Gateway HTTP API + custom domain |
| `SERVERLESS_HTTP_API_COGNITO` | Cognito JWT for inbound auth |
| `SERVERLESS_LAMBDA_POWERTOOLS` | Inference Lambda with logger/tracer/metrics/idempotency |
| `LLMOPS_BEDROCK` | Bedrock InvokeModel patterns |
| `LAYER_FRONTEND` | React UI integration |
| `LAYER_OBSERVABILITY` | Dashboards + alarms |

---

## Architecture

```
   React SPA (custom UI)
        │
        │ Cognito JWT
        ▼
   ┌────────────────────────────────────────────────────────┐
   │ API Gateway HTTP API (rag.example.com)                  │
   │   - JWT authorizer (Cognito)                              │
   │   - WAF + rate limit                                       │
   │   - Custom domain                                            │
   └────────────────────────────┬───────────────────────────┘
                                │
                                ▼
   ┌────────────────────────────────────────────────────────┐
   │ Inference Lambda (Python 3.12, Powertools)             │
   │   1. Validate JWT, extract tenant_id                     │
   │   2. Call Retrieve API:                                   │
   │        - hybrid search                                     │
   │        - filter by tenant_id (multi-tenant)                │
   │        - rerank top-20 → top-5                              │
   │   3. Construct prompt (custom template)                    │
   │   4. Call RetrieveAndGenerate OR direct InvokeModel       │
   │   5. Apply Bedrock Guardrails                              │
   │   6. Format response + citations                            │
   │   7. Log to CW + emit metrics (latency, cost, citations)    │
   │   Returns: {answer, citations, confidence}                  │
   └────────────────────────────┬───────────────────────────┘
                                │
                                ▼
   ┌────────────────────────────────────────────────────────┐
   │ Bedrock Knowledge Base                                  │
   │   Chunking: hierarchical (parent 1500 / child 300)       │
   │   Embedding: Titan v2 1024                                │
   │   Vector store: OS Serverless VECTORSEARCH                │
   │   Custom Lambda parser (PDFs with tables)                 │
   │   Metadata: tenant_id, year, department, security         │
   │   Hybrid search ON                                          │
   │   Reranking: Cohere rerank v3.5                              │
   │   Guardrails: PII anonymize, content filter, prompt attack │
   └────────────────────────────────────────────────────────┘
                                ▲
                                │ ingestion
                                │
   Source S3 / SharePoint / Confluence / etc.
```

---

## Day-by-day execution (4-6 day deploy, 1 dev)

### Day 1 — KB foundation + first ingestion
- KMS multi-region CMK
- OS Serverless VECTORSEARCH collection (CMK, multi-AZ standby)
- IAM role for Bedrock KB
- Bedrock KB with hierarchical chunking + Titan v2 1024
- S3 source bucket with `.metadata.json` sidecar files (tenant_id, year, department)
- Initial ingestion job — ~1-2 hours for 10K docs
- Test Retrieve API via boto3 — verify chunks returned with metadata
- **Deliverable:** End of Day 1: KB ACTIVE; sample query returns relevant chunks.

### Day 2 — Hybrid search + reranking + multi-tenant filter
- Tune retrieval: enable hybrid search; over-retrieve N=20 with rerank to N=5
- Verify multi-tenant isolation: tenant A's query doesn't return tenant B's docs
- Add Bedrock Guardrails (PII anonymize + content filter + prompt attack)
- Test edge cases: empty filter, missing metadata, large docs
- **Deliverable:** End of Day 2: tenant isolation verified; precision improved 15-30% with reranking.

### Day 3 — Custom Lambda chunker (if needed)
- For complex sources (scientific papers, legal docs, code):
  - Custom Lambda chunker — extract per-section chunks + metadata
  - Custom transformation in data source config
  - Re-ingest sample subset → verify chunk quality
- For PDFs with tables/images:
  - Configure parsing with `BEDROCK_FOUNDATION_MODEL` strategy + Claude Haiku (vision)
- **Deliverable:** End of Day 3: chunker handles 95% of source documents correctly.

### Day 4 — Inference API (Lambda + API Gateway)
- Inference Lambda with Powertools + Pydantic event parsing
- Custom prompt template optimized for citation format
- Cognito JWT validation + tenant_id extraction from claims
- Bedrock RetrieveAndGenerate API call
- Custom domain via API Gateway + ACM
- WAF v2 with rate limiting
- **Deliverable:** End of Day 4: `curl -H 'Authorization: Bearer <jwt>' https://rag.example.com/query` returns grounded answer + citations.

### Day 5 — React UI integration
- React SPA scaffolded with Vite + Cognito Hosted UI
- Custom hook `useRagQuery(question)` returning `{answer, citations, confidence, loading}`
- UI: chat-style interface + cited sources with click-to-source-doc
- Streaming response (use RetrieveAndGenerateStream API)
- Embed-ready (postMessage + CSP for parent frame)
- **Deliverable:** End of Day 5: UI live; sample queries demo end-to-end with citations clickable.

### Day 6 — Grounding eval + observability + tests
- (If `ENABLE_GROUNDING_EVAL`) automated grounding test:
  - Set of 100 known Q&A pairs from source docs
  - Daily Lambda runs queries → checks if expected source cited
  - Track grounding accuracy as CW metric
- CloudWatch dashboard: query volume, latency p99, citation count avg, grounding accuracy
- 6 alarms: ingestion failures, query errors, latency p99 > 5s, grounding < 90%, Guardrail blocks > 5%, cost spike
- Pytest suite covering all validation criteria
- **Deliverable:** End of Day 6: grounding accuracy ≥ 90% over test set; all alarms green.

---

## Validation criteria

- [ ] **KB ACTIVE** + KMS-encrypted
- [ ] **Vector store collection ACTIVE** with multi-AZ standby
- [ ] **Latest ingestion job COMPLETE** with `numberOfDocumentsFailed: 0`
- [ ] **Hybrid search active** — keyword queries (e.g., "ORD-12345") return correct doc
- [ ] **Reranking improves precision** — top-5 after rerank > top-5 before (eval metric)
- [ ] **Multi-tenant isolation** — verified by 2-tenant test (A cannot see B's docs)
- [ ] **Bedrock Guardrails active** — PII (emails, SSNs) anonymized in output
- [ ] **Custom Lambda chunker** (if used) handles 95%+ source docs
- [ ] **API Gateway custom domain** reachable + JWT-protected
- [ ] **WAF rate limit** enforced (test by burst)
- [ ] **React UI** can query + render citations with links
- [ ] **Streaming response** works (token-by-token)
- [ ] **Grounding accuracy ≥ 90%** on 100-sample eval set
- [ ] **Latency p99 < 5s** for typical query (5-chunk RAG)
- [ ] **CW alarms** all OK baseline; firing on injected failures
- [ ] **Cost per query** within budget (typical: $0.01-0.05 per query for Sonnet RAG)

---

## Common gotchas (claude must address proactively)

- **Hierarchical chunking** is best default. For < 500-token docs (FAQ entries), use fixed chunking.
- **Hybrid search OFF by default** in Retrieve API. Always pass `overrideSearchType: HYBRID` for keyword precision.
- **Reranking model latency** adds 200-500ms — measure end-to-end. If p99 too high, drop reranking or use AWS rerank (faster).
- **Multi-tenant single-KB filter** is cheaper than per-tenant KBs but requires correct metadata setup. Get sidecar JSON format right (`{"metadataAttributes": {"tenant_id": "..."}}`).
- **Bedrock Guardrails latency overhead 200-500ms** — bake into latency budget.
- **Custom Lambda chunker timeout 15 min max** — for huge docs, chunk upstream.
- **Citation render UX** — show source title + clickable link + confidence score. Bad citation UX = users distrust the system.
- **Streaming with RetrieveAndGenerateStream** — citations come at end (after stream); plan UX accordingly.
- **Cost transparency** — track per-query: chunking + embedding (one-time) + retrieval + generation. Sonnet RAG ≈ $0.02/query; Haiku ≈ $0.005.
- **Grounding regression** — re-run eval set after every ingestion or model swap. Without this, drift is invisible.
- **Tenant onboarding** — adding a new tenant = adding metadata. Don't recreate KB. But test isolation per new tenant.
- **Embedding dimensions vs cost** — Titan v2 256-dim is 4× cheaper to store than 1024-dim. For < 100K docs, savings small. For > 1M docs, material.

---

## Output artifacts

1. **CDK stacks**:
   - `KbStack` — KB + vector store + IAM + KMS
   - `IngestionStack` — data source + custom Lambda chunker (if needed)
   - `InferenceStack` — Lambda + API Gateway + WAF + Cognito + custom domain
   - `GuardrailStack` — Bedrock Guardrails config
   - `EvalStack` — automated grounding eval Lambda + CW metrics

2. **Custom Lambda chunker** (`src/kb_chunker/`) — section-aware chunking + metadata extraction

3. **Inference Lambda** (`src/rag_inference/`) — Powertools + custom prompt + RetrieveAndGenerate

4. **Bedrock Guardrails JSON** — content filter + PII + word policy

5. **React UI** — Vite + Cognito + chat component + citation renderer + streaming hook

6. **Grounding eval suite** — 100 Q&A pairs in YAML; eval Lambda; CW metric publisher

7. **Pytest validation suite** covering all criteria

8. **Cost dashboard** (CW) — per-query cost breakdown

9. **6 alarms YAML** — wired to SNS

10. **README** — sample queries + curl + UI URL + cost guide

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 2.0 | 2026-04-27 | Initial composite template. Bedrock KB + hybrid + reranking + multi-tenant + Guardrails + custom UI. 4-6 day deploy. Wave 15. |
