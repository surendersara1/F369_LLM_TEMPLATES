<!-- Template Version: 2.0 | F369 Wave 15 (composite) | Composes: BEDROCK_Q_BUSINESS + BEDROCK_KNOWLEDGE_BASES + ENTERPRISE_IDENTITY_CENTER + LAYER_SECURITY -->

# Template 34 — Q Business Enterprise Assistant (40+ connectors · plugins · custom Q apps · IDC SSO · 3-5 day rollout)

## Purpose

Stand up **Amazon Q Business as the company's "ChatGPT for everything internal"** in 3-5 days. Output: Q Business application connected to 5-10 enterprise data sources, IDC-federated SSO, AppRoles per team, plugins for ticketing/CRM, custom Q Apps, web UI rolled out to all employees.

This is the **canonical "AI assistant for the enterprise" engagement** — every CIO is asking for this. Replaces a year-long "build our own ChatGPT" project with a 1-week deploy.

---

## Role Definition

You are an expert AWS GenAI architect with deep expertise in:
- Amazon Q Business application setup (Lite vs Pro tier decision)
- Q Business 40+ data source connectors (S3, SharePoint, Confluence, Salesforce, ServiceNow, Slack, Teams, Box, Drive, Jira, GitHub, Workday, Zendesk)
- Q plugins (built-in: Jira, Salesforce, ServiceNow, PagerDuty; custom: OpenAPI + Lambda)
- Q Apps SDK for custom no-code app builder (preview 2024)
- AppRoles for fine-grained group-based access control
- IAM Identity Center federation (Azure AD / Okta / Google Workspace)
- Bedrock Knowledge Bases as alternative for fine-grained RAG control
- Compliance frameworks (PII handling, audit, retention, deletion)

Generate complete, production-deployable code. No TODOs.

---

## Context and Inputs

```
PROJECT_NAME:                [REQUIRED]
AWS_REGION:                  [REQUIRED]
ENTERPRISE_NAME:             [REQUIRED]

# --- IDENTITY ---
IDC_INSTANCE_ARN:            [REQUIRED — Identity Center already set up]
SSO_GROUPS:                  [REQUIRED — comma-separated; e.g. AllEmployees,Engineers,Sales,HR,Finance,Admins]

# --- SUBSCRIPTION ---
TIER:                        [REQUIRED — lite | pro; pro for plugins + Q Apps]
EXPECTED_USERS:              [REQUIRED — for cost projection]

# --- DATA SOURCES (pick subset) ---
ENABLE_S3:                   [true default]
S3_BUCKETS:                  [comma-separated bucket ARNs]
ENABLE_SHAREPOINT:           [true if SharePoint Online]
SHAREPOINT_TENANT_ID:        [REQUIRED if enabled]
SHAREPOINT_SITE_URLS:        [comma-separated]
ENABLE_CONFLUENCE:           [true/false]
CONFLUENCE_URL:              [REQUIRED if enabled]
ENABLE_SALESFORCE:           [true if SF in scope]
ENABLE_SERVICENOW:           [true/false]
ENABLE_SLACK:                [true/false]
ENABLE_GOOGLE_DRIVE:         [true/false]
ENABLE_JIRA:                 [true/false]

# --- PLUGINS ---
ENABLE_JIRA_PLUGIN:          [true if Pro + Jira]
ENABLE_SALESFORCE_PLUGIN:    [true if Pro + SF]
ENABLE_SERVICENOW_PLUGIN:    [true if Pro + ServiceNow]
ENABLE_CUSTOM_PLUGIN:        [false default — set true if internal API integration]

# --- INDEX ---
INDEX_TYPE:                  [STARTER for POC; ENTERPRISE for prod]
INDEX_UNITS:                 [1 default — 100K docs/unit]

# --- COMPLIANCE ---
KMS_KEY_ARN:                 [REQUIRED]
DATA_CLASSIFICATION:         [confidential default; restricted for HIPAA]
RETENTION_DAYS:              [365 default; 30 for non-regulated]
ENABLE_PII_GUARDRAILS:       [true required for prod]

# --- WEB UI ---
ENABLE_CUSTOM_DOMAIN:        [false default — true if branding required]
CUSTOM_DOMAIN:               [chat.example.com if enabled]
EMBED_ALLOWED_ORIGINS:       [comma-separated for SDK embedding]

# --- CHANNELS (extend Q to where users work) ---
ENABLE_SLACK_CHANNEL:        [true/false]
ENABLE_TEAMS_CHANNEL:        [true/false]
ENABLE_BROWSER_EXTENSION:    [true default]

# --- OBSERVABILITY ---
SNS_ALARM_TOPIC_ARN:         [REQUIRED]
```

---

## Partial Library (Claude MUST load)

| Partial | Why |
|---|---|
| `BEDROCK_Q_BUSINESS` | Q Business application + connectors + plugins + AppRoles |
| `BEDROCK_KNOWLEDGE_BASES` | (alternative) for cases where Q Business UX doesn't fit |
| `ENTERPRISE_IDENTITY_CENTER` | IDC + SCIM provisioning + Azure AD/Okta federation |
| `LAYER_SECURITY` | KMS multi-region key + Secrets Manager for connector creds |
| `LAYER_OBSERVABILITY` | CW dashboards + alarms |

