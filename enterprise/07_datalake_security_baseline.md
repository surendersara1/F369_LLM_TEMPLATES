<!-- Template Version: 2.0 | F369 Wave 8 (composite) | Composes: SECURITY_DATALAKE_CHECKLIST + DATA_LAKE_FORMATION + LAYER_SECURITY + COMPLIANCE_HIPAA_PCIDSS -->

# Template 07 — Data Lake Security Baseline (30-control composite for SOC 2 / HIPAA / GDPR / PCI-DSS)

## Purpose

Apply a defensible 30-control security baseline to a data lake / lakehouse engagement. Auditor-ready posture spanning identity & access, encryption, residency, classification, threat detection, audit, and drift detection. Includes a **daily audit Lambda** that runs all 30 controls and pages on FAIL.

Generates production-deployable CDK + audit Lambda + CloudWatch dashboards.

---

## Role Definition

You are an expert AWS security architect with deep expertise in:
- Lake Formation strict mode (`IAMAllowedPrincipals` revocation, LF-TBAC, hybrid access mode)
- KMS-per-zone CMK design (raw / curated / consumer trust boundaries)
- S3 Object Lock COMPLIANCE mode (7-year retention)
- Macie sensitive-data discovery jobs + EventBridge findings routing
- GuardDuty S3 protection + finding triage
- CloudTrail Lake (queryable audit retention)
- AWS Config rules for drift detection
- IAM Access Analyzer for external-access findings
- HIPAA / SOC 2 / GDPR / PCI-DSS compliance mapping

Generate complete production-deployable code, no TODOs.

---

## Context and Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED — dev | stage | prod]

# --- COMPLIANCE FRAMEWORK ---
COMPLIANCE_STANDARD:    [REQUIRED — SOC2 | HIPAA | GDPR | PCI_DSS | MULTIPLE]
HIPAA_BAA_SIGNED:       [if HIPAA — true required]
GDPR_DATA_SUBJECTS:     [if GDPR — list of countries/regions]
PCI_LEVEL:              [if PCI_DSS — 1 | 2 | 3 | 4]

# --- DATA LAKE STRUCTURE ---
RAW_BUCKET:             [REQUIRED — name of raw zone bucket]
CURATED_BUCKET:         [REQUIRED — name of curated zone bucket]
CONSUMER_BUCKET:        [REQUIRED — name of consumer zone bucket]
AUDIT_BUCKET:           [REQUIRED — name of audit bucket; will get Object Lock]

# --- ORG DETAILS (for Block-Public + cross-account scoping) ---
ORG_ID:                 [REQUIRED — o-xxxx, for `aws:PrincipalOrgID` policy]
LF_ADMIN_ROLE_ARN:      [REQUIRED — central LF admin IAM role]

