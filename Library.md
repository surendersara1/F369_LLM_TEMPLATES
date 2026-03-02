# F369 LLM Templates — Complete Library

```
F369_LLM_TEMPLATES/
├── README.md                           Master index + usage guide
├── PROMPT_GUIDE.md                     How to use templates effectively
├── Library.md                          This file — full library overview
│
├── mlops/                              (14 templates)
│   ├── 00_sagemaker_ai_workspace       **DEPLOY FIRST** SageMaker Domain + Studio + Users + Canvas + JumpStart
│   ├── 01_sagemaker_training_pipeline  ML training pipeline
│   ├── 02_llm_finetuning_pipeline      LoRA/QLoRA fine-tuning
│   ├── 03_llm_inference_deployment     SageMaker Endpoint / Inference Components / ECS+vLLM
│   ├── 04_rag_pipeline                 Bedrock KB / OpenSearch RAG
│   ├── 05_model_monitoring_drift       Model Monitor + alarms
│   ├── 06_experiment_tracking          SageMaker Experiments / Managed MLflow / Self-hosted MLflow
│   ├── 07_feature_store                Online + offline Feature Store
│   ├── 08_sagemaker_pipelines_e2e      Full SageMaker Pipeline
│   ├── 09_bedrock_finetuning           Bedrock custom model fine-tuning
│   ├── 10_model_registry_versioning    Model Registry + approvals
│   ├── 11_ml_governance_responsible_ai Model Cards + Lineage + Clarify Bias + Audit + Compliance
│   ├── 12_bedrock_guardrails_agents    Bedrock Guardrails + Agents + Prompt Mgmt + Eval
│   └── 13_continuous_training_data_ver Continuous Training + Data Versioning + Champion-Challenger
│
├── cicd/                               (5 templates)
│   ├── 01_codebuild_ml_training        buildspec.yml for ML
│   ├── 02_codedeploy_model_appspec     appspec.yml + lifecycle hooks
│   ├── 03_codepipeline_multistage_ml   dev→stage→prod CodePipeline
│   ├── 04_github_actions_aws           OIDC + ECR + SageMaker
│   └── 05_bitbucket_pipelines_aws      CodeStar + Jira integration
│
├── iac/                                (4 templates)
│   ├── 01_terraform_sagemaker_modules  Terraform modules
│   ├── 02_cdk_ml_llm_infrastructure    CDK Python (recommended)
│   ├── 03_terraform_bedrock_opensearch Terraform RAG infra
│   └── 04_cdk_ecs_llm_inference        CDK ECS + vLLM/TGI
│
└── devops/                             (4 templates)
    ├── 01_ecr_ml_docker                Dockerfiles + ECR
    ├── 02_vpc_networking_ml            VPC + PrivateLink
    ├── 03_cloudwatch_monitoring        Dashboards + alarms + Slack
    └── 04_iam_roles_policies_mlops     Least-privilege IAM (build first!)
```

**Total: 29 template files** across 4 categories.

---

## How to Use

1. Open any template `.md` file
2. Fill in your `[PARAMETERS]` at the top
3. Paste the entire template as a prompt to Claude
4. Receive production-ready AWS infrastructure code

See [PROMPT_GUIDE.md](./PROMPT_GUIDE.md) for chaining templates and advanced usage.

---

## Recommended Build Order for New Projects

1. `devops/04` (IAM) → `devops/02` (VPC) → `devops/01` (ECR)
2. `iac/02` (CDK) or `iac/01` (Terraform)
3. `mlops/00` **(Workspace: Domain + Studio + Users + Canvas + JumpStart)**
4. `mlops/07` (Feature Store) → `mlops/01` (Training Pipeline)
5. `mlops/06` (Experiments) → `mlops/10` (Registry) → `mlops/05` (Monitoring)
6. `mlops/02` (Fine-tuning) → `mlops/09` (Bedrock) → `mlops/04` (RAG)
7. `mlops/12` (Bedrock Guardrails + Agents + Prompt Management)
8. `mlops/03` (Inference + Inference Components) or `iac/04` (ECS)
9. `mlops/08` (E2E SageMaker Pipeline)
10. `mlops/11` (Governance + Model Cards + Bias + Compliance)
11. `mlops/13` (Continuous Training + Data Versioning)
12. `cicd/03` (CodePipeline) → `cicd/04` (GitHub) or `cicd/05` (Bitbucket)
