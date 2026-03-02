<!-- Template Version: 1.0 | Terraform: 1.9+ | AWS Provider: 5.70+ -->

# Template IaC 01 — Terraform Modules for SageMaker Infrastructure

## Purpose
Generate production-ready Terraform modules for complete SageMaker ML infrastructure: SageMaker Domain/Studio, training infrastructure, real-time endpoints, Model Monitor, Feature Store, S3 buckets, KMS encryption, and IAM roles — parameterized for dev/stage/prod.

---

## Role Definition

You are an expert AWS infrastructure engineer and Terraform specialist with expertise in:
- Terraform 1.9+: modules, workspaces, state management, resource lifecycle
- AWS Provider 5.70+: SageMaker, S3, IAM, KMS, VPC, CloudWatch resources
- Terraform module design: reusable, composable, DRY modules
- Remote state: S3 backend with DynamoDB state locking
- Terraform best practices: tagging, naming conventions, data sources
- Multi-environment deployment via tfvars or workspaces

Generate complete, production-deployable Terraform configurations.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

COMPONENTS:             [OPTIONAL: all | domain | training | endpoints | monitoring | feature-store]
                        all: Complete SageMaker infrastructure
                        domain: SageMaker Domain + Studio only
                        training: Training job infrastructure only
                        endpoints: Inference endpoint infrastructure only

VPC_ID:                 [OPTIONAL - existing VPC ID, or create new]
PRIVATE_SUBNET_IDS:     [OPTIONAL - comma-separated]
ENABLE_STUDIO:          [OPTIONAL: true]

STATE_BUCKET:           [OPTIONAL: {PROJECT_NAME}-{AWS_ACCOUNT_ID}-terraform-state]
STATE_LOCK_TABLE:       [OPTIONAL: {PROJECT_NAME}-terraform-locks]
```

---

## Task

Generate Terraform module structure:

```
terraform/
├── main.tf                        # Root module composition
├── variables.tf                   # All input variables
├── outputs.tf                     # All outputs
├── providers.tf                   # AWS provider + backend config
├── terraform.tfvars               # Default values
├── environments/
│   ├── dev.tfvars
│   ├── stage.tfvars
│   └── prod.tfvars
├── modules/
│   ├── networking/
│   │   ├── main.tf                # VPC, subnets, security groups, VPC endpoints
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── sagemaker-domain/
│   │   ├── main.tf                # SageMaker Domain + Studio + user profiles
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── sagemaker-training/
│   │   ├── main.tf                # S3 buckets, ECR repos, training IAM role
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── sagemaker-endpoints/
│   │   ├── main.tf                # Endpoint configs, auto-scaling, ALB
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── model-monitor/
│   │   ├── main.tf                # Monitoring schedule, S3, CloudWatch alarms
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── feature-store/
│   │   ├── main.tf                # Feature groups, offline store S3, Glue catalog
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── iam/
│   │   ├── main.tf                # All IAM roles and policies
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── kms/
│   │   ├── main.tf                # KMS keys for S3, SageMaker, EBS
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── s3/
│       ├── main.tf                # S3 buckets with lifecycle, versioning, encryption
│       ├── variables.tf
│       └── outputs.tf
└── scripts/
    ├── init.sh                    # terraform init with backend config
    └── apply.sh                   # terraform plan + apply per environment
```

**Root main.tf**: Compose all modules with environment-specific variables. Use `locals` for naming conventions: `{PROJECT_NAME}-{component}-{ENV}`.

**modules/networking**: VPC with private subnets, SageMaker VPC endpoints (sagemaker.api, sagemaker.runtime, s3, ecr.api, ecr.dkr, logs, sts), NAT gateway, security groups.

**modules/sagemaker-domain**: `aws_sagemaker_domain` with VPC mode, default user settings, Jupyter Server/Kernel Gateway apps, lifecycle config.

**modules/sagemaker-training**: S3 bucket for training data/artifacts, ECR repository with lifecycle policy, IAM execution role for training jobs.

**modules/iam**: Separate roles for SageMaker execution, CodeBuild, CodeDeploy, Lambda, with least-privilege policies.

**modules/kms**: Customer-managed KMS keys for S3, SageMaker volumes, EBS. Key policies granting access to SageMaker execution role.

**environments/*.tfvars**: Instance sizes, scaling params, retention periods per environment.

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**State:** S3 backend with DynamoDB locking. Separate state per environment. Never store state locally.

**Security:** KMS encryption everywhere. VPC-only mode for SageMaker. No public subnets for ML resources. IAM least-privilege per module.

**Naming:** `{PROJECT_NAME}-{component}-{ENV}` for all resources. Tags: `Project`, `Environment`, `ManagedBy=terraform`, `CostCenter`.

**Lifecycle:** `prevent_destroy = true` for prod S3 buckets and KMS keys. `create_before_destroy = true` for endpoint configs.

---

## Integration Points

- **Alternative to**: `iac/02` (CDK version of same infrastructure)
- **Upstream**: `devops/02` → can import existing VPC
- **Downstream**: All `mlops/` templates use infrastructure created here
- **Downstream**: `cicd/03` → CodePipeline uses IAM roles created here
