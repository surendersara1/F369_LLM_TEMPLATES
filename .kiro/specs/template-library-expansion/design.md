# Design Document: Template Library Expansion

## Overview

This design expands the F369 AWS MLOps & AI/LLM CI/CD Prompt Template Library from 29 templates across 4 directories to 67 templates across 8 directories. The expansion adds 38 new markdown prompt templates organized into 7 thematic areas: AI/ML Gaps (6 templates), Data Engineering (5 templates), Security & Compliance (5 templates), Cost Management / FinOps (5 templates), Multi-Account / Enterprise (5 templates), Observability (5 templates), and Edge / Hybrid (3 templates). Four new directories are created: `data/`, `finops/`, `enterprise/`, and `edge/`.

Each template is an LLM prompt file — a carefully structured markdown document that users paste into Claude or another LLM to generate production-ready AWS infrastructure code. Templates are NOT executable code; they are prompt engineering artifacts containing role definitions, parameterized inputs, file-by-file generation instructions, code scaffolding hints (concrete AWS API calls), and integration points for chaining.

### Design Goals

- Maintain strict consistency with the established F369 template format across all 38 new templates
- Ensure accurate AWS API references (boto3 1.35+, Terraform AWS Provider 5.70+, CDK 2.170+)
- Build a coherent integration graph where templates chain together across all 8 directories
- Organize templates so each expansion area is self-contained but connects to the existing MLOps core

### Key Decisions

1. **One file per template**: Each template is a single markdown file. No shared includes or partials — each file is self-contained for copy-paste into an LLM.
2. **Consistent parameter block**: All templates share the same first four required parameters (`PROJECT_NAME`, `AWS_REGION`, `AWS_ACCOUNT_ID`, `ENV`) for cross-template consistency.
3. **Integration via SSM and ARN references**: Templates chain by referencing outputs from other templates (SSM parameter paths, ARNs, S3 paths) rather than direct imports.
4. **Code scaffolding over prose**: Each template embeds concrete AWS SDK calls and API patterns rather than abstract descriptions, steering the LLM toward correct implementations.

---

## Architecture

### Expanded Directory Structure

