<!-- Template Version: 1.0 | boto3: 1.35+ | sagemaker-python-sdk: 2.230+ -->

# Template 11 — ML Governance & Responsible AI

## Purpose
Generate a production-ready ML governance framework on AWS: SageMaker Model Cards for documentation, SageMaker Model Dashboard for centralized visibility, lineage tracking (data → training → model → endpoint), audit trails in DynamoDB + CloudTrail, compliance reporting, bias detection with Clarify, and responsible AI policies.

---

## Role Definition

You are an expert AWS ML governance engineer specializing in responsible AI with expertise in:
- SageMaker Model Cards: create, update, export, share across accounts
- SageMaker Model Dashboard: centralized model visibility, endpoint health, lineage graphs
- SageMaker Lineage Tracking: Artifacts, Actions, Contexts, Associations
- SageMaker Clarify: pre-training bias detection, post-training explainability (SHAP), fairness reports
- AWS CloudTrail: audit logging for all SageMaker API calls
- Compliance frameworks: SOC2, HIPAA, GDPR, EU AI Act model documentation requirements
- AWS Config rules for ML resource compliance
- Responsible AI: fairness, transparency, accountability, safety

Generate complete, production-deployable code.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

MODEL_PACKAGE_GROUP:    [REQUIRED - from mlops/10, e.g. customer-churn-models]
ENDPOINT_NAMES:         [OPTIONAL - comma-separated endpoints to track in dashboard]

COMPLIANCE_FRAMEWORKS:  [OPTIONAL: soc2,hipaa,gdpr,eu-ai-act]
SENSITIVE_ATTRIBUTES:   [OPTIONAL - for bias detection, e.g. age,gender,race,income_bracket]
BIAS_METRICS:           [OPTIONAL: DPPL,DI,DCA,RD,DAR,FT,TE]
                        DPPL: Difference in Positive Proportions in Labels
                        DI: Disparate Impact
                        DCA: Difference in Conditional Acceptance
                        RD: Recall Difference

RISK_RATING:            [OPTIONAL: low | medium | high | critical]
                        EU AI Act: high-risk models require full documentation
MODEL_OWNER:            [OPTIONAL - team or person responsible]
AUDIT_RETENTION_DAYS:   [OPTIONAL: 365 for prod, 90 for dev]

CROSS_ACCOUNT_SHARE:    [OPTIONAL: false - share model cards across accounts]
SHARE_ACCOUNT_IDS:      [OPTIONAL - comma-separated account IDs]
```

---

## Task

Generate complete ML governance system:

```
{PROJECT_NAME}-governance/
├── model_cards/
│   ├── create_model_card.py           # Create Model Card for a registered model
│   ├── update_model_card.py           # Update card with eval results, bias findings
│   ├── export_model_card.py           # Export as PDF for compliance review
│   ├── share_model_card.py            # Cross-account sharing
│   └── templates/
│       ├── classification_card.json   # Model Card template: classification
│       ├── regression_card.json       # Model Card template: regression
│       ├── llm_card.json              # Model Card template: LLM/generative
│       └── rag_card.json              # Model Card template: RAG system
├── lineage/
│   ├── track_lineage.py               # Record lineage: data → processing → model → endpoint
│   ├── query_lineage.py               # Query: "what data trained this model?"
│   ├── lineage_visualizer.py          # Generate lineage graph (DOT/Mermaid format)
│   └── point_in_time_recovery.py      # Recover exact model state at any timestamp
├── bias_detection/
│   ├── pre_training_bias.py           # Clarify: detect bias in training data
│   ├── post_training_bias.py          # Clarify: detect bias in model predictions
│   ├── explainability_report.py       # SHAP values for feature attribution
│   └── fairness_dashboard.py          # CloudWatch dashboard for bias metrics
├── audit/
│   ├── cloudtrail_setup.py            # CloudTrail trail for SageMaker events
│   ├── audit_logger.py                # DynamoDB audit table: approvals, deploys, changes
│   ├── compliance_report.py           # Generate compliance report (SOC2/HIPAA/GDPR)
│   └── config_rules.py                # AWS Config rules for ML resources
├── dashboard/
│   ├── model_dashboard_setup.py       # SageMaker Model Dashboard configuration
│   └── governance_dashboard.py        # Custom CloudWatch dashboard for governance KPIs
├── policies/
│   ├── responsible_ai_policy.md       # Responsible AI policy document
│   ├── model_approval_policy.md       # Who approves what, when, audit requirements
│   └── data_retention_policy.md       # Data and model artifact retention rules
├── config/
│   └── governance_config.py
└── run_governance.py                  # CLI: setup governance for a model group
```

### model_cards/create_model_card.py

Create SageMaker Model Card with comprehensive documentation:

```python
model_card_content = {
    "model_overview": {
        "model_name": model_name,
        "model_description": description,
        "model_version": version,
        "model_owner": MODEL_OWNER,
        "problem_type": "Binary Classification",
        "algorithm_type": "XGBoost / PyTorch / LLM",
        "model_artifact_uri": s3_model_path,
        "inference_environment": {"container_image": ecr_uri}
    },
    "intended_uses": {
        "purpose_of_model": "...",
        "intended_uses": ["production inference for ..."],
        "factors_affecting_model_efficiency": ["data quality", "feature drift"],
        "risk_rating": RISK_RATING,
        "explanations_for_risk_rating": "..."
    },
    "training_details": {
        "training_job_arn": training_job_arn,
        "training_data": {"s3_uri": training_data_path, "dataset_version": "v2.3"},
        "training_environment": {"instance_type": "ml.g5.2xlarge", "instance_count": 1},
        "hyperparameters": {...},
        "training_metrics": [{"name": "accuracy", "value": 0.94}]
    },
    "evaluation_details": [{
        "name": "held-out test set evaluation",
        "evaluation_job_arn": eval_job_arn,
        "datasets": ["s3://bucket/test-set/"],
        "metric_groups": [{
            "name": "Classification Metrics",
            "metric_data": [
                {"name": "accuracy", "value": 0.94, "type": "number"},
                {"name": "f1_score", "value": 0.91, "type": "number"},
                {"name": "auc_roc", "value": 0.97, "type": "number"}
            ]
        }]
    }],
    "additional_information": {
        "ethical_considerations": "...",
        "caveats_and_recommendations": "...",
        "custom_details": {"compliance_review_date": "...", "reviewer": "..."}
    }
}

