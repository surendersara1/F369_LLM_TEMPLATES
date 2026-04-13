<!-- Template Version: 1.0 | boto3: 1.35+ | terraform-aws: 5.70+ | cdk: 2.170+ -->

# Template Enterprise 02 — Cross-Account ML Model Deployment Pipeline

## Purpose
Generate a production-ready cross-account ML model deployment pipeline spanning development, staging, and production AWS accounts: a CodePipeline in a dedicated pipeline account that orchestrates model promotion across accounts using cross-account IAM assume-role chains, a shared S3 artifact bucket with KMS encryption and cross-account access policies, SageMaker Model Registry cross-account resource policies for model package sharing, a model promotion workflow that registers models in dev, approves in staging, and deploys endpoints in production, EventBridge rules that trigger pipeline stages on model approval status changes, and rollback procedures that revert to the previous model version on deployment failure.

---

## Role Definition

You are an expert AWS cloud architect and MLOps engineer specializing in multi-account ML deployment pipelines with expertise in:
- AWS CodePipeline: cross-account pipeline actions, artifact stores, action role ARNs, manual approval stages, Lambda invoke actions
- Cross-account IAM: trust policies with `sts:AssumeRole`, `sts:ExternalId` conditions, role chaining across 3+ accounts, least-privilege scoping per pipeline stage
- SageMaker Model Registry: model package groups, model package versions, approval status transitions (`PendingManualApproval` → `Approved` → `Rejected`), cross-account resource policies via `sagemaker.put_model_package_group_policy()`
- S3 cross-account access: bucket policies granting access to multiple account principals, KMS key policies with `kms:ViaService` and cross-account grants for artifact encryption/decryption
- KMS cross-account key sharing: key policies granting `kms:Decrypt`, `kms:GenerateDataKey` to pipeline and target account roles, key grants for SageMaker service principals
- EventBridge cross-account patterns: event bus policies for cross-account `PutEvents`, rules matching `SageMaker Model Package State Change` events, target account event forwarding
- SageMaker endpoint deployment: `create_model()`, `create_endpoint_config()`, `create_endpoint()`, blue/green deployment with production variants, rollback via `update_endpoint()` to previous config
- CloudFormation StackSets and cross-account stack deployments for infrastructure provisioning in target accounts
- Model lineage and artifact tracing across account boundaries using SageMaker Lineage Tracking
- Pipeline security: artifact encryption in transit and at rest, VPC-scoped pipeline actions, CloudTrail audit logging across all accounts

Generate complete, production-deployable cross-account ML deployment pipeline code.

---

## Context & Inputs

```
PROJECT_NAME:               [REQUIRED]
AWS_REGION:                 [REQUIRED]
AWS_ACCOUNT_ID:             [REQUIRED]
ENV:                        [REQUIRED - dev | stage | prod]

DEV_ACCOUNT_ID:             [REQUIRED - AWS account ID for the development environment]
                            Example: "111111111111"
                            This account hosts SageMaker training jobs, experiment tracking,
                            and initial model registration in the Model Registry.

STAGING_ACCOUNT_ID:         [REQUIRED - AWS account ID for the staging environment]
                            Example: "222222222222"
                            Models are deployed here for integration testing and manual
                            approval before production promotion.

PROD_ACCOUNT_ID:            [REQUIRED - AWS account ID for the production environment]
                            Example: "333333333333"
                            Final deployment target for approved models. Endpoints serve
                            live inference traffic.

PIPELINE_ACCOUNT_ID:        [REQUIRED - AWS account ID hosting the CodePipeline]
                            Example: "444444444444"
                            Dedicated tooling/CI-CD account that owns the CodePipeline,
                            shared artifact bucket, and orchestration resources. Can be
                            the same as DEV_ACCOUNT_ID for simpler setups.

SHARED_ARTIFACT_BUCKET:     [REQUIRED - S3 bucket name for cross-account pipeline artifacts]
                            Example: "myai-ml-pipeline-artifacts"
                            Bucket lives in PIPELINE_ACCOUNT_ID. Stores model artifacts,
                            CodePipeline artifacts, and deployment configs. Must have a
                            cross-account bucket policy granting access to all four accounts.

KMS_KEY_ARN:                [REQUIRED - KMS key ARN for encrypting pipeline artifacts]
                            Example: "arn:aws:kms:us-east-1:444444444444:key/abc123-..."
                            Key must reside in PIPELINE_ACCOUNT_ID with cross-account
                            grants for dev, staging, and prod account roles. Used for
                            S3 artifact encryption and SageMaker model artifact encryption.

MODEL_PACKAGE_GROUP:        [REQUIRED - SageMaker Model Package Group name]
                            Example: "customer-churn-models"
                            Model package group in the dev account where trained models
                            are registered. Cross-account resource policy allows staging
                            and prod accounts to describe and deploy model packages.

DEPLOY_INSTANCE_TYPE:       [OPTIONAL: ml.m5.xlarge]
                            SageMaker endpoint instance type for staging and production
                            deployments. Can differ per stage using stage-specific overrides.

DEPLOY_INSTANCE_COUNT:      [OPTIONAL: 1]
                            Number of instances per endpoint. Production typically uses
                            2+ for high availability across AZs.

APPROVAL_NOTIFICATION_EMAIL:[OPTIONAL: none]
                            Email address for SNS notifications when a model requires
                            manual approval in the staging stage.

ENABLE_DATA_CAPTURE:        [OPTIONAL: true]
                            Enable SageMaker Model Monitor data capture on production
                            endpoints for drift detection.

ROLLBACK_ON_ALARM:          [OPTIONAL: true]
                            Automatically rollback to the previous endpoint config when
                            CloudWatch alarms fire on the new deployment.

IAC_TOOL:                   [REQUIRED - boto3 | terraform | cdk]
```

---

## Task

Generate complete cross-account ML model deployment pipeline infrastructure:

```
{PROJECT_NAME}-cross-account-ml-deploy/
├── iam/
│   ├── pipeline_account_roles.py           # IAM roles in the pipeline account
│   ├── dev_account_roles.py                # Cross-account role in dev account
│   ├── staging_account_roles.py            # Cross-account role in staging account
│   ├── prod_account_roles.py               # Cross-account role in prod account
│   └── trust_policies.py                   # Trust policy document builders
├── artifacts/
│   ├── shared_bucket.py                    # S3 artifact bucket with cross-account policy
│   └── kms_key_policy.py                   # KMS key policy for cross-account access
├── registry/
│   ├── model_package_group.py              # Model package group + cross-account policy
│   ├── register_model.py                   # Register a trained model version
│   └── approve_model.py                    # Approve/reject model package versions
├── pipeline/
│   ├── codepipeline_definition.py          # CodePipeline with cross-account stages
│   ├── source_stage.py                     # S3 source trigger on model registration
│   ├── staging_deploy_stage.py             # Deploy to staging + run integration tests
│   ├── approval_stage.py                   # Manual approval gate
│   └── prod_deploy_stage.py               # Deploy to production
├── deploy/
│   ├── deploy_model.py                     # SageMaker model + endpoint deployment
│   ├── rollback.py                         # Rollback to previous endpoint config
│   └── endpoint_health_check.py            # Post-deployment health validation
├── events/
│   ├── eventbridge_rules.py                # EventBridge rules for model approval events
│   └── cross_account_event_bus.py          # Cross-account event forwarding
├── infrastructure/
│   └── config.py                           # Central configuration
├── run_setup.py                            # CLI orchestrator
└── requirements.txt
```

**config.py**: Central configuration dataclass with all parameters. Load from environment variables or CLI args. Validate required fields. Construct resource names using `{PROJECT_NAME}-{component}-{ENV}` convention. Validate AWS account ID format (12 digits). Validate KMS key ARN format. Parse optional parameters with defaults. Store account ID mapping: `{"dev": DEV_ACCOUNT_ID, "staging": STAGING_ACCOUNT_ID, "prod": PROD_ACCOUNT_ID, "pipeline": PIPELINE_ACCOUNT_ID}`.

**trust_policies.py**: Build IAM trust policy documents for cross-account role assumption:
- `get_pipeline_trust_policy(pipeline_account_id)`: Returns trust policy allowing the pipeline account's CodePipeline service role to assume roles in target accounts. Principal is `arn:aws:iam::{pipeline_account_id}:root` with `sts:AssumeRole` action and optional `sts:ExternalId` condition.
- `get_sagemaker_trust_policy(account_id)`: Returns trust policy allowing the SageMaker service principal (`sagemaker.amazonaws.com`) to assume the execution role in the target account.
- `get_eventbridge_trust_policy(source_account_id)`: Returns trust policy allowing EventBridge in the source account to invoke targets in the target account.

**pipeline_account_roles.py**: Create IAM roles in the pipeline account:
- Pipeline service role: `{PROJECT_NAME}-pipeline-service-role-{ENV}` — allows CodePipeline to assume cross-account roles, access the shared S3 bucket, use the KMS key, and invoke Lambda functions. Attach managed policy `AWSCodePipelineFullAccess` scoped down with inline policy.
- Pipeline action role: `{PROJECT_NAME}-pipeline-action-role-{ENV}` — assumed by CodePipeline for S3 source and deploy actions. Grants `s3:GetObject`, `s3:PutObject` on the artifact bucket and `kms:Decrypt`, `kms:GenerateDataKey` on the KMS key.

**dev_account_roles.py**: Create cross-account role in the dev account:
- Role: `{PROJECT_NAME}-pipeline-dev-role-{ENV}` — trusted by PIPELINE_ACCOUNT_ID. Grants `sagemaker:DescribeModelPackage`, `sagemaker:ListModelPackages`, `sagemaker:DescribeModelPackageGroup` on the model package group ARN. Grants `s3:GetObject` on the shared artifact bucket for reading model artifacts. Grants `kms:Decrypt` on the KMS key.

**staging_account_roles.py**: Create cross-account role in the staging account:
- SageMaker execution role: `{PROJECT_NAME}-sagemaker-exec-staging-{ENV}` — trusted by `sagemaker.amazonaws.com`. Grants `s3:GetObject` on the shared artifact bucket (model artifacts), `kms:Decrypt` on the KMS key, `ecr:GetDownloadUrlForLayer` for inference container images, `cloudwatch:PutMetricData` for endpoint metrics.
- Pipeline cross-account role: `{PROJECT_NAME}-pipeline-staging-role-{ENV}` — trusted by PIPELINE_ACCOUNT_ID. Grants `sagemaker:CreateModel`, `sagemaker:CreateEndpointConfig`, `sagemaker:CreateEndpoint`, `sagemaker:UpdateEndpoint`, `sagemaker:DescribeEndpoint`, `sagemaker:DeleteEndpoint`, `sagemaker:DescribeModelPackage`. Grants `iam:PassRole` for the SageMaker execution role.

**prod_account_roles.py**: Create cross-account role in the production account:
- SageMaker execution role: `{PROJECT_NAME}-sagemaker-exec-prod-{ENV}` — same structure as staging but in the prod account. Additional permissions for `sagemaker:InvokeEndpoint` data capture if ENABLE_DATA_CAPTURE is true.
- Pipeline cross-account role: `{PROJECT_NAME}-pipeline-prod-role-{ENV}` — trusted by PIPELINE_ACCOUNT_ID. Same SageMaker deployment permissions as staging. Additional `cloudwatch:DescribeAlarms` for rollback alarm checks.

**shared_bucket.py**: Create and configure the shared S3 artifact bucket in the pipeline account:
- Bucket name: `{SHARED_ARTIFACT_BUCKET}` (user-provided, must be globally unique)
- Enable versioning for artifact rollback
- Enable server-side encryption with the KMS key (SSE-KMS)
- Bucket policy granting `s3:GetObject`, `s3:PutObject`, `s3:GetBucketLocation`, `s3:ListBucket` to IAM roles in all four accounts (dev, staging, prod, pipeline)
- Block public access: all four block public access settings enabled
- Lifecycle rule: expire incomplete multipart uploads after 7 days, transition old artifacts to Glacier after 90 days

**kms_key_policy.py**: Configure the KMS key policy for cross-account access:
- Key administrators: pipeline account root and designated admin roles
- Key users: pipeline service role, dev account cross-account role, staging account cross-account role and SageMaker execution role, prod account cross-account role and SageMaker execution role
- Grant `kms:Decrypt`, `kms:DescribeKey`, `kms:GenerateDataKey`, `kms:ReEncryptFrom`, `kms:ReEncryptTo` to all key user principals
- Add `kms:ViaService` condition restricting key usage to `s3.{AWS_REGION}.amazonaws.com` and `sagemaker.{AWS_REGION}.amazonaws.com`
- Use `kms.put_key_policy()` to update the existing key (referenced by KMS_KEY_ARN)

