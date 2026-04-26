# F369 Engagement Kits — Decision Tree + Navigation

**Location:** `E:\F369_LLM_TEMPLATES\kits\`
**Count:** 6 kits (1 at v2.0 reference standard · 5 grandfathered v1.x as of 2026-04-23)
**Authoring standard (mandatory for new kits):** [`_template/README.md`](./_template/README.md) — **Business-First Kit Standard v1.0**
**Design docs:** [`_design/`](./_design/README.md)

Kits are **2-week consulting engagement playbooks**. Each kit is a business-ask-focused artifact (not an AWS-service-focused one) that chains LLM templates + CDK partials + a React/Next.js UI + compliance overlay into a single deliverable a consulting team can ship in ~10 developer-days.

**As of 2026-04-23,** every NEW kit MUST follow the [Business-First Kit Standard v1.0](./_template/README.md): §1 business outcome → §2 personas + workflows → §3 frontend reference spec → §4 AWS architecture → §5 delivery plan. The reference implementation is [`qualitative-research-audio-analytics.md`](./qualitative-research-audio-analytics.md) (v2.0). Existing v1.x kits are grandfathered; retrofit only when a paying engagement lights them up.

---

## How to use a kit

1. **Match the client ask to a kit** — see §Decision Tree below.
2. **Read the kit's design doc** (`_design/<kit>.md`) — business case, pricing, UX rationale, 3 business-flow examples.
3. **Fill kit parameters** — each kit has a `## 1. Kit-wide parameters` section with 15-25 knobs (PROJECT_NAME, DOMAIN_PACK, TARGET_LANGUAGE, COMPLIANCE_STANDARD, model IDs, cost ceilings, UI settings, etc.).
4. **Execute the day-by-day plan** — each day references the specific LLM template(s) to paste into Claude + the partials Claude must load.
5. **Run the deliverables checklist** at end of Week 2.

Most kits expose 3–5 optional flags (`MULTIMODAL_ENABLED`, `ZERO_ETL_ENABLED`, `DATAZONE_ENABLED`, `APPROVAL_GATE`, etc.) that scope the engagement up or down without changing the architecture.

---

## Kit Registry

| # | Kit | Standard | Client-ask quote | Engagement | Pitch price | Partials (new + reused) | Design doc |
|---|---|---|---|---|---|---|---|
| 1 | [hr-interview-analyzer](./hr-interview-analyzer.md) | v1.x | "Upload interview video → AI scoring" | 2 wk · 2 devs | $100K–$175K | 3 new + ~8 reused | (design merged into kit) |
| 2 | [rag-chatbot-per-client](./rag-chatbot-per-client.md) | v1.x | "ChatGPT over our docs, per tenant" | 2 wk · 2 devs | $125K–$225K | 2 new + ~10 reused | (design merged into kit) |
| 3 | [deep-research-agent](./deep-research-agent.md) | v1.x | "Perplexity for our data + tools" | 2 wk · 2 devs | $175K–$350K | 2 new + ~14 reused | [_design/deep-research-agent.md](./_design/deep-research-agent.md) |
| 4 | [acoustic-fault-diagnostic-agent](./acoustic-fault-diagnostic-agent.md) | v1.x | "Diagnose equipment from its sound" | 2 wk · 2 devs | $200K–$350K | 2 new + ~12 reused | [_design/acoustic-fault-diagnostic-agent.md](./_design/acoustic-fault-diagnostic-agent.md) |
| 5 | [ai-native-lakehouse](./ai-native-lakehouse.md) | v1.x | "Single chat over structured + unstructured data" | 2 wk · 2 devs | $150K–$500K | 12 new + ~6 reused | [_design/ai-native-lakehouse.md](./_design/ai-native-lakehouse.md) |
| 6 | **[qualitative-research-audio-analytics](./qualitative-research-audio-analytics.md)** | **v2.0 ★** | "Industrial qualitative audio analytics (1K+/day, multi-persona)" | 2 wk POC · 2 devs | $250K–$500K | 7 new + ~15 reused | [_design/qualitative-research-audio-analytics.md](./_design/qualitative-research-audio-analytics.md) |

**★ Reference implementation** for the [Business-First Kit Standard v1.0](./_template/README.md). Read this kit end-to-end as the canonical example before authoring a new kit.

**Planned kit slots (not yet authored, priority for engagement-driven build):**