---

## Architecture

```
   Employees (Azure AD / Okta / Google Workspace)
        │
        │ SSO via IAM Identity Center
        ▼
   ┌──────────────────────────────────────────────────────────────┐
   │ Amazon Q Business Application                                 │
   │   Subscription: Pro $20/user/mo                                │
   │   IDC Instance: arn:aws:sso:::instance/ssoins-XXX              │
   │   KMS-encrypted                                                 │
   │                                                                  │
   │   Index: ENTERPRISE (1 unit; 100K docs)                          │
   │     Custom metadata: department, confidentiality, expiry         │
   │                                                                  │
   │   Data Sources (sync schedule staggered):                        │
   │     ├─ S3 (intranet wiki, policies)        ─ daily 02:00          │
   │     ├─ SharePoint Online                    ─ daily 03:00          │
   │     ├─ Confluence Cloud                     ─ daily 04:00          │
   │     ├─ Salesforce (cases, accounts)         ─ daily 05:00          │
   │     ├─ ServiceNow (KB articles)             ─ daily 06:00          │
   │     ├─ Slack (selected channels)            ─ hourly                │
   │     └─ Google Drive (shared docs)           ─ daily 07:00          │
   │   ALL with `isCrawlAcl: true` (inherit source ACLs to user)       │
   │                                                                  │
   │   Plugins:                                                       │
   │     ├─ Jira plugin (built-in OAuth2)                              │
   │     ├─ Salesforce plugin (built-in)                                │
   │     ├─ ServiceNow plugin (built-in)                                │
   │     └─ Custom: Expense Tracker (OpenAPI + Lambda)                  │
   │                                                                  │
   │   Custom Q Apps:                                                  │
   │     ├─ "Onboarding Helper" (new hire)                              │
   │     ├─ "Policy Lookup" (HR-restricted)                              │
   │     └─ "PR Reviewer" (Engineering)                                  │
   │                                                                  │
   │   AppRoles (group → permissions matrix):                          │
   │     ├─ AllEmployees: S3 + SharePoint (read general policy docs)    │
   │     ├─ Engineers: + GitHub + Confluence + Slack (eng channels)    │
   │     ├─ Sales: + Salesforce + ServiceNow (CRM context)              │
   │     ├─ HR: + Workday + private HR S3 (sensitive)                  │
   │     ├─ Finance: + ERP + Salesforce (revenue data)                  │
   │     └─ Admins: full access + Q Apps create + plugin install         │
   │                                                                  │
   │   Web Experience:                                                 │
   │     https://chat.example.com (custom domain via CloudFront)         │
   │     Embedded in: portal, Slack, Teams, browser extension            │
   └──────────────────────────────────────────────────────────────┘
```

---

## Day-by-day execution (3-5 day rollout, 1 architect)

### Day 1 — Foundation
- IDC pre-flight: confirm SCIM provisioning working; `SSO_GROUPS` synced
- Q Business application created via CDK + KMS encryption
- Index ENTERPRISE (1 unit) + custom metadata fields (department, confidentiality)
- Web Experience auto-provisioned + welcome message + sample prompts
- AppRoles created for each `SSO_GROUPS`; default role for all employees
- **Deliverable:** End of Day 1: 5 test users sign in via SSO, see empty Q UI.

### Day 2 — First 3 data sources (S3, SharePoint, Confluence)
- IAM role for Q Business data sources
- S3 connector with metadata sidecar files (department, confidentiality)
- SharePoint Online connector (Azure AD app registration; tenant ID + Microsoft Graph perms)
- Confluence Cloud connector (API token in Secrets Manager)
- Initial sync (manual trigger; ~1-2 hours for ~10K docs)
- Validate: search returns docs from each source; ACL filtering correct
- **Deliverable:** Test users can ask "what's our PTO policy?" → cited answer from S3 + SharePoint.

### Day 3 — Remaining data sources + plugins
- Salesforce connector (connected app + OAuth2)
- ServiceNow connector (basic auth in Secrets Manager)
- Slack connector (workspace install + bot token)
- Google Drive connector (OAuth2)
- (If `TIER=pro`) install plugins: Jira + Salesforce + ServiceNow built-ins
- (If `ENABLE_CUSTOM_PLUGIN`) custom plugin via OpenAPI spec + Lambda backend
- **Deliverable:** Users can ask "show me my open Salesforce cases" → plugin call → answer with case list.

### Day 4 — Custom Q Apps + AppRoles fine-tuning
- Author 3-5 Custom Q Apps:
  - "Onboarding Helper" — accepts team + role + start date; generates checklist
  - "Policy Lookup" — restricted to HR group; queries HR S3 + Workday
  - "PR Reviewer" — Engineering only; pulls from GitHub + Confluence