sm_client.create_model_card(
    ModelCardName=f"{model_name}-card",
    Content=json.dumps(model_card_content),
    ModelCardStatus="Draft",  # Draft → PendingReview → Approved
    SecurityConfig={"KmsKeyId": kms_key_id},
    Tags=standard_tags
)
```

### lineage/track_lineage.py

Record full lineage chain using SageMaker Lineage entities:

```python
# Data Artifact → Processing Action → Processed Data → Training Action → Model Artifact → Deploy Action → Endpoint
from sagemaker.lineage.artifact import Artifact
from sagemaker.lineage.action import Action
from sagemaker.lineage.association import Association

# 1. Register raw data as artifact
raw_data = Artifact.create(
    artifact_name="training-data-v2.3",
    source_uri=training_data_s3,
    artifact_type="DataSet",
    properties={"version": "2.3", "records": "50000", "hash": data_hash}
)

# 2. Register model as artifact
model_artifact = Artifact.create(
    artifact_name=f"{model_name}-v{version}",
    source_uri=model_s3_path,
    artifact_type="Model",
    properties={"framework": "pytorch", "accuracy": "0.94"}
)

# 3. Link: data --contributed_to--> model
Association.create(
    source_arn=raw_data.artifact_arn,
    destination_arn=model_artifact.artifact_arn,
    association_type="ContributedTo"
)
```

### bias_detection/pre_training_bias.py

SageMaker Clarify pre-training bias analysis:

```python
from sagemaker.clarify import SageMakerClarifyProcessor, DataConfig, BiasConfig

clarify = SageMakerClarifyProcessor(role=role_arn, instance_count=1, instance_type="ml.m5.xlarge")
data_config = DataConfig(s3_data_input_path=training_data_s3, label="target_column",
                         s3_output_path=bias_report_s3, dataset_type="text/csv")
bias_config = BiasConfig(label_values_or_threshold=[1],
                         facet_name=SENSITIVE_ATTRIBUTES.split(","),
                         facet_values_or_threshold=[[25], [1]])  # age < 25, gender = 1
clarify.run_pre_training_bias(data_config=data_config, data_bias_config=bias_config)
```

### audit/compliance_report.py

Generate compliance report aggregating:
- Model Card content (who, what, when, why)
- Lineage graph (data provenance)
- Bias/fairness findings from Clarify
- All approval events from DynamoDB audit table
- CloudTrail events for model lifecycle
- Output: JSON + HTML + PDF report uploaded to S3

### dashboard/model_dashboard_setup.py

Configure SageMaker Model Dashboard:
- Auto-discovers all models in account
- Shows endpoint health, monitoring violations, lineage
- Link Model Cards to registered models
- Filter by project, environment, risk rating

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**Model Cards:** Every model deployed to stage/prod MUST have an approved Model Card. Card status workflow: Draft → PendingReview → Approved. Export as PDF for compliance archives.

**Lineage:** Track: raw data → processed data → model → endpoint. Every entity has properties (version, hash, timestamp). Query must answer: "What data trained the model serving endpoint X?"

**Bias:** Run Clarify pre-training bias on every dataset. Run post-training bias on every model before registration. Store reports in S3 with model artifacts. Block registration if critical bias detected.

**Audit:** DynamoDB audit table: immutable (no deletes). CloudTrail: enabled for all SageMaker API calls. Retention: AUDIT_RETENTION_DAYS. Compliance report generated monthly (automated).

**EU AI Act (if applicable):** High-risk models require: purpose documentation, training data description, bias assessment, human oversight plan, performance metrics, known limitations.

---

## Code Scaffolding Hints

```python
# Create Model Card
sm_client.create_model_card(ModelCardName=name, Content=json.dumps(content), ModelCardStatus="Draft")
# Update status
sm_client.update_model_card(ModelCardName=name, ModelCardStatus="Approved")
# Export as PDF
response = sm_client.create_model_card_export_job(ModelCardName=name, OutputConfig={"S3OutputPath": s3_path})
```

```python
# Query lineage
from sagemaker.lineage.query import LineageQuery, LineageFilter
query = LineageQuery(sagemaker_session)
results = query.query(start_arns=[model_artifact_arn],
                      query_filter=LineageFilter(entities=["Artifact"], sources=["DataSet"]))
```

---

## Integration Points

- **Upstream**: `mlops/10` → Model Registry triggers Model Card creation
- **Upstream**: `mlops/01` → training pipeline records lineage
- **Upstream**: `mlops/05` → monitoring violations feed into governance dashboard
- **Downstream**: `cicd/03` → pipeline approval gate checks Model Card status
- **Downstream**: compliance team reviews exported PDF reports
