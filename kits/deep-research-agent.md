<!-- Kit Version: 1.0 | Template library: F369_LLM_TEMPLATES v1.x | Partials: F369_CICD_Template partials v2.0 -->

# Kit — Deep Research Agent (Supervisor + Specialist Sub-Agents)

**Client ask:** "Give us our own Perplexity Deep Research / Claude Research — on our data, our tools, our policies. User asks a research question, the agent plans, researches across live web + enterprise KB + domain data, and delivers a cited report."
**Engagement:** 2 weeks · 2 developers · regulated-ready (SOC 2 default; HIPAA / PCI variants via params).
**Deliverable:** CDK repo (Python or TypeScript) + React chat UI + AgentCore Runtime agents + domain tool set wired through Gateway + memory schema + eval dashboards.

**Scope rationale:** See companion design doc [`kits/_design/deep-research-agent.md`](_design/deep-research-agent.md) — business case, UX design, 3 business flow examples (PE due diligence / SaaS competitive intel / regulatory horizon scanning), and AgentCore GA feature mapping.

---

## 1. Kit-wide parameters (fill ONCE; carried to every template + Claude call)

```
# --- Identity -----------------------------------------------------------
PROJECT_NAME:               deep-research-{client_slug}
AWS_REGION:                 us-east-1  (or any region with AgentCore Runtime + Browser Tool + Code Interpreter + S3 Vectors)
AWS_ACCOUNT_ID:             [12-digit]
ENV:                        dev | stage | prod
CLIENT_SLUG:                acme
TARGET_LANGUAGE:            python | typescript

# --- DOMAIN PACK — drives tool set + memory schema + report template ----
DOMAIN_PACK:                pe_diligence | saas_competitive | regulatory_scan | custom
                            DEFAULT: saas_competitive (most demo-friendly)

# --- RESEARCH BEHAVIOUR -------------------------------------------------
RESEARCH_DEPTH:             quick (1-2 min, surface)
                            | standard (~5 min, full plan)        ← DEFAULT
                            | deep (15 min+, exhaustive, multi-session)

CITATION_POLICY:            every_claim (strict, every fact cited)   ← DEFAULT
                            | key_facts (balanced)
                            | style_only (loose — internal tool only)

APPROVAL_GATE:              plan_review (user approves plan before run) ← DEFAULT for PE/regulatory
                            | auto (skip review — for recurring sweeps)

AUTO_RUN_SCHEDULE:          on_demand (user triggers)                  ← DEFAULT
                            | daily (recurring sweep at set time)
                            | weekly (e.g. Thursday 7 AM)

# --- AGENT HOSTING ------------------------------------------------------
AGENT_RUNTIME:              agentcore   (DEFAULT — 8h sessions, microVM, scales to zero)
                            | ecs_fargate  (cost-constrained alt; no microVM isolation)

MEMORY_MODE:                agentcore_memory  (DEFAULT — STM + LTM strategies)
                            | s3_session_manager (simpler, no LTM)
                            | none (stateless)

# --- TOOL SET (Gateway targets — drives DOMAIN_PACK variations) ---------
ENABLE_BROWSER_TOOL:        true   (headless browsing, public web pages)
ENABLE_CODE_INTERPRETER:    true   (Analyst sub-agent uses pandas / matplotlib)
ENABLE_ENTERPRISE_KB:       true   (S3 Vectors + client's docs)
ENABLE_DOMAIN_MCP:          true   (DOMAIN_PACK-specific: SEC / news / Federal Register / CRM)
ENABLE_A2A:                 false  (set true if calling specialist external agents)

# --- KNOWLEDGE BASE (for ENABLE_ENTERPRISE_KB=true) --------------------
KB_VECTOR_STORE_MODE:       custom_s3_vectors  (DEFAULT — matches partial DATA_S3_VECTORS)
                            | bedrock_kb_s3_vectors (managed chunking + embedding)
EMBEDDING_MODEL_ID:         amazon.titan-embed-text-v2:0
EMBEDDING_DIMENSION:        1024
VECTOR_DISTANCE_METRIC:     cosine
VECTOR_TOP_K:               5

# --- MODELS -------------------------------------------------------------
SUPERVISOR_MODEL_ID:        us.anthropic.claude-sonnet-4-7-20260109-v1:0  (Sonnet 4.7 — planning + synthesis)
RESEARCHER_MODEL_ID:        us.anthropic.claude-haiku-4-5-20251001-v1:0   (Haiku 4.5 — cheap per-source research)
REVIEWER_MODEL_ID:          us.anthropic.claude-sonnet-4-7-20260109-v1:0  (Sonnet 4.7 — grounding + quality gate)
MAX_CONTEXT_TOKENS:         32000   (research runs accumulate retrieved text)
ENABLE_PROMPT_CACHING:      true

# --- COST CONTROLS ------------------------------------------------------
MAX_COST_PER_RUN_USD:       1.00   (hard alarm; soft-warn at 0.60)
MAX_RUNS_PER_USER_PER_DAY:  20
MAX_RESEARCH_SECONDS:       900    (15 min ceiling; hard kill)

# --- UI / UX ------------------------------------------------------------
STREAM_RESPONSES:           true   (WebSocket)
SHOW_PLAN_BEFORE_RUN:       true   (matches APPROVAL_GATE)
SHOW_PROGRESS_EVENTS:       true   (🔍 Researcher-A: browsing acme.com…)
SHOW_CITATIONS:             true
CITATION_FORMAT:            inline [¹] [²]  (DEFAULT)
                            | footnote (end of section)
                            | hyperlink (no number)
EXPORT_FORMATS:             [pdf, docx, markdown]

# --- GOVERNANCE ---------------------------------------------------------
PII_REDACTION_FIELDS:       [name, email, phone, ssn, credit_card]
DENIED_TOPICS:              [legal_advice, medical_advice, financial_advice, competitor_confidential]
CEDAR_POLICY_S3_PATH:       s3://{PROJECT_NAME}-config-{env}/policies/research-agent.cedar
COMPLIANCE_STANDARD:        SOC2 | HIPAA | PCI_DSS | NONE
RETENTION_DAYS:             365   (research + citations + feedback; 2555 for HIPAA)

# --- BROWSER TOOL SAFETY (new for this kit) -----------------------------
BROWSER_RESPECT_ROBOTS_TXT: true   (DEFAULT on — obey site robots.txt)
BROWSER_ALLOWLIST_DOMAINS:  []     (empty = unrestricted public web; populate to restrict)
BROWSER_DENYLIST_DOMAINS:   [linkedin.com, facebook.com]  (login-walled / TOS-sensitive)
BROWSER_MAX_PAGES_PER_RUN:  50
```

