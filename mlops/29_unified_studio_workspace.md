<!-- Template Version: 2.0 | F369 Wave 8 (composite) | Composes: MLOPS_SAGEMAKER_UNIFIED_STUDIO + MLOPS_STUDIO_SPACES_LIFECYCLE + MLOPS_CANVAS_NO_CODE + MLOPS_LINEAGE_TRACKING -->

# Template 29 — Unified Studio Workspace (DataZone + per-user Spaces + Canvas + MLflow + Lineage)

## Purpose

Stand up a **modern Unified Studio workspace** for an ML platform engagement: DataZone-integrated workspace with per-user Studio Spaces, Canvas no-code UI for citizen data scientists, MLflow Apps for experiment tracking, lineage capture, and Bedrock integration. Replaces classic Studio for new builds.

Generates production-deployable CDK + bootstrap script + user onboarding runbook.

---

## Role Definition

You are an expert AWS ML platform engineer with deep expertise in:
- DataZone domain + project structure (data mesh)
- SageMaker Unified Studio (2024+ enhancements: Data Agent, MLflow Apps, TIP, S3 Tables IAM mode)
- Studio Spaces (private + shared) with custom images + lifecycle configurations
- Canvas (no-code ML, AutoML, GenAI Q&A, JumpStart UI)
- MLflow 3.0 managed tracking servers
- Trusted Identity Propagation (TIP) for SSO context across Glue/Athena/Redshift
- Bedrock invoke from Studio notebooks

Generate complete production-deployable code, no TODOs.

---

## Context and Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED — dev | stage | prod]

# --- IDENTITY ---
IAM_IDC_INSTANCE_ARN:   [REQUIRED — IAM Identity Center instance arn]
DEFAULT_GROUP_ID:       [REQUIRED — IDC group for all data team users]

# --- DATAZONE DOMAIN ---
DOMAIN_NAME:            [REQUIRED — e.g. acme-data-domain]
DOMAIN_VERSION:         [OPTIONAL — V2 default]

# --- PROJECTS ---
PROJECTS:               [REQUIRED — list of (name, member_group_id) tuples]
                        Example: [("finance-team", "g-fin-1234"), ("ml-platform", "g-ml-5678")]

# --- DATA SOURCES PER PROJECT ---
GLUE_DATABASE_PATTERNS: [list per project — e.g. "finance_*"]
ATHENA_WORKGROUP_ARN:   [optional]
REDSHIFT_CLUSTER_ARN:   [optional]
SNOWFLAKE_GLUE_CONN:    [optional]
S3_TABLES_BUCKET_ARN:   [optional]

# --- STUDIO SPACES ---
ENABLE_PER_USER_SPACES: [OPTIONAL — true; default true]
DEFAULT_SPACE_INSTANCE: [OPTIONAL — ml.t3.medium]
SPACE_EBS_GB:           [OPTIONAL — 100]
CUSTOM_IMAGE_URI:       [OPTIONAL — ECR image with company packages]

# --- CANVAS ---
ENABLE_CANVAS:          [OPTIONAL — true]
CANVAS_DIRECT_DEPLOY:   [OPTIONAL — false in regulated industries]
CANVAS_BEDROCK_MODELS:  [OPTIONAL — list of Bedrock model ARNs accessible from Canvas]

# --- MLFLOW APPS ---
ENABLE_MLFLOW:          [OPTIONAL — true]
MLFLOW_APP_SIZE:        [OPTIONAL — Small | Medium | Large]

# --- LINEAGE ---
ENABLE_LINEAGE:         [OPTIONAL — true]
MODEL_PACKAGE_GROUP:    [REQUIRED if lineage]

