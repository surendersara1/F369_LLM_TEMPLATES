# F369 — AWS MLOps & AI/LLM CI/CD Prompt Template Library

A curated collection of **67 LLM prompt templates** across **8 directories** for generating production-ready AWS infrastructure code. Feed any template to Claude (or another capable LLM) to get fully deployable code for MLOps pipelines, LLM inference, data engineering, security, FinOps, enterprise governance, observability, edge deployment, CI/CD, and IaC.

---

## Quick Start

1. Open the template that matches your need (see map below)
2. Fill in your `[PARAMETERS]` at the top of the template
3. Paste the entire template as a prompt to Claude
4. Receive deployable AWS infrastructure code

See **[PROMPT_GUIDE.md](./PROMPT_GUIDE.md)** for advanced usage, chaining templates, and tips.

---

## Template Map

### MLOps for AI (Primary)

| # | File | What It Builds |
|---|------|----------------|
| 00 | [mlops/00_sagemaker_ai_workspace.md](./mlops/00_sagemaker_ai_workspace.md) | **DEPLOY FIRST** — SageMaker Domain, Studio, Users, Canvas, JumpStart, HyperPod, notebook-to-pipeline bridge |
| 01 | [mlops/01_sagemaker_training_pipeline.md](./mlops/01_sagemaker_training_pipeline.md) | SageMaker ML training pipeline (Processing → Training → HPO → Evaluation) |
| 02 | [mlops/02_llm_finetuning_pipeline.md](./mlops/02_llm_finetuning_pipeline.md) | LLM fine-tuning with LoRA/QLoRA on SageMaker (Llama, Mistral, etc.) |
| 03 | [mlops/03_llm_inference_deployment.md](./mlops/03_llm_inference_deployment.md) | LLM inference: SageMaker Endpoint or ECS Fargate with vLLM/TGI |
| 04 | [mlops/04_rag_pipeline.md](./mlops/04_rag_pipeline.md) | Full RAG pipeline: ingest → embed → index → retrieve → generate |
| 05 | [mlops/05_model_monitoring_drift.md](./mlops/05_model_monitoring_drift.md) | SageMaker Model Monitor + CloudWatch alarms + auto-retrain trigger |
| 06 | [mlops/06_experiment_tracking.md](./mlops/06_experiment_tracking.md) | Experiment tracking: SageMaker Experiments or MLflow on AWS |
| 07 | [mlops/07_feature_store.md](./mlops/07_feature_store.md) | SageMaker Feature Store (online + offline) + ingestion pipeline |
| 08 | [mlops/08_sagemaker_pipelines_e2e.md](./mlops/08_sagemaker_pipelines_e2e.md) | End-to-end SageMaker Pipeline (all steps orchestrated) |
| 09 | [mlops/09_bedrock_finetuning.md](./mlops/09_bedrock_finetuning.md) | Bedrock custom model fine-tuning + provisioned throughput deployment |
| 10 | [mlops/10_model_registry_versioning.md](./mlops/10_model_registry_versioning.md) | SageMaker Model Registry + approval workflows + lineage |
| 11 | [mlops/11_ml_governance_responsible_ai.md](./mlops/11_ml_governance_responsible_ai.md) | Model Cards, lineage tracking, Clarify bias detection, audit trails, compliance |
| 12 | [mlops/12_bedrock_guardrails_agents.md](./mlops/12_bedrock_guardrails_agents.md) | Bedrock Guardrails, Agents, Prompt Management, Model Evaluation |
| 13 | [mlops/13_continuous_training_data_versioning.md](./mlops/13_continuous_training_data_versioning.md) | Continuous training, drift-triggered retraining, data versioning, champion-challenger |
| 14 | [mlops/14_bedrock_agents_action_groups.md](./mlops/14_bedrock_agents_action_groups.md) | Bedrock Agents with Action Groups, OpenAPI schemas, multi-agent collaboration |
| 15 | [mlops/15_multimodal_processing_pipelines.md](./mlops/15_multimodal_processing_pipelines.md) | Multimodal pipelines: image (Claude vision + Rekognition), audio (Transcribe), document (Textract) |
| 16 | [mlops/16_bedrock_flows_orchestration.md](./mlops/16_bedrock_flows_orchestration.md) | Bedrock Flows visual orchestration: prompt chains, RAG flows, conditional branching |
| 17 | [mlops/17_llm_evaluation_pipeline.md](./mlops/17_llm_evaluation_pipeline.md) | LLM evaluation: ROUGE/BLEU/BERTScore, Bedrock Model Evaluation, human-in-the-loop, A/B testing |
| 18 | [mlops/18_prompt_caching_patterns.md](./mlops/18_prompt_caching_patterns.md) | Prompt caching: Bedrock native caching, ElastiCache semantic cache, DynamoDB exact-match cache |
| 19 | [mlops/19_bedrock_marketplace_models.md](./mlops/19_bedrock_marketplace_models.md) | Bedrock Marketplace models, provisioned throughput, cross-region inference profiles |

