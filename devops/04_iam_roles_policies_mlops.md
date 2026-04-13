<!-- Template Version: 1.0 | Terraform/CDK/CloudFormation compatible -->

# Template DevOps 04 — IAM Roles & Policies for MLOps

## Purpose
Generate production-ready least-privilege IAM roles and policies for the entire MLOps stack: SageMaker execution role, CodeBuild service role, CodeDeploy role, CodePipeline role, Lambda execution roles, GitHub Actions OIDC role, Bedrock access role, and cross-account deployment roles.

---

## Role Definition

You are an expert AWS security engineer specializing in IAM for ML workloads with expertise in:
- IAM policy design: least-privilege, condition keys, resource-level permissions
- Service-linked roles and service roles for SageMaker, CodeBuild, CodeDeploy, CodePipeline
- OIDC identity providers for GitHub Actions and Bitbucket Pipelines
- Cross-account IAM roles for multi-account deployment strategies
- IAM Access Analyzer for policy validation
- SCP (Service Control Policies) for organizational guardrails
- Permission boundaries for delegated administration

Generate complete IAM configurations.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

IAC_TOOL:               [REQUIRED - terraform | cdk | cloudformation]

ROLES_TO_CREATE:        [OPTIONAL: all]
                        Options: sagemaker-execution, codebuild, codedeploy, codepipeline,
                                 lambda-execution, github-oidc, bitbucket-oidc, bedrock-access,
                                 cross-account, all

GITHUB_ORG:             [OPTIONAL - for OIDC trust policy: e.g. my-org]
GITHUB_REPO:            [OPTIONAL - for OIDC trust policy: e.g. my-org/ml-project]
BITBUCKET_WORKSPACE:    [OPTIONAL - workspace UUID for Bitbucket OIDC]

CROSS_ACCOUNT_IDS:      [OPTIONAL - comma-separated account IDs for cross-account roles]
S3_BUCKET_ARNS:         [OPTIONAL - specific S3 bucket ARNs to grant access to]
ECR_REPO_ARNS:          [OPTIONAL - specific ECR repo ARNs]
KMS_KEY_ARNS:           [OPTIONAL - specific KMS key ARNs]

