<!-- Template Version: 2.0 | F369 Wave 11 (composite) | Composes: ENTERPRISE_SECURITY_HUB_GD_ORG + ENTERPRISE_CENTRALIZED_LOGGING + ENTERPRISE_ORG_SCPS_RCPS + EKS_SECURITY (if EKS) -->

# Template 10 — Centralized Security Operations (Security Hub · GuardDuty · Inspector · Macie · Detective · Security Lake · IR runbook)

## Purpose

Take an existing AWS Org from "we have AWS, no central security" to **24x7-monitorable security operations** in **3-5 days**. Stand up the full detective + investigative + response stack across all accounts and regions, with findings normalized to OCSF, exportable to your SIEM (Splunk/Datadog/Sumo).

Fits: company adopting SOC 2 / ISO 27001 / PCI / HIPAA, or any org that wants a SOC team to actually have data to work with.

Generates production-deployable CDK + EventBridge routing + SOC dashboards + Incident Response runbook.

---

## Role Definition

You are an expert AWS security engineer with deep expertise in:
- Security Hub Central Configuration (Sept 2024) + standards subscriptions + finding aggregation
- GuardDuty all 6 features (Foundational + EKS Audit + EKS Runtime + S3 + Lambda + RDS + EBS Malware)
- Inspector v2 (EC2 + ECR + Lambda + Lambda Code) + CIS scan configurations
- Macie sensitive data discovery + automated S3 classification
- Detective graph-based investigation
- IAM Access Analyzer (external + unused access)
- AWS Audit Manager (PCI/HIPAA/SOC 2 frameworks)
- AWS Security Lake — OCSF normalization + cross-tool federation
- Incident Response on AWS — IR-1 through IR-7 NIST framework

Generate complete, production-deployable code. No TODOs.

---

## Context and Inputs

```
PROJECT_NAME:                  [REQUIRED]
PRIMARY_REGION:                [REQUIRED]
ENTERPRISE_NAME:               [REQUIRED]

# --- ACCOUNTS ---
AUDIT_ACCOUNT_ID:              [REQUIRED — delegated admin destination]
LOG_ARCHIVE_ACCOUNT_ID:        [REQUIRED]
ALL_WORKLOAD_ACCOUNT_IDS:      [REQUIRED — comma-separated]
ALL_GOVERNED_REGIONS:          [REQUIRED — comma-separated]

# --- COMPLIANCE FRAMEWORK ---
COMPLIANCE_FRAMEWORKS:         [REQUIRED — comma-separated; soc2, hipaa, pci-dss-4, nist-800-53-r5, fedramp]

# --- WORKLOAD CONTEXT ---
EKS_CLUSTERS_PRESENT:          [true/false — drives EKS Audit + Runtime enablement]
LAMBDA_HEAVY:                  [true/false — drives Lambda Network Logs + Inspector Lambda]
RDS_PRESENT:                   [true/false — drives RDS Login Events]
SENSITIVE_S3_BUCKETS:          [comma-separated bucket ARNs — Macie continuous scan target]

# --- SIEM / RESPONSE ---
SIEM_TYPE:                     [REQUIRED — splunk | datadog | sumo | elastic | none]
SIEM_AWS_ACCOUNT_ID:           [REQUIRED if SIEM via Security Lake]
PAGERDUTY_INTEGRATION_KEY:     [REQUIRED for CRITICAL findings routing]
SLACK_WEBHOOK_URL:             [REQUIRED for HIGH findings routing]
INCIDENT_RESPONSE_TEAM_EMAIL:  [REQUIRED]

# --- OBSERVABILITY ---
SNS_ALARM_TOPIC_ARN:           [REQUIRED]
ENABLE_AUDIT_MANAGER:          [true if compliance evidence collection needed]

# --- COMPLIANCE ---
KMS_KEY_ARN:                   [REQUIRED — Security Lake + CloudTrail Lake encryption]
LOG_RETENTION_YEARS:           [7 default; 10 for some HIPAA cases]
```

---

## Partial Library (Claude MUST load)

