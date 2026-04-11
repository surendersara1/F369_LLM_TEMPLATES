<!-- Template Version: 1.0 | boto3: 1.35+ | sagemaker-python-sdk: 2.230+ -->

# Template 01 — SageMaker ML Training Pipeline

## 🎯 Purpose
Generate a production-ready AWS SageMaker ML training pipeline with data preprocessing, model training, hyperparameter optimization, evaluation, and S3 artifact management.

---

## 📋 Role Definition

You are an expert AWS MLOps engineer and data scientist with deep expertise in:
- Amazon SageMaker (Training Jobs, Processing Jobs, HPO, Experiments)
- SageMaker Python SDK and boto3
- ML frameworks: PyTorch, TensorFlow, Scikit-learn, XGBoost, Hugging Face
- AWS best practices for cost optimization, security, and scalability
- Python packaging for ML containers

Generate complete, production-deployable code. Do not use placeholder comments — write actual implementation.

---

## 🧩 Context & Inputs

Fill in these parameters before sending this template:

```
PROJECT_NAME:      [REQUIRED - e.g. customer-churn-predictor]
AWS_REGION:        [REQUIRED - e.g. us-east-1]
AWS_ACCOUNT_ID:    [REQUIRED - 12-digit AWS account ID]
ENV:               [REQUIRED - dev | stage | prod]

ML_FRAMEWORK:      [REQUIRED - pytorch | tensorflow | sklearn | xgboost | huggingface]
FRAMEWORK_VERSION: [REQUIRED - e.g. 2.1.0 for pytorch, 2.13 for tensorflow]
PYTHON_VERSION:    [OPTIONAL: 3.11]

TRAINING_INSTANCE_TYPE:    [OPTIONAL: ml.m5.xlarge for cpu / ml.g4dn.xlarge for gpu]
PROCESSING_INSTANCE_TYPE:  [OPTIONAL: ml.m5.large]
USE_SPOT_INSTANCES:        [OPTIONAL: true for dev/stage, false for prod]
MAX_SPOT_WAIT_SECS:        [OPTIONAL: 3600]

TRAINING_DATA_S3_PATH:     [REQUIRED - e.g. s3://my-bucket/data/training/]
VALIDATION_DATA_S3_PATH:   [OPTIONAL - e.g. s3://my-bucket/data/validation/]
OUTPUT_S3_PATH:            [OPTIONAL: s3://{PROJECT_NAME}-{AWS_ACCOUNT_ID}-{ENV}-artifacts/models/]

PRIMARY_METRIC:     [REQUIRED - e.g. validation:rmse | validation:accuracy | test:f1]
METRIC_DIRECTION:   [REQUIRED - Minimize | Maximize]

HPO_ENABLED:        [OPTIONAL: true]
HPO_MAX_JOBS:       [OPTIONAL: 10 for dev, 50 for prod]
HPO_MAX_PARALLEL:   [OPTIONAL: 2 for dev, 5 for prod]

EXPERIMENT_NAME:    [OPTIONAL: {PROJECT_NAME}-experiments]
MODEL_PACKAGE_GROUP:[OPTIONAL: {PROJECT_NAME}-models]
```

---

## 📐 Task

Generate a complete SageMaker training pipeline with ALL of the following components:

### 1. Project Structure
Create this file layout:
```
{PROJECT_NAME}/
├── pipeline/
│   ├── __init__.py
│   ├── pipeline.py              # Main SageMaker Pipeline definition
│   ├── steps/
│   │   ├── __init__.py
│   │   ├── preprocessing.py     # ProcessingStep
│   │   ├── training.py          # TrainingStep
│   │   ├── evaluation.py        # ProcessingStep for eval
│   │   └── registration.py      # ModelStep + ConditionStep
│   └── utils/
│       ├── __init__.py
│       └── helpers.py
├── scripts/
│   ├── preprocess.py            # Runs inside SageMaker Processing container
│   ├── train.py                 # Runs inside SageMaker Training container
│   └── evaluate.py              # Runs inside SageMaker Processing container
├── config/
│   ├── pipeline_config.py       # All configuration in one place
│   └── hyperparameters.json     # Default and HPO search ranges
├── requirements.txt
├── run_pipeline.py              # CLI entrypoint to trigger pipeline run
└── README.md
```

