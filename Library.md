# F369 LLM Templates — Complete Library

```
F369_LLM_TEMPLATES/
├── README.md                                     Master index + usage guide
├── PROMPT_GUIDE.md                               How to use templates effectively
├── Library.md                                    This file — full library overview
│
├── mlops/                                        (32 templates — Wave 8 added 7 composite templates)
│   ├── 00_sagemaker_ai_workspace                 **DEPLOY FIRST** SageMaker Domain + Studio + Users + Canvas + JumpStart
│   ├── 01_sagemaker_training_pipeline            ML training pipeline
│   ├── 02_llm_finetuning_pipeline                LoRA/QLoRA fine-tuning
│   ├── 03_llm_inference_deployment               SageMaker Endpoint / Inference Components / ECS+vLLM
│   ├── 04_rag_pipeline                           Bedrock KB / OpenSearch RAG
│   ├── 05_model_monitoring_drift                 Model Monitor + alarms
│   ├── 06_experiment_tracking                    SageMaker Experiments / Managed MLflow / Self-hosted MLflow
│   ├── 07_feature_store                          Online + offline Feature Store
│   ├── 08_sagemaker_pipelines_e2e                Full SageMaker Pipeline
│   ├── 09_bedrock_finetuning                     Bedrock custom model fine-tuning
│   ├── 10_model_registry_versioning              Model Registry + approvals
│   ├── 11_ml_governance_responsible_ai           Model Cards + Lineage + Clarify Bias + Audit + Compliance
│   ├── 12_bedrock_guardrails_agents              Bedrock Guardrails + Agents + Prompt Mgmt + Eval
│   ├── 13_continuous_training_data_ver           Continuous Training + Data Versioning + Champion-Challenger
│   ├── 14_bedrock_agents_action_groups           Bedrock Agents + Action Groups + Multi-Agent Collaboration
│   ├── 15_multimodal_processing_pipelines        Image/Audio/Document pipelines (Claude Vision, Rekognition, Transcribe, Textract)
│   ├── 16_bedrock_flows_orchestration            Bedrock Flows visual DAG orchestration
│   ├── 17_llm_evaluation_pipeline                LLM eval metrics + A/B testing + human-in-the-loop
│   ├── 18_prompt_caching_patterns                Bedrock prompt caching + semantic cache + DynamoDB cache
│   ├── 19_bedrock_marketplace_models             Marketplace model deployment + provisioned throughput + cross-region inference
│   ├── 20_strands_agent_lambda_deployment        Strands Agent Lambda + CDK + MCP tools + conversation management
│   ├── 21_strands_multi_agent_patterns           Graph/Swarm/Workflow multi-agent orchestration
│   ├── 22_strands_agentcore_deployment           AgentCore Runtime + identity + auto-scaling
│   ├── 23_agent_sop_authoring                    Agent SOP markdown authoring with RFC 2119 keywords
│   ├── 24_bedrock_prompt_management              Bedrock Prompt Management API lifecycle + A/B testing
│   ├── 25_react_portal_cloudfront                React portal hosted on CloudFront + OAC
│   ├── 26_fargate_ml_container                   Fargate-based ML container hosting
│   │
│   │  — Wave 8 composite templates (2026-04-26) —
│   ├── 27_fm_training_with_hyperpod              **NEW** FM training (Llama 3 70B/405B) on HyperPod + Distributed + Smart Sifting + Lineage
│   ├── 28_llm_production_serving                 **NEW** Multi-tenant LoRA adapters + sync + async + Inference Recommender
│   ├── 29_unified_studio_workspace               **NEW** DataZone + Studio Spaces + Canvas + MLflow + Lineage workspace
│   ├── 30_enterprise_ml_governance               **NEW** 3-account ML governance (RAM share + drift + lineage + auto-rollback)
│   ├── 31_aws_silicon_cost_optimization          **NEW** Trainium2 training + Inferentia2 inference + Smart Sifting (40-75% cost cut)
│   ├── 32_geospatial_ml                          **NEW** Earth Observation Jobs + pre-built models + custom segmentation
│   └── 33_managed_labeling_pipeline              **NEW** Ground Truth Plus → trigger Lambda → training pipeline
│
├── cicd/                                         (5 templates)
│   ├── 01_codebuild_ml_training                  buildspec.yml for ML
│   ├── 02_codedeploy_model_appspec               appspec.yml + lifecycle hooks
│   ├── 03_codepipeline_multistage_ml             dev→stage→prod CodePipeline
│   ├── 04_github_actions_aws                     OIDC + ECR + SageMaker
│   └── 05_bitbucket_pipelines_aws                CodeStar + Jira integration
│
├── iac/                                          (4 templates)
│   ├── 01_terraform_sagemaker_modules            Terraform modules
│   ├── 02_cdk_ml_llm_infrastructure              CDK Python (recommended)
│   ├── 03_terraform_bedrock_opensearch            Terraform RAG infra
│   └── 04_cdk_ecs_llm_inference                  CDK ECS + vLLM/TGI
│
├── devops/                                       (17 templates — Wave 9 added 1 EKS composite)
│   ├── 01_ecr_ml_docker                          Dockerfiles + ECR
│   ├── 02_vpc_networking_ml                      VPC + PrivateLink
│   ├── 03_cloudwatch_monitoring                  Dashboards + alarms + Slack
│   ├── 04_iam_roles_policies_mlops               Least-privilege IAM (build first!)
│   ├── 05_config_rules_ml_compliance             AWS Config managed + custom rules for ML compliance
│   ├── 06_guardduty_securityhub_ml               GuardDuty + Security Hub threat detection + remediation
│   ├── 07_macie_pii_training_data                Macie PII detection in training data + sanitization
│   ├── 08_kms_encryption_ml                      KMS key management per ML purpose + cross-account grants
│   ├── 09_vpc_endpoint_policies_ml               VPC endpoint policies for SageMaker, Bedrock, S3
│   ├── 10_opentelemetry_ml_tracing               OpenTelemetry + ADOT + X-Ray distributed tracing
│   ├── 11_custom_cloudwatch_model_quality         Custom CloudWatch metrics for model quality + anomaly detection
│   ├── 12_bedrock_invocation_logging             Bedrock invocation logging + Athena queries + Logs Insights
│   ├── 13_cost_per_inference_dashboards          Cost-per-inference tracking + per-customer metering
│   ├── 14_clarify_realtime_bias_monitoring       SageMaker Clarify real-time bias + explainability monitoring
│   ├── 15_strands_agent_observability            Agent OTel tracing + CloudWatch metrics + dashboards
│   ├── 16_agent_guardrails_control               Agent Control + Bedrock Guardrails + consent + defense
│   ├── 17_step_functions_orchestration           Step Functions ML pipeline orchestration
│   │
│   │  — Wave 9 composite template (2026-04-26) —
│   └── 18_eks_production_baseline                **NEW** EKS 1.32 + Karpenter v1 + LBC + Pod Identity + ArgoCD + Container Insights (3-day POC)
│
├── data/                                         (9 templates — Wave 8 added 4 composite templates)
│   ├── 01_glue_etl_ml_features                   Glue ETL PySpark feature engineering + data quality
│   ├── 02_kinesis_realtime_features               Kinesis streaming feature computation + Flink aggregations
│   ├── 03_lake_formation_ml_governance            Lake Formation fine-grained access + LF-TBAC + cross-account sharing
│   ├── 04_s3_data_lifecycle_ml                    S3 Intelligent-Tiering + lifecycle rules + batch operations
│   ├── 05_eventbridge_ml_orchestration            EventBridge ML event bus + SageMaker event rules + drift→retrain
│   │
│   │  — Wave 8 composite templates (2026-04-26) —
│   ├── 06_operational_db_to_lakehouse            **NEW** DMS + EventBridge Pipes + AppFlow → S3 → Iceberg (3 ingest paths)
│   ├── 07_multi_db_federation_query              **NEW** Athena Federated Query (30+ connectors) + Glue Federation + LF
│   ├── 08_resilient_db_dr                         **NEW** RDS Multi-AZ + Aurora Global DR + AWS Backup cross-region
│   └── 09_emr_serverless_spark_iceberg            **NEW** EMR Serverless 7.12 + Spark on Iceberg + Lake Formation
│
├── finops/                                       (6 templates — Wave 9 added 1 EKS composite)
│   ├── 01_cost_allocation_ml                      Cost allocation tags + Budgets + Anomaly Detection
│   ├── 02_spot_instance_strategies_ml             SageMaker managed spot training + checkpointing + fallback
│   ├── 03_savings_plans_ml                        Savings Plans analysis + utilization reports + right-sizing
│   ├── 04_inference_cost_optimization             Auto-scaling + scale-to-zero + batch transform + multi-model endpoints
│   ├── 05_finops_dashboards_ml                    CloudWatch + QuickSight FinOps dashboards + chargeback reports
│   │
│   │  — Wave 9 composite template (2026-04-26) —
│   └── 06_eks_cost_optimization                  **NEW** EKS 40-70% bill cut: Karpenter consolidation + VPA + Spot + Graviton + Kubecost + SP (1-weekend)
│
├── enterprise/                                   (8 templates — Wave 8 + Wave 9 composites)
│   ├── 01_organizations_scps_ml                   SCPs for ML instance types, regions, encryption, Bedrock models
│   ├── 02_cross_account_model_deployment          Cross-account CodePipeline + shared artifacts + model promotion
│   ├── 03_service_catalog_ml                      Service Catalog self-service ML products + launch constraints
│   ├── 04_control_tower_ml                        Control Tower landing zone + ML account baselines + guardrails
│   ├── 05_centralized_model_registry              Centralized cross-account Model Registry + lineage + EventBridge
│   ├── 06_compliance_blueprint                   HIPAA / SOC 2 / PCI compliance baseline
│   │
│   │  — Wave 8 composite template (2026-04-26) —
│   ├── 07_datalake_security_baseline             **NEW** 30-control composite for SOC 2 / HIPAA / GDPR / PCI-DSS + daily audit Lambda
│   │
│   │  — Wave 9 composite template (2026-04-26) —
│   └── 08_eks_security_hardening                 **NEW** EKS regulated-workload posture: 7-layer defense (PSS + NetPol + ECR/Inspector + GuardDuty + Kyverno + signing + IR runbook)
│
└── edge/                                         (3 templates)
    ├── 01_sagemaker_edge_deployment               SageMaker Neo compilation + Edge Manager + IoT Jobs OTA
    ├── 02_iot_greengrass_ml_inference              IoT Greengrass V2 ML components + Stream Manager
    └── 03_outposts_ml_patterns                    Outposts local ML training + inference + DataSync
```

