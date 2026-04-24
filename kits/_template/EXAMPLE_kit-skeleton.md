<!-- Kit Version: 0.1 (DRAFT) | Standard: Business-First v1.0 | Reference frontend SHA: <fill-in> -->

# Kit — <Kit Name> (<one-line domain pitch>)

> **Status:** DRAFT against [Business-First Kit Standard v1.0](_template/README.md). Not yet audit-passed.

**Client ask (verbatim from buyer):** "<paste the actual quote you got from a buyer interview, or a near-verbatim from a published case study / job posting>"
**Engagement:** N weeks · N devs · N designers · pitch range $<min>K–$<max>K
**Deliverable summary:** <2-3 sentence shape of what gets shipped — frontend repo + CDK stacks + runbook>

**Reference frontend:** [`<repo path>`](<URL or path>) @ commit `<sha>`

---

## §1 — Business outcome

### §1.1 Buyer
- **Title:** <e.g. VP Research / CFO / Chief Risk Officer>
- **Company stage:** <PE-backed mid-market / Fortune 500 BU / public company>
- **Org size signal:** <employee count band, revenue band>
- **Reports to:** <CEO / Board / etc.>

### §1.2 P&L lever
- **Lever type:** <revenue growth / cost reduction / risk reduction / competitive defense / retention>
- **Annual $ magnitude per client:** $<6+ figure number>
- **Math derivation:**
  ```
  <Show the math. e.g.:
   - Current state: 600 focus groups/mo × 12 hr synthesis × $150K/yr loaded = $X
   - Future state: 600 × 3 hr × same rate = $Y
   - Annual savings = (X - Y) × 12 = $Z>
  ```
- **Ratio sentence:** <e.g. "1 dollar of AWS spend displaces $80 of labour cost.">

### §1.3 Strategic context — why NOW
<2-4 paragraphs. What changed in the buyer's market in the last 6-18 months that makes this an urgent priority? Tie to a regulation date / competitor move / PE exit timeline / earnings pressure. Not generic "AI is hot.">

### §1.4 Anti-patterns specific to the domain
The buyer rejects:
- <anti-pattern 1 — e.g. "no auto-publish — editor must approve every report">
- <anti-pattern 2 — e.g. "no inline PII — always behind a reveal control with audit log">
- <anti-pattern 3>
- <anti-pattern 4>
- <anti-pattern 5>
- <(more if applicable)>

Each anti-pattern constrains the architecture. Trace these in §4 — at least one §4 design choice should cite each one.

### §1.5 Win condition (90 days post-deployment)
Success = ALL of:
- <measurable outcome 1 — e.g. "synthesis time per session drops from Xh to Yh, validated against 50 sampled studies">
- <measurable outcome 2 — e.g. "audit CSV export passes SOC 2 auditor review on first attempt">
- <measurable outcome 3 — e.g. "client viewer NPS ≥ 50 across N clients">
- <measurable outcome 4>

NOT aspirational language. Each outcome must be measurable with a concrete artifact.

---

## §2 — Personas & workflows

### §2.1 Persona matrix

| Persona | Role group / scope | Primary daily workflow (3-7 steps) | Auth scope | Buying influence |
|---|---|---|---|---|
| <e.g. Senior Analyst> | Analyst / own-tenant | 1) Open today's queue · 2) Review session · 3) Pin verbatims · 4) Draft report · 5) Submit for review | own-tenant read+write | user |
| <e.g. Research Director> | Moderator / own-tenant | 1) Approve queue · 2) Edit verbatims inline · 3) Sign off · 4) Deliver to client | own-tenant approve | champion |
| <e.g. CTO> | Admin / global | 1) Cost dashboard · 2) Audit CSV · 3) Pipeline health · 4) Prompt versioning | global read+write | buyer (§1.1) |
| <(more)> | | | | |

### §2.2 Day-in-the-life narratives

#### <Persona 1>
<200-400 words. Concrete moments. Times. Names of the systems they currently use. Where this product slots in. The artifacts they hand off.>

#### <Persona 2>
<200-400 words.>

### §2.3 Business artifacts

