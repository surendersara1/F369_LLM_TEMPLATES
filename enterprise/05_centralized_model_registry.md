<!-- Template Version: 1.0 | boto3: 1.35+ | terraform-aws: 5.70+ | cdk: 2.170+ -->

# Template Enterprise 05 — Centralized Model Registry

## Purpose
Generate a production-ready centralized SageMaker Model Registry operating in a shared services account for organization-wide model governance: cross-account resource policies granting producer accounts write access and consumer accounts read access to model package groups, an EventBridge approval notification workflow that publishes model approval events across account boundaries, model lineage metadata tracking (training data S3 URI, training job ARN, evaluation metrics, Git commit SHA) embedded in every registered model package via `CustomerMetadataProperties`, cross-account event publishing on model approval status changes for triggering downstream deployment pipelines, and a model catalog query utility for discovering, filtering, and comparing model versions across all registered model package groups.

---

## Role Definition

You are an expert AWS cloud architect and MLOps engineer specializing in centralized model governance with expertise in:
- SageMaker Model Registry: model package groups, model package versions, approval status transitions (`PendingManualApproval` → `Approved` → `Rejected`), cross-account resource policies via `sagemaker.put_model_package_group_policy()`
- Cross-account model sharing: resource policies granting producer accounts `sagemaker:CreateModelPackage` and consumer accounts `sagemaker:DescribeModelPackage`, `sagemaker:ListModelPackages` on centralized model package groups in a shared services account
- Model lineage metadata: `CustomerMetadataProperties` for embedding training data S3 URI, training job ARN, evaluation metrics JSON path, Git commit SHA, experiment run ID, and dataset version into every model package registration
- EventBridge cross-account patterns: event bus policies for cross-account `PutEvents`, rules matching `SageMaker Model Package State Change` events, forwarding approval events from the registry account to consumer accounts for triggering deployment pipelines
- Model catalog management: querying model packages with filters (approval status, creation time, model package group), comparing model versions by metrics, generating catalog reports for governance dashboards
- SageMaker Model Registry API: `create_model_package_group()`, `put_model_package_group_policy()`, `create_model_package()`, `update_model_package()`, `list_model_packages()`, `describe_model_package()`
- IAM cross-account trust policies for SageMaker service principals and pipeline roles accessing the centralized registry
- Approval workflows: SNS notification on model registration, manual and automated approval gates, audit logging of approval decisions to DynamoDB

Generate complete, production-deployable centralized model registry code.

---

## Context & Inputs

```
PROJECT_NAME:               [REQUIRED]
AWS_REGION:                 [REQUIRED]
AWS_ACCOUNT_ID:             [REQUIRED]
ENV:                        [REQUIRED - dev | stage | prod]

REGISTRY_ACCOUNT_ID:        [REQUIRED - AWS account ID hosting the centralized Model Registry]
                            Example: "555555555555"
                            The shared services account that owns all model package groups.
                            All model registrations, approvals, and catalog queries happen
                            in this account. Producer and consumer accounts access the
                            registry via cross-account resource policies.

PRODUCER_ACCOUNTS:          [REQUIRED - JSON list of AWS account IDs that register models]
                            Example: ["111111111111", "222222222222"]
                            Accounts where SageMaker training jobs run and models are
                            produced. These accounts are granted write access to register
                            new model package versions in the centralized registry.
                            Producers can create model packages and update metadata but
                            cannot approve models.

CONSUMER_ACCOUNTS:          [REQUIRED - JSON list of AWS account IDs that deploy models]
                            Example: ["333333333333", "444444444444"]
                            Accounts where approved models are deployed to SageMaker
                            endpoints. These accounts are granted read access to describe
                            and list model packages. Consumer accounts receive EventBridge
                            events on model approval for triggering deployment pipelines.

MODEL_PACKAGE_GROUPS:       [REQUIRED - JSON object mapping group names to descriptions]
                            Example: {
                              "customer-churn-models": "Customer churn prediction models",
                              "fraud-detection-models": "Real-time fraud detection models",
                              "recommendation-models": "Product recommendation models"
                            }
                            Each key becomes a SageMaker Model Package Group in the
                            registry account. Cross-account policies are applied to
                            each group individually.

APPROVAL_WORKFLOW_ENABLED:  [OPTIONAL: true]
                            When true, all registered models start with
                            PendingManualApproval status and require explicit approval.
                            When false, models from trusted producer accounts are
                            auto-approved based on metric thresholds.

AUTO_APPROVE_METRIC:        [OPTIONAL: none]
                            Metric name for auto-approval (e.g., "accuracy", "f1_score").
                            Only used when APPROVAL_WORKFLOW_ENABLED is false.

AUTO_APPROVE_THRESHOLD:     [OPTIONAL: 0.90]
                            Minimum metric value for auto-approval. Models below this
                            threshold remain in PendingManualApproval status.

LINEAGE_TRACKING:           [OPTIONAL: true]
                            When true, enforces that every model registration includes
                            lineage metadata (training data S3, job ARN, eval metrics,
                            Git commit). Registrations missing required metadata are
                            rejected with a validation error.

NOTIFICATION_EMAIL:         [OPTIONAL: none]
                            Email address for SNS notifications on model registration
                            and approval status changes.

KMS_KEY_ARN:                [OPTIONAL: none]
                            KMS key ARN in the registry account for encrypting model
                            artifacts. If provided, cross-account key grants are created
                            for producer and consumer accounts.

IAC_TOOL:                   [REQUIRED - boto3 | terraform | cdk]
```

---

## Task

Generate complete centralized model registry infrastructure:

```
{PROJECT_NAME}-centralized-registry/
├── registry/
│   ├── create_model_groups.py              # Create all model package groups
│   ├── cross_account_policies.py           # Cross-account resource policies per group
│   ├── register_model.py                   # Register model with lineage metadata
│   ├── approve_model.py                    # Approve/reject model package versions
│   └── list_models.py                      # List and filter model packages
├── lineage/
│   ├── lineage_metadata.py                 # Build lineage metadata for registration
│   └── validate_lineage.py                 # Validate required lineage fields
├── catalog/
│   ├── query_catalog.py                    # Query and filter model catalog
│   ├── compare_models.py                   # Compare model versions by metrics
│   └── catalog_report.py                   # Generate catalog report (JSON/CSV)
├── events/
│   ├── event_bus.py                        # Custom event bus + cross-account policies
│   ├── approval_rules.py                   # EventBridge rules for approval events
│   └── cross_account_forwarding.py         # Forward events to consumer accounts
├── approval/
│   ├── approval_workflow.py                # SNS notification + approval gate
│   ├── auto_approve_lambda/
│   │   └── handler.py                      # Lambda: auto-approve if metric > threshold
│   └── audit_log.py                        # DynamoDB audit log for approvals
├── infrastructure/
│   └── config.py                           # Central configuration
├── run_setup.py                            # CLI orchestrator
└── requirements.txt
```

**config.py**: Central configuration dataclass with all parameters. Load from environment variables or CLI args. Validate required fields. Construct resource names using `{PROJECT_NAME}-{component}-{ENV}` convention. Parse PRODUCER_ACCOUNTS and CONSUMER_ACCOUNTS JSON lists. Parse MODEL_PACKAGE_GROUPS JSON object. Validate AWS account ID format (12 digits) for REGISTRY_ACCOUNT_ID and all producer/consumer accounts. Validate KMS key ARN format if provided. Store combined account list for cross-account policy generation.

**create_model_groups.py**: Create all model package groups in the registry account:
- Iterate over MODEL_PACKAGE_GROUPS dict
- For each group, call `sagemaker.create_model_package_group()` with group name, description, and tags (Project, Environment, ManagedBy)
- Handle `ResourceInUse` exception gracefully if group already exists
- Tag each group with `{PROJECT_NAME}`, `{ENV}`, and `registry-account` tags
- Print summary of created/existing groups with ARNs

**cross_account_policies.py**: Apply cross-account resource policies to each model package group:
- For each group in MODEL_PACKAGE_GROUPS:
  - Build resource policy JSON granting PRODUCER_ACCOUNTS:
    - `sagemaker:CreateModelPackage` on `arn:aws:sagemaker:{AWS_REGION}:{REGISTRY_ACCOUNT_ID}:model-package/{group_name}/*`
    - `sagemaker:DescribeModelPackageGroup` on `arn:aws:sagemaker:{AWS_REGION}:{REGISTRY_ACCOUNT_ID}:model-package-group/{group_name}`
    - `sagemaker:DescribeModelPackage` on `arn:aws:sagemaker:{AWS_REGION}:{REGISTRY_ACCOUNT_ID}:model-package/{group_name}/*`
    - `sagemaker:ListModelPackages` on `arn:aws:sagemaker:{AWS_REGION}:{REGISTRY_ACCOUNT_ID}:model-package-group/{group_name}`
  - Build resource policy JSON granting CONSUMER_ACCOUNTS:
    - `sagemaker:DescribeModelPackageGroup` on the group ARN
    - `sagemaker:DescribeModelPackage` on `arn:aws:sagemaker:{AWS_REGION}:{REGISTRY_ACCOUNT_ID}:model-package/{group_name}/*`
    - `sagemaker:ListModelPackages` on the group ARN
  - Call `sagemaker.put_model_package_group_policy()` with the combined policy
- Print summary of applied policies per group

**register_model.py**: Register a trained model version with full lineage metadata:
- Accept parameters: model_package_group_name, model_artifact_s3_uri, inference_image_uri, training_job_arn, training_data_s3_uri, eval_metrics_s3_uri, git_commit_sha, dataset_version, experiment_run_id
- If LINEAGE_TRACKING is true, call `validate_lineage.py` to ensure all required metadata fields are present; raise ValueError if any are missing
- Build `CustomerMetadataProperties` dict with lineage fields:
  - `training_data_s3`: training data S3 URI
  - `training_job_arn`: SageMaker training job ARN
  - `eval_metrics_s3`: evaluation metrics JSON S3 URI
  - `git_commit`: Git commit SHA
  - `dataset_version`: dataset version identifier
  - `experiment_run_id`: experiment tracking run ID
  - `registered_by_account`: caller's AWS account ID
  - `registration_timestamp`: ISO 8601 timestamp
- Set `ModelApprovalStatus` to `PendingManualApproval` if APPROVAL_WORKFLOW_ENABLED, else `Approved`
- Call `sagemaker.create_model_package()` with:
  - `ModelPackageGroupName`
  - `InferenceSpecification` (container image, supported content types `application/json,text/csv`, supported instance types)
  - `ModelApprovalStatus`
  - `CustomerMetadataProperties`
  - `ModelMetrics` with `ModelQuality` pointing to eval_metrics_s3_uri
- Print model package ARN and version number
- If NOTIFICATION_EMAIL is set, publish registration event to SNS

**approve_model.py**: Approve or reject model package versions:
- Accept parameters: model_package_arn, approval_status (Approved/Rejected), approver_name, approval_reason
- Call `sagemaker.update_model_package()` with `ModelPackageArn`, `ModelApprovalStatus`, and `ApprovalDescription` containing approver name, reason, and timestamp
- Log approval decision to DynamoDB audit table via `audit_log.py`
- If approved, publish custom event to EventBridge for cross-account notification
- Print approval confirmation with model package ARN and new status