**Total: 78 template files** across 8 categories. (Wave 9 added 3 EKS composite templates.)

---

## How to Use

1. Open any template `.md` file
2. Fill in your `[PARAMETERS]` at the top
3. Paste the entire template as a prompt to Claude
4. Receive production-ready AWS infrastructure code

See [PROMPT_GUIDE.md](./PROMPT_GUIDE.md) for chaining templates and advanced usage.

---

## Recommended Build Order for New Projects

### Phase 1 — Foundation & Security
1. `devops/04` (IAM) → `devops/02` (VPC) → `devops/01` (ECR)
2. `devops/08` (KMS Encryption) → `devops/09` (VPC Endpoints)
3. `devops/05` (Config Rules) → `devops/06` (GuardDuty + Security Hub) → `devops/07` (Macie PII)
4. `iac/02` (CDK) or `iac/01` (Terraform)

### Phase 2 — Data Engineering
5. `data/03` (Lake Formation Governance) → `data/04` (S3 Lifecycle)
6. `data/01` (Glue ETL Features) → `data/02` (Kinesis Real-Time Features)
7. `data/05` (EventBridge ML Orchestration)

### Phase 3 — ML Platform Core
8. `mlops/00` **(Workspace: Domain + Studio + Users + Canvas + JumpStart)**
9. `mlops/07` (Feature Store) → `mlops/01` (Training Pipeline)
10. `mlops/06` (Experiments) → `mlops/10` (Registry) → `mlops/05` (Monitoring)

