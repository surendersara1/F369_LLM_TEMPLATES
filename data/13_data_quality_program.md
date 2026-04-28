<!-- Template Version: 2.0 | F369 Wave 19 (composite) | Composes: DATA_GLUE_QUALITY + DATA_GLUE_CATALOG + DATA_DATAZONE_V2 + LAYER_OBSERVABILITY -->

# Template 13 — Data Quality Program (Glue Data Quality · DQDL · data contracts · drift detection · SLO dashboards · 4-6 week deploy)

## Purpose

Stand up a **production-grade data quality program** in 4-6 weeks. Output: 50+ DQDL rulesets across critical tables, recommendations engine in CI for new tables, data contracts enforced across producer-consumer boundaries, drift detection, SLO dashboards, on-call escalation.

This is the **canonical "improve our data quality" engagement** — every regulated/data-driven enterprise needs this. Pairs directly with `enterprise/13_data_mesh_governance` for full governance.

---

## Role Definition

You are an expert AWS data quality + governance architect with deep expertise in:
- AWS Glue Data Quality (DQDL syntax, recommendations engine, evaluation)
- Data contracts pattern (producer-consumer agreement on schema + rules + SLO)
- Drift detection (statistical baselines + ML-based anomaly detection)
- CloudWatch metrics + alarms for data SLOs
- EventBridge → SNS / Slack / PagerDuty alert routing
- Glue ETL inline DQ vs scheduled DQ trade-offs
- Data quality dashboard design

Generate complete, production-deployable code. No TODOs.

---

## Context and Inputs

```
PROJECT_NAME:                [REQUIRED]
AWS_REGION:                  [REQUIRED]
ENV:                         [REQUIRED]

# --- TABLES IN SCOPE ---
TABLES:                      [REQUIRED — JSON array of {database, table, owner_team, criticality}:
                              [
                                {"database":"prod_orders","table":"orders","owner":"orders-team","criticality":"high"},
                                {"database":"prod_users","table":"users","owner":"users-team","criticality":"high"},
                                {"database":"prod_inventory","table":"products","owner":"inventory-team","criticality":"medium"}
                              ]
                             ]

# --- VALIDATION TIMING ---
INLINE_DQ_TABLES:            [comma-separated table names with critical inline DQ]
SCHEDULED_DQ_TABLES:         [comma-separated tables with scheduled DQ (post-load)]

# --- CONTRACTS ---
ENABLE_DATA_CONTRACTS:       [true default for prod]
CONTRACT_REPO:               [REQUIRED if enabled — Git repo URL for contracts]
CONTRACT_REVIEW_WORKFLOW:    [REQUIRED — github_pr | gitlab_mr | manual]

# --- DRIFT DETECTION ---
ENABLE_DRIFT:                [true default]
BASELINE_REFRESH_DAYS:       [90 default — re-baseline quarterly]
DRIFT_THRESHOLD_STDDEV:      [2 default — alert at 2σ]

# --- ALERTING ---
DEFAULT_ALERT_DESTINATION:   [REQUIRED — sns | slack | pagerduty]
PAGERDUTY_INTEGRATION_KEY:   [REQUIRED if pagerduty for high-critical]
SLACK_WEBHOOK_URL:           [REQUIRED if slack]
ESCALATION_FOR_CRITICAL:     [REQUIRED — pagerduty default]

# --- SLO DASHBOARDS ---
ENABLE_DATAZONE_SLO_LINK:    [true if DataZone in use]

# --- COMPLIANCE ---
KMS_KEY_ARN:                 [REQUIRED]
LOG_RETENTION_DAYS:          [90 default; 365 for regulated]

# --- OBSERVABILITY ---
SNS_ALARM_TOPIC_ARN:         [REQUIRED]
DASHBOARD_VIEWERS:           [comma-separated IDC group names]
```

---

## Partial Library (Claude MUST load)

| Partial | Why |
|---|---|
| `DATA_GLUE_QUALITY` | Glue DQ + DQDL + recommendations + scheduled rules + drift |
| `DATA_GLUE_CATALOG` | Catalog tables + schema |
| `DATA_DATAZONE_V2` | (if integrated) link DQ results to data products |
| `EVENT_DRIVEN_PATTERNS` | EventBridge for DQ failure routing |
| `LAYER_OBSERVABILITY` | CW dashboards + alarms |
| `LAYER_SECURITY` | KMS for DQ result S3 |

