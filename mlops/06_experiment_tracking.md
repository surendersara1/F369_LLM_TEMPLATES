<!-- Template Version: 1.0 | boto3: 1.35+ | sagemaker-python-sdk: 2.230+ | mlflow: 2.16+ -->

# Template 06 — Experiment Tracking

## Purpose
Generate experiment tracking infrastructure using SageMaker Experiments (AWS-native) OR SageMaker Managed MLflow (serverless, zero-infra) OR self-hosted MLflow on ECS (portable), with metric logging, artifact versioning, experiment comparison, and integration with SageMaker training pipelines.

---

## Role Definition

You are an expert AWS MLOps engineer specializing in ML experiment management with expertise in:
- SageMaker Experiments: Runs, Trials, TrialComponents, Tracker
- **SageMaker Managed MLflow (serverless)**: zero-infra MLflow tracking, launches in 2 min, auto-scales (NEW 2025)
- MLflow on AWS (self-hosted): tracking server on ECS/EC2, S3 artifact store, RDS backend
- Experiment comparison, hyperparameter analysis, model selection
- Integration with SageMaker Training Jobs, Pipelines, and Model Registry
- CloudWatch dashboards for experiment metrics visualization

Generate complete, production-deployable code.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

TRACKING_BACKEND:       [REQUIRED - sagemaker | managed-mlflow | mlflow]
                        sagemaker: SageMaker Experiments (zero infra, native integration)
                        managed-mlflow: SageMaker Managed MLflow (serverless, zero-infra, NEW 2025)
                        mlflow: MLflow tracking server on ECS (self-hosted, portable)

EXPERIMENT_NAME:        [REQUIRED - e.g. customer-churn-v2]
TRACKED_METRICS:        [REQUIRED - comma-separated: train_loss,val_loss,accuracy,f1,auc_roc]

# Managed MLflow-specific (only if TRACKING_BACKEND = managed-mlflow)
MLFLOW_TRACKING_SERVER_NAME: [OPTIONAL: {PROJECT_NAME}-{ENV}-mlflow]
MLFLOW_TRACKING_SERVER_SIZE: [OPTIONAL: Small for dev, Medium for stage, Large for prod]
                             Small: up to 25 concurrent users
                             Medium: up to 50 concurrent users
                             Large: up to 100 concurrent users

# Self-hosted MLflow-specific (only if TRACKING_BACKEND = mlflow)
MLFLOW_INSTANCE_TYPE:   [OPTIONAL: t3.medium for dev, m5.large for prod]
MLFLOW_RDS_INSTANCE:    [OPTIONAL: db.t3.small for dev, db.r5.large for prod]
MLFLOW_S3_BUCKET:       [OPTIONAL: {PROJECT_NAME}-{AWS_ACCOUNT_ID}-{ENV}-mlflow-artifacts]
```

---

## Task

### OPTION A: TRACKING_BACKEND = sagemaker

```
experiment_tracking/
├── tracking/
│   ├── experiment_manager.py      # Create/manage experiments and runs
│   ├── metric_logger.py           # Log metrics, params, artifacts to SageMaker
│   ├── comparison.py              # Compare runs, select best model
│   └── callbacks.py               # Training callbacks for auto-logging
├── analysis/
│   ├── experiment_dashboard.py    # CloudWatch dashboard for experiments
│   └── report_generator.py       # Generate experiment comparison report
├── integration/
│   └── pipeline_integration.py   # Hook into SageMaker Pipelines
└── config/
    └── tracking_config.py
```

**experiment_manager.py**: Create experiments, start/end runs, tag runs with metadata (git hash, dataset version, env).

**metric_logger.py**: Log metrics at step intervals, log hyperparameters, log model artifacts (S3 URIs), log custom metadata (training time, GPU utilization).

**comparison.py**: Query all runs in an experiment, rank by primary metric, generate comparison table, select best model for registration.

**callbacks.py**: `SageMakerExperimentCallback` class for PyTorch/HuggingFace trainers - auto-logs loss, lr, epoch, memory at each logging step.

### OPTION B: TRACKING_BACKEND = managed-mlflow

```
managed_mlflow/
├── setup/
│   ├── create_tracking_server.py   # Create SageMaker Managed MLflow server
│   ├── server_lifecycle.py         # Start/stop/delete server, manage size
│   └── iam_setup.py                # IAM roles for MLflow access
├── tracking/
│   ├── experiment_manager.py       # MLflow experiment and run management
│   ├── autolog_setup.py            # Framework-specific autologging
│   └── model_registry_bridge.py    # Sync MLflow Registry → SageMaker Registry
├── analysis/
│   ├── query_experiments.py        # MLflow Client API for experiment queries
│   └── comparison_report.py        # Side-by-side run comparison
└── config/
    └── managed_mlflow_config.py    # Tracking URI from server, artifact location
