<!-- Template Version: 2.0 | F369 Wave 18 (composite) | Composes: AI_DEV_Q_DEVELOPER + ENTERPRISE_IDENTITY_CENTER -->

# Template 19 — Q Developer Team Rollout (Pro tier · Customizations · IDC SSO · adoption telemetry · 1-2 week deploy)

## Purpose

Roll out **Amazon Q Developer Pro** to a 50+ developer organization in 1-2 weeks. Output: IDC-integrated subscription, Customizations indexed on private codebase, IDE plugins distributed, adoption metrics dashboard, training program, governance + cost controls.

This is the **canonical "AI coding assistant for enterprise" engagement** — directly tied to "AI-augmented engineering" CIO pitch + productivity multiplier story.

---

## Role Definition

You are an expert AWS GenAI productivity architect with deep expertise in:
- Amazon Q Developer Pro setup + Customizations
- IAM Identity Center federation (Azure AD / Okta) + group-based subscriptions
- IDE plugin distribution (VSCode, JetBrains, VS, Cloud9, SM Studio)
- Customizations source code staging in S3
- Adoption telemetry (CloudWatch metrics) + ROI tracking
- Q Developer governance (content controls, code references, anonymous suggestions)
- Change management for new tooling

Generate complete, production-deployable code. No TODOs.

---

## Context and Inputs

```
PROJECT_NAME:                [REQUIRED]
AWS_REGION:                  [REQUIRED]
ENTERPRISE_NAME:             [REQUIRED]

# --- ORG ---
TOTAL_DEV_USERS:             [REQUIRED — drives cost projection]
SUBSCRIPTION_TIER:           [pro default for 50+ orgs]
IDC_INSTANCE_ARN:            [REQUIRED]

# --- DEVELOPER GROUPS (assigned subscriptions) ---
DEV_GROUPS:                  [REQUIRED — comma-separated IDC group names; e.g. Engineering,DataScience,Platform]

# --- CUSTOMIZATIONS ---
ENABLE_CUSTOMIZATIONS:       [true default for orgs > 50 devs]
CODEBASE_REPOS:              [REQUIRED if customizations — list of repos to index]
EXCLUDED_PATHS:              [comma-separated; .git/, node_modules/, dist/, build/]
INCLUDED_EXTENSIONS:         [py,ts,js,java,go,rb,cs,kt default]

# --- IDE TARGETS ---
TARGET_IDES:                 [REQUIRED — vscode,jetbrains,visualstudio (whichever team uses)]
DISTRIBUTE_VIA:              [REQUIRED — manual | mdm (corporate-managed) | scripted]

# --- GOVERNANCE ---
BLOCK_CODE_REFERENCES:       [false default — true for legal-strict orgs]
ALLOW_ANONYMOUS_SUGGESTIONS: [false default — false = best privacy posture]
RESTRICTED_LANGUAGES:        [comma-separated; empty = all languages allowed]

# --- CHANGE MGMT ---
TRAINING_FORMAT:             [video default — also: workshop | self-paced]
ADOPTION_TARGET_WEEK_4:      [50 default — % of subscribed users active]

# --- COMPLIANCE ---
KMS_KEY_ARN:                 [REQUIRED]

# --- OBSERVABILITY ---
SNS_ALARM_TOPIC_ARN:         [REQUIRED]
DASHBOARD_VIEWERS:           [comma-separated IDC group names — for engineering leaders]
```

---

## Partial Library (Claude MUST load)

| Partial | Why |
|---|---|
| `AI_DEV_Q_DEVELOPER` | Q Developer Pro + IDE plugins + Customizations + agentic commands + governance |
| `ENTERPRISE_IDENTITY_CENTER` | IDC + SCIM + group provisioning + Permission Sets |
| `LAYER_SECURITY` | KMS for Customizations source bucket |
| `LAYER_OBSERVABILITY` | CW metrics + alarms |

---

## Architecture