---

## Architecture

```
   Per-table DQDL ruleset (Git-managed)
        │
        ▼
   ┌─────────────────────────────────────────────────────────────────┐
   │ Glue Data Quality                                                │
   │   Rulesets:                                                       │
   │     - prod_orders.orders          (40 rules; inline + scheduled)  │
   │     - prod_users.users             (35 rules; inline + scheduled)  │
   │     - prod_inventory.products      (25 rules; scheduled)            │
   │   Evaluation:                                                      │
   │     - Inline (in Glue ETL job)                                       │
   │     - Scheduled (EventBridge cron)                                    │
   │     - Recommendations engine (new tables → suggested DQDL)            │
   │   Drift detection:                                                   │
   │     - Per-table baseline (mean, stddev, distinct counts)               │
   │     - Daily comparison; alert if drift > threshold                      │
   │   Outputs:                                                            │
   │     - CloudWatch metrics (DataQuality.{rule}.PassRate)                  │
   │     - S3 DQ result history (per ruleset, per evaluation)                 │
   │     - EventBridge events (Glue Data Quality Evaluation Results)          │
   └─────────────────────────────────────────────────────────────────┘
                        │
        ┌───────────────┼───────────────────────────────────────────┐
        ▼               ▼                                            ▼
   ┌──────────┐    ┌──────────────────────┐              ┌──────────────────┐
   │ CW       │    │ EventBridge          │              │ S3 DQ history    │
   │ Dashboard│    │   Filter:             │              │ (queryable via   │
   │ (per-    │    │     runState=FAILED   │              │  Athena)          │
   │ table    │    │   Routes:              │              │                   │
   │ SLO,     │    │     critical → PD       │              │                   │
   │ trend)   │    │     high → Slack #data  │              │                   │
   └──────────┘    │     medium → email      │              └──────────────────┘
                   │     low → log only       │
                   │                          │
                   │  Auto-remediation:      │
                   │     quarantine bad data  │
                   │     OR rollback ETL      │
                   │     OR notify only        │
                   └──────────────────────────┘
                        │
                        ▼
                   ┌──────────────────────────┐
                   │ Data Council review     │
                   │   - Weekly DQ review     │
                   │   - SLO breaches tracked │
                   │   - Producer ownership   │
                   └──────────────────────────┘
```

---

## Day-by-day execution (4-6 week deploy, 1-2 dev + 1 PM)

### Week 1 — Inventory + recommendations
- Inventory all `TABLES` in scope; categorize by criticality
- For each table, run Glue DQ recommendations engine → auto-generate baseline DQDL
- Producer team owners review + refine DQDL (add domain-specific rules)
- Store DQDL in Git (`data-contracts/<table>.dqdl`)
- **Deliverable:** End of Week 1: 80% of tables have reviewed DQDL ready for evaluation.

### Week 2 — Inline + scheduled DQ deployment
- For `INLINE_DQ_TABLES`: update Glue ETL jobs with `EvaluateDataQuality` transform
- For `SCHEDULED_DQ_TABLES`: EventBridge cron rule + Lambda triggering DQ runs
- Deploy DQDL rulesets via CDK
- Run first evaluation; capture baseline metrics
- **Deliverable:** End of Week 2: 50% of tables have active DQ evaluations.

### Week 3 — Alert routing + dashboards
- EventBridge rules for DQ failures:
  - High criticality → PagerDuty
  - Medium → Slack channel
  - Low → email digest
- CloudWatch dashboards per table:
  - Pass rate trend
  - Failed rules
  - Volume / freshness
  - Drift indicators
- 3+ alarms per critical table (pass-rate < 95%, freshness > SLO, volume drop)
- **Deliverable:** End of Week 3: alerts firing correctly; dashboards live.

### Week 4 — Drift detection + remediation
- Capture baseline statistics per critical table (mean, stddev, distinct counts, row count)
- Add drift rules to DQDL: `Mean "amount" between X and Y` based on baseline ± stddev × 2
- Auto-baseline refresh every 90 days via Lambda
- Auto-remediation Lambda templates (3 patterns):
  - Quarantine: move bad data to `_quarantine` prefix; investigate
  - Rollback: revert ETL output to last good snapshot
  - Notify-only: alert producer team
- **Deliverable:** End of Week 4: drift detection active; auto-remediation tested for 1 table.

