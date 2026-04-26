<!-- Kit Version: 1.0 | Template library: F369_LLM_TEMPLATES v1.x | Partials: F369_CICD_Template partials v2.0 -->

# Kit — AI-Native Lakehouse (Vector-Enhanced Iceberg + Enterprise Chat over Structured + Unstructured Data)

**Client ask:** "Give our business users a single chat window that can answer any question over our data — structured tables in our warehouse, contracts and policies in our doc store, engineering diagrams in our image corpus — in English, with citations, governed by our existing role-based access." The *Bloomberg-terminal-for-our-enterprise-data* experience, built on AWS native services, on their data, in 2 weeks.
**Engagement:** 2 weeks · 2 developers · governed-by-default (Lake Formation LF-TBAC; HIPAA / PCI / SOC 2 variants via params).
**Deliverable:** CDK repo (Python or TypeScript) + React/Next.js chat UI + Iceberg lakehouse + 3-level catalog embedding index + multimodal image index + text-to-SQL agent + doc RAG + enterprise chat router + QuickSight Q dashboard.

**Scope rationale:** Combines two common asks — "AI-Ready Lakehouse" (text-to-SQL + QuickSight Q over governed data) and "Vector-Native Lake" (semantic discovery across structured + unstructured) — into one engagement. Vectors are the INDEX, SQL + RAG + multimodal are the retrieval. See companion design doc [`kits/_design/ai-native-lakehouse.md`](_design/ai-native-lakehouse.md) for business case, UX design, and 3 business-flow examples (finance analytics, operations Q&A, compliance audit).

---

## 1. Kit-wide parameters (fill ONCE; carried to every template + Claude call)

