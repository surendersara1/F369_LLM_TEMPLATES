<!-- Template Version: 2.0 | F369 Wave 19 (composite) | Composes: DATA_MESH_PATTERNS + DATA_DATAZONE_V2 + DATA_GLUE_QUALITY + DATA_LAKE_FORMATION + ENTERPRISE_CONTROL_TOWER -->

# Template 13 — Data Mesh Governance (multi-domain · DataZone v2 · LF-Tag ABAC · Service Catalog blueprints · cross-domain SLO · 3-6 month program)

## Purpose

Implement **AWS-native data mesh** across 5+ data domains in 3-6 months. Output: per-domain AWS accounts, central catalog with DataZone, LF-Tag-based ABAC, Service Catalog self-serve blueprints, federated governance council, data product SLOs, cross-domain subscriptions.

This is the **canonical "data mesh transformation" engagement** — multi-quarter program, biggest enterprise data initiatives. Pairs with `data/13_data_quality_program` (DQ as enabler), `enterprise/09_landing_zone_baseline` (multi-account foundation), `enterprise/10_centralized_security_ops` (security at scale).

---

## Role Definition

You are an expert AWS data mesh architect with deep expertise in:
- 4 Data Mesh principles (Dehghani) — domain ownership, data as product, self-serve infrastructure, federated governance
- Multi-account architecture: central catalog + per-domain producer + consumer accounts
- DataZone v2 — domains, projects, environments, data products, glossary, subscriptions
- Lake Formation TBAC + LF-Tags as governance ABAC layer
- AWS Service Catalog for self-serve infrastructure blueprints
- Account Factory for Terraform (AFT) for new account provisioning
- OpenLineage for cross-domain lineage
- Federated computational governance (central council + domain enforcement)
- Change management for organizational transformation

Generate complete, production-deployable code. No TODOs.

---

## Context and Inputs

```
PROJECT_NAME:                [REQUIRED]
ENTERPRISE_NAME:             [REQUIRED]
PRIMARY_REGION:              [REQUIRED]

# --- DOMAINS ---
DOMAINS:                     [REQUIRED — JSON array of domains:
                              [
                                {"name":"product","desc":"Orders, products, inventory","data_eng_capacity":3},
                                {"name":"finance","desc":"Invoices, GL","data_eng_capacity":2},
                                {"name":"hr","desc":"People, payroll (restricted)","data_eng_capacity":1},
                                {"name":"marketing","desc":"Campaigns, ads","data_eng_capacity":2},
                                {"name":"engineering","desc":"Telemetry","data_eng_capacity":4}
                              ]
                             ]

# --- ACCOUNTS ---
CENTRAL_CATALOG_ACCOUNT_ID:  [REQUIRED]
ENABLE_PER_DOMAIN_ACCOUNTS:  [true required for Stage 4 mesh]
USE_AFT:                     [true required if multi-account]
LANDING_ZONE_ALREADY_DEPLOYED: [REQUIRED — Wave 11 done?]

# --- IDENTITY ---
IDC_INSTANCE_ARN:            [REQUIRED]
DATA_COUNCIL_GROUP:          [REQUIRED — IDC group name; e.g. "Data-Council-Members"]

# --- SLO ---
DEFAULT_FRESHNESS_HOURS:     [1 default — for high-criticality]
DEFAULT_AVAILABILITY_PCT:    [99.9 default]
DEFAULT_COMPLETENESS_PCT:    [99 default]

# --- TAXONOMY (LF-Tags) ---
LF_TAGS:                     [REQUIRED — JSON map; e.g.
                              {
                                "Domain": ["product","finance","hr","marketing","engineering"],
                                "Classification": ["public","internal","confidential","restricted"],
                                "PII": ["true","false"],
                                "DataClass": ["customer","financial","operational","health","marketing"]
                              }
                             ]

# --- COMPLIANCE ---
KMS_KEY_ARN:                 [REQUIRED — multi-region]
COMPLIANCE_FRAMEWORKS:       [REQUIRED — soc2 | hipaa | pci | gdpr | combined]

# --- OBSERVABILITY ---
SNS_ALARM_TOPIC_ARN:         [REQUIRED]
DASHBOARD_VIEWERS:           [comma-separated IDC group names]
```

---

## Partial Library (Claude MUST load)

| Partial | Why |
|---|---|
| `DATA_MESH_PATTERNS` | Mesh principles + multi-account + LF-Tag ABAC + Service Catalog blueprints |
| `DATA_DATAZONE_V2` | DataZone domains + projects + environments + data products + glossary |
| `DATA_GLUE_QUALITY` | DQ as enabler for data product SLOs |
| `DATA_LAKE_FORMATION` | LF-Tag based access control |
| `ENTERPRISE_CONTROL_TOWER` | Multi-account foundation (AFT for new domain accounts) |
| `ENTERPRISE_IDENTITY_CENTER` | IDC federation across domains |
| `LAYER_SECURITY` | KMS multi-region + Secrets Manager |
| `LAYER_OBSERVABILITY` | Dashboards + cross-account observability |

---

## Architecture (5-domain example)

