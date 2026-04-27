<!-- Template Version: 2.0 | F369 Wave 11 (composite) | Composes: ENTERPRISE_CONTROL_TOWER + ENTERPRISE_IDENTITY_CENTER + ENTERPRISE_ORG_SCPS_RCPS + ENTERPRISE_NETWORK_HUB_TGW + ENTERPRISE_CENTRALIZED_LOGGING -->

# Template 09 — Landing Zone Baseline (Control Tower · Identity Center · SCPs/RCPs · TGW Hub · Log Archive · ready for workloads)

## Purpose

Stand up a **production-grade AWS Landing Zone in 5-7 days** that's ready to host workloads safely. Replaces the "we just opened an AWS account, now what?" trap with a complete multi-account governance + identity + network + logging foundation.

This is the **canonical "landing zone" engagement** — every enterprise client asks for this when starting AWS adoption. Generates a fully-deployed environment that passes a Well-Architected Review out of the box.

Generates production-deployable: Control Tower setup checklist, CDK stacks for OUs/SCPs/RCPs/Network Hub, IaC for Identity Center permission sets + Azure AD/Okta federation, runbooks.

---

## Role Definition

You are an expert AWS multi-account architect with deep expertise in:
- AWS Control Tower landing zone v3 + Account Factory for Terraform (AFT)
- AWS Organizations + OU strategy + delegated administration
- IAM Identity Center + Azure AD/Okta federation + SCIM provisioning + ABAC
- SCPs (Service Control Policies) + RCPs (Resource Control Policies, Nov 2024) + Declarative Policies
- Transit Gateway hub-and-spoke + Network Firewall + centralized egress + RAM share
- Log Archive account hardening (Object Lock COMPLIANCE + cross-region replication)
- CloudTrail org trail + CloudTrail Lake + AWS Security Lake (OCSF)
- AWS Well-Architected Framework alignment

Generate complete, production-deployable code. No TODOs.

---

## Context and Inputs

```
PROJECT_NAME:                [REQUIRED]
PRIMARY_REGION:              [REQUIRED — e.g. us-east-1]
ADDITIONAL_REGIONS:          [comma-separated; will be governed by Control Tower]
ENTERPRISE_NAME:             [REQUIRED]

# --- ACCOUNT EMAILS (must be NEW emails, unique across AWS) ---
MGMT_ACCT_EMAIL:             [REQUIRED — already created or to-create]
LOG_ARCHIVE_EMAIL:           [REQUIRED]
AUDIT_EMAIL:                 [REQUIRED]
INFRASTRUCTURE_EMAIL:        [REQUIRED]
PROD_DATA_PLATFORM_EMAIL:    [REQUIRED — first workload account]
NONPROD_EMAIL:               [REQUIRED]
SANDBOX_EMAIL:               [REQUIRED]

# --- IDP ---
IDP_TYPE:                    [REQUIRED — built_in | azure_ad | okta | google_workspace]
AZURE_AD_TENANT_ID:          [REQUIRED if azure_ad]
AZURE_AD_SCIM_TOKEN:         [REQUIRED if azure_ad — generated in Identity Center]
USER_GROUPS_TO_PROVISION:    [comma-separated AD/Okta group names — e.g. AWS-Admins,AWS-Developers,AWS-ReadOnly]

# --- NETWORK ---
TGW_HUB_REGION:              [equals PRIMARY_REGION typically]
ALLOWED_AWS_REGIONS:         [comma-separated for SCP — e.g. us-east-1,us-west-2,eu-west-1]
ON_PREM_CIDRS:               [optional, for DX/VPN; comma-separated]
ON_PREM_DNS_DOMAINS:         [optional, for R53 Resolver outbound]
SPOKE_VPC_CIDRS:             [REQUIRED — list of CIDR per planned spoke; e.g. 10.0.0.0/16,10.1.0.0/16]

# --- COMPLIANCE ---
COMPLIANCE_FRAMEWORK:        [REQUIRED — soc2 | hipaa | pci | nist-800-53 | combined]
LOG_RETENTION_YEARS:         [7 default]
DATA_CLASSIFICATION:         [confidential default; restricted for HIPAA]

# --- BUDGET / FINOPS ---
ANNUAL_AWS_BUDGET_USD:       [REQUIRED — for Anomaly Detection threshold]
COST_CENTER_TAGS:            [comma-separated mandatory tag keys]
```

---

## Partial Library (Claude MUST load)