### 2. pipeline_config.py
Complete configuration dataclass with:
- All resource names following `{PROJECT_NAME}-{component}-{ENV}` convention
- Instance types per environment (dev/stage/prod)
- S3 paths for data, artifacts, monitoring
- Experiment and model registry settings
- Spot instance configuration
- IAM role ARN (from SSM Parameter Store: `/mlops/{PROJECT_NAME}/{ENV}/sagemaker-role-arn`)

### 3. scripts/preprocess.py
Full preprocessing script that:
- Reads raw data from `/opt/ml/processing/input/`
- Performs train/validation/test splits (70/15/15 default)
- Applies feature engineering (scaling, encoding, feature selection)
- Saves outputs to `/opt/ml/processing/output/{train,validation,test}/`
- Logs stats and data quality metrics
- Uses argparse for configurable split ratios and feature parameters

### 4. scripts/train.py
Full training script that:
- Reads processed data from `/opt/ml/input/data/{train,validation}/`
- Implements model training loop with the specified ML_FRAMEWORK
- Logs metrics to CloudWatch via stdout print statements matching `metric_definitions` regex patterns (e.g., `print(f"train_loss: {loss}")`, `print(f"val_accuracy: {acc}")`)
- Saves model artifacts to `/opt/ml/model/`
- Supports checkpointing for spot instance interruption recovery
- Includes early stopping logic
- Uses argparse for all hyperparameters

### 5. scripts/evaluate.py
Evaluation script that:
- Loads the trained model artifact
- Runs predictions on test set
- Computes comprehensive metrics (accuracy, F1, AUC-ROC, RMSE etc. based on task type)
- Outputs `evaluation.json` to `/opt/ml/processing/output/evaluation/`
- Format: `{"metrics": {"PRIMARY_METRIC": {"value": X, "standard_deviation": Y}}}`

### 6. pipeline/steps/preprocessing.py
SageMaker ProcessingStep using SKLearnProcessor or FrameworkProcessor:
- Instance type from config
- Input/output channel definitions
- Job name with timestamp suffix
- Container image from AWS public ECR

### 7. pipeline/steps/training.py
SageMaker TrainingStep with:
- Estimator configured for ML_FRAMEWORK
- Hyperparameter ranges for HPO (if HPO_ENABLED)
- Metric definitions for CloudWatch
- Spot instance configuration
- Checkpoint S3 path for interruption recovery
- SageMaker Experiments integration

### 8. pipeline/steps/evaluation.py
ProcessingStep for model evaluation:
- Reads model artifact + test data
- Runs evaluate.py
- PropertyFile for conditional registration

### 9. pipeline/steps/registration.py
ConditionStep + ModelStep:
- Condition: if PRIMARY_METRIC meets threshold
- ModelStep: creates SageMaker Model object
- RegisterModel: registers to MODEL_PACKAGE_GROUP with metadata
- Includes model approval status (PendingManualApproval for stage/prod, Approved for dev)

### 10. pipeline/pipeline.py
Main pipeline definition:
- Pipeline parameters (PipelineParameters for runtime overrides)
- All steps assembled in correct dependency order
- Pipeline upsert (create or update) logic
- Pipeline execution with wait option
- Cleanup helper for dev iterations

### 11. run_pipeline.py
CLI entrypoint:
```
python run_pipeline.py --env dev --wait --clean-on-fail
python run_pipeline.py --env prod --no-wait --slack-notify
```

### 12. config/hyperparameters.json
```json
{
  "default": {
    "learning_rate": 0.001,
    "batch_size": 32,
    "epochs": 10
  },
  "hpo_ranges": {
    "learning_rate": {"type": "Continuous", "min": 0.0001, "max": 0.1},
    "batch_size": {"type": "Categorical", "values": ["16", "32", "64"]},
    "epochs": {"type": "Integer", "min": 5, "max": 50}
  }
}
```

---

## 📦 Output Format

Produce ALL files listed in the project structure above. Output each file with a clear header:

### FILE: {PROJECT_NAME}/pipeline/pipeline.py
```python
[complete file content]
```

### FILE: {PROJECT_NAME}/scripts/train.py
```python
[complete file content]
```