---

## 2. Architecture

```
User ⇌ Supervisor (Research Agent on AgentCore Runtime, 8h session)
              │
              ├── Planner      (supervisor's first step — emits JSON plan; APPROVAL_GATE shows to user)
              │
              ├──▶ Researcher-A ── MCP Gateway ── Browser Tool            (live web)
              ├──▶ Researcher-B ── MCP Gateway ── Enterprise KB           (S3 Vectors; via PATTERN_DOC_INGESTION_RAG)
              ├──▶ Researcher-C ── MCP Gateway ── Domain MCP              (per DOMAIN_PACK: SEC / news / Federal-Register / CRM / etc.)
              │   (parallel; each a Strands @tool-wrapped sub-agent on AgentCore Runtime)
              │
              ├──▶ Analyst      ── Code Interpreter                       (pandas, matplotlib, numpy)
              │
              ├──▶ Writer       (synthesises findings into structured report per DOMAIN_PACK template)
              │
              └──▶ Reviewer    (grounding validator: every [¹] → source check; Cedar policy gate)
                    └── if fail: loops back to relevant Researcher with specific gap
                    └── if pass: Supervisor streams final report to user

Memory:        AgentCore Memory (LTM: user preferences + past research; STM: current session)
Guardrail:     Bedrock Guardrail (PII + deny topics) + AgentCore Agent Control (Cedar)
Observability: AgentCore Observability (token cost, tool latency, grounding trend, feedback)
Chat plumbing: React + CloudFront + API GW WebSocket → Supervisor on AgentCore Runtime
```