```
   Engineering Org (50+ devs)
        │
        │ SSO via IAM Identity Center
        ▼
   ┌──────────────────────────────────────────────────────────────┐
   │ AWS Q Developer Pro                                           │
   │   $19/user/mo × N users                                        │
   │   Subscriptions assigned to IDC groups                          │
   │     - Engineering → Pro                                          │
   │     - DataScience → Pro                                          │
   │     - Platform → Pro                                              │
   │     - Sales/Finance → not subscribed (no need)                    │
   │                                                                  │
   │   Customizations: indexed on private codebase                    │
   │     Source: S3 bucket of monorepo (KMS-encrypted)                 │
   │     Re-indexed: weekly via Lambda                                  │
   │     Coverage: ~95% of services                                      │
   │                                                                  │
   │   Governance:                                                    │
   │     - Block code references (legal posture)                       │
   │     - Anonymous suggestions OFF                                    │
   │     - Telemetry collection enabled (admin only)                    │
   └──────────────────────────────────────────────────────────────┘
                            │
        ┌───────────────────┼─────────────────────────┐
        ▼                   ▼                          ▼
   Developer A        Developer B               Developer C
   VSCode             JetBrains                  Visual Studio
   Q plugin           Q plugin                    Q plugin
        │                   │                          │
        ▼                   ▼                          ▼
   Inline + chat       Inline + chat              Inline + chat
   /dev /test          /dev /test                  /dev /test
   /review             /review                     /review
   /transform          /transform                  /transform
                            │
                            ▼
                    CloudWatch Metrics
                    (DAU, suggestions, acceptance rate, agentic invocations)
                            │
                            ▼
                    Adoption Dashboard
                    (engineering leaders + IT ops)
```

---

## Day-by-day execution (1-2 week deploy)

### Day 1-2 — IDC + subscriptions
- IDC pre-flight: confirm `DEV_GROUPS` synced via SCIM
- Subscribe groups to Q Developer Pro (console: Q Developer admin → Subscriptions → Add Group)
- Validate billing model (per active user / month)
- Create separate IDC group for early adopters (initial pilot)
- **Deliverable:** End of Day 2: 5-10 pilot users have Pro subscriptions assigned + can sign in.

### Day 3-4 — Customizations setup (Pro)
- (If `ENABLE_CUSTOMIZATIONS`) S3 bucket for Customizations source (KMS-encrypted, read-only access for Q service)
- Lambda + EventBridge schedule to sync `CODEBASE_REPOS` to S3 weekly (excluding `EXCLUDED_PATHS`)
- Console: Q Developer → Customizations → Create
  - Source: S3 bucket
  - File extensions: `INCLUDED_EXTENSIONS`
  - Encryption: KMS CMK
- Indexing runs (~1-4h for typical monorepo)
- Activate Customization for `DEV_GROUPS`
- Verify: pilot user gets context-aware suggestions matching team conventions
- **Deliverable:** End of Day 4: Customizations active; pilot user demo shows internal library suggestions.

### Day 5 — IDE plugin distribution
- (If `DISTRIBUTE_VIA=manual`) Wiki page with install instructions per IDE
- (If `DISTRIBUTE_VIA=mdm`) Corporate MDM (Jamf, Intune, etc.) deploys plugin to all dev workstations
- (If `DISTRIBUTE_VIA=scripted`) `bootstrap-q.sh` script run by devs at onboarding
- Validate plugin sign-in via IDC SSO flow on 5 test workstations
- **Deliverable:** End of Day 5: pilot users have plugin installed + signed in across IDEs.

### Day 6-7 — Pilot rollout (5-10 users)
- Onboard pilot users:
  - 30-min onboarding video
  - Cheat sheet (canonical use cases: inline + chat + /dev + /test + /review)
  - Slack channel #q-developer-pilot for Q&A
  - Optional 1:1 sessions for power users
- Collect feedback: weekly survey + observed usage
- **Deliverable:** Pilot week complete; first ROI signals (suggestion acceptance rate, time-to-PR-open).

### Day 8-10 — Phased rollout to broader org
- Week 2 / Day 8: roll out to Tier 1 group (most engaged team)
- Day 9: Tier 2 (early-adopter teams)
- Day 10: Tier 3 (rest of org)
- Each rollout includes: announcement + onboarding video + Slack support
- **Deliverable:** End of Week 2: all `TOTAL_DEV_USERS` subscribed + signed in.

### Day 11-12 — Adoption tracking + governance + handoff
- CloudWatch dashboard built: DAU, suggestion acceptance rate, agentic invocations
- 4 alarms:
  - DAU drops below 50% of subscribed (engagement issue)
  - Suggestion acceptance rate < 20% (training issue)
  - Cost > 110% of projected (over-subscription)
  - Customization re-indexing failures