# --- COMPLIANCE ---
KMS_KEY_ARN:            [REQUIRED]
LF_TBAC_ENABLED:        [OPTIONAL — true; required for data mesh]
TIP_ENABLED:            [OPTIONAL — true; SSO context propagation]
```

---

## Partial Library

| Partial | Why |
|---|---|
| `MLOPS_SAGEMAKER_UNIFIED_STUDIO` | Domain + project + environment + MLflow Apps + TIP setup |
| `MLOPS_STUDIO_SPACES_LIFECYCLE` | Per-user Spaces + Custom Images + Lifecycle Configs |
| `MLOPS_CANVAS_NO_CODE` | Canvas enable + per-user opt-in + Bedrock integration |
| `MLOPS_LINEAGE_TRACKING` | Auto-capture from Pipelines + Model Cards |
| `DATA_DATAZONE` | DataZone domain mesh patterns |
| `DATA_LAKE_FORMATION` | LF-TBAC required for TIP |
| `DATA_GLUE_CATALOG` | Glue Catalog data sources |

---

## Architecture

```
   IAM Identity Center (out-of-band)
        │
        │  Trusted Identity Propagation
        ▼
   ┌─────────────────────────────────────────────────────────────────┐
   │  DataZone Domain: acme-data-domain                              │
   │     - Domain version V2                                          │
   │     - SSO via IAM Identity Center                                │
   │     - KMS-encrypted                                              │
   └────────────────┬────────────────────────────────────────────────┘
                    │
        ┌───────────┼────────────────────────┐
        │           │                        │
        ▼           ▼                        ▼
   Project:        Project:             Project:
   finance-team    ml-platform          research
        │             │                       │
        └─────────────┼───────────────────────┘
                      │
                      ▼
   ┌─────────────────────────────────────────────────────────────────┐
   │  Per-project resources:                                          │
   │     - Glue Catalog data source (auto-import tables)               │
   │     - Athena workgroup (analyst-tier)                             │
   │     - Redshift connection                                         │
   │     - S3 Tables in IAM mode                                       │
   │     - MLflow App (project-scoped tracking server)                 │
   │     - SageMaker AI domain (Studio Spaces)                          │
   └────────────────┬────────────────────────────────────────────────┘
                    │
                    ▼
   ┌─────────────────────────────────────────────────────────────────┐
   │  Per-user (in each project):                                     │
   │     - User Profile w/ scoped IAM execution role                    │
   │     - Private Space — JupyterLab v4 (custom image, LCC)            │
   │     - Private Space — Code Editor (VSCode-based)                    │
   │     - Canvas access (if enabled, with per-user S3 prefix)          │
   │     - 100 GB EBS persistent across restarts                         │
   └─────────────────────────────────────────────────────────────────┘

   Lineage: every notebook run + Pipeline → auto-captured Artifacts/Actions
   Bedrock: invokable from notebooks + Canvas (via dedicated invoke role)
```

---

## Day-by-day execution (10-day POC)

### Day 1-2 — DataZone domain + identity
- IAM Identity Center prerequisites verified (out-of-band)
- DataZone domain (V2) with SSO + KMS
- Custom resource: enable `DefaultDataLake` blueprint
- Per-project DataZone projects + members
- **Deliverable:** domain portal accessible via SSO; 2+ projects visible

### Day 3-4 — Per-project environments + data sources
- DataZone environment per project (DataLake blueprint)
- Glue Catalog data source w/ table-pattern includes + LF-TBAC tagging
- Athena workgroup + result bucket connections
- Redshift connection (if applicable)
- S3 Tables data source in IAM mode
- **Deliverable:** project members can browse Glue tables in Catalog UI

### Day 5-6 — MLflow Apps + Lineage
- MLflow App per project (managed tracking server, project-scoped IAM)
- Auto-register MLflow runs to SageMaker Model Registry
- Lineage capture from Pipelines (auto-on)
- Model Card template
- **Deliverable:** sample MLflow run logged from notebook; auto-registered model package

### Day 7-8 — Studio Spaces
- Custom Studio Image (Dockerfile w/ company packages, push to ECR)
- StudioLifecycleConfig (mount FSx, install corp packages, configure Git)
- Per-user UserProfile w/ Spaces enabled
- 2 spaces per user: JupyterLab (GPU-enabled) + Code Editor (CPU)
- 100 GB EBS persistent
- **Deliverable:** user opens Studio → sees both spaces; spaces persist EBS across stops

### Day 9 — Canvas
- Canvas enabled at domain (TimeSeriesForecasting, ModelRegister, DirectDeploy=false in prod, GenerativeAI w/ Bedrock)
- Per-user CanvasAppSettings with KMS-encrypted workspace
- Pre-create Glue Connections (Snowflake, Salesforce) so Canvas users see them
- Canvas → MLOps handoff Lambda
- **Deliverable:** sample tabular ML built in Canvas; Model Package created in target group

### Day 10 — TIP + UAT
- Glue connections upgraded to TIP authentication
- Verify CloudTrail shows actual user (not service role) on Athena scans
- UAT: 3 users in different projects; verify isolation, lineage, MLflow per project
- **Deliverable:** TIP working; per-project isolation verified; user onboarding runbook

---

## Validation criteria

- [ ] DataZone portal opens via SSO; users see only their projects
- [ ] Project member can browse Glue tables; non-member cannot
- [ ] LF-TBAC blocks unauthorized table+column reads
- [ ] MLflow runs auto-register to Model Registry
- [ ] Studio Spaces persist EBS across restarts
- [ ] Canvas user successfully builds + registers model
- [ ] Canvas user CANNOT deploy to prod (DirectDeploy=false)
- [ ] TIP propagates user identity to Athena CloudTrail logs
- [ ] Bedrock invoke works from notebook + Canvas

---

## Output artifacts

1. CDK app (DomainStack + ProjectsStack + StudioSpacesStack + CanvasStack)
2. Bootstrap script for project environment + data source creation
3. Custom Studio Image Dockerfile + ECR push script
4. Lifecycle Configuration scripts (Jupyter + Code Editor)
5. Per-project IAM role templates
6. Glue Connection templates (Snowflake, Salesforce, Redshift)
7. User onboarding runbook (how to add a user, request a space, build first model)
8. Lineage query Lambda + sample compliance report
9. Cost dashboard (per-project Studio + MLflow + Bedrock spend)

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 2.0 | 2026-04-26 | Initial composite. Composes Unified Studio + Spaces + Canvas + Lineage. Wave 8. |
