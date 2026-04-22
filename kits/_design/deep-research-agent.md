<!-- Design Doc | Companion to: kits/deep-research-agent.md | Last-updated: 2026-04-22 -->

# Design — Deep Research Agent Kit

> **Purpose of this doc.** Captures the rationale, UX shape, architecture, and three business-flow examples that frame why the kit has the shape it does. The kit itself (`kits/deep-research-agent.md`) is the executable playbook; this doc is the "why we built it this way" reference for partners, pitch decks, and internal training.

---

## 1. Why this kit — business case

Autonomous research agents are the benchmark AI category of 2026 — Perplexity Deep Research, Claude Research, Gemini Deep Research, and OpenAI Deep Research all ship the same shape. Every enterprise wants "the Perplexity for our data + our tools". For a consulting firm doing 30 engagements/year, this is the highest-hit-rate AI kit:

- **One shape, many domains.** Same supervisor + sub-agent topology works for PE due diligence, SaaS competitive intel, regulatory horizon scanning, sales prospect research, pharma literature review, patent analysis. Consultant picks a `DOMAIN_PACK` at kickoff that swaps tool set + memory schema + report template — not the architecture.
- **Full AgentCore GA surface.** This is the one kit that genuinely exercises Browser Tool + Code Interpreter + A2A — three AgentCore GA features that have no natural home in the HR or RAG kits. It's the demo that sells re:Invent booth visitors.
- **Highest-value dollar per engagement.** PE / legal / strategy clients pay the most for "saves analyst-days per deal"; the kit's value prop lands cleanly.

---

## 2. Frontend UX — what the end user sees

**Single chat window. User talks to the Supervisor (Research Agent). Sub-agents are invisible workers.** The supervisor drives the conversation, proposes a plan, streams progress as sub-agents run, then renders a cited report.

```
┌─ Chat panel (React + WebSocket streaming) ─────────────────────┐
│                                                                │
│  User:     Research Acme Corp's competitive position vs        │
│            Globex and Initech for enterprise data platforms.   │
│                                                                │
│  Research ▶ Proposed plan:                                     │
│  Agent     1. Each company's product positioning (websites)    │
│            2. Customer reviews (G2, Gartner Peer Insights)     │
│            3. Pricing tiers (their pricing pages)              │
│            4. Recent product announcements (6 months)          │
│            5. Financial health indicators (if public)          │
│                                                                │
│            [Approve]  [Edit]  [Cancel]                         │
│                                                                │
│  User:     [clicks Approve]                                    │
│                                                                │
│  Research ▶ 🔍 Researcher-A: browsing acme.com/product …       │
│  Agent       🔍 Researcher-B: fetching g2.com/globex reviews   │
│              🔍 Researcher-C: parsing 10-K filings via SEC MCP │
│              📊 Analyst: computing pricing comparison matrix   │
│              ✍️ Writer: drafting competitive summary           │
│              ✅ Reviewer: checking 47 citations v sources      │
│                                                                │
│            Research complete. 4m 23s · $0.87 LLM cost.         │
│                                                                │
│  Research ▶ [Renders full report with inline citations as      │
│  Agent       clickable [¹] [²] [³] linking to source quote]    │
│                                                                │
│  User:     Drill into Globex's pricing changes over 2025       │
│                                                                │
│  Research ▶ [Uses memory: recalls earlier research, fetches    │
│  Agent       new pricing data, shows delta table]              │
│                                                                │
└────────────────────────────────────────────────────────────────┘

Sidebar: [Past research] [Saved preferences] [Export: PDF / DOCX / MD]
```

UX design principles, baked into the kit defaults:
- **One agent, one chat.** User never sees more than the supervisor, even though 4–6 sub-agents are orchestrated underneath.
- **Plan review gate** (`APPROVAL_GATE=plan_review`) by default. User sees the plan before the agent spends minutes and dollars. Gate can be skipped for recurring routines (competitive-sweep kind).
- **Progress is streamed.** Every tool-use event becomes a visible row: "🔍 Researcher-B: fetching …". Transparency builds trust; the alternative — a black-box spinner for 4 minutes — fails user-testing every time.
- **Citations are first-class.** Every factual claim in the report links to a source (web page, doc page, SEC filing, etc.). The reviewer sub-agent gates on grounding before returning. Hallucinated citations kill enterprise trust — this is the table stakes.
- **Memory is user-scoped, not tenant-wide.** Each user has their own preferences + past-research context; same chatbot, different context per user.