| Partial | Why |
|---|---|
| `ENTERPRISE_CONTROL_TOWER` | Landing zone v3 + canonical OU shape + AFT + 30+ guardrails |
| `ENTERPRISE_IDENTITY_CENTER` | Permission Sets + Azure AD federation + SCIM + ABAC |
| `ENTERPRISE_ORG_SCPS_RCPS` | 5 canonical SCPs + 2 RCPs + Declarative Policies + delegated admin |
| `ENTERPRISE_NETWORK_HUB_TGW` | TGW hub + Egress VPC + Inspection VPC + RAM share + R53 Resolver |
| `ENTERPRISE_CENTRALIZED_LOGGING` | Org trail + Log Archive Object Lock + CloudTrail Lake + Security Lake |
| `LAYER_NETWORKING` | VPC + endpoints (re-verify alignment with TGW) |
| `LAYER_SECURITY` | KMS multi-region keys + IAM least-privilege patterns |
| `COMPLIANCE_HIPAA_PCIDSS` | Audit bucket + Backup Vault Lock + Config rules |

---

## Architecture

```
   ┌──────────────────────────────────────────────────────────────────┐
   │ Management Account                                               │
   │   - AWS Organizations + Control Tower landing zone v3            │
   │   - AFT (or CfCT) — IaC for new accounts                          │
   │   - Delegated admin → Audit, Log Archive, Infrastructure          │
   └──────────────────────────────────────────────────────────────────┘
                               │
       ┌─────────┬─────────┬───┴────┬──────────┬─────────────┐
       ▼         ▼         ▼        ▼          ▼             ▼
   Security  Workloads  Sandbox  Infrastructure  Suspended    (future)
       │         │
       │         ├── Production OU (prod-data, prod-app, ...)
       │         ├── Non-Production OU (stage-*)
       │         └── Development OU (dev-*)
       │
       ├── Log Archive Account
       │     - S3 bucket (Object Lock COMPLIANCE, 7y retention, CRR to us-west-2)
       │     - Receives: Org Trail, VPC Flow Logs, R53 Query Logs, ELB logs
       │
       └── Audit Account
             - Security Hub (delegated admin, Central Config, AWS FSBP+CIS+PCI+NIST)
             - GuardDuty (delegated admin, all 6 features org-wide auto-enable)
             - Inspector v2 (delegated admin)
             - Macie (delegated admin)
             - Detective (delegated admin)
             - Access Analyzer (delegated admin)
             - CloudTrail Lake event data store (org-wide, 7y)
             - Security Lake (OCSF Iceberg, 4 sources)
             - Audit Manager (PCI/HIPAA frameworks)

   Infrastructure Account
     - Transit Gateway (auto-accept, RAM-shared to org)
     - Egress VPC (NAT GW × 3 AZ, IGW)
     - Inspection VPC (Network Firewall, intercepts egress)
     - Centralized R53 Resolver inbound + outbound endpoints
     - Centralized PrivateLink endpoints (KMS, Secrets, SSM, Logs, STS, ECR.*)

   Each Workload Account (prod-data, prod-app, stage-*, dev-*)
     - Spoke VPC (no IGW, no NAT) — egress via TGW → Egress VPC → IGW
     - PSS-restricted namespaces (if EKS)
     - CDK app deploys workload using F369 partials

   IAM Identity Center (in Mgmt account)
     - Identity source: Azure AD (SAML+SCIM)
     - 4 Permission Sets: AdministratorAccess, DeveloperAccess,
       ReadOnlyAccess, SecurityAuditor
     - Group assignments: Admins → all accts, Devs → non-prod, Readonly → all,
       SecOps → Audit account only
     - ABAC tags propagated from Azure AD: team, cost_center
```

---

## Day-by-day execution (5-7 day deployment, 1-2 platform engineers)

### Day 1 — Org + Control Tower setup
- Create Management account (or use existing)
- Console: AWS Organizations setup
- Console: Identity Center enable (no users yet)
- Console: Control Tower → Set up landing zone (1h wait)
- Validate Log Archive + Audit accounts created
- **Deliverable:** Landing zone "Available" in Control Tower console; mandatory guardrails active.

### Day 2 — Identity Center federation + Permission Sets
- (If `IDP_TYPE=azure_ad`) Configure Azure AD enterprise app + SAML metadata exchange + SCIM provisioning
- Provision `USER_GROUPS_TO_PROVISION` from Azure AD → Identity Center (SCIM, 30-min wait)
- Create 4 Permission Sets via CDK (`ENTERPRISE_IDENTITY_CENTER` §3.1)
- Group assignments per OU (Admins → all, Devs → non-prod only)
- Verify: developer logs into Identity Center portal → sees only Non-Prod accounts
- **Deliverable:** End of Day 2: 3 testers (admin, developer, readonly) can sign in via SSO and access correct accounts.