```
F369_LLM_TEMPLATES/
├── README.md                              Updated master index
├── PROMPT_GUIDE.md                        Updated usage guide + interaction map
├── Library.md                             Updated full library overview
│
├── mlops/                                 (20 templates: 00-19)
│   ├── 00-13                              Existing templates (unchanged)
│   ├── 14_bedrock_agents_action_groups.md        NEW — Bedrock Agents + Action Groups
│   ├── 15_multimodal_processing_pipelines.md     NEW — Image/Audio/Document pipelines
│   ├── 16_bedrock_flows_orchestration.md         NEW — Bedrock Flows DAGs
│   ├── 17_llm_evaluation_pipeline.md             NEW — LLM eval + A/B testing
│   ├── 18_prompt_caching_patterns.md             NEW — Prompt caching strategies
│   └── 19_bedrock_marketplace_models.md          NEW — Marketplace model deployment
│
├── cicd/                                  (5 templates: unchanged)
│
├── iac/                                   (4 templates: unchanged)
│
├── devops/                                (14 templates: 01-14)
│   ├── 01-04                              Existing templates (unchanged)
│   ├── 05_config_rules_ml_compliance.md          NEW — AWS Config for ML
│   ├── 06_guardduty_securityhub_ml.md            NEW — GuardDuty + Security Hub
│   ├── 07_macie_pii_training_data.md             NEW — Macie PII detection
│   ├── 08_kms_encryption_ml.md                   NEW — KMS key management
│   ├── 09_vpc_endpoint_policies_ml.md            NEW — VPC endpoint policies
│   ├── 10_opentelemetry_ml_tracing.md            NEW — OpenTelemetry + X-Ray
│   ├── 11_custom_cloudwatch_model_quality.md     NEW — Custom model metrics
│   ├── 12_bedrock_invocation_logging.md          NEW — Bedrock logging + Athena
│   ├── 13_cost_per_inference_dashboards.md       NEW — Cost-per-inference
│   └── 14_clarify_realtime_bias_monitoring.md    NEW — Clarify bias monitoring
│
├── data/                                  (5 templates: NEW directory)
│   ├── 01_glue_etl_ml_features.md                NEW — Glue ETL for ML
│   ├── 02_kinesis_realtime_features.md           NEW — Kinesis streaming features
│   ├── 03_lake_formation_ml_governance.md        NEW — Lake Formation governance
│   ├── 04_s3_data_lifecycle_ml.md                NEW — S3 lifecycle management
│   └── 05_eventbridge_ml_orchestration.md        NEW — EventBridge ML events
│
├── finops/                                (5 templates: NEW directory)
│   ├── 01_cost_allocation_ml.md                  NEW — Cost allocation + tagging
│   ├── 02_spot_instance_strategies_ml.md         NEW — Spot training patterns
│   ├── 03_savings_plans_ml.md                    NEW — Savings Plans analysis
│   ├── 04_inference_cost_optimization.md         NEW — Inference cost optimization
│   └── 05_finops_dashboards_ml.md                NEW — FinOps dashboards
│
├── enterprise/                            (5 templates: NEW directory)
│   ├── 01_organizations_scps_ml.md               NEW — SCPs for ML governance
│   ├── 02_cross_account_model_deployment.md      NEW — Cross-account pipelines
│   ├── 03_service_catalog_ml.md                  NEW — Service Catalog products
│   ├── 04_control_tower_ml.md                    NEW — Control Tower for ML
│   └── 05_centralized_model_registry.md          NEW — Centralized registry
│
└── edge/                                  (3 templates: NEW directory)
    ├── 01_sagemaker_edge_deployment.md           NEW — SageMaker Edge Manager
    ├── 02_iot_greengrass_ml_inference.md          NEW — IoT Greengrass ML
    └── 03_outposts_ml_patterns.md                NEW — Outposts hybrid ML
```

### Template Integration Graph

The following diagram shows how the 38 new templates (bold) integrate with existing templates and each other:

```mermaid
graph TD
    subgraph Foundation["Foundation Layer (Existing)"]
        IAM[devops/04 IAM]
        VPC[devops/02 VPC]
        ECR[devops/01 ECR]
        CW[devops/03 CloudWatch]
    end

    subgraph Security["Security & Compliance (NEW)"]
        CONFIG[devops/05 Config Rules]
        GD[devops/06 GuardDuty+SecHub]
        MACIE[devops/07 Macie PII]
        KMS[devops/08 KMS Encryption]
        VPCE[devops/09 VPC Endpoints]
    end

    subgraph DataEng["Data Engineering (NEW)"]
        GLUE[data/01 Glue ETL]
        KINESIS[data/02 Kinesis Features]
        LF[data/03 Lake Formation]
        S3LC[data/04 S3 Lifecycle]
        EB[data/05 EventBridge ML]
    end

    subgraph MLOpsNew["AI/ML Gaps (NEW)"]
        AGENTS[mlops/14 Bedrock Agents]
        MULTI[mlops/15 Multimodal]
        FLOWS[mlops/16 Bedrock Flows]
        EVAL[mlops/17 LLM Evaluation]
        CACHE[mlops/18 Prompt Caching]
        MARKET[mlops/19 Marketplace]
    end

    subgraph Observability["Observability (NEW)"]
        OTEL[devops/10 OpenTelemetry]
        CWCUST[devops/11 Custom Metrics]
        BLOG[devops/12 Bedrock Logging]
        CPI[devops/13 Cost-per-Inference]
        BIAS[devops/14 Clarify Bias]
    end

    subgraph FinOps["Cost Management (NEW)"]
        COST[finops/01 Cost Allocation]
        SPOT[finops/02 Spot Strategies]
        SP[finops/03 Savings Plans]
        ICO[finops/04 Inference Cost Opt]
        DASH[finops/05 FinOps Dashboards]
    end

    subgraph Enterprise["Multi-Account (NEW)"]
        SCP[enterprise/01 SCPs]
        XACCT[enterprise/02 Cross-Account]
        SC[enterprise/03 Service Catalog]
        CT[enterprise/04 Control Tower]
        CREG[enterprise/05 Central Registry]
    end

    subgraph Edge["Edge/Hybrid (NEW)"]
        EDGE[edge/01 SageMaker Edge]
        GG[edge/02 IoT Greengrass]
        OP[edge/03 Outposts]
    end

    IAM --> CONFIG & GD & KMS & VPCE
    IAM --> GLUE & LF & KINESIS
    IAM --> AGENTS & FLOWS & MARKET
    IAM --> SCP & XACCT & SC
    VPC --> VPCE
    KMS --> CONFIG & XACCT
    EB --> GLUE & KINESIS & EVAL
    GLUE --> KINESIS & LF
    CW --> CWCUST & OTEL & BLOG & CPI
    CWCUST --> DASH & CPI
    BLOG --> CPI & DASH
    COST --> DASH & SP
    SPOT --> COST
    ICO --> CPI & DASH
    XACCT --> CREG
    SCP --> CT
    CT --> SC
    EDGE --> GG
    BIAS --> CW

---

## Components and Interfaces

### Template Anatomy (Shared Structure)

Every new template follows this exact section order, matching the established F369 format:

```markdown
<!-- Template Version: 1.0 | boto3: 1.35+ | [additional tool versions] -->