```
   ┌──────────────────────────────────────────────────────────────────────┐
   │ Central Catalog Account (data-platform)                               │
   │                                                                        │
   │ DataZone Domain: acme                                                  │
   │   - 5 sub-domains (per business unit)                                   │
   │   - Business Glossary: 100+ terms                                        │
   │   - 100+ data products published across domains                          │
   │                                                                        │
   │ Glue Data Catalog                                                       │
   │   - Cross-account databases (RAM-shared)                                  │
   │   - LF-Tag taxonomy: Domain + Classification + PII + DataClass             │
   │                                                                        │
   │ Lake Formation                                                          │
   │   - Tag-based access control (TBAC)                                       │
   │   - Cross-account permissions via RAM                                       │
   │                                                                        │
   │ Service Catalog Portfolio                                               │
   │   - Data Domain Blueprint (Glue + Athena + S3 bucket + LF setup)            │
   │   - ETL Pipeline Blueprint                                                  │
   │   - Data Product Publishing Blueprint                                       │
   │                                                                        │
   │ Data Council                                                            │
   │   - 1 rep per domain + central platform                                     │
   │   - Weekly cadence; quarterly review                                         │
   │   - Owns: glossary, taxonomy, contract standards                              │
   └──────────────────────────────────────────────────────────────────────┘
                       ▲                                          ▲
                       │ catalog                                   │
       ┌───────────────┴────────────┐         ┌───────────────────┴─────────────────────┐
       ▼                            ▼         ▼                                          ▼
   Domain Accounts (per business unit)            Consumer Accounts (per consumer team)
                                                    
   ┌──────────────────────────┐                ┌──────────────────────────┐
   │ Product Domain (acct-A)   │                │ Analytics Account         │
   │   - DataZone Project      │                │   - DataZone Project      │
   │   - Environments (Glue)   │                │   - Environments (Athena, │
   │   - 25 data products       │                │     QuickSight, SM)        │
   │   - Producer team           │                │   - Subscriptions to     │
   └──────────────────────────┘                │     30+ data products     │
                                                  └──────────────────────────┘
   ┌──────────────────────────┐                ┌──────────────────────────┐
   │ Finance Domain (acct-B)   │                │ ML Platform Account       │
   │   - 15 data products       │                │   - SageMaker Studio      │
   └──────────────────────────┘                │   - Subscriptions          │
                                                  └──────────────────────────┘
   ┌──────────────────────────┐                
   │ HR Domain (acct-C)        │                ...
   │   - 10 data products       │                
   │   (restricted-tier)        │                
   └──────────────────────────┘                
                                                  
   ┌──────────────────────────┐                
   │ Marketing Domain (acct-D) │                
   │   - 12 data products       │                
   └──────────────────────────┘                
                                                  
   ┌──────────────────────────┐                
   │ Engineering Domain (E)   │                
   │   - 20 data products       │                
   └──────────────────────────┘                
                                                  
   All federated via IAM Identity Center.
   Cross-domain access via DataZone subscription workflow → LF auto-grant.
```

---

## Quarter-by-quarter execution (3-6 month program)

### Quarter 1 (Months 1-3) — Foundation + first 2 domains

#### Month 1: Foundation
- Pre-flight: confirm `LANDING_ZONE_ALREADY_DEPLOYED` (else block until done)
- Central catalog account: DataZone domain + Glue Catalog + LF-Tag taxonomy
- IAM Identity Center: data council group + initial domain admins
- Service Catalog portfolio + 3 blueprints (Data Domain, ETL Pipeline, Data Product)
- Data Council established + weekly cadence + charter signed
- **Deliverable:** Central catalog operational; council functional.

#### Month 2: First domain (highest-readiness, e.g., Engineering)
- Provision domain account via AFT
- Apply Service Catalog blueprint to set up domain Glue + S3 + Athena
- Migrate existing engineering data sources to domain account
- Publish first 5 data products to DataZone catalog
- Internal training + onboarding
- First subscription request from analytics team → approved
- **Deliverable:** Engineering domain operational; first cross-domain subscription end-to-end.

#### Month 3: Second domain (Product)
- Same as Month 2 for Product domain
- 10 data products published
- Lessons learned applied to onboarding process
- **Deliverable:** 2 domains live; 15 total data products; subscription workflow tested across multiple producers/consumers.

### Quarter 2 (Months 4-6) — Remaining domains + governance + maturity

#### Month 4: Domains 3-5 (Finance, HR, Marketing)
- Onboard in parallel (ideally — given AFT automation)
- Restricted-tier domain (HR): extra LF tag (Confidential, PII), gated subscription approval
- Cross-domain dependencies surfaced (e.g., marketing needs customer data from product domain)
- **Deliverable:** All 5 domains operational; ~70 total data products.

#### Month 5: Federated governance + SLO program
- Per-domain DQ programs in flight (using `data/13_data_quality_program`)
- Data product SLOs published per product
- Cross-domain lineage captured (OpenLineage emitters in Glue jobs)
- Quarterly council review: glossary additions, contract standards, SLO breaches
- **Deliverable:** SLO dashboards per data product; lineage graph across domains.