**list_models.py**: List and filter model packages across groups:
- Accept parameters: model_package_group_name (optional), approval_status_filter (optional), sort_by (CreationTime/default), max_results
- If group name provided, call `sagemaker.list_model_packages()` with `ModelPackageGroupName` and optional `ModelApprovalStatusEquals` filter
- If no group name, iterate over all MODEL_PACKAGE_GROUPS and aggregate results
- For each model package, call `sagemaker.describe_model_package()` to retrieve `CustomerMetadataProperties` and `ModelMetrics`
- Return list of model packages with: ARN, version, approval status, creation time, lineage metadata, and metrics summary
- Support pagination via `NextToken`

**lineage_metadata.py**: Build lineage metadata dict for model registration:
- `build_lineage_metadata()`: Accept training_data_s3, training_job_arn, eval_metrics_s3, git_commit, dataset_version, experiment_run_id. Return `CustomerMetadataProperties` dict with all fields plus `registration_timestamp` (ISO 8601) and `registered_by_account` (from STS `get_caller_identity()`).
- `extract_lineage_from_training_job()`: Accept training_job_arn. Call `sagemaker.describe_training_job()` to extract input data S3 URI, output model artifact S3 URI, and hyperparameters. Return partial lineage dict.
- `enrich_lineage_with_eval()`: Accept eval_metrics_s3_uri. Read evaluation metrics JSON from S3. Extract key metrics (accuracy, f1_score, precision, recall) and embed as summary in lineage metadata.

**validate_lineage.py**: Validate required lineage fields before registration:
- `validate_required_fields()`: Check that training_data_s3, training_job_arn, eval_metrics_s3, and git_commit are non-empty strings. Validate S3 URI format (`s3://`). Validate training job ARN format (`arn:aws:sagemaker:`). Validate Git commit SHA format (40-character hex). Return list of validation errors or empty list if valid.
- `validate_training_job_exists()`: Call `sagemaker.describe_training_job()` to verify the training job ARN exists and is in `Completed` status. Return validation result.

**query_catalog.py**: Query and filter the model catalog:
- `search_models()`: Accept filters dict (group_name, approval_status, min_creation_date, max_creation_date, metadata_filters). Query model packages matching filters. Return paginated results.
- `get_model_details()`: Accept model_package_arn. Return full model details including lineage metadata, metrics, approval history, and inference specification.
- `get_latest_approved()`: Accept model_package_group_name. Return the most recently approved model package in the group. Used by consumer accounts to discover the latest production-ready model.

**compare_models.py**: Compare model versions by metrics:
- `compare_versions()`: Accept list of model_package_arns. For each, retrieve `CustomerMetadataProperties` and `ModelMetrics`. Read evaluation metrics JSON from S3. Build comparison table with columns: version, approval status, accuracy, f1_score, latency_p99, training_data_size, training_duration, git_commit.
- `recommend_best()`: Accept model_package_group_name and metric_name. List all approved models in the group, compare by the specified metric, and return the model package ARN with the best metric value.

**catalog_report.py**: Generate catalog report for governance:
- `generate_report()`: Query all model package groups and their model packages. Build report with: total groups, total models, models by approval status, models by producer account, latest model per group, compliance summary (lineage completeness percentage).
- Output report as JSON to stdout and optionally upload to S3.
- Output CSV summary with columns: group_name, model_version, approval_status, producer_account, registration_date, git_commit, accuracy.

**event_bus.py**: Create custom event bus with cross-account policies in the registry account:
- Create custom event bus: `{PROJECT_NAME}-model-registry-events-{ENV}`
- Add event bus policy allowing all PRODUCER_ACCOUNTS to call `events:PutEvents` (for publishing registration events from producer accounts)
- Add event bus policy allowing all CONSUMER_ACCOUNTS to call `events:DescribeEventBus` (for verifying event bus connectivity)
- Tag event bus with Project and Environment tags

**approval_rules.py**: Create EventBridge rules for model approval events:
- Rule 1 — Model Approved: Match `SageMaker Model Package State Change` events where `ModelApprovalStatus` is `Approved`. Target: SNS topic for notification + Lambda for cross-account event forwarding.
- Rule 2 — Model Registered: Match `SageMaker Model Package State Change` events where `ModelApprovalStatus` is `PendingManualApproval`. Target: SNS topic for reviewer notification.
- Rule 3 — Model Rejected: Match `SageMaker Model Package State Change` events where `ModelApprovalStatus` is `Rejected`. Target: SNS topic for producer notification.
- All rules filter on `ModelPackageGroupName` matching any group in MODEL_PACKAGE_GROUPS.
- Rule names: `{PROJECT_NAME}-model-{status}-{ENV}`

**cross_account_forwarding.py**: Forward approval events to consumer accounts:
- For each CONSUMER_ACCOUNT:
  - Create EventBridge rule in the registry account that matches `Approved` model events
  - Add target: the consumer account's default event bus ARN (`arn:aws:events:{AWS_REGION}:{consumer_account}:event-bus/default`)
  - Create IAM role in the registry account with trust policy for `events.amazonaws.com` and permission to call `events:PutEvents` on the consumer account's event bus
- Consumer accounts must have an event bus policy allowing the registry account to put events (document this as a prerequisite)
- Print summary of forwarding rules created per consumer account

**approval_workflow.py**: SNS notification and approval gate:
- Create SNS topic: `{PROJECT_NAME}-model-approval-{ENV}` in the registry account
- Subscribe NOTIFICATION_EMAIL to the topic (if provided)
- `notify_pending_approval()`: Publish message to SNS with model package ARN, group name, producer account, registration timestamp, and evaluation metrics summary
- `notify_approval_decision()`: Publish message to SNS with approval/rejection details, approver name, and reason