# Template [Category] [Number] — [Title]

## Purpose
[One paragraph describing what the LLM will generate]

## Role Definition
[Expert persona the LLM adopts — domain expertise, AWS services, tools]

## Context & Inputs
[Parameterized block with REQUIRED and OPTIONAL fields]

## Task
[ASCII directory tree + file-by-file generation instructions with API calls]

## Output Format
[Standard: "Output ALL files with headers: `### FILE: [path]`"]

## Requirements & Constraints
[Security, cost, naming, compliance constraints]

## Code Scaffolding Hints
[Working AWS SDK code snippets that steer the LLM]

## Integration Points
[Upstream/Downstream references using category/number format]
```

### Shared Parameter Convention

All 38 new templates begin their Context & Inputs section with:

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]
```

Additional parameters are template-specific. Templates supporting multiple IaC tools include:
```
IAC_TOOL:               [REQUIRED - boto3 | terraform | cdk]
```

### Resource Naming Convention

All generated AWS resources follow: `{PROJECT_NAME}-{component}-{ENV}`

Examples:
- `myai-config-rule-sagemaker-encryption-prod`
- `myai-glue-feature-engineering-dev`
- `myai-bedrock-agent-prod`

### Integration Points Format

Each template's Integration Points section uses this format:
```markdown
- **Upstream**: `category/number` → [description of consumed output]
- **Downstream**: `category/number` → [description of produced output]
- **Alternative to**: `category/number` → [when applicable]
```

### Expansion Area Component Details

#### 1. AI/ML Gaps (mlops/14-19)

| Template | Primary AWS Services | Key API Calls |
|----------|---------------------|---------------|
| mlops/14 — Bedrock Agents | Bedrock Agent, Lambda | `bedrock_agent.create_agent()`, `create_agent_action_group()`, `bedrock_agent_runtime.invoke_agent()` |
| mlops/15 — Multimodal | Bedrock Runtime, Rekognition, Transcribe, Textract | `bedrock_runtime.invoke_model()` (vision), `rekognition.detect_labels()`, `transcribe.start_transcription_job()`, `textract.start_document_analysis()` |
| mlops/16 — Bedrock Flows | Bedrock Agent | `bedrock_agent.create_flow()`, `create_flow_version()`, `create_flow_alias()`, `bedrock_agent_runtime.invoke_flow()` |
| mlops/17 — LLM Evaluation | Bedrock, SageMaker, Step Functions | `bedrock.create_evaluation_job()`, ROUGE/BLEU/BERTScore libraries, Step Functions human-in-the-loop |
| mlops/18 — Prompt Caching | Bedrock Runtime, ElastiCache, DynamoDB | `bedrock_runtime.converse()` with `cachePoint`, Redis vector similarity, DynamoDB TTL |
| mlops/19 — Marketplace Models | Bedrock | `bedrock.list_foundation_models()`, `create_provisioned_model_throughput()`, `create_inference_profile()` |