```

**create_tracking_server.py**: Create SageMaker Managed MLflow tracking server using `boto3.client("sagemaker").create_mlflow_tracking_server()`. Zero infrastructure — no ECS, RDS, or ALB to manage. Server launches in ~2 minutes. Auto-scales based on usage. S3 artifact store is auto-configured.

**server_lifecycle.py**: Start/stop server to save costs in dev (stopped server costs $0). Resize server (Small → Medium → Large). Export server data for migration.

**autolog_setup.py**: Same MLflow autologging as self-hosted (`mlflow.pytorch.autolog()`, etc.) — the API is identical, only the tracking URI changes. Point to `mlflow_tracking_server_arn` instead of self-hosted URL.

**model_registry_bridge.py**: Sync promoted MLflow models to SageMaker Model Registry. Managed MLflow integrates natively with SageMaker — models registered in MLflow Registry are automatically visible in SageMaker.

### OPTION C: TRACKING_BACKEND = mlflow

```
mlflow_setup/
├── infrastructure/
│   ├── mlflow_server_cdk.py       # CDK stack: ECS Fargate + RDS + S3 + ALB
│   ├── Dockerfile.mlflow          # MLflow server container
│   └── terraform_mlflow.tf        # Alternative: Terraform for MLflow server
├── tracking/
│   ├── experiment_manager.py      # MLflow experiment and run management
│   ├── autolog_setup.py           # Framework-specific autologging
│   └── model_registry_bridge.py   # Sync MLflow Registry → SageMaker Registry
├── analysis/
│   ├── query_experiments.py       # MLflow Client API for experiment queries
│   └── comparison_report.py       # Side-by-side run comparison
└── config/
    └── mlflow_config.py           # MLflow tracking URI, artifact location
```

**mlflow_server_cdk.py**: Complete CDK stack with ECS Fargate (MLflow server), RDS PostgreSQL (backend store), S3 (artifact store), ALB with HTTPS, security groups, IAM roles.

**autolog_setup.py**: `mlflow.pytorch.autolog()`, `mlflow.transformers.autolog()`, `mlflow.sklearn.autolog()` wrappers with SageMaker-compatible logging.

**model_registry_bridge.py**: Sync promoted MLflow models to SageMaker Model Registry for deployment.

---

## Output Format

Output ALL files for chosen TRACKING_BACKEND.

---

## Requirements & Constraints

**SageMaker Experiments:** Zero infrastructure cost. Limited to 10K runs per experiment. Metrics queryable via SageMaker SDK `analytics` module. Native integration with SageMaker Pipelines TrialComponent.

**SageMaker Managed MLflow (NEW 2025):** Zero infrastructure — fully serverless. Launches in 2 minutes. Auto-scales to handle concurrent users. Native S3 artifact store. Full MLflow UI accessible via presigned URL. Standard MLflow API — existing MLflow code works with zero changes (just update tracking URI). Stop server when not in use ($0 when stopped). Cost: ~$2/hour when running (Small).

**MLflow on AWS (self-hosted):** Runs as ECS Fargate service. RDS for metadata, S3 for artifacts. Cost: ~$150/month for dev, ~$500/month for prod. More portable (can migrate off AWS). Richer UI for experiment comparison. Full control over server configuration, networking, and scaling.

**Both:** Tag every run with `git_commit_hash`, `dataset_version`, `env`, `triggered_by`. Log all hyperparameters, metrics at step-level granularity, and final model artifact location.

---

## Code Scaffolding Hints

**SageMaker Experiments:**
```python
from sagemaker.experiments.run import Run
with Run(experiment_name=EXPERIMENT_NAME, run_name=f"run-{timestamp}",
         sagemaker_session=session) as run:
    run.log_parameter("learning_rate", lr)
    run.log_parameter("batch_size", batch_size)
    for epoch in range(num_epochs):
        run.log_metric("train_loss", train_loss, step=epoch)
        run.log_metric("val_accuracy", val_acc, step=epoch)
    run.log_artifact("model", model_s3_uri)
```

**SageMaker Managed MLflow:**
```python
import boto3, mlflow

# Create tracking server (one-time)
sm_client = boto3.client("sagemaker")
sm_client.create_mlflow_tracking_server(
    TrackingServerName=MLFLOW_TRACKING_SERVER_NAME,
    ArtifactStoreUri=f"s3://{PROJECT_NAME}-{AWS_ACCOUNT_ID}-{ENV}-mlflow-artifacts",
    TrackingServerSize=MLFLOW_TRACKING_SERVER_SIZE,  # "Small" | "Medium" | "Large"
    RoleArn=role_arn
)

# Get tracking server ARN and use it as the tracking URI (requires sagemaker-mlflow plugin)
# Note: Use the TrackingServerArn (not TrackingServerUrl, which is the MLflow UI URL)
tracking_server_arn = sm_client.describe_mlflow_tracking_server(
    TrackingServerName=MLFLOW_TRACKING_SERVER_NAME
)["TrackingServerArn"]
mlflow.set_tracking_uri(tracking_server_arn)
mlflow.set_experiment(EXPERIMENT_NAME)
with mlflow.start_run(run_name=f"run-{timestamp}"):
    mlflow.log_params({"learning_rate": lr, "batch_size": bs})
    mlflow.log_metrics({"train_loss": loss, "val_accuracy": acc}, step=epoch)
    mlflow.pytorch.log_model(model, "model")
```

**MLflow (self-hosted):**
```python
import mlflow
mlflow.set_tracking_uri(f"https://mlflow.{PROJECT_NAME}.internal")
mlflow.set_experiment(EXPERIMENT_NAME)
with mlflow.start_run(run_name=f"run-{timestamp}"):
    mlflow.log_params({"learning_rate": lr, "batch_size": bs})
    mlflow.log_metrics({"train_loss": loss, "val_accuracy": acc}, step=epoch)
    mlflow.pytorch.log_model(model, "model")
```

---

## Integration Points

- **Upstream**: `mlops/01` → training pipeline logs to experiments
- **Upstream**: `mlops/02` → fine-tuning logs to experiments
- **Downstream**: `mlops/10` → best run promoted to model registry
- **Downstream**: `devops/03` → experiment metrics on CloudWatch dashboard