| Artifact | Producer | Consumer | Format | Retention | Compliance notes |
|---|---|---|---|---|---|
| <e.g. Final report> | analyst | client | DOCX/PPTX/PDF | 7 yr | client name redacted pre-publication |
| <e.g. Audit CSV> | admin (export) | auditor | CSV | append-only 7 yr | SOC 2 / GDPR Art. 17 |
| <e.g. Pinned verbatim> | analyst | report builder | JSON in DDB | retention follows session | PII-redacted speaker labels |
| <(more)> | | | | | |

### §2.4 Cross-persona handoffs

- **<handoff 1>:** <e.g. Analyst submits draft report → Moderator review queue → Moderator approves OR returns with comments → Final delivered to Client viewer via shared link>
- **<handoff 2>:** <e.g. Admin promotes new prompt version → Bedrock auto-test against 10 recent transcripts → Win-rate published → 1-click promote to active>
- **<(more)>**

Each handoff = a §4 state-machine transition or notification rule.

---

## §3 — Frontend reference spec

### §3.1 Routes

| Path | Persona(s) | Purpose | Key data shown | Notable interactions | Mobile |
|---|---|---|---|---|---|
| `/login` | all | Auth | — | role-aware redirect | yes |
| `/<persona-1>/<entity>` | <persona-1> | Master list | <list of entities + filters> | search · filter · pagination | yes |
| `/<persona-1>/<entity>/:id` | <persona-1> | Detail hub w/ N tabs | <core fields + tabs> | tab nav · drill-down | partial |
| `/<persona-1>/<entity>/:id/<sub>` | <persona-1> | Sub-view | <sub data> | <interactions> | partial |
| `/<persona-2>/<dashboard>` | <persona-2> | Dashboard | <KPIs + charts> | refresh · drill | desktop |
| `/admin/<area>` | admin | Admin tool | <admin data> | <admin actions> | desktop |
| `/admin/audit` | admin | Audit log | <events table> | filter · CSV export | desktop |
| `/admin/cost` | admin | Cost ledger | <cost rollups> | filter · drill-down | desktop |
| `/admin/health` | admin | System health | <SLA cards> | — | desktop |
| `/admin/prompts` | admin | Prompt versioning | <prompts + history> | edit · A/B · rollback | desktop |
| `/<persona-X>/notifications` | <persona-X> | Inbox | <notif list> | filter · mark-read | yes |
| `/403` / `/404` / `/login` | all | Errors | — | — | yes |

Minimum 12 rows. Persona segregated by URL prefix. Admin routes locked to `/admin/*`.

### §3.2 Domain types

```typescript
// Top-level entities (≥25 total across the kit)

export interface <Entity1> {
  id: string;
  // ≥8 fields. Field choices reveal business logic.
  // Example: pinned_report_ids: string[]  // reveals curation workflow
  // Example: low_confidence_flags: number  // reveals hallucination-mitigation feature
  // Example: <field>: <type>;  // reveals <business meaning>
  ...
}

export interface <Entity2> { ... }
// ...continue to ≥25 entities
```

Group entities into sections: **Identity / Tenancy** · **Core domain** · **Workflow state** · **Insights / LLM output** · **Compliance / Audit** · **Operations / Cost**.

### §3.3 Visualizations

| Component | What it shows | Chart type | Library | Interaction | Persona |
|---|---|---|---|---|---|
| <e.g. SentimentTimeline> | sentiment per N-sec window | line w/ ref at 0 | recharts | click → seek transcript | analyst |
| <e.g. TopicBubbles> | topic mentions × sentiment | custom SVG bubbles | inline | hover tooltip · click drill | analyst |
| <e.g. SpeakerBreakdown> | talk % per speaker × sentiment color | stacked bar | recharts | hover | analyst |
| <e.g. CostLedger trend> | $/day × service | multi-line | recharts | filter date range | admin |
| <e.g. PipelineHealth> | per-stage in-flight + p95 | metric cards | custom | color thresholds | admin |
| <(more)> | | | | | |

Minimum 5. The viz is what makes insight actionable.

### §3.4 Standout business features

The N features that represent meaningful business value beyond a basic scaffold. For each: 2-3 sentences on what it does and why it differentiates.

