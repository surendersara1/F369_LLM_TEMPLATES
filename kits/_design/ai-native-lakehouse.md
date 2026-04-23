<!-- Design Doc | Companion to: kits/ai-native-lakehouse.md | Last-updated: 2026-04-23 -->

# Design — AI-Native Lakehouse Kit

> **Purpose of this doc.** Captures the rationale, UX shape, architecture, and three business-flow examples that frame why the kit has the shape it does. The kit itself (`kits/ai-native-lakehouse.md`) is the executable playbook; this doc is the "why we built it this way" reference for partners, pitch decks, and internal training.

---

## 1. Why this kit — business case

Every enterprise above 500 people has the same three-sided data problem:

1. **Structured data** lives in a warehouse (Redshift, Snowflake, or a managed Iceberg lake). Analysts query it with SQL.
2. **Unstructured data** lives in a doc store or sharepoint-style content system (contracts, policies, SOPs, customer notes).
3. **Images and diagrams** live in scattered drives (engineering drawings, wireframes, whiteboard photos).

Each has its own query tool (SQL, search, grep). Business users who need to ask questions **across these** either: (a) burn an analyst-day getting joined data, or (b) give up and operate on gut instinct. **Every client we've worked with in 2025-2026 has asked the same question in various forms** — "can we get ChatGPT over our data?".

The naïve answer is: "build a RAG chatbot". That's our `rag-chatbot-per-client` kit, and it's the right answer when the data is 90% documents. It's the wrong answer when the question genuinely mixes structured and unstructured: *"show me revenue trends for customers whose contracts mention renewal bonuses."* That's two sources, blended. A RAG-only solution hallucinates the revenue numbers.

**This kit is the answer to that mixed question.** An agentic router that knows when to run SQL, when to retrieve docs, when to search images, and when to do all three and blend the results — with citations, governed access, and in-browser visualization.

**For a consulting firm doing 30 engagements/year**, this kit sits at the top of the engagement-value rankings:

- **Highest-complexity-per-kit, in a 2-week window.** Integrates 12 partials (7 new + 5 from prior kits). Exercises Athena, S3 Tables, S3 Vectors, Lake Formation, Bedrock, AgentCore Memory, AgentCore Identity, Strands Agents, QuickSight Q. Every major data + AI service touched.
- **Client-side value is easy to pitch.** Replace 3 data analysts' slowest workflows with a chat window. A 2-week build + 2-week pilot is enough for a Finance or Operations team to stop opening tickets.
- **Defensible service.** The governance overlay (LF-TBAC + identity propagation) is what keeps this from being a "ChatGPT leaks our database" story. That's a meaningful differentiation vs. DIY attempts.

---

## 2. The key design decision — why combine A and C

Back when we scoped this kit, we considered three shapes (see chat history):

- **A — AI-Ready Lakehouse**: Iceberg + Lake Formation + text-to-SQL + QuickSight Q. The "BI upgrade with AI" angle.
- **B — Multi-Domain Data Mesh**: DataZone + cross-domain subscriptions. The "data mesh enablement" angle.
- **C — Vector-Native Lake**: Every row + doc + image gets a vector; semantic similarity is the primary query. The "find anything" angle.

Each alone has a problem:

- **Pure A** fails at text-to-SQL for a well-known reason: the LLM doesn't know which of 200 tables and 2,000 columns to use, so it hallucinates schemas. ~40% of its queries fail on first try without schema grounding.
- **Pure C** can't answer *"sum of Q3 revenue by region"*. Vectors are good for similarity (*"find me tables about revenue"*); they can't do exact arithmetic.
- **Pure B** is a 6-12 month transformation, not a 2-week engagement. We can't deliver "a data mesh" in 10 developer-days; we can bolt DataZone onto a single-domain kit as an option.

**The combination of A + C is structurally better than either alone because vectors are the INDEX and SQL + RAG are the RETRIEVAL.**

