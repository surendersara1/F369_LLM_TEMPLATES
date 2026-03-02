# F369 — AWS MLOps & AI/LLM CI/CD Prompt Template Library

A curated collection of **LLM prompt templates** for generating production-ready AWS infrastructure code. Feed any template to Claude (or another capable LLM) to get fully deployable code for MLOps pipelines, LLM inference, CI/CD, and IaC.

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

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     Source Control                           │
│              GitHub / Bitbucket (templates 04-05)           │
└──────────────────────────┬──────────────────────────────────┘
                           │ push / PR
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                  CI/CD Layer (templates 01-03)               │
│         CodePipeline → CodeBuild → CodeDeploy               │
│              dev  ──►  stage  ──►  prod                     │
└──────────────────────────┬──────────────────────────────────┘
                           │ triggers
                           ▼
┌─────────────────────────────────────────────────────────────┐
│               MLOps Pipeline (templates 08)                  │
│  Processing → Training → Evaluation → Register → Deploy     │
│         ┌─────────────┬──────────────┬──────────┐          │
│         │ Fine-tuning │   RAG Setup  │ Feature  │          │
│         │  (02, 09)   │    (04)      │ Store(07)│          │
│         └─────────────┴──────────────┴──────────┘          │
└──────────────────────────┬──────────────────────────────────┘
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
┌─────────────────────┐   ┌────────────────────────┐
│  Model Registry(10) │   │   Inference (03)        │
│  + Experiments (06) │   │  SageMaker Endpoint OR  │
│  + Monitor (05)     │   │  Inference Components   │
│  + Governance (11)  │   │  OR ECS + vLLM/TGI      │
│  + Cont. Train (13) │   │  + Guardrails (12)      │
└─────────────────────┘   └────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────┐
│               Infrastructure as Code                         │
│          Terraform (01, 03) | CDK Python (02, 04)           │
│  VPC (devops/02) | ECR (devops/01) | IAM (devops/04)        │
└─────────────────────────────────────────────────────────────┘
```

---

## Recommended Build Order for New Projects

1. **DevOps foundation**: `devops/04` (IAM) → `devops/02` (VPC) → `devops/01` (ECR)
2. **IaC skeleton**: `iac/02` (CDK) or `iac/01` (Terraform)
3. **Data Scientist Workspace**: `mlops/00` (Domain + Studio + Users + Canvas + JumpStart)
4. **Data + Features**: `mlops/07` (Feature Store) → `mlops/01` (Training Pipeline)
5. **Model lifecycle**: `mlops/06` (Experiments) → `mlops/10` (Registry) → `mlops/05` (Monitoring)
6. **LLM specifics**: `mlops/02` (Fine-tuning) → `mlops/09` (Bedrock) → `mlops/04` (RAG)
7. **Guardrails**: `mlops/12` (Bedrock Guardrails + Agents + Prompt Management)
8. **Serving**: `mlops/03` (Inference + Inference Components) or `iac/04` (ECS)
9. **Full pipeline**: `mlops/08` (E2E SageMaker Pipeline)
10. **Governance**: `mlops/11` (Model Cards + Lineage + Bias + Audit + Compliance)
11. **Continuous training**: `mlops/13` (Drift-triggered retraining + Data Versioning)
12. **CI/CD**: `cicd/03` (CodePipeline) → `cicd/04` (GitHub) or `cicd/05` (Bitbucket)

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