### CI/CD Pipelines

| # | File | What It Builds |
|---|------|----------------|
| 01 | [cicd/01_codebuild_ml_training.md](./cicd/01_codebuild_ml_training.md) | CodeBuild `buildspec.yml` for ML training jobs |
| 02 | [cicd/02_codedeploy_model_appspec.md](./cicd/02_codedeploy_model_appspec.md) | CodeDeploy `appspec.yml` for model endpoint deployments |
| 03 | [cicd/03_codepipeline_multistage_ml.md](./cicd/03_codepipeline_multistage_ml.md) | Multi-stage CodePipeline (dev → stage → prod) for ML |
| 04 | [cicd/04_github_actions_aws_integration.md](./cicd/04_github_actions_aws_integration.md) | GitHub Actions → OIDC → ECR → CodePipeline → SageMaker |
| 05 | [cicd/05_bitbucket_pipelines_aws.md](./cicd/05_bitbucket_pipelines_aws.md) | Bitbucket Pipelines → CodeStar → ECR → CodePipeline |

### Infrastructure as Code

| # | File | What It Builds |
|---|------|----------------|
| 01 | [iac/01_terraform_sagemaker_modules.md](./iac/01_terraform_sagemaker_modules.md) | Terraform modules for full SageMaker ML infrastructure |
| 02 | [iac/02_cdk_ml_llm_infrastructure.md](./iac/02_cdk_ml_llm_infrastructure.md) | CDK Python stack for ML/LLM infrastructure (recommended) |
| 03 | [iac/03_terraform_bedrock_opensearch_rag.md](./iac/03_terraform_bedrock_opensearch_rag.md) | Terraform for Bedrock + OpenSearch Serverless RAG |
| 04 | [iac/04_cdk_ecs_llm_inference.md](./iac/04_cdk_ecs_llm_inference.md) | CDK Python: ECS Fargate + ALB + vLLM/TGI + auto-scaling |

### DevOps Supporting