---

## 3. Architecture under the chat

```
User ⇌ Supervisor (Research Agent, on AgentCore Runtime — 8h session)
              │
              ├── Planner      (same agent, earlier step — emits JSON plan)
              │
              ├──▶ Researcher-A ── MCP Gateway ── Browser Tool (live web)
              ├──▶ Researcher-B ── MCP Gateway ── Enterprise KB (S3 Vectors)
              ├──▶ Researcher-C ── MCP Gateway ── Domain MCP (SEC / news / CRM / etc.)
              │   (parallel; each a Strands @tool-wrapped sub-agent on AgentCore Runtime)
              │
              ├──▶ Analyst      ── Code Interpreter (pandas, matplotlib, numpy)
              │
              ├──▶ Writer       (synthesises findings into structured report)
              │
              └──▶ Reviewer     (grounding validator: every citation [¹] → source check;
                                 Cedar-policy gate: topic deny, PII redact, legal-advice block)
                    └── if fail: loops back to relevant Researcher with specific gap

Memory:        AgentCore Memory (LTM: user preferences, recurring tracking lists,
                                 past research summaries; STM: current session state)

Guardrail:     Bedrock Guardrail (PII + topics) + AgentCore Agent Control (Cedar policy)

Observability: AgentCore Observability
               - token cost per run (plan + parallel research + synthesis + review)
               - tool latency distribution
               - grounding score trend over time
               - user feedback (thumbs up/down → eval dataset)
```

---

## 4. How the 9 AgentCore GA features + Strands get exercised

| AgentCore / Strands feature | Role in this kit |
|---|---|
| **Runtime** (8h microVM session) | Supervisor holds plan + partial findings across a multi-minute research run |
| **Gateway** (MCP endpoint) | All external tools flow through Gateway: browser, enterprise KB, domain MCPs |
| **Identity** (OAuth2 in/out) | Outbound OAuth to SharePoint, Jira, Slack, GitHub, paid data sources (Crunchbase, PitchBook) |
| **Memory** (STM + LTM strategies) | User preferences (research style, citation format), recurring tracking lists, past-research summaries |
| **Observability** (managed traces) | Token cost per run, tool latency, grounding score trend, feedback ratio |
| **Browser Tool** | Fetch live web pages: company sites, news, regulatory portals, review sites |
| **Code Interpreter** | Analyst sub-agent: pandas for financial / review / pricing analysis, matplotlib for charts |
| **Agent Control** (Guardrail + Cedar) | Block competitor-confidential leak, PII redact, legal-advice block, no-competitor-scraping policy |
| **A2A Protocol** | Cross-agent call to specialist domain sub-agents (e.g. in-house regulatory-expert agent) |
| **Strands `Agent` + `@tool`** | Planner / Researcher / Analyst / Writer / Reviewer roles implemented as @tool-wrapped sub-agents |
| **Strands multi-agent** (Graph/Swarm/Workflow) | Parallel researchers (fan-out), synthesised by Writer (fan-in) |
| **Strands hooks/plugins** | RBAC middleware (per-user tool allowlist), circuit breaker (fail fast if a source is down), token tracker |
| **Strands session manager** (S3) | Resume research runs after user disconnect (browser closed mid-research) |
| **Strands eval** (grounding validator) | Reviewer uses this on every citation before emitting the report |

No other single kit shape exercises all 9 AgentCore GA features. This is what makes it the "hero" demo kit.

---

## 5. Three business-flow examples — same kit, three verticals

### 5.1 M&A Due Diligence — Private Equity client