**auto_approve_lambda/handler.py**: Lambda function for auto-approval:
- Triggered by EventBridge rule on `PendingManualApproval` model registration events
- Extract model package ARN from event detail
- Call `sagemaker.describe_model_package()` to retrieve `CustomerMetadataProperties`
- Read evaluation metrics from the `eval_metrics_s3` URI in `CustomerMetadataProperties`
- If AUTO_APPROVE_METRIC value >= AUTO_APPROVE_THRESHOLD: call `sagemaker.update_model_package()` with `ModelApprovalStatus='Approved'` and `ApprovalDescription` noting auto-approval with metric value
- If metric below threshold: leave as `PendingManualApproval` and publish SNS notification for manual review
- Log decision to DynamoDB audit table

**audit_log.py**: DynamoDB audit log for approval decisions:
- Create DynamoDB table: `{PROJECT_NAME}-model-approval-audit-{ENV}` with partition key `model_package_arn` (String) and sort key `timestamp` (String)
- `log_approval()`: Write item with model_package_arn, timestamp, approval_status, approver_name, approval_reason, producer_account, model_package_group, metric_values
- `get_approval_history()`: Query all approval decisions for a given model_package_arn, sorted by timestamp descending
- `get_approvals_by_group()`: Query approvals for all models in a given model package group using a GSI on `model_package_group`
- Enable TTL on the table with 2-year retention

**run_setup.py**: CLI orchestrator that runs setup steps in order:
1. Load and validate configuration
2. Create model package groups in the registry account
3. Apply cross-account resource policies to each group
4. Create custom EventBridge event bus with cross-account policies
5. Create EventBridge rules for approval events
6. Configure cross-account event forwarding to consumer accounts
7. Create SNS topic and subscriptions for approval notifications
8. Create DynamoDB audit table
9. Deploy auto-approval Lambda (if APPROVAL_WORKFLOW_ENABLED is false)
10. Print summary with group ARNs, event bus ARN, SNS topic ARN, audit table name, and forwarding rule ARNs

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints


**Centralized Registry Ownership:** The REGISTRY_ACCOUNT_ID account is the single source of truth for all model package groups. No model package groups should be created in producer or consumer accounts for models governed by this registry. Producer accounts register models into the central registry via cross-account API calls. Consumer accounts read from the central registry to discover and deploy approved models.

**Cross-Account Resource Policy Least Privilege:** Producer accounts receive write access limited to `sagemaker:CreateModelPackage` and read access (`sagemaker:DescribeModelPackage`, `sagemaker:ListModelPackages`, `sagemaker:DescribeModelPackageGroup`). Consumer accounts receive read-only access (`sagemaker:DescribeModelPackage`, `sagemaker:ListModelPackages`, `sagemaker:DescribeModelPackageGroup`). Only designated approver roles in the registry account should have `sagemaker:UpdateModelPackage` permission for approval status changes. Never grant `sagemaker:*` — enumerate specific actions.

**Lineage Metadata Enforcement:** When LINEAGE_TRACKING is true, every `create_model_package()` call must include `CustomerMetadataProperties` with at minimum: `training_data_s3`, `training_job_arn`, `eval_metrics_s3`, and `git_commit`. Registrations missing required metadata must be rejected before the API call. Validate S3 URI format, training job ARN format, and Git commit SHA format (40-character hex string). Store `registration_timestamp` and `registered_by_account` automatically.

**Approval Workflow Security:** Only IAM roles in the REGISTRY_ACCOUNT_ID should have permission to call `sagemaker:UpdateModelPackage` for changing approval status. Producer accounts can register models but cannot approve them. Consumer accounts can read models but cannot modify approval status. Approval decisions must be logged to the DynamoDB audit table with approver identity, timestamp, and justification.

**EventBridge Cross-Account:** Use event bus policies (not IAM policies) to grant cross-account `PutEvents` permission on the custom event bus. Filter events by `ModelPackageGroupName` to avoid triggering on unrelated model approvals. Forward only `Approved` status events to consumer accounts — do not forward `PendingManualApproval` or `Rejected` events to consumer accounts. Use the custom event bus in the registry account for isolation from other EventBridge traffic.

**Audit and Compliance:** Every approval and rejection decision must be logged to the DynamoDB audit table with: model_package_arn, timestamp, approval_status, approver_name, approval_reason, producer_account, model_package_group, and metric_values. The audit table must have a TTL of 2 years for compliance retention. Enable DynamoDB point-in-time recovery for the audit table. Generate audit reports on demand via `catalog_report.py`.

**Idempotency:** All setup scripts must be idempotent. If a model package group already exists, skip creation and proceed to policy application. If the event bus already exists, update policies rather than failing. If the DynamoDB audit table already exists, skip creation. Handle `ResourceInUse`, `ConflictException`, and `ResourceAlreadyExistsException` gracefully.

**Encryption:** If KMS_KEY_ARN is provided, all model artifacts stored in S3 must be encrypted with the specified KMS key. Cross-account key grants must be created for producer accounts (`kms:GenerateDataKey`, `kms:Decrypt`) and consumer accounts (`kms:Decrypt`, `kms:DescribeKey`). Use `kms:ViaService` conditions restricting key usage to `s3.{AWS_REGION}.amazonaws.com` and `sagemaker.{AWS_REGION}.amazonaws.com`.

**Naming:** All resources follow `{PROJECT_NAME}-{component}-{ENV}` convention:
- Model package groups: names from MODEL_PACKAGE_GROUPS keys (user-defined)
- Event bus: `{PROJECT_NAME}-model-registry-events-{ENV}`
- SNS topic: `{PROJECT_NAME}-model-approval-{ENV}`
- DynamoDB table: `{PROJECT_NAME}-model-approval-audit-{ENV}`
- EventBridge rules: `{PROJECT_NAME}-model-{status}-{ENV}`
- IAM roles: `{PROJECT_NAME}-registry-{purpose}-{ENV}`
- Lambda function: `{PROJECT_NAME}-auto-approve-{ENV}`
- SSM parameters: `/ml/{PROJECT_NAME}/{ENV}/registry/{parameter-name}`