| # | File | What It Builds |
|---|------|----------------|
| 01 | [devops/01_ecr_ml_docker.md](./devops/01_ecr_ml_docker.md) | Multi-stage Dockerfiles for ML + ECR lifecycle policies |
| 02 | [devops/02_vpc_networking_ml.md](./devops/02_vpc_networking_ml.md) | VPC with private subnets + PrivateLink for ML workloads |
| 03 | [devops/03_cloudwatch_monitoring_alerting.md](./devops/03_cloudwatch_monitoring_alerting.md) | CloudWatch dashboards + alarms + SNS + Lambda remediation |
| 04 | [devops/04_iam_roles_policies_mlops.md](./devops/04_iam_roles_policies_mlops.md) | Least-privilege IAM roles for SageMaker, CodeBuild, CodeDeploy |
| 05 | [devops/05_config_rules_ml_compliance.md](./devops/05_config_rules_ml_compliance.md) | AWS Config rules for ML compliance: managed rules, custom evaluators, conformance packs |
| 06 | [devops/06_guardduty_securityhub_ml.md](./devops/06_guardduty_securityhub_ml.md) | GuardDuty + Security Hub for ML: threat detection, automation rules, remediation workflows |
| 07 | [devops/07_macie_pii_training_data.md](./devops/07_macie_pii_training_data.md) | Macie PII detection in training data: classification jobs, custom identifiers, sanitization |
| 08 | [devops/08_kms_encryption_ml.md](./devops/08_kms_encryption_ml.md) | KMS encryption for ML: per-purpose keys, rotation, cross-account grants, ViaService policies |
| 09 | [devops/09_vpc_endpoint_policies_ml.md](./devops/09_vpc_endpoint_policies_ml.md) | VPC endpoint policies for ML services: SageMaker, Bedrock, S3 gateway, resource-scoped policies |
| 10 | [devops/10_opentelemetry_ml_tracing.md](./devops/10_opentelemetry_ml_tracing.md) | OpenTelemetry + X-Ray distributed tracing for ML inference pipelines |
| 11 | [devops/11_custom_cloudwatch_model_quality.md](./devops/11_custom_cloudwatch_model_quality.md) | Custom CloudWatch metrics for model quality: latency, accuracy, drift scores, anomaly detection |
| 12 | [devops/12_bedrock_invocation_logging.md](./devops/12_bedrock_invocation_logging.md) | Bedrock invocation logging: CloudWatch Logs + S3 + Athena queries for usage analysis |
| 13 | [devops/13_cost_per_inference_dashboards.md](./devops/13_cost_per_inference_dashboards.md) | Cost-per-inference dashboards: metric math, per-customer tracking, QuickSight analysis |
| 14 | [devops/14_clarify_realtime_bias_monitoring.md](./devops/14_clarify_realtime_bias_monitoring.md) | SageMaker Clarify real-time bias monitoring: explainability, bias drift, scheduled checks |

### Data Engineering

| # | File | What It Builds |
|---|------|----------------|
| 01 | [data/01_glue_etl_ml_features.md](./data/01_glue_etl_ml_features.md) | Glue ETL for ML feature engineering: PySpark transforms, data quality, crawlers, incremental processing |
| 02 | [data/02_kinesis_realtime_features.md](./data/02_kinesis_realtime_features.md) | Kinesis real-time feature computation: streaming aggregations, Firehose, Flink, Feature Store integration |
| 03 | [data/03_lake_formation_ml_governance.md](./data/03_lake_formation_ml_governance.md) | Lake Formation ML governance: LF-TBAC, column/row-level permissions, cross-account data sharing |
| 04 | [data/04_s3_data_lifecycle_ml.md](./data/04_s3_data_lifecycle_ml.md) | S3 data lifecycle for ML: Intelligent-Tiering, lifecycle rules, Batch Operations, Access Points |
| 05 | [data/05_eventbridge_ml_orchestration.md](./data/05_eventbridge_ml_orchestration.md) | EventBridge ML orchestration: SageMaker event rules, custom event bus, drift→retrain workflows |

### Cost Management / FinOps

| # | File | What It Builds |
|---|------|----------------|
| 01 | [finops/01_cost_allocation_ml.md](./finops/01_cost_allocation_ml.md) | ML cost allocation: tag policies, Budgets, Cost Explorer queries, Anomaly Detection |
| 02 | [finops/02_spot_instance_strategies_ml.md](./finops/02_spot_instance_strategies_ml.md) | Spot instance strategies for ML training: managed spot, checkpointing, fallback patterns |
| 03 | [finops/03_savings_plans_ml.md](./finops/03_savings_plans_ml.md) | Savings Plans analysis for ML: utilization reports, purchase recommendations, right-sizing |
| 04 | [finops/04_inference_cost_optimization.md](./finops/04_inference_cost_optimization.md) | Inference cost optimization: auto-scaling, scale-to-zero, batch transform, multi-model endpoints |
| 05 | [finops/05_finops_dashboards_ml.md](./finops/05_finops_dashboards_ml.md) | FinOps dashboards: CloudWatch ML spend widgets, QuickSight drill-down, chargeback reports |

### Multi-Account / Enterprise