**model_package_group.py**: Create model package group with cross-account resource policy:
- Create model package group: `sagemaker.create_model_package_group()` with group name `{MODEL_PACKAGE_GROUP}` in the dev account
- Set cross-account resource policy using `sagemaker.put_model_package_group_policy()` granting `sagemaker:DescribeModelPackageGroup`, `sagemaker:DescribeModelPackage`, `sagemaker:ListModelPackages`, `sagemaker:UpdateModelPackage` to staging and prod account root principals
- Tag with Project, Environment, and ManagedBy tags

**register_model.py**: Register a trained model version in the Model Registry:
- Call `sagemaker.create_model_package()` with `ModelPackageGroupName`, `InferenceSpecification` (container image, supported content types, instance types), `ModelApprovalStatus='PendingManualApproval'`, `ModelMetrics` (quality, bias, explainability), and `CustomerMetadataProperties` for custom tags
- Store model artifact S3 URI pointing to the shared artifact bucket
- Print model package ARN for downstream pipeline consumption

**approve_model.py**: Approve or reject model package versions:
- Call `sagemaker.update_model_package()` with `ModelPackageArn` and `ModelApprovalStatus='Approved'` or `'Rejected'`
- Add `ApprovalDescription` with reviewer name and justification
- Publish approval event to EventBridge custom event bus for cross-account notification

**codepipeline_definition.py**: Create the CodePipeline with cross-account stages:
- Pipeline name: `{PROJECT_NAME}-ml-deploy-pipeline-{ENV}`
- Artifact store: shared S3 bucket with KMS encryption key
- Stage 1 — Source: S3 source action triggered by model artifact upload to `s3://{SHARED_ARTIFACT_BUCKET}/models/{MODEL_PACKAGE_GROUP}/latest/`
- Stage 2 — StagingDeploy: cross-account deploy action using `staging_account_roles`. Invokes a Lambda function that calls `deploy_model.py` logic in the staging account via assumed role.
- Stage 3 — Approval: manual approval action with SNS notification to APPROVAL_NOTIFICATION_EMAIL. Approval URL includes model metrics summary.
- Stage 4 — ProdDeploy: cross-account deploy action using `prod_account_roles`. Same Lambda-based deployment pattern targeting the prod account.
- Each cross-account action specifies `RoleArn` pointing to the target account's pipeline role.

**source_stage.py**: Configure the S3 source stage:
- S3 source action watching `{SHARED_ARTIFACT_BUCKET}/models/{MODEL_PACKAGE_GROUP}/latest/model.tar.gz`
- PollForSourceChanges set to false — use EventBridge rule for S3 object creation instead
- Output artifact: `ModelArtifact` passed to subsequent stages

**staging_deploy_stage.py**: Configure the staging deployment stage:
- Lambda invoke action that assumes `{PROJECT_NAME}-pipeline-staging-role-{ENV}` in the staging account
- Lambda function calls `deploy_model.py` with staging-specific parameters (instance type, instance count)
- Post-deploy: invoke `endpoint_health_check.py` to validate the staging endpoint returns 200 on test payloads
- On failure: skip approval stage and mark pipeline as failed

**approval_stage.py**: Configure the manual approval stage:
- Manual approval action with `NotificationArn` pointing to the SNS topic
- Custom data includes: model package ARN, staging endpoint URL, evaluation metrics summary
- Approval/rejection triggers pipeline continuation or termination

**prod_deploy_stage.py**: Configure the production deployment stage:
- Lambda invoke action that assumes `{PROJECT_NAME}-pipeline-prod-role-{ENV}` in the prod account
- Lambda function calls `deploy_model.py` with production parameters (higher instance count, data capture enabled)
- Post-deploy: invoke `endpoint_health_check.py` against the prod endpoint
- If ROLLBACK_ON_ALARM is true: configure CloudWatch alarm check — if alarm fires within 10 minutes of deployment, automatically invoke `rollback.py`

**deploy_model.py**: Deploy a SageMaker model from the Model Registry to an endpoint:
- Assume the target account's SageMaker execution role
- Call `sagemaker.create_model()` with the model package ARN from the registry, referencing the model artifact in the shared S3 bucket
- Call `sagemaker.create_endpoint_config()` with instance type, instance count, and optional data capture configuration
- If endpoint exists: call `sagemaker.update_endpoint()` with the new endpoint config (blue/green deployment)
- If endpoint does not exist: call `sagemaker.create_endpoint()`
- Wait for endpoint status `InService` using `sagemaker.get_waiter('endpoint_in_service')`
- Store the previous endpoint config name in SSM Parameter Store for rollback: `/ml/{PROJECT_NAME}/{ENV}/previous-endpoint-config`

**rollback.py**: Rollback to the previous endpoint configuration:
- Read previous endpoint config name from SSM Parameter Store
- Call `sagemaker.update_endpoint()` with the previous endpoint config name
- Wait for endpoint status `InService`
- Publish rollback event to EventBridge with details (reason, previous config, new config)
- Send SNS notification to the approval topic about the rollback

**endpoint_health_check.py**: Validate endpoint health after deployment:
- Call `sagemaker_runtime.invoke_endpoint()` with a test payload
- Validate response status code is 200
- Validate response body is parseable JSON
- Check response latency is below a configurable threshold (default 5 seconds)
- Return pass/fail result to the pipeline stage

**eventbridge_rules.py**: Create EventBridge rules for model approval events:
- Rule 1: Match `SageMaker Model Package State Change` events where `ModelApprovalStatus` transitions to `Approved`. Target: start the CodePipeline execution via `codepipeline.start_pipeline_execution()`.
- Rule 2: Match `SageMaker Model Package State Change` events where `ModelApprovalStatus` transitions to `Rejected`. Target: SNS notification to the approval topic.
- Event pattern filters on `ModelPackageGroupName` matching `{MODEL_PACKAGE_GROUP}`.

**cross_account_event_bus.py**: Configure cross-account EventBridge event forwarding:
- Create custom event bus in the pipeline account: `{PROJECT_NAME}-ml-events-{ENV}`
- Add event bus policy allowing `events:PutEvents` from the dev account (where model approval events originate)
- Create rule in the dev account's default event bus that forwards `SageMaker Model Package State Change` events to the pipeline account's custom event bus
- Create rule in the pipeline account's custom event bus that triggers the CodePipeline on model approval

**run_setup.py**: CLI orchestrator that runs setup steps in order:
1. Load and validate configuration
2. Create IAM roles in all four accounts (requires credentials or profiles for each account)
3. Configure shared S3 bucket and KMS key policy
4. Create model package group with cross-account policy
5. Create CodePipeline with all stages
6. Create EventBridge rules and cross-account event bus
7. Print summary with role ARNs, pipeline ARN, bucket name, and event bus ARN

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**Cross-Account IAM Least Privilege:** Each cross-account role must be scoped to the minimum permissions required for its pipeline stage. Dev account roles only need read access to the Model Registry. Staging and prod roles need SageMaker deployment permissions but not training permissions. Never use `*` resource ARNs — scope to specific model package group ARNs, endpoint ARNs, and S3 prefixes.