| # | Slot | Trigger to author | Will use new partials |
|---|---|---|---|
| 7 | `multi-source-data-platform` | First client engagement requiring Oracle/SQL-Server migration + 60+ SaaS ingest + cross-region DR + active-passive reads — typically a finance / healthcare / retail BI engagement at $300K-$1M | `DATA_DMS_REPLICATION`, `DATA_RDS_MULTIAZ_CLUSTER`, `DATA_EVENTBRIDGE_PIPES`, `DATA_APPFLOW_SAAS_INGEST`, `DATA_AURORA_GLOBAL_DR`, `DATA_ATHENA_FEDERATED_QUERY` |
| 8 | `regulated-lakehouse-soc2-hipaa` | First HIPAA / SOC 2 Type II audit-prep engagement | `SECURITY_DATALAKE_CHECKLIST`, `COMPLIANCE_HIPAA_PCIDSS`, `DATA_LAKE_FORMATION` |
| 9 | `spark-heavy-feature-engineering` | First ML feature-engineering engagement requiring custom UDFs / Hudi upserts at scale | `DATA_EMR_SERVERLESS_SPARK`, `DATA_LAKEHOUSE_ICEBERG`, `MLOPS_SAGEMAKER_TRAINING` |

These slots are **not built speculatively**. They author when a paying engagement lights them up, per the Business-First Kit Standard's retrofit policy. The 8 underlying partials (Wave 5, 2026-04-26) are already authored and audit-pending — when an engagement lands, the kit composes them rather than building from scratch.

**Status:** All 6 published kits have been audited through at least one round. The 12 new partials introduced by kit 5 were audited in round R3 (2026-04-23) with surgical fixes applied. Kit 6 v2.0 surfaced 2 additional new partials (`PROMPT_LAB_VERSIONING`, `EVIDENCE_LINKED_LLM_OUTPUT`) — not yet audited. **Wave 5 (2026-04-26) added 8 data-platform partials** (DMS, RDS Multi-AZ, EventBridge Pipes, AppFlow, EMR Serverless, Aurora Global, Athena Federated, Security Datalake Checklist) — not yet audited. See `../F369_CICD_Template/docs/audit_report_partials_v2*.md` for the audit trail.

---

## Decision Tree — "Which kit for which client ask?"

Read the client's ask verbatim. Match the **bold phrase** below to pick the kit:

### Start here

- **"We want to upload [video | audio | media files] and have the system analyze / score / summarise / extract insights"**
  → **hr-interview-analyzer** (if HR/talent) or **acoustic-fault-diagnostic-agent** (if equipment/manufacturing)
  - Both share a media-preprocessing pipeline pattern; differ in the analysis stage (NLP scoring vs acoustic classification)

- **"We want a chatbot over [our documents | our contracts | our policies | our SOPs]"**
  → **rag-chatbot-per-client**
  - Multi-tenant by default (per-client isolation via tenant-prefixed namespaces)
  - Pure doc-RAG; no structured-data side

- **"We want something like [Perplexity | Claude Research | Gemini Deep Research] but on our data"**
  → **deep-research-agent**
  - Supervisor + 4-6 specialist sub-agents
  - Hero kit for exercising all 9 AgentCore GA features
  - DOMAIN_PACK variants: PE diligence / SaaS competitive / regulatory scan / custom

- **"We want business users to ask questions over [our data warehouse | our data lake | our Iceberg tables] in English, with [visuals | citations]"**
  → **ai-native-lakehouse**
  - The flagship kit: text-to-SQL + RAG + multimodal + QuickSight Q, blended
  - DOMAIN_PACK variants: finance_analytics / operations_ops / sales_revenue / mixed_multi_domain

- **"We run [focus groups | in-depth interviews | telephone surveys] and need AI to transcribe + summarize + extract themes + enable cross-study search across thousands of recordings per month"**
  → **qualitative-research-audio-analytics**
  - First kit with industrial-scale media ingest (1K+ audio files/day)
  - Multi-persona UX: Admin ops cockpit + Analyst research workspace
  - DOMAIN_PACK: qualitative_research; CLIENT_VERTICALS for pharma/retail/finance/automotive/healthcare

### Tiebreakers — overlapping asks

- **"Our team does [diligence research | competitive intel | regulatory horizon scanning] — we want to automate it"**
  → **deep-research-agent** with `DOMAIN_PACK=pe_diligence` / `saas_competitive` / `regulatory_scan`

