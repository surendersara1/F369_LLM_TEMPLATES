<!-- Business-First Kit Standard | v1.0 | Last-updated: 2026-04-23 -->

# Business-First Kit Standard — v1.0

**Status:** MANDATORY for every new kit authored after 2026-04-23.
**Reference implementation:** [`kits/qualitative-research-audio-analytics.md`](../qualitative-research-audio-analytics.md) (v2.0+).
**Author:** SSS_AGENT / NorthBay Solutions / 369Forecast Elite Capital.

---

## Why this standard exists

The original kit model (v1.x) was **technology-first**: each kit started with "here's an AWS capability (AgentCore, S3 Vectors, Bedrock), here's how to wire it up, here's a thin domain skin on top." That produced kits that demo well to AWS architects and land flat with the actual buyer (CFO, CMO, VP Research, COO).

The Research America Inc. engagement (2026-04) exposed how much depth a real product needs to land:

- 40+ domain entities (`Verbatim.pinned_report_ids`, `Study.low_confidence_flags`, `CostLedgerEntry.study_id`)
- Persona-specific workflows (analyst pins → moderator approves → report builder → client delivery)
- Compliance / P&L concerns that pick the tech (7-year audit retention picks DynamoDB write-once + Object-Lock; per-study cost attribution picks Cost Explorer tagging; prompt A/B picks SSM versioning)
- A frontend that speaks the buyer's language (Verbatim, Discussion Guide, MetrixMatrix, SOC 2 — not Lambda, S3, Step Functions)

**The reordering this standard enforces:** business outcome → personas & workflows → frontend reference → AWS architecture → delivery plan. The first three layers block on each other. **You cannot ship a kit until §1, §2, §3 are written.**

---

## What a kit IS and what it is NOT

A kit IS:
- A **2-week consulting engagement playbook** that produces a real, deployable product
- A **buyer-language artifact** — readable by the client's CFO/CMO/VP Research without translation
- A **durable sales asset** that retains value across multiple engagements
- A **technology choice that flows from business need**, not the inverse

A kit is NOT:
- An AWS capability demo
- A "here's how to use Bedrock" tutorial
- A generic agent template
- An infrastructure-only blueprint

A kit MUST have a frontend reference spec. **Every kit. No exceptions.** A kit without a frontend spec is an architecture document, not a kit.

---

## The 5 mandatory sections

Every kit file under `kits/<kit-name>.md` must contain all five sections in this order. Optional sections (parameter matrix, partial reference, golden tests, etc.) come after §5.

### §1 — Business outcome (the why)

Required subsections:

- **§1.1 — The buyer.** Who signs the check? Title (CFO / CMO / VP Research / COO / Chief Risk Officer). Their typical company stage (PE-backed mid-market / public company business unit / Fortune 500 transformation office).
- **§1.2 — The P&L lever.** Specifically what does this kit move? Pick from: revenue growth (new pricing tier / new market) · cost reduction (labor displacement / infrastructure consolidation) · risk reduction (regulatory exposure / compliance penalties) · competitive defense (must-match-or-die feature) · customer retention (churn delta). State the dollar magnitude — a 6+ figure number per year per client.
- **§1.3 — The strategic context.** Why does the buyer care NOW? What's the trigger event in the market that made this a Q3-or-die problem? (e.g. "GenAI is eating their consulting margin", "new regulation kicks in 18 months", "PE owner demands EBITDA expansion before 2027 exit").
- **§1.4 — Anti-patterns specific to the domain.** What does the buyer reject? (e.g. "no auto-publish — editor must approve" / "no inline PII — always behind reveal control" / "no Slack notifications during market hours" / "no model output without confidence score"). At least 5 anti-patterns. These constrain the architecture.
- **§1.5 — The win condition.** What does success look like 90 days post-deployment? Specific measurable outcomes (e.g. "synthesis time per focus group drops from 12h to 3h, validated against 50 sampled studies" / "audit CSV export passes SOC 2 auditor review on first attempt"). Not aspirational language — measurable artifacts.

### §2 — Personas & workflows (the who)

Required subsections:

- **§2.1 — Persona matrix.** Table with columns: Persona name · Role group / scope · Primary daily workflow (3-7 steps) · Auth scope (full / own-tenant / read-only) · Buying influence (champion / user / blocker). Minimum 3 personas. The buyer (§1.1) must appear here.
- **§2.2 — Day-in-the-life narratives.** For the 2-3 highest-value personas, a 200-400 word narrative of a representative day. Mention the artifacts they produce, the systems they currently use, the friction they hit, where this product slots in. No generic "increases productivity" language — concrete moments ("9:14am Maria opens the tool, sees 3 sessions transcribed overnight, opens the Pfizer cardiology IDI...").
- **§2.3 — Business artifacts.** Catalog every artifact the personas produce or consume: report (PDF / DOCX / PPTX), audit CSV, pinned quote, delivered deck, signed-off recommendation, exported dataset, share link to client. For each: who produces, who consumes, what compliance/retention rules apply.
- **§2.4 — Cross-persona handoffs.** Where does work transition between personas? (e.g. "analyst submits draft report → moderator review queue → moderator approves → client viewer sees in shared link"). These are the workflow seams where state machines, notifications, and approval gates live.

### §3 — Frontend reference spec (what the user sees)

This section is what makes the kit demo-able to a non-technical buyer. The reference frontend is a contract — the engagement team builds something that conforms.

Required subsections:

- **§3.1 — Routes.** Every URL in the product. Table with columns: Path · Persona(s) · Purpose · Key data shown · Notable interactions · Mobile/desktop. Minimum 12 routes for a real kit. Persona-segregated by URL prefix (`/analyst/*`, `/admin/*`, `/client/*`).
- **§3.2 — Domain types.** Every business entity, with field-level detail. Use TypeScript-style declarations. **Field choices reveal business logic** — e.g. `Verbatim.pinned_report_ids: string[]` reveals there's a curation workflow; `Study.low_confidence_flags: number` reveals there's a hallucination-mitigation feature; `CostLedgerEntry.study_id?: string` reveals per-study cost attribution. Minimum 25 entities, average 8 fields each.
- **§3.3 — Visualizations.** The 5-10 viz components that make insights actionable. For each: name · what it shows · chart type · interaction model · who uses it. (e.g. "Sentiment Timeline · sentiment per 60-sec window · line chart with reference at 0 · click to seek transcript · analyst").
- **§3.4 — Standout business features.** The 10-20 features that represent business value beyond what a basic scaffold would have. For each: 2-3 sentences on what it does and why it differentiates. (See `qualitative-research-audio-analytics.md §3.4` for the worked example with 20 entries.)
- **§3.5 — Persona ribbon / branding.** How do you visually distinguish admin from analyst from client? Color tokens (HSL), persona badges, navigation patterns. Distinct mental models, no confusion.
- **§3.6 — Reference implementation pointer.** Link to the actual repo + commit SHA where this frontend lives. **A kit without a working reference frontend is incomplete.**

### §4 — AWS architecture (the how)

This section is intentionally fourth. It flows from the prior three. Every architectural choice should trace back to a §1, §2, or §3 driver.

Required subsections:

- **§4.1 — Architecture diagram.** ASCII or Mermaid. The standard 4-tier (browser → API → orchestration → data) is fine; the trick is **annotating which §1-§3 driver each component serves**. (e.g. "DDB `audit_log` table · 7-year TTL · drives §1.4 anti-pattern: SOC 2 audit ready" / "S3 Vectors `quotes` index · drives §3.4 cross-study search").
- **§4.2 — Stack inventory.** Table of CDK stacks. Columns: Stack name · Purpose · Driven by (§1.X / §2.X / §3.X) · Key partials referenced.
- **§4.3 — Data model.** Table of every persistence target (DDB tables, RDS tables, S3 buckets, S3 Vector indexes). Columns: Store · Table/bucket · Key/PK design · Driven by (§3.2 entity).
- **§4.4 — Architecture non-negotiables.** The 5-10 rules generated code MUST pass — both the global ones (`LAYER_BACKEND_LAMBDA §4.1`) and the kit-specific ones (e.g. "every Bedrock call wraps cost-tracking decorator" / "PII redaction runs before embedding").
- **§4.5 — Partial reference.** Which existing partials this kit composes. Which new partials this kit surfaces (the post-engagement extraction list).

### §5 — Delivery plan (the when)

Required subsections:

- **§5.1 — Engagement shape.** Total weeks · # devs · # designers · pricing range · payment milestones.
- **§5.2 — Day-by-day execution.** Day 1 through Day N, with: template(s) to load · partials Claude must load · expected deliverable at end of day · gating criteria to advance.
- **§5.3 — Phase 1 vs Phase 2.** What ships in this engagement vs deferred. Phase 2 list creates the next engagement.
- **§5.4 — Deliverables checklist.** End-of-engagement checklist (~20-30 items) the engagement team runs before sign-off.
- **§5.5 — Golden-set tests.** Nightly regression assertions with pass thresholds (latency, accuracy, cost, recall@k). Failing any = pager alert.
- **§5.6 — Handover & go-live runbook.** What documents land with the client. UAT script. Production cutover plan.

---

## Optional sections (encouraged, not required)