- The catalog-embedding index (`PATTERN_CATALOG_EMBEDDINGS`) gives the text-to-SQL agent a 3-pass discovery flow that tells it **which tables and columns are relevant** before it writes SQL. Hallucination rate drops from ~40% to ~5%.
- The multimodal index (`PATTERN_MULTIMODAL_EMBEDDINGS`) lets the router answer *"show me a wiring diagram"* by searching image embeddings — the lake stops being blind to its visual content.
- The unstructured layer (`PATTERN_DOC_INGESTION_RAG`) handles contract Q&A, policy lookups, customer notes.
- **The router blends** — *"revenue for customers whose contracts mention renewal bonuses"* fires SQL + doc-RAG in parallel, receives both answers, and synthesizes.

This is the shape Databricks and Snowflake are converging on with their "semantic layer + AI-native query" programs (Databricks Genie, Snowflake Cortex). **Nobody on AWS has packaged it as a 2-week consulting artifact yet.** This kit is that packaging.

B (DataZone) stays opt-in — a flag the consultant flips if the client signals a multi-BU mesh requirement. It layers cleanly on top because DataZone consumes our Glue catalog + LF governance.

---

## 3. Frontend UX — what the end user sees

**Single chat window. User talks to the Router (enterprise chat router). Tools are invisible workers.** The router proposes a response strategy, streams progress as tools run, then renders a blended answer with inline citations.

```
┌─ AI-Native Lakehouse Chat ──────────────────────────────────────────┐
│                                                                     │
│  User:   Show me revenue trends for our top 20 customers this       │
│          quarter, plus any renewal or escalation notes from          │
│          their contracts.                                           │
│                                                                     │
│  Chat  ▶ 🔎 Running text_to_sql (finding top 20 by revenue)…        │
│         ▶ 📑 Running doc_rag (contracts for customer IDs 1-20)…     │
│         ▶ 🎨 Running multimodal_search for any attached             │
│            escalation ticket screenshots…                           │
│                                                                     │
│  Chat  ▶ Here are the top 20 customers by QTD revenue [SQL-1]:      │
│                                                                     │
│          | Customer       | QTD Revenue | Renewal Date | Notes       │
│          |----------------|-------------|--------------|-------------│
│          | Acme Corp      | $1.2M       | 2026-06-30   | [DOC-1]    │
│          | Globex         | $980K       | 2026-09-15   | [DOC-2]    │
│          | ...                                                       │
│                                                                     │
│          Key contract notes:                                        │
│          - Acme: renewal bonus threshold at $1M exceeded [DOC-1]    │
│          - Globex: escalation pending — see attached ticket [IMG-1] │
│                                                                     │
│  Sources:                                                           │
│          [SQL-1] SELECT customer_id, SUM(amount)… scanned 240 MB    │
│          [DOC-1] contracts/acme-msa-2024.pdf#page=12                │
│          [DOC-2] contracts/globex-msa-2023.pdf#page=8               │
│          [IMG-1] escalation-screenshots/jira-glb-447.png            │
│                                                                     │
│  User:   [clicks IMG-1, sees full-size screenshot]                  │
│          Export this to CSV and schedule a quarterly refresh.       │
│                                                                     │
│  Chat  ▶ [Triggers QuickSight dashboard export + schedule]          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

Sidebar:  [Recent chats]  [QuickSight dashboards]  [Data catalog]  [Export]
```

UX design principles baked into the kit:

- **One chat, one router.** User never sees more than the router. The 4-tool orchestration is streamed (🔎/📑/🎨 emoji progress events) but framed as steps in a single thinking process, not separate agents.
- **Citations are first-class.** Every table row has a `[SQL-N]` linking to the SQL + scan stats. Every narrative claim has a `[DOC-N]` or `[IMG-N]`. Ungrounded claims don't get sent (the router's system prompt enforces it).
- **Progress is streamed.** Every tool invocation becomes a visible progress event. Transparency builds trust; a black-box spinner for 8 seconds kills conversions.
- **QuickSight Q is a peer.** "Show me the revenue dashboard" delegates to an embedded QuickSight Q panel that renders the chart inline. Chat and Q share the same Cognito auth, the same LF-enforced data, the same namespace.
- **Access is governed by identity, not agent.** User Alice with `domain=finance, max_sensitivity=internal` permissions sees only finance-domain data at internal sensitivity. The router's OBO token propagates her identity to each tool Lambda — the LF grants are CHECKED AT TOOL EXECUTION TIME.
- **No PII leaks via "semantic search".** Discovery shows columns tagged `sensitivity=pii` only if Alice's JWT says she can see them. The 3-pass retrieve filters BEFORE similarity sort; not after.