**Trust Policy Security:** All cross-account trust policies must use `sts:AssumeRole` with the specific pipeline account ID as principal — never use wildcard account IDs in trust policies. Consider adding `sts:ExternalId` conditions for additional security. Limit `iam:PassRole` to specific SageMaker execution role ARNs.

**KMS Key Policy:** The KMS key must grant `kms:Decrypt` and `kms:GenerateDataKey` to all cross-account roles that access encrypted artifacts. Use `kms:ViaService` conditions to restrict key usage to S3 and SageMaker services only. The key must reside in the pipeline account. Never grant `kms:*` — enumerate specific actions.

**S3 Artifact Bucket:** The shared bucket must have a bucket policy granting cross-account access to specific IAM role ARNs (not account roots in production). Enable versioning for artifact rollback. Enable server-side encryption with the KMS key. Block all public access. Use S3 object prefixes to organize artifacts by model package group and version.

**Model Registry Cross-Account Policy:** Use `sagemaker.put_model_package_group_policy()` to grant cross-account access. The policy must grant `sagemaker:DescribeModelPackage` and `sagemaker:DescribeModelPackageGroup` to staging and prod accounts. Only the dev account should have `sagemaker:CreateModelPackage` permission. Only designated approver roles should have `sagemaker:UpdateModelPackage` for approval status changes.

**Pipeline Idempotency:** All deployment scripts must be idempotent. If an endpoint already exists, use `update_endpoint()` instead of `create_endpoint()`. If a model with the same name exists, handle the `ResourceInUse` exception gracefully. Store deployment state in SSM Parameter Store for rollback tracking.

**Rollback Strategy:** Store the previous endpoint configuration name before every deployment. On rollback, revert to the stored config using `update_endpoint()`. Publish rollback events to EventBridge for audit. If ROLLBACK_ON_ALARM is enabled, create a CloudWatch alarm on `Invocation5XXErrors` and `ModelLatency` that triggers an automatic rollback Lambda within 10 minutes of deployment.

**EventBridge Cross-Account:** Use event bus policies (not IAM policies) to grant cross-account `PutEvents` permission. Filter events by `ModelPackageGroupName` to avoid triggering on unrelated model approvals. Use the default event bus in the source account and a custom event bus in the pipeline account for isolation.

**Encryption:** All artifacts must be encrypted at rest using the shared KMS key. CodePipeline artifact store must reference the KMS key ARN. SageMaker model artifacts in S3 must use SSE-KMS. Data capture output (if enabled) must use the same KMS key.

**Naming:** All resources follow `{PROJECT_NAME}-{component}-{ENV}` convention:
- IAM roles: `{PROJECT_NAME}-pipeline-{account}-role-{ENV}` (e.g., `myai-pipeline-staging-role-prod`)
- SageMaker execution roles: `{PROJECT_NAME}-sagemaker-exec-{account}-{ENV}`
- Pipeline: `{PROJECT_NAME}-ml-deploy-pipeline-{ENV}`
- Endpoints: `{PROJECT_NAME}-endpoint-{ENV}` (in each target account)
- Event bus: `{PROJECT_NAME}-ml-events-{ENV}`
- SSM parameters: `/ml/{PROJECT_NAME}/{ENV}/{parameter-name}`

**Testing:** Test the full pipeline in a sandbox set of accounts before deploying to production accounts. Verify cross-account role assumption works from the pipeline account to each target account. Test rollback by intentionally deploying a broken model. Verify EventBridge events flow correctly across account boundaries.

---

## Code Scaffolding Hints

**Cross-account IAM trust policy for pipeline role assumption:**
```python
import boto3
import json

iam = boto3.client("iam")

# Trust policy allowing the pipeline account to assume this role
cross_account_trust_policy = {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": f"arn:aws:iam::{PIPELINE_ACCOUNT_ID}:root"
            },
            "Action": "sts:AssumeRole",
            "Condition": {
                "StringEquals": {
                    "sts:ExternalId": f"{PROJECT_NAME}-pipeline-{ENV}"
                }
            },
        }
    ],
}

# Create cross-account role in the staging account
response = iam.create_role(
    RoleName=f"{PROJECT_NAME}-pipeline-staging-role-{ENV}",
    AssumeRolePolicyDocument=json.dumps(cross_account_trust_policy),
    Description=f"Cross-account role for ML pipeline staging deployments — {PROJECT_NAME}",
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)
role_arn = response["Role"]["Arn"]

# Attach inline policy scoped to SageMaker deployment actions
staging_deploy_policy = {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "SageMakerDeployment",
            "Effect": "Allow",
            "Action": [
                "sagemaker:CreateModel",
                "sagemaker:CreateEndpointConfig",
                "sagemaker:CreateEndpoint",
                "sagemaker:UpdateEndpoint",
                "sagemaker:DescribeEndpoint",
                "sagemaker:DescribeEndpointConfig",
                "sagemaker:DeleteEndpoint",
                "sagemaker:DeleteEndpointConfig",
                "sagemaker:DeleteModel",
                "sagemaker:DescribeModelPackage",
                "sagemaker:ListModelPackages",
            ],
            "Resource": [
                f"arn:aws:sagemaker:{AWS_REGION}:{STAGING_ACCOUNT_ID}:model/{PROJECT_NAME}-*",
                f"arn:aws:sagemaker:{AWS_REGION}:{STAGING_ACCOUNT_ID}:endpoint/{PROJECT_NAME}-*",
                f"arn:aws:sagemaker:{AWS_REGION}:{STAGING_ACCOUNT_ID}:endpoint-config/{PROJECT_NAME}-*",
                f"arn:aws:sagemaker:{AWS_REGION}:{DEV_ACCOUNT_ID}:model-package/{MODEL_PACKAGE_GROUP}/*",
                f"arn:aws:sagemaker:{AWS_REGION}:{DEV_ACCOUNT_ID}:model-package-group/{MODEL_PACKAGE_GROUP}",
            ],
        },
        {
            "Sid": "PassSageMakerExecutionRole",
            "Effect": "Allow",
            "Action": "iam:PassRole",
            "Resource": f"arn:aws:iam::{STAGING_ACCOUNT_ID}:role/{PROJECT_NAME}-sagemaker-exec-staging-{ENV}",
            "Condition": {
                "StringEquals": {
                    "iam:PassedToService": "sagemaker.amazonaws.com"
                }
            },
        },
        {
            "Sid": "ArtifactBucketAccess",
            "Effect": "Allow",
            "Action": ["s3:GetObject", "s3:GetBucketLocation", "s3:ListBucket"],
            "Resource": [
                f"arn:aws:s3:::{SHARED_ARTIFACT_BUCKET}",
                f"arn:aws:s3:::{SHARED_ARTIFACT_BUCKET}/*",
            ],
        },
        {
            "Sid": "KMSDecrypt",
            "Effect": "Allow",
            "Action": ["kms:Decrypt", "kms:DescribeKey"],
            "Resource": KMS_KEY_ARN,
        },
    ],
}

iam.put_role_policy(
    RoleName=f"{PROJECT_NAME}-pipeline-staging-role-{ENV}",
    PolicyName=f"{PROJECT_NAME}-staging-deploy-policy",
    PolicyDocument=json.dumps(staging_deploy_policy),
)
print(f"Created staging cross-account role: {role_arn}")
```

