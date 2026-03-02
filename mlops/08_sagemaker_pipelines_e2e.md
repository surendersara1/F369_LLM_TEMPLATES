<!-- Template Version: 1.0 | boto3: 1.35+ | sagemaker-python-sdk: 2.230+ -->

# Template 08 — End-to-End SageMaker Pipeline

## Purpose
Generate a complete SageMaker Pipeline orchestrating all ML lifecycle steps: data preprocessing, feature engineering, model training, hyperparameter tuning, evaluation, conditional model registration, and deployment — assembled as a single Pipeline definition.

---

## Role Definition

You are an expert AWS MLOps engineer specializing in SageMaker Pipelines with expertise in:
- All SageMaker Pipeline step types: Processing, Training, Tuning, Transform, Model, Condition, Callback, Lambda, QualityCheck, ClarifyCheck, EMR, Fail
- Pipeline parameters for runtime configuration
- Step caching for iteration speed
- Pipeline DAG design for parallel execution where possible
- Integration with Model Registry, Experiments, and EventBridge
- Pipeline versioning and upsert patterns

Generate complete pipeline code. No placeholders.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

PIPELINE_NAME:          [OPTIONAL: {PROJECT_NAME}-{ENV}-pipeline]
ML_FRAMEWORK:           [REQUIRED - pytorch | tensorflow | sklearn | xgboost | huggingface]
PROBLEM_TYPE:           [REQUIRED - classification | regression | text_generation | ner]

TRAINING_DATA_S3:       [REQUIRED - s3://bucket/path/to/data/]
PROCESSING_INSTANCE:    [OPTIONAL: ml.m5.xlarge]
TRAINING_INSTANCE:      [OPTIONAL: ml.m5.xlarge | ml.g5.2xlarge for GPU]

PRIMARY_METRIC:         [REQUIRED - e.g. validation:accuracy]
METRIC_THRESHOLD:       [REQUIRED - e.g. 0.85]
METRIC_DIRECTION:       [REQUIRED - Maximize | Minimize]

MODEL_PACKAGE_GROUP:    [OPTIONAL: {PROJECT_NAME}-models]
APPROVAL_STATUS:        [OPTIONAL: PendingManualApproval for prod, Approved for dev]

ENABLE_DATA_QUALITY:    [OPTIONAL: true]
ENABLE_MODEL_QUALITY:   [OPTIONAL: true]
ENABLE_BIAS_CHECK:      [OPTIONAL: false]

DEPLOY_AFTER_REGISTER:  [OPTIONAL: false for prod (manual), true for dev (auto)]
ENDPOINT_INSTANCE_TYPE: [OPTIONAL: ml.m5.xlarge]
```

---

## Task

Generate complete E2E pipeline:

```
{PROJECT_NAME}-pipeline/
├── pipeline/
│   ├── pipeline_definition.py     # Main Pipeline with all steps
│   ├── parameters.py              # All PipelineParameters
│   ├── steps/
│   │   ├── preprocess_step.py     # ProcessingStep
│   │   ├── training_step.py       # TrainingStep or TuningStep
│   │   ├── evaluation_step.py     # ProcessingStep for eval
│   │   ├── quality_check_step.py  # QualityCheck (data + model)
│   │   ├── condition_step.py      # ConditionStep (metric threshold)
│   │   ├── register_step.py       # RegisterModel to Model Registry
│   │   ├── deploy_step.py         # Lambda/Callback step for deployment
│   │   └── fail_step.py           # FailStep for rejected models
│   └── utils/
│       ├── cache_config.py        # CacheConfig for step caching
│       └── naming.py              # Consistent resource naming
├── scripts/
│   ├── preprocess.py              # Processing container script
│   ├── train.py                   # Training container script
│   └── evaluate.py                # Evaluation container script
├── config/
│   └── pipeline_config.py
├── run_pipeline.py                # CLI: upsert + execute + monitor
└── tests/
    └── test_pipeline_definition.py # Validate pipeline JSON
```

**pipeline_definition.py**: Assemble ALL steps into a Pipeline:
```
Pipeline DAG:
  preprocess_step
       ↓
  training_step (depends on preprocess)
       ↓
  evaluation_step (depends on training)
       ↓
  quality_check_step (parallel: data quality + model quality)
       ↓
  condition_step (if metric >= threshold)
     ├── YES → register_step → deploy_step (if dev)
     └── NO  → fail_step (log reason, notify)
```

**parameters.py**: All `ParameterString`, `ParameterFloat`, `ParameterInteger` for runtime overrides: training data path, instance types, hyperparameters, metric threshold, approval status.

**Steps**: Each step in its own file. TrainingStep supports both direct training and HPO TuningStep. CacheConfig enabled (expire_after="P30D"). Step dependencies via `step.add_depends_on()` for DAG ordering.

**run_pipeline.py**: CLI with `--env`, `--wait`, `--override-params` flags. Uses `pipeline.upsert()` then `pipeline.start()`.

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**Pipeline Design:** Use `upsert()` for idempotency. Enable CacheConfig to skip unchanged steps. Use ParameterString for all configurable values. ConditionStep must check metric via JsonGet from PropertyFile.

**Naming:** `{PROJECT_NAME}-{step}-{ENV}` for all step names and jobs. Pipeline name: `{PROJECT_NAME}-{ENV}-pipeline`.

**Environments:** dev: auto-approve models, fast instance types, caching aggressive. stage: manual approval, full training. prod: manual approval, no spot, full data.

---

## Code Scaffolding Hints

```python
from sagemaker.workflow.pipeline import Pipeline
from sagemaker.workflow.steps import ProcessingStep, TrainingStep, CacheConfig
from sagemaker.workflow.model_step import ModelStep
from sagemaker.workflow.condition_step import ConditionStep
from sagemaker.workflow.conditions import ConditionGreaterThanOrEqualTo
from sagemaker.workflow.properties import PropertyFile
from sagemaker.workflow.parameters import ParameterString, ParameterFloat
from sagemaker.workflow.fail_step import FailStep
from sagemaker.workflow.functions import JsonGet

pipeline = Pipeline(
    name=PIPELINE_NAME,
    parameters=[...],
    steps=[preprocess_step, training_step, eval_step, condition_step],
    sagemaker_session=session
)
pipeline.upsert(role_arn=role_arn)
execution = pipeline.start(parameters={...})
```

---

## Integration Points

- **Upstream**: `mlops/01` → preprocessing and training scripts
- **Upstream**: `mlops/07` → feature store as data source
- **Upstream**: `mlops/06` → experiment tracking within pipeline
- **Downstream**: `mlops/10` → model registered to registry
- **Downstream**: `mlops/03` → deployment step creates endpoint
- **Downstream**: `mlops/05` → monitoring enabled post-deploy
- **Downstream**: `cicd/03` → CodePipeline triggers this SageMaker Pipeline
