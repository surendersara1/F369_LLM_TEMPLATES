<!-- Template Version: 2.0 | F369 Wave 8 (composite) | Composes: MLOPS_CROSS_ACCOUNT_DEPLOY + MLOPS_LINEAGE_TRACKING + MLOPS_MODEL_MONITOR_ADVANCED -->

# Template 30 — Enterprise ML Governance (3-account · cross-account RAM share · drift detection · model lineage · auto-rollback)

## Purpose

Enterprise-grade ML governance for regulated organisations: 3-account isolation (training → staging → prod) with RAM share for Model Package Groups, full lineage tracking, 4-type drift detection (data + model + bias + feature attribution), auto-rollback on drift, and Model Card per release.

Generates production-deployable CDK across 3 accounts + cross-account governance Lambda + compliance dashboards.

---

## Role Definition

You are an expert AWS ML governance + multi-account architect with deep expertise in:
- Multi-account org structure (training / staging / prod accounts) + AWS Organizations + SCPs
- AWS RAM (Resource Access Manager) for cross-account Model Package Group sharing
- Cross-account IAM trust patterns (KMS, ECR, S3, MPG resource policies)
- ML Lineage Tracking + Model Cards
- Model Monitor (4 types: data quality, model quality, bias drift, feature attribution drift)
- Auto-rollback wiring on CloudWatch alarms
- Cross-account observability (CloudWatch cross-account, central audit account)

Generate complete production-deployable code, no TODOs.

---

## Context and Inputs

```
PROJECT_NAME:           [REQUIRED]
TRAINING_ACCOUNT:       [REQUIRED — 12-digit, owns Pipelines + MLflow + MPG]
STAGING_ACCOUNT:        [REQUIRED — 12-digit, owns staging endpoint + golden-set tests]
PROD_ACCOUNT:           [REQUIRED — 12-digit, owns prod endpoint + auto-rollback]
AUDIT_ACCOUNT:          [OPTIONAL — for centralised observability]

AWS_REGION:             [REQUIRED]
ENV:                    [REQUIRED — typically prod]

# --- MODEL ---
MODEL_PACKAGE_GROUP:    [REQUIRED — name of MPG to share across accounts]

# --- APPROVAL FLOW ---
AUTO_PROMOTE_STAGING_TO_PROD: [OPTIONAL — true if golden-set passes]
GOLDEN_SET_PASS_THRESHOLD: [REQUIRED — 0.85 default]
MANUAL_PROD_APPROVAL:     [OPTIONAL — true for healthcare/finance]

# --- MONITORING ---
MONITOR_TYPES:          [REQUIRED — list from: data_quality, model_quality, bias, attribution]
MONITOR_SCHEDULE:        [OPTIONAL — cron(0 8 * * ? *) daily]
AUTO_ROLLBACK_ON_DRIFT:  [OPTIONAL — true; with 3-period evaluation to avoid flapping]

# --- LINEAGE ---
ENABLE_LINEAGE:          [REQUIRED — true]
CREATE_MODEL_CARD:       [REQUIRED — true]
GDPR_DATA_SUBJECT_TAGS:  [OPTIONAL — true for "delete my data" support]

# --- COMPLIANCE ---
COMPLIANCE_STANDARD:     [SOC2 | HIPAA | GDPR | PCI_DSS]
KMS_KEY_ARN_TRAINING:    [REQUIRED]
KMS_KEY_ARN_STAGING:     [REQUIRED]
KMS_KEY_ARN_PROD:        [REQUIRED]
ORG_ID:                  [REQUIRED — for SCP scope-down]
```

---

## Partial Library

| Partial | Why |
|---|---|
| `MLOPS_CROSS_ACCOUNT_DEPLOY` | 3-account ML governance + RAM share + cross-account deployer Lambda |
| `MLOPS_LINEAGE_TRACKING` | Auto-capture from Pipelines + Model Card + GDPR queries |
| `MLOPS_MODEL_MONITOR_ADVANCED` | 4-monitor pattern + auto-rollback wiring |
| `MLOPS_SAGEMAKER_SERVING` | Endpoint deployment, blue/green, data capture |
| `LAYER_SECURITY` | IAM permission boundary + KMS cross-account grants |
| `COMPLIANCE_HIPAA_PCIDSS` | Compliance overlay |

---

## Architecture (3-account)