---

## 3. Execution plan

### Week 1 — Foundation + tool set + supervisor/sub-agents

| Day | Template (fill with kit params, paste to Claude) | Partials Claude MUST load |
|---|---|---|
| 1 | [`iac/02_cdk_ml_llm_infrastructure`](../iac/02_cdk_ml_llm_infrastructure.md) — scaffold CDK app with stacks: `NetworkStack`, `SecurityStack`, `DataStack`, `KBStack` (S3 Vectors), `RuntimeStack` (AgentCore runtimes), `GatewayStack`, `MemoryStack`, `ApiStack`, `FrontendStack`, `ObservabilityStack`, `ComplianceStack` | `LAYER_NETWORKING`, `LAYER_SECURITY`, `LAYER_DATA`, `LAYER_BACKEND_LAMBDA`, `LAYER_OBSERVABILITY`, `AGENTCORE_RUNTIME` |
| 2 | Enterprise KB — instruct Claude "Generate `KBStack` per `DATA_S3_VECTORS` partial (S3 vector bucket + `{project_name}-docs` index, cosine/1024/non_filterable_metadata=['source_text']) and `IngestionStack` per `PATTERN_DOC_INGESTION_RAG` partial. Use Textract parser + fixed-512/64 chunker + Titan v2 embedder." | `DATA_S3_VECTORS`, `PATTERN_DOC_INGESTION_RAG`, `LLMOPS_BEDROCK` |
| 3 | Gateway + tool set — instruct Claude "Generate `GatewayStack` per `AGENTCORE_GATEWAY` partial. Wire targets for DOMAIN_PACK={kit}: Browser Tool + Code Interpreter + Enterprise KB (Lambda proxy to KBStack) + domain MCP (SEC / news / Federal Register per pack)." Use [`mlops/04_rag_pipeline`](../mlops/04_rag_pipeline.md) as the RAG-tool template reference. | `AGENTCORE_GATEWAY`, `AGENTCORE_IDENTITY`, `STRANDS_MCP_TOOLS` · **AGENTCORE_BROWSER_TOOL** (NEW — see §7 gap) · **AGENTCORE_CODE_INTERPRETER** (NEW — see §7 gap) |
| 4 | Supervisor + sub-agents — [`mlops/22_strands_agentcore_deployment`](../mlops/22_strands_agentcore_deployment.md) for the supervisor, then one runtime per sub-agent role (researcher-template, analyst, writer, reviewer). Instruct Claude "Follow `STRANDS_MULTI_AGENT` supervisor + fan-out pattern; supervisor = Sonnet 4.7, researchers = Haiku 4.5, reviewer = Sonnet 4.7. Expose each sub-agent as a Strands `@tool` on the supervisor." | `STRANDS_AGENT_CORE`, `STRANDS_TOOLS`, `STRANDS_MODEL_PROVIDERS`, `STRANDS_MULTI_AGENT`, `STRANDS_DEPLOY_ECS`, `AGENTCORE_RUNTIME`, `AGENTCORE_IDENTITY` |
| 5 | Memory + session — [`mlops/22_strands_agentcore_deployment`] + instruct Claude "Wire AgentCore Memory per `AGENTCORE_MEMORY` partial: LTM strategies = SUMMARY + USER_PREFERENCE + SEMANTIC. Memory schema per DOMAIN_PACK (deal_pipeline / competitor_list / reg_tracking)." | `AGENTCORE_MEMORY` |

### Week 2 — UI + guardrails + eval + observability + go-live