### Phase 4 — LLM & GenAI
11. `mlops/02` (Fine-tuning) → `mlops/09` (Bedrock Fine-tuning)
12. `mlops/04` (RAG) → `mlops/12` (Bedrock Guardrails)
13. `mlops/14` (Bedrock Agents + Action Groups) → `mlops/16` (Bedrock Flows)
14. `mlops/15` (Multimodal Pipelines) → `mlops/18` (Prompt Caching)
15. `mlops/19` (Marketplace Models) → `mlops/17` (LLM Evaluation)

### Phase 5 — Inference & Deployment
16. `mlops/03` (Inference Deployment) or `iac/04` (ECS)
17. `mlops/08` (E2E SageMaker Pipeline)
18. `mlops/11` (Governance + Model Cards + Bias + Compliance)
19. `mlops/13` (Continuous Training + Data Versioning)

### Phase 6 — Observability
20. `devops/10` (OpenTelemetry Tracing) → `devops/11` (Custom Model Metrics)
21. `devops/12` (Bedrock Invocation Logging) → `devops/13` (Cost-per-Inference Dashboards)
22. `devops/14` (Clarify Real-Time Bias Monitoring)

### Phase 7 — FinOps & Cost Management
23. `finops/01` (Cost Allocation + Tagging) → `finops/02` (Spot Strategies)
24. `finops/03` (Savings Plans) → `finops/04` (Inference Cost Optimization)
25. `finops/05` (FinOps Dashboards)

