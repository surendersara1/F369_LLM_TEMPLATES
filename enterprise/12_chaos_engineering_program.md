<!-- Template Version: 2.0 | F369 Wave 14 (composite) | Composes: DR_RESILIENCE_HUB_FIS + DR_ROUTE53_ARC + EKS_OBSERVABILITY + LAYER_OBSERVABILITY -->

# Template 12 — Chaos Engineering Program (Resilience Hub · FIS · game days · SLO-guarded experiments · resilience score CI gate)

## Purpose

Stand up a **production-grade chaos engineering program in 4-6 weeks**. Replaces "we hope DR works" with "we proved DR works yesterday and the day before." Applies to any AWS workload — not just multi-region; can target single-region resilience (AZ failure, DB failure, dependency failure).

This is the **canonical "we keep getting paged for stuff that 'shouldn't have happened'" engagement** — chaos engineering surfaces hidden assumptions, missing alarms, untested failure modes.

Output: Resilience Hub assessments + FIS experiments + game day cadence + SLO-guarded automation + retrospective culture.

---

## Role Definition

You are an expert AWS chaos engineering practitioner with deep expertise in:
- AWS Resilience Hub assessments + recommendations
- AWS Fault Injection Service (FIS) — Action Library + Experiment Templates + Stop Conditions
- Hypothesis-driven experiment design (per Principles of Chaos)
- SLO definition + CW alarm authoring as stop conditions
- Game day facilitation (preparation, execution, retrospective)
- Resilience score tracking in CI/CD
- Anti-patterns: chaos in prod without practice, missing observability, no rollback plan

Generate complete, production-deployable code. No TODOs.

---

## Context and Inputs

```
PROJECT_NAME:                [REQUIRED]
AWS_REGION:                  [REQUIRED]
ENV:                         [REQUIRED — typically prod, but practice in stage first]

# --- TARGET WORKLOAD ---
WORKLOAD_NAME:               [REQUIRED]
WORKLOAD_RUNTIME:            [REQUIRED — ec2 | ecs_fargate | ecs_ec2 | eks | lambda | mixed]
WORKLOAD_DB:                 [REQUIRED — aurora | rds | ddb | none]
SLO_AVAILABILITY:            [REQUIRED — e.g. 99.9 (= 43 min/mo)]
SLO_LATENCY_P99_MS:          [REQUIRED]
SLO_ERROR_RATE_PCT:          [REQUIRED — e.g. 1.0]

# --- EXPERIMENT SCOPE ---
START_IN_ENV:                [REQUIRED — stage | prod_canary | prod_full]
                             # Always start in stage; promote after 30 days clean
EXPERIMENTS_INITIAL:         [REQUIRED — comma-separated; e.g. ec2_kill,ecs_kill_task,rds_failover,az_partition,latency_inject]
GAME_DAY_CADENCE:            [REQUIRED — weekly | monthly | quarterly]

# --- SAFETY ---
STOP_CONDITION_ALARMS:       [REQUIRED — CW alarm ARNs that stop FIS if breached]
EXPERIMENT_TIME_WINDOW:      [business_hours | off_hours default — when experiments run]
EXPERIMENT_BLACKOUT_DATES:   [comma-separated — e.g. 2026-12-25,2026-12-31,2027-01-01]
EXPERIMENT_TAG:              [FisExperiment default — IAM scoping tag]

# --- RESILIENCE HUB ---
RESILIENCE_POLICY_TIER:      [REQUIRED — MissionCritical | Important | CoreServices | Standard]
ASSESSMENT_CADENCE:          [REQUIRED — daily | weekly | per_deploy]
SCORE_THRESHOLD:             [80 default — block deploys if score drops below]

# --- OBSERVABILITY ---
SNS_ALARM_TOPIC_ARN:         [REQUIRED]
PAGERDUTY_INTEGRATION_KEY:   [REQUIRED]
SLACK_WEBHOOK_URL:           [REQUIRED for game day comms]

# --- COMPLIANCE ---
KMS_KEY_ARN:                 [REQUIRED]
```

---

## Partial Library (Claude MUST load)

| Partial | Why |
|---|---|
| `DR_RESILIENCE_HUB_FIS` | Resilience Hub + FIS detailed setup |
| `DR_ROUTE53_ARC` | (if running region-failure experiments) |
| `EKS_OBSERVABILITY` | (if EKS workload) Container Insights for experiment monitoring |
| `LAYER_OBSERVABILITY` | CW dashboards + alarms baseline (stop conditions) |
| `LAYER_SECURITY` | KMS + IAM tag-based scoping for FIS |

---

## Architecture (program-level)