| Day | Template | Partials |
|---|---|---|
| 6 | Chat UI + WebSocket — instruct Claude "Generate `ApiStack` (API GW v2 WebSocket + REST for config) per `LAYER_API` and `FrontendStack` (React + CloudFront + OAC) per `LAYER_FRONTEND`. Implement plan-review gate (user approves plan before supervisor runs), progress-event streaming (`🔍 Researcher-A: …`), citation-clickable inline [¹] rendering, export to PDF/DOCX/MD." | `LAYER_API`, `LAYER_FRONTEND`, `STRANDS_FRONTEND` |
| 7 | Guardrails + Cedar policy — [`mlops/12_bedrock_guardrails_agents`](../mlops/12_bedrock_guardrails_agents.md) + instruct Claude "Generate `GuardrailStack` per `AGENTCORE_AGENT_CONTROL`. Load Cedar policy from CEDAR_POLICY_S3_PATH (DOMAIN_PACK-specific: PE confidentiality / no-legal-advice / trade-secret-neutral). Attach Bedrock Guardrail to all 3 model tiers (supervisor/researcher/reviewer). Configure Browser Tool denylist = BROWSER_DENYLIST_DOMAINS." | `AGENTCORE_AGENT_CONTROL` |
| 8 | Eval + grounding — [`mlops/17_llm_evaluation_pipeline`](../mlops/17_llm_evaluation_pipeline.md) + instruct Claude "Implement reviewer sub-agent using `STRANDS_EVAL` grounding validator: every citation [¹] is checked against its source; fail → loops back to relevant Researcher with specific gap. Block report emission if ungrounded citations remain after 1 retry." | `STRANDS_EVAL` |
| 9 | Observability + feedback — [`devops/15_strands_agent_observability`](../devops/15_strands_agent_observability.md) + instruct Claude "Generate `ObservabilityStack` per `AGENTCORE_OBSERVABILITY`. Dashboards: research-token-cost-per-run, tool-latency-p95, grounding-score-distribution, feedback-ratio (thumbs up/down), plans-approved-vs-canceled. Alarm thresholds: cost > MAX_COST_PER_RUN_USD, latency > MAX_RESEARCH_SECONDS." | `AGENTCORE_OBSERVABILITY`, `LAYER_OBSERVABILITY` |
| 10 | Compliance + network + go-live — [`devops/02_vpc_networking_ml`](../devops/02_vpc_networking_ml.md) + [`devops/07_macie_pii_training_data`](../devops/07_macie_pii_training_data.md) + instruct Claude "Generate `ComplianceStack` per `COMPLIANCE_HIPAA_PCIDSS` with COMPLIANCE_STANDARD={kit}. WAF on CloudFront + API GW. Macie on KB upload bucket. CloudTrail → Object-Lock audit bucket. Validate BROWSER_RESPECT_ROBOTS_TXT enforced in Browser Tool config." | `LAYER_NETWORKING`, `COMPLIANCE_HIPAA_PCIDSS`, `SECURITY_WAF_SHIELD_MACIE` |

---

## 4. Complete partial reference (load each into Claude along with the template)

> **URL template:** `https://github.com/surendersara1/F369_CICD_Template/blob/main/prompt_templates/partials/<NAME>.md`

**Core platform (ALWAYS load):**
- `LAYER_BACKEND_LAMBDA` — 5 non-negotiables + grant helpers
- `LAYER_NETWORKING` — VPC + PrivateLink
- `LAYER_SECURITY` — KMS + IAM + permission boundary
- `LAYER_DATA` — DDB for session/preferences/feedback + S3 curated
- `LAYER_OBSERVABILITY` — dashboards, alarms
- `EVENT_DRIVEN_PATTERNS` — S3 → EventBridge for KB ingestion

**AgentCore (most of the 9 GA features):**
- `AGENTCORE_RUNTIME` — supervisor + sub-agent containers with microVM isolation
- `AGENTCORE_GATEWAY` — MCP endpoint + targets (Browser / Code Interpreter / KB / domain)
- `AGENTCORE_IDENTITY` — per-agent roles, OAuth2 outbound for paid data sources
- `AGENTCORE_MEMORY` — STM + LTM strategies, user preferences, recurring tracking
- `AGENTCORE_OBSERVABILITY` — token cost, tool latency, grounding trend, feedback
- `AGENTCORE_AGENT_CONTROL` — Bedrock Guardrail + Cedar policy engine
- `AGENTCORE_A2A` *(if ENABLE_A2A=true)* — cross-agent calls to specialist external agents
- **`AGENTCORE_BROWSER_TOOL`** *(NEW — to be written; see §7)*
- **`AGENTCORE_CODE_INTERPRETER`** *(NEW — to be written; see §7)*