### Phase 8 — Enterprise & Multi-Account
26. `enterprise/01` (SCPs) → `enterprise/04` (Control Tower)
27. `enterprise/03` (Service Catalog) → `enterprise/02` (Cross-Account Deployment)
28. `enterprise/05` (Centralized Model Registry)

### Phase 9 — Edge & Hybrid (Optional)
29. `edge/01` (SageMaker Edge) → `edge/02` (IoT Greengrass ML)
30. `edge/03` (Outposts ML Patterns)

### Phase 10 — CI/CD Pipelines
31. `cicd/03` (CodePipeline) → `cicd/01` (CodeBuild) → `cicd/02` (CodeDeploy)
32. `cicd/04` (GitHub Actions) or `cicd/05` (Bitbucket Pipelines)

### Phase 11 — Strands Agents
33. `devops/04` (IAM) → `mlops/23` (Agent SOP Authoring) → `mlops/24` (Bedrock Prompt Management)
34. `mlops/20` (Strands Lambda Deployment) or `mlops/22` (AgentCore Deployment)
35. `mlops/21` (Multi-Agent Patterns) → `devops/15` (Agent Observability) → `devops/16` (Agent Guardrails & Control)

### Phase 12 — EKS / Kubernetes platform (Wave 9)
36. `devops/18` (EKS Production Baseline) — 3-day cluster + Karpenter + LBC + Pod Identity + ArgoCD
37. `enterprise/08` (EKS Security Hardening) — 3-day regulated-posture: PSS + NetPol + ECR/Inspector + GuardDuty + Kyverno
38. `finops/06` (EKS Cost Optimization) — 1-weekend 40-70% bill cut: Karpenter + VPA + Spot + Graviton + Kubecost