PERMISSION_BOUNDARY:    [OPTIONAL - ARN of permission boundary to attach to all roles]
```

---

## Task

Generate all IAM roles and policies:

```
iam/
├── roles/
│   ├── sagemaker_execution_role.tf    # (or .py for CDK)
│   ├── codebuild_role.tf
│   ├── codedeploy_role.tf
│   ├── codepipeline_role.tf
│   ├── lambda_execution_role.tf
│   ├── github_oidc_role.tf
│   ├── bitbucket_oidc_role.tf
│   ├── bedrock_access_role.tf
│   └── cross_account_role.tf
├── policies/
│   ├── sagemaker_policy.json          # SageMaker permissions
│   ├── s3_data_access_policy.json     # S3 read/write for ML data
│   ├── ecr_policy.json                # ECR push/pull
│   ├── cloudwatch_policy.json         # Logs + metrics
│   ├── secrets_policy.json            # Secrets Manager + SSM
│   └── bedrock_policy.json            # Bedrock model access
├── oidc/
│   ├── github_oidc_provider.tf        # OIDC identity provider for GitHub
│   └── bitbucket_oidc_provider.tf     # OIDC identity provider for Bitbucket
├── ssm_outputs.tf                     # Write role ARNs to SSM
├── variables.tf
└── outputs.tf
```

**sagemaker_execution_role**: Trust: `sagemaker.amazonaws.com`. Permissions:
- `sagemaker:*` on project resources, scoped with IAM condition keys:
  - `Condition: { "StringEquals": { "aws:ResourceTag/Project": "${PROJECT_NAME}" } }` for actions on existing resources
  - `Condition: { "StringEquals": { "aws:RequestTag/Project": "${PROJECT_NAME}" } }` for resource creation actions
  - `Condition: { "StringEquals": { "sagemaker:ResourceTag/Project": "${PROJECT_NAME}" } }` for SageMaker-specific tag-based authorization
  - Note: Use `aws:ResourceTag` for global tag conditions and `sagemaker:ResourceTag` for SageMaker service-specific condition keys
- `s3:GetObject/PutObject/ListBucket` on data and artifact buckets
- `ecr:GetAuthorizationToken/BatchGetImage/GetDownloadUrlForLayer` on project ECR repos
- `logs:CreateLogGroup/CreateLogStream/PutLogEvents`
- `kms:Decrypt/GenerateDataKey` on project KMS keys
- `sts:AssumeRole` for cross-account if needed
- Condition: `aws:RequestedRegion` restricted to `AWS_REGION`

**codebuild_role**: Trust: `codebuild.amazonaws.com`. Permissions:
- `sagemaker:CreateTrainingJob/DescribeTrainingJob/CreatePipeline/StartPipelineExecution`
- ECR permissions on project repos (minimum required for Docker build+push):
  - `ecr:GetAuthorizationToken` (on `Resource: *` — required for registry auth)
  - `ecr:BatchCheckLayerAvailability`
  - `ecr:GetDownloadUrlForLayer`
  - `ecr:BatchGetImage`
  - `ecr:PutImage`
  - `ecr:InitiateLayerUpload`
  - `ecr:UploadLayerPart`
  - `ecr:CompleteLayerUpload`
  - Note: Do NOT use `ecr:*` — it includes destructive actions like `ecr:DeleteRepository` and `ecr:DeleteLifecyclePolicy` that CodeBuild should never need
- `s3:GetObject/PutObject` on artifact/cache buckets
- `ssm:GetParameter` for config
- `secretsmanager:GetSecretValue` for tokens
- `logs:*` for CloudWatch

**github_oidc_role**: Trust: OIDC provider `token.actions.githubusercontent.com` with condition:
```json
"Condition": {
  "StringEquals": {
    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
  },
  "StringLike": {
    "token.actions.githubusercontent.com:sub": "repo:{GITHUB_ORG}/{GITHUB_REPO}:*"
  }
}
```

**bedrock_access_role**: Trust: `bedrock.amazonaws.com`. Permissions:
- Inference actions on specific model ARNs:
  - `bedrock:InvokeModel`
  - `bedrock:InvokeModelWithResponseStream`
- Model customization (fine-tuning) actions:
  - `bedrock:CreateModelCustomizationJob`
  - `bedrock:GetModelCustomizationJob`
  - `bedrock:ListModelCustomizationJobs`
  - `bedrock:StopModelCustomizationJob`
  - `bedrock:GetCustomModel`
  - `bedrock:ListCustomModels`
- Provisioned throughput actions:
  - `bedrock:CreateProvisionedModelThroughput`
  - `bedrock:GetProvisionedModelThroughput`
  - `bedrock:ListProvisionedModelThroughputs`
  - `bedrock:UpdateProvisionedModelThroughput`
- Foundation model discovery:
  - `bedrock:GetFoundationModel`
  - `bedrock:ListFoundationModels`
- Note: Do NOT grant `bedrock:*` — it includes `bedrock:DeleteCustomModel`, `bedrock:DeleteProvisionedModelThroughput`, and `bedrock:DeleteGuardrail` which should not be granted to the service role
- `s3:GetObject` on training data, `s3:PutObject` on output path

**cross_account_role**: Trust: specific account IDs. Permissions: scoped to deployment actions (SageMaker endpoint update, ECS service update).

**SSM outputs**: Write all role ARNs to `/mlops/{PROJECT_NAME}/{ENV}/`:
- `sagemaker-role-arn`, `codebuild-role-arn`, `lambda-role-arn`, etc.
- Other templates read from these SSM paths.

---

## Output Format

Output ALL files in chosen IAC_TOOL format with headers: `### FILE: [path]`