```
# --- Identity -----------------------------------------------------------
PROJECT_NAME:               lakehouse-{client_slug}
AWS_REGION:                 us-east-1  (or any region with S3 Tables + S3 Vectors + Athena v3
                                        + QuickSight Q + AgentCore Memory/Identity GA)
AWS_ACCOUNT_ID:             [12-digit]
ENV:                        dev | stage | prod
CLIENT_SLUG:                acme
TARGET_LANGUAGE:            python | typescript

# --- DOMAIN PACK — drives schema seed + glossary + demo questions ------
DOMAIN_PACK:                finance_analytics       ← DEFAULT (most demo-friendly)
                            | operations_ops
                            | sales_revenue
                            | mixed_multi_domain    (requires DATAZONE_ENABLED=true)
                            | custom

# --- LAKEHOUSE STORAGE --------------------------------------------------
ICEBERG_BACKEND:            s3_tables        (DEFAULT — managed compaction; see
                                              partial DATA_ICEBERG_S3_TABLES)
                            | self_managed   (classic S3 + Glue ETL; see
                                              partial DATA_LAKEHOUSE_ICEBERG)
LAKE_ZONES:                 [raw, curated, consumer]     ← DEFAULT (3-tier)
                            | [raw, curated]              (2-tier, simpler)

# --- CATALOG EMBEDDING INDEX --------------------------------------------
CATALOG_INDEX_LEVELS:       3   (db + table + column)    ← DEFAULT
                            | 2 (table + column only — use for < 5 databases)
EMBEDDING_MODEL_ID:         amazon.titan-embed-text-v2:0
EMBEDDING_DIM:              1024   (1024 for default; 256 only if catalog < 1000 tables)
CATALOG_REFRESH_MODE:       event_driven   (EB on glue:* events)   ← DEFAULT
                            | scheduled    (hourly SFN sweep — fallback for older regions)

# --- MULTIMODAL (OPTIONAL — flip false for pure structured/text) -------
MULTIMODAL_ENABLED:         true    ← DEFAULT (diagrams, engineering drawings, slides)
MULTIMODAL_EMBED_MODEL:     amazon.titan-embed-image-v1
MULTIMODAL_DIM:             1024
MULTIMODAL_SOURCES:         [images, pdfs, diagrams]

# --- AGENT / CHAT ROUTER -----------------------------------------------
SUPERVISOR_MODEL_ID:        us.anthropic.claude-sonnet-4-7-20260109-v1:0
TOOL_DECISION_MODEL_ID:     us.anthropic.claude-haiku-4-5-20251001-v1:0   (cheaper per-tool decisions)
MAX_TOOL_CALLS_PER_TURN:    8
TEXT_TO_SQL_RETRIES:        3
AGENT_SCAN_CUTOFF_GB:       1     (per-query Athena workgroup cutoff)
DAILY_SCAN_BUDGET_GB:       100   (per-caller daily cumulative cutoff)
SESSION_TOKEN_BUDGET:       200000  (per-session cap)
DAILY_TOKEN_BUDGET:         5000000 (per-caller-per-day)

# --- GOVERNANCE (LF-TBAC is MANDATORY) ---------------------------------
LF_TAGS:                    [domain, sensitivity, environment]   ← DEFAULT tag set
LF_DOMAIN_VALUES:           [finance, hr, legal, product]
LF_SENSITIVITY_VALUES:      [public, internal, confidential, pii]
LF_ENFORCE:                 strict    (CreateDatabase/TableDefaultPermissions=[])
                            | hybrid  (IAM-plus-LF; for legacy accounts)

# --- BI / VISUALISATION -------------------------------------------------
QUICKSIGHT_Q_ENABLED:       true     ← DEFAULT (requires QuickSight Enterprise +
                                                Q Generative Capabilities add-on)
QUICKSIGHT_REFRESH:         spice_hourly
QUICKSIGHT_EMBED_MODE:      registered_user   (Cognito-backed)
                            | anonymous       (customer-facing)
                            | console         (for analyst power-users)

# --- ZERO-ETL (OPTIONAL) ------------------------------------------------
ZERO_ETL_ENABLED:           false    ← DEFAULT OFF (use when client has live Aurora/DDB
                                                    that should appear in lake analytics)
ZERO_ETL_SOURCES:           []       (e.g. [aurora_prod, ddb_orders])
ZERO_ETL_TARGET:            redshift_serverless  (minimum 8 RPU)

# --- DATA MESH (OPTIONAL — for multi-domain engagements) ---------------
DATAZONE_ENABLED:           false   ← DEFAULT OFF — flip for mixed_multi_domain DOMAIN_PACK
IDENTITY_CENTER_INSTANCE:   arn:aws:sso:::instance/ssoins-xxxxxxxx  (REQUIRED if DATAZONE_ENABLED)

# --- FRONTEND CHAT UI ---------------------------------------------------
UI_FRAMEWORK:               nextjs   (DEFAULT — SSR + CloudFront + OAC)
                            | react_spa
AUTH:                       cognito_user_pool   (DEFAULT — SSO via SAML/OIDC federation)
STREAM_RESPONSES:           true    (WebSocket via API GW v2)
SHOW_CITATIONS:             true
CITATION_FORMAT:            inline   ([SQL-1] [DOC-1] [IMG-1] style)
ALLOWED_EMBED_DOMAINS:      [https://app.{client_slug}.com]

# --- COMPLIANCE / GOVERNANCE --------------------------------------------
COMPLIANCE_STANDARD:        SOC2   (DEFAULT)
                            | HIPAA | PCI_DSS | NONE
AUDIT_LOG_RETENTION_DAYS:   365    (2555 for HIPAA)
PII_REDACTION_ENABLED:      true   (strip PII from catalog-embedding source_text)

# --- COST CONTROLS ------------------------------------------------------
MAX_COST_PER_CHAT_TURN_USD: 0.50
MAX_COST_PER_DAY_USD:       500
SPICE_CAPACITY_GB:          100    (for QuickSight Enterprise)
```

---

## 2. Architecture