#### 2. Data Engineering (data/01-05)

| Template | Primary AWS Services | Key API Calls |
|----------|---------------------|---------------|
| data/01 — Glue ETL | Glue, S3 | `glue.create_job()`, `create_crawler()`, `create_data_quality_ruleset()`, PySpark `GlueContext` |
| data/02 — Kinesis Features | Kinesis, Lambda, Firehose, Flink | `kinesis.put_record()`, Lambda event source mapping, Firehose to S3/OpenSearch, Flink SQL |
| data/03 — Lake Formation | Lake Formation, Glue Catalog | `lakeformation.register_resource()`, `grant_permissions()`, LF-TBAC tags |
| data/04 — S3 Lifecycle | S3, S3 Control | `s3control.create_job()`, `create_access_point()`, Intelligent-Tiering, lifecycle rules |
| data/05 — EventBridge ML | EventBridge, Step Functions | `events.put_rule()`, `put_targets()`, `put_events()`, custom event bus, schema registry |

#### 3. Security & Compliance (devops/05-09)

| Template | Primary AWS Services | Key API Calls |
|----------|---------------------|---------------|
| devops/05 — Config Rules | AWS Config, SSM Automation | `config.put_config_rule()`, `put_conformance_pack()`, custom Lambda evaluators |
| devops/06 — GuardDuty + SecHub | GuardDuty, Security Hub | `securityhub.create_automation_rule()`, `create_insight()`, Lambda remediation |
| devops/07 — Macie PII | Macie, Step Functions | `macie2.create_classification_job()`, `create_custom_data_identifier()`, remediation workflow |
| devops/08 — KMS Encryption | KMS | `kms.create_key()`, `enable_key_rotation()`, key policies with `kms:ViaService` |
| devops/09 — VPC Endpoints | EC2 (VPC) | `ec2.create_vpc_endpoint()`, endpoint policies, security groups |

#### 4. Cost Management / FinOps (finops/01-05)

| Template | Primary AWS Services | Key API Calls |
|----------|---------------------|---------------|
| finops/01 — Cost Allocation | Budgets, Cost Explorer, Organizations | `budgets.create_budget()`, `ce.get_cost_and_usage()`, `ce.create_anomaly_monitor()` |
| finops/02 — Spot Strategies | SageMaker | `sagemaker.create_training_job()` with `EnableManagedSpotTraining`, checkpointing, `pricing.get_products()` |
| finops/03 — Savings Plans | Savings Plans, Cost Explorer | `savingsplans.describe_savings_plans()`, `ce.get_savings_plans_purchase_recommendation()`, `ce.get_rightsizing_recommendation()` |
| finops/04 — Inference Cost Opt | SageMaker, Application Auto Scaling | `application_autoscaling.put_scaling_policy()`, scale-to-zero, batch transform, multi-model endpoints |
| finops/05 — FinOps Dashboards | CloudWatch, QuickSight | `cloudwatch.put_dashboard()`, `quicksight.create_data_set()`, metric math for cost-per-inference |

#### 5. Multi-Account / Enterprise (enterprise/01-05)