- Governance:
  - Quarterly cost review
  - License compliance audit (referenced code citations)
  - User feedback collection
- Handoff doc to engineering leaders
- **Deliverable:** Dashboard live + adoption ≥ `ADOPTION_TARGET_WEEK_4`% by end of Week 4 (post-deploy).

---

## Validation criteria

- [ ] **All `DEV_GROUPS` subscribed to Pro** (verified via Q Developer console)
- [ ] **IDC SSO works** for plugin sign-in (test on 5 workstations)
- [ ] **Customizations ACTIVE** + indexed (`status: ACTIVE`)
- [ ] **Customization re-indexing scheduled** weekly (Lambda + EventBridge)
- [ ] **CloudWatch metrics populated** — DAU, suggestion count, acceptance rate
- [ ] **Pilot users see context-aware suggestions** matching team conventions (manual demo)
- [ ] **Adoption dashboard accessible** to engineering leaders
- [ ] **Training materials published** (video + cheat sheet + Slack channel)
- [ ] **4 alarms** OK baseline; firing on injected anomalies
- [ ] **Cost projection vs actual** within 10%
- [ ] **Adoption rate week 4 ≥ `ADOPTION_TARGET_WEEK_4`%**
- [ ] **Suggestion acceptance rate week 4 ≥ 25%**
- [ ] **Code references blocked** (if enabled) — verified via test prompt
- [ ] **Anonymous suggestions OFF** — verified in admin

---

## Common gotchas (claude must address proactively)

- **MAU billing surprise** — $19 × 200 devs = $3800/mo. Run cost projection upfront.
- **Customizations indexing time** — 1-4h for typical codebase. Allow 1 day buffer.
- **Customizations re-indexing** must be scheduled (Lambda + EventBridge) to keep suggestions current.
- **Adoption stalls** without proper change management. Allocate dedicated champion per team.
- **Suggestion acceptance < 20% week 4** = training gap. Run targeted workshop.
- **Code references** flag suggestions matching public OSS — block in admin if legal posture requires.
- **License risk** — Q tries hard to be original; ~5-10% suggestions cite. Always review citations.
- **Privacy / Anonymous suggestions** — OFF means Q doesn't use your code to train. Verify in admin.
- **Cross-account** — for multi-account orgs, install in single management account; use IDC for users.
- **CLI separately** — `q chat` CLI has separate setup; bundle install scripts.
- **JetBrains plugin auto-update** — devs must manually update; communicate version upgrades.
- **Onboarding fatigue** — don't overwhelm new joiners with too many tools. Phase Q with onboarding.

---

## Output artifacts

1. **CDK stacks**:
   - `QDeveloperIdcStack` — IDC group subscriptions
   - `QCustomizationsStack` — S3 bucket + Lambda + EventBridge for re-indexing
   - `QObservabilityStack` — CW dashboard + alarms

2. **Customizations source sync Lambda** — pulls `CODEBASE_REPOS` to S3 weekly

3. **IDE plugin install scripts** — `install-q-vscode.sh`, `install-q-jetbrains.sh`, `install-q-vs.ps1`

4. **MDM deployment configs** (if applicable) — Jamf/Intune profiles

5. **Onboarding kit**:
   - 30-min training video script
   - Cheat sheet PDF (inline + chat + /dev /test /review /transform)
   - FAQ wiki page
   - Slack channel + community guidelines

6. **Pytest validation suite**

7. **CloudWatch dashboard** — adoption + cost overview

8. **4 alarms YAML** wired to SNS

9. **Cost model**:
   - Pro $19/MAU × N users
   - Customizations cost (free for indexing; storage included in S3)
   - Re-index Lambda runtime cost

10. **ROI baseline** — measured 30/60/90 days post-launch:
    - Time-to-first-commit (vs baseline)
    - Lines of code per dev per week (vs baseline)
    - PR-open rate (vs baseline)
    - Self-reported satisfaction (5-pt scale)

11. **Quarterly governance review template** — usage by team, cost trend, license review

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 2.0 | 2026-04-28 | Initial composite template. Q Developer Pro rollout for 50+ devs in 1-2 weeks; Customizations + IDC + adoption tracking. Wave 18. |