```
   ┌──────────────────────────────────────────────────────────────────┐
   │  TRAINING ACCOUNT (111111111111)                                  │
   │     - Pipelines (training + eval + register)                       │
   │     - MLflow Tracking Server                                       │
   │     - Model Package Group: qra-mpg                                 │
   │     - RAM share to staging + prod                                  │
   │     - Lineage entities + Model Cards                               │
   └──────────────────┬───────────────────────────────────────────────┘
                      │
                      │  EventBridge: ModelPackage Approved → SNS bridge
                      ▼
   ┌──────────────────────────────────────────────────────────────────┐
   │  STAGING ACCOUNT (222222222222)                                   │
   │     - DeployerLambda picks up shared MPG                          │
   │     - Staging endpoint                                             │
   │     - Golden-set test runner (auto, on endpoint InService)        │
   │     - Model Monitor (data + model quality, daily)                  │
   │     - On test pass: SNS bridge → prod account                      │
   └──────────────────┬───────────────────────────────────────────────┘
                      │
                      │  Auto OR manual gate (per MANUAL_PROD_APPROVAL)
                      ▼
   ┌──────────────────────────────────────────────────────────────────┐
   │  PROD ACCOUNT (333333333333)                                      │
   │     - DeployerLambda picks up validated MPG                       │
   │     - Blue/Green deployment to prod endpoint                       │
   │     - Model Monitor (all 4 types — data, model, bias, attribution)│
   │     - Auto-rollback Lambda on drift alarm (3-period evaluation)    │
   └──────────────────┬───────────────────────────────────────────────┘
                      │
                      ▼
   ┌──────────────────────────────────────────────────────────────────┐
   │  AUDIT ACCOUNT (optional)                                         │
   │     - CloudWatch cross-account observability                      │
   │     - Aggregated lineage queries (compliance reporting)            │
   │     - SOC 2 evidence package automation                             │
   └──────────────────────────────────────────────────────────────────┘
```

---

## Day-by-day execution (10-day POC)

### Day 1-2 — Cross-account foundation
- AWS Organizations OU structure (training + staging + prod OUs)
- SCP at each OU level (least privilege)
- 3 KMS CMKs (one per account, region-locked)
- IAM Identity Center for human access
- ECR cross-account pull policies
- **Deliverable:** SCPs prevent cross-account principal escalation; KMS keys exist

### Day 3-4 — Training account: MPG + RAM share
- Model Package Group in training account
- RAM resource share to staging + prod accounts
- MPG resource policy with `AccountPrincipal` for downstream accounts
- S3 artifacts bucket policy + KMS grant for cross-account decrypt
- ECR repo policy for cross-account image pull
- EventBridge rule on `ModelPackageStateChange.Approved` → notify Lambda → SNS bridge to staging
- **Deliverable:** approved MPG visible in staging account via `aws ram list-resource-share-associations`

### Day 5-6 — Staging account: deployer + golden-set tests
- DeployerLambda subscribes to staging SNS topic; creates staging endpoint
- Golden-set test runner Lambda (runs after endpoint InService)
- Model Monitor (data quality + model quality) on staging endpoint
- On golden-set pass + threshold met: SNS bridge to prod account
- **Deliverable:** end-to-end: training approves → staging deploys → tests run → notification fires

### Day 7-8 — Prod account: blue/green + monitoring
- DeployerLambda for prod (subscribes to staging's SNS bridge)
- Blue/green endpoint deployment (variant weights 100/0 → 50/50 → 100/0)
- Model Monitor (all 4 types: data quality + model quality + bias drift + feature attribution drift)
- Daily monitoring jobs + CloudWatch alarms
- Auto-rollback Lambda (fires on alarm `state=ALARM` for 3 consecutive periods)
- **Deliverable:** prod deployment via blue/green; auto-rollback drill works

### Day 9 — Lineage + Model Cards
- Auto-capture Lineage entities for training run
- Model Card auto-generated per Approved Model Package (training data uri, hyperparams, eval, ethical considerations)
- GDPR data subject query Lambda (input: data subject ID → output: list of models trained on their data)
- **Deliverable:** Model Card visible per approved package; GDPR query works end-to-end

### Day 10 — UAT + observability
- CloudWatch cross-account observability (audit account)
- Compliance dashboard (per-account: ApprovalCount, DriftAlarmCount, RollbackEvents)
- DR drill: simulate drift → verify auto-rollback → verify alert routing
- **Deliverable:** UAT passes; SOC 2 / HIPAA evidence packages auto-generated

---

## Validation criteria

- [ ] **Cross-account RAM share** working — staging can see training's MPG
- [ ] **3-stage promotion flow** end-to-end without manual intervention (when AUTO_PROMOTE=true)
- [ ] **Manual gate** enforced when MANUAL_PROD_APPROVAL=true
- [ ] **All 4 monitors** running daily; alarms fire on synthetic drift
- [ ] **Auto-rollback** triggers within 15 min of 3-consecutive-period alarm
- [ ] **Lineage graph** complete: dataset → training → MPG → staging endpoint → prod endpoint
- [ ] **Model Card** auto-populated per release
- [ ] **GDPR query** returns models trained on test subject data
- [ ] **Idempotent deployers** — duplicate event doesn't double-deploy
- [ ] **Per-account KMS isolation** — staging cannot decrypt prod artifacts (different CMKs)

---

## Output artifacts

1. CDK apps (3 accounts: TrainingShareStack, StagingDeployStack, ProdDeployStack)
2. AWS Organizations OU structure + SCPs
3. EventBridge cross-account bridge config (SNS topics + rules)
4. DeployerLambda code (parametrized for staging vs prod)
5. Golden-set test runner Lambda + sample test set
6. 4-monitor CDK + auto-rollback Lambda
7. Model Card template + auto-population script
8. GDPR data subject query Lambda
9. Cross-account observability config
10. SOC 2 / HIPAA evidence-package generator script
11. DR drill runbook
12. User access runbook (how to approve a model, how to override auto-rollback)

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 2.0 | 2026-04-26 | Initial composite. Composes Cross-Account Deploy + Lineage + Model Monitor Advanced. Wave 8. |