- **§6 — Kit-wide parameters.** Fill-once parameter block for templating (PROJECT_NAME, AWS_REGION, model IDs, scale knobs, etc.). 15-50 parameters is typical. (See current kits — every v1.x kit has this block; the standard keeps it.)
- **§7 — Bedrock prompt library.** Per-vertical prompt template list with file paths under `cdk/lambda/bedrock_analysis/prompts/`.
- **§8 — Composition patterns.** Which other kits compose with this one (see `kits/README.md §Kit composition patterns`).
- **§9 — Pricing & ROI math.** The numbers that justify the engagement spend. Show your work — labor displacement math, infrastructure-cost math, payback period.
- **§10 — Competitive positioning.** Which incumbents this displaces. Where this loses to incumbents. Demo-day anchor stories.
- **§11 — Changelog.** Version history.

---

## Authoring sequence (do not skip the order)

A kit author MUST follow this sequence. Reverse-order writing produces tech-first kits — the failure mode this standard exists to prevent.

1. **Buyer interview (or proxy).** Get on a call with someone in the buyer role. If unavailable, find a published case study / job posting / earnings transcript that exposes the buyer's priorities. Capture verbatim language. Write §1 from this.
2. **Persona workflow mapping.** Sketch the 3-5 personas. Write the day-in-the-life narratives BEFORE thinking about screens. Write §2 from this.
3. **Frontend reference spec.** Sketch the routes. List every domain entity with fields. List every visualization. Write §3.
4. **Reference frontend (working code).** Build the actual SPA. Field-tune the domain types as build reveals friction. The reference frontend lives in a sibling repo. **You cannot publish §3 without a working reference frontend.**
5. **Architecture from constraints.** Now read §1.4 anti-patterns + §3.2 domain types + §3.4 standout features. The architecture is a function of these. Write §4.
6. **Delivery plan.** Day-by-day, partials referenced, deliverables, golden tests. Write §5.
7. **Optional sections.** §6-§11 as appropriate.
8. **Audit gate.** Before publishing, run the kit through the audit checklist below.

The "skip steps 1-3, write §4 first" path is FORBIDDEN. If you find yourself doing it, stop. Go interview a buyer.

---

## Audit gate — the kit cannot ship until ALL pass

Run this checklist before submitting a kit for inclusion in the registry. **Reviewer (a peer kit author or SSS_AGENT) signs off on each item.**

### §1 audit
- [ ] §1.1 names a specific buyer title, not "the customer" or "users"
- [ ] §1.2 cites a 6+ figure annual P&L number with the math derivation
- [ ] §1.3 explains why NOW (not generic "AI is hot")
- [ ] §1.4 lists ≥5 specific anti-patterns the buyer rejects
- [ ] §1.5 lists measurable 90-day outcomes (no aspirational language)

### §2 audit
- [ ] §2.1 has ≥3 personas, including the §1.1 buyer
- [ ] §2.2 has ≥2 day-in-the-life narratives (200-400 words each), with concrete moments and times
- [ ] §2.3 catalogs ≥5 business artifacts with producer/consumer/retention rules
- [ ] §2.4 calls out workflow handoffs / approval gates

### §3 audit
- [ ] §3.1 lists ≥12 routes with persona segregation
- [ ] §3.2 declares ≥25 domain entities with field-level detail
- [ ] §3.3 names ≥5 visualizations with interaction model
- [ ] §3.4 names ≥10 standout business features
- [ ] §3.6 links to a working reference frontend repo + commit SHA
- [ ] **The reference frontend renders all §3.1 routes with §3.2 mock data**

### §4 audit
- [ ] §4.1 annotates each architecture component with the §1-§3 driver it serves
- [ ] §4.2 lists every CDK stack with its driver
- [ ] §4.3 lists every persistence target tied to a §3.2 entity
- [ ] §4.4 includes the 5 `LAYER_BACKEND_LAMBDA §4.1` rules + ≥3 kit-specific rules
- [ ] §4.5 lists existing partials reused + new partials surfaced

### §5 audit
- [ ] §5.2 has Day-1-through-Day-N table with templates + partials per day
- [ ] §5.3 separates Phase 1 (this engagement) from Phase 2 (next engagement)
- [ ] §5.4 has ≥20 checklist items
- [ ] §5.5 has ≥5 nightly regression assertions with pass thresholds

### Cross-section audit
- [ ] Every §4 architectural choice traces back to a §1-§3 driver (no AWS-for-AWS-sake decisions)
- [ ] Every §3.2 domain field maps to a §4.3 persistence target
- [ ] Every §2.4 handoff has a corresponding §4 state-machine transition or notification rule