**Auto-Approval Safety:** When APPROVAL_WORKFLOW_ENABLED is false and auto-approval is active, the auto-approve Lambda must still validate that the metric value is a valid number and that the evaluation metrics file exists in S3. If the metrics file is missing or unparseable, the model must remain in `PendingManualApproval` status and an SNS notification must be sent for manual review. Never auto-approve a model without verifiable metrics.

**Testing:** Test cross-account resource policies by assuming a role in a producer account and calling `sagemaker.create_model_package()` against the registry account. Test consumer access by assuming a role in a consumer account and calling `sagemaker.list_model_packages()`. Test EventBridge forwarding by approving a model and verifying the event arrives in consumer accounts. Test the auto-approve Lambda with models above and below the threshold.

---

## Code Scaffolding Hints

**Create model package groups and apply cross-account resource policies:**
```python
import boto3
import json

sagemaker = boto3.client("sagemaker", region_name=AWS_REGION)

# Create model package group in the registry account
for group_name, description in MODEL_PACKAGE_GROUPS.items():
    try:
        sagemaker.create_model_package_group(
            ModelPackageGroupName=group_name,
            ModelPackageGroupDescription=description,
            Tags=[
                {"Key": "Project", "Value": PROJECT_NAME},
                {"Key": "Environment", "Value": ENV},
                {"Key": "ManagedBy", "Value": "centralized-registry"},
                {"Key": "RegistryAccount", "Value": REGISTRY_ACCOUNT_ID},
            ],
        )
        print(f"Created model package group: {group_name}")
    except sagemaker.exceptions.ClientError as e:
        if "ResourceInUse" in str(e):
            print(f"Model package group already exists: {group_name}")
        else:
            raise

    # Build cross-account resource policy
    cross_account_policy = {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "AllowProducerAccountsRegister",
                "Effect": "Allow",
                "Principal": {
                    "AWS": [
                        f"arn:aws:iam::{acct}:root"
                        for acct in PRODUCER_ACCOUNTS
                    ]
                },
                "Action": [
                    "sagemaker:CreateModelPackage",
                    "sagemaker:DescribeModelPackageGroup",
                    "sagemaker:DescribeModelPackage",
                    "sagemaker:ListModelPackages",
                ],
                "Resource": [
                    f"arn:aws:sagemaker:{AWS_REGION}:{REGISTRY_ACCOUNT_ID}:model-package-group/{group_name}",
                    f"arn:aws:sagemaker:{AWS_REGION}:{REGISTRY_ACCOUNT_ID}:model-package/{group_name}/*",
                ],
            },
            {
                "Sid": "AllowConsumerAccountsRead",
                "Effect": "Allow",
                "Principal": {
                    "AWS": [
                        f"arn:aws:iam::{acct}:root"
                        for acct in CONSUMER_ACCOUNTS
                    ]
                },
                "Action": [
                    "sagemaker:DescribeModelPackageGroup",
                    "sagemaker:DescribeModelPackage",
                    "sagemaker:ListModelPackages",
                ],
                "Resource": [
                    f"arn:aws:sagemaker:{AWS_REGION}:{REGISTRY_ACCOUNT_ID}:model-package-group/{group_name}",
                    f"arn:aws:sagemaker:{AWS_REGION}:{REGISTRY_ACCOUNT_ID}:model-package/{group_name}/*",
                ],
            },
        ],
    }

    sagemaker.put_model_package_group_policy(
        ModelPackageGroupName=group_name,
        ResourcePolicy=json.dumps(cross_account_policy),
    )
    print(f"Applied cross-account policy to: {group_name}")
```

**Register model with lineage metadata via CustomerMetadataProperties:**
```python
import boto3
import json
from datetime import datetime, timezone

sagemaker = boto3.client("sagemaker", region_name=AWS_REGION)
sts = boto3.client("sts")

caller_identity = sts.get_caller_identity()
caller_account = caller_identity["Account"]

# Build lineage metadata
lineage_metadata = {
    "training_data_s3": "s3://myai-data-bucket/datasets/churn/v3/",
    "training_job_arn": "arn:aws:sagemaker:us-east-1:111111111111:training-job/churn-xgboost-2024-01-15",
    "eval_metrics_s3": "s3://myai-data-bucket/eval/churn-xgboost-2024-01-15/metrics.json",
    "git_commit": "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2",
    "dataset_version": "v3.2",
    "experiment_run_id": "exp-churn-xgboost-042",
    "registered_by_account": caller_account,
    "registration_timestamp": datetime.now(timezone.utc).isoformat(),
}

# Validate lineage fields (when LINEAGE_TRACKING is true)
required_fields = ["training_data_s3", "training_job_arn", "eval_metrics_s3", "git_commit"]
for field in required_fields:
    if not lineage_metadata.get(field):
        raise ValueError(f"Missing required lineage field: {field}")

# Register model package with lineage
response = sagemaker.create_model_package(
    ModelPackageGroupName="customer-churn-models",
    ModelPackageDescription="XGBoost churn model v3.2 — accuracy=0.943",
    InferenceSpecification={
        "Containers": [
            {
                "Image": f"{REGISTRY_ACCOUNT_ID}.dkr.ecr.{AWS_REGION}.amazonaws.com/{PROJECT_NAME}-inference:latest",
                "ModelDataUrl": "s3://myai-artifacts/models/churn-xgboost/model.tar.gz",
            }
        ],
        "SupportedContentTypes": ["application/json", "text/csv"],
        "SupportedResponseMIMETypes": ["application/json"],
        "SupportedRealtimeInferenceInstanceTypes": [
            "ml.m5.xlarge", "ml.m5.2xlarge", "ml.g5.xlarge"
        ],
        "SupportedTransformInstanceTypes": ["ml.m5.xlarge"],
    },
    ModelApprovalStatus="PendingManualApproval",
    CustomerMetadataProperties=lineage_metadata,
    ModelMetrics={
        "ModelQuality": {
            "Statistics": {
                "ContentType": "application/json",
                "S3Uri": lineage_metadata["eval_metrics_s3"],
            }
        }
    },
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
        {"Key": "ProducerAccount", "Value": caller_account},
    ],
)

model_package_arn = response["ModelPackageArn"]
print(f"Registered model package: {model_package_arn}")
```