```
                         ┌───────────────────────────────────┐
                         │       Business user / analyst      │
                         └────────────────┬──────────────────┘
                                          │
                          chat UI         │  WebSocket (wss)
                          QuickSight Q    │  HTTPS (signed)
                                          ▼
                         ┌───────────────────────────────────┐
                         │  API Gateway WS + REST (Cognito)  │
                         └────────────────┬──────────────────┘
                                          │
                                          ▼
       ┌─────────────────────────────────────────────────────────────────┐
       │   ChatRouterSupervisor  (Strands Agent — Claude Sonnet 4.7)     │
       │                                                                 │
       │   Tools:                                                        │
       │   ├── text_to_sql        → TextToSqlFn (Lambda)                 │
       │   ├── semantic_discovery → DiscoveryFn (Lambda)                 │
       │   ├── doc_rag            → DocRagFn    (Lambda)                 │
       │   ├── multimodal_search  → MmQueryFn   (Lambda, if enabled)     │
       │   └── (optional) compute → Code Interpreter (AgentCore)         │
       │                                                                 │
       │   Memory:  AgentCore Memory (per-session recall)                │
       │   OBO:     AgentCore Identity (caller's LF perms propagate)     │
       └──┬──────────────┬──────────────┬──────────────┬─────────────────┘
          │              │              │              │
          ▼              ▼              ▼              ▼
    ┌─────────┐   ┌─────────────┐   ┌─────────┐   ┌───────────────┐
    │Text-to- │   │ Semantic    │   │ Doc     │   │ Multimodal    │
    │SQL Fn   │   │ Discovery   │   │ RAG Fn  │   │ Query Fn      │
    │         │   │ Fn          │   │         │   │               │
    │ 4-phase │   │ 3-pass      │   │ Chunk   │   │ Image + text  │
    │ pipeline│   │ retrieve +  │   │ retrieve│   │ (same 1024-d  │
    │ +preflt │   │ Haiku rerank│   │ + cite  │   │ cosine space) │
    └────┬────┘   └──────┬──────┘   └────┬────┘   └───────┬───────┘
         │               │               │                 │
         │               ▼               ▼                 ▼
         │      ┌────────────────────────────┐   ┌──────────────────────┐
         │      │ Catalog Embedding Indexes  │   │ Multimodal Index     │
         │      │ (S3 Vectors × 3)           │   │ (S3 Vectors × 2)     │
         │      │   db / table / column      │   │   images / text      │
         │      │                            │   │                      │
         │      │ Refresh: EB on glue:*      │   │ Refresh: EB on s3:   │
         │      │ Rerank:  Haiku 4.5         │   │ upload               │
         │      └──────────┬─────────────────┘   └────────┬─────────────┘
         │                 │                              │
         ▼                 ▼                              ▼
    ┌─────────────────────────────────────────────────────────────────┐
    │  Athena Workgroups                                               │
    │    - lakehouse-analyst-{env}   (10 GB cutoff, human)             │
    │    - lakehouse-agent-{env}      (1 GB cutoff, agent)             │
    │    - discovery-sample-{env}     (100 MB cutoff, discovery)       │
    │    - QuickSight Q default WG                                     │
    └──────────────────────────┬───────────────────────────────────────┘
                               │
                               ▼
    ┌─────────────────────────────────────────────────────────────────┐
    │  Lake Formation (LF-TBAC)                                        │
    │    LF-Tags: domain × sensitivity × environment                   │
    │    Gen-3 CfnPrincipalPermissions on role × tag expression        │
    │    CfnDataCellsFilter for row/column masking                     │
    └──────────────────────────┬───────────────────────────────────────┘
                               │
                               ▼
    ┌─────────────────────────────────────────────────────────────────┐
    │  Data Sources                                                    │
    │    - S3 Tables (Iceberg, managed) ─── via Glue auto-federation   │
    │    - Self-managed Iceberg on S3   ─── via Glue Catalog           │
    │    - Doc RAG corpus                ─── S3 Vectors + PATTERN_DOC… │
    │    - Multimodal corpus             ─── S3 + Titan Multimodal     │
    │    - (optional) Zero-ETL: Aurora/DDB → Redshift Serverless       │
    │    - (optional) DataZone domain: fin_project + hr_project + ...  │
    └─────────────────────────────────────────────────────────────────┘

    BI:    QuickSight Enterprise + Q (Topic = "Finance Analytics")
    Obs:   CloudWatch dashboards + alarms (scan GB, lag, Lambda errors)
    Gov:   CloudTrail → Object-Lock audit bucket; Config rules
```

---

## 3. Execution plan (2 weeks, 2 developers — mirrors Waves 1–5 of the partial build)

### Week 1 — Foundation + vector layer + text-to-SQL