| Partial | Why |
|---|---|
| `ENTERPRISE_SECURITY_HUB_GD_ORG` | Org-wide enable of all 7 services with Central Config + delegated admin |
| `ENTERPRISE_CENTRALIZED_LOGGING` | CloudTrail Lake + Security Lake (the data foundation for SOC) |
| `ENTERPRISE_ORG_SCPS_RCPS` | Required because delegated admin uses these |
| `EKS_SECURITY` | (if EKS) GuardDuty for EKS specifics + Pod Security Standards + Kyverno |
| `LAYER_SECURITY` | KMS multi-region key foundation |
| `LAYER_OBSERVABILITY` | EventBridge → SNS → notification baseline |
| `SECURITY_DATALAKE_CHECKLIST` | 30-control composite for data-lake compliance |
| `COMPLIANCE_HIPAA_PCIDSS` | Audit bucket + Backup Vault Lock + Config rules baseline |

---

## Architecture

```
   ┌─────────────────────────────────────────────────────────────────┐
   │  Audit Account (delegated admin for 6 services)                 │
   │                                                                  │
   │  Detective Layer:                                                │
   │   - GuardDuty (org-wide, 6 features) ──┐                        │
   │   - Inspector v2 (EC2+ECR+Lambda+Code) ─┤                        │
   │   - Macie (continuous discovery on tag) ┤                        │
   │   - Access Analyzer (external + unused) ┤                        │
   │                                         ├──► Security Hub        │
   │   Standards subscribed: AWS FSBP +     │       (Central Config)  │
   │     CIS v3 + PCI-DSS v4 + NIST 800-53  │                        │
   │     + (HITRUST if HIPAA)                │                        │
   │                                         ▼                        │
   │  Investigative Layer:                                            │
   │   - Detective (graph: identity ↔ activity ↔ findings)            │
   │   - CloudTrail Lake (SQL queries on org-wide CT events, 7y)      │
   │                                                                  │
   │  Normalized data lake:                                           │
   │   - Security Lake (OCSF Iceberg in S3)                           │
   │     Sources: GuardDuty, Security Hub, CloudTrail, VPC Flow,      │
   │              R53 Query, AppFabric (SaaS audit), custom logs       │
   │     Subscribers: SIEM (Splunk/Datadog), QuickSight, Athena       │
   │                                                                  │
   │  Response routing (EventBridge):                                 │
   │   CRITICAL → SNS → PagerDuty (24x7 page)                         │
   │   HIGH → SNS → Slack (#sec-ops)                                  │
   │   MEDIUM → SNS → Email digest (daily)                            │
   │   AUTO-REMEDIATE → Lambda (e.g., revoke leaked key, isolate pod) │
   │                                                                  │
   │  Evidence layer:                                                 │
   │   - Audit Manager (auto-collects evidence per framework)         │
   │   - Bucket: org-compliance-evidence-{audit-acct}                  │
   └─────────────────────────────────────────────────────────────────┘
```

---

## Day-by-day execution (3-5 day deployment, 1 dev)

### Day 1 — Delegated admin + Security Hub Central Config
- Run `SecurityOrgAdminStack` in Mgmt account (`ENTERPRISE_SECURITY_HUB_GD_ORG` §3.1)
  - Delegates Security Hub, GuardDuty, Inspector, Macie, Detective, Access Analyzer, Audit Manager → Audit account
- Run `SecurityAuditStack` in Audit account
  - Security Hub Central Configuration
  - Configuration Policy enabling 4 standards (FSBP + CIS v3 + PCI-DSS v4 + NIST 800-53 r5)
  - Associated to root OU (auto-enables every existing + future account)
  - Finding aggregator: ALL_REGIONS
- Validate: in any workload account, `aws securityhub describe-hub` returns active hub linked to Audit
- **Deliverable:** Security Hub UI in Audit account shows findings flowing in from all workload accounts within 30 min.

### Day 2 — GuardDuty all 6 features + Inspector + Macie
- GuardDuty: enable detector + 6 features org-wide auto-enable
- Inspector v2: enable for EC2 + ECR + Lambda + Lambda Code; configure weekly CIS scans
- Macie: enable + auto-enable for new accounts; create classification job for `SENSITIVE_S3_BUCKETS`
- Detective: auto-enabled by GuardDuty findings stream
- Access Analyzer: external + unused access analyzers
- **Deliverable:** Run sample threat: `curl http://169.254.169.254/latest/meta-data/iam/security-credentials/` from a test EC2 → GuardDuty `UnauthorizedAPICall:IAMUser/InstanceCredentialExfiltration` finding within 10 min.