**Approve model and publish cross-account event:**
```python
import boto3
import json
from datetime import datetime, timezone

sagemaker = boto3.client("sagemaker", region_name=AWS_REGION)
events = boto3.client("events", region_name=AWS_REGION)

MODEL_PACKAGE_ARN = "arn:aws:sagemaker:us-east-1:555555555555:model-package/customer-churn-models/3"

# Approve the model package
sagemaker.update_model_package(
    ModelPackageArn=MODEL_PACKAGE_ARN,
    ModelApprovalStatus="Approved",
    ApprovalDescription=json.dumps({
        "approver": "ml-lead@company.com",
        "reason": "Passed evaluation — accuracy 0.943 exceeds 0.90 threshold",
        "timestamp": datetime.now(timezone.utc).isoformat(),
    }),
)
print(f"Approved model package: {MODEL_PACKAGE_ARN}")

# Publish approval event to custom event bus for cross-account forwarding
events.put_events(
    Entries=[
        {
            "Source": f"{PROJECT_NAME}.model-registry",
            "DetailType": "Model Package Approved",
            "Detail": json.dumps({
                "model_package_arn": MODEL_PACKAGE_ARN,
                "model_package_group": "customer-churn-models",
                "approval_status": "Approved",
                "approver": "ml-lead@company.com",
                "registry_account": REGISTRY_ACCOUNT_ID,
                "project": PROJECT_NAME,
                "environment": ENV,
            }),
            "EventBusName": f"{PROJECT_NAME}-model-registry-events-{ENV}",
        }
    ],
)
print(f"Published approval event to {PROJECT_NAME}-model-registry-events-{ENV}")
```

**List and filter model packages with lineage metadata:**
```python
import boto3
import json

sagemaker = boto3.client("sagemaker", region_name=AWS_REGION)


def list_models_with_lineage(group_name, approval_status=None, max_results=20):
    """List model packages with lineage metadata from a model package group."""
    list_params = {
        "ModelPackageGroupName": group_name,
        "SortBy": "CreationTime",
        "SortOrder": "Descending",
        "MaxResults": max_results,
    }
    if approval_status:
        list_params["ModelApprovalStatusEquals"] = approval_status

    response = sagemaker.list_model_packages(**list_params)
    models = []

    for summary in response["ModelPackageSummaryList"]:
        # Get full details including lineage metadata
        detail = sagemaker.describe_model_package(
            ModelPackageName=summary["ModelPackageArn"]
        )
        model_info = {
            "model_package_arn": detail["ModelPackageArn"],
            "model_package_version": detail.get("ModelPackageVersion"),
            "approval_status": detail["ModelApprovalStatus"],
            "creation_time": detail["CreationTime"].isoformat(),
            "lineage": detail.get("CustomerMetadataProperties", {}),
            "description": detail.get("ModelPackageDescription", ""),
        }
        models.append(model_info)

    print(f"Found {len(models)} models in {group_name}")
    for m in models:
        lineage = m["lineage"]
        print(
            f"  v{m['model_package_version']} | {m['approval_status']} | "
            f"git={lineage.get('git_commit', 'N/A')[:8]} | "
            f"data={lineage.get('dataset_version', 'N/A')}"
        )
    return models


def get_latest_approved(group_name):
    """Get the most recently approved model in a group."""
    models = list_models_with_lineage(
        group_name, approval_status="Approved", max_results=1
    )
    if models:
        print(f"Latest approved: {models[0]['model_package_arn']}")
        return models[0]
    print(f"No approved models found in {group_name}")
    return None
```

**EventBridge cross-account event bus and forwarding rules:**
```python
import boto3
import json

events = boto3.client("events", region_name=AWS_REGION)

# --- Create custom event bus in the registry account ---
events.create_event_bus(
    Name=f"{PROJECT_NAME}-model-registry-events-{ENV}",
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)
print(f"Created event bus: {PROJECT_NAME}-model-registry-events-{ENV}")

# Allow producer accounts to put events
for idx, producer_acct in enumerate(PRODUCER_ACCOUNTS):
    events.put_permission(
        EventBusName=f"{PROJECT_NAME}-model-registry-events-{ENV}",
        Action="events:PutEvents",
        Principal=producer_acct,
        StatementId=f"AllowProducer{idx}PutEvents",
    )
    print(f"Granted PutEvents to producer account: {producer_acct}")

# --- Rule: forward approval events to consumer accounts ---
# Match model approval events on the default event bus (SageMaker publishes here)
for group_name in MODEL_PACKAGE_GROUPS:
    events.put_rule(
        Name=f"{PROJECT_NAME}-model-approved-{group_name}-{ENV}",
        EventPattern=json.dumps({
            "source": ["aws.sagemaker"],
            "detail-type": ["SageMaker Model Package State Change"],
            "detail": {
                "ModelPackageGroupName": [group_name],
                "ModelApprovalStatus": ["Approved"],
            },
        }),
        State="ENABLED",
        Description=f"Forward model approval events for {group_name} to consumer accounts",
    )

    # Add consumer account event buses as targets
    targets = []
    for idx, consumer_acct in enumerate(CONSUMER_ACCOUNTS):
        targets.append({
            "Id": f"Consumer{idx}-{consumer_acct}",
            "Arn": f"arn:aws:events:{AWS_REGION}:{consumer_acct}:event-bus/default",
            "RoleArn": f"arn:aws:iam::{REGISTRY_ACCOUNT_ID}:role/{PROJECT_NAME}-registry-event-forward-{ENV}",
        })

    if targets:
        events.put_targets(
            Rule=f"{PROJECT_NAME}-model-approved-{group_name}-{ENV}",
            Targets=targets,
        )
        print(f"Configured forwarding for {group_name} to {len(targets)} consumer accounts")

# --- Rule: notify on pending approval ---
events.put_rule(
    Name=f"{PROJECT_NAME}-model-pending-{ENV}",
    EventPattern=json.dumps({
        "source": ["aws.sagemaker"],
        "detail-type": ["SageMaker Model Package State Change"],
        "detail": {
            "ModelPackageGroupName": list(MODEL_PACKAGE_GROUPS.keys()),
            "ModelApprovalStatus": ["PendingManualApproval"],
        },
    }),
    State="ENABLED",
    Description=f"Notify on new model registration pending approval",
)

events.put_targets(
    Rule=f"{PROJECT_NAME}-model-pending-{ENV}",
    Targets=[
        {
            "Id": "NotifySNS",
            "Arn": f"arn:aws:sns:{AWS_REGION}:{REGISTRY_ACCOUNT_ID}:{PROJECT_NAME}-model-approval-{ENV}",
        }
    ],
)
print(f"Created pending approval notification rule")
```