---

## Requirements & Constraints

**Least Privilege:** Every policy uses resource-level ARNs (not `*`). Use `aws:RequestTag` and `aws:ResourceTag` conditions to scope to project resources. Use `aws:RequestedRegion` to restrict to target region.

**Boundaries:** Attach permission boundary to all roles if `PERMISSION_BOUNDARY` provided. This prevents privilege escalation.

**Naming:** `{PROJECT_NAME}-{role-purpose}-{ENV}-role` for all roles. Policies: `{PROJECT_NAME}-{purpose}-{ENV}-policy`.

**Audit:** All roles tagged with `Project`, `Environment`, `ManagedBy`. Enable CloudTrail for IAM events. Use IAM Access Analyzer to validate policies.

**OIDC Security:** GitHub OIDC: restrict `sub` claim to specific repo + branch. Bitbucket OIDC: restrict to workspace UUID + repository UUID. Never use `*` in OIDC trust conditions.

---

## Integration Points

- **Downstream**: ALL templates → every template reads role ARNs from SSM
- **Downstream**: `cicd/01-03` → CodeBuild/Deploy/Pipeline roles
- **Downstream**: `cicd/04` → GitHub Actions uses OIDC role
- **Downstream**: `cicd/05` → Bitbucket uses OIDC role
- **Downstream**: `mlops/01-10` → SageMaker execution role
- **Downstream**: `mlops/14` → Bedrock Agent execution roles
- **Downstream**: `mlops/15` → Bedrock, Rekognition, Transcribe, Textract roles
- **Downstream**: `mlops/16` → Bedrock Flows execution roles
- **Downstream**: `mlops/17` → Bedrock evaluation job roles
- **Downstream**: `mlops/18` → Bedrock, ElastiCache, DynamoDB roles
- **Downstream**: `mlops/19` → Bedrock provisioned throughput roles
- **Downstream**: `devops/05` → Config rule Lambda evaluator roles
- **Downstream**: `devops/06` → GuardDuty, Security Hub, remediation Lambda roles
- **Downstream**: `devops/07` → Macie classification job roles
- **Downstream**: `devops/08` → KMS key administration roles
- **Downstream**: `devops/09` → VPC endpoint access policy roles
- **Downstream**: `devops/10` → ADOT collector and X-Ray roles
- **Downstream**: `devops/11` → CloudWatch metrics publishing roles
- **Downstream**: `devops/12` → Bedrock invocation logging roles
- **Downstream**: `devops/13` → CloudWatch and QuickSight roles
- **Downstream**: `devops/14` → SageMaker Clarify monitoring roles
- **Downstream**: `data/01` → Glue job execution roles
- **Downstream**: `data/02` → Kinesis, Lambda, Firehose roles
- **Downstream**: `data/03` → Lake Formation admin and data access roles
- **Downstream**: `data/04` → S3 batch operations roles
- **Downstream**: `data/05` → EventBridge rule target roles
- **Downstream**: `finops/01` → Budgets and Cost Explorer roles
- **Downstream**: `finops/02` → SageMaker spot training roles
- **Downstream**: `finops/03` → Cost Explorer read roles
- **Downstream**: `finops/04` → Auto-scaling and SageMaker roles
- **Downstream**: `finops/05` → CloudWatch and QuickSight roles
- **Downstream**: `enterprise/01` → Organizations policy management roles
- **Downstream**: `enterprise/02` → Cross-account deployment roles
- **Downstream**: `enterprise/03` → Service Catalog launch constraint roles
- **Downstream**: `enterprise/04` → Control Tower customization roles
- **Downstream**: `enterprise/05` → Cross-account model registry roles
- **Downstream**: `edge/01` → Edge Manager and IoT roles
- **Downstream**: `edge/02` → Greengrass component roles
- **Downstream**: `edge/03` → Outposts ML workload roles
- This template should be created FIRST in any new project.