**SageMaker Model Registry cross-account resource policy:**
```python
import boto3
import json

sagemaker = boto3.client("sagemaker", region_name=AWS_REGION)

# Create model package group
sagemaker.create_model_package_group(
    ModelPackageGroupName=MODEL_PACKAGE_GROUP,
    ModelPackageGroupDescription=f"Model package group for {PROJECT_NAME} — cross-account enabled",
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)

# Cross-account resource policy allowing staging and prod to read model packages
cross_account_policy = {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowCrossAccountDescribe",
            "Effect": "Allow",
            "Principal": {
                "AWS": [
                    f"arn:aws:iam::{STAGING_ACCOUNT_ID}:root",
                    f"arn:aws:iam::{PROD_ACCOUNT_ID}:root",
                ]
            },
            "Action": [
                "sagemaker:DescribeModelPackageGroup",
                "sagemaker:DescribeModelPackage",
                "sagemaker:ListModelPackages",
            ],
            "Resource": [
                f"arn:aws:sagemaker:{AWS_REGION}:{DEV_ACCOUNT_ID}:model-package-group/{MODEL_PACKAGE_GROUP}",
                f"arn:aws:sagemaker:{AWS_REGION}:{DEV_ACCOUNT_ID}:model-package/{MODEL_PACKAGE_GROUP}/*",
            ],
        },
        {
            "Sid": "AllowPipelineAccountApproval",
            "Effect": "Allow",
            "Principal": {
                "AWS": f"arn:aws:iam::{PIPELINE_ACCOUNT_ID}:root"
            },
            "Action": [
                "sagemaker:DescribeModelPackageGroup",
                "sagemaker:DescribeModelPackage",
                "sagemaker:ListModelPackages",
                "sagemaker:UpdateModelPackage",
            ],
            "Resource": [
                f"arn:aws:sagemaker:{AWS_REGION}:{DEV_ACCOUNT_ID}:model-package-group/{MODEL_PACKAGE_GROUP}",
                f"arn:aws:sagemaker:{AWS_REGION}:{DEV_ACCOUNT_ID}:model-package/{MODEL_PACKAGE_GROUP}/*",
            ],
        },
    ],
}

sagemaker.put_model_package_group_policy(
    ModelPackageGroupName=MODEL_PACKAGE_GROUP,
    ResourcePolicy=json.dumps(cross_account_policy),
)
print(f"Applied cross-account policy to model package group: {MODEL_PACKAGE_GROUP}")
```

**Shared S3 artifact bucket with cross-account bucket policy:**
```python
import boto3
import json

s3 = boto3.client("s3", region_name=AWS_REGION)

# Create bucket with encryption
s3.create_bucket(
    Bucket=SHARED_ARTIFACT_BUCKET,
    CreateBucketConfiguration={"LocationConstraint": AWS_REGION},
)

# Enable versioning
s3.put_bucket_versioning(
    Bucket=SHARED_ARTIFACT_BUCKET,
    VersioningConfiguration={"Status": "Enabled"},
)

# Enable SSE-KMS default encryption
s3.put_bucket_encryption(
    Bucket=SHARED_ARTIFACT_BUCKET,
    ServerSideEncryptionConfiguration={
        "Rules": [
            {
                "ApplyServerSideEncryptionByDefault": {
                    "SSEAlgorithm": "aws:kms",
                    "KMSMasterKeyID": KMS_KEY_ARN,
                },
                "BucketKeyEnabled": True,
            }
        ]
    },
)

# Block public access
s3.put_public_access_block(
    Bucket=SHARED_ARTIFACT_BUCKET,
    PublicAccessBlockConfiguration={
        "BlockPublicAcls": True,
        "IgnorePublicAcls": True,
        "BlockPublicPolicy": True,
        "RestrictPublicBuckets": True,
    },
)

# Cross-account bucket policy
bucket_policy = {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowCrossAccountPipelineAccess",
            "Effect": "Allow",
            "Principal": {
                "AWS": [
                    f"arn:aws:iam::{PIPELINE_ACCOUNT_ID}:role/{PROJECT_NAME}-pipeline-service-role-{ENV}",
                    f"arn:aws:iam::{DEV_ACCOUNT_ID}:role/{PROJECT_NAME}-pipeline-dev-role-{ENV}",
                    f"arn:aws:iam::{STAGING_ACCOUNT_ID}:role/{PROJECT_NAME}-pipeline-staging-role-{ENV}",
                    f"arn:aws:iam::{PROD_ACCOUNT_ID}:role/{PROJECT_NAME}-pipeline-prod-role-{ENV}",
                ]
            },
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:GetBucketLocation",
                "s3:ListBucket",
            ],
            "Resource": [
                f"arn:aws:s3:::{SHARED_ARTIFACT_BUCKET}",
                f"arn:aws:s3:::{SHARED_ARTIFACT_BUCKET}/*",
            ],
        },
        {
            "Sid": "AllowSageMakerServiceAccess",
            "Effect": "Allow",
            "Principal": {
                "AWS": [
                    f"arn:aws:iam::{STAGING_ACCOUNT_ID}:role/{PROJECT_NAME}-sagemaker-exec-staging-{ENV}",
                    f"arn:aws:iam::{PROD_ACCOUNT_ID}:role/{PROJECT_NAME}-sagemaker-exec-prod-{ENV}",
                ]
            },
            "Action": ["s3:GetObject", "s3:GetBucketLocation", "s3:ListBucket"],
            "Resource": [
                f"arn:aws:s3:::{SHARED_ARTIFACT_BUCKET}",
                f"arn:aws:s3:::{SHARED_ARTIFACT_BUCKET}/*",
            ],
        },
        {
            "Sid": "DenyUnencryptedUploads",
            "Effect": "Deny",
            "Principal": "*",
            "Action": "s3:PutObject",
            "Resource": f"arn:aws:s3:::{SHARED_ARTIFACT_BUCKET}/*",
            "Condition": {
                "StringNotEquals": {
                    "s3:x-amz-server-side-encryption": "aws:kms"
                }
            },
        },
    ],
}

s3.put_bucket_policy(
    Bucket=SHARED_ARTIFACT_BUCKET,
    Policy=json.dumps(bucket_policy),
)
print(f"Configured shared artifact bucket: {SHARED_ARTIFACT_BUCKET}")
```