**Auto-approve Lambda handler:**
```python
import boto3
import json
import os

sagemaker = boto3.client("sagemaker")
s3 = boto3.client("s3")
dynamodb = boto3.resource("dynamodb")
sns = boto3.client("sns")

AUTO_APPROVE_METRIC = os.environ.get("AUTO_APPROVE_METRIC", "accuracy")
AUTO_APPROVE_THRESHOLD = float(os.environ.get("AUTO_APPROVE_THRESHOLD", "0.90"))
AUDIT_TABLE = os.environ["AUDIT_TABLE_NAME"]
SNS_TOPIC_ARN = os.environ.get("SNS_TOPIC_ARN", "")


def handler(event, context):
    """Auto-approve model if evaluation metric exceeds threshold."""
    detail = event.get("detail", {})
    model_package_arn = detail.get("ModelPackageArn", "")

    if not model_package_arn:
        print("No ModelPackageArn in event detail")
        return {"status": "skipped", "reason": "missing_arn"}

    # Describe model package to get lineage metadata
    pkg = sagemaker.describe_model_package(ModelPackageName=model_package_arn)
    metadata = pkg.get("CustomerMetadataProperties", {})
    eval_metrics_s3 = metadata.get("eval_metrics_s3", "")

    if not eval_metrics_s3:
        print(f"No eval_metrics_s3 in metadata — leaving as PendingManualApproval")
        _notify_manual_review(model_package_arn, "Missing evaluation metrics S3 URI")
        return {"status": "pending", "reason": "missing_eval_metrics"}

    # Read evaluation metrics from S3
    try:
        bucket, key = eval_metrics_s3.replace("s3://", "").split("/", 1)
        obj = s3.get_object(Bucket=bucket, Key=key)
        metrics = json.loads(obj["Body"].read().decode("utf-8"))
    except Exception as e:
        print(f"Failed to read eval metrics: {e}")
        _notify_manual_review(model_package_arn, f"Cannot read metrics: {e}")
        return {"status": "pending", "reason": "metrics_read_error"}

    metric_value = metrics.get(AUTO_APPROVE_METRIC)
    if metric_value is None:
        print(f"Metric '{AUTO_APPROVE_METRIC}' not found in eval metrics")
        _notify_manual_review(model_package_arn, f"Metric {AUTO_APPROVE_METRIC} not found")
        return {"status": "pending", "reason": "metric_not_found"}

    metric_value = float(metric_value)

    if metric_value >= AUTO_APPROVE_THRESHOLD:
        # Auto-approve
        sagemaker.update_model_package(
            ModelPackageArn=model_package_arn,
            ModelApprovalStatus="Approved",
            ApprovalDescription=json.dumps({
                "approver": "auto-approve-lambda",
                "reason": f"{AUTO_APPROVE_METRIC}={metric_value:.4f} >= {AUTO_APPROVE_THRESHOLD}",
                "timestamp": context.invoked_function_arn,
            }),
        )
        decision = "Approved"
        print(f"Auto-approved: {model_package_arn} ({AUTO_APPROVE_METRIC}={metric_value:.4f})")
    else:
        decision = "PendingManualApproval"
        _notify_manual_review(
            model_package_arn,
            f"{AUTO_APPROVE_METRIC}={metric_value:.4f} < {AUTO_APPROVE_THRESHOLD}",
        )
        print(f"Below threshold — manual review required: {model_package_arn}")

    # Log to audit table
    table = dynamodb.Table(AUDIT_TABLE)
    from datetime import datetime, timezone
    table.put_item(
        Item={
            "model_package_arn": model_package_arn,
            "timestamp": datetime.now(timezone.utc).isoformat(),
            "approval_status": decision,
            "approver_name": "auto-approve-lambda",
            "approval_reason": f"{AUTO_APPROVE_METRIC}={metric_value:.4f} threshold={AUTO_APPROVE_THRESHOLD}",
            "model_package_group": detail.get("ModelPackageGroupName", ""),
            "metric_values": json.dumps(metrics),
        }
    )

    return {"status": decision, "metric": AUTO_APPROVE_METRIC, "value": metric_value}


def _notify_manual_review(model_package_arn, reason):
    """Send SNS notification for manual review."""
    if SNS_TOPIC_ARN:
        sns.publish(
            TopicArn=SNS_TOPIC_ARN,
            Subject="Model Requires Manual Approval",
            Message=json.dumps({
                "model_package_arn": model_package_arn,
                "reason": reason,
            }),
        )
```