### Day 3 — SCPs + RCPs + Declarative Policies
- Create custom OUs: Production, Non-Production, Development under Workloads
- Apply 5 canonical SCPs at Workloads OU (`ENTERPRISE_ORG_SCPS_RCPS` §3)
- Apply 2 RCPs (DenyExternalS3, RestrictAssumeRole) at Workloads OU
- Apply 2 Declarative Policies (EBS encrypt-by-default, IMDSv2-required) at root
- Sandbox SCP: limit instance types + budget
- Test: sandbox account tries to launch m6i.4xlarge → denied
- Test: dev account tries to disable CloudTrail → denied
- Test: prod account tries to use eu-west-3 → denied
- **Deliverable:** SCP test matrix passes (8 scenarios verified).

### Day 4 — Network Hub (TGW + Egress + Inspection)
- Deploy Network Hub stack in Infrastructure account (`ENTERPRISE_NETWORK_HUB_TGW`)
- Create Egress VPC + 3 NAT Gateways
- Create Inspection VPC + Network Firewall (if `COMPLIANCE_FRAMEWORK in [pci, hipaa]`)
- Create Transit Gateway + 3 route tables (spoke / egress / inspection)
- Share TGW via RAM with org
- Centralized Route 53 Resolver (inbound + outbound) + share rules via RAM
- Centralized PrivateLink endpoints (15+ services)
- Deploy first spoke VPC in `prod-data-platform` account; verify it egresses via TGW
- **Deliverable:** Spoke VPC pod can `curl https://www.example.com` (egresses through Network Firewall + NAT); Reachability Analyzer confirms TGW path.

### Day 5 — Centralized logging (Log Archive + CloudTrail Lake + Security Lake)
- Deploy Log Archive stack in Log Archive account
  - S3 bucket Object Lock COMPLIANCE (7y) + cross-region replication to us-west-2
  - KMS multi-region key
  - Bucket policy denies delete + requires TLS + restricts to org
- Deploy Org Trail in Management account → Log Archive bucket
- Configure VPC Flow Logs + R53 Query Logs from spoke VPCs → Log Archive
- Deploy CloudTrail Lake event data store in Audit account (org-wide, 7y, termination protection on)
- Deploy Security Lake in Audit account with 4 sources: CloudTrail Mgmt, VPC Flow, R53 Query, Security Hub findings
- Add 1 SIEM subscriber (Splunk/Datadog) if customer has one
- **Deliverable:** Sample CloudTrail Lake SQL query returns `AssumeRole` events from last 24h across all accounts.

### Day 6 — Org-wide security services + finding routing
- Delegate 6 security services from Mgmt → Audit (`ENTERPRISE_SECURITY_HUB_GD_ORG` §3.1)
- Enable Security Hub Central Configuration + 4 standards
- Enable GuardDuty all 6 features org-wide
- Enable Inspector v2 (EC2 + ECR + Lambda + Lambda Code) org-wide
- Enable Macie + Detective + Access Analyzer
- EventBridge rule: Security Hub CRITICAL/HIGH NEW findings → SNS → PagerDuty/Slack
- **Deliverable:** GuardDuty produces sample finding in dev account; alert reaches Slack within 15 min.

### Day 7 — Validation + handoff
- Run Well-Architected Review tool against landing zone — score baseline
- Run Pytest validation suite (300+ assertions across all 5 partials)
- Generate compliance evidence pack:
  - All Permission Sets export
  - All SCPs/RCPs export
  - Security Hub config policy export
  - GuardDuty org configuration
  - Log Archive bucket configuration
  - CloudTrail Lake event store ARN + sample queries
- Author handoff runbook:
  - "Adding a new account" (AFT workflow)
  - "Adding a new region" (re-baseline)
  - "Onboarding a new team" (Identity Center group + Permission Set assignment)
  - "Investigating a security finding" (Detective + Security Lake)
- **Deliverable:** Customer can self-serve account creation + new team onboarding within 30 min.

---

## Validation criteria