**CodePipeline with cross-account stages:**
```python
import boto3
import json

codepipeline = boto3.client("codepipeline", region_name=AWS_REGION)

pipeline_definition = {
    "name": f"{PROJECT_NAME}-ml-deploy-pipeline-{ENV}",
    "roleArn": f"arn:aws:iam::{PIPELINE_ACCOUNT_ID}:role/{PROJECT_NAME}-pipeline-service-role-{ENV}",
    "artifactStore": {
        "type": "S3",
        "location": SHARED_ARTIFACT_BUCKET,
        "encryptionKey": {
            "id": KMS_KEY_ARN,
            "type": "KMS",
        },
    },
    "stages": [
        {
            "name": "Source",
            "actions": [
                {
                    "name": "ModelArtifactSource",
                    "actionTypeId": {
                        "category": "Source",
                        "owner": "AWS",
                        "provider": "S3",
                        "version": "1",
                    },
                    "configuration": {
                        "S3Bucket": SHARED_ARTIFACT_BUCKET,
                        "S3ObjectKey": f"models/{MODEL_PACKAGE_GROUP}/latest/model.tar.gz",
                        "PollForSourceChanges": "false",
                    },
                    "outputArtifacts": [{"name": "ModelArtifact"}],
                    "runOrder": 1,
                }
            ],
        },
        {
            "name": "StagingDeploy",
            "actions": [
                {
                    "name": "DeployToStaging",
                    "actionTypeId": {
                        "category": "Invoke",
                        "owner": "AWS",
                        "provider": "Lambda",
                        "version": "1",
                    },
                    "configuration": {
                        "FunctionName": f"{PROJECT_NAME}-deploy-staging-{ENV}",
                        "UserParameters": json.dumps({
                            "target_account_id": STAGING_ACCOUNT_ID,
                            "role_arn": f"arn:aws:iam::{STAGING_ACCOUNT_ID}:role/{PROJECT_NAME}-pipeline-staging-role-{ENV}",
                            "endpoint_name": f"{PROJECT_NAME}-endpoint-staging",
                            "instance_type": DEPLOY_INSTANCE_TYPE,
                            "instance_count": DEPLOY_INSTANCE_COUNT,
                        }),
                    },
                    "inputArtifacts": [{"name": "ModelArtifact"}],
                    "roleArn": f"arn:aws:iam::{PIPELINE_ACCOUNT_ID}:role/{PROJECT_NAME}-pipeline-action-role-{ENV}",
                    "runOrder": 1,
                }
            ],
        },
        {
            "name": "ManualApproval",
            "actions": [
                {
                    "name": "ApproveForProduction",
                    "actionTypeId": {
                        "category": "Approval",
                        "owner": "AWS",
                        "provider": "Manual",
                        "version": "1",
                    },
                    "configuration": {
                        "NotificationArn": f"arn:aws:sns:{AWS_REGION}:{PIPELINE_ACCOUNT_ID}:{PROJECT_NAME}-ml-approval-{ENV}",
                        "CustomData": f"Model from {MODEL_PACKAGE_GROUP} deployed to staging. Review staging endpoint before approving production deployment.",
                    },
                    "runOrder": 1,
                }
            ],
        },
        {
            "name": "ProductionDeploy",
            "actions": [
                {
                    "name": "DeployToProduction",
                    "actionTypeId": {
                        "category": "Invoke",
                        "owner": "AWS",
                        "provider": "Lambda",
                        "version": "1",
                    },
                    "configuration": {
                        "FunctionName": f"{PROJECT_NAME}-deploy-prod-{ENV}",
                        "UserParameters": json.dumps({
                            "target_account_id": PROD_ACCOUNT_ID,
                            "role_arn": f"arn:aws:iam::{PROD_ACCOUNT_ID}:role/{PROJECT_NAME}-pipeline-prod-role-{ENV}",
                            "endpoint_name": f"{PROJECT_NAME}-endpoint-prod",
                            "instance_type": DEPLOY_INSTANCE_TYPE,
                            "instance_count": max(2, DEPLOY_INSTANCE_COUNT),
                            "enable_data_capture": ENABLE_DATA_CAPTURE,
                        }),
                    },
                    "inputArtifacts": [{"name": "ModelArtifact"}],
                    "roleArn": f"arn:aws:iam::{PIPELINE_ACCOUNT_ID}:role/{PROJECT_NAME}-pipeline-action-role-{ENV}",
                    "runOrder": 1,
                }
            ],
        },
    ],
}

response = codepipeline.create_pipeline(pipeline=pipeline_definition)
pipeline_arn = response["pipeline"]["name"]
print(f"Created cross-account ML pipeline: {pipeline_arn}")
```