---

## 4. Architecture under the chat

```
 Browser (Next.js UI)
    │
    │  ws://...       HTTPS (signed with Cognito JWT)
    │
    ▼
 API Gateway WS                API Gateway REST   ←─── QuickSight Q embed SDK
    │                              │                   (separate session path)
    ▼                              │
 ChatRouterSupervisor              │
  (Strands, Claude Sonnet 4.7)     │
  Tools:                           │
  ├── text_to_sql  ────────┐       │
  ├── semantic_discovery ──┤       │
  ├── doc_rag ─────────────┤       │
  └── multimodal_search ───┼───────┴───▶ QuickSight Q (via embed URL)
                           │
         (invoke: Lambda)  │
                           │
                           ▼
 ┌──────────────────────────────────────────────────────────────────┐
 │ Tool Lambdas (one each)                                           │
 │                                                                   │
 │  text_to_sql: discover → generate (Sonnet 4.7) → preflight        │
 │               (EXPLAIN) → execute → budget track                  │
 │                                                                   │
 │  semantic_discovery: 3-pass retrieve (db → tbl → col) → Haiku     │
 │                      rerank + NL summary → cache                  │
 │                                                                   │
 │  doc_rag: vector query → chunk retrieve → cite                    │
 │                                                                   │
 │  multimodal_search: embed query → vector query → signed preview   │
 └──────────────────────────────────────────────────────────────────┘
                           │
         ┌─────────────────┼─────────────────┬──────────────┐
         ▼                 ▼                 ▼              ▼
    ┌──────────┐      ┌──────────┐      ┌──────────┐   ┌────────────┐
    │ Athena   │      │ Catalog  │      │ Doc RAG  │   │ Multimodal │
    │ v3       │      │ Embed    │      │ S3 Vec   │   │ S3 Vec     │
    │          │      │ S3 Vec×3 │      │          │   │            │
    │ WG:      │      │ db,tbl,  │      │          │   │            │
    │ agent    │      │ col      │      │          │   │            │
    │ 1 GB cap │      │          │      │          │   │            │
    └────┬─────┘      └────┬─────┘      └────┬─────┘   └────┬───────┘
         │                 │                 │              │
         ▼                 ▼                 ▼              ▼
    ┌──────────┐      ┌──────────┐      ┌──────────┐   ┌────────────┐
    │ S3 Tables│      │ Glue Cat │      │ S3 doc   │   │ S3 raw     │
    │ Iceberg  │      │ (+ LF    │      │ bucket + │   │ bucket +   │
    │ auto-    │      │ Tags)    │      │ Textract │   │ Pillow +   │
    │ comp     │      │          │      │ pipeline │   │ PyMuPDF +  │
    │          │      │          │      │          │   │ Rekognit.  │
    └──────────┘      └──────────┘      └──────────┘   └────────────┘

 Governance (cross-cutting):   Lake Formation LF-TBAC; CfnDataCellsFilter;
                                AgentCore Identity OBO tokens
 Observability:                 CloudWatch dashboards on scan GB / rerank
                                latency / chat-turn cost / tool-routing freq
 Compliance:                    Object-Lock audit bucket, Config rules,
                                CloudTrail, WAF on CloudFront + API GW
```

---

## 5. Three business-flow examples

### 5.1 Finance Analytics (DEFAULT)

**Client:** A 2,500-person SaaS company with $400M ARR. Finance team of 8 people. Data warehouse is a messy mix of Redshift (for BI) + S3 Parquet files (for raw exports). Contract PDFs live in SharePoint. Nobody has a single view.

