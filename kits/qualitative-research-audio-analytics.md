<!-- Kit Version: 2.0 | Standard: Business-First v1.0 (REFERENCE IMPLEMENTATION) | Reference frontend: E:\NBS_Research_America_regen\frontend\ (local snapshot 2026-04-23) | Supersedes: v1.0 (technology-first) -->

# Kit — Qualitative Research Audio Analytics (Reference Implementation, Business-First v2.0)

> **This kit is the canonical reference for the [Business-First Kit Standard v1.0](_template/README.md).** It is structured business-first: §1 (P&L lever the buyer cares about) → §2 (personas + workflows) → §3 (frontend reference spec) → §4 (AWS architecture, derived from §1-§3) → §5 (delivery plan). Authors of new kits should read this end-to-end as the worked example.
>
> **Supersedes v1.0** (which started from AWS architecture and bolted business framing on top — the failure mode the standard exists to prevent).

**Client ask (verbatim, Research America Inc. 2026-04 SOW):** "Analyze thousands of qualitative research audio recordings per month — focus groups, in-depth interviews, telephone surveys — with AI-driven transcription, summaries, sentiment, themes, quotes-with-audio-seek, and cross-study semantic search. Production-grade, multi-persona (Ops admin + Analyst + Moderator + Client-viewer), multi-tenant, governed."

**Engagement:** 2-week POC · 2 developers · 1 designer · path-to-production well-documented.
**Deliverable summary:** CDK Python repo (13 stacks) + React/Vite SPA with 4 personas + Step Functions audio workflow + RDS PostgreSQL + DynamoDB ops tracking + S3 Vectors semantic search + ⌘K command palette + drag-drop report builder + 7-year audit log + Prompt Lab + cost ledger + mock data + runbook.