- **"We have [docs + data] and want blended answers"** (mixed structured + unstructured)
  → **ai-native-lakehouse** (NOT rag-chatbot — that's docs-only)

- **"We have [multiple business units / domains] each wanting their own data but with shared governance"**
  → **ai-native-lakehouse** with `DATAZONE_ENABLED=true`

- **"We want [dashboards | BI] with AI"** (structured only, no unstructured side)
  → **ai-native-lakehouse** with `MULTIMODAL_ENABLED=false` + `QUICKSIGHT_Q_ENABLED=true`

- **"Our Amazon Q in QuickSight isn't producing good answers — can you improve it?"**
  → **ai-native-lakehouse** (the kit's catalog-embedding layer + text-to-SQL grounding directly addresses the Q-quality problem)

### Client asks that do NOT match a kit (consulting engagement, not a kit)

- **"Build us a bespoke classifier / regression model"** — use `mlops/01_sagemaker_training_pipeline.md` directly; kit not needed.
- **"Migrate our on-prem ML workflows to AWS"** — combination of `iac/02` + `mlops/00` + custom migration work. Scope too variable for a kit.
- **"Fine-tune an LLM on our data"** — use `mlops/02_llm_finetuning_pipeline.md` directly.
- **"Set up our CI/CD for ML"** — use `cicd/03` + `cicd/04` or `cicd/05` directly.

---

## Kit parameter matrix

Common parameters across all 5 kits (each kit may add domain-specific ones):

| Parameter | All kits use? | Values | Notes |
|---|---|---|---|
| `PROJECT_NAME` | ✅ | `<kit>-<client_slug>` | Resource-name prefix |
| `AWS_REGION` | ✅ | e.g. `us-east-1` | Pick one with all required GA services for the kit |
| `AWS_ACCOUNT_ID` | ✅ | 12-digit | For ARN construction |
| `ENV` | ✅ | `dev` / `stage` / `prod` | Usually deploy all three stages |
| `CLIENT_SLUG` | ✅ | e.g. `acme` | Short lowercase client identifier |
| `TARGET_LANGUAGE` | ✅ | `python` / `typescript` | CDK language; affects path anchoring idioms (see partial `LAYER_BACKEND_LAMBDA`) |
| `DOMAIN_PACK` | 4 of 5 | kit-specific enum | Drives prompt library + schema seed + demo questions |
| `COMPLIANCE_STANDARD` | ✅ | `SOC2` / `HIPAA` / `PCI_DSS` / `NONE` | Drives `ComplianceStack` (partial `COMPLIANCE_HIPAA_PCIDSS`) |
| `AUTH` | 4 of 5 | `cognito_user_pool` / `iam_identity_center` / `okta_oidc` | SSO strategy; `iam_identity_center` required if DataZone is on |
| `SUPERVISOR_MODEL_ID` | 3 of 5 (agent-driven kits) | `claude-sonnet-4-7` / `claude-opus-4-7` / etc. | Always prefix with inference-profile ARN shape |
| `MAX_COST_PER_RUN_USD` | all agent kits | $0.10–$5.00 | Hard ceiling per turn/research |
| `STREAM_RESPONSES` | 3 of 5 (chat-UI kits) | `true` / `false` | WebSocket streaming; disable only for API-only consumers |

**Hint:** If the client insists on a specific LLM model not in the default list, pick the nearest Claude 4.x and flag `TODO(verify): model availability in <region>`. Do not use Claude 3.x unless fine-tuning a fine-tunable model (Claude 3 Haiku is the only currently-fine-tunable variant).

---

## Kit composition patterns

Some clients want two kits merged into one engagement (e.g. "lakehouse + HR analyzer"). The patterns below show how to compose cleanly.

### Pattern A — Shared horizontal stacks

Most kits can share these horizontals (deploy once, consume from multiple kit stacks):

- `NetworkStack` (VPC, subnets, PrivateLink endpoints) — from `devops/02_vpc_networking_ml.md`
- `SecurityStack` (KMS keys, permission boundaries) — from `LAYER_SECURITY` partial
- `AuthStack` (Cognito user pool, IAM Identity Center) — from `LAYER_SECURITY` + custom
- `ComplianceStack` (Object-Lock audit bucket, CloudTrail, Config rules) — from `COMPLIANCE_HIPAA_PCIDSS`
- `ObservabilityStack` (cross-kit dashboards, alarms) — from `LAYER_OBSERVABILITY`

**When combining kits:** deploy these 5 horizontals ONCE in a shared "Platform" account; each kit's vertical stacks consume via SSM-published ARNs.

### Pattern B — Lakehouse + RAG chatbot

Client wants "analysts query tables in English AND the same chatbot answers from contracts/policies".

- Deploy `ai-native-lakehouse` as the base.
- RAG corpus shares the lakehouse's `DocRagStack` (already part of the lakehouse kit's `doc_rag` tool).
- No need for a separate `rag-chatbot-per-client` kit.
- Savings: ~$50K off the combined engagement vs deploying both kits independently.

### Pattern C — Deep Research + Lakehouse

Client wants "autonomous research on external sources + grounded in our internal data".

- Deploy `ai-native-lakehouse` for internal-data tools.
- Deploy `deep-research-agent` with `ENABLE_ENTERPRISE_KB=true` pointing at the lakehouse's catalog + doc indexes.
- The deep-research supervisor can invoke the lakehouse's chat-router as a sub-tool via AgentCore A2A.

### Pattern D — HR + Acoustic (both media-analysis)

Client is a manufacturing company that wants BOTH interview analysis AND equipment-sound diagnostics.

- Share the media-ingest pipeline (S3 upload → Textract/Transcribe → feature extraction).
- Two separate analysis stacks (NLP for HR, acoustic classifier for equipment).
- Share observability + compliance stacks.
- Savings: ~$40K off the combined engagement.

### Pattern E — Multi-tenant SaaS variant

Client is a SaaS vendor wanting to RESELL any kit's capability to their own customers.

- Use `rag-chatbot-per-client` as the base template (already multi-tenant).
- Layer the target kit's tools on top, scoped per tenant via Cognito user-pool namespace + S3 Vectors per-tenant prefix + DDB per-tenant partition key.
- Budget +50% on the base-kit price for the multi-tenancy premium.

---

## Authoring a new kit (checklist)

> **NEW KITS MUST FOLLOW [Business-First Kit Standard v1.0](./_template/README.md).** Read it end-to-end before starting. Copy [`_template/EXAMPLE_kit-skeleton.md`](./_template/EXAMPLE_kit-skeleton.md) as your starting file.

Authoring sequence (the standard enforces this order — DO NOT skip):

1. **Buyer interview (or proxy)** — get on a call with someone in the §1.1 buyer role. If unavailable, find a published case study / job posting / earnings transcript that exposes the buyer's priorities. Capture verbatim language. Write §1.
2. **Persona workflow mapping** — sketch the 3-5 personas. Write the day-in-the-life narratives BEFORE thinking about screens. Write §2.
3. **Frontend reference spec** — sketch routes, declare every domain entity with fields, list every visualization, list standout features. Write §3.
4. **Reference frontend (working code)** — build the actual SPA in a sibling repo (e.g. `E:\NBS_<client>_regen\frontend\`). Field-tune §3.2 domain types as build reveals friction. **You cannot publish §3 without a working reference frontend.**
5. **Architecture from constraints** — read §1.4 anti-patterns + §3.2 types + §3.4 features; the architecture is a function of these. Write §4. Each component cites its §1-§3 driver.
6. **Delivery plan** — day-by-day execution table, partials referenced, deliverables checklist, golden-set assertions. Write §5.
7. **Identify partial gaps** — list new partials surfaced in §4.5. Build them in `F369_CICD_Template/prompt_templates/partials/` per Canonical-Copy Rule.
8. **Audit the new partials** — follow `F369_CICD_Template/prompt_templates/partials/_prompts/audit_partials_v2.md`.
9. **Run the kit audit gate** — see `_template/README.md §Audit gate` (~30 sign-off items across §1-§5). All MUST pass.
10. **Register the kit** — add to the Registry table above with `Standard: v2.0`. Update top-level `../README.md`.

Each prior kit has surfaced 2-12 new partials. Budget this in when estimating.

**Effort:** A v2.0 kit takes 5-10 days to author (vs. 1-2 days for v1.x). The extra effort buys durable sales assets that retain value across multiple engagements. See `_template/README.md §Why this is worth the extra effort` for the tradeoff.

---

## Cross-repo dependency

Kits depend on **partials** in the companion repo `F369_CICD_Template`.

- **Partial library index:** `../../F369_CICD_Template/prompt_templates/partials/README.md`
- **Canonical registry:** same file, §Canonical Partials Registry
- **The 5 non-negotiables:** `../../F369_CICD_Template/prompt_templates/partials/LAYER_BACKEND_LAMBDA.md §4.1` — echoed in every dual-variant partial
- **Audit history:** `../../F369_CICD_Template/docs/audit_report_partials_v2*.md` (three rounds to date)

If you edit a kit to reference a NEW partial, ensure that partial exists + has been audited in the companion repo. If it doesn't exist, list it in the kit's §7 "Known partial gaps" and create an issue.

---

## Known gaps (aggregate across kits)

| Gap | Which kits surface it | Mitigation |
|---|---|---|
| Multi-tenant partitioning | all 5 | Use `DATA_MULTITENANT_DDB` partial + per-tenant S3 prefixes |
| Cross-region DR | all 5 | Not covered in default kits; add Aurora Global / S3 CRR per-client |
| Human-in-the-loop approval UI | hr, deep-research | Approval-gate pattern exists in `deep-research-agent`; copy for other kits |
| FinOps auto-scaling rules | all agent kits | Use `finops/04_inference_cost_optimization.md` as a bolt-on |
| Streamlit / Gradio "quick-demo" UI | none | Deliberately not supported — for real engagements, use Next.js (see `LAYER_FRONTEND`) |

---

## Related

- [`../README.md`](../README.md) — top-level library index + three-layer model
- [`_design/README.md`](./_design/README.md) — strategic view: why these kits, in this order, for which clients
- [`../../F369_CICD_Template/`](../../F369_CICD_Template/) — companion partial library (75 partials at v2.0, all audited)