**Deploy model from registry to endpoint (cross-account):**
```python
import boto3
import json
import time

def deploy_model_cross_account(
    target_account_role_arn,
    model_package_arn,
    endpoint_name,
    instance_type,
    instance_count,
    enable_data_capture=False,
):
    """Deploy a model from the registry to a SageMaker endpoint in a target account."""
    # Assume cross-account role
    sts = boto3.client("sts")
    assumed = sts.assume_role(
        RoleArn=target_account_role_arn,
        RoleSessionName=f"{PROJECT_NAME}-deploy-session",
        ExternalId=f"{PROJECT_NAME}-pipeline-{ENV}",
    )
    credentials = assumed["Credentials"]

    # Create SageMaker client with assumed credentials
    sm = boto3.client(
        "sagemaker",
        region_name=AWS_REGION,
        aws_access_key_id=credentials["AccessKeyId"],
        aws_secret_access_key=credentials["SecretAccessKey"],
        aws_session_token=credentials["SessionToken"],
    )

    ssm = boto3.client(
        "ssm",
        region_name=AWS_REGION,
        aws_access_key_id=credentials["AccessKeyId"],
        aws_secret_access_key=credentials["SecretAccessKey"],
        aws_session_token=credentials["SessionToken"],
    )

    model_name = f"{PROJECT_NAME}-model-{int(time.time())}"
    endpoint_config_name = f"{PROJECT_NAME}-epc-{int(time.time())}"
    target_account_id = target_account_role_arn.split(":")[4]

    # Create model from model package
    create_model_params = {
        "ModelName": model_name,
        "PrimaryContainer": {
            "ModelPackageName": model_package_arn,
        },
        "ExecutionRoleArn": f"arn:aws:iam::{target_account_id}:role/{PROJECT_NAME}-sagemaker-exec-{ENV}",
        "Tags": [
            {"Key": "Project", "Value": PROJECT_NAME},
            {"Key": "Environment", "Value": ENV},
            {"Key": "ModelPackageArn", "Value": model_package_arn},
        ],
    }
    sm.create_model(**create_model_params)
    print(f"Created model: {model_name}")

    # Create endpoint config
    endpoint_config_params = {
        "EndpointConfigName": endpoint_config_name,
        "ProductionVariants": [
            {
                "VariantName": "AllTraffic",
                "ModelName": model_name,
                "InstanceType": instance_type,
                "InitialInstanceCount": instance_count,
                "InitialVariantWeight": 1.0,
            }
        ],
        "KmsKeyId": KMS_KEY_ARN,
        "Tags": [
            {"Key": "Project", "Value": PROJECT_NAME},
            {"Key": "Environment", "Value": ENV},
        ],
    }

    if enable_data_capture:
        endpoint_config_params["DataCaptureConfig"] = {
            "EnableCapture": True,
            "InitialSamplingPercentage": 100,
            "DestinationS3Uri": f"s3://{SHARED_ARTIFACT_BUCKET}/data-capture/{endpoint_name}/",
            "CaptureOptions": [
                {"CaptureMode": "Input"},
                {"CaptureMode": "Output"},
            ],
            "CaptureContentTypeHeader": {
                "CsvContentTypes": ["text/csv"],
                "JsonContentTypes": ["application/json"],
            },
        }

    sm.create_endpoint_config(**endpoint_config_params)
    print(f"Created endpoint config: {endpoint_config_name}")

    # Store previous endpoint config for rollback
    try:
        existing = sm.describe_endpoint(EndpointName=endpoint_name)
        previous_config = existing["EndpointConfigName"]
        ssm.put_parameter(
            Name=f"/ml/{PROJECT_NAME}/{ENV}/previous-endpoint-config",
            Value=previous_config,
            Type="String",
            Overwrite=True,
        )
        # Update existing endpoint
        sm.update_endpoint(
            EndpointName=endpoint_name,
            EndpointConfigName=endpoint_config_name,
        )
        print(f"Updating endpoint: {endpoint_name} (previous config: {previous_config})")
    except sm.exceptions.ClientError:
        # Endpoint does not exist — create it
        sm.create_endpoint(
            EndpointName=endpoint_name,
            EndpointConfigName=endpoint_config_name,
            Tags=[
                {"Key": "Project", "Value": PROJECT_NAME},
                {"Key": "Environment", "Value": ENV},
            ],
        )
        print(f"Creating new endpoint: {endpoint_name}")

    # Wait for endpoint to be in service
    waiter = sm.get_waiter("endpoint_in_service")
    waiter.wait(
        EndpointName=endpoint_name,
        WaiterConfig={"Delay": 30, "MaxAttempts": 60},
    )
    print(f"Endpoint {endpoint_name} is InService")
    return endpoint_name
```

**Rollback to previous endpoint configuration:**
```python
def rollback_endpoint(target_account_role_arn, endpoint_name):
    """Rollback an endpoint to its previous configuration."""
    sts = boto3.client("sts")
    assumed = sts.assume_role(
        RoleArn=target_account_role_arn,
        RoleSessionName=f"{PROJECT_NAME}-rollback-session",
        ExternalId=f"{PROJECT_NAME}-pipeline-{ENV}",
    )
    credentials = assumed["Credentials"]

    sm = boto3.client(
        "sagemaker",
        region_name=AWS_REGION,
        aws_access_key_id=credentials["AccessKeyId"],
        aws_secret_access_key=credentials["SecretAccessKey"],
        aws_session_token=credentials["SessionToken"],
    )
    ssm = boto3.client(
        "ssm",
        region_name=AWS_REGION,
        aws_access_key_id=credentials["AccessKeyId"],
        aws_secret_access_key=credentials["SecretAccessKey"],
        aws_session_token=credentials["SessionToken"],
    )

    # Retrieve previous endpoint config from SSM
    previous_config = ssm.get_parameter(
        Name=f"/ml/{PROJECT_NAME}/{ENV}/previous-endpoint-config"
    )["Parameter"]["Value"]

    current = sm.describe_endpoint(EndpointName=endpoint_name)
    current_config = current["EndpointConfigName"]

    print(f"Rolling back {endpoint_name}: {current_config} → {previous_config}")

    sm.update_endpoint(
        EndpointName=endpoint_name,
        EndpointConfigName=previous_config,
    )

    waiter = sm.get_waiter("endpoint_in_service")
    waiter.wait(
        EndpointName=endpoint_name,
        WaiterConfig={"Delay": 30, "MaxAttempts": 60},
    )
    print(f"Rollback complete: {endpoint_name} now using {previous_config}")

    # Publish rollback event
    events = boto3.client("events", region_name=AWS_REGION)
    events.put_events(
        Entries=[
            {
                "Source": f"{PROJECT_NAME}.ml-pipeline",
                "DetailType": "ML Endpoint Rollback",
                "Detail": json.dumps({
                    "endpoint_name": endpoint_name,
                    "previous_config": current_config,
                    "rolled_back_to": previous_config,
                    "project": PROJECT_NAME,
                    "environment": ENV,
                }),
                "EventBusName": f"{PROJECT_NAME}-ml-events-{ENV}",
            }
        ]
    )
    return previous_config
```