**Client persona:** Mid-market PE firm, 3 analysts reviewing 20+ targets/quarter, each target review takes 2–3 analyst-days.

**Chat ask:** *"Full diligence report on TargetCo — focus on revenue concentration, management team, technology moat, and any regulatory risks. Our hold thesis is 3–5 years."*

| Step | What happens | Primary feature |
|---|---|---|
| 1. Plan | Supervisor emits 14-step plan grouped into 5 workstreams | Runtime |
| 2. Financial | Researcher-C pulls 3y financials via SEC / Crunchbase MCP | Gateway + Identity |
| 3. Market | Researcher-A browses analyst reports on IBISWorld, Gartner | Browser Tool |
| 4. Technology | Researcher-B queries client's past-deal KB in S3 Vectors for similar stacks | Memory (LTM: firm's prior deals) |
| 5. Regulatory | Researcher-D scrapes court records + FTC filings | Browser + A2A (specialist sub-agent) |
| 6. Analysis | Analyst computes customer concentration (HHI), CAGR, EV/EBITDA comps | Code Interpreter |
| 7. Synthesis | Writer drafts 25-page diligence memo following firm's template | Writer |
| 8. Review | Reviewer grounds every financial figure to source; flags unverifiable claims | Eval + Grounding |
| 9. Output | Report rendered in chat; export to DOCX for IC pack | — |

**Deliverable:** IC-ready diligence memo, cited. Saves 2–3 analyst-days per target.
**Sell price:** ~$150k / quarter (unlimited use + model costs pass-through).

---

### 5.2 Competitive Intelligence — B2B SaaS product team

**Client persona:** SaaS company, Product VP, weekly competitor sweep before Thursday planning.

**Chat ask:** *"What did Globex ship this week? Any pricing changes, new features, or customer announcements I should know about before our Thursday planning?"*

| Step | What happens | Primary feature |
|---|---|---|
| 1. Recall | Agent reads user's LTM preferences: tracked competitors = Globex + Initech, depth = brief, format = slide-ready bullets | Memory (LTM) |
| 2. Plan | Supervisor skips plan review — user has `AUTO_RUN_SCHEDULE=weekly` setting | Runtime |
| 3. Scrape | Researcher-A browses Globex product-updates blog, changelog, pricing page | Browser Tool |
| 4. News | Researcher-B pulls last-7-day Globex mentions via news MCP / Google Alerts | Gateway + Identity |
| 5. Social | Researcher-C scrapes LinkedIn, Twitter, Hacker News | Browser + A2A |
| 6. Diff | Analyst computes delta vs last week's sweep (in memory) | Code Interpreter + Memory (STM) |
| 7. Synthesis | Writer renders 6 bullet slides | Writer |
| 8. Output | User opens chat Thursday 7 AM, report ready; follow-up questions OK | Observability ($/run trending down over weeks as memory caches) |

**Deliverable:** 6-bullet weekly sweep, Thursday 7 AM. Replaces 3-hour manual scrape.
**Sell price:** ~$25k / year, recurring. Sticky (every week, every product team).

---

### 5.3 Regulatory Horizon Scanning — General Counsel, regulated industry

**Client persona:** Health-tech company's GC + 2 in-house lawyers, tracking FDA / HHS / state AG rule changes affecting product.

**Chat ask:** *"What's new in the last 30 days on AI-in-medical-devices rules? Anything we need to file a comment on, and what's the current status of the ONC HTI-2 rule?"*

| Step | What happens | Primary feature |
|---|---|---|
| 1. Classify | Supervisor identifies "scheduled horizon scan" mode, pulls tracking list (regs, agencies, deadlines) | Memory |
| 2. Fed scan | Researcher-A fetches Federal Register API via MCP, filters `FDA + medical-device + AI` | Gateway |
| 3. Agency sites | Researcher-B browses FDA / ONC / HHS press pages | Browser Tool |
| 4. Docket | Researcher-C fetches public comments on HTI-2 docket, clusters by theme | Browser + Code Interpreter |
| 5. Precedent | Researcher-D queries client's regulatory-response KB for similar past filings | S3 Vectors KB via Gateway |
| 6. Deadlines | Analyst computes filing-window calendar, flags conflicts with user's bar events | Code Interpreter |
| 7. Guardrail | Reviewer blocks anything resembling legal advice per Cedar policy ("no legal advice to non-lawyers") | Agent Control (Cedar + Guardrail) |
| 8. Output | Report + calendar entries + suggested comment-letter outline | — |