**Reference frontend:** [`E:\NBS_Research_America_regen\frontend\`](file:///E:/NBS_Research_America_regen/frontend/) — local snapshot 2026-04-23. 13+ routes (7 analyst + 6 admin) · 40+ domain types · 8 visualization components · MOCK_MODE auto-on. `pnpm dev` from `frontend/` boots the full demo without any backend dependency.

**Pitch range:** $250K–$500K POC · $1.2M–$2.5M production build follow-on.

---

## §1 — Business outcome

### §1.1 The buyer

- **Title:** **Chief Operating Officer / Chief Strategy Officer / VP Insights** at a mid-to-large traditional market research firm. At Research America Inc. specifically: founder + COO who jointly own the AWS account relationship.
- **Company stage:** Privately-held, services-led, $50M–$500M revenue, 30+ years in market.
- **Org size signal:** 200–2,000 FTEs of which 60–80% are billable researchers. 14+ acquired companies typical (rolled-up boutique structure). 16+ industry verticals served. 30M+ proprietary panel members.
- **Reports to:** CEO / Founder. P&L authority over services margin and platform investment.
- **Why this title (not CTO):** The COO/CSO owns the labour-margin equation. CTO will be brought in for technical buy-off, but the funding decision is COO-level because the math is "displace $1.6M/month of researcher time with $19.7K/month of AWS spend." That conversation belongs to a P&L owner, not a CIO.

### §1.2 The P&L lever

- **Lever type:** **Cost reduction (labour displacement) PLUS revenue growth (product-tier introduction).**

- **Cost-reduction math (the headline):**
  ```
  Current state — manual synthesis:
    Senior researcher loaded cost: ~$150K/yr
    Hours per focus group synthesis: ~12h (transcript review + theme extraction + report drafting)
    Firm-wide volume per SOW: 600 focus groups + 6,400 IDIs/surveys / month
    Total synthesis hours/mo:  600 × 12  +  6,400 × 6  ≈  45,600 hrs
    Loaded labour cost:         45,600 × ($150K / 2080 hrs)  ≈  $3.3M/mo
    Synthesis-share of that:    ~$2.3M/mo (the rest is fieldwork, recruiting, etc.)

  Future state — platform-augmented:
    Hours per focus group synthesis: ~3h (AI drafts; researcher reviews + polishes)
    Effective synthesis labour cost: ~$700K/mo
    Monthly savings:                 $1.6M

  AWS run-cost (per SOW projection, 7K recordings/mo):  ~$19.7K/mo
  RATIO:  $1 of AWS spend displaces ~$80 of researcher labour cost.
  Annual savings per client:                   $19.2M
  ```

- **Revenue-growth math (the second order):** Once Research America has the platform working internally, they license it to clients as a SaaS product (in-house insights teams at pharma / CPG / auto / finance who want to BYO their qualitative work). Conservatively: 50 SaaS clients × $80K/yr ARR = $4M new ARR within 24 months. This converts Research America from a services-only firm into a services + software firm — the multiple-arbitrage move.

- **Annual $ magnitude per client (services side):** **$19.2M labour displacement** (with ~$240K AWS run-cost).

### §1.3 Strategic context — why NOW

Three forces converging in 2025-2026 make this a now-or-never moment for Research America's category:

1. **Tech-first research platforms are eating the bottom of the market.** Qualtrics, Dscout, User Interviews, Remesh have moved from "DIY survey tools" to "DIY qualitative research with AI summarization." In-house insights teams at Research America's own clients are running their own focus groups via Dscout + having ChatGPT summarise the transcripts. The entry-level and mid-market of Research America's book is being commoditized.

2. **Strategy consultancies are eating the top of the market.** Bain, McKinsey, BCG have each launched in-house qualitative AI tools (Bain Vector, McKinsey QuantumBlack, BCG GAMMA). They use them to upsell their own consulting work, displacing Research America's relationship with the C-suite buyer.

3. **The "AI table-stakes" inflection.** RFPs in 2026 from pharma + CPG buyers explicitly require "AI-augmented qualitative methodology" as a vendor qualification criterion. Research America has lost 3 RFPs in Q1 2026 against Ipsos, Kantar, GfK on this exact gap. Without AI capability, they're disqualified before pricing is even discussed.

The existential question: **if an in-house team can run focus groups via Dscout + AI-summarise, why pay Research America's full-service rate?** The answer Research America bets on with this engagement: become "the AI-augmented full-service research firm" — combining 35 years of human methodological depth with AI tuned for deep qualitative work (better than Dscout's surface-level summaries because grounded in actual researcher craft). The Intelligent Audio Analytics POC is the foundation. If it works, Phase 2 is licensing to clients as SaaS.

### §1.4 Anti-patterns specific to the domain

The buyer rejects ALL of the following. Each one constrains the architecture and is cited downstream in §4.

1. **No auto-publish.** Every report passes through a moderator review queue before delivery to client. AI drafts; humans approve. **(drives §4.4 rule #11: state-machine approval gate before client delivery)**
2. **No inline PII.** Respondent names / emails / phones / addresses are masked by default. Reveal control exists with audit log — every reveal logs (user, when, which respondent, why). **(drives §4.4 rule #9: PII redaction runs BEFORE embedding generation, otherwise vectors leak forever)**
3. **No model output without confidence score.** Every LLM-generated summary bullet, sentiment score, and quote attribution carries a confidence percentage and a citation back to the source transcript span. Researchers will not stake their reputation on an unsourced claim. **(drives §3.4 #6 evidence-linked summaries; §3.4 #10 low-confidence flag surfacing)**
4. **No "AI did it" framing in client-facing artifacts.** Reports say "Research America's analysts find..." not "AI says...". The brand premium is human expertise, AI-augmented. UI copy and report templates reflect this. **(drives §3.4 branding choices, report template wording in §7 Prompt library)**
5. **No pre-publication client-name leakage.** Until a report is signed off and delivered, client names + brand names + competitive intel are scoped to the analyst's tenant. ClientViewer role does not see project pipeline; only delivered reports. **(drives §4.4 rule #6: strict tenant isolation by Cognito custom:tenant_id)**
6. **No cost numbers visible to ClientViewer.** Per-client / per-study AWS cost is admin-only. ClientViewer sees research deliverables, not infrastructure pricing. **(drives §3.1 route gating + §4.4 rule #10)**
7. **No analyst self-serve tenant creation.** Tenant onboarding is admin-only with explicit billing setup. Prevents analyst-driven cost runaway.
8. **No prompt edits land in production without 2-admin approval + 10-transcript A/B test.** Bedrock prompts directly drive every analyst's output quality; an unreviewed change breaks every downstream report. **(drives §3.4 #7 Prompt Lab; §4.4 rule #12)**

### §1.5 Win condition (90 days post-deployment)

Success at Day 90 = ALL of:

- [ ] Synthesis time per focus group drops from 12h to ≤4h, validated against a sample of 50 completed studies (researcher time tracked in Harvest / Toggl / equivalent before vs. after).
- [ ] WER (word error rate) on transcripts ≤ 5% on en-US clean audio, ≤ 8% on en-US accented audio, validated against a 20-sample human-labeled golden set per quarter.
- [ ] PII recall ≥ 99% on a synthetic-PII test corpus (zero-leak target). Any failure = pager.
- [ ] At least one full client-deliverable report (DOCX + PPTX) generated via the platform, reviewed by a client, with positive feedback documented.
- [ ] SOC 2 Type II auditor reviews the audit-log export and signs off on the trail (no findings on the data layer).
- [ ] Cost per session ≤ $3.00 averaged over 7K monthly recordings (per SOW budget envelope).
- [ ] Admin ops cockpit p95 read latency < 200 ms (verified via CloudWatch RUM).
- [ ] Zero P1 / P2 incidents in the trailing 30 days at 1K+ recordings/day load.

If any criterion fails by Day 90, that becomes a Phase 2 priority and the engagement defers to Quarter 4 review.

---

## §2 — Personas & workflows

### §2.1 Persona matrix

| Persona | Role group / scope | Primary daily workflow | Auth scope | Buying influence |
|---|---|---|---|---|
| **Senior Research Analyst** (PhD or 10+ yrs) | Analyst / own tenant | Open queue → review session → pin verbatims → draft report → submit for review | own-tenant read+write | user (champion) |
| **Research Director / Moderator** | Moderator / own tenant | Approve queue → edit verbatims inline → sign off → deliver to client | own-tenant approve | champion (technical) |
| **Client Viewer** (insights lead at client co.) | ClientViewer / own tenant, read-only | Browse delivered reports → search verbatims → export PDF | own-tenant read-only | user (NPS metric) |
| **Ops Admin / COO** (Research America) | Admin / global | Cost ledger → audit CSV → pipeline health → prompt versioning → tenant onboarding | global read+write | **buyer (§1.1)** |

### §2.2 Day-in-the-life narratives

#### Maria — Senior Research Analyst, PhD, Pfizer account

**Tuesday 9:14am.** Maria opens the platform from her browser. She's the lead on Pfizer's Vyndamax cardiologist IDI series — 16 sessions across Boston, Chicago, Atlanta, Houston. The Studies dashboard shows the Vyndamax study at the top with `12/16 sessions complete · avg sentiment +0.24 · 3 flags to triage`. She clicks in.

The Study Detail page opens to the Sessions tab. Three sessions ran overnight (Houston 6 EPs, Atlanta 6 HF specialists). She opens the Atlanta session — 53 minutes, 6 participants, transcript confidence 89%. The amber banner at the top warns: "Transcript 89% confidence · 2 segments flagged for review." She doesn't ignore it — last quarter she got burned trusting a 76% confidence transcript and ended up with a misquote in a client-facing report. She clicks Review Flags, jumps to the transcript, fixes the two amber-underlined words (one is "amyloid" mistranscribed as "amalo" — Atlanta accent + medical jargon), saves.

Back to the Insights tab. The Sentiment Timeline shows a dip around 28:00-32:00. She hovers — that's the discussion guide question on biopsy resistance. The Speaker Breakdown shows P3 (the high-volume EP) doing 22% of talk time with sentiment -0.35. Topic Bubbles surface "echo workflow gaps" as the largest dark-amber bubble. She pins the verbatim "If I had a simple algorithm that flagged ATTR-CM at echo, I'd diagnose three times more patients" — she'll use it in the Pfizer whitepaper.

By 11:30am she's drafted the Atlanta session's executive summary. She drags it into the Reports tab → Pfizer Vyndamax — Wave 3 Preliminary report → Drag-drop section reordering. Submits to Sarina (Research Director) for review. **Total time on this session: 2h15m.** Pre-platform that was 8-10 hours.

She moves to her next study at 11:45am.

#### Tom — Ops Admin (COO, Research America)

**Tuesday 7:30am.** Tom opens `/admin/ops`. The pipeline diagram shows 14 sessions transcribing (amber — queue pressure but not failing), 4 in Bedrock analysis. P95 transcribe latency 47 minutes (within SLA). Yesterday's spend: $612. Top client: Pfizer ($289). On track for the $19.7K monthly budget.

He clicks `/admin/audit`. SOC 2 Type II prep starts next month — he runs the export filter (last 30 days, all subject types) and CSVs it for the auditor's "data subject access" simulation test. The CSV shows every action by every user, with metadata, in an immutable append-only table. The auditor will love it.

Next, `/admin/prompts`. The Sentiment v2 prompt has been running for 3 weeks at 76% A/B win rate vs. v1. He promotes it to active across all tenants. Then he opens the Cost Ledger — Marriott's Bonvoy study has crept to $0.34 cost-per-session (above the $0.30 target). He drills in: it's the long IDIs (90+ minutes, double the average). He notes it for Sarina's pricing-conversation backlog.

By 8:15am he's done. Pre-platform he'd be hand-pulling CloudWatch logs and asking the dev team for cost rollups by client — a half-day exercise.

### §2.3 Business artifacts

| Artifact | Producer | Consumer | Format | Retention | Compliance notes |
|---|---|---|---|---|---|
| Final client report | Analyst → Moderator (approves) | Client | DOCX / PPTX / PDF | 7 yr | Client name + brand released only after sign-off. PII fully redacted. |
| Topline preliminary report | Analyst | Internal team | DOCX | 90 d (working draft) | Pre-publication; never client-shared |
| Pinned verbatim | Analyst | Report builder · Client (in delivered report) | JSON in DDB / RDS | Tied to session retention (90d audio, 365d transcript, 7yr insights) | Speaker labels are pseudonymized panel IDs, not real names |
| Audit CSV export | Admin | SOC 2 / GDPR auditor · client legal team (data subject request) | CSV | Generated on-demand from 7yr DDB log | Immutable, append-only at source; CloudTrail Lake cross-check |
| Cost rollup (per-client / per-study) | Admin | Internal pricing team | CSV / dashboard | Computed from DDB cost_ledger; archived monthly | Admin-only; never visible to ClientViewer (§1.4 anti-pattern #6) |
| Discussion guide | Moderator (pre-fieldwork) | Aligner (Bedrock) → Analyst | PDF / DOCX upload → parsed JSON | 7 yr (study lifetime) | Confidential to study tenant |
| Theme codebook (per study) | Analyst (with AI suggestions) | Reports · Crosstabs | JSON (RDS) | 7 yr | Edited inline; Bedrock can re-suggest after edits |
| Shared client link (Phase 2) | Moderator | ClientViewer | tokenized URL | 30d default · admin-configurable | Token rotation; revoke-on-suspicion |
| Prompt version | Admin | Bedrock InvokeModel (every run) | text in SSM Parameter Store | Forever (full version history kept) | 2-admin approval + 10-transcript A/B before promotion |
| Activity event | All actors | Audit log + Activity tab on Study | row in DDB audit_log | 7 yr | Immutable; (actor, action, subject, timestamp, metadata) |

### §2.4 Cross-persona handoffs

These are the workflow seams where state machines, notifications, and approval gates live. Each one becomes a §4 architectural choice.

1. **Analyst submits draft report → Moderator review queue.** Triggers: notification to Moderator role · status transition `draft → in_review` · email/Slack ping (Phase 2). Moderator can approve OR return-with-comments. Returns reset to `draft`. **(§4 → state machine in `report_status` field; SES/SNS notification publisher)**

2. **Moderator approves → Final delivery to ClientViewer via shared link.** Triggers: status transition `in_review → final → delivered` · token generation · audit log entry · share-link email to client. Client viewer can read the report + search pinned verbatims, but cannot see study pipeline or other reports. **(§4 → tokenized presigned URL service; ClientViewer role IAM scope)**

3. **Admin promotes new prompt version → Auto A/B test → 1-click promote to active.** Triggers: SSM Parameter Store write to `/audio-analytics/prompts/<name>/candidate` · Step Functions workflow runs candidate against 10 recent transcripts · human-judge UI surfaces win-rate · admin clicks Promote → active version updates. **(§4 → SSM versioning + dedicated A/B SFN workflow + human-judge Lambda)**

4. **Session uploaded → Pipeline → Insights ready.** Triggers: S3 ObjectCreated → EventBridge → Step Functions AudioWorkflow (10 stages) → on completion, notification to session owner + Discussion Guide auto-align (Bedrock) → status `complete`. The pipeline health is what the Admin watches in `/admin/ops`. **(§4 → primary SFN AudioProcessingWorkflow)**

5. **Verbatim pinned to report → Live update in Report Builder preview.** Triggers: write to `verbatims.pinned_report_ids` array · React Query cache invalidation · live update in Report Builder section "verbatims" / "top_verbatims". Curation happens in Insights view; preview updates in Report Builder. **(§4 → React Query optimistic update + DDB GSI on `pinned_report_ids`)**

6. **ASR confidence < 70% → Low-confidence flag → Analyst review required.** Triggers: at transcribe-parse time, segments with confidence < 0.7 are marked `low_confidence` in the segments table; study + session counters increment; UI surfaces amber banner; analyst can correct inline (Phase 2: edit + re-embed) or accept. Trust gate before insights are surfaced. **(§4 → segments table low_confidence column + study counter + UI alerting; this directly mitigates §1.4 anti-pattern #3)**

---

## §3 — Frontend reference spec

### §3.1 Routes

| Path | Persona(s) | Purpose | Key data shown | Notable interactions | Mobile |
|---|---|---|---|---|---|
| `/login` | all | Auth | Cognito form · 4 demo persona buttons | role-aware redirect; honors `from` only if new user's roles permit it | yes |
| `/analyst/studies` | Analyst, Moderator | Master list of studies w/ filters | QuickStats cards · StudiesTable (status, sessions, sentiment, flags, deadline) | search · status filter · scope filter (all/mine) | yes |
| `/analyst/studies/:id` | Analyst, Moderator | Study detail hub w/ 8 tabs | Header · 4 KPI cards w/ data-source tooltips · 8 tabs (Overview, Sessions, Discussion Guide, Themes, Verbatims, Crosstabs, Reports, Activity) | tab navigation; sub-route for upload/reports | partial |
| `/analyst/studies/:id/upload` | Analyst | Drag-drop session upload + per-session metadata | dropzone · per-file form · progress bars | drag-drop · validation · simulated upload | partial |
| `/analyst/studies/:studyId/reports/:reportId` | Analyst, Moderator | Report Builder (drag-drop sections) | Section palette (left) · live preview (right) · cover page | dnd-kit drag-drop · print-to-PDF · template selection | desktop |
| `/analyst/sessions/:id/insights` | Analyst, Moderator | Session insights drill-down | Summary w/ evidence drawer · Sentiment Timeline · Speaker Breakdown · Topic Bubbles · Sentiment Gauge · Verbatims · Action Items kanban | click verbatim to pin · click bullet to open evidence drawer · sentiment legend | desktop |
| `/analyst/sessions/:id/transcript` | Analyst, Moderator | Audio + diarized transcript w/ word-level highlight | waveform player · speaker sidebar · word-active highlighting · low-confidence amber underlines · search box | Space=play · ←/→ ±5s · Shift+arrows ±30s · click word to seek · click speaker to filter | desktop |
| `/analyst/notifications` | Analyst, Moderator | Inbox of job/mention/review/system events | notification list · kind-filter chips · channel preview | mark read · mark all · click → drill-through | yes |
| `/admin/ops` | Admin | Operations console | KPI row · 7-stage pipeline diagram (in-flight + p95 + failures) · service spend pie · client spend bar · recent activity | refresh · drill-through | desktop |
| `/admin/cost` | Admin | Cost ledger | KPI cards (MTD spend, POC budget, forecast, anomalies) · daily trend · service pie · client bar · per-study table · raw line items | filter date · drill to study | desktop |
| `/admin/audit` | Admin | Audit trail | 6 category cards · filter rail · event table w/ metadata | search · filter actor/category/subject · CSV export | desktop |
| `/admin/health` | Admin | System health (SLA + pipeline metrics) | 4 threshold-colored KPI cards · per-stage p95 + throughput bars | hover for legend | desktop |
| `/admin/jobs` | Admin | All jobs cross-tenant | Status counters · search · filter · per-session table | search · filter status | desktop |
| `/admin/prompts` | Admin | Prompt Lab (versioned LLM prompts) | 4-prompt picker · active version w/ A/B win rate · Edit / History / A/B Compare tabs | edit · revert · save-as-new · promote · A/B run | desktop |
| `/admin/users` | Admin | Cognito user mgmt | user list · role assignments · MFA status | invite · disable · reset MFA | desktop |
| `/admin/tenants` | Admin (legacy v1.x) | Per-client config | tenant list · retention · budget · HIPAA toggle | edit per-tenant config | desktop |
| `/forbidden` · `/404` | all | Error states | message + CTA | redirect on switch | yes |
| `+ ⌘K command palette` | all (role-aware) | Global search & nav | studies / sessions / verbatims / actions; admin sees admin nav | keyboard-driven; Cmd/Ctrl+K opens | desktop |

**Total: 17 routes + ⌘K palette.** Persona segregated by URL prefix (`/analyst/*` vs `/admin/*`). Admin routes locked server-side via Cognito group check in Lambda authorizer.

### §3.2 Domain types

40+ entities. Field choices encode business logic — every non-obvious field is annotated with what it reveals.

```typescript
// ────────── Identity / Tenancy ──────────

export type UserRole = "analyst" | "admin" | "reviewer" | "client_viewer";

export interface RADomainUser {
  id: string;
  email: string;
  name: string;
  role: UserRole;
  office?: string;            // 1-of-9 RA offices; not all users have one (admin: "—")
  phd?: boolean;              // PhD signals senior-tier billing rate
  avatar_url?: string;
  study_ids: string[];        // assignments; reveals multi-study analyst
}

export const RA_OFFICES = [
  "Syracuse, NY", "Philadelphia, PA", "Atlanta, GA", "Chicago, IL",
  "Dallas, TX", "Denver, CO", "Los Angeles, CA", "San Francisco, CA", "Seattle, WA",
];  // hardcoded list — Research America operates from exactly these 9 cities

// ────────── Service taxonomy (drives billing tier + prompt vertical) ──────────

export type ServiceLine =
  | "innovation_development" | "ad_tracking_brand_messaging" | "landscaping_benchmarking"
  | "qualitative_research" | "quantitative_data_collection" | "strategic_consulting" | "international";

export type Vertical =
  | "pharmaceutical" | "healthcare" | "retail_consumer" | "agriculture"
  | "utilities" | "manufacturing" | "lottery" | "marketing_advertising"
  | "entertainment" | "automotive" | "finance_legal" | "food_beverage"
  | "travel_hospitality" | "apparel_footwear" | "home_hardware"
  | "health_beauty" | "hitech_ecommerce" | "alternative_energy";  // 18 verticals; per-vertical prompt overrides

export type Methodology =
  | "focus_group" | "idi" | "face_to_face" | "dyad" | "triad"
  | "ethnography" | "online_community" | "sensory" | "crisis_comms";

// ────────── Core domain ──────────

export interface Client {
  id: string; name: string; vertical: Vertical; logo_url?: string;
}

export interface Project {
  id: string; client_id: string; name: string; brand: string;
  campaign?: string; brief?: string; service_line: ServiceLine;
}

export type StudyStatus =
  | "setup" | "collecting" | "transcribing" | "analyzing"
  | "in_review" | "delivered" | "archived" | "failed";

export interface Study {
  id: string; project_id: string; client_id: string; client_name: string;
  project_name: string; brand: string; service_line: ServiceLine;
  name: string; methodology: Methodology; vertical: Vertical;
  cities: string[];                        // multi-city design — drives MetrixMatrix recruiting
  status: StudyStatus;
  owner: string;                           // analyst email
  office: string;                          // 1-of-9 RA offices
  collaborators: string[];                 // multi-analyst studies are common
  objectives: string[];                    // appears verbatim in cover page of report
  session_count: number;
  sessions_complete: number;
  avg_sentiment?: number;                  // -1..+1; computed from completed sessions
  low_confidence_flags: number;            // ASR-level signal; reveals review-required count (§1.4 #3)
  created_at: string; updated_at: string;
  deadline?: string;
  tags: string[];                          // Gen-Z, HCP, brand-tracker, etc. — used for filtering/search
}

export type SessionStatus =
  | "pending" | "uploading" | "stored" | "transcribing"
  | "transcribed" | "analyzing" | "analyzed" | "complete" | "failed";

export interface Session {
  id: string; study_id: string; name: string; city: string;
  moderator: string; recording_date: string;
  duration_sec: number; participant_count: number;
  media_url?: string;                      // s3:// or mock
  media_kind: "audio" | "video";
  status: SessionStatus;                   // 9-state lifecycle
  created_at: string;
  transcript_confidence?: number;          // 0..1; surfaces in UI as % and color
  sentiment_score?: number;                // -1..+1; rolls up to study.avg_sentiment
}

export interface Participant {
  id: string; session_id: string; pseudonym: string;          // never real name
  age_band: "18-24" | "25-34" | "35-44" | "45-54" | "55-64" | "65+";
  gender: "female" | "male" | "non_binary" | "prefer_not";
  region: string;
  income_tier: "<$35K" | "$35K-$75K" | "$75K-$125K" | "$125K-$200K" | ">$200K";
  panel_member_id?: string;                // MetrixMatrix panel ID — RA's proprietary panel
  custom_tags: string[];                   // analyst-applied (e.g. "Heavy soda drinker")
}

// ────────── Discussion guide + transcript ──────────

export interface DiscussionGuide {
  id: string; study_id: string;
  questions: DiscussionQuestion[];
}

export interface DiscussionQuestion {
  n: number; text: string; est_minutes: number;
  segment_matches_count?: number;          // populated post-Bedrock-alignment
  aggregate_sentiment?: number;            // per-question sentiment across study
}

export interface TranscriptWord {
  word: string; start_ms: number; end_ms: number;
  speaker_id: string;
  confidence: number;                      // <0.7 = amber underline + flag count
}

export interface Transcript {
  id: string; session_id: string; language: string;
  words: TranscriptWord[]; speakers: Speaker[];
}

export interface Speaker {
  id: string; label: string;               // "Moderator", "P1", "P2", ...
  participant_id?: string;
  talk_time_sec: number;
  avg_sentiment?: number;                  // computed for participants only; moderator is procedural
}

export interface Segment {
  id: string; session_id: string;
  question_n?: number;                     // aligned discussion guide question
  start_ms: number; end_ms: number;
  speaker_id: string; transcript_text: string;
}

// ────────── Insights (LLM output, evidence-linked) ──────────

export interface InsightCitation {
  segment_id: string; start_ms: number; end_ms: number; quote: string;
}

export interface InsightSummaryBullet {
  text: string;
  citations: InsightCitation[];            // EVERY bullet has citations — §1.4 anti-pattern #3
  confidence: number;                      // surfaces in UI as %
}

export interface SentimentPoint {
  start_ms: number; end_ms: number; score: number;
}

export interface SpeakerStat {
  speaker_id: string; speaker_label: string;
  talk_time_sec: number; talk_share_pct: number;
  avg_sentiment: number;                   // bar-chart color in UI
}

export interface KeyTopic {
  topic: string;
  mentions: number;                        // bubble size in UI
  sentiment: number;                       // bubble color in UI
  quotes: string[];
}

export interface ActionItem {
  id: string; text: string;
  owner?: string;                          // function (Marketing / Innovation / Strategy)
  priority: "low" | "medium" | "high";
  status: "todo" | "in_progress" | "done"; // 3-column kanban
}

export interface Insight {
  session_id: string;
  summary_bullets: InsightSummaryBullet[];
  sentiment_score: number;
  sentiment_label: "Negative" | "Neutral" | "Positive";
  sentiment_confidence: number;
  sentiment_timeline: SentimentPoint[];
  speaker_breakdown: SpeakerStat[];
  key_topics: KeyTopic[];
  action_items: ActionItem[];
  generated_at: string;
  low_confidence_segment_count: number;    // surfaces as session-level amber banner
}

// ────────── Verbatims (the curation surface) ──────────

export interface Verbatim {
  id: string; session_id: string; study_id: string;
  quote: string;
  speaker_id: string; speaker_label: string;
  start_ms: number; end_ms: number;
  sentiment: number;
  tags: string[];                          // analyst-applied (e.g. "Concept B", "Use-occasion")
  pinned_report_ids: string[];             // reveals curation: pinned verbatims appear in delivered reports
  note?: string;                           // analyst note shown in amber callout box
}

// ────────── Themes (codebook) ──────────

export interface Theme {
  id: string; study_id: string;
  parent_id?: string;                      // 2-level hierarchy
  name: string; description?: string;
  coded_segment_count: number;
  avg_sentiment?: number;
}

// ────────── Reports ──────────

export type ReportStatus = "draft" | "in_review" | "final" | "delivered";

export interface Report {
  id: string; study_id: string; title: string;
  template: "focus_group_standard" | "idi_summary" | "brand_tracker_wave" | "custom";
  status: ReportStatus;                    // 4-state lifecycle (§2.4 handoff #1)
  sections: string[];                      // ordered keys; drag-drop reorders
  author: string; reviewer?: string;
  created_at: string; updated_at: string;
  shared_token?: string;                   // present when delivered + share-link generated
}

// ────────── Activity / Audit ──────────

export interface Activity {
  id: string;
  subject_type: "study" | "session" | "report" | "theme" | "verbatim";
  subject_id: string;
  actor: string;                           // email or "system"
  action: string;                          // canonical action string (see ACTION_LABEL map)
  ts: string;
  metadata?: Record<string, string>;       // free-form k/v shown as chips in UI
}

// ────────── Notifications ──────────

export interface Notification {
  id: string;
  kind: "job_complete" | "mention" | "review_request" | "system" | "delivery";
  title: string; body?: string; link?: string;
  read: boolean; created_at: string;
}

// ────────── Operations / Cost ──────────

export type PipelineStage =
  | "upload" | "stored" | "extract" | "transcribing"
  | "analyzing" | "storing" | "delivered" | "failed";

export interface StageMetric {
  stage: PipelineStage;
  in_flight: number;
  throughput_per_hour: number;
  p50_ms: number; p95_ms: number;
  failed_last_hour: number;
}

export interface CostLedgerEntry {
  date: string;
  service: "transcribe" | "bedrock" | "rds" | "s3" | "lambda" | "other";
  amount_usd: number;
  study_id?: string;                       // OPTIONAL — shared infra has no study attribution
  client_name?: string;                    // for client rollup
}

export interface PromptVersion {
  version: string;                         // "v1", "v2", "v3"
  active: boolean;
  created_at: string;
  author: string;
  body: string;                            // the prompt text
  notes?: string;                          // post-A/B notes
  ab_test_wins?: number;
  ab_test_plays?: number;                  // win rate = wins/plays
}

export interface PromptTemplate {
  key: string;                             // "summary" | "sentiment" | "key-topics" | "action-items"
  ssm_path: string;                        // "/audio-analytics/prompts/<key>"
  title: string; description: string;
  model: string;                           // e.g. "anthropic.claude-3-sonnet-20240229-v1:0"
  versions: PromptVersion[];
}
```

**40+ entities across 7 sections** (Identity / Tenancy · Service taxonomy · Core domain · Discussion guide + transcript · Insights · Verbatims/Themes/Reports · Activity/Notifications/Operations).

### §3.3 Visualizations

| Component | What it shows | Chart type | Library | Interaction | Persona |
|---|---|---|---|---|---|
| **SentimentGauge** | -1..+1 needle on conic gradient | semi-circle gauge | custom CSS | static; color-coded score | analyst |
| **SentimentTimeline** | sentiment per 60-sec window across recording | line w/ ref at 0 | recharts | hover tooltip; activeDot; (Phase 2: click to seek transcript) | analyst |
| **TopicBubbles** | topic mentions × sentiment | custom SVG bubbles (size=mentions, color=sentiment) | inline | hover tooltip with sample quote | analyst |
| **SpeakerBreakdown** | talk % per speaker, colored by speaker's avg sentiment | stacked bar | recharts | hover; legend below | analyst |
| **InsightSummary + Evidence Drawer** | LLM bullets w/ inline citation links | numbered list + slide-in drawer | inline + Dialog | click "evidence" to open right-side drawer with source quotes + timestamps | analyst |
| **ActionItemsBoard** | 3-column kanban (To Do / In Progress / Done) | kanban cards | inline | (Phase 2: drag-drop) | analyst |
| **VerbatimsList** | curated quotes w/ pin button | card list | inline | click Pin → toggles `pinned_report_ids` membership | analyst |
| **Pipeline diagram** | 7-stage pipeline w/ in-flight + p95 + failures, color-thresholded | horizontal card sequence | inline | hover for detail; Phase 2 drill-down | admin |
| **StageMetric bars (Health)** | per-stage p95 latency + throughput | bar charts | recharts | hover | admin |
| **Spend by service / client** | cost rollups | pie + horizontal bar | recharts | hover; legend | admin |
| **Daily spend trend** | $/day rolling | line | recharts | hover; date filter | admin |

**11 visualizations.** All recharts or custom-CSS — no chart-bloat libraries. All work in dark mode.

### §3.4 Standout business features

20 features that distinguish this from "we built a Bedrock demo." Each maps to §3.2 fields and §3.3 viz.

1. **Sentiment Timeline with drill-down evidence.** Line chart of sentiment per 60-sec window paired with `InsightSummary` bullets that cite source transcript spans. Click "evidence" → side drawer slides in with the exact quotes that drove the sentiment. **Closes the loop: emotion trend → specific driver quotes → defensible client insight.**

2. **Topic Bubbles, click-to-drill ready.** Custom SVG bubbles where size = mentions, color = sentiment (-1..+1 → red→amber→gray→lime→emerald). Hover → tooltip with sample quote. Visual at-a-glance topic landscape; phase-2 click-through to filtered verbatims.

3. **Action Items Kanban.** 3-column board (To Do / In Progress / Done) for AI-derived strategic recommendations. Each card: action text, owner (function: Marketing/Innovation/Strategy), priority (color-coded). Semantic grouping for stakeholder accountability.

4. **Speaker Breakdown with sentiment-weighted talk time.** Stacked bar: % talk time per speaker × bar color = speaker's avg sentiment. Highlights group dynamics (which participants were positive/negative; who dominated). Moderator excluded from sentiment scoring (procedural).

5. **Audit Trail with CSV export & retention notes.** Compliance-grade event log: every study/session/report/theme/verbatim action with actor, timestamp, metadata. Filters by actor/category/subject type. CSV export. SOC 2 / GDPR retention notes embedded in UI. **First-pass auditor-ready.**

6. **Cost Ledger with multi-perspective analytics.** Aggregates AWS spend by service (pie), by client (bar), by study (table with cost-per-session for proposal pricing). Daily trend. Budget tracking (POC % used, production forecast). Per-line ledger. **Sales: cost-per-session is the lever for tiered client pricing.**

7. **Prompt Lab with version control & A/B testing.** Manages 4 Bedrock prompts (summary, sentiment, key-topics, action-items) versioned in SSM Parameter Store. Each prompt: active version, older with author/date/notes, A/B win-rate, edit-with-revert. History tab with promote/view. A/B compare runs against 10 recent transcripts. **MLOps for LLM prompts without code redeploy.**

8. **Report Builder with drag-drop section reordering.** dnd-kit for section reorder (grip, delete, +add). Left palette · live preview (print-safe HTML). Cover page auto-populated. Sections render dynamically. Export PDF (browser print). Print stylesheet hides chrome. Template-based.

9. **Study-centric multi-tab hub.** All study data in 8 tabs (Overview / Sessions / Discussion Guide / Themes / Verbatims / Crosstabs / Reports / Activity). Each summary card has a data-source tooltip explaining where the number came from (e.g. "Sentiment score | source: Amazon Bedrock Claude 3 Sonnet | …"). **Data lineage visible; defensible to clients.**

10. **Low-confidence transcript flagging.** Amber banner on sessions where transcript_confidence < 70% or low_confidence_segment_count > 0. Study cards show flag count. Visible risk signal throughout. Mitigates §1.4 anti-pattern #3 (no model output without confidence).

11. **Role-based home redirect with clean UX.** Admin landing on `/analyst` → `/admin/ops`; Analyst landing on `/admin` → `/analyst/studies`. No error page; transparent. Two personas cleanly separated.

12. **Command Palette with multi-entity search (⌘K).** Quick nav: jump to study (name/client/brand), session (city/moderator), verbatim (quote/tags/speaker). Role-aware groupings. Phase 2: semantic search via Bedrock Knowledge Base.

13. **Notifications Center with kind filtering & channels.** Job completions, mentions, review requests, system alerts, deliveries. Filter by kind/unread. Mark-as-read. Channel preview (in-app on, email/Slack Phase 2).

14. **Operations Console with live pipeline diagram.** 7-stage pipeline (upload → delivered) as horizontal card sequence. Each stage: in-flight count, p95 latency, failures. Color-coded health. KPI row above. Service/client spend pies. **DevOps-style monitoring for research platform.**

15. **Quick Study Stats dashboard.** Total studies + status breakdown (setup, collecting, transcribing, analyzing, in_review, delivered, archived, failed). Useful for project planning/forecasting.

16. **Verbatim pinning to reports.** `Verbatim.pinned_report_ids` tracks pinning. List shows pin button (outline → filled when pinned). Curation flow: analyst browses session verbatims → pins high-salience → ReportBuilder surfaces in `top_verbatims` section.

17. **Health Metrics with threshold-based coloring.** 4 KPI cards (jobs/hr, failure %, DLQ depth, p95 latency). Each card accepts threshold={warn, crit}. Colors transition green → amber → red. Per-stage p95 + throughput bar charts.

18. **Study progress tracking.** `sessions_complete / session_count` graphical bar on each study card. Drill to Sessions tab for per-session detail. Study-level avg_sentiment metric (weighted mean of completed sessions).

19. **Cross-tabulation & demographic breakdowns.** StudyDetail Crosstabs tab: slice insights by age × gender × market × usage tier. Cell color = sentiment, cell number = mention count. Phase 2: chi-square significance + 2-way pivot.

20. **Evidence-linked LLM summary bullets.** InsightSummary renders bullets w/ citation links. Click "N evidence" → drawer with supporting quotes + segment timestamps. Confidence % per bullet. **Addresses the analyst-trust problem head-on (§1.4 anti-pattern #3).**

### §3.5 Persona ribbon / branding

- **Primary brand:** HSL `221 83% 53%` (Research America deep blue)
- **Admin ribbon:** HSL `199 89% 48%` (cyan) — used in Admin badge + sidebar accent + persona-specific avatar accent
- **Analyst ribbon:** HSL `27 96% 45%` (amber) — same pattern for analyst persona
- **Sentiment palette (-1..+1):** rose (`#e11d48`) → amber (`#f59e0b`) → slate (`#6b7280`) → lime (`#84cc16`) → emerald (`#059669`)
- **Status palette (study lifecycle):** slate (setup) → sky (collecting) → blue (transcribing) → indigo (analyzing) → amber (in_review) → emerald (delivered) → slate (archived) → rose (failed)
- **Persona badge placement:** top-left of header, immediately right of logo, uppercase tracking-wider. Differentiates personas at a glance without changing nav structure.
- **Navigation pattern:** persona-segregated top-nav. Analyst sees Studies + Notifications. Admin sees Operations + Cost + Health + Prompts + All Jobs + Audit + Users. Same shell, different inventory.
- **Typography:** Inter (UI) · JetBrains Mono (code/numbers) · Source Serif 4 (published reports — gives client deliverables a "published research" feel).
- **Radius token:** 0.5rem (rounded-md) baseline; rounded-full on chips/badges; rounded-lg on Cards; rounded-xl on KPI hero cards.

### §3.6 Reference implementation pointer

- **Repo:** `E:\NBS_Research_America_regen\` (local; not yet pushed to a remote)
- **Frontend path:** `E:\NBS_Research_America_regen\frontend\`
- **Snapshot date:** 2026-04-23 (no git remote; local working tree is the source of truth at this snapshot)
- **Run command:** from `frontend/`: `pnpm install && pnpm dev` → http://localhost:5173/ (port may shift if 5173-5175 are in use)
- **MOCK_MODE:** auto-on when no `VITE_INSIGHTS_API` / `VITE_ADMIN_API` / `VITE_UPLOAD_API` env vars are set. The `/login` page exposes 4 one-click demo users (admin, analyst1, moderator, clientviewer1).
- **Demo recipe** (10 minutes):
  1. `/login` → click "Analyst" → `/analyst/studies` (QuickStats + 8 demo studies)
  2. Click PepsiCo Zero Sugar (highest-data study) → 8 tabs, click each
  3. Sessions tab → click LA session → `/analyst/sessions/ses-002-01/insights` (the full 6-viz page)
  4. Click "evidence" on a summary bullet → side drawer
  5. Click "Audio + Transcript" → `/analyst/sessions/ses-002-01/transcript` (waveform + diarized words + active highlight)
  6. Back to Study → Reports tab → open the in_review report → drag-reorder a section
  7. ⌘K → search "cherry" → see verbatim match
  8. Logout, login as Admin → `/admin/ops` (pipeline) → `/admin/cost` (ledger) → `/admin/audit` (CSV export) → `/admin/prompts` (Prompt Lab w/ A/B win rate)

A 2-minute screen-record of this recipe should ship as `kits/_design/qra-v2-demo.mp4` (TODO — not yet recorded).

The reference frontend is built per the [Claude React Frontend Prompt Framework v2.0](../frontend_prompt/Claude_React_Frontend_Prompt_Framework.md). Stack: React 18 + TS strict + Vite + Tailwind + shadcn/ui + React Query + RHF+Zod + react-router v6 + Zustand + date-fns + sonner + next-themes + recharts + lucide-react + dnd-kit + cmdk.

---

## §4 — AWS architecture

### §4.1 Architecture diagram

Each component annotated with the §1-§3 driver it serves.

```
                       ┌───────────────────────────────────────────────┐
                       │   Browser SPA (React + Vite + CF + OAC)        │
                       │                                                 │
                       │   /login   /analyst/*   /admin/*   ⌘K palette  │  ← §3.1 routes
                       └─────────────────┬───────────────────────────────┘
                                         │  HTTPS (WAF + API GW REST)
                                         │  Cognito JWT auth (§4.4 #6 strict tenant)
                                         ▼
  ┌──────────────────────────────────────────────────────────────────────────────────┐
  │  API Gateway (REST, regional, with WAF)                                           │
  │                                                                                    │
  │  /upload           → UploadLambda (presigned URL)                                 │
  │  /jobs · /jobs/:id → JobsApiLambda                                                │
  │  /jobs/:id/status  → StatusLambda (DDB read, fast)                                │
  │  /sessions/:id     → InsightsApiLambda                       ← §3.4 #20 evidence  │
  │  /studies/*        → StudiesApiLambda                                             │
  │  /quotes/:id/audio → AudioSeekLambda (range-GET presigned)   ← §4.4 #7 scoped url │
  │  /search           → SemanticSearchLambda  (S3 Vectors)      ← §3.4 #12 ⌘K        │
  │  /export           → ExportLambda (DOCX/PPTX gen)            ← §3.4 #8 ReportBldr │
  │                                                                                    │
  │  /admin/ops        → AdminOpsLambda                          ← §3.4 #14           │
  │  /admin/cost       → AdminCostLambda                         ← §3.4 #6            │
  │  /admin/audit      → AdminAuditLambda                        ← §3.4 #5 SOC 2     │
  │  /admin/health     → AdminHealthLambda                       ← §3.4 #17          │
  │  /admin/prompts    → AdminPromptsLambda (SSM r/w + SFN A/B)  ← §3.4 #7 Prompt Lab│
  │  /admin/users      → AdminUsersLambda (Cognito mgmt)                              │
  └──────────────────┬─────────────────────────────────────────────────────────────────┘
                     │
  ┌──────────────────┼─────────────────────────────────────────────────────────────────┐
  │                  ▼                                                                 │
  │  S3 audio-raw  ──► EventBridge ──► Step Functions  AudioProcessingWorkflow        │
  │                                                                                    │
  │  ┌──────────────────────────────────────────────────────────────────────┐         │
  │  │ 1.  ValidateAudio (format / size / virus scan gate)                  │         │
  │  │ 2.  CreateJobRow (DDB jobs row = QUEUED)                             │         │
  │  │ 3.  StartTranscriptionJob (Transcribe async)                          │         │
  │  │ 4.  WaitForTranscription (Wait + Poll)                                │         │
  │  │ 5.  ParseTranscript → mark low_confidence segments  ← §3.4 #10 flag  │         │
  │  │ 6.  PiiRedaction (Comprehend + Bedrock Guardrails)  ← §4.4 #9 PII    │         │
  │  │ 7.  BedrockAnalysis (parallel: summary | sentiment | themes | quotes  │         │
  │  │       | actions) — every InvokeModel writes cost_ledger ← §4.4 #8    │         │
  │  │ 8.  EmbeddingGeneration (Titan v2; AFTER PII redaction)              │         │
  │  │ 9.  PersistInsights (RDS writes + S3 Vectors PutVectors)              │         │
  │  │ 10. UpdateJobRow (DDB = COMPLETE) + EmitMetrics                      │         │
  │  │                                                                        │         │
  │  │ Per-step Catch → DLQ + audit_log + user notification                  │         │
  │  └──────────────────────────────────────────────────────────────────────┘         │
  │                                                                                    │
  │  Step Functions  PromptABTestWorkflow (admin-triggered)                           │
  │     - Replay 10 recent transcripts against candidate vs active prompt              │
  │     - Surface diff to human-judge UI; admin promotes via /admin/prompts            │
  │     ← drives §3.4 #7 Prompt Lab + §1.4 anti-pattern #8 (2-admin approval)          │
  └──────────────────────────────────────────────────────────────────────────────────┘
                     │                                     │
                     ▼                                     ▼
  ┌──────────────────────────┐              ┌──────────────────────────────────────┐
  │  DynamoDB (operational)  │              │  RDS Aurora Postgres Serverless v2   │
  │                          │              │                                      │
  │  Table: jobs             │              │  Table: clients · projects · studies │
  │    PK: tenant#job_id     │              │  Table: sessions · participants      │
  │  Table: audit_log        │              │  Table: transcripts (full-text GIN)  │
  │    PK: tenant#date       │              │  Table: insights (JSONB)             │
  │    SK: actor#timestamp   │              │  Table: verbatims (GSI: pin)         │ ← §3.4 #16 pin-to-rpt
  │    TTL: 7 yrs (§1.4 SOC2)│              │  Table: themes · codes                │
  │  Table: cost_ledger      │              │  Table: reports · exports             │
  │    PK: tenant#date       │              │  Table: tenant_configs                │
  │    SK: job_id            │              │                                      │
  │  Table: session_state    │              │  PITR snapshots: 7 days              │
  │  Table: rate_limit       │              └──────────────────────────────────────┘
  └──────────────────────────┘
                                            ┌──────────────────────────────────────┐
                                            │  S3 Vectors (semantic search)         │
                                            │  Index: quotes · transcript_chunks ·  │
                                            │         themes                        │
                                            │  Metadata: tenant_id (filter pushdown)│
                                            └──────────────────────────────────────┘

  Observability:  CloudWatch dashboards (ops + cost + quality) + SNS → PagerDuty alarms
  Compliance:      CloudTrail Lake → Object-Lock audit bucket; Config rules; Macie on audio
  Security:        WAF (managed core + sqli + 1000 rpm), 3 KMS CMKs (audio | db | reports),
                   Cognito user pool + MFA for Admin, IAM ResourceTag tenant_id condition
  Prompts:         SSM Parameter Store /audio-analytics/prompts/<key>  (versioned + A/B)
```

### §4.2 Stack inventory

| Stack | Purpose | Driven by | Key partials |
|---|---|---|---|
| `NetworkStack` | VPC + PrivateLink (Transcribe, Bedrock, S3, RDS, DDB) | global | `LAYER_NETWORKING` |
| `SecurityStack` | KMS (3 CMKs: audio, db, reports) + IAM permission boundaries | §1.4 #5 (tenant isolation) | `LAYER_SECURITY` |
| `AuthStack` | Cognito user pool + 4 role groups + MFA on Admin | §2.1 (4 personas) | `LAYER_SECURITY` |
| `DataStack` | RDS Aurora v2 (14 tables) + 5 DDB tables | §3.2 entities | `DATA_AURORA_SERVERLESS_V2`, `LAYER_DATA` |
| `MediaStack` | 3 S3 buckets (audio-raw, transcripts, reports) + OAC | §3.4 #8 + §1.4 #2 (PII) | `LAYER_BACKEND_LAMBDA` |
| `VectorStack` | S3 Vectors 3 indexes (quotes, chunks, themes) | §3.4 #12 (⌘K + cross-study search) | `DATA_S3_VECTORS` |
| `OrchestrationStack` | Step Functions AudioProcessingWorkflow + PromptABTestWorkflow | §2.4 #4 + §2.4 #3 | `WORKFLOW_STEP_FUNCTIONS`, `EVENT_DRIVEN_PATTERNS` |
| `IngestApiStack` | Upload presigned + jobs API | §2.4 #4 | `LAYER_API` |
| `InsightsApiStack` | Studies + sessions + insights + verbatims + reports APIs | §3.1 routes (analyst) | `LAYER_API` |
| `AdminApiStack` | Ops + cost + audit + health + prompts + users APIs | §3.1 routes (admin) + §1.4 #6 (cost gate) | `LAYER_API`, `LAYER_OBSERVABILITY` |
| `FrontendStack` | React SPA → S3 + CloudFront + OAC + WAF | §3 (entire frontend reference) | `LAYER_FRONTEND` |
| `ObservabilityStack` | CloudWatch dashboards + alarms + SNS → PagerDuty | §3.4 #14 + #17 + §1.5 SLOs | `LAYER_OBSERVABILITY` |
| `ComplianceStack` | Object-Lock audit bucket + CloudTrail Lake + Config rules + Macie on audio | §1.4 #1 + #5 + SOC 2 win condition | `COMPLIANCE_HIPAA_PCIDSS`, `SECURITY_WAF_SHIELD_MACIE` |

**13 stacks.** Each one traces back to a §1-§3 driver in the right column.

### §4.3 Data model

| Store | Table / bucket / index | Key design | Driven by §3.2 entity |
|---|---|---|---|
| RDS Aurora | `clients` | (id) | Client |
| RDS Aurora | `projects` | (client_id, id) | Project |
| RDS Aurora | `studies` | (tenant_id, id) | Study |
| RDS Aurora | `sessions` | (study_id, id) | Session |
| RDS Aurora | `participants` | (session_id, id) | Participant |
| RDS Aurora | `discussion_guides` | (study_id, id) | DiscussionGuide |
| RDS Aurora | `transcripts` (GIN full-text idx) | (session_id, id) | Transcript |
| RDS Aurora | `segments` | (session_id, id, low_confidence) | Segment |
| RDS Aurora | `insights` (JSONB) | (session_id) | Insight |
| RDS Aurora | `verbatims` + GSI on `pinned_report_ids` (GIN) | (study_id, id) | Verbatim — §3.4 #16 |
| RDS Aurora | `themes` | (study_id, id, parent_id) | Theme |
| RDS Aurora | `reports` | (study_id, id, status) | Report |
| RDS Aurora | `exports` | (report_id, id, format) | (export artifact) |
| RDS Aurora | `tenant_configs` | (tenant_id) | (per-client knobs) |
| DDB | `jobs` | PK `tenant#job_id` · GSIs: status, analyst, date | Job-state tracking |
| DDB | `audit_log` (TTL 7yr) | PK `tenant#date` · SK `actor#timestamp` | Activity (immutable) |
| DDB | `cost_ledger` | PK `tenant#date` · SK `job_id` | CostLedgerEntry |
| DDB | `session_state` (TTL = session_max) | PK `session_id` | (Cognito session shadow) |
| DDB | `rate_limit` (TTL 5min) | PK `tenant#endpoint` · SK `minute_bucket` | (per-tenant API throttling) |
| S3 | `audio-raw-{env}` (KMS audio CMK) | `uploads/{tenant}/{study}/{session}/<uuid>.<ext>` | Session.media_url |
| S3 | `transcripts-{env}` (KMS db CMK) | `transcripts/{tenant}/{session}.json` | Transcript |
| S3 | `reports-{env}` (KMS reports CMK; CF + OAC) | `reports/{tenant}/{report}/{version}.<ext>` | Report exports |
| S3 | `audit-{env}` (Object-Lock COMPLIANCE 7yr) | `audit/{tenant}/{date}/{event}.json` | Activity (mirror) |
| S3 Vectors | `quotes_idx` (1024-dim cosine) | metadata: tenant_id, study_id, session_id, sentiment | Verbatim (semantic) |
| S3 Vectors | `transcript_chunks_idx` | metadata: tenant_id, session_id | Segment (semantic) |
| S3 Vectors | `themes_idx` | metadata: tenant_id, study_id | Theme (semantic) |
| SSM Parameter Store | `/audio-analytics/prompts/<key>` (history kept) | path | PromptTemplate.versions |

### §4.4 Architecture non-negotiables

Global rules from `LAYER_BACKEND_LAMBDA §4.1` (every kit must enforce):

1. Lambda / container asset paths use `Path(__file__).resolve().parents[N] / "..."`. Never CWD-relative.
2. Cross-stack resource access is identity-side only. Never `bucket.grant_*(crossStackRole)`.
3. Cross-stack EventBridge → Lambda uses L1 `events.CfnRule` with static-ARN target.
4. Bucket + CloudFront OAC live in the SAME stack. Never split.
5. KMS ARNs cross-stack are STRINGS via SSM. Not imported L2 `kms.Key`.

Kit-specific additions (each cites its §1-§3 driver):

6. **Strict tenant isolation.** Every DDB / RDS / S3 / S3-Vectors query MUST include the `tenant_id` (Cognito `custom:tenant_id` claim) as PK or filter. IAM conditions enforce `aws:ResourceTag/tenant_id == request.tenant_id` on data-plane reads. No tenant-from-cookie. **(drives §1.4 anti-pattern #5)**

7. **Audio presigned URLs use Range-GET + per-clip-scoped path restriction.** A "seek to 04:22" URL authorises the byte range of that clip ONLY, not the full file. Built via S3 GetObject with Range header + scoped presigned URL. **(drives §1.4 anti-pattern #2)**

8. **Cost attribution on every Bedrock call.** Every `InvokeModel` wraps a decorator that writes to DDB `cost_ledger` with (tenant_id, study_id, job_id, model_id, input_tokens, output_tokens, usd_estimate) BEFORE returning. **(drives §3.4 #6 + §1.5 cost-per-session win condition)**

9. **PII redaction runs BEFORE embedding generation.** Embedding a transcript with respondent names bakes PII into vectors forever (vectors aren't easily redactable). `EmbeddingGeneration` step in SFN comes AFTER `PiiRedaction`. **(drives §1.4 anti-pattern #2)**

10. **Admin routes isolated to `/admin/*` server-side.** Lambda authorizer enforces Cognito group check on every `/admin/*` request. Frontend role-gates client-side for UX, but server-side authorization is the security boundary. **(drives §1.4 anti-pattern #6 — ClientViewer must not see costs)**

11. **Approval gate before client delivery.** Reports cannot transition from `in_review → final → delivered` without a Moderator-role action. State machine in `report_status` field; status transitions logged in audit_log. **(drives §1.4 anti-pattern #1 + §2.4 handoff #1)**

12. **Prompt promotion requires 2-admin approval + 10-transcript A/B.** SSM write to `candidate` triggers `PromptABTestWorkflow` SFN. Promotion requires (a) human-judge UI 2-admin approval click + (b) win-rate ≥ 50%. **(drives §1.4 anti-pattern #8 + §3.4 #7)**

### §4.5 Partial reference

**Existing partials (load in Claude prompts):**
- `LAYER_BACKEND_LAMBDA`, `LAYER_NETWORKING`, `LAYER_SECURITY`, `LAYER_DATA`, `LAYER_API`, `LAYER_FRONTEND`, `LAYER_OBSERVABILITY`
- `DATA_AURORA_SERVERLESS_V2` (canonical), `DATA_S3_VECTORS` (canonical)
- `WORKFLOW_STEP_FUNCTIONS`, `EVENT_DRIVEN_PATTERNS`, `EVENT_DRIVEN_FAN_IN_AGGREGATOR`
- `COMPLIANCE_HIPAA_PCIDSS`, `SECURITY_WAF_SHIELD_MACIE`
- `DATA_MULTITENANT_DDB` *(verify exists; if not, this kit's first new partial)*

**New partials this kit surfaces** (extract post-engagement per Canonical-Copy Rule):

1. **`PATTERN_AUDIO_TRANSCRIPTION_PIPELINE.md`** — Amazon Transcribe + SFN orchestration + speaker diarization + auto-language-id + retry/DLQ. Distinct from `MLOPS_AUDIO_PIPELINE` (acoustic ML).
2. **`PATTERN_QUOTE_MINING_WITH_TIMESTAMPS.md`** — Bedrock prompts for quote+timestamp+speaker-role+sentiment extraction; canonical DB + vector schema.
3. **`PATTERN_AI_REPORT_DRAFTING.md`** — Bedrock-driven DOCX/PPTX auto-draft with citation insertion + editable draft persistence.
4. **`PATTERN_OPERATIONAL_COST_TRACKING.md`** — DDB-based cost ledger (job → study → client), CloudWatch metric feeders, anomaly alarm.
5. **`PATTERN_AUDIO_SEEK_INTEGRATION.md`** — wavesurfer.js + timestamp URL params + range-GET S3 presigned URLs for jump-to-moment.
6. **`PATTERN_PROMPT_LAB_VERSIONING.md`** *(NEW for v2.0)* — SSM Parameter Store prompt versioning + A/B test SFN + human-judge UI + 2-admin promotion gate.
7. **`PATTERN_EVIDENCE_LINKED_LLM_OUTPUT.md`** *(NEW for v2.0)* — every LLM bullet/quote/score carries citations + confidence; UI surfaces evidence drawer.

---

## §5 — Delivery plan

### §5.1 Engagement shape
- **Total:** 2 weeks (10 working days) · 2 developers · 1 designer (50%) · 1 PM (25%)
- **Pricing:** $250K POC · path-to-prod build adds $1.2M–$2.5M
- **Milestones:** 30% on kickoff · 30% on Week-1 demo · 40% on UAT sign-off

### §5.2 Day-by-day execution

| Day | Template(s) | Partials Claude must load | Deliverable | Gating criteria |
|---|---|---|---|---|
| **1** | [`iac/02_cdk_ml_llm_infrastructure`](../iac/02_cdk_ml_llm_infrastructure.md) | `LAYER_BACKEND_LAMBDA`, `LAYER_NETWORKING`, `LAYER_SECURITY` | CDK app w/ 13 stacks scaffolded, all synth-clean | `cdk synth` succeeds for every stack |
| **2** | [`devops/02_vpc_networking_ml`](../devops/02_vpc_networking_ml.md), [`devops/08_kms_encryption_ml`](../devops/08_kms_encryption_ml.md) | `LAYER_NETWORKING`, `LAYER_SECURITY` | NetworkStack + SecurityStack deployed (PrivateLink for Transcribe/Bedrock/S3/RDS/DDB; 3 KMS CMKs) | All endpoints reachable; KMS keys exist; tenant tags on resources |
| **3** | Data layer | `DATA_AURORA_SERVERLESS_V2`, `LAYER_DATA`, `DATA_S3_VECTORS` | DataStack + MediaStack + VectorStack deployed; schema migrations applied; 3 vector indexes seeded | Sample query against Aurora returns rows; vectors searchable |
| **4** | [`devops/02`](../devops/02_vpc_networking_ml.md) + [`mlops/15_event_driven_inference`](../mlops/15_event_driven_inference.md) | `LAYER_API`, `WORKFLOW_STEP_FUNCTIONS`, `EVENT_DRIVEN_PATTERNS` | UploadApiStack + OrchestrationStack; S3 → EB → SFN wired; audio dropped triggers SFN | Sample audio drop triggers SFN execution to "started" |
| **5** | Bedrock prompts (§7 below) + [`mlops/24_bedrock_prompt_management`](../mlops/24_bedrock_prompt_management.md) | (prompt files in repo) | All 11 SFN Lambdas deployed; sample audio processes end-to-end (transcribe → analyze → persist); cost_ledger writes verified | E2E sample audio → insights row in RDS + vectors in S3 Vectors |
| **6** | AuthStack + InsightsApiStack + AdminApiStack | `LAYER_SECURITY`, `LAYER_API` | Cognito w/ 4 role groups + MFA on Admin; 4 demo users seeded; APIs auth-gated | All `/admin/*` reject non-admin tokens; tenant claim enforced server-side |
| **7** | [`mlops/25_react_portal_cloudfront`](../mlops/25_react_portal_cloudfront.md) | `LAYER_FRONTEND` | Frontend scaffold deployed (S3 + CF + OAC + WAF); login + protected routes work | Browser opens login, demo creds redirect by role |
| **8** | Analyst persona UI build (§3.1 routes) | `LAYER_FRONTEND` | All 7 analyst routes rendered with mock data; ⌘K palette wired; SessionInsights drill-down working; ReportBuilder drag-drop | Demo recipe steps 1-7 (§3.6) all work |
| **9** | Admin persona UI build | `LAYER_OBSERVABILITY`, `LAYER_FRONTEND` | All 6 admin routes rendered; pipeline diagram live from DDB; cost ledger live; audit CSV export works; Prompt Lab edit/promote flow works | Demo recipe step 8 (§3.6) works; CSV downloaded successfully |
| **10** | ObservabilityStack + ComplianceStack + UAT | `COMPLIANCE_HIPAA_PCIDSS`, `LAYER_OBSERVABILITY`, `SECURITY_WAF_SHIELD_MACIE` | Dashboards + alarms live; CloudTrail Lake + Object-Lock audit bucket + Macie on audio; UAT script executed; runbook + deployment guide complete; handover session done; AWS Public Use Case draft submitted | All §5.4 deliverables checked; §5.5 golden-set tests pass |

### §5.3 Phase 1 vs Phase 2

**Phase 1 (this 2-week POC, $250K):**
- All 17 routes render with rich mock data
- E2E pipeline: audio upload → transcribe → analyze → insights → audit log
- Multi-tenant isolation enforced (Cognito custom:tenant_id)
- Admin Ops cockpit · Cost Ledger · Audit · Prompt Lab · Health
- Report Builder with drag-drop + print-to-PDF
- ⌘K command palette (keyword search)
- 4 Bedrock prompts versioned in SSM, A/B test workflow
- 20 sample audio files processed for demo
- SOC 2-ready audit log + immutable storage

**Phase 2 (next engagement, $800K–$1.2M):**
- Real-time WebSocket streaming for Bedrock output
- Semantic search (S3 Vectors-backed; Bedrock Knowledge Base for cross-study)
- Inline transcript editing (Phase 1 is read-only)
- Drag-drop on Action Items kanban (Phase 1 is read-only)
- 2-way crosstabs (currently 1-way demographic × topic)
- Email + Slack notification channels (Phase 1 is in-app only)
- PPTX + DOCX rich export (Phase 1 is print-to-PDF only)
- Client share-link with token rotation
- Multi-language support (Phase 1 is en-US + es-US)
- HIPAA toggle ON for pharma clients (Phase 1 is SOC 2 only)
- Discussion-guide auto-alignment (Phase 1 is mock alignment)

**Phase 3 (SaaS productization, $2M+):**
- White-label tenant onboarding wizard for client SaaS resale
- Per-client custom branding + domain
- Cost-per-tenant billing rollup with Stripe integration
- Usage-based pricing tiers
- Public API for in-house insights team integration

### §5.4 Deliverables checklist

- [ ] CDK Python app, 13 stacks, all synth-clean
- [ ] Step Functions AudioProcessingWorkflow deployed + executes E2E on sample audio
- [ ] Step Functions PromptABTestWorkflow deployed + executes against test corpus
- [ ] All 11 Lambda functions deployed + IAM-scoped
- [ ] RDS Aurora Postgres v2 with 14-table schema + migrations applied
- [ ] 5 DDB tables (jobs, audit_log, cost_ledger, session_state, rate_limit) with TTLs configured
- [ ] 3 S3 Vectors indexes (quotes, transcript_chunks, themes) with seed data
- [ ] 4 SSM Parameter Store prompt entries versioned (v1, v2 active)
- [ ] Cognito user pool with 4 role groups + MFA for Admin + 4 POC test users seeded
- [ ] React SPA deployed to CloudFront + OAC + WAF
- [ ] All 7 analyst routes render with MOCK_MODE off (real API)
- [ ] All 6 admin routes render with MOCK_MODE off (real API)
- [ ] Admin ops cockpit live (5-sec refresh from DDB)
- [ ] Cost Ledger shows real cost data (cost_ledger writes verified)
- [ ] Audit CSV export downloads + validates against schema
- [ ] Prompt Lab: edit prompt → save as new version → A/B run completes → promote works
- [ ] Report Builder: drag-reorder → save → print-to-PDF works
- [ ] Cross-study ⌘K palette returns matches across tenants (admin only)
- [ ] CloudWatch dashboards (ops + cost + quality) operational
- [ ] WAF + Macie + CloudTrail Lake + Object-Lock audit bucket in place
- [ ] 20+ mock audio files processed end-to-end for demo
- [ ] UAT script + outcomes doc
- [ ] Runbook + deployment guide
- [ ] AWS Public Use Case draft submitted to AWS PR team
- [ ] Handover session with Research America team
- [ ] Demo screen-record (2 min) saved as `kits/_design/qra-v2-demo.mp4`

### §5.5 Golden-set tests

Nightly regression. 20 sample audio files across verticals. Failing any = pager + deployment freeze.

| Assertion | Metric | Pass threshold | Driver |
|---|---|---|---|
| E2E latency (60-min recording) | total SFN duration | p95 < 15 min | §1.5 |
| Transcribe WER on sampled reference | WER | < 5% (en-US clean), < 8% (accented) | §1.5 |
| Quote extraction recall vs human golden | Recall@30 | > 80% | §1.5 |
| Sentiment accuracy vs human-labeled | F1 | > 0.75 | §1.5 |
| PII redaction recall on synthetic-PII audio | Recall | > 99% (zero-leak target) | §1.4 #2, §1.5 |
| Cross-study search recall@5 | Recall@5 | > 85% | §3.4 #12 |
| Admin ops cockpit refresh latency | p95 DDB read | < 200 ms | §1.5 |
| Audio-seek click → audio playing delay | p95 | < 500 ms | §3.4 #6 audio-seek |
| Prompt A/B win-rate computation accuracy | matches manual recount | 100% | §1.4 #8 |
| Cost-per-session attribution accuracy | matches Cost Explorer ±5% | within 5% | §3.4 #6, §1.5 |

### §5.6 Handover & go-live runbook

Documents handed over:
- `runbook.md` — incident response, common ops tasks, on-call procedures
- `architecture-decisions.md` — ADRs for the 12 non-negotiables (§4.4)
- `deployment-guide.md` — bootstrap → deploy → seed → verify steps
- `uat-outcomes.md` — UAT script results with screenshots
- `client-onboarding.md` — adding a new tenant playbook (Cognito user pool, S3 prefix, RDS row, KMS grant)
- `cost-model.md` — cost-per-session math + projection at 10x volume

UAT script: at `docs/uat/qra-v2-uat.md` in the engagement repo. Includes 8 personas-times-3-flows (24 walkthroughs).

Production cutover plan:
- Stage 1: dev tenant, internal testing only (Days 1-30)
- Stage 2: stage tenant, 1 paying client real audio (Days 30-60)
- Stage 3: prod tenant, full Research America load (Day 60+)
- Backout: blue-green via per-stage CDK env. SLA to roll back: 15 min.

---

## §6 — Kit-wide parameters

```
# --- Identity -----------------------------------------------------------
PROJECT_NAME:                qra-{client_slug}
AWS_REGION:                  us-east-1
AWS_ACCOUNT_ID:              [12-digit]
ENV:                         dev | stage | prod
CLIENT_SLUG:                 research_america
TARGET_LANGUAGE:             python

# --- DOMAIN PACK --------------------------------------------------------
DOMAIN_PACK:                 qualitative_research
CLIENT_VERTICALS:            [pharma, retail, finance, healthcare, automotive]
RECORDING_TYPES:             [focus_group, in_depth_interview, telephone_survey, ethnography]

# --- SCALE --------------------------------------------------------------
EXPECTED_DAILY_LOAD:         250
EXPECTED_PEAK_HOURLY_LOAD:   40
MAX_RECORDING_DURATION_MIN:  180
AUDIO_FORMATS:               [mp3, wav, m4a, mp4, aac, flac]
MAX_FILE_SIZE_MB:            2048

# --- TRANSCRIPTION ------------------------------------------------------
TRANSCRIBE_LANGUAGES:        [en-US, es-US]
SPEAKER_DIARIZATION:         true
MAX_SPEAKERS:                10
CUSTOM_VOCAB_BY_VERTICAL:    true
AUTO_LANGUAGE_ID:            true

# --- AI ANALYSIS --------------------------------------------------------
SUMMARIZATION_MODEL_ID:      us.anthropic.claude-sonnet-4-7-20260109-v1:0
CODING_MODEL_ID:             us.anthropic.claude-haiku-4-5-20251001-v1:0
EMBEDDING_MODEL_ID:          amazon.titan-embed-text-v2:0
EMBEDDING_DIM:               1024
SENTIMENT_METHOD:            bedrock
QUOTE_MAX_PER_SESSION:       30
PER_QUOTE_SENTIMENT:         true
CROSS_SESSION_AGGREGATION:   enabled
REPORT_DRAFTING:             enabled

# --- PII / GOVERNANCE ---------------------------------------------------
PII_REDACTION_ENABLED:       true
PII_REDACTION_TARGETS:       [name, phone, email, ssn, address, dob, medical_record_number]
TRANSCRIBE_CONTENT_REDACTION: true
BEDROCK_GUARDRAILS_ENABLED:  true
HIPAA_READY_FLAG:            false

# --- STORAGE ------------------------------------------------------------
RDS_ENGINE:                  aurora_postgres_v16
RDS_SERVERLESS_V2:           true
RDS_MIN_ACU:                 0.5
RDS_MAX_ACU:                 8
DDB_TABLES:                  [jobs, audit_log, cost_ledger, session_state, rate_limit]
S3_VECTOR_INDEXES:           [quotes, transcript_chunks, themes]

# --- MULTI-TENANCY ------------------------------------------------------
TENANT_MODEL:                per_client_isolation
TENANT_PK_STRATEGY:          client_id_prefix_on_every_pk
CROSS_TENANT_DENY:           strict

# --- RETENTION ----------------------------------------------------------
RETENTION_DAYS_AUDIO:        90
RETENTION_DAYS_TRANSCRIPT:   365
RETENTION_DAYS_INSIGHTS:     2555
RETENTION_DAYS_AUDIT_LOG:    2555
S3_LIFECYCLE_TO_GLACIER_DAYS: 180

# --- AUTH ---------------------------------------------------------------
AUTH:                        cognito_user_pool
ROLES:                       [Admin, Analyst, Moderator, ClientViewer]
SAML_FEDERATION_READY:       true
MFA_REQUIRED_ROLES:          [Admin]
SESSION_MAX_MINUTES:         60

# --- COST GUARDRAILS ----------------------------------------------------
COST_BUDGET_MONTHLY_USD:     25000
COST_PER_JOB_TARGET_USD:     3.00
TRANSCRIBE_DAILY_BUDGET_MIN: 15000
BEDROCK_DAILY_TOKEN_BUDGET:  5_000_000
COST_ANOMALY_ALARM_PCT:      30

# --- UI / UX ------------------------------------------------------------
UI_FRAMEWORK:                react_vite
UI_LIBRARY:                  shadcn_ui_tailwind     # ← CHANGED in v2.0 (was tailwind_radix raw)
AUDIO_PLAYER_WAVEFORM:       true
ADMIN_DASHBOARD_REFRESH_SEC: 5
EXPORT_FORMATS:              [docx, pptx, pdf, md, csv]
DARK_MODE:                   true                    # ← NEW in v2.0
COMMAND_PALETTE_ENABLED:     true                    # ← NEW in v2.0 (⌘K)

# --- COMPLIANCE ---------------------------------------------------------
COMPLIANCE_STANDARD:         SOC2
CLOUDTRAIL_ENABLED:          true
CLOUDTRAIL_LAKE_ENABLED:     true                    # ← NEW in v2.0 (audit query layer)
CONFIG_RULES_ENABLED:        true
MACIE_ON_AUDIO_BUCKET:       true
OBJECT_LOCK_AUDIT_BUCKET:    true
```

---

## §7 — Bedrock prompt library

Templates at `cdk/lambda/bedrock_analysis/prompts/v1/`, versioned in SSM Parameter Store at `/audio-analytics/prompts/<key>` (admin-managed via `/admin/prompts` Prompt Lab).

- `summary.txt` — 3–5 executive bullets w/ evidence citations to transcript spans (`InsightSummaryBullet[]`)
- `sentiment.txt` — per-turn sentiment, weighted by word count, moderator excluded; aggregates to session score + per-minute timeline
- `key-topics.txt` — top 10 topics w/ mentions, sentiment, 1–3 quotes (drives `TopicBubbles`)
- `action-items.txt` — 3–5 actions w/ owner (function), priority (low/med/high), starting verb
- `quotes.txt` — 15–30 verbatims w/ speaker, timestamp, theme tag, sentiment, salience score (drives `Verbatim[]`)
- `themes.txt` — codebook proposal (5-10 themes w/ frequency)
- `report_draft.docx.txt` / `report_draft.pptx.txt` — structured report drafts

Per-vertical overrides at `prompts/v1/<vertical>/<prompt>.txt` (e.g. `pharma/quotes.txt` knows about drug names, patient-journey framing).

---

## §8 — Composition patterns

See [`kits/README.md §Kit composition patterns`](README.md#kit-composition-patterns). This kit composes naturally with:
- **`ai-native-lakehouse`** — adds structured-data side (DataZone-governed query of insights table); Phase 3 SaaS productization.
- **`deep-research-agent`** — supervisor agent that orchestrates QRA cross-study search + external sources for competitive intelligence.

---

## §9 — Pricing & ROI math

See [`kits/_design/qualitative-research-audio-analytics.md`](_design/qualitative-research-audio-analytics.md) for the full version.

**Headline:** $19.7K/mo AWS displaces $1.6M/mo of researcher labour. **80:1 leverage.** Annual savings per client: $19.2M. POC pricing $250K is recouped in <1 month of post-deployment savings. Production build $1.2M-$2.5M is recouped in <2 months.

---

## §10 — Competitive positioning

| Incumbent | What they do | Where this wins | Where they win |
|---|---|---|---|
| **Qualtrics** | DIY survey + AI-summarise | Deeper qualitative methodology, evidence-linked outputs | Better self-serve UX, larger panel |
| **Dscout** | Mobile-first qual, AI summarise | Multi-persona platform (admin/analyst/moderator), enterprise-grade audit, multi-tenant isolation | Mobile-native field-app |
| **User Interviews** | Recruit + interview | Built-in analysis pipeline, not just recording | Better recruiting marketplace |
| **Bain Vector / McKinsey QB / BCG GAMMA** | In-house consultancy AI | Decoupled platform — sells to research firms vs. building competitor | Deeper consulting wraparound |

Demo-day anchor stories: PepsiCo Zero Sugar Gen-Z relaunch (CPG) · Pfizer Vyndamax cardiologist IDIs (pharma) · Toyota RAV4 family driver IDIs (auto) · Neutrogena Crisis Response (brand crisis) · LEGO AFOL ethnography (entertainment).

---

## §11 — Changelog

| Version | Date | Change |
|---|---|---|
| 1.0 | 2026-04-23 | Initial kit. Technology-first structure (architecture led, business framing as wrap-around). 5 new partials surfaced. |
| **2.0** | **2026-04-23** | **Reference implementation for [Business-First Kit Standard v1.0](_template/README.md). Restructured §1 (P&L lever first) → §2 (4 personas + day-in-the-life narratives + 10 business artifacts + 6 cross-persona handoffs) → §3 (frontend reference spec: 17 routes + 40+ domain types + 11 viz components + 20 standout features + persona ribbons + reference frontend pointer at `E:\NBS_Research_America_regen\frontend\`) → §4 (architecture annotated with §1-§3 drivers; 13 stacks; 12 non-negotiables traced to §1-§3) → §5 (delivery plan unchanged; added Phase 3 SaaS productization roadmap). 2 new partials surfaced: PROMPT_LAB_VERSIONING + EVIDENCE_LINKED_LLM_OUTPUT. Supersedes v1.0; v1.0 removed.** |