**Strands:**
- `STRANDS_AGENT_CORE` — supervisor + sub-agent pattern, planner → researchers → writer → reviewer
- `STRANDS_TOOLS` — `@tool` wrapping for each sub-agent + Code Interpreter shim
- `STRANDS_MODEL_PROVIDERS` — Bedrock + tiered model selection (Sonnet for supervisor/reviewer, Haiku for researchers)
- `STRANDS_MULTI_AGENT` — Graph/Swarm/Workflow fan-out + synthesis pattern
- `STRANDS_MCP_TOOLS` — client-side MCP via SigV4 to Gateway
- `STRANDS_HOOKS_PLUGINS` — RBAC middleware + circuit breaker + token tracker
- `STRANDS_EVAL` — grounding validator (reviewer uses this on every citation)
- `STRANDS_FRONTEND` — WebSocket streaming callback handler for chat UI
- `STRANDS_DEPLOY_ECS` — container build + deploy to AgentCore Runtime

**Knowledge base (for ENABLE_ENTERPRISE_KB=true):**
- `DATA_S3_VECTORS` — vector bucket + index CDK + query APIs
- `PATTERN_DOC_INGESTION_RAG` — parse → chunk → embed → store pipeline
- `LLMOPS_BEDROCK` — Titan embedder + guardrail wiring

**UI:**
- `LAYER_API` — REST + WebSocket streaming
- `LAYER_FRONTEND` — React + CloudFront + OAC

**Governance (ALWAYS for this kit):**
- `COMPLIANCE_HIPAA_PCIDSS` — audit bucket + Backup Vault Lock + Config rules
- `SECURITY_WAF_SHIELD_MACIE` — WAF + Macie

---

## 5. Architecture non-negotiables (repeat in every Claude prompt)

> **Rules from `LAYER_BACKEND_LAMBDA §4.1` — all generated code MUST pass these, regardless of `TARGET_LANGUAGE`:**
>
> 1. Lambda / container asset paths use `Path(__file__).resolve().parents[N] / "..."` (Python) or `path.join(__dirname, ...)` (TypeScript). Never CWD-relative.
> 2. Cross-stack resource access is **identity-side only** — attach `PolicyStatement` to the consumer's role. Never `bucket.grantReadWrite(crossStackRole)` or `runtime.grant_invoke(crossStackRole)`.
> 3. Cross-stack EventBridge → Lambda uses L1 `events.CfnRule` with a static-ARN target.
> 4. Bucket + CloudFront OAC live in the **same stack** — never split.
> 5. Never `encryption_key=ext_key` where the key came from another stack. Pass KMS ARN as a **string** via SSM.
> 6. Every `iam:PassRole` on a Bedrock / AgentCore / SageMaker role includes `Condition: StringEquals iam:PassedToService <service>.amazonaws.com`.
> 7. **Research agent-specific:** scope `bedrock-agentcore:InvokeAgentRuntime` to specific sub-agent runtime ARNs (supervisor role → researcher ARN, reviewer ARN, etc.), NOT `runtime/*`. Runtime ARN shape: `arn:aws:bedrock-agentcore:{region}:{account}:runtime/{name}`.
> 8. **Browser Tool-specific:** enforce `BROWSER_RESPECT_ROBOTS_TXT=true` and validate against `BROWSER_ALLOWLIST_DOMAINS` / `BROWSER_DENYLIST_DOMAINS` inside the tool invocation, not just at IAM boundary.

---

## 6. Deliverables checklist (end of Week 2)