| # | File | What It Builds |
|---|------|----------------|
| 01 | [enterprise/01_organizations_scps_ml.md](./enterprise/01_organizations_scps_ml.md) | Organizations SCPs for ML: instance type restrictions, encryption enforcement, region/model guardrails |
| 02 | [enterprise/02_cross_account_model_deployment.md](./enterprise/02_cross_account_model_deployment.md) | Cross-account model deployment: CodePipeline spanning dev→stage→prod, shared artifacts, Model Registry |
| 03 | [enterprise/03_service_catalog_ml.md](./enterprise/03_service_catalog_ml.md) | Service Catalog ML products: self-service SageMaker Domain, training env, inference endpoint |
| 04 | [enterprise/04_control_tower_ml.md](./enterprise/04_control_tower_ml.md) | Control Tower for ML: landing zone, account baselines, detective/preventive guardrails |
| 05 | [enterprise/05_centralized_model_registry.md](./enterprise/05_centralized_model_registry.md) | Centralized model registry: cross-account Model Registry, EventBridge sync, lineage metadata |

### Edge / Hybrid

| # | File | What It Builds |
|---|------|----------------|
| 01 | [edge/01_sagemaker_edge_deployment.md](./edge/01_sagemaker_edge_deployment.md) | SageMaker Edge Manager: Neo compilation, device fleets, edge packaging, OTA updates via IoT Jobs |
| 02 | [edge/02_iot_greengrass_ml_inference.md](./edge/02_iot_greengrass_ml_inference.md) | IoT Greengrass ML inference: component recipes, Stream Manager, model optimization |
| 03 | [edge/03_outposts_ml_patterns.md](./edge/03_outposts_ml_patterns.md) | Outposts hybrid ML: local training/inference, S3 Outposts, DataSync, offline continuity |

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                          Enterprise Governance                                │
│   Organizations SCPs (enterprise/01) | Control Tower (04) | Service Cat (03) │
│   Cross-Account Deploy (02) | Centralized Model Registry (05)               │
└──────────────────────────────┬───────────────────────────────────────────────┘
                               │ guardrails
                               ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                          Source Control                                       │
│                   GitHub / Bitbucket (cicd/04-05)                            │
└──────────────────────────────┬───────────────────────────────────────────────┘
                               │ push / PR
                               ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                       CI/CD Layer (cicd/01-03)                                │
│              CodePipeline → CodeBuild → CodeDeploy                           │
│                   dev  ──►  stage  ──►  prod                                │
└──────────────────────────────┬───────────────────────────────────────────────┘
                               │ triggers
                               ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  Data Engineering                  │           MLOps Pipeline (mlops/08)      │
│  Glue ETL (data/01)               │  Processing → Training → Eval → Deploy  │
│  Kinesis Streaming (data/02)      │  ┌──────────┬──────────┬──────────┐     │
│  Lake Formation (data/03)         │  │Fine-tune │ RAG (04) │ Feature  │     │
│  S3 Lifecycle (data/04)           │  │(02, 09)  │ Agents   │ Store(07)│     │
│  EventBridge Orch (data/05)       │  │          │(14,16)   │          │     │
│                                    │  └──────────┴──────────┴──────────┘     │
└────────────────────────────────────┴─────────────────────────────────────────┘
                               │
              ┌────────────────┴────────────────┐
              ▼                                  ▼
┌──────────────────────────┐   ┌─────────────────────────────────┐
│  Model Registry (10)     │   │   Inference (03)                 │
│  + Experiments (06)      │   │  SageMaker Endpoint OR ECS+vLLM │
│  + Monitor (05)          │   │  + Guardrails (12)               │
│  + Governance (11)       │   │  + Prompt Caching (18)           │
│  + Cont. Train (13)      │   │  + Marketplace Models (19)       │
│  + LLM Evaluation (17)  │   │  + Multimodal (15)               │
└──────────────────────────┘   └─────────────────────────────────┘
              │                                  │
              ▼                                  ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                        Observability Layer                                    │
│  OpenTelemetry (devops/10) | Custom Metrics (11) | Bedrock Logging (12)     │
│  Cost-per-Inference (13) | Clarify Bias Monitoring (14)                     │
└──────────────────────────────────────────────────────────────────────────────┘
              │
              ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                     Security & Compliance Layer                               │
│  Config Rules (devops/05) | GuardDuty+SecHub (06) | Macie PII (07)          │
│  KMS Encryption (08) | VPC Endpoint Policies (09)                           │
└──────────────────────────────────────────────────────────────────────────────┘
              │
              ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                     Cost Management / FinOps Layer                            │