### Day 3 — Security Lake + SIEM integration + CloudTrail Lake
- Security Lake setup in Audit account (Primary Region)
  - 4 AWS-native sources: GuardDuty findings, Security Hub findings, CloudTrail Mgmt, VPC Flow
  - Org-wide ingestion (all `ALL_WORKLOAD_ACCOUNT_IDS`)
- (If `SIEM_TYPE != none`) Add Subscriber for SIEM_AWS_ACCOUNT_ID
  - Access type: S3 (for raw OCSF parquet) + Lake Formation (for SQL grants)
- CloudTrail Lake event data store (org-wide, 7y retention, termination protection)
- Build 10 canonical SQL queries (saved):
  - "Failed sign-ins in last 24h"
  - "Root user activity in last 7d"
  - "S3 buckets made public in last 24h"
  - "IAM key creation by automation accounts in last 7d"
  - "GuardDuty CRITICAL findings ungrouped"
  - "VPC Flow Logs to known-bad CIDR"
  - "Macie sensitive data findings (PII/PHI) by bucket"
  - "Pod Identity AssumeRole anomaly (rare-source-IP)"
  - "Cross-account API calls to Audit account"
  - "AssumeRole from external (out-of-org) principals"
- **Deliverable:** SIEM team confirms ingestion working; sample query returns rows in CloudTrail Lake console.

### Day 4 — Finding routing + auto-remediation + Audit Manager
- EventBridge rules for 3 severity tiers:
  - CRITICAL/HIGH `Workflow.Status: NEW` → SNS → PagerDuty (Lambda formats payload + posts to PagerDuty Events API)
  - MEDIUM → daily digest Lambda → email
  - LOW → log only
- Auto-remediation Lambdas (5 quick wins):
  - **GuardDuty UnauthorizedAccess:IAMUser/AnomalousBehavior** → quarantine IAM user (deny-all policy attached)
  - **Security Hub `S3.1` (S3 bucket public)** → set bucket public access block ON
  - **Inspector CRITICAL CVE in ECR** → CodeBuild redeploy with patched base image
  - **Macie `SensitiveData:S3Object/Personal`** → set bucket policy deny + tag bucket `Macie-Action: review`
  - **GuardDuty `Discovery:S3/MaliciousIPCaller`** → set CloudFront geo-block / WAF block
- (If `ENABLE_AUDIT_MANAGER`) Set up Audit Manager assessment per framework in `COMPLIANCE_FRAMEWORKS`
  - Auto-collects evidence from Security Hub, Config, CloudTrail
  - Generates SOC 2 Type II / PCI-DSS / HIPAA evidence pack
- **Deliverable:** Inject a test finding (make S3 bucket public in dev account) → CRITICAL alert hits PagerDuty within 15 min + auto-remediation Lambda re-locks the bucket within 5 min.

### Day 5 — IR runbook + dashboards + handoff
- Build SOC dashboards in QuickSight (or AMG):
  - **Findings by severity** (CRITICAL count over 30d trend)
  - **Findings by account** (top-10 noisy accounts)
  - **Findings by service** (GuardDuty / Inspector / Macie / Security Hub controls)
  - **Mean time to suppress** (workflow lifecycle)
  - **Open CRITICAL findings > 24h** (oncall worklist)
  - **PII bucket count + scan coverage** (Macie)
  - **Inspector CVE backlog by severity** (CRITICAL/HIGH)
- Author IR runbook (`incident-response.md`) covering NIST IR-1 through IR-7:
  - **Detection** triage matrix per finding type
  - **Containment**: AWS API runbook per resource type (EC2, IAM, S3, EKS pod)
  - **Eradication**: forensic snapshot + clean redeploy
  - **Recovery**: workload restoration + verification
  - **Lessons learned** template
- Run **tabletop exercise** with security team (45 min):
  - Scenario 1: leaked IAM access key found in public Github
  - Scenario 2: container compromise via vulnerable image
  - Scenario 3: PII exfiltration alert from Macie