**Before the kit:** A typical "where are we on Q3 revenue vs. at-risk renewals" question takes:
- 20 min: analyst opens Redshift, writes SQL for QTD revenue
- 45 min: analyst opens SharePoint, greps contract PDFs for "renewal" or "churn risk" terms for each top-20 customer
- 30 min: analyst cross-references the two, builds an Excel
- **Total: 95 min per question, 3× per week × 8 analysts = 38 hours/week = nearly $50K/month analyst cost**

**After the kit:**
- User types question into chat
- Router runs SQL on fact_revenue (2 s) + RAG over contracts (4 s)
- Blended answer in <10 s, with citations
- Excel export button for the skeptical analyst to verify

**Value:** 38 hours/week saved → $200K / year, plus finance gets to questions they'd have skipped because they weren't worth 95 min.

**Pitch price:** $150K-$250K for the kit + 2-week pilot. Payback in 4-6 months.

### 5.2 Operations Q&A

**Client:** A 10,000-person industrial manufacturer. Operations teams run equipment across 50 sites. Maintenance logs in a CMMS (Oracle-backed), equipment schematics in engineering drives (CAD exports → PDFs), and incident summaries in an internal ticketing system.

**Before the kit:** A plant engineer diagnosing a recurring failure on pump model XJ-550 needs to:
- 30 min: pull the CMMS history for that pump model across 50 sites
- 60 min: find the right wiring schematic in a 12,000-drawing Sharepoint
- 45 min: search ticket history for similar failure signatures
- 30 min: cross-reference with vendor maintenance bulletins (also PDFs)
- **Total: 165 min per diagnostic; 15× per week × 40 engineers = 165 hours/week, ~$9,000/week**

**After the kit:**
- Engineer: *"What are recent failures of XJ-550 pumps, with their wiring schematics and any vendor bulletins?"*
- Router: text_to_sql (CMMS via Athena federation) + doc_rag (bulletins) + multimodal_search (schematics by model number) → blended answer in 15 s

**Value:** 165 hours/week → 50 hours/week = 115 hours/week saved = $468K/year direct labor. Harder to measure: faster MTTR means less downtime, which is often 10× the labor number.

**Pitch price:** $200K-$350K. Often bolts onto an existing CMMS modernization.

**This flow uses `DOMAIN_PACK=operations_ops` + `MULTIMODAL_ENABLED=true` — the multimodal layer is NON-OPTIONAL here because schematic retrieval is the hero capability.**

### 5.3 Compliance Audit

**Client:** A 500-person financial services company (PCI-DSS + SOC 2 scope). Auditors arrive quarterly. Every audit cycle, the compliance team spends 4 weeks preparing: pulling transaction samples, finding control policy documents, correlating with control-execution logs.

**Before the kit:** Audit prep is literally one person × 4 weeks × every quarter = 17 weeks/year = $170K+ of fully-loaded comp officer time. And the audit questions often can't be answered cleanly because the structured + unstructured data is siloed.

**After the kit:**
- Auditor asks: *"Show me all transactions over $100K from customers flagged in our enhanced-due-diligence list during Q3, cross-referenced to the KYC policy in effect at the time."*
- Router: text_to_sql (fact_transactions + dim_customer with EDD flag) + doc_rag (KYC policies, point-in-time via document-versioning) + citation package
- Export as audit-ready PDF with SQL provenance + doc provenance

**Value:** 17 weeks → 4 weeks = 13 weeks/year saved. Plus faster audit cycles reduce follow-up requests.

**Pitch price:** $175K-$275K. Often bundled with `COMPLIANCE_STANDARD=PCI_DSS` + WAF + Macie + Object-Lock audit bucket for a fully governed deployment.

**This flow uses `DOMAIN_PACK=finance_analytics` + `COMPLIANCE_STANDARD=PCI_DSS` + `DATAZONE_ENABLED=true` when the client has separate AML, Fraud, and Compliance teams each wanting domain-scoped access.**

---

## 6. Build sequencing — why these 4 waves, in this order

Each wave depends on the prior; later waves fail meaningfully if the earlier ones aren't rock-solid. This is why the 2-week engagement follows the same 4-wave structure we used to build the partials:

**Wave 1 (Days 1-4): Lakehouse foundation.** Iceberg storage, Glue catalog with well-commented columns, Lake Formation LF-Tags, Athena workgroups. This is the BORING part — but EVERYTHING after depends on it. Skimping on column comments here means text-to-SQL hallucinates later. Skimping on LF-Tags means governance leaks. Do this right.

**Wave 2 (Day 5): Vector index.** Catalog embeddings — 3 S3 Vectors indexes populated by a bulk-reindex SFN. This is where hallucination protection lives. Also multimodal index if enabled. These indexes don't do anything user-facing yet, but everything above them needs them ready.

**Wave 3 (Days 6-8): Agents.** Text-to-SQL Lambda, discovery Lambda, chat router. This is where the kit comes alive — a chat window backed by indexed data with governance. 3 days is tight but doable because the partials do the heavy lifting.

**Wave 4 (Days 9-10): BI + compliance + go-live.** QuickSight dashboard, CloudFront UI, audit bucket, pilot with real users. "Polish" day; the architecture is done, we're hardening and enabling.

Order matters because:
- You can't do text-to-SQL (Wave 3) before catalog embeddings (Wave 2).
- You can't do embeddings (Wave 2) before column comments + LF-Tags (Wave 1).
- You can't demo to business users (Wave 4) without a chat router (Wave 3).

Skipping ahead means re-doing work. We've seen teams try to "just get text-to-SQL running" first, then retrofit embeddings — they end up regenerating every test query after adding catalog embeddings, for a net 40% slower ship.

---

## 7. What we considered and didn't do

A running list of "why not" decisions that come up in every engagement:

**"Why not use Bedrock Agents native (not Strands)?"**
Bedrock Agents is simpler but less flexible — it's a fixed action-group model without custom tool-call loops, streaming control, or multi-agent patterns. For the chat router we need streaming tool-call events (for UI transparency) + fine control over which tool fires in which order. Strands gives us that; Bedrock Agents doesn't. *If the client has < 3 tools and doesn't need streaming, we'd use Bedrock Agents.*

**"Why not OpenSearch for catalog embeddings?"**
S3 Vectors is new (GA 2025), cheaper by ~3-10× for the "infrequent query" pattern we're in, and integrates with KMS + LF in a cleaner way. OpenSearch Serverless is better for hybrid BM25+vector search, which we might add as a secondary index in a large deployment. *If query volume > 100 qps sustained, add OpenSearch as a companion.*

**"Why not Bedrock Knowledge Bases (managed RAG)?"**
KB is simpler for pure doc RAG (no custom chunking, auto-embedding). The downside: it's a black box for governance — you can't easily apply LF-Tags to chunks, can't do 3-pass discovery, can't debug retrieval. For enterprise deployments where governance is the #1 concern, custom RAG via `PATTERN_DOC_INGESTION_RAG` is the right choice. *For PoCs where governance isn't priority-1, KB is 3× faster to ship.*

**"Why not use Redshift as the primary analytics engine?"**
Athena is better for this kit because: (1) serverless — no always-on compute cost; (2) Iceberg DML is first-class; (3) Glue Catalog + LF integration is tighter; (4) text-to-SQL generation is well-trained on Presto/Trino dialect; (5) `USING FUNCTION invoke_model` for Bedrock-from-SQL is Athena-only. Redshift shines for heavy-concurrency BI; add it as a companion (Zero-ETL or Spectrum) when the client has serious dashboard load. *`ZERO_ETL_ENABLED=true` flag covers this.*

**"Why not DataZone as the default?"**
DataZone adds ops complexity (IAM Identity Center prerequisite, environment blueprint management, subscription workflow) that doesn't pay off until the client has 3+ data domains wanting to share. Most first-engagement clients have one data team that owns everything; DataZone is over-engineering. *Flip `DATAZONE_ENABLED=true` for multi-BU engagements.*

**"Why force LF-TBAC as MANDATORY?"**
We debated letting consultants flip LF off for speed. The cost: later, when the client scales, every single Athena query needs to retrofitted with LF grants — an architectural tax. Doing it right from Day 1 costs ~4 hours of extra setup and saves months of rework. `LF_ENFORCE=strict` is the default for this reason.