│  Cost Allocation (finops/01) | Spot Strategies (02) | Savings Plans (03)    │
│  Inference Cost Opt (04) | FinOps Dashboards (05)                           │
└──────────────────────────────────────────────────────────────────────────────┘
              │
              ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                     Infrastructure as Code                                    │
│            Terraform (iac/01, 03) | CDK Python (iac/02, 04)                 │
│    VPC (devops/02) | ECR (devops/01) | IAM (devops/04)                      │
└──────────────────────────────┬───────────────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                        Edge / Hybrid Layer                                    │
│  SageMaker Edge (edge/01) | IoT Greengrass ML (02) | Outposts (03)         │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## Recommended Build Order for New Projects

1. **Enterprise governance**: `enterprise/01` (SCPs) → `enterprise/04` (Control Tower) → `enterprise/03` (Service Catalog)
2. **DevOps foundation**: `devops/04` (IAM) → `devops/02` (VPC) → `devops/01` (ECR)
3. **Security baseline**: `devops/08` (KMS) → `devops/09` (VPC Endpoints) → `devops/05` (Config Rules)
4. **IaC skeleton**: `iac/02` (CDK) or `iac/01` (Terraform)
5. **Data Scientist Workspace**: `mlops/00` (Domain + Studio + Users + Canvas + JumpStart)
6. **Data engineering**: `data/03` (Lake Formation) → `data/04` (S3 Lifecycle) → `data/01` (Glue ETL) → `data/02` (Kinesis Streaming)
7. **Data + Features**: `mlops/07` (Feature Store) → `mlops/01` (Training Pipeline)
8. **Model lifecycle**: `mlops/06` (Experiments) → `mlops/10` (Registry) → `mlops/05` (Monitoring)
9. **LLM specifics**: `mlops/02` (Fine-tuning) → `mlops/09` (Bedrock) → `mlops/04` (RAG)
10. **Agentic AI**: `mlops/12` (Guardrails) → `mlops/14` (Bedrock Agents) → `mlops/16` (Bedrock Flows)
11. **Multimodal + Marketplace**: `mlops/15` (Multimodal Pipelines) → `mlops/19` (Marketplace Models)
12. **Serving**: `mlops/03` (Inference) or `iac/04` (ECS) → `mlops/18` (Prompt Caching)
13. **Evaluation + Governance**: `mlops/17` (LLM Evaluation) → `mlops/11` (Governance + Responsible AI)
14. **Continuous training**: `mlops/13` (Drift-triggered retraining) → `data/05` (EventBridge ML Orchestration)
15. **Cross-account deployment**: `enterprise/02` (Cross-Account) → `enterprise/05` (Centralized Registry)
16. **Observability**: `devops/10` (OpenTelemetry) → `devops/11` (Custom Metrics) → `devops/12` (Bedrock Logging) → `devops/14` (Clarify Bias)
17. **Security operations**: `devops/06` (GuardDuty + Security Hub) → `devops/07` (Macie PII)
18. **FinOps**: `finops/01` (Cost Allocation) → `finops/02` (Spot) → `finops/03` (Savings Plans) → `finops/04` (Inference Cost Opt) → `finops/05` (Dashboards) → `devops/13` (Cost-per-Inference)
19. **CI/CD**: `cicd/03` (CodePipeline) → `cicd/04` (GitHub) or `cicd/05` (Bitbucket)
20. **Edge / Hybrid** (if needed): `edge/01` (SageMaker Edge) → `edge/02` (IoT Greengrass) → `edge/03` (Outposts)

---

## Environment Conventions Used in Templates

| Variable | Values | Description |
|----------|--------|-------------|
| `ENV` | `dev` / `stage` / `prod` | Deployment environment |
| `AWS_REGION` | e.g. `us-east-1` | Target AWS region |
| `PROJECT_NAME` | e.g. `myai-app` | Used as prefix for all resources |
| `AWS_ACCOUNT_ID` | 12-digit account ID | For ARN construction |
| `MODEL_NAME` | e.g. `llama-3-8b` | The ML/LLM model identifier |

---

## License

MIT — use freely in your projects.
