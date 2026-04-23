<!-- Design-doc index | Companion to: kits/README.md | Last-updated: 2026-04-23 -->

# Kit Design Index — Strategic View

**Location:** `E:\F369_LLM_TEMPLATES\kits\_design\`
**Purpose:** The "why" behind the kit library — business case, UX rationale, composition patterns, evolution roadmap. Written for partners, sales leads, and internal training. The executable kit playbooks are one level up in `kits/`.

---

## The business case for kits (at a consulting firm)

A consulting firm doing ~30 AWS-sponsored AI engagements per year faces two structural problems:

1. **Every engagement starts from scratch.** Even when two clients have similar asks ("ChatGPT over our docs", "analyze uploaded audio"), each engagement re-derives architecture, re-writes CDK, re-invents UI patterns. 50-70% of the Week 1 effort is repeated work the firm has already shipped elsewhere.

2. **Client asks don't map cleanly to AWS services.** A partner hears "we want Perplexity over our data" and has to translate that into: Bedrock + AgentCore Runtime + Browser Tool + S3 Vectors + Strands + ... + 12 other services. The translation is the bottleneck.

Kits solve both. Each kit is:

- **A translated client ask** — the top of the file is the quote a partner actually hears ("Upload interview video → AI scoring"), and the architecture + parameters + delivery plan flow from that quote.
- **A reusable playbook** — a second client asking the same question gets the kit, not a from-scratch rebuild. Consulting margins improve by 40-60% on repeat engagements.
- **A salable artifact** — the design docs here (`_design/<kit>.md`) are direct inputs to pitch decks and SOWs. Partners don't need to re-invent the pitch every quarter.

---

## Design-doc conventions

Each kit has a corresponding design doc in this folder. The conventions:

**Location:**
```
kits/<kit-name>.md            ← executable playbook (engineering)
kits/_design/<kit-name>.md    ← this folder: strategic rationale (partners, sales)
```

**Structure** (all design docs share this shape):

1. **Why this kit — business case** — the structural client problem the kit solves.
2. **Frontend UX — what the end user sees** — ASCII mockup of the chat/portal UI, UX principles.
3. **Architecture under the chat** — technical diagram + key services.
4. **3 business flow examples** — concrete scenarios with pricing + payback math.
5. **Build sequencing** — why the wave structure + what depends on what.
6. **What we considered and didn't do** — the "why not Databricks / why not Bedrock Agents / why not Streamlit" rationale.
7. **Open questions** — known unknowns + when they'll land.
8. **Pricing + sales angles** — typical prices per DOMAIN_PACK, common objections + responses.

**Some kits merge design into the kit file** when the kit is small enough (hr-interview-analyzer, rag-chatbot-per-client). Larger kits (deep-research-agent, acoustic-fault-diagnostic-agent, ai-native-lakehouse) keep the design doc separate because it's long enough to warrant its own navigation.

---

## The 5 kits — strategic view

### 1. hr-interview-analyzer — "the first kit that validates the three-layer model"

**Why it exists:** A consulting client (Emapta) was building an interview analyzer and asked us to help. We built it, then extracted the pattern.

**Target client profile:** Talent/HR tech companies, staffing agencies, enterprise internal-recruiting teams. Sweet spot: 500-5,000 person companies doing 100+ interviews/month.

**Payback math:** A recruiter spends ~30 min scoring + summarising each interview. At 100 interviews/month × 30 min × $100K loaded salary ÷ 2000 hours ≈ $2,500/month recruiter time saved per 100 interviews. For 500 interviews/month → $12,500/month → $150K/year. Kit pitch price $100-175K → 8-14 month payback.

**Why we built it first:** simplest end-to-end kit (media ingest → ML scoring → dashboard). Validated the kit-template-partial three-layer model. Surfaced 3 new partials.

**Strategic value:** low sales price, high repeat rate (every HR-tech client wants a variant). Use as an "easy win" to open doors.

### 2. rag-chatbot-per-client — "the ubiquitous ask"

**Why it exists:** Every enterprise above 200 people has a "ChatGPT over our docs" backlog. RAG is a commodity; the value is in governance + per-client isolation + citations.

**Target client profile:** Any B2B company with > 1,000 internal documents. Stronger fit: legal, financial services, professional services.

**Payback math:** Varies widely. Common pattern: 20-30 FTEs each spend 5-10 min/day looking up policy/contract information. At 25 × 7.5 min × 220 days × $100/hr ≈ $70K/year. Kit pitch $125-225K → 2-3 year payback (ROI mostly from faster decision-making, not direct labour savings).

**Why it's #2:** multi-tenant from day 1. The pattern generalises to SaaS-vendor scenarios (where the CLIENT is reselling RAG to their own customers), which unlocks a different revenue model.

**Strategic value:** highest-volume, lowest differentiation. Win on delivery speed + governance quality, not architecture novelty.

### 3. deep-research-agent — "the re:Invent booth demo"

**Why it exists:** Autonomous research agents (Perplexity Deep Research, Claude Research, Gemini Deep Research) are the benchmark AI category of 2026. Every enterprise wants one on their data.

**Target client profile:** PE / legal / strategy / pharma firms. The common property: analyst labour is expensive ($200K+ loaded), research tasks are repetitive, output is structured reports.

**Payback math:** Best-case payback is 1-2 months. A PE deal-team analyst spends 2-4 weeks on a diligence memo; the agent produces a cited draft in 10 minutes. At $250K loaded salary and 15 memos/year per analyst, saving 2 weeks per memo × 15 = 30 weeks freed up ≈ $150K/analyst/year. 5 analysts → $750K/year. Kit pitch $175-350K → 3-6 month payback.

**Why it's #3:** highest-complexity kit. Exercises all 9 AgentCore GA features. The hero kit for re:Invent booth visits + AWS co-sell conversations.

**Strategic value:** highest sales price + highest strategic margin for AWS co-sell. But requires the most senior delivery team (Bedrock + AgentCore + Strands expertise).

### 4. acoustic-fault-diagnostic-agent — "the replacement for Lookout for Equipment"

**Why it exists:** AWS announced the retirement of Amazon Lookout for Equipment (Oct 7, 2026). Hundreds of manufacturing customers need a replacement. The OSS audio-ML stack (librosa + PyTorch + AST/Wav2Vec2) on SageMaker MME + AgentCore is a cleaner replacement than it is a new architecture.

**Target client profile:** Manufacturing (automotive, heavy industrial, process), with deployed sensors or microphones on equipment. Sweet spot: 10,000+ employee companies with 50+ plants.

**Payback math:** Typical MTTR improvement 20-40%. A plant with $1M/day downtime exposure and 5 failures/year avoiding 2-hour downtime each = $400K/year saved. Kit pitch $200-350K → 6-9 month payback.

**Why it's #4:** niche. Huge value for clients in-category, zero value outside. Strategic only if the consulting firm has 2+ manufacturing clients asking for Lookout replacement.

**Strategic value:** time-boxed opportunity (until Lookout sunset fully propagates). Window: Oct 2026 – Oct 2027.

### 5. ai-native-lakehouse — "the most requested kit"

**Why it exists:** Combines "chat over our data" + "semantic discovery" + "governed text-to-SQL" into one engagement. Every enterprise with an analytics backlog wants this; nobody on AWS has packaged it as a 2-week deliverable.

**Target client profile:** Any enterprise with BOTH a data warehouse/lake AND significant unstructured data (contracts, policies, customer notes). Sweet spot: 1,000-20,000 employees, $50M+ data-team annual budget.

**Payback math:** Finance-analytics flow: 8 analysts × 38 hours/week saved × $120/hr = ~$47K/month = $560K/year. Kit pitch $150-250K → 3-5 month payback. (See `_design/ai-native-lakehouse.md` for flows across operations, sales, compliance.)

**Why it's #5:** biggest kit (12 new partials, the most in any kit). Requires discipline (Canonical-Copy Rule — born from this kit's audit). Represents where the industry is headed (Databricks Genie, Snowflake Cortex analogue).

**Strategic value:** flagship. Use as the anchor in sales conversations with large enterprises. Lower-cost kits (hr, rag-chatbot) are door-openers; lakehouse is the large-scope follow-on.

---

## Kit composition patterns — when a client wants two

Briefly restated from `../README.md §Kit composition patterns` with the strategic rationale:

- **Shared horizontal stacks** (network, security, auth, compliance, observability) deploy ONCE per account. The consulting firm charges the horizontal once + incremental vertical cost per kit. Economics improve ~30% on multi-kit engagements.
- **Lakehouse + RAG chatbot** — the lakehouse's `doc_rag` tool IS a per-client-scoped RAG. Don't sell two kits; sell the lakehouse with a "per-tenant scoping" upsell.
- **Deep Research + Lakehouse** — the research supervisor can invoke lakehouse tools via AgentCore A2A. This is the "autonomous agent operating on enterprise data" demo — the highest-value pitch for PE/strategy clients.
- **HR + Acoustic** — both are media-analysis kits with shared ingest. Deploy the shared ingest pipeline + two analysis stacks; total engagement cost ~1.6x a single kit, not 2x.
- **Multi-tenant SaaS variant** — any kit can be sold in a "reseller" mode where the CLIENT's CUSTOMERS are the end users. Charge +50% for the multi-tenancy premium.

---

## Discarded / deferred kits

Kits we considered but did NOT build, and why:

### Fraud Detection Agent

**Ask:** "Real-time fraud detection with agent-driven investigation."
**Why deferred:** Streaming architecture shape differs significantly from existing kits (Kinesis + Flink vs. batch/event-driven Lambdas). Would require 4-6 new partials (streaming feature store, real-time SageMaker endpoint with < 50 ms p99, agent-assisted alert triage). Budget for 4-week kit, not 2-week.
**When to build:** when a client signals $100K+ willingness-to-pay AND the fraud-specific partial investment amortises across 3+ future clients.

### Customer Service Agent (Contact Center)

**Ask:** "Amazon Connect + Lex + Bedrock for CS automation."
**Why deferred:** Amazon Connect has its own deep learning curve + existing AWS partner ecosystem. Consulting firms specialised in Connect already have playbooks; this kit wouldn't differentiate.
**When to build:** if client has NO existing Connect deployment + wants green-field CS automation. Rare.

### Autonomous Code Review Agent

**Ask:** "GitHub PR reviewer powered by Bedrock."
**Why deferred:** Crowded market (GitHub Copilot Workspace, Cursor, Claude Code). Hard to differentiate without unique training data access. Would be a consulting engagement more than a packaged kit.
**When to build:** if a client has a unique code corpus (e.g. proprietary DSL, legacy mainframe) + sophisticated review requirements. Bespoke, not templatable.

### Supply Chain Optimization Agent

**Ask:** "Agentic optimization over supply chain data + external signals."
**Why deferred:** AWS Supply Chain service has a specific data model that's hard to generalise. Most clients have custom supply-chain platforms (SAP, Oracle) that would make the kit too client-specific.
**When to build:** inside an existing AWS Supply Chain deployment that needs AI augmentation.

### Healthcare / Clinical Agent (HealthLake + Bedrock + HIPAA)

**Ask:** "AI agent for clinical decision support / patient-record Q&A."
**Why deferred:** Extremely high regulatory burden (HIPAA + clinical validation + FDA 510(k) for some use cases). The kit's $125-500K pitch price would be dwarfed by compliance-audit costs ($250K+). Better as a bespoke engagement with a regulated-AI specialist.
**When to build:** only with an in-house clinical AI SME on the delivery team.

---

## Evolution roadmap

How the kit library evolves over the next 12 months:

### Q2 2026 (current)

- Audit rounds R1, R2, R3 complete (17 + 9 + 12 = 38 partials audited).
- 5 kits shipped + audited. Canonical-Copy Rule institutionalized.
- Focus: pilot the 5 kits with 2-3 real clients. Find bugs via actual `cdk synth` + deploy.

### Q3 2026

- **Kit 1.1 iterations based on pilot feedback.** Expect 2-3 new partials per kit (observability UX tweaks, cost-dashboard enhancements, multi-tenant hardening).
- **Audit round R4** — audit any new partials + re-audit kits 1-5 against the current Canonical Registry.
- **Consider Kit 6 — Fraud Detection Agent** IF a client pilot signals demand.

### Q4 2026

- **Kit 6 (if demand materializes)** — Fraud Detection OR Multi-Tenant SaaS wrapper for an existing kit.
- **Design-doc standardisation** — current design docs are slightly inconsistent in length + depth; normalise to a fixed 8-section structure.
- **Sales collateral** — 1-page summary per kit derived automatically from the kit file's header + the design doc's §5 (pricing).

### Q1 2027

- **Audit round R5** — full library re-audit against latest CDK version + AWS service GA changes (especially AgentCore post-GA).
- **Library maintenance pass** — fix drift in alpha packages that have been promoted to stable.
- **Consider Kit 7+ based on accumulated client demand signals.**

---

## Maintenance

When a kit or design doc changes materially:

1. Update the kit file's `§Changelog` with the version + date + change summary.
2. Update `../README.md` top-level kit registry if the client-ask quote or pitch price changes.
3. Update `README.md` in this folder (the Registry table + this strategic view) if a new kit is added or a kit is deprecated.
4. Update `../../F369_CICD_Template/prompt_templates/partials/README.md` Canonical Registry if the kit introduced new canonical partials.
5. Commit with `[Kit v<N.N>]` or `[Design]` prefix in the commit message for traceability.

---

## Related

- [`../README.md`](../README.md) — kit navigation + decision tree + composition patterns (engineering-facing)
- [`../../README.md`](../../README.md) — top-level F369 library index
- Per-kit design docs: [`deep-research-agent.md`](./deep-research-agent.md), [`acoustic-fault-diagnostic-agent.md`](./acoustic-fault-diagnostic-agent.md), [`ai-native-lakehouse.md`](./ai-native-lakehouse.md)
- Companion partial library: [`../../../F369_CICD_Template/prompt_templates/partials/README.md`](../../../F369_CICD_Template/prompt_templates/partials/README.md)