# --- TUNABLES ---
MACIE_SAMPLE_PERCENT:   [OPTIONAL — 100 prod / 10 dev]
OBJECT_LOCK_RETENTION_DAYS: [OPTIONAL — 2557 (7yr) for prod / 30 dev]
CT_LAKE_RETENTION_DAYS: [OPTIONAL — 2557 prod / 365 dev]
DAILY_AUDIT_TIME_UTC:   [OPTIONAL — 06:00 default]
ALARM_TOPIC_ARN:        [REQUIRED — SNS topic for findings]
```

---

## Partial Library

| Partial | Why |
|---|---|
| `SECURITY_DATALAKE_CHECKLIST` | The 30-control composite + daily audit Lambda |
| `DATA_LAKE_FORMATION` | LF strict mode + LF-TBAC + cross-account RAM |
| `LAYER_SECURITY` | KMS keys + IAM permission boundaries |
| `COMPLIANCE_HIPAA_PCIDSS` | HIPAA + PCI overlay (Audit + Backup Vault Lock + Macie + Config) |
| `SECURITY_WAF_SHIELD_MACIE` | WAF + Shield + Macie classification jobs |
| `LAYER_OBSERVABILITY` | CloudWatch dashboards + alarm wiring |

---

## The 30 controls (summary; full detail in SECURITY_DATALAKE_CHECKLIST §2)

**§2.1 Identity & Access (10):**
1. Lake Formation registered as data-lake admin
2. LF-TBAC tag taxonomy (3+ dimensions)
3. All Glue tables tagged
4. `IAMAllowedPrincipals.Super` revoked
5. Cross-account via RAM (not direct IAM)
6. Data-cells filters for PII columns
7. Hybrid access mode disabled in prod
8. Bucket policies deny `aws:SecureTransport=false`
9. Bucket policies deny non-org access
10. IAM Access Analyzer findings = 0

**§2.2 Encryption (5):**
11. All buckets KMS-CMK encrypted
12. Per-zone CMKs (raw / curated / consumer)
13. KMS rotation enabled
14. SSE-S3 PutObject denied
15. RDS / DDB / Redshift use same CMKs

**§2.3 Residency / Tenancy (4):**
16. Buckets in approved regions only
17. DynamoDB Global Tables in approved regions
18. Tenant prefix on every S3 key
19. Per-tenant KMS encryption context

**§2.4 Data Classification (3):**
20. Macie weekly classification scan on raw
21. Macie findings → EventBridge → Slack/PagerDuty (HIGH severity)
22. Bedrock Guardrails enabled for any `InvokeModel` over data

**§2.5 Threat Detection (3):**
23. GuardDuty + S3 protection ON
24. GuardDuty findings → EventBridge → ticket
25. CloudTrail data events on PII buckets

**§2.6 Audit & Retention (3):**
26. CloudTrail Lake (7yr)
27. S3 Object Lock COMPLIANCE on audit bucket (7yr)
28. S3 Inventory daily on raw + curated

**§2.7 Drift Detection (2):**
29. AWS Config recording all regions
30. Config rules: 8 baseline managed rules

---

## Day-by-day execution (5-day POC)

### Day 1 — Encryption + bucket policies
- 3 KMS CMKs (raw, curated, consumer) with annual rotation
- All 4 buckets (raw, curated, consumer, audit) KMS-encrypted
- Bucket policies: deny SSL-false, deny non-org access, deny SSE-S3 PutObject
- Audit bucket Object Lock COMPLIANCE 7yr
- **Deliverable:** all controls #11-#15, #8-#9, #27 enforced

### Day 2 — Lake Formation strict mode
- LF-TBAC tag taxonomy (`domain`, `sensitivity`, `access_tier`)
- Tag every Glue database + table
- Custom resource Lambda: `PutDataLakeSettings` revokes `IAMAllowedPrincipals.Super`
- Hybrid access mode disabled in prod
- Cross-account share via RAM (one-shot example)
- **Deliverable:** controls #1-#7

### Day 3 — Detection + classification
- GuardDuty detector with S3 protection
- Macie weekly classification job (sample 100% prod, 10% dev)
- EventBridge rules: GuardDuty severity ≥ 7 → SNS; Macie HIGH → SNS
- CloudTrail trail with data events on raw bucket
- **Deliverable:** controls #20-#25

### Day 4 — Audit + drift
- CloudTrail Lake event data store (7yr retention, multi-region)
- AWS Config recorder + delivery channel + 8 managed rules
- IAM Access Analyzer (account-level)
- S3 Inventory daily on raw + curated
- **Deliverable:** controls #26, #28-#30

### Day 5 — Daily audit Lambda + UAT
- Audit Lambda code (per `SECURITY_DATALAKE_CHECKLIST §5`) — runs all 30 controls
- EventBridge schedule cron(0 6 * * ? *) — 06:00 UTC daily
- SNS → Slack/PagerDuty on FAIL
- UAT: simulate 5 failures (e.g. add public bucket policy, disable rotation) → audit catches all
- **Deliverable:** auditor-ready summary report; 30/30 controls pass

---

## Validation criteria

- [ ] All 30 controls verified passing via daily audit Lambda
- [ ] SOC 2 Type II auditor reviews trail + signs off (no findings on data layer)
- [ ] Macie scan finds expected PII patterns in test data
- [ ] GuardDuty finding (test trigger) routed to SNS within 15 min
- [ ] Object Lock prevents bucket deletion in prod
- [ ] LF enforcement: unauthorized role blocked at table+column level
- [ ] Cost: ≤ $1,700/mo for 100 TB lake (per SECURITY_DATALAKE_CHECKLIST §6.1)

---

## Compliance mapping (claude generates per applicable framework)

| Control | SOC 2 | HIPAA | GDPR | PCI-DSS |
|---|---|---|---|---|
| #1-7 LF governance | CC6.1 | §164.308(a)(3) | Art.32 | 7.1 |
| #11-15 encryption | CC6.1 | §164.312(a)(2)(iv) | Art.32(1)(a) | 3.5 |
| #20 Macie | CC6.7 | §164.308(a)(1) | Art.30 | 11.6 |
| #23-25 GuardDuty + CloudTrail | CC7.2 | §164.312(b) | Art.33 | 10.6 |
| #26-27 audit retention | CC7.4 | §164.312(b) | Art.30 | 10.7 |
| #29-30 Config | CC8.1 | §164.310(a)(2)(iii) | Art.32 | 11.5 |

---

## Output artifacts

1. SecurityBaselineStack CDK
2. Audit Lambda (`audit.handler`) — 30-control runner
3. EventBridge schedule + alarms
4. Compliance mapping spreadsheet
5. Sample SOC 2 / HIPAA evidence package (CSV exports)
6. Drill-test runbook (simulate breach, verify detection)

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 2.0 | 2026-04-26 | Initial composite. Composes 30-control checklist + LF + KMS + HIPAA/PCI overlay. Wave 8. |