1. **<feature 1>** — <description>. <why it differentiates>.
2. **<feature 2>** — <description>. <why it differentiates>.
3. **<feature 3>** — <description>. <why it differentiates>.
4. <continue to ≥10>

These map to §3.2 domain fields and §3.3 visualizations. They're the "wow moments" in the demo.

### §3.5 Persona ribbon / branding

- **Primary brand:** HSL `<H S% L%>` (e.g. `221 83% 53%` deep blue)
- **<Persona 1> ribbon:** HSL `<...>` (e.g. amber)
- **<Persona 2> ribbon:** HSL `<...>` (e.g. cyan)
- **Persona badge placement:** <header right of logo>
- **Navigation pattern:** <e.g. persona-segregated sidebar; admin-vs-analyst share top header but different left rail>
- **Typography:** <e.g. Inter UI · JetBrains Mono code · Source Serif 4 published>

### §3.6 Reference implementation pointer

- **Repo:** `<path or URL>`
- **Commit at kit publication:** `<sha>`
- **Run command:** `pnpm dev` from `<repo>/frontend/`
- **MOCK_MODE flag:** auto-on when no `VITE_*_API` env vars set
- **Demo script:** `<path to demo doc / link to screen-record>`

The reference frontend MUST render every §3.1 route with §3.2 mock data. CI gate: a reference-render test runs before merge.

---

## §4 — AWS architecture

### §4.1 Architecture diagram

```
<ASCII or Mermaid diagram>

Each component annotated with the §1-§3 driver it serves:
  - DDB audit_log table  ← drives §1.4 anti-pattern (SOC 2 audit ready)
  - S3 Vectors quotes idx ← drives §3.4 cross-study search
  - SFN AudioWorkflow    ← drives §2.2 analyst persona day-in-the-life step 3
```

### §4.2 Stack inventory

