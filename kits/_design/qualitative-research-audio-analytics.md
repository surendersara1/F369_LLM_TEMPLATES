<!-- Design Doc | Companion to: kits/qualitative-research-audio-analytics.md | Last-updated: 2026-04-23 -->

# Design — Qualitative Research Audio Analytics Kit

> **Purpose of this doc.** The "why" behind Kit 6's shape — the business case for an AI-enabled market research platform, the UX design for two personas (Ops Admin + Analyst), three business-flow examples with pricing, competitive positioning vs. Qualtrics / Dscout / SPSS, and the Phase 2 / Phase 3 evolution roadmap.

---

## 1. The business case

### Research America's strategic context

Research America is a traditional market research firm — 30M panel members, 35 years of history, 14 acquired companies, 16+ industry verticals. **They do not currently claim AI or advanced-technology capabilities** on their public website. Their differentiation today is human expertise + methodological flexibility ("boutique of boutiques").

Their competitive pressure comes from two directions:

- **Below:** Tech-first research platforms (Qualtrics, Dscout, User Interviews, Remesh) that sell DIY research tools to in-house insights teams at Research America's clients. These platforms are eating the entry-level and mid-market of Research America's book.
- **Above:** Strategy consultancies (Bain, McKinsey, BCG) that own the top-tier insights work and increasingly build their own AI research tools in-house.

**The existential question:** if an in-house insights team at a pharma company can run focus groups themselves via Dscout + have AI summarise the transcripts, why pay Research America's full-service rate?