| Day | Template (paste to Claude with kit params) | Partials Claude MUST load |
|---|---|---|
| **1** | [`iac/02_cdk_ml_llm_infrastructure`](../iac/02_cdk_ml_llm_infrastructure.md) — scaffold CDK app with stacks: `NetworkStack`, `SecurityStack`, `TableBucketStack` (S3 Tables), `GovernanceStack` (LF-TBAC), `CatalogStack` (Glue), `AthenaStack`, `CatalogEmbeddingStack`, `TextToSqlStack`, `DiscoveryStack`, `RagStack`, `MultimodalStack`, `ChatRouterStack`, `QuickSightStack`, `FrontendStack`, `ObservabilityStack`, `ComplianceStack` | `LAYER_NETWORKING`, `LAYER_SECURITY`, `LAYER_DATA`, `LAYER_BACKEND_LAMBDA`, `LAYER_OBSERVABILITY` |
| **2** | Lakehouse storage — instruct Claude "Generate `TableBucketStack` per `DATA_ICEBERG_S3_TABLES` partial. Namespace = `lakehouse_{env}`, Iceberg tables seeded per DOMAIN_PACK ({fact_revenue, dim_customer, stg_event} for finance). CMK per-stack, KMS ARN published via SSM." Then `CatalogStack` per `DATA_GLUE_CATALOG` — database definition + column comments + Glue crawler for `stg_events/` + DQ ruleset. | `DATA_ICEBERG_S3_TABLES`, `DATA_GLUE_CATALOG` |
| **3** | Governance — instruct Claude "Generate `GovernanceStack` per `DATA_LAKE_FORMATION` partial. `CfnDataLakeSettings` with CI deploy role as admin + CreateDatabase/TableDefaultPermissions=[] for LF_ENFORCE=strict. LF-Tags = [domain, sensitivity, environment]. Register the S3 Tables bucket. Tag fact_revenue → domain=finance, sensitivity=internal." | `DATA_LAKE_FORMATION` |
| **4** | Query engine — instruct Claude "Generate `AthenaStack` per `DATA_ATHENA` partial. 3 workgroups: `lakehouse-analyst-{env}` (10 GB cutoff, human), `lakehouse-agent-{env}` (1 GB cutoff, agent), `discovery-sample-{env}` (100 MB cutoff). Engine v3, enforce_workgroup_configuration=true, SSE-KMS on result bucket, 30-day result lifecycle." | `DATA_ATHENA` |
| **5** | Catalog embedding (Wave 2 partial #1) — instruct Claude "Generate `CatalogEmbeddingStack` per `PATTERN_CATALOG_EMBEDDINGS` partial. 3 S3 Vectors indexes (db + table + column). Fingerprint-diff refresh Lambda wired to EB rules on `Glue Data Catalog Database/Table State Change`. Bulk-reindex Step Function. Grant refresh Lambda `glue:*`, `lakeformation:GetResourceLFTags`, `bedrock:InvokeModel` on Titan v2." | `PATTERN_CATALOG_EMBEDDINGS`, `DATA_S3_VECTORS` |

### Week 2 — Agents + BI + frontend + compliance + go-live

| Day | Template | Partials |
|---|---|---|
| **6** | Text-to-SQL + Discovery (Wave 3 partials #1 + #2) — instruct Claude "Generate `TextToSqlStack` per `PATTERN_TEXT_TO_SQL` partial (four-phase pipeline: discover → generate → preflight → execute; Claude Sonnet 4.7 SQL generator + Haiku 4.5 parameter extractor; DDB daily scan budget; 3 retry loop). Generate `DiscoveryStack` per `PATTERN_SEMANTIC_DATA_DISCOVERY` partial (identity-from-JWT; 3-pass retrieve + Haiku rerank; DDB 10-min cache; API GW + Cognito)." | `PATTERN_TEXT_TO_SQL`, `PATTERN_SEMANTIC_DATA_DISCOVERY` |
| **7** | Doc RAG + Multimodal (if MULTIMODAL_ENABLED) — instruct Claude "Generate `RagStack` per `PATTERN_DOC_INGESTION_RAG` partial (S3 upload → Textract → chunk → embed → S3 Vectors; query-side Lambda returns top-k chunks with citations). If MULTIMODAL_ENABLED=true, generate `MultimodalStack` per `PATTERN_MULTIMODAL_EMBEDDINGS` partial (PDF page-render via PyMuPDF + text via Textract; EXIF strip; Rekognition face blur; thumbnail preview bucket)." | `PATTERN_DOC_INGESTION_RAG`, `PATTERN_MULTIMODAL_EMBEDDINGS`, `DATA_S3_VECTORS` |
| **8** | Chat router (Wave 3 partial #3 — the hero) — instruct Claude "Generate `ChatRouterStack` per `PATTERN_ENTERPRISE_CHAT_ROUTER` partial. API GW WebSocket API with Cognito authoriser on $connect. Supervisor = Strands Agent (Claude Sonnet 4.7) with 4 tools (text_to_sql, semantic_discovery, doc_rag, multimodal_search). AgentCore Memory for session recall; AgentCore Identity for OBO — CALLER's LF perms propagate to tool Lambdas (not the router's). Inline citations [SQL-N] [DOC-N] [IMG-N] + Sources block." Use `mlops/20_strands_agent_lambda_deployment.md` + `mlops/21_strands_multi_agent_patterns.md` as framing refs. | `PATTERN_ENTERPRISE_CHAT_ROUTER`, `AGENTCORE_MEMORY`, `AGENTCORE_GATEWAY`, `STRANDS_AGENT_CORE`, `STRANDS_MULTI_AGENT` |
| **9** | BI + frontend — instruct Claude "Generate `QuickSightStack` per `MLOPS_QUICKSIGHT_Q` partial (CfnDataSource for Athena, CfnDataSet with RLS for fact_revenue, CfnTopic with column synonyms + semantic types, CfnDashboard). Generate `FrontendStack` per `LAYER_FRONTEND` — Next.js build + S3 + CloudFront + OAC. Embed QuickSight Q + chat router in one UI." Use [`mlops/25_react_portal_cloudfront`](../mlops/25_react_portal_cloudfront.md) as the frontend thin router. | `MLOPS_QUICKSIGHT_Q`, `LAYER_FRONTEND`, `LAYER_API` |
| **10** | Compliance + observability + go-live — instruct Claude "Generate `ComplianceStack` per `COMPLIANCE_HIPAA_PCIDSS` with COMPLIANCE_STANDARD={kit}. Object-Lock audit bucket, CloudTrail, Config rules. Generate `ObservabilityStack` per `LAYER_OBSERVABILITY` — dashboards for scan bytes, Lambda errors, WG queries, chat-turn cost, rerank latency. Optional: ZeroETL integration per `DATA_ZERO_ETL` if ZERO_ETL_ENABLED=true; DataZone domain per `DATA_DATAZONE` if DATAZONE_ENABLED=true." Use [`enterprise/06_compliance_blueprint`](../enterprise/06_compliance_blueprint.md) as the compliance thin-router. | `COMPLIANCE_HIPAA_PCIDSS`, `LAYER_OBSERVABILITY`, (optional) `DATA_ZERO_ETL`, (optional) `DATA_DATAZONE` |

---

## 4. Complete partial reference (load each into Claude along with the template)

> **URL template:** `https://github.com/surendersara1/F369_CICD_Template/blob/main/prompt_templates/partials/<NAME>.md`

**Core platform (ALWAYS load):**
- `LAYER_BACKEND_LAMBDA` — 5 non-negotiables (identity-side grants, Path(__file__) anchoring, CfnRule cross-stack EB, same-stack bucket+OAC, KMS ARNs as strings)
- `LAYER_NETWORKING` — VPC + PrivateLink + interface endpoints for S3/S3vectors/Bedrock/Athena
- `LAYER_SECURITY` — KMS + IAM + permission boundaries
- `LAYER_DATA` — DDB patterns for session/budget/cache tables
- `LAYER_API` — API GW REST + WebSocket v2
- `LAYER_FRONTEND` — React/Next + CloudFront + OAC
- `LAYER_OBSERVABILITY` — CloudWatch dashboards + alarms + X-Ray
- `EVENT_DRIVEN_FAN_IN_AGGREGATOR` — for catalog-refresh fan-in + DQ event aggregation

**Wave 1 — Lakehouse foundation:**
- `DATA_ICEBERG_S3_TABLES` — fully managed Iceberg on S3 Tables (the NEW service, distinct from self-managed)
- `DATA_LAKEHOUSE_ICEBERG` — self-managed Iceberg option (swap-in if S3 Tables not available in region)
- `DATA_LAKE_FORMATION` — Gen-3 LF-TBAC, cross-account via RAM, data cells filters
- `DATA_GLUE_CATALOG` — database + tables + crawlers + DQ rulesets; column comments are the AI substrate
- `DATA_ATHENA` — workgroups with scan cutoffs, Iceberg DML, EXPLAIN preflight, `USING FUNCTION invoke_model` Bedrock-from-SQL

**Wave 2 — Vector layer:**
- `DATA_S3_VECTORS` — underlying vector storage (used by catalog embeddings + multimodal + doc RAG)
- `PATTERN_CATALOG_EMBEDDINGS` — 3-level semantic index over Glue catalog with LF-Tag filter pushdown
- `PATTERN_MULTIMODAL_EMBEDDINGS` — Titan Multimodal G1 for images + PDF pages; cross-modal same-space search

**Wave 3 — Agent/query patterns:**
- `PATTERN_TEXT_TO_SQL` — four-phase pipeline (discover → generate → preflight → execute) with error-correction retry + DDB daily budget + lex veto
- `PATTERN_SEMANTIC_DATA_DISCOVERY` — thin "find my data" API with identity-from-JWT + Haiku rerank + sample-values
- `PATTERN_ENTERPRISE_CHAT_ROUTER` — Strands supervisor + 4 tools + AgentCore Memory + OBO identity propagation + inline citations
- `PATTERN_DOC_INGESTION_RAG` — the doc-RAG tool's backing service (document chunking + embed + query)
- `AGENTCORE_MEMORY` — session memory for chat router
- `AGENTCORE_GATEWAY` — alt tool-hosting framework (MCP)
- `AGENTCORE_CODE_INTERPRETER` — optional `compute` tool for numeric analysis
- `AGENTCORE_BROWSER_TOOL` — optional `web_search` tool for external enrichment

**Wave 4 — BI + bolts:**
- `MLOPS_QUICKSIGHT_Q` — Q in QuickSight (Topics + RLS + embed SDK)
- `DATA_ZERO_ETL` *(if ZERO_ETL_ENABLED=true)* — Aurora/DDB → Redshift Serverless managed CDC
- `DATA_DATAZONE` *(if DATAZONE_ENABLED=true)* — domain/project/data-product mesh

**Governance + compliance:**
- `COMPLIANCE_HIPAA_PCIDSS` — Object-Lock audit bucket + Backup Vault Lock + Config rules
- `SECURITY_WAF_SHIELD_MACIE` — WAF + Macie on raw doc bucket

**Strands Agents (supporting):**
- `STRANDS_AGENT_CORE` — supervisor + tool-library pattern
- `STRANDS_TOOLS` — `@tool` wrapping for tool Lambdas
- `STRANDS_MULTI_AGENT` — supervisor + sub-agent fan-out (used if text-to-SQL splits into a dedicated specialist)
- `STRANDS_FRONTEND` — WebSocket streaming callback handler

---

## 5. Architecture non-negotiables (repeat in every Claude prompt)

> **Rules from `LAYER_BACKEND_LAMBDA §4.1` — all generated code MUST pass these, regardless of `TARGET_LANGUAGE`:**
>
> 1. Lambda / container asset paths use `Path(__file__).resolve().parents[N] / "..."` (Python) or `path.join(__dirname, ...)` (TypeScript). Never CWD-relative.
> 2. Cross-stack resource access is **identity-side only** — attach `PolicyStatement` to the consumer's role. Never `bucket.grantReadWrite(crossStackRole)` or similar.
> 3. Cross-stack EventBridge → Lambda uses L1 `events.CfnRule` with a static-ARN target.
> 4. Bucket + CloudFront OAC live in the **same stack** — never split.
> 5. KMS ARN references across stacks are **strings** via SSM, not imported `kms.Key` L2 constructs.
>
> **Kit-specific additions (critical):**
>
> 6. **LF-TBAC is MANDATORY.** Every table tagged with `domain` + `sensitivity` + `environment`. No IAM-only grants on governed resources. LF_ENFORCE=strict mode means `CreateDatabase/TableDefaultPermissions=[]`.
> 7. **Identity propagation via AgentCore Identity (OBO).** The chat router's execution role does NOT get extended grants; tool Lambdas invoke with the CALLER's OBO token. If AgentCore Identity is unavailable in the region, degrade to passing `caller_id` + re-authenticating in each tool Lambda (LF enforcement at the tool Lambda's role must be tight — do NOT broad-grant).
> 8. **Catalog embeddings store source_text as non-filterable** metadata to enable one-hop retrieval without a filter-table cost. PII columns NEVER get sample-value signatures; their name + comment only are indexable.
> 9. **Scan cost is capped at 3 levels:** per-query via workgroup cutoff (1 GB agent, 10 GB analyst, 100 MB discovery); per-caller-per-day via DDB budget table; per-account via CloudWatch alarm on `DataScannedInBytes`.
> 10. **Text-to-SQL preflight is `EXPLAIN (FORMAT JSON)`** — parses SQL for free, extracts touched tables, validates against allowlist from discovery, feeds errors back to the LLM for up to 3 retries.

---

## 6. Deliverables checklist (end of Week 2)

- [ ] CDK repo (`TARGET_LANGUAGE`) with ~15 stacks (see Day 1 list)
- [ ] Iceberg lakehouse on S3 Tables with 3+ tables seeded per DOMAIN_PACK
- [ ] Glue catalog with column comments + crawlers + DQ rulesets
- [ ] Lake Formation LF-TBAC with 3 tags × 4 domains + 3-4 sample role grants
- [ ] Athena workgroups: agent (1 GB), analyst (10 GB), discovery (100 MB) — all engine v3 + KMS result bucket
- [ ] Catalog embedding indexes (3 × S3 Vectors) populated via bulk-reindex SFN
- [ ] Text-to-SQL Lambda: discover → generate → preflight → execute, with DDB daily budget
- [ ] Semantic discovery API (Cognito-auth) with 3-pass retrieve + Haiku rerank
- [ ] Doc RAG pipeline with sample corpus (20+ PDFs, governance docs)
- [ ] Multimodal index with sample image corpus (20+ diagrams/PDFs) — *if MULTIMODAL_ENABLED*
- [ ] Chat router Lambda (Strands supervisor + 4 tools + AgentCore Memory + OBO)
- [ ] WebSocket API with Cognito authoriser on $connect
- [ ] React/Next.js chat UI with inline citations + QuickSight Q embedded panel
- [ ] QuickSight dashboard (finance-overview-style) + Q Topic with column synonyms
- [ ] Compliance: Object-Lock audit bucket, CloudTrail, Config rules per COMPLIANCE_STANDARD
- [ ] WAF on CloudFront + API GW
- [ ] Observability dashboards: scan bytes, Lambda errors, turn cost, tool latency, chat turns/day
- [ ] Runbook with 10 golden-set questions per DOMAIN_PACK (expected tool-routing + citations) — nightly regression test
- [ ] Pilot test with 2-3 users over 48 hours, feedback captured

---

## 7. Golden-set questions (for nightly regression test — pick 3 per DOMAIN_PACK)

**finance_analytics (DEFAULT):**
- "Total revenue by region for the last quarter" → `text_to_sql` → [SQL-1]
- "Top 10 customers by renewal value" → `text_to_sql` → [SQL-1]
- "What does our master agreement with Globex say about SLA credits" → `doc_rag` → [DOC-1]
- "Show me the revenue dashboard" → QuickSight Q embed (delegated)
- "Revenue for customers whose contracts mention renewal bonuses" → `text_to_sql` + `doc_rag` → [SQL-1] [DOC-1]
- "Where is our customer billing data" → `semantic_discovery` → tables list

**operations_ops:**
- "Show me a wiring diagram for the XJ-550 pump" → `multimodal_search` → [IMG-1]
- "Equipment failures in the EMEA region last week with repair notes" → `text_to_sql` + `doc_rag` → [SQL-1] [DOC-1]
- "What's our MTTR trend by equipment type" → `text_to_sql` → [SQL-1]

**sales_revenue:**
- "Pipeline coverage by account exec for Q3" → `text_to_sql` → [SQL-1]
- "Show me deal playbook for Fortune 500 enterprise accounts" → `doc_rag` → [DOC-1]
- "Who are our top 20 at-risk customers based on renewal sentiment" → `text_to_sql` + `doc_rag` → [SQL-1] [DOC-1]

---

## 8. Known partial gaps — 0 new partials needed

This kit ships with **100% partial coverage** — the 12 new partials built in Waves 1-4 (+ `DATA_S3_VECTORS` + `PATTERN_DOC_INGESTION_RAG` from prior kits) fully cover the engagement. No new partials to write during the first client engagement.

**Known template gaps surfaced by prior kits (now CLOSED in Wave 5, 2026-04-26):**

The following 8 partials were authored in Wave 5 (2026-04-26) and are available as **opt-in extensions** to this kit. Add them when the client engagement requires migration / multi-DB / cross-region DR / SaaS ingest patterns:

| Extension | When to add | New partial |
|---|---|---|
| Lift-and-shift Oracle / SQL Server / on-prem Postgres into the lakehouse | Client has legacy DB to retire | [`DATA_DMS_REPLICATION`](../../F369_CICD_Template/prompt_templates/partials/DATA_DMS_REPLICATION.md) |
| Non-Aurora HA (RDS Multi-AZ DB cluster, 1 writer + 2 readable standbys) | Client wants RDS not Aurora; needs in-region HA | [`DATA_RDS_MULTIAZ_CLUSTER`](../../F369_CICD_Template/prompt_templates/partials/DATA_RDS_MULTIAZ_CLUSTER.md) |
| Cross-region DR (RPO ≤ 1s, RTO ≤ 1 min) | Regulated workload requires cross-region | [`DATA_AURORA_GLOBAL_DR`](../../F369_CICD_Template/prompt_templates/partials/DATA_AURORA_GLOBAL_DR.md) |
| DB-stream → S3 / Iceberg without Lambda glue | DDB Streams / Kinesis CDC → lakehouse | [`DATA_EVENTBRIDGE_PIPES`](../../F369_CICD_Template/prompt_templates/partials/DATA_EVENTBRIDGE_PIPES.md) |
| Salesforce / Slack / ServiceNow / 60+ SaaS sources → lakehouse | Marketing/sales/CRM data needs to land in lakehouse | [`DATA_APPFLOW_SAAS_INGEST`](../../F369_CICD_Template/prompt_templates/partials/DATA_APPFLOW_SAAS_INGEST.md) |
| Spark on Iceberg/Hudi/Delta — heavy transforms beyond Athena's scope | Custom UDFs, ML feature engineering, large compaction | [`DATA_EMR_SERVERLESS_SPARK`](../../F369_CICD_Template/prompt_templates/partials/DATA_EMR_SERVERLESS_SPARK.md) |
| Cross-DB JOIN / Snowflake / BigQuery in a single Athena SQL | Federation across multiple DBs from Athena | [`DATA_ATHENA_FEDERATED_QUERY`](../../F369_CICD_Template/prompt_templates/partials/DATA_ATHENA_FEDERATED_QUERY.md) |
| SOC 2 / HIPAA / GDPR audit-ready data lake security baseline | Compliance audit prep | [`SECURITY_DATALAKE_CHECKLIST`](../../F369_CICD_Template/prompt_templates/partials/SECURITY_DATALAKE_CHECKLIST.md) |

**Known template gaps still open:**
- Multi-tenant partitioning — use `DATA_MULTITENANT_DDB` partial directly if client has multi-tenant isolation requirements.

---

## 9. Design companion

The "why" behind this kit's shape — the A+C combination rationale, business case, UX design, and 3 detailed business-flow examples (finance analytics / operations Q&A / compliance audit) — lives at:

[`kits/_design/ai-native-lakehouse.md`](_design/ai-native-lakehouse.md)

Partners and sales leads should read the design doc before pitching. Engineers running a 2-week engagement should read this kit.

---

## 10. Changelog

| Version | Date | Change |
|---|---|---|
| 1.0 | 2026-04-23 | Initial kit. Combines AI-Ready Lakehouse (text-to-SQL + QuickSight Q over governed data) + Vector-Native Lake (semantic discovery across structured + unstructured) into one 2-week engagement. Exercises all 12 new partials from Waves 1-4 plus 2 reusables from prior kits (`DATA_S3_VECTORS`, `PATTERN_DOC_INGESTION_RAG`). 100% partial coverage — no new partials required for first engagement. 4 domain-pack variants (finance_analytics / operations_ops / sales_revenue / mixed_multi_domain) share one architecture; consultant sets `DOMAIN_PACK` + optional flags (MULTIMODAL_ENABLED, ZERO_ETL_ENABLED, DATAZONE_ENABLED) at kickoff. Companion design doc at `kits/_design/ai-native-lakehouse.md`. Golden-set of 10+ questions per DOMAIN_PACK for nightly regression. |