**Deliverable:** Weekly regulatory scan + filing-deadline digest. Replaces a paralegal-hours task, reduces miss risk.
**Sell price:** ~$60k / year, recurring. Highest-stickiness engagement type.

---

### 5.4 Why all three are the same kit

Same supervisor, same sub-agent roster, same frontend, same non-negotiables. What swaps at kickoff is:

- **Tool set on Gateway** — SEC MCP + Crunchbase vs news-MCP + social-scrape vs Federal-Register MCP + court-records MCP
- **Memory schema** — `deal_pipeline[]` vs `competitor_list[]` vs `reg_tracking[]`
- **Report template** — IC memo (25p) vs slide bullets (6) vs filing digest
- **Cedar policy** — PE confidentiality vs trade-secret-neutral vs no-legal-advice
- **Schedule** — on-demand vs weekly vs weekly + trigger-on-new-filing

**One kit, three sell motions.** That's the value.

---

## 6. Critical decisions baked into the kit

Each becomes a parameter the consultant sets at engagement kickoff (also listed in the kit itself):

- `DOMAIN_PACK` — picks tool set + memory schema + report template (3 built-in packs: `pe_diligence`, `saas_competitive`, `regulatory_scan`; `custom` for new verticals)
- `RESEARCH_DEPTH` — `quick` (1–2 min, surface) | `standard` (~5 min, full plan) | `deep` (15 min+, exhaustive)
- `CITATION_POLICY` — `every_claim` (strict) | `key_facts` (balanced) | `style_only` (loose)
- `AUTO_RUN_SCHEDULE` — `on_demand` (user asks) | `daily` | `weekly` (recurring sweep mode)
- `APPROVAL_GATE` — `plan_review` (user approves plan first) | `auto` (skip review, just run)
- `AGENT_RUNTIME` — `agentcore` (recommended for 8h sessions + parallel sub-agents) | `ecs_fargate` (cost-constrained alt)
- `MEMORY_MODE` — `agentcore_memory` (recommended) | `s3_session_manager` (simpler) | `none` (stateless)
- `TOOL_SET` — which MCP targets to wire on the Gateway
- standard kit params — `TARGET_LANGUAGE`, compliance, retention, model IDs

---

## 7. Known partial gaps this kit surfaces

The existing 59 partials at v2.0 cover ~80% of this kit's surface. Two genuine new partials are needed:

| Gap | Severity | Why partials don't cover it today |
|---|---|---|
| **AgentCore Browser Tool** — CDK + IAM + session config for the headless browser tool | HIGH — central to the "live web research" capability | GA feature, not yet covered; `STRANDS_TOOLS.md §3.3` has Code Interpreter as a `@tool` but Browser needs its own treatment (session lifecycle, sandbox config, cost controls, screenshot-return schema) |
| **AgentCore Code Interpreter** — dedicated deep-dive on the sandbox, file I/O via S3, library set, cost controls | MED — `STRANDS_TOOLS.md §3.3` has a shim, but a standalone partial would codify the idioms (pandas + matplotlib + output-to-S3 presigned URLs + session isolation) | Same: GA feature, not yet fully partial-covered |

Plan: **write these 2 new partials alongside the kit** (same pattern we used for HR kit's 3 partials and RAG kit's 2 partials). After that, this kit has 100% partial coverage.