**The answer Research America is betting on with this engagement:** Research America is positioning to become the "AI-augmented full-service research firm" — combining their human methodological expertise with AI that's better than Dscout's (because it's tuned for deep qualitative work, not just screener surveys). The Intelligent Audio Analytics POC is the foundation. If it works, the next phase is licensing the platform to CLIENTS as a SaaS product — turning Research America from a services firm into a services + software firm.

### The economics

Current state — manual synthesis:
- Senior researcher salary: ~$150K loaded
- Hours per focus group synthesis: ~12 hours (transcription review + theme extraction + report drafting)
- Focus groups per month (firm-wide, per SOW): ~600 (plus 6,400 other audio = IDIs + surveys)
- Total manual hours: 600 × 12 + 6,400 × 6 ≈ 45,600 hours/month
- Labour cost attribution (at loaded rates): ~$2.3M/month of researcher + analyst labour just on synthesis

Future state — platform-augmented:
- Time per focus group synthesis drops to ~3 hours (AI drafts; researcher reviews + polishes)
- Effective labour savings: **$1.6M/month** on synthesis alone

AWS cost (per SOW projection): ~$19,700/month at 7K recordings. Ratio: **1 dollar of AWS spend displaces ~$80 of labour cost.** That's an extraordinary ROI — it's the headline number for the pitch deck.

This is why the kit is worth building even at 2-week POC scope: the business case for the platform is so strong that the POC's only job is to prove the technology works. Once proven, Phase 2 + Phase 3 investment is a rounding error compared to the labour savings.

### Why this is a kit, not a one-off engagement

Three reasons:

1. **Second-client repeatability.** Qualitative research audio analytics is a generic workload. Once Kit 6 is proven at Research America, it sells to other research firms (Ipsos, GfK, Kantar, mid-tier boutiques) + to in-house insights teams at large brands. Estimated addressable market: 200+ research firms + 500+ in-house insights teams × $150-500K per engagement.

2. **Partial-library growth.** The kit surfaces 5 new partials (Transcribe pipeline, quote mining, report drafting, cost tracking, audio-seek). Once extracted + audited, future kits (healthcare clinical-note audio analytics, legal-deposition analytics, consumer-call-center QA) reuse these.

3. **Pricing leverage.** A kit sells for a fixed price ($250-500K) with a known delivery cost. An ad-hoc engagement is priced by hours + time-and-materials. Kit-based pricing has higher margin AND is easier to pitch because the price is decoupled from delivery uncertainty.

---

## 2. The two personas — UX design

Two personas share one platform, two completely different experiences.

### Persona 1 — Ops Admin (Platform operator)

**Who:** CTO / VP Engineering / Director of Research Operations. 1-2 per Research America. Same person typically owns the AWS account relationship.

**What they care about:** Is the platform healthy, cost-efficient, compliant, and utilized?

**What they reject:** Any UX that looks like an analyst tool. They want "operations cockpit" vibes — CloudWatch-esque but better. Dense, factual, realtime.

#### Admin home — /admin/ops (realtime ops cockpit)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  [Research America — Admin Console]        alice@research    🔔 3  ⚙️  🚪  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Operations Cockpit                                 Auto-refresh: 5s ⏸      │
│                                                                              │
│  ┌─────────────────────────┐  ┌─────────────────────────┐                   │
│  │  Jobs in flight          │  │  Throughput (rolling 1h) │                  │
│  │                          │  │                          │                  │
│  │  Queued:        12       │  │    ▁▂▃▅▆█▇▆▅▃▂ 42/hr    │                  │
│  │  Transcribing:  8        │  │    avg: 38  peak: 54    │                  │
│  │  Analyzing:     3        │  │                          │                  │
│  │  Complete:    247 today  │  │    Est time to clear: 4m │                  │
│  │  Failed:        1        │  │                          │                  │
│  └─────────────────────────┘  └─────────────────────────┘                   │
│                                                                              │
│  ┌─────────────────────────┐  ┌─────────────────────────┐                   │
│  │  Failure breakdown       │  │  SLA breach watch        │                  │
│  │                          │  │                          │                  │
│  │  Transcribe timeout: 1   │  │  🔴 0 jobs > 4h pending  │                  │
│  │  Bedrock throttle:   0   │  │  🟡 2 jobs > 2h pending  │                  │
│  │  Corrupt audio:      0   │  │                          │                  │
│  │  PII filter fail:    0   │  │  [View detail]           │                  │
│  └─────────────────────────┘  └─────────────────────────┘                   │
│                                                                              │
│  Recent jobs (last 50)                                                      │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ Job ID    Client         Project             Status    Analyst  Age  │  │
│  │ j-8a71... pharma-acme    q3-diligence-fg     ✓ Done   bob     12m   │  │
│  │ j-8a70... retail-init    shopper-journey     ⟳ Analy. carol   18m   │  │
│  │ j-8a6f... auto-newco     test-drive-ethno    ⟳ Trans. carol   21m   │  │
│  │ j-8a6e... pharma-acme    q3-diligence-fg     ⚠ Fail   bob     45m   │  │
│  │ ...                                                                    │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  [Ops cockpit] [Cost & usage] [Quality & compliance] [Users] [Tenants]      │
└─────────────────────────────────────────────────────────────────────────────┘
```

Design principles:
- **Auto-refresh every 5s** — DDB point-reads, ~20ms; fast enough to look realtime without burning backend cost
- **Failure is first-class** — failure breakdown is a dashboard, not buried in logs
- **SLA breach watch** is the alarming system — 2h yellow / 4h red, on mouse-hover → show which jobs + why slow
- **Recent jobs table** is the operator's audit trail — every row clickable → full job history + retry button

#### Admin cost — /admin/cost

Cost per client / per project / per analyst, grouped by day + month.

Key chart: **daily spend vs. budget** line chart with cost-anomaly overlay. The kit sets `COST_ANOMALY_ALARM_PCT=30` which means a day that spends > 30% above the rolling 7-day average triggers an alarm + shows a red band in the chart.

Tables:
- Top 10 clients by monthly spend (clickable → client detail)
- Top 10 projects by spend (clickable → project detail)
- Cost per job type (focus_group vs IDI vs survey)
- Bedrock tokens + Transcribe minutes consumed (unit cost × volume)

**This page answers billing questions directly** — Research America can bill their clients pass-through AWS cost via this view.

#### Admin quality — /admin/quality

- **Transcription quality** (WER on sampled reference audio) — weekly trend + per-language breakdown
- **PII redaction hit rate** — how many recordings had PII flagged + redacted (high number = something wrong with participant consent forms, not just a tech metric)
- **Data retention watchdog** — recordings approaching client retention boundary (auto-expire in N days)
- **Cross-tenant isolation check** — continuous probe that attempts cross-tenant read; alerts if any succeed

#### Admin users + tenants

Standard Cognito + tenant management UI. Skip detail — commodity.

### Persona 2 — Analyst (Research producer)

**Who:** Senior market research analyst / research director / moderator. 40-80 of them at Research America firm-wide.

**What they care about:** My projects, my deliverables, my client relationships. Speed of insight production.

**What they reject:** Ops-cockpit aesthetic. They want a **professional research workspace** — think Notion meets Figma meets a media player.

#### Analyst home — /analyst/projects

Three sections:

1. **My active projects** — cards with client + study type + deadline + completion bar + recent activity badge
2. **Shared with me** — projects other analysts invited me into
3. **Recent activity feed** — "New transcript ready: pharma-acme fg #3", "Bob added 4 quotes to retail shopper-journey", "Client viewer alice@pharma opened report", etc.

Top bar: **quick-search** (cross-project semantic search — our shocking feature).

#### Analyst project detail — /analyst/projects/:id

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Pharma Acme — Q3 Diligence Focus Groups                 ✏  ⬇ Export  ⋯   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  8 sessions · 6 complete · 2 in progress  ·  Client: Acme Pharma  ·  Due 4/30│
│                                                                              │
│  ┌── Sessions (8) ─────────────────────────────────────────────────────┐  │
│  │                                                                        │  │
│  │  ○ FG-1  moderator: Sarah   Oct 15  2h 3m   6 resp  ✓ Analyzed       │  │
│  │  ○ FG-2  moderator: Sarah   Oct 15  1h 57m  6 resp  ✓ Analyzed       │  │
│  │  ○ FG-3  moderator: Bob     Oct 16  2h 12m  6 resp  ⟳ Analyzing      │  │
│  │  ○ FG-4  moderator: Bob     Oct 16  ---           ⬚ Uploading       │  │
│  │  ...                                                                    │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  ┌── Cross-session synthesis (auto-drafted) ─────────────────────────┐    │
│  │                                                                      │    │
│  │  Executive Summary (updates as sessions complete)                   │    │
│  │  "Across 6 focus groups with healthcare providers, respondents     │    │
│  │   expressed strong positive sentiment (+0.67 avg) about the         │    │
│  │   clinical efficacy of Acme's Drug-X prototype, but consistently   │    │
│  │   raised pricing concerns (mentioned in 42/48 respondents)..."      │    │
│  │                                                                      │    │
│  │  Top themes (ranked by respondent freq)                             │    │
│  │  1. Clinical efficacy trust           46/48 respondents  +0.82     │    │
│  │  2. Price sensitivity                 42/48 respondents  -0.45     │    │
│  │  3. Payor coverage uncertainty        35/48 respondents  -0.31     │    │
│  │  4. Side-effect tolerance             31/48 respondents  +0.22     │    │
│  │  ...                                                                  │    │
│  │                                                                      │    │
│  │  [View draft report] [Edit themes] [Export to DOCX] [Client share] │    │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌── Competitive mentions (auto) ────────────────────────────────────┐    │
│  │                                                                      │    │
│  │  Rival brand "Globex Y" mentioned 23× across 6 sessions            │    │
│  │  Sentiment: +0.12 (neutral-positive) | most common: "similar      │    │
│  │  efficacy but better co-pay". Quotes →                              │    │
│  └──────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

Design principles:
- **Auto-drafted synthesis is the hero** — the analyst's first impression is "the platform wrote 80% of my report before I even opened the project"
- **Themes are interactive** — click a theme → drill into all quotes across all sessions that reference it
- **Competitive mentions are a lead-in to Phase 2** — in POC, static; in Phase 2, realtime alerts to client

#### Analyst session detail — /analyst/session/:id (the hero page)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  FG-3 · 2h 12m · 6 respondents · Analyzed 12m ago           ⬇ Export  ⋯   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  [Play ▶]  ▁▂▃▁▂▅▆▇▇▆▃▁▂▁▁▂▂▁▁▂▅▇▆▇▅▃▂▁▁▂▂▁  0:00 ─────●──── 2:12:34   │
│                                                                              │
│  Tabs:  Transcript   Summary   Quotes   Themes   Codes   Audit              │
│                                                                              │
│  ┌── Transcript (speaker-labeled) ──────────────────────────────────────┐   │
│  │                                                                        │   │
│  │  [0:00:12] Moderator (Sarah): "Thanks for joining us tonight. Let's  │   │
│  │  start with how you feel about your current treatment options..."    │   │
│  │                                                                        │   │
│  │  [0:00:48] Respondent A (HCP, age 45-54): "Honestly, I feel like we │   │
│  │  haven't had a real breakthrough in this category in five years..."  │   │
│  │  ⭐ quote  🏷 sentiment:-0.42  🔗 theme:clinical-frustration         │   │
│  │                                                                        │   │
│  │  [0:01:23] Respondent B (HCP, age 35-44): "The efficacy of the      │   │
│  │  current standard-of-care is about what you'd expect, but side      │   │
│  │  effects are why I hesitate to prescribe more aggressively..."      │   │
│  │  ⭐ quote  🏷 sentiment:-0.15  🔗 theme:side-effect-tolerance       │   │
│  │                                                                        │   │
│  │  ...                                                                    │   │
│  │                                                                        │   │
│  │  [0:04:22] Moderator (Sarah): "Can you tell me about your reaction  │   │
│  │  to the Drug-X clinical-efficacy profile we showed you?"             │   │
│  │                                                                        │   │
│  │  [0:04:51] Respondent A: "This is the first thing I've seen in a    │   │
│  │  long time that genuinely excites me as a clinician. The remission  │   │
│  │  data is striking."                                                   │   │
│  │  ⭐ quote  🏷 sentiment:+0.92  🔗 theme:clinical-efficacy-trust     │   │
│  │  [▶ play from here]                                                   │   │
│  │                                                                        │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

**The single most-demoed feature:** click "▶ play from here" on any line of transcript → audio player jumps to that timestamp. Click a `⭐ quote` badge → opens the quote detail pane with the surrounding 30-second context playable.

Implementation detail: the backend produces per-quote presigned S3 URLs with `Range` headers restricted to the byte range of that ±15-second clip. Leaking one quote's URL does NOT leak the whole recording.

#### Analyst search — /analyst/search (the other hero)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Cross-study semantic search                                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ▣  Tell me what respondents said about price sensitivity in pharma         │
│     studies in 2025                                                         │
│  [Search]    Filter: [Client: all ▾]  [Vertical: pharma ▾]  [Year: 2025 ▾] │
│                                                                              │
│                                                                              │
│  Found 87 quotes across 34 sessions in 12 projects. Top results:            │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ 1. Respondent B (HCP) — Acme Pharma Q2 2025 FG-4                    │  │
│  │    "Look, the efficacy is there, but at $4500 a month, my patients  │  │
│  │    are going to skip doses. That's the reality."                    │  │
│  │    sentiment: -0.72 | theme: price-sensitivity | ▶ listen           │  │
│  │                                                                        │  │
│  │ 2. Respondent A (HCP) — Acme Pharma Q3 2025 IDI-7                   │  │
│  │    "I'd prescribe it more if the co-pay wasn't a barrier. Right now │  │
│  │    I'm basically reserving it for patients with exceptional..."      │  │
│  │    sentiment: -0.56 | theme: payor-coverage | ▶ listen              │  │
│  │                                                                        │  │
│  │ ... (85 more)                                                          │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  [Save as "pharma price sensitivity 2025"] [Export quotes to DOCX]          │
└─────────────────────────────────────────────────────────────────────────────┘
```

This is a capability **Qualtrics does not have** out of the box. Dscout has "transcript search" but it's keyword-based. Our vector-based search finds semantically-similar quotes even when the phrasing differs ("out-of-pocket burden" matches a query for "price sensitivity"). Tied to the audit-validated `PATTERN_CATALOG_EMBEDDINGS` pattern applied at quote level.

---

## 3. Three business-flow examples

### 3.1 Pharma focus group — Acme Pharma Q3 Drug-X diligence

**Setup:** Acme Pharma is evaluating their Drug-X clinical trial positioning. They hire Research America to run **8 focus groups** with HCPs (2h each, 6 respondents) + **20 IDIs** with patients who've tried competing drugs (45min each).

**Traditional workflow (without platform):**
- 8 × 2h + 20 × 45min = **31 hours of audio**
- Transcription services: 3-5 days, $2-4K cost
- Manual synthesis: 8 × 12h + 20 × 4h = **176 hours of analyst time** → $15K labour at loaded rate
- Report drafting: ~40 hours → $3.5K
- **Total analyst + transcription: ~216 hours / 2-3 weeks / ~$25K direct cost**

**Platform workflow:**
- Upload 28 audio files (automated batch via analyst portal)
- Platform auto-processes: transcription ~15 min, Bedrock analysis ~5 min per file, **all 28 files done in ~45 min**
- Analyst reviews + edits draft report: ~20 hours → $1.8K
- Platform auto-generates DOCX + PPTX deliverables
- **Total analyst time: ~20 hours / 3 days / ~$2.5K labour + ~$85 AWS cost**

**Savings per project:** $25K → $2.5K = **$22.5K saved.** 8x more projects per analyst-month → Research America can scale to more Acme-size clients without headcount growth.

**Pitch price for this engagement type (when Research America RE-sells the platform capability to Acme):** $250-400K per year per client for a "premium-tier" analytics service. Research America's margin: 60%+.

### 3.2 Retail consumer insights — Initech shopper-journey ethnography

**Setup:** Initech (large retailer) wants to redesign their in-store experience. Research America runs a **5-site mobile ethnography** (25 shoppers × 45 min accompanied-shop recordings), plus **3 focus groups** post-field (2h each, 8 respondents), plus **48 short IDIs** (15min each, phone-based follow-ups).

**Why platform matters:** 48 IDIs × 15min = 12 hours of audio that's uneconomic to fully-synthesize manually. Pre-platform, Research America samples 20% + assumes the rest. Platform processes 100% → deeper insights, defensible methodology.

**Competitive capability:** *cross-study theme detection* — Initech ran similar studies in 2024 + 2025 with Research America. Analyst queries the corpus: "what changed in shopper sentiment about our loyalty program between 2024 and 2025?" → platform answers in seconds. No other research firm can do this.

**Pitch price:** $175-275K for the Initech engagement itself + $60K/year ongoing for "insights as a subscription" (continuous query access to the Initech historical corpus).

### 3.3 Telephone-survey quant-qual hybrid — USAF mid-tier financial institution

**Setup:** Regional bank doing customer churn analysis. Research America runs **500 telephone interviews** (15-20 min each, mostly quantitative survey + open-end responses) + **8 focus groups** with flagged-at-risk customers.

**Why platform matters:** 500 × 17 min = 142 hours of mostly-short audio. Each interview's open-end responses (2-3 min of free-form "why would you leave the bank?" narrative) is the qualitative GOLD that traditional quant surveys throw away because synthesizing it is too slow. Platform transcribes + themes all 500 open-ends → surfaces 15-20 themes the quant survey missed.

**Deliverable differentiation:** the bank gets a "verbatim insights report" — 50 of the most-representative customer quotes organized by theme, each with audio playback. The CEO can literally listen to 30 seconds of real customer voices for each theme.

**Pitch price:** $200-300K for the study + $40K/year ongoing for quarterly churn-tracking subscription.

---

## 4. Pricing + sales angles

| Engagement shape | Pitch price (initial) | Ongoing subscription | Typical payback |
|---|---|---|---|
| POC (this engagement) | $22,800 (AWS-funded) | — | n/a — proof-of-concept only |
| Production platform build | $250-400K | — | 3-5 months (vs. labour savings) |
| "AI-augmented project" add-on | $50-100K per project | — | Per-project ROI |
| Platform subscription (client re-sells) | $250-400K annual | $250-400K/yr | 12-18 months |
| Cross-study search + library access | $60-120K annual | Add-on to above | — |

**Common objections + responses:**

- *"We already have Dscout / Qualtrics — what's different?"* → "Both are DIY tools for in-house teams. This is built for a full-service research firm's workflow — cross-study insights, client-facing deliverables, multi-tenant with client-view portals. The AI is also tuned specifically for deep qualitative work, not screener surveys."

- *"What if AWS changes pricing?"* → "The $19,700/month projection has 30% headroom. Worst-case doubling = $40K/month vs $2.3M/month labour cost displaced. Not a material risk."

- *"We're worried about IP leakage — our client data going to Bedrock."* → "Bedrock invocations are not used for model training (AWS contractual guarantee). All data encrypted at rest with customer-managed KMS. Per-client tenant isolation is IAM-enforced + continuously probed. Transcripts never leave your AWS account."

- *"Our regulated clients (pharma, healthcare) need HIPAA."* → "Kit has a HIPAA flag. When flipped: BAA signed with AWS, dedicated Business Associate services only (Transcribe Medical + Bedrock under BAA), enhanced audit logging, longer retention on audit bucket. Delivered in 1-2 week uplift on top of POC foundation."

- *"What about the researcher's craft — are we automating them out of a job?"* → "The platform handles the tedious 80% (transcription review, theme extraction, report drafting). The creative 20% — methodology design, stimulus selection, interpretation + recommendation — is amplified by giving the researcher more time + data to work with. We see researchers producing 4-5x more deliverables, not fewer researchers."

---

## 5. What we considered and didn't do

**"Use Amazon Q in QuickSight for the analyst search experience"**
QuickSight Q is optimised for structured data. Our use case is semantic search over transcripts (text + vectors). Would have required pushing all text into a QuickSight dataset with awkward modelling. Direct S3 Vectors + custom UI is faster + cheaper + more flexible.

**"Build directly on Bedrock Knowledge Bases (managed RAG)"**
KB is great for simple doc-RAG but lacks the per-tenant isolation controls + quote-level metadata + timestamp-coupling we need. Would have required shadow-indexing the same data in both KB and S3 Vectors. Custom RAG on S3 Vectors is tighter.

**"Transcribe Medical for pharma clients by default"**
Transcribe Medical is HIPAA-eligible + has medical vocabulary but costs more + narrower language support. Kept as a TOGGLE (flip when pharma client with BAA) instead of default.

**"Real-time streaming transcription in POC"**
Explicitly excluded by SOW §Real-Time Streaming. Technically possible (Transcribe streaming + WebSocket) but triples the POC complexity. Flagged as Phase 3 capability ("moderator live-assist").

**"Next.js SSR + server components for the UI"**
SOW says "React.js Web Application". Next.js SSR adds SEO benefits we don't need (internal app, not public), and adds deploy complexity (edge vs client rendering). React + Vite SPA + CloudFront SSG is simpler + matches SOW literally.

**"Speaker identification (not just diarization) — assign names automatically"**
Transcribe gives us "Speaker 1 / Speaker 2 / Speaker 3" (diarization = separate voices); not "this is Sarah the moderator". True speaker ID requires enrollment audio per speaker + custom SageMaker model. Out of POC scope; UI handles name-assignment manually with speaker-role inference (moderator usually speaks first / most).

**"A native iOS / Android app for field researchers"**
SOW explicitly excludes mobile apps. Mobile-responsive web (which we deliver) covers 95% of use cases.

---

## 6. Phase 2 + Phase 3 evolution roadmap

Post-POC, the kit evolves across two more phases.

### Phase 2 — Production-readiness + core differentiators (3-4 months, $250-400K)

Features that any real client deployment needs but which the 2-week POC defers:

- PII redaction via Comprehend + Bedrock Guardrails (POC has stub)
- Quote-with-audio-seek UX at full polish (POC has the back-end; the UI is the hero)
- Project-level cross-session theme aggregation (POC does per-file; Phase 2 does project-level)
- DOCX + PPTX auto-drafting with editable templates (POC stub)
- Automatic verbatim coding (pre-apply traditional market research codes per vertical codebook)
- Multi-user auth with Cognito groups + MFA + SAML federation to client SSO (POC uses static creds)
- Client read-only portal (pharma company sees their project's outputs)
- Audit log with CloudTrail integration + exportable compliance reports
- HIPAA-toggle when pharma client + BAA signed (Transcribe Medical, enhanced encryption, audit retention)
- Cost anomaly alarms with Slack integration
- Batch status dashboard + failure/retry workflow for analysts

### Phase 3 — Competitive differentiators (6+ months, $500K-1M)

Features that differentiate Research America from Qualtrics / Dscout / SPSS:

- **Cross-study semantic search** (sold as subscription add-on) — the "search your research history" capability
- **Emerging-theme proactive alerts** — platform notifies analyst "pricing concerns spiked 40% this month"
- **Competitive-mention tracking** with realtime alerts to client
- **Moderator live-assist during focus groups** (realtime transcription streaming + AI next-question suggestions) — requires Phase 3 budget for realtime architecture
- **Sentiment-drift longitudinal dashboards** (quarter-over-quarter sentiment change for Brand X)
- **Speaker identification model** (SageMaker custom endpoint trained on enrolled voices)
- **Multi-language support** (Spanish, Mandarin, French for global focus groups)
- **Qualtrics / SurveyMonkey / SPSS integration** (bridge qualitative + quantitative)
- **Mobile ethnography app** (iOS / Android field recording with background upload)

---

## 7. Risks + mitigations

| Risk | Mitigation |
|---|---|
| **Transcribe accuracy on accented English** | Enable Transcribe's auto-language-id + custom vocabulary per vertical; measure WER on sampled calibration set weekly |
| **Bedrock Claude cost blow-up** | Per-call cost logging to DDB + daily budget alarm; use Haiku for high-volume per-quote coding, Sonnet only for final report draft |
| **PII leak into vectors** | Redaction runs BEFORE embedding generation (non-negotiable #9); validate with continuous probe |
| **Cross-tenant data leak** | IAM conditions enforce tenant ID on every resource; continuous probe Lambda attempts cross-tenant reads + alerts if any succeed |
| **Pharma client requires HIPAA** | HIPAA flag in kit params; uplift estimated at 1-2 weeks + ~$25K additional; BAA must be in place before enabling |
| **Bedrock availability in client region** | Kit supports regional failover; cross-region inference profile ARN pattern documented |
| **S3 Tables / S3 Vectors regional availability** | Check region support before deploy; fallback is OpenSearch Serverless for semantic search (higher cost) |
| **Client wants on-prem deployment** | Not in POC scope. Flag as "not supported" — if a real requirement, scope as a separate engagement ($500K+) using Outposts |

---

## 8. Related

- [`../qualitative-research-audio-analytics.md`](../qualitative-research-audio-analytics.md) — the executable kit playbook (engineering)
- [`../README.md`](../README.md) — kit library navigation + decision tree
- [`../../README.md`](../../README.md) — top-level F369 library
- [`../../../F369_CICD_Template/prompt_templates/partials/README.md`](../../../F369_CICD_Template/prompt_templates/partials/README.md) — partial library + Canonical Registry