### Week 5 — Data contracts (if enabled)
- Define data contract YAML schema
- Migrate top-10 critical tables to formal contracts in `CONTRACT_REPO`
- Producer-consumer review workflow via PRs
- CI validation: contract changes require approval from impacted consumers
- Auto-generate DQDL from contract
- **Deliverable:** End of Week 5: 10 contracts in place; PR-based change workflow.

### Week 6 — Data Council + ongoing ops + handoff
- Establish weekly Data Council meeting (cross-team)
- Review SLO breaches, drift events, contract changes
- Documentation: runbook for producer/consumer/oncall
- DataZone integration (if applicable): link DQ results to data products
- Monthly trend report template
- **Deliverable:** End of Week 6: Data Council active; on-call rotation established; handoff complete.

---

## Validation criteria

- [ ] **All tables in scope have DQDL rulesets** (Glue DataQualityRuleset count == TABLES count)
- [ ] **Inline DQ active** in Glue ETL jobs for INLINE_DQ_TABLES
- [ ] **Scheduled DQ runs** for SCHEDULED_DQ_TABLES (EventBridge rules verified firing)
- [ ] **DQ pass rate ≥ 95%** for all critical tables (CW metric)
- [ ] **Alert routing works** — test failed evaluation triggers correct destination
- [ ] **CW dashboards live** with > 7 days of data
- [ ] **3+ alarms per critical table** in OK baseline
- [ ] **Drift baseline captured** for each critical table
- [ ] **Auto-remediation tested** for 1 critical table (forced bad data → handled)
- [ ] **Data contracts** in Git for top-10 critical tables (if enabled)
- [ ] **PR-based contract review** working (test by submitting a contract change PR)
- [ ] **DataZone integration** (if applicable): DQ results visible on data product page
- [ ] **Data Council meeting cadence** established + first meeting held
- [ ] **Runbooks signed off** by producer + consumer + oncall

---

## Common gotchas (claude must address proactively)

- **DQ rules can be over-strict** at first — pass rate < 80% common in week 1. Iterate with team.
- **Recommendations engine takes 5-30 min per table** — schedule for batch processing.
- **Inline DQ adds 10-50% to ETL runtime** — for huge tables, scheduled DQ post-load is cheaper.
- **Drift threshold tuning** — start at 2σ; adjust based on false positive rate.
- **Auto-remediation is risky** — start with notify-only; promote to quarantine after team trust; rollback only with senior eng.
- **Data contracts feel bureaucratic** — start with high-criticality tables; expand gradually.
- **Producer pushback** — "this is more work for us." Frame as quality + downstream value. Show consumer satisfaction post-program.
- **CW alarm fatigue** — too many alarms = ignored alarms. Tier by criticality; minimize medium/low alerts.
- **DQ history in S3** can grow large — lifecycle policy: 90d hot, 1y archive, 7y compliance.
- **Cost** — Glue DQ runs cost compute. Per-table per-day = $0.50-5 typical.

---

## Output artifacts

1. **CDK stacks**:
   - `DqRulesetsStack` — DQDL rulesets per table
   - `DqEvaluationStack` — scheduled rules + EventBridge
   - `DqAlertingStack` — EventBridge → SNS / Slack / PagerDuty routing
   - `DqDriftStack` — baseline capture + comparison Lambdas
   - `DqRemediationStack` — quarantine + rollback Lambdas (per-table parametrized)
   - `DqObservabilityStack` — dashboards + alarms

2. **DQDL rulesets** (Git-managed) per table

3. **Data contracts** (YAML, Git-managed) for top-10 critical tables

4. **Drift baseline definitions** per critical table (initial + refresh schedule)

5. **CloudWatch dashboards** — one per table + global program overview

6. **Alarms** — 3-5 per critical table; ~3 per medium

7. **Auto-remediation Lambda templates** (3 patterns)

8. **Runbooks** (Markdown):
   - Producer team: how to update DQDL when schema changes
   - Consumer team: how to subscribe to DQ failures
   - On-call: how to triage failed evaluations

9. **Pytest validation suite** covering all criteria

10. **Data Council artifacts**:
    - Weekly meeting agenda template
    - Quarterly SLO review template
    - Annual data quality report template

11. **Cost projection** — Glue DQ runs + storage + alerts

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 2.0 | 2026-04-28 | Initial composite template. Glue DQ + DQDL + contracts + drift + alerts + dashboards. 4-6 week program. Wave 19. |