- **Deliverable:** SOC team completes 3 tabletop scenarios using runbook + dashboards in < 30 min each.

---

## Validation criteria

- [ ] **Security Hub Central Config policy** associated to root OU; ALL workload accounts show as enabled
- [ ] **4 standards subscribed**: AWS FSBP + CIS v3 + PCI-DSS v4 + NIST 800-53 r5 (verified per account)
- [ ] **GuardDuty 6 features auto-enabled org-wide** (`describe-organization-configuration` confirms)
- [ ] **Sample threat triggers GuardDuty finding** within 10 min
- [ ] **Inspector v2 scanning all in-scope EC2 + ECR images** (zero "NOT_SCANNED" status after 24h)
- [ ] **Macie classification jobs running** on `SENSITIVE_S3_BUCKETS`; first results within 24h
- [ ] **Detective enabled and ingesting GuardDuty + CloudTrail + VPC Flow**
- [ ] **Security Lake CREATED**, 4 sources ingesting parquet to S3
- [ ] **SIEM subscriber receiving data** (verified by SIEM team query)
- [ ] **CloudTrail Lake event store ACTIVE**, sample query returns rows
- [ ] **EventBridge rules** routing CRITICAL → PagerDuty, HIGH → Slack
- [ ] **Auto-remediation Lambdas** test-fired successfully (5 scenarios)
- [ ] **Audit Manager assessment running** per framework (if enabled)
- [ ] **SOC dashboards live** with non-zero data
- [ ] **IR runbook validated** via 3 tabletop scenarios

---

## Common gotchas (claude must address proactively)

- **Security Hub findings flood at first enablement** — expect 1000+ findings on day 1. Plan a triage window.
- **GuardDuty Runtime Monitoring on EKS requires the agent DaemonSet** to be installed via add-on; without it, Runtime findings won't generate.
- **Macie continuous discovery is $$$ at scale.** Use one-shot classification jobs scoped to bucket tags. Schedule weekly, not real-time.
- **Detective consumes ~30 GB/day for a medium org** — $60/day at $2/GB. Scope to investigation regions.
- **Inspector v2 ECR enrollment requires ECR repos opt-in** unless org-wide auto-enable. Check `inspector2:GetEcrRegistryConfiguration`.
- **Auto-remediation Lambdas can amplify mistakes** — start with Lambdas in dry-run mode (log action, don't perform), promote to enforce after week 1.
- **Standards subscription "PCI-DSS v4.0.1"** has different control IDs than v3.2.1. If migrating, expect 30%+ control ID changes.
- **Security Hub Central Config replaces the old "auto-enable" flag.** Don't run both — uninstall old config first.
- **SOC dashboards in QuickSight require Lake Formation grants** on Security Lake databases. ABAC policies via LF tags.
- **Audit Manager evidence collection has lag (24-48h)** — assessments run nightly, not real-time.
- **Cross-account EventBridge for security findings** requires resource policy on target buses + `aws events put-permission` on source. Common miss.

---

## Output artifacts

1. **CDK stacks** — `SecurityOrgAdminStack` (Mgmt) + `SecurityAuditStack` (Audit) + `SecurityLakeStack` + `IrAutoRemediationStack`
2. **Security Hub Configuration Policy** as JSON
3. **GuardDuty org config** as CDK
4. **5 Auto-remediation Lambdas** with Pytest coverage
5. **10 saved SQL queries** for CloudTrail Lake (`.sql` files)
6. **EventBridge routing rules** YAML
7. **PagerDuty + Slack notification Lambdas** (formats finding payload to channel-friendly message)
8. **SOC dashboards** as QuickSight analyses (export JSON)
9. **Audit Manager assessment templates** per framework
10. **`incident-response.md` runbook** — NIST IR-1 to IR-7
11. **Tabletop exercise pack** (3 scenarios + facilitator notes + scoring rubric)
12. **Pytest validation suite** — covers all criteria

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 2.0 | 2026-04-26 | Initial composite template. Composes ENTERPRISE_SECURITY_HUB_GD_ORG + CENTRALIZED_LOGGING + 30+ controls. 3-5 day SOC stand-up. Wave 11. |