#### Month 6: Maturity + handoff
- All domains have ≥ 95% products with SLO defined
- Data Council monthly review process documented
- Producer-consumer feedback loops established (subscription satisfaction)
- 6-month program retrospective: what worked, what didn't, roadmap for Q3+
- **Deliverable:** Mature mesh; council ownership; clear roadmap for next quarter.

---

## Validation criteria (program-level)

- [ ] **Central catalog account** has DataZone domain ACTIVE
- [ ] **All `DOMAINS`** have own AWS account (per-domain isolation)
- [ ] **LF-Tag taxonomy** defined and enforced via TBAC
- [ ] **Service Catalog portfolio** shared with all domain accounts
- [ ] **All domains have IDC permission sets** for their teams
- [ ] **Data Council** has met ≥ 12 times (weekly Q1; bi-weekly Q2)
- [ ] **≥ 50 data products published** in catalog
- [ ] **Cross-domain subscriptions ≥ 20 active** (proof of value)
- [ ] **All published products have SLO YAML** in Git
- [ ] **DQ programs active for ≥ 80% of critical tables** (high criticality only)
- [ ] **Cross-domain lineage** captured for ≥ 70% of data products
- [ ] **Quarterly council review** completed with action items tracked

## Validation criteria (per-domain)

- [ ] Domain has dedicated AWS account
- [ ] Domain has DataZone project + ≥ 1 environment
- [ ] Domain has published ≥ 5 data products
- [ ] Domain has data eng capacity (≥ 1 dedicated engineer)
- [ ] Domain participates in Data Council (1 rep)
- [ ] Domain has runbooks for producer responsibilities

---

## Common gotchas (claude must address proactively)

- **Don't try mesh on Day 1** — wait until org has 3+ data teams + ≥ 50 sources. Smaller orgs benefit more from central data lake.
- **Domain readiness varies** — Engineering team may be ready; HR not. Onboard in order of readiness, not org chart.
- **Federated governance fails without buy-in** — Data Council needs senior sponsorship; without it, decays into anarchy.
- **HR / restricted domains need extra controls** — separate LF tags + tighter subscription approval + audit trail required for compliance.
- **Cross-domain dependencies form cycles** — "orders depends on customer; customer team needs orders for analytics." Council resolves; document.
- **Data quality is the enabler** — without DQ, data products are unreliable; consumers don't trust mesh. Run `data/13_data_quality_program` in parallel.
- **OpenLineage gaps** — non-Glue sources (e.g., Spark on EMR, custom ETL) need emitter integration. Phased rollout.
- **Service Catalog blueprint maintenance** — central platform owns. Without dedicated owner, blueprints rot in 6 months.
- **AFT account provisioning** takes 30-60 min per account; allow buffer.
- **Cross-account RAM shares** require trust + approval. Document setup; rotate periodically.
- **DataZone API stability** — preview features may change; pin SDK versions.
- **Cost transparency** — domain accounts make per-domain cost visible. Some domains panic at first. Reassure with comparisons.
- **Migration from monolith** — many orgs have years of central data lake; migrating to mesh = phased; keep both running.
- **Cultural shift** — biggest barrier. "We're our own data team now?" Train + invest in data eng per domain.

---

## Output artifacts

1. **CDK stacks**:
   - `CentralCatalogStack` — DataZone domain + Glue Catalog + LF-Tags + Service Catalog
   - `DomainAccountTemplate` — AFT-deployable per-domain account setup
   - `DataCouncilStack` — IDC group + meeting cadence resources
   - `LineageStack` — OpenLineage receiver + cross-domain integration

2. **Service Catalog blueprints**:
   - Data Domain Setup
   - ETL Pipeline (Glue)
   - Data Product Publishing
   - Glue Data Quality Ruleset

3. **AFT account customizations** for new domain accounts

4. **DataZone artifacts**:
   - Business glossary (100+ terms)
   - Asset publishing automation
   - Subscription approval workflow

5. **Data product SLO templates** (YAML)

6. **Federated governance artifacts**:
   - Data Council charter
   - Glossary curation guide
   - Contract review process
   - Quarterly review template

7. **Cross-domain lineage** (OpenLineage emitters, Glue plugin)

8. **CloudWatch dashboards**:
   - Mesh program overview (domain count, product count, subscription count)
   - Per-domain dashboards
   - SLO dashboards across products

9. **Pytest validation suite**

10. **Change management pack**:
    - Domain onboarding checklist
    - Data council training video
    - Producer team training (data product responsibility)
    - Consumer team training (subscription workflow)

11. **Cost model**:
    - Per-domain account base cost
    - DataZone per project + environment
    - Glue Data Catalog cross-account
    - LF + RAM
    - 6-month + 12-month projections

12. **Roadmap** for Q3+ (mesh evolution, automation, additional domains)

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 2.0 | 2026-04-28 | Initial composite template. Multi-domain data mesh + DataZone + LF-Tag ABAC + Service Catalog + federated governance + data products + SLOs + lineage. 3-6 month program. Wave 19. |