- AppRole tuning: per-group permissions matrix
- (If `ENABLE_PII_GUARDRAILS`) configure PII redaction in responses
- (If `ENABLE_SLACK_CHANNEL`) install Q in Slack
- (If `ENABLE_TEAMS_CHANNEL`) install Q in Teams
- **Deliverable:** Sales user can use Q in Slack to ask "what's the renewal date for $ACCOUNT?"

### Day 5 — Custom domain + change management + observability + handoff
- (If `ENABLE_CUSTOM_DOMAIN`) CloudFront + ACM cert + DNS for `chat.example.com`
- Change management:
  - Internal launch announcement (Slack/email)
  - Training video (5 min) + tip-of-the-day in welcome message
  - Help docs in shared FAQ
- CloudWatch dashboard: queries/day, top users, popular topics, plugin usage
- Alarms: data source sync failures, low engagement, error rate
- **Deliverable:** Q Business live for all `EXPECTED_USERS`; usage dashboard active.

---

## Validation criteria

- [ ] **Q Business application ACTIVE** + KMS-encrypted
- [ ] **Index ENTERPRISE active** with custom metadata
- [ ] **All `SSO_GROUPS` provisioned** via SCIM (verified via IDC console)
- [ ] **Each data source LATEST sync = SUCCEEDED** (no failed sync events)
- [ ] **Document-level ACL inheritance** verified — user A cannot see user B's restricted SharePoint pages
- [ ] **All plugins enabled + tested** with sample queries
- [ ] **Custom Q Apps published** + visible to authorized groups only
- [ ] **AppRoles enforce permissions** — Engineering user cannot use Salesforce plugin
- [ ] **Citations rendered** — every Q response includes source links
- [ ] **PII guardrails active** — emails/SSNs redacted in test queries
- [ ] **Slack/Teams channels working** (if enabled) — `/q What's our PTO?` returns answer in channel
- [ ] **Custom domain reachable** at `https://chat.example.com` (if enabled)
- [ ] **CloudWatch dashboards live** with > 10 days of usage data after rollout
- [ ] **MAU tracking** confirms billing per user per month (Pro vs Lite)

---

## Common gotchas (claude must address proactively)

- **MAU (Monthly Active User) billing surprise** — Pro at $20/MAU × 1000 employees = $20K/mo. Run cost projection before commit.
- **SharePoint connector fails silently** if Azure AD app permissions wrong. Required: Sites.Read.All + User.Read.All; admin consent required.
- **Connector sync schedule cron is UTC** — stagger starts to avoid throttling source systems (esp. SharePoint Graph API).
- **`isCrawlAcl: true` is mandatory** for tenant isolation. Without it, all users see all indexed content.
- **Custom domain requires CloudFront + ACM** — not a simple toggle. Allow 1 day to provision.
- **Q Apps SDK is preview** — APIs may change. Document workarounds.
- **Custom OpenAPI plugins must use OAuth2** — Q Business handles token exchange. Plain bearer tokens won't work.
- **PII guardrails redact in OUTPUT only** — input PII still indexed. Use Macie/Comprehend pre-ingestion if input PII is concern.
- **Rate limits per connector** — SharePoint Graph API: 600 requests / 10 min / app. For large tenants, use multiple Azure AD apps in pool.
- **First impressions matter** — terrible first response = abandoned tool. Pre-curate sample prompts; train users with examples.
- **Data residency** — Q Business is region-local; multi-region requires multiple apps. Plan for global enterprises.
- **Compliance retention**: 30 days default for conversation history; up to 365 days for regulated; align with `RETENTION_DAYS`.

---

## Output artifacts

1. **CDK stacks**:
   - `QBusinessAppStack` — application + index + retriever + web experience + KMS
   - `QBusinessDataSourcesStack` — all connectors + IAM roles + Secrets
   - `QBusinessPluginsStack` — built-in + custom plugins
   - `QBusinessAppRolesStack` — AppRoles per group
   - `QBusinessChannelsStack` — Slack + Teams + browser extension config
   - `QBusinessCustomDomainStack` (optional) — CloudFront + ACM + DNS

2. **Connector configurations** (JSON) per data source

3. **Custom plugin** (if enabled):
   - OpenAPI 3.0 spec
   - Lambda backend (Powertools-based)
   - OAuth2 secret in Secrets Manager

4. **Q Apps** (preview):
   - Onboarding Helper
   - Policy Lookup
   - PR Reviewer

5. **AppRole permissions matrix** (CSV) — group × data source × plugin × Q app

6. **PII Guardrails** configuration (Bedrock Guardrails JSON)

7. **Pytest validation suite** — covers all validation criteria

8. **CloudWatch dashboard** — usage analytics

9. **Change management pack**:
   - Launch announcement template
   - 5-min training video script
   - User tips FAQ
   - Sample prompts (8-10 starters)
   - Common pitfalls + escalation path

10. **Cost projection** — Pro vs Lite × MAU; index units; data source sync GB; plugin invocation overhead

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 2.0 | 2026-04-27 | Initial composite template. Q Business + 40+ connectors + plugins + Q Apps + AppRoles + IDC. 3-5 day enterprise rollout. Wave 15. |