- [ ] CDK repo (`TARGET_LANGUAGE`) with ~12 stacks: Network, Security, Data, KB (Vector + Ingestion), Gateway, Runtime (supervisor + 4 sub-agents), Memory, API, Frontend, Observability, Compliance
- [ ] 5 containerised Strands agents on AgentCore Runtime: supervisor, researcher-template (spawnable), analyst, writer, reviewer
- [ ] React chat UI with:
    - plan-review gate (user approves before run)
    - progress event stream (`🔍 Researcher-A: browsing acme.com…`)
    - citation-clickable inline [¹] [²] rendering
    - past-research history sidebar
    - export to PDF / DOCX / Markdown
    - thumbs up/down feedback → eval store
- [ ] Gateway with DOMAIN_PACK tool set wired: Browser Tool + Code Interpreter + Enterprise KB + domain MCP
- [ ] Enterprise KB provisioned (S3 Vectors + Titan v2), sample corpus ingested
- [ ] AgentCore Memory with LTM schemas + sample user preferences for 2 pilot users
- [ ] Bedrock Guardrail + Cedar policy per COMPLIANCE_STANDARD
- [ ] Reviewer sub-agent enforces grounding before report emission
- [ ] Observability dashboards: cost/run, latency-p95, grounding score, feedback ratio, plan-approval ratio
- [ ] Compliance: Object-Lock audit bucket, Backup Vault Lock, CloudTrail, Config rules
- [ ] WAF on CloudFront + API GW
- [ ] Runbook with 3 sample research questions per DOMAIN_PACK, expected outputs, rollback steps
- [ ] Pilot test with 2–3 users over 48 hours, feedback captured in eval store

---

## 7. Known partial gaps — NEW partials needed to ship this kit

Existing 59 partials cover ~80% of this kit's surface. 2 genuine new partials are needed:

| File to create | Severity | Why partials don't cover it today |
|---|---|---|
| `AGENTCORE_BROWSER_TOOL.md` — AgentCore Browser Tool CDK + session lifecycle + sandbox + cost controls + screenshot-return + robots.txt enforcement | HIGH — central to live-web research | GA feature; `STRANDS_TOOLS.md §3.3` has Code Interpreter but nothing for Browser Tool |
| `AGENTCORE_CODE_INTERPRETER.md` — dedicated deep-dive on sandbox, file I/O via S3, supported libraries (pandas/matplotlib/numpy/scipy/seaborn), cost controls, output-to-S3 presigned-URL idiom | MED — `STRANDS_TOOLS.md §3.3` has a shim; standalone partial would codify idioms + CDK wiring | GA feature; current shim is a `@tool` example only, no infra patterns |

**Plan:** write both alongside the kit's first real engagement (same pattern we used for HR kit's 3 partials and RAG kit's 2 partials). After that, this kit hits 100% partial coverage.

**Other gaps (same as prior kits, unchanged):**
- No frontend / portal thin-router template — use `LAYER_FRONTEND` partial directly
- No SFN orchestration thin-router template — not needed for this kit
- No compliance-blueprint thin-router template — use `COMPLIANCE_HIPAA_PCIDSS` partial directly
- Library-wide: Claude 3.x / 4.0 IDs in template defaults — override via kit params

---

## 8. Design companion

The "why" behind this kit's shape — business case, UX rationale, 3 business-flow examples with sell prices and hour-savings — lives at:

[`kits/_design/deep-research-agent.md`](_design/deep-research-agent.md)

Partners and sales leads should read the design doc before pitching. Engineers running a 2-week engagement should read this kit.

---

## 9. Changelog

| Version | Date | Change |
|---|---|---|
| 1.0 | 2026-04-22 | Initial kit. Exercises all 9 AgentCore GA features (Runtime / Gateway / Identity / Memory / Observability / Browser Tool / Code Interpreter / Agent Control / A2A) + full Strands multi-agent surface. Four domain-pack variants (PE diligence / SaaS competitive / regulatory scan / custom) share one supervisor-and-specialist-sub-agents architecture — consultant sets `DOMAIN_PACK` at kickoff. Companion design doc at `kits/_design/deep-research-agent.md` captures rationale + UX + 3 business flows. Flags 2 new partials to write (`AGENTCORE_BROWSER_TOOL`, `AGENTCORE_CODE_INTERPRETER`) — ~20% coverage gap, consistent with prior kits that each surfaced 2-3 new partials. |