```
   ┌─────────────────────────────────────────────────────────────────┐
   │ Workload (target of chaos)                                       │
   │   - SLO-monitored (CW alarms for availability, latency, errors)  │
   │   - Synthetic monitoring (Canaries running every 1 min)          │
   └─────────────────────────────┬───────────────────────────────────┘
                                 │
                                 │ chaos injection
                                 ▼
   ┌─────────────────────────────────────────────────────────────────┐
   │ AWS Fault Injection Service                                      │
   │   - 8+ experiment templates                                      │
   │   - Stop conditions: SLO alarms                                  │
   │   - IAM scoped to tag `FisExperiment: allowed`                    │
   │   - Schedule: EventBridge rule weekly Thursday 2pm                │
   └─────────────────────────────┬───────────────────────────────────┘
                                 │ measure
                                 ▼
   ┌─────────────────────────────────────────────────────────────────┐
   │ AWS Resilience Hub                                               │
   │   - Application: workload                                          │
   │   - Resiliency Policy: tier-based RPO/RTO targets                  │
   │   - Assessments run on every deploy + daily                         │
   │   - Score tracked over time                                          │
   │   - Recommendations exported to Jira                                 │
   └─────────────────────────────┬───────────────────────────────────┘
                                 │ feeds
                                 ▼
   ┌─────────────────────────────────────────────────────────────────┐
   │ Game Day Cadence                                                 │
   │   Weekly: scheduled FIS experiments (low-risk, off-hours)         │
   │   Monthly: facilitated game day (1 hour, 1 scenario)              │
   │   Quarterly: executive game day (multi-team, full DR drill)        │
   │   Annual: full chaos festival (1 week, multiple scenarios)         │
   └─────────────────────────────┬───────────────────────────────────┘
                                 │ output
                                 ▼
   ┌─────────────────────────────────────────────────────────────────┐
   │ Retrospective + Improvement                                      │
   │   - Hypothesis vs reality (what we learned)                        │
   │   - Action items → Jira (track to closure)                          │
   │   - Runbook updates                                                  │
   │   - Resilience score trend                                            │
   └─────────────────────────────────────────────────────────────────┘
```

---

## Day-by-day execution (4-6 week deployment, 1 SRE/dev)

### Week 1 — SLOs + observability + Resilience Hub setup
- Define SLOs explicitly: availability (99.9 = 43 min/mo), latency p99 (e.g., 500ms), error rate (1%)
- Author CW alarms for each SLO violation (these become FIS stop conditions)
- Synthetic monitoring (Canaries) for end-to-end user-flow probing
- Resilience Hub: import workload as App; assign resiliency policy by tier
- Run baseline assessment — capture initial score
- **Deliverable:** All SLOs measured; resilience score baseline = 70+ typical first run.

### Week 2 — FIS setup + first experiments in stage
- IAM role for FIS with tag-based scoping
- Tag stage resources with `FisExperiment: allowed`
- 5 initial experiments (most universal):
  1. `ec2_stop_random_instance` — kill 1 random EC2 in target ASG
  2. `ecs_stop_random_task` — kill 1 random ECS task
  3. `rds_failover_cluster` — Aurora cluster failover (test reconnection)
  4. `network_latency_inject` — 200ms latency on outbound HTTPS for 5 min
  5. `network_packet_loss` — 5% packet loss for 5 min
- Each experiment has CW alarm stop conditions
- Run each experiment manually in stage; observe behavior
- **Deliverable:** End of Week 2: 5 experiments runnable in stage; all auto-stop on SLO breach.

### Week 3 — Schedule + tag-based dynamic targeting
- EventBridge rule: weekly Thursday 2pm runs random experiment in stage (if no blackout)
- Lambda picks random experiment from list, starts FIS via boto3
- Slack notification: "Chaos experiment X starting at 2pm; stops in 5 min OR on SLO breach"
- Tag-based dynamic targeting (target_tag selection)
- Run 4 scheduled experiments in stage; review behavior + CW dashboard
- **Deliverable:** Automated weekly chaos in stage with comms.

### Week 4 — Resilience score CI gate + game day prep
- CI/CD pipeline integration:
  - On every deploy: trigger Resilience Hub assessment
  - Compare score against `SCORE_THRESHOLD` (80)
  - Fail deploy if score regresses > 5 points
- Game day playbook authored with 5 scenarios:
  1. Database failure
  2. Single-AZ network partition
  3. Service dependency timeout (downstream API slow)
  4. Cache failure (ElastiCache eviction storm)
  5. Region partition (full DR drill)
- Multi-team invitees identified (SRE, app team, security, customer support, business)
- **Deliverable:** First scheduled game day for end of Week 5.

### Week 5 — First game day (in stage) + retrospective
- 1-hour kickoff: scenario walkthrough; hypothesis stated; success criteria
- 30-min FIS experiment + observation
- 30-min retrospective:
  - Hypothesis vs reality (PASS/FAIL)
  - Top 3 surprises
  - Action items → Jira