(continue for all files)

---

## ✅ Requirements & Constraints

**Security:**
- Never hardcode AWS credentials — use IAM role from SSM Parameter Store
- All S3 paths must use server-side encryption (SSE-KMS)
- VPC configuration: use subnet IDs and security group IDs from SSM Parameter Store `/mlops/{PROJECT_NAME}/{ENV}/subnet-ids` and `/mlops/{PROJECT_NAME}/{ENV}/sg-ids`
- No public endpoints

**Cost Optimization:**
- Use spot instances for dev and stage (set `use_spot_instances=True`, `max_wait=MAX_SPOT_WAIT_SECS`)
- Enable S3 lifecycle policy for artifact bucket (30 days for dev, 90 for stage, 1 year for prod)
- Scale processing instance to minimum needed

**Reliability:**
- Spot checkpointing: save model checkpoints every epoch to S3
- Retry failed pipeline steps up to 2 times
- Pipeline idempotency: use `upsert()` not `create()`
- Metric-based conditional registration with threshold guard

**Observability:**
- SageMaker Experiments: log every training run as an experiment trial
- CloudWatch metrics for training loss, validation metric, epoch time
- Pipeline execution ARN logged to SSM for downstream reference

**Naming:**
- All resources: `{PROJECT_NAME}-{component}-{ENV}-{timestamp_suffix}`
- S3 artifact path: `s3://{PROJECT_NAME}-{AWS_ACCOUNT_ID}-{ENV}-artifacts/`

---

## 💡 Code Scaffolding Hints

**Key SageMaker SDK imports:**
```python
from sagemaker.workflow.pipeline import Pipeline
from sagemaker.workflow.steps import ProcessingStep, TrainingStep, CacheConfig
from sagemaker.workflow.model_step import ModelStep
from sagemaker.workflow.condition_step import ConditionStep
from sagemaker.workflow.conditions import ConditionGreaterThanOrEqualTo
from sagemaker.workflow.properties import PropertyFile
from sagemaker.workflow.parameters import ParameterString, ParameterFloat
from sagemaker.sklearn.processing import SKLearnProcessor
from sagemaker.processing import ProcessingInput, ProcessingOutput
from sagemaker.tuner import HyperparameterTuner, ContinuousParameter, IntegerParameter, CategoricalParameter
from sagemaker.model_metrics import MetricsSource, ModelMetrics
from sagemaker.drift_check_baselines import DriftCheckBaselines
```

**Spot instance config:**
```python
checkpoint_s3_uri = f"s3://{artifact_bucket}/checkpoints/{job_name}"
estimator = PyTorch(
    ...,
    use_spot_instances=True,
    max_wait=3600,
    max_run=3000,
    checkpoint_s3_uri=checkpoint_s3_uri,
    checkpoint_local_path="/opt/ml/checkpoints"
)
```

**Metric definition for CloudWatch:**
```python
metric_definitions = [
    {"Name": "train:loss", "Regex": "train_loss: ([0-9\.]+)"},
    {"Name": "validation:accuracy", "Regex": "val_accuracy: ([0-9\.]+)"}
]
```

**Conditional registration:**
```python
from sagemaker.workflow.conditions import ConditionGreaterThanOrEqualTo
cond_gte = ConditionGreaterThanOrEqualTo(
    left=JsonGet(step_name=eval_step.name, property_file=eval_report, json_path="metrics.accuracy.value"),
    right=0.85
)
```

---

## 🔗 Integration Points

- **Upstream**: `devops/04_iam_roles_policies_mlops.md` → provides `sagemaker_execution_role_arn`
- **Upstream**: `devops/02_vpc_networking_ml.md` → provides `subnet_ids`, `security_group_ids`
- **Upstream**: `mlops/07_feature_store.md` → training data may come from offline feature store
- **Downstream**: `mlops/05_model_monitoring_drift.md` → uses baseline dataset from this pipeline
- **Downstream**: `mlops/10_model_registry_versioning.md` → manages the registered model package
- **Downstream**: `mlops/08_sagemaker_pipelines_e2e.md` → this pipeline is a component of the E2E pipeline
- **Downstream**: `cicd/01_codebuild_ml_training.md` → CodeBuild runs `python run_pipeline.py`