Other existing partials that carry most of the weight:
- `AGENTCORE_RUNTIME` (supervisor + sub-agent hosting)
- `AGENTCORE_GATEWAY` (MCP tool endpoint)
- `AGENTCORE_IDENTITY` (OAuth2 outbound)
- `AGENTCORE_MEMORY` (STM + LTM)
- `AGENTCORE_OBSERVABILITY` (traces, cost, drift)
- `AGENTCORE_AGENT_CONTROL` (Cedar + Guardrail)
- `AGENTCORE_A2A` (cross-agent specialist calls)
- `STRANDS_AGENT_CORE` (supervisor + sub-agents)
- `STRANDS_TOOLS` (@tool pattern, incl. current Code Interpreter shim)
- `STRANDS_MULTI_AGENT` (Graph/Swarm/Workflow orchestration)
- `STRANDS_MCP_TOOLS` (client-side MCP via SigV4)
- `STRANDS_HOOKS_PLUGINS` (RBAC + circuit breaker + token tracker)
- `STRANDS_EVAL` (grounding validator)
- `STRANDS_FRONTEND` (WebSocket streaming callback to the chat UI)
- `DATA_S3_VECTORS` (KB for enterprise docs)
- `PATTERN_DOC_INGESTION_RAG` (ingestion side of the enterprise KB)
- `LAYER_API`, `LAYER_FRONTEND`, `LAYER_BACKEND_LAMBDA`, `LAYER_SECURITY`, `LAYER_NETWORKING`, `LAYER_OBSERVABILITY`, `LAYER_DATA`, `COMPLIANCE_HIPAA_PCIDSS`

---

## 8. What a consultant delivers at end of Week 2

- **CDK app** with ~12 stacks (Network, Security, Data, KB, Gateway, Runtime-supervisor, Runtime-researchers, Runtime-writer, Runtime-reviewer, Memory, API/Frontend, Observability, Compliance)
- **React chat UI** with streaming + plan-review gate + citation sidebar + export (PDF/DOCX/MD) + past-research history
- **4–6 agent containers** (supervisor + researcher template + analyst + writer + reviewer) deployed to AgentCore Runtime
- **Tool set on Gateway** — the client's `DOMAIN_PACK` selected tools wired in
- **Memory schema + sample preferences** populated for 2–3 pilot users
- **Cedar policy + Bedrock Guardrail** per the client's compliance stance
- **Observability dashboards** — token cost per run, grounding score, feedback ratio, tool latency
- **Compliance stack** — CloudTrail + audit bucket + Config rules per `COMPLIANCE_STANDARD`
- **Runbook + pilot test cases** — 3 sample research questions per domain pack, expected outputs, rollback steps

---

## 9. Open questions for the consulting lead to validate with client

1. **Which domain pack is primary?** Drives tool-set wiring on Gateway + report template on Writer.
2. **Which data sources does the client pay for already?** Crunchbase / PitchBook / Bloomberg / LexisNexis / S&P Capital IQ / similar. Their existing licenses determine MCP targets.
3. **What's the compliance posture?** SOC 2 is kit default; HIPAA for health-tech GC; none for internal R&D tools.
4. **Human-in-the-loop required on final output?** If yes, reviewer sub-agent gates the report on a Cedar policy + sends to a human approval queue before render. If no, auto-return.
5. **Browser Tool scope** — is the client OK with public web scraping (CI, PE, regulatory) or are there terms-of-service concerns (e.g. LinkedIn scraping)? Kit defaults to public pages; add robots.txt compliance by default.
6. **Cost caps** — max $/research run, max runs/day/user. Goes into AgentCore Observability alarm thresholds.

---

## 10. References

- AgentCore GA announcement: https://aws.amazon.com/blogs/aws/amazon-bedrock-agentcore-generally-available/
- Strands Agents SDK: https://strandsagents.com
- Perplexity Deep Research (inspiration): https://www.perplexity.ai/hub/blog/introducing-perplexity-deep-research
- Claude Research / Agent SDK (inspiration): https://www.anthropic.com/news/research
- Related v2.0 partials: see §7 above for the full loadout + flagged gaps

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 1.0 | 2026-04-22 | Initial design doc. Rationale, UX, architecture, 3 business flows, AgentCore/Strands feature mapping, gaps, open client questions. Companion to the forthcoming kit at `kits/deep-research-agent.md`. |