- Improve runbook based on learnings; close action items
- **Deliverable:** First game day completed; ≥ 5 action items captured + assigned.

### Week 6 — Promote to prod (canary first)
- After 30 days clean in stage (FIS auto-stop never triggered, all experiments pass), promote to prod canary
- Tag prod canary resources `FisExperiment: allowed`
- First prod experiment: `ec2_stop_random_instance` (lowest blast radius)
- Observe for 1 week; if all-clear, expand to other prod experiments
- **Deliverable:** First prod experiment successfully completed.

---

## Validation criteria

- [ ] **SLOs explicitly defined** + CW alarms in place
- [ ] **Synthetic monitoring (Canaries)** running every 1 min for end-to-end user flows
- [ ] **Resilience Hub baseline score** captured; trend tracked weekly
- [ ] **FIS IAM role tag-scoped** (`aws:ResourceTag/FisExperiment: allowed` condition)
- [ ] **All FIS experiments have CW alarm stop conditions**
- [ ] **Weekly scheduled experiment in stage** runs without manual intervention
- [ ] **Resilience score CI gate** in pipeline; tested with intentional regression
- [ ] **First game day held** with multi-team observation
- [ ] **Retrospective document** captured + action items in Jira
- [ ] **30-day clean run in stage** before promoting to prod canary
- [ ] **Slack/PagerDuty notifications** firing during experiments
- [ ] **Blackout dates honored** — experiments do NOT run on holidays / release windows

---

## Common gotchas (claude must address proactively)

- **Don't run chaos in prod on day 1.** Practice in stage 30+ days minimum. Build muscle memory + trust.
- **Stop conditions are evaluated every 30 sec** — fast-failing experiments may breach SLO before stop fires. Add explicit `duration` in action params.
- **Tag-based IAM is the primary safety mechanism.** Without `aws:ResourceTag/FisExperiment` condition, FIS can act on production untagged resources.
- **Synthetic monitoring is non-negotiable.** Internal CW metrics may not reflect customer-impact. Synthetics = ground truth.
- **Game day retrospectives matter more than experiments.** Without rigorous retro + action item closure, you're paying for entertainment, not learning.
- **Resilience Hub recommendations are best-effort.** Score is a heuristic, not gospel. Engineering judgment overrides.
- **CI gate threshold drift** — start with 80, may drop to 75 as more resources tracked. Re-baseline quarterly.
- **Chaos in prod NEVER on Friday afternoons.** Always within business hours when on-call is staffed.
- **EventBridge rule for scheduled FIS** — must NOT pre-stage role with admin perms. Use the FIS role only.
- **PagerDuty page suppression during scheduled experiments** — otherwise on-call gets paged for chaos. Schedule maintenance windows.
- **First experiments often reveal more about observability gaps than resilience gaps.** "We can't tell if we recovered." Fix monitoring first; chaos test later.
- **Cost monitoring**: FIS at $0.10/action-minute. Frequent experiments × 100 instances = noticeable bill. Set CW billing alarm.

---

## Output artifacts

1. **CDK stacks**:
   - `ChaosObservabilityStack` — CW alarms + Synthetics canaries
   - `ResilienceHubStack` — App + policy + assessments
   - `FisStack` — Role + 5+ experiment templates + stop conditions
   - `GameDayAutomationStack` — EventBridge schedule + Lambda invoker + Slack notifications

2. **5+ FIS experiment templates** as JSON (one per scenario)
3. **CW alarms** for each SLO (availability, latency p99, error rate, dependency latency, retry count)
4. **CloudWatch Synthetics canaries** — 3+ for critical user flows
5. **Resilience Hub resiliency policy** with explicit RPO/RTO per disruption type
6. **CI/CD pipeline integration** (CodeBuild buildspec / GitHub Actions workflow):
   - Run Resilience Hub assessment on every deploy
   - Compare score; block on regression
7. **Game day playbook** (Markdown) — 5+ scenarios, each with:
   - Hypothesis
   - Setup
   - Execution steps
   - Observation rubric
   - Expected behavior
   - Retrospective template
8. **Quarterly chaos calendar** + PagerDuty maintenance windows + Slack announcement bot
9. **Pytest validation suite** — covers all validation criteria
10. **CW dashboard** — chaos observability:
    - Active experiments
    - Recent experiment outcomes
    - SLO trend during/after experiments
    - Resilience Hub score trend
11. **Retrospective template** + action item tracking spreadsheet
12. **Blackout-date Lambda** — pre-experiment check that skips on holidays/release windows

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 2.0 | 2026-04-27 | Initial composite template. Composes DR_RESILIENCE_HUB_FIS + ROUTE53_ARC + observability. 4-6 week chaos program. Wave 14. |