- [ ] **Control Tower landing zone status: AVAILABLE**
- [ ] **All required OUs exist**: Security, Workloads (Production, Non-Production, Development), Sandbox, Infrastructure, Suspended
- [ ] **30+ mandatory + strongly recommended guardrails enabled** at Workloads OU
- [ ] **Identity Center federated with `IDP_TYPE`**, SCIM auto-provisioning active (test: add to AD group → user in IDC within 30 min)
- [ ] **4 Permission Sets created**, assignments correct per group
- [ ] **AdministratorAccess `session_duration: PT1H`** (verified via `describe-permission-set`)
- [ ] **5 canonical SCPs + 2 RCPs + 2 Declarative Policies** attached to expected OUs
- [ ] **TGW deployed + RAM-shared to org**; spoke VPC in prod-data-platform attaches successfully
- [ ] **Centralized egress works**: spoke pod can `curl https://www.example.com` via Egress VPC NAT
- [ ] **Network Firewall blocks denylisted domains** (e.g., `tor.com`)
- [ ] **Log Archive bucket Object Lock COMPLIANCE active**, 7y retention, CRR ENABLED
- [ ] **Org Trail status: LOGGING**, multi-region, log file validation enabled
- [ ] **CloudTrail Lake event store ACTIVE**, sample query returns rows from all accounts
- [ ] **Security Lake data lake CREATED**, 4 sources ingesting
- [ ] **Security Hub Central Config policy associated to root OU**
- [ ] **GuardDuty org-wide auto-enable: ALL** for all 6 features
- [ ] **Inspector v2 enrolling new accounts automatically**
- [ ] **Macie + Detective + Access Analyzer delegated to Audit**
- [ ] **EventBridge rule routes CRITICAL/HIGH Security Hub findings → SNS** (test by triggering a known finding)
- [ ] **Well-Architected Review baseline ≥ 70%** on Security pillar

---

## Common gotchas (claude must address proactively)

- **Account email uniqueness across all of AWS** — use `aws-prod-data@yourcompany.com` not `prod@yourcompany.com`. Bouncing emails delay landing zone setup.
- **Identity Center can only run in ONE region per Org.** Migrating later is painful — pick the long-term primary region upfront.
- **Federation SCIM lag is 30-60 min.** New hires can't sign in immediately. Document in onboarding runbook.
- **SCPs do NOT grant permissions** — they only filter. IAM still required.
- **`FullAWSAccess` SCP at root must remain attached** — without it, member accounts have NO permissions.
- **TGW data transfer + per-attachment hours add up** — for tiny VPCs (< 10 GB/mo egress), a per-VPC NAT may be cheaper. Estimate at deployment.
- **Network Firewall costs ~$1.50/hr per AZ** — only enable for `COMPLIANCE_FRAMEWORK in [pci, hipaa, fedramp]`.
- **Object Lock COMPLIANCE is permanent** — root user CANNOT shorten retention. Pick GOVERNANCE if you need an emergency override; COMPLIANCE for regulated.
- **CloudTrail S3 data events** generate millions of events/day — only enable on sensitive prefixes via `S3EventSelector`.
- **Security Hub CRITICAL/HIGH alerts can flood at first** — schedule a triage day post-launch to mark known-benign as `Workflow.Status: SUPPRESSED`.
- **Control Tower drift detection runs hourly** — manual changes to CT-managed resources trigger drift alerts. Use AFT/CfCT for legitimate customizations.

---

## Output artifacts

1. **CDK Python apps** (multiple, per account):
   - `ScpStack` (Mgmt account)
   - `IdentityCenterStack` (Mgmt account)
   - `LogArchiveStack` (Log Archive account)
   - `OrgTrailStack` (Mgmt account)
   - `SecurityOrgAdminStack` (Mgmt) + `SecurityAuditStack` (Audit)
   - `NetworkHubStack` (Infrastructure account)
   - `SpokeVpcStack` (per workload account, parameterized)
2. **Control Tower setup checklist** (Console steps + screenshots)
3. **AFT (Terraform) bootstrap repo** for IaC new account creation
4. **Azure AD federation runbook** (SAML metadata + SCIM token + group mapping)
5. **5 SCPs + 2 RCPs + 2 Declarative Policies** as JSON
6. **Pytest suite** — 300+ assertions covering each validation criterion (split per partial)
7. **Compliance evidence pack generator** (`generate-evidence.sh`) → ZIP for assessor
8. **Operational runbooks** (Markdown):
   - Adding a new account
   - Onboarding a team
   - Investigating a finding
   - Emergency break-glass (root user MFA recovery)
   - Cross-region failover for Log Archive
9. **Well-Architected Review export** — initial baseline + improvement roadmap
10. **Cost forecasting spreadsheet** — TGW + GuardDuty + Inspector + Macie + Security Lake + Detective per-account/per-month projection

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 2.0 | 2026-04-26 | Initial composite template. Composes 5 ENTERPRISE partials. 5-7 day full-stack landing zone. Wave 11. |