**"Why not build the UI in Streamlit or Gradio?"**
For a demo, sure. For a real engagement that'll be used by 50+ business users, Next.js + CloudFront + OAC is what they'll need for auth, CORS, caching, and integration into their existing SSO. Streamlit is banned past Day 2.

---

## 8. Open questions + when they'll land

**Q: Can we auto-tune the agent workgroup scan cutoff based on question complexity?**
A: Research question. Current 1 GB is a safe default but costs false-denials on legitimate multi-billion-row aggregations. Future work: a classifier (Haiku) that categorises questions as "small" / "medium" / "large" and routes to different workgroups.

**Q: Can we let users edit the SQL before execution?**
A: Yes — we expose `sql` in the response and the UI can show it + offer "run again with my edits". Requires a separate `execute_sql` tool. Not in the default kit; add when a client requests it.

**Q: What happens when a user asks about data they don't have permission for?**
A: 3-pass discovery filter returns zero results for their domain. The router says *"I couldn't find data matching that question in your authorized scope. If you believe you should have access, contact your data team."* This is the RIGHT behaviour, but the UX is abrupt — a follow-up is "help me request access" — deferred to a later kit version.

**Q: How do we handle questions that span a time range longer than our data?**
A: Today: the LLM usually writes a query that returns an empty result, and the system answers "no data for that range". Better: a pre-check Lambda that validates the time range against the table's partition bounds. Future work.

**Q: When does this kit become stale?**
A: The Strands Agents SDK is the single most churny dependency (we've seen 2 breaking changes in the last 6 months). AgentCore Memory/Identity are alpha-ish. Budget a quarterly refresh of the kit against SDK updates. The partials' `TODO(verify)` markers flag the alpha-API surfaces.

---

## 9. Pricing + sales angles

| DOMAIN_PACK | Typical client | Typical kit price | Pilot length | Payback |
|---|---|---|---|---|
| finance_analytics | 500-5,000 person SaaS / services | $150K-$250K | 2-4 weeks | 4-6 months |
| operations_ops | 1,000-20,000 person industrial | $200K-$350K | 4-6 weeks | 3-5 months |
| sales_revenue | 200-2,000 person B2B | $125K-$225K | 2-4 weeks | 5-8 months |
| mixed_multi_domain | 3,000+ person enterprise (DATAZONE_ENABLED=true) | $300K-$500K | 6-10 weeks | 6-12 months |

**Common objections + responses:**
- *"We already have Databricks / Snowflake — why AWS?"* → "This kit runs on whatever catalog you have; if your lakehouse is Databricks, we federate via `DATA_GLUE_CATALOG` external catalog. The kit is about the AGENT + GOVERNANCE layer, not locking you to one storage vendor."
- *"ChatGPT / Claude already answer data questions — why pay for this?"* → "They don't have your data, and they don't enforce your access controls. The pitch is governance + blended sources + citations, not 'an LLM'."
- *"Won't users just ask the LLM to bypass access controls?"* → "LF is runtime-enforced at Athena, not a prompt instruction. Even if the LLM writes the 'right' SQL, the query returns empty for columns/rows the user can't see. Defence-in-depth: prompt + preflight allowlist + LF runtime."

---

## 10. Next engagements — evolution pattern

The kit's partial library is designed to support a **long tail** of derivative engagements:

- **Kit v1.1 (+ 6 months):** Add `PATTERN_CROSS_SOURCE_SQL` — federated Athena queries across Iceberg + Redshift + Snowflake.
- **Kit v1.2:** Add `PATTERN_AGENT_AUDIT_TRAIL` — every chat turn logged with SQL + docs to an immutable audit bucket, with search UI for auditors.
- **Kit v1.3:** Add `PATTERN_CUSTOM_REPORT_GEN` — LLM-authored scheduled reports (weekly revenue summary, daily ops digest) emailed to stakeholders.
- **Kit v2.0:** Multi-tenant SaaS variant — each tenant gets their own isolated lakehouse + chat router + QuickSight namespace.

Each evolution adds 2-3 partials; none require re-architecting the base kit. That's the payoff of the three-layer model (kits / templates / partials) we built this stack around.