| Stack | Purpose | Driven by | Key partials |
|---|---|---|---|
| <NetworkStack> | VPC + PrivateLink | global | `LAYER_NETWORKING` |
| <SecurityStack> | KMS + IAM | §1.4 (#3) | `LAYER_SECURITY` |
| <DataStack> | RDS + DDB tables | §3.2 entities | `DATA_AURORA_SERVERLESS_V2`, `LAYER_DATA` |
| <OrchestrationStack> | Step Functions | §2.4 handoff #1 | `WORKFLOW_STEP_FUNCTIONS` |
| <(N stacks total)> | | | |

### §4.3 Data model

| Store | Table / bucket / index | PK / key design | Driven by §3.2 entity |
|---|---|---|---|
| RDS Aurora | sessions | (tenant_id, session_id) | Session |
| RDS Aurora | verbatims | (tenant_id, verbatim_id) | Verbatim |
| DDB | audit_log | tenant#date, user#timestamp | Activity |
| DDB | cost_ledger | tenant#date, job_id | CostLedgerEntry |
| S3 Vectors | quotes_idx | tenant_id metadata | Verbatim (semantic) |
| S3 | audio-raw bucket | uploads/{tenant}/{study}/{session}/* | Session.media_url |
| <(more)> | | | |

### §4.4 Architecture non-negotiables

Global rules from `LAYER_BACKEND_LAMBDA §4.1`:
1. Lambda asset paths use `Path(__file__).resolve().parents[N] / "..."`.
2. Cross-stack resource access is identity-side only.
3. Cross-stack EventBridge → Lambda uses L1 `events.CfnRule`.
4. Bucket + CloudFront OAC live in the same stack.
5. KMS ARNs cross-stack are strings via SSM.

Kit-specific additions (each one cites its driver):
6. **<rule>** ← drives §1.4 anti-pattern <which one>
7. **<rule>** ← drives §3.4 feature <which one>
8. **<rule>** ← drives §2.4 handoff <which one>
9. <(more)>

### §4.5 Partial reference

**Existing partials (load these in Claude prompts):**
- `LAYER_BACKEND_LAMBDA`, `LAYER_NETWORKING`, `LAYER_SECURITY`, `LAYER_DATA`, `LAYER_API`, `LAYER_FRONTEND`, `LAYER_OBSERVABILITY`
- <(kit-specific partials)>

**New partials this kit surfaces** (extract post-engagement per Canonical-Copy Rule):
1. `<PARTIAL_NEW_1>` — <one-line description>
2. `<PARTIAL_NEW_2>` — <one-line description>
3. <(more)>

---

## §5 — Delivery plan

### §5.1 Engagement shape
- **Total:** N weeks · N devs · N designers
- **Pricing:** $<min>K – $<max>K
- **Milestones:** 50% on kickoff · 25% on Week-1 demo · 25% on UAT sign-off

### §5.2 Day-by-day execution

| Day | Template(s) | Partials Claude must load | Deliverable | Gating criteria |
|---|---|---|---|---|
| 1 | <iac/02 ...> | <list> | <CDK scaffold synth-clean> | <criteria> |
| 2 | ... | ... | ... | ... |
| ... | ... | ... | ... | ... |
| N | ... | ... | ... | ... |

### §5.3 Phase 1 vs Phase 2

**Phase 1 (this engagement):**
- <feature>
- <feature>
- ...

**Phase 2 (next engagement, $<...>K):**
- <deferred feature>
- <deferred feature>
- ...

### §5.4 Deliverables checklist

End-of-engagement (~20-30 items):
- [ ] CDK app, all stacks synth-clean
- [ ] <kit-specific deliverable>
- [ ] <kit-specific deliverable>
- [ ] Reference frontend deployed + MOCK_MODE demo recorded
- [ ] UAT script executed + outcomes doc
- [ ] Runbook + deployment guide
- [ ] Handover session with client team
- [ ] (more)

### §5.5 Golden-set tests

| Assertion | Metric | Pass threshold |
|---|---|---|
| <e.g. End-to-end latency> | p95 SFN duration | <threshold> |
| <e.g. Quote extraction recall> | recall@30 vs human golden set | >X% |
| <e.g. PII leak rate> | recall on synthetic-PII | >99% |
| <(≥5 total)> | | |

Failing any = pager + deployment freeze.

### §5.6 Handover & go-live runbook

- **Documents handed over:** runbook · ADRs · deployment guide · UAT outcomes · client-onboarding playbook
- **UAT script:** at `<path>`
- **Production cutover plan:** <staging promotion gates · backout plan · rollback SLA>

---

## §6 — Kit-wide parameters (optional)

```
PROJECT_NAME:                <kit>-{client_slug}
AWS_REGION:                  us-east-1
AWS_ACCOUNT_ID:              <12-digit>
ENV:                         dev | stage | prod
CLIENT_SLUG:                 <slug>
TARGET_LANGUAGE:             python
COMPLIANCE_STANDARD:         SOC2 | HIPAA | PCI_DSS | NONE
AUTH:                        cognito_user_pool
SUPERVISOR_MODEL_ID:         <claude-sonnet-4-7 / etc.>
MAX_COST_PER_RUN_USD:        <$0.10–$5.00>
DOMAIN_PACK:                 <enum>
<(15-50 parameters total)>
```

---

## §7 — Bedrock prompt library (optional, if LLM-heavy)

Templates at `cdk/lambda/<analyzer>/prompts/v1/`:
- `<prompt-1>.txt` — <purpose>
- <(more)>

Per-vertical overrides at `prompts/v1/<vertical>/<prompt>.txt`.

---

## §8 — Composition patterns (optional)

How this kit composes with other kits — see [`kits/README.md §Kit composition patterns`](../README.md#kit-composition-patterns).

---

## §9 — Pricing & ROI math (optional)

<Detailed payback math, labour displacement, infrastructure-cost model. See `_design/<kit>.md` for the full version.>

---

## §10 — Competitive positioning (optional)

| Incumbent | What they do | Where this wins | Where this loses |
|---|---|---|---|
| <competitor 1> | <what they do> | <where this wins> | <where they win> |
| <competitor 2> | ... | ... | ... |

---

## §11 — Changelog

| Version | Date | Change |
|---|---|---|
| 0.1 | <date> | Initial draft against Business-First Standard v1.0. |