### Buyer-language audit
- [ ] The §1 + §2 sections can be read by the §1.1 buyer without translation
- [ ] The §3 section is demo-able to the §1.1 buyer (frontend shows their domain language)
- [ ] No §1 / §2 / §3 sentence starts with "AWS" or "Lambda" or "Bedrock" — those words appear in §4 and later

If any item fails, the kit goes back for revision. Do not publish.

---

## File layout convention

```
kits/
  _template/
    README.md                      ← this file (the standard)
    EXAMPLE_kit-skeleton.md        ← optional empty skeleton for new kits
  _design/
    <kit-name>.md                  ← strategic companion (business case, pricing, competitive positioning)
  <kit-name>.md                    ← the kit itself (the 5 sections + optional)
```

The `<kit-name>.md` file holds the §1-§5 mandatory sections. The `_design/<kit-name>.md` companion is for content too long for the kit itself — extended business case, persona UX wireframes, pricing payback math, competitive teardowns. Companion is optional but recommended for any kit pitching above $200K.

---

## Reference frontend convention

Every kit's §3.6 must point to a working reference frontend. Conventions:

- The reference frontend lives in a separate repo (NOT in `F369_LLM_TEMPLATES`), typically under `E:\NBS_<client_name>_regen\frontend\` for the canonical client engagement.
- The kit lists the exact commit SHA at time of publication, so future readers can reproduce.
- The frontend MUST follow the [Claude React Frontend Prompt Framework v2.0](../../frontend_prompt/Claude_React_Frontend_Prompt_Framework.md). Stack: React 18 + TS strict + Vite + Tailwind + shadcn/ui + React Query + RHF+Zod + react-router v6 + Zustand + date-fns + sonner + next-themes + recharts + lucide-react.
- The frontend MUST render against MOCK_MODE data (no backend dependency to demo). Mock data lives in `frontend/src/lib/mocks/*.ts` and matches §3.2 domain types verbatim.
- A 2-minute screen-record of the frontend running in MOCK_MODE should ship with the kit (in `kits/_design/<kit>-demo.mp4` or a hosted link).

---

## Migration plan for existing v1.x kits

The 6 kits authored before this standard (HR Interview Analyzer, RAG Chatbot, Deep Research, Acoustic Fault, AI-Native Lakehouse, Qualitative Research Audio Analytics) are **grandfathered as v1.x** and remain in the registry as-is.

**Retrofit policy:** Do NOT proactively retrofit. Boil-the-ocean retrofits will rot. Retrofit when **a client engagement lights up the kit** — i.e. when there's a paying engagement that needs the v2.0 depth. The retrofit is part of the engagement scope.

The Research America kit is the EXCEPTION — it gets retrofitted to v2.0 immediately, because it becomes the reference implementation for the standard.

Tracking: kits/README.md registry table gains a "Standard Version" column showing v1.x or v2.0 per kit.

---

## Why this is worth the extra effort

**Cost.** A v1.x kit took 1-2 days to author. A v2.0 kit takes 5-10 days because §1-§3 require buyer interviews + frontend reference build. **3-5x the effort upfront.**

**Payoff.** v1.x kits land as "interesting demos." v2.0 kits land as "this is exactly what we need" — because §1.1 buyer's exact language is in the kit. Pitch-to-close ratio improves materially. Engagement scope is clearer (§3 is a contract, not a wishlist). Phase 2 + Phase 3 follow-on engagements are pre-architected (§5.3).

**Durability.** v1.x kits decay fast — AWS service GA shifts make them stale within 6 months. v2.0 kits decay slowly — the §1-§3 layer is buyer reality, which doesn't change at AWS-release-cycle speed. The §4 layer can be refreshed without rewriting the kit.

**Repeatability.** A v2.0 kit sells to N clients in the same buyer category with minor §1.3 + §3.5 customization. The §3.2 domain types + §3.4 features carry across clients. v1.x kits had to be re-pitched per-client because the technology pitch had no buyer hook.

---

## Maintenance

**Owner of this standard:** SSS_AGENT.
**Review cadence:** quarterly. The standard is bumped (v1.0 → v1.1 → v2.0) when a new pattern surfaces from real engagements that warrants codifying.
**Change log:** at the bottom of this file.

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 1.0 | 2026-04-23 | Initial standard. Codifies the 5-section structure (business outcome → personas → frontend reference → AWS → delivery). Authoring sequence enforced (no §4-first writing). Audit gate with sign-off checklist. Reference implementation: `kits/qualitative-research-audio-analytics.md` v2.0. Triggered by Research America Inc. engagement (2026-04) which exposed the gap between technology-first kits and what real buyers respond to. |