**Model catalog comparison utility:**
```python
import boto3
import json

sagemaker = boto3.client("sagemaker", region_name=AWS_REGION)
s3 = boto3.client("s3")


def compare_model_versions(model_package_arns):
    """Compare multiple model versions by their evaluation metrics."""
    comparison = []

    for arn in model_package_arns:
        pkg = sagemaker.describe_model_package(ModelPackageName=arn)
        metadata = pkg.get("CustomerMetadataProperties", {})

        # Read evaluation metrics from S3
        metrics = {}
        eval_s3 = metadata.get("eval_metrics_s3", "")
        if eval_s3:
            try:
                bucket, key = eval_s3.replace("s3://", "").split("/", 1)
                obj = s3.get_object(Bucket=bucket, Key=key)
                metrics = json.loads(obj["Body"].read().decode("utf-8"))
            except Exception as e:
                print(f"Warning: could not read metrics for {arn}: {e}")

        comparison.append({
            "model_package_arn": arn,
            "version": pkg.get("ModelPackageVersion"),
            "approval_status": pkg["ModelApprovalStatus"],
            "creation_time": pkg["CreationTime"].isoformat(),
            "git_commit": metadata.get("git_commit", "N/A"),
            "dataset_version": metadata.get("dataset_version", "N/A"),
            "training_job_arn": metadata.get("training_job_arn", "N/A"),
            "accuracy": metrics.get("accuracy"),
            "f1_score": metrics.get("f1_score"),
            "precision": metrics.get("precision"),
            "recall": metrics.get("recall"),
            "latency_p99_ms": metrics.get("latency_p99_ms"),
        })

    # Print comparison table
    print(f"\n{'Version':<10} {'Status':<25} {'Accuracy':<10} {'F1':<10} {'Git':<10} {'Dataset':<10}")
    print("-" * 80)
    for m in comparison:
        print(
            f"v{m['version']:<9} {m['approval_status']:<25} "
            f"{m['accuracy'] or 'N/A':<10} {m['f1_score'] or 'N/A':<10} "
            f"{m['git_commit'][:8]:<10} {m['dataset_version']:<10}"
        )

    return comparison


def recommend_best_model(group_name, metric_name="accuracy"):
    """Recommend the best approved model in a group by a given metric."""
    response = sagemaker.list_model_packages(
        ModelPackageGroupName=group_name,
        ModelApprovalStatusEquals="Approved",
        SortBy="CreationTime",
        SortOrder="Descending",
        MaxResults=50,
    )

    best_arn = None
    best_value = -1

    for summary in response["ModelPackageSummaryList"]:
        pkg = sagemaker.describe_model_package(
            ModelPackageName=summary["ModelPackageArn"]
        )
        metadata = pkg.get("CustomerMetadataProperties", {})
        eval_s3 = metadata.get("eval_metrics_s3", "")
        if not eval_s3:
            continue

        try:
            bucket, key = eval_s3.replace("s3://", "").split("/", 1)
            obj = s3.get_object(Bucket=bucket, Key=key)
            metrics = json.loads(obj["Body"].read().decode("utf-8"))
            value = float(metrics.get(metric_name, 0))
            if value > best_value:
                best_value = value
                best_arn = summary["ModelPackageArn"]
        except Exception:
            continue

    if best_arn:
        print(f"Best model by {metric_name}: {best_arn} ({metric_name}={best_value:.4f})")
    else:
        print(f"No approved models with {metric_name} found in {group_name}")
    return best_arn, best_value
```

---

## Integration Points

- **Upstream**: `mlops/10` (Model Registry & Versioning) → Provides the single-account model registry patterns (model package groups, versioned packages, approval workflows) that this template centralizes into a shared services account with cross-account policies; producer accounts use `mlops/10` patterns locally before registering to the central registry
- **Upstream**: `mlops/17` (LLM Evaluation Pipeline) → Provides evaluation metrics (ROUGE, BLEU, BERTScore, accuracy) stored in S3 that are referenced in `CustomerMetadataProperties.eval_metrics_s3` during model registration; the auto-approve Lambda reads these metrics to make approval decisions
- **Upstream**: `devops/04` (IAM Roles & Policies) → Provides IAM role patterns for the registry account roles (event forwarding role, auto-approve Lambda execution role, DynamoDB audit table access role) and cross-account trust policies for producer and consumer account access
- **Upstream**: `devops/08` (KMS Encryption) → Provides KMS key management patterns for the optional KMS_KEY_ARN parameter; cross-account key grants for producer and consumer accounts follow the key policy patterns established in `devops/08`
- **Upstream**: `data/05` (EventBridge ML Orchestration) → Provides EventBridge patterns for ML event routing; the custom event bus and cross-account forwarding rules in this template extend the event-driven patterns from `data/05`
- **Downstream**: `enterprise/02` (Cross-Account Model Deployment) → Approved models in the central registry are consumed by cross-account deployment pipelines; consumer accounts use `sagemaker.describe_model_package()` to retrieve model artifacts and deploy to SageMaker endpoints via the pipeline defined in `enterprise/02`
- **Downstream**: `mlops/03` (LLM Inference Deployment) → Consumer accounts deploy approved models from the central registry to inference endpoints; the `get_latest_approved()` utility provides the model package ARN for deployment
- **Downstream**: `enterprise/04` (Control Tower) → Account baselines provisioned by Control Tower ensure each ML account has the IAM permissions and network connectivity to access the central registry; Config aggregator compliance data includes registry access audit
- **Related**: `enterprise/01` (Organizations SCPs) → SCPs may restrict which accounts can call `sagemaker:CreateModelPackage` or `sagemaker:UpdateModelPackage`, providing an additional governance layer beyond resource policies
- **Related**: `mlops/13` (Continuous Training) → Continuous training pipelines in producer accounts register new model versions to the central registry after each training run, creating a continuous flow of model candidates for approval