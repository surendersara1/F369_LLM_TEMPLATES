<!-- Template Version: 1.0 | boto3: 1.35+ | sagemaker-python-sdk: 2.230+ -->

# Template 10 — SageMaker Model Registry & Versioning

## Purpose
Generate a production-ready model registry system using SageMaker Model Registry: model package groups, versioned model packages, approval workflows, lineage tracking, deployment from registry, and automated promotion across environments.

---

## Role Definition

You are an expert AWS MLOps engineer specializing in ML model governance with expertise in:
- SageMaker Model Registry: Model Package Groups, Model Packages, Approval Status
- Model versioning, lineage tracking, and metadata management
- Approval workflows: manual approval gates for stage/prod promotion
- EventBridge integration for model approval events
- Cross-account model sharing for multi-account AWS strategies
- Model deployment from registry to SageMaker Endpoints

Generate complete, production-deployable code.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

MODEL_PACKAGE_GROUP:    [REQUIRED - e.g. customer-churn-models]
MODEL_DESCRIPTION:      [OPTIONAL: "Customer churn prediction model"]

APPROVAL_WORKFLOW:      [OPTIONAL: auto for dev, manual for stage/prod]
AUTO_APPROVE_METRIC:    [OPTIONAL - e.g. accuracy]
AUTO_APPROVE_THRESHOLD: [OPTIONAL - e.g. 0.90]

SUPPORTED_CONTENT_TYPES:[OPTIONAL: application/json,text/csv]
SUPPORTED_RESPONSE_TYPES:[OPTIONAL: application/json]

CROSS_ACCOUNT_IDS:      [OPTIONAL - comma-separated AWS account IDs for sharing]
NOTIFICATION_TOPIC_ARN: [OPTIONAL - SNS topic for approval notifications]
```

---

## Task

Generate complete model registry system:

```
model_registry/
├── registry/
│   ├── create_model_group.py      # Create Model Package Group
│   ├── register_model.py          # Register new model version
│   ├── approve_model.py           # Approve/reject model version
│   ├── list_models.py             # List all versions with status
│   └── deploy_from_registry.py    # Deploy approved model to endpoint
├── lineage/
│   ├── track_lineage.py           # Model lineage (data → training → model)
│   └── query_lineage.py           # Query model provenance
├── automation/
│   ├── auto_approval_lambda/
│   │   └── handler.py             # Lambda: auto-approve if metric > threshold
│   ├── eventbridge_rules.py       # EventBridge rules for model events
│   └── cross_account_share.py     # Share models across AWS accounts
├── config/
│   └── registry_config.py
└── run_registry.py
```

**create_model_group.py**: Create ModelPackageGroup with description, tags, cross-account resource policy.

**register_model.py**: Register model version with:
- Model artifact S3 path, container image URI
- Model metrics (ModelMetrics with evaluation report)
- Drift check baselines (DriftCheckBaselines)
- Approval status (PendingManualApproval for prod, Approved for dev)
- Metadata: git hash, training job ARN, dataset version, experiment run ID

**approve_model.py**: Update approval status via `update_model_package()`. Log approval to DynamoDB audit table.

**deploy_from_registry.py**: Deploy approved model:
- Describe model package to get artifact + container
- Create SageMaker Model
- Create EndpointConfig with prod settings
- Create/Update Endpoint
- Wait for InService + smoke test

**auto_approval_lambda/handler.py**: EventBridge → Lambda:
- Triggered on `SageMaker Model Package State Change` event
- Read evaluation metrics from model package
- If metric > threshold → approve, else → notify for manual review

**cross_account_share.py**: Resource policy for sharing model packages across accounts (dev account → prod account).

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**Versioning:** Each `register_model` call creates new version (auto-incrementing). Never overwrite existing versions. Tag each version with training metadata.

**Approval:** Dev: auto-approve all. Stage: auto-approve if metrics pass, else manual. Prod: always manual approval (human-in-the-loop).

**Audit:** Log every approval/rejection to DynamoDB: `{model_version, approver, timestamp, reason, metrics}`.

**Cross-account:** Use Model Package Group resource policy for multi-account sharing. Prod account should only deploy from shared registry.

---

## Code Scaffolding Hints

```python
from sagemaker.model_metrics import MetricsSource, ModelMetrics
model_metrics = ModelMetrics(
    model_statistics=MetricsSource(s3_uri=eval_report_s3, content_type="application/json")
)

model_package = model.register(
    content_types=["application/json"],
    response_types=["application/json"],
    inference_instances=["ml.m5.xlarge", "ml.g5.2xlarge"],
    transform_instances=["ml.m5.xlarge"],
    model_package_group_name=MODEL_PACKAGE_GROUP,
    approval_status=APPROVAL_STATUS,
    model_metrics=model_metrics,
    description=f"v{version} - accuracy={accuracy:.4f}"
)
```

---

## Integration Points

- **Upstream**: `mlops/01` → training pipeline registers models here
- **Upstream**: `mlops/08` → E2E pipeline RegisterModel step
- **Upstream**: `mlops/06` → experiment tracking provides metrics for registration
- **Downstream**: `mlops/03` → deploy approved model to endpoint
- **Downstream**: `cicd/03` → CodePipeline approval stage maps to registry approval
- **Downstream**: `mlops/05` → monitoring baseline from registered model