| Template | Primary AWS Services | Key API Calls |
|----------|---------------------|---------------|
| enterprise/01 — SCPs | Organizations | `organizations.create_policy()`, `attach_policy()`, condition keys for ML services |
| enterprise/02 — Cross-Account Deploy | CodePipeline, SageMaker, IAM | Cross-account IAM roles, `sagemaker.put_model_package_group_policy()`, shared artifact bucket |
| enterprise/03 — Service Catalog | Service Catalog | `servicecatalog.create_portfolio()`, `create_product()`, `create_constraint()`, CloudFormation products |
| enterprise/04 — Control Tower | Control Tower, Config, Organizations | CfCT customizations, detective/preventive guardrails, account baseline configs |
| enterprise/05 — Central Registry | SageMaker Model Registry | `sagemaker.put_model_package_group_policy()`, cross-account EventBridge, lineage metadata |

#### 6. Observability (devops/10-14)

| Template | Primary AWS Services | Key API Calls |
|----------|---------------------|---------------|
| devops/10 — OpenTelemetry | X-Ray, ADOT, Lambda | ADOT collector config, `opentelemetry-api` instrumentation, `xray.create_group()`, custom spans |
| devops/11 — Custom Metrics | CloudWatch | `cloudwatch.put_metric_data()`, metric math, `put_anomaly_detector()`, statistic sets |
| devops/12 — Bedrock Logging | Bedrock, Athena, CloudWatch Logs | `bedrock.put_model_invocation_logging_configuration()`, Athena DDL, Logs Insights queries |
| devops/13 — Cost-per-Inference | CloudWatch, QuickSight, DynamoDB | Metric math (cost/invocations), per-customer DynamoDB tracking, QuickSight analysis |
| devops/14 — Clarify Bias | SageMaker Clarify | `sagemaker.create_endpoint_config()` with `ClarifyExplainerConfig`, `create_monitoring_schedule()` with `ModelBiasMonitor` |

#### 7. Edge / Hybrid (edge/01-03)

| Template | Primary AWS Services | Key API Calls |
|----------|---------------------|---------------|
| edge/01 — SageMaker Edge | SageMaker Neo, Edge Manager, IoT | `sagemaker.create_compilation_job()`, `create_device_fleet()`, `create_edge_packaging_job()`, IoT Jobs |
| edge/02 — IoT Greengrass | IoT Greengrass V2 | `greengrassv2.create_deployment()`, component recipes (YAML), Stream Manager, model optimization |
| edge/03 — Outposts | SageMaker, S3 Outposts, ECS | `s3outposts.create_bucket()`, Outposts subnet targeting, DataSync, local inference continuity |

---

## Data Models

### Template File Schema

Each template markdown file conforms to this logical schema:

```
TemplateFile:
  version_comment: string          # <!-- Template Version: 1.0 | boto3: 1.35+ | ... -->
  title: string                    # "# Template [Category] [Number] — [Title]"
  purpose: string                  # One-paragraph description
  role_definition: string          # Expert persona paragraph
  context_inputs:                  # Parameter block
    required_params:
      - PROJECT_NAME: string
      - AWS_REGION: string
      - AWS_ACCOUNT_ID: string
      - ENV: enum[dev, stage, prod]
      - [template-specific required params]
    optional_params:
      - [template-specific optional params with defaults]
  task:                            # Generation instructions
    directory_tree: string         # ASCII art project structure
    file_instructions: list        # Per-file generation details with API calls
  output_format: string            # Standard output header format
  requirements_constraints: string # Security, cost, naming rules
  code_scaffolding_hints: string   # Working AWS SDK code snippets
  integration_points:              # Cross-template references
    upstream: list[{ref: string, description: string}]
    downstream: list[{ref: string, description: string}]
    alternative_to: list[{ref: string, description: string}]  # optional
```

### Integration Points Reference Map

Key cross-template data flows for the new templates:

| Producer Template | Output | Consumer Templates |
|-------------------|--------|--------------------|
| devops/04 (IAM) | Role ARNs via SSM | All 38 new templates |
| devops/08 (KMS) | KMS key ARNs | devops/05, enterprise/02, data/01, data/03 |
| data/01 (Glue) | Feature datasets in S3/Parquet | mlops/07, mlops/01, data/02 |
| data/05 (EventBridge) | ML event bus + rules | mlops/13, devops/05, enterprise/05 |
| mlops/14 (Agents) | Agent ID + alias ARN | mlops/16, devops/10 |
| mlops/17 (Evaluation) | Eval metrics JSON | mlops/10, enterprise/05 |
| enterprise/05 (Central Registry) | Model package ARNs | enterprise/02, mlops/03 |
| devops/10 (OpenTelemetry) | X-Ray traces | devops/13, devops/11 |
| devops/12 (Bedrock Logging) | Invocation logs in S3 | devops/13, finops/01 |
| finops/01 (Cost Allocation) | Tagged cost data | finops/05, devops/13 |

---

## Error Handling

Since templates are LLM prompt files (not executable code), error handling applies at two levels:

### Template Authoring Errors
- **Missing sections**: Each template is validated against the F369 section checklist during review
- **Invalid API references**: Code scaffolding hints are validated against boto3 1.35+ / Terraform AWS Provider 5.70+ / CDK 2.170+ API signatures
- **Broken integration references**: All `category/number` references are checked for bidirectional consistency and file existence
- **Missing required parameters**: Every template's Context & Inputs section is reviewed for the mandatory four parameters

### Error Patterns Embedded in Templates
Each template that involves AWS API calls includes error handling guidance for the generated code:
- **Retry patterns**: Templates for async operations (Glue jobs, Bedrock fine-tuning, SageMaker training) include polling with exponential backoff
- **Dead letter queues**: Streaming templates (data/02 Kinesis) include DLQ routing for malformed records
- **Remediation workflows**: Security templates (devops/05-07) include automated remediation via Lambda + Step Functions
- **Fallback patterns**: Edge templates (edge/01-03) include offline/degraded mode handling
- **Budget guardrails**: FinOps templates (finops/01-05) include budget breach actions that prevent runaway costs

---

## Testing Strategy

### Why Property-Based Testing Does Not Apply

This feature produces LLM prompt templates — markdown files with embedded code examples. There are no pure functions, parsers, serializers, or algorithms to test with property-based testing. The templates are static documents consumed by humans and LLMs, not executable software with input/output behavior.

PBT is not appropriate because:
- Templates are declarative markdown content, not functions with inputs/outputs
- There is no meaningful "for all inputs X, property P(X) holds" statement for markdown files
- Template quality is validated through structural checks and human review, not randomized input testing

### Applicable Testing Approaches

#### 1. Structural Validation (Automated)
- Verify each template contains all required F369 sections in correct order
- Verify the version comment format: `<!-- Template Version: 1.0 | boto3: 1.35+ | ... -->`
- Verify the first four parameters are `PROJECT_NAME`, `AWS_REGION`, `AWS_ACCOUNT_ID`, `ENV`
- Verify all `[REQUIRED]` and `[OPTIONAL: default]` markers are present
- Verify the output format section contains the standard header instruction

#### 2. Integration Reference Validation (Automated)
- Parse all Integration Points sections across all 67 templates
- Verify every `category/number` reference points to an existing template file
- Verify bidirectional consistency: if template A lists B as upstream, B should list A as downstream
- Verify no orphaned templates (every template has at least one integration point)

#### 3. AWS API Reference Validation (Manual + Spot Check)
- Spot-check code scaffolding hints against current boto3 API documentation
- Verify Terraform resource names against AWS Provider 5.70+ registry
- Verify CDK construct names against CDK 2.170+ API reference
- Flag any deprecated API calls (e.g., `invoke_model()` where `converse()` is preferred)

#### 4. Content Review (Manual)
- Review each template for completeness against its requirement's acceptance criteria
- Verify role definitions are specific and domain-appropriate
- Verify task sections include concrete file-by-file instructions (not vague prose)
- Verify code scaffolding hints contain working, copy-pasteable code snippets

#### 5. Index Consistency (Automated)
- Verify README.md template map includes all 67 templates
- Verify Library.md directory tree matches actual file system
- Verify PROMPT_GUIDE.md interaction map includes new template connections
- Verify recommended build order incorporates all expansion areas