**EventBridge cross-account rules for model approval events:**
```python
import boto3
import json

events = boto3.client("events", region_name=AWS_REGION)

# --- In the pipeline account: create custom event bus ---
events.create_event_bus(
    Name=f"{PROJECT_NAME}-ml-events-{ENV}",
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)

# Allow dev account to put events on the pipeline account's custom bus
events.put_permission(
    EventBusName=f"{PROJECT_NAME}-ml-events-{ENV}",
    Action="events:PutEvents",
    Principal=DEV_ACCOUNT_ID,
    StatementId=f"AllowDevAccountPutEvents",
)

# Rule: trigger pipeline on model approval
events.put_rule(
    Name=f"{PROJECT_NAME}-model-approved-trigger-{ENV}",
    EventBusName=f"{PROJECT_NAME}-ml-events-{ENV}",
    EventPattern=json.dumps({
        "source": ["aws.sagemaker"],
        "detail-type": ["SageMaker Model Package State Change"],
        "detail": {
            "ModelPackageGroupName": [MODEL_PACKAGE_GROUP],
            "ModelApprovalStatus": ["Approved"],
        },
    }),
    State="ENABLED",
    Description=f"Trigger ML pipeline when model is approved in {MODEL_PACKAGE_GROUP}",
)

# Target: start CodePipeline execution
events.put_targets(
    Rule=f"{PROJECT_NAME}-model-approved-trigger-{ENV}",
    EventBusName=f"{PROJECT_NAME}-ml-events-{ENV}",
    Targets=[
        {
            "Id": "StartMLPipeline",
            "Arn": f"arn:aws:codepipeline:{AWS_REGION}:{PIPELINE_ACCOUNT_ID}:{PROJECT_NAME}-ml-deploy-pipeline-{ENV}",
            "RoleArn": f"arn:aws:iam::{PIPELINE_ACCOUNT_ID}:role/{PROJECT_NAME}-eventbridge-pipeline-role-{ENV}",
        }
    ],
)
print(f"Created EventBridge rule for model approval trigger")

# --- In the dev account: forward model approval events to pipeline account ---
# (Run with dev account credentials)
dev_events = boto3.client("events", region_name=AWS_REGION)

dev_events.put_rule(
    Name=f"{PROJECT_NAME}-forward-model-events-{ENV}",
    EventPattern=json.dumps({
        "source": ["aws.sagemaker"],
        "detail-type": ["SageMaker Model Package State Change"],
        "detail": {
            "ModelPackageGroupName": [MODEL_PACKAGE_GROUP],
        },
    }),
    State="ENABLED",
    Description=f"Forward model package events to pipeline account",
)

dev_events.put_targets(
    Rule=f"{PROJECT_NAME}-forward-model-events-{ENV}",
    Targets=[
        {
            "Id": "ForwardToPipelineAccount",
            "Arn": f"arn:aws:events:{AWS_REGION}:{PIPELINE_ACCOUNT_ID}:event-bus/{PROJECT_NAME}-ml-events-{ENV}",
            "RoleArn": f"arn:aws:iam::{DEV_ACCOUNT_ID}:role/{PROJECT_NAME}-eventbridge-forward-role-{ENV}",
        }
    ],
)
print(f"Created event forwarding rule in dev account")
```

**KMS key policy for cross-account artifact encryption:**
```python
import boto3
import json

kms = boto3.client("kms", region_name=AWS_REGION)

key_policy = {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "EnableRootAccountAccess",
            "Effect": "Allow",
            "Principal": {
                "AWS": f"arn:aws:iam::{PIPELINE_ACCOUNT_ID}:root"
            },
            "Action": "kms:*",
            "Resource": "*",
        },
        {
            "Sid": "AllowCrossAccountDecrypt",
            "Effect": "Allow",
            "Principal": {
                "AWS": [
                    f"arn:aws:iam::{DEV_ACCOUNT_ID}:role/{PROJECT_NAME}-pipeline-dev-role-{ENV}",
                    f"arn:aws:iam::{STAGING_ACCOUNT_ID}:role/{PROJECT_NAME}-pipeline-staging-role-{ENV}",
                    f"arn:aws:iam::{STAGING_ACCOUNT_ID}:role/{PROJECT_NAME}-sagemaker-exec-staging-{ENV}",
                    f"arn:aws:iam::{PROD_ACCOUNT_ID}:role/{PROJECT_NAME}-pipeline-prod-role-{ENV}",
                    f"arn:aws:iam::{PROD_ACCOUNT_ID}:role/{PROJECT_NAME}-sagemaker-exec-prod-{ENV}",
                ]
            },
            "Action": [
                "kms:Decrypt",
                "kms:DescribeKey",
                "kms:GenerateDataKey",
                "kms:ReEncryptFrom",
                "kms:ReEncryptTo",
            ],
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "kms:ViaService": [
                        f"s3.{AWS_REGION}.amazonaws.com",
                        f"sagemaker.{AWS_REGION}.amazonaws.com",
                    ]
                }
            },
        },
        {
            "Sid": "AllowPipelineServiceRole",
            "Effect": "Allow",
            "Principal": {
                "AWS": f"arn:aws:iam::{PIPELINE_ACCOUNT_ID}:role/{PROJECT_NAME}-pipeline-service-role-{ENV}"
            },
            "Action": [
                "kms:Decrypt",
                "kms:DescribeKey",
                "kms:GenerateDataKey",
                "kms:ReEncryptFrom",
                "kms:ReEncryptTo",
            ],
            "Resource": "*",
        },
    ],
}

# Extract key ID from ARN
key_id = KMS_KEY_ARN.split("/")[-1]

kms.put_key_policy(
    KeyId=key_id,
    PolicyName="default",
    Policy=json.dumps(key_policy),
)
print(f"Updated KMS key policy for cross-account access: {KMS_KEY_ARN}")
```

---

## Integration Points

- **Upstream**: `devops/04` → IAM roles and policies foundation; cross-account roles in this template extend the IAM patterns established in devops/04 with trust policies and least-privilege scoping
- **Upstream**: `devops/08` → KMS encryption keys; the shared KMS key referenced by KMS_KEY_ARN should be created using devops/08 patterns with cross-account key policy grants
- **Upstream**: `mlops/10` → SageMaker Model Registry; model package groups and versioned model packages are registered using mlops/10 patterns, then consumed by this pipeline for cross-account deployment
- **Downstream**: `enterprise/05` → Centralized Model Registry; approved models deployed through this pipeline can be registered in the central registry for organization-wide discovery and governance
- **Downstream**: `cicd/03` → CodePipeline multi-stage ML patterns; this template extends cicd/03 with cross-account stages, IAM assume-role chains, and shared artifact encryption
- **Related**: `enterprise/01` → SCPs may restrict which instance types, regions, and Bedrock models can be used in staging and prod accounts; ensure SCP allowlists align with deployment configurations
- **Related**: `data/05` → EventBridge ML orchestration; model approval events from this pipeline can feed into broader event-driven ML workflows defined in data/05
- **Related**: `devops/02` → VPC networking; SageMaker endpoints in staging and prod accounts may require VPC configuration from devops/02 for private inference
