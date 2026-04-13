<!-- Template Version: 1.0 | boto3: 1.35+ | sagemaker-python-sdk: 2.230+ -->

# Template 13 — Continuous Training & Data Versioning

## Purpose
Generate a production-ready continuous training (CT) system: automated retraining triggered by data drift or schedule, data versioning using S3 versioning + SageMaker Lineage, dataset quality validation gates, incremental training support, and automated model comparison before promotion.

---

## Role Definition

You are an expert AWS MLOps engineer specializing in continuous training with expertise in:
- Automated retraining pipelines triggered by drift detection, schedule, or data changes
- Data versioning: S3 versioning, SageMaker Lineage Artifacts, dataset fingerprinting
- Data quality validation: Great Expectations, SageMaker Processing, Deequ
- Incremental / warm-start training for efficiency
- Champion-challenger model comparison before promotion
- EventBridge rules for pipeline orchestration
- Step Functions for complex retraining workflows with approval gates

Generate complete, production-deployable code.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

RETRAIN_TRIGGER:        [REQUIRED - schedule | drift | data-change | manual]
                        schedule: cron-based (daily, weekly, monthly)
                        drift: triggered by mlops/05 model monitoring violations
                        data-change: S3 event when new data arrives
                        manual: on-demand via API/CLI

SCHEDULE_EXPRESSION:    [OPTIONAL: rate(7 days) | cron(0 2 ? * MON *)]
DRIFT_SNS_TOPIC_ARN:    [OPTIONAL - SNS topic from mlops/05 monitoring alerts]
DATA_S3_PATH:           [OPTIONAL - s3://bucket/path/to/data/ for S3 event trigger]

SAGEMAKER_PIPELINE_NAME:[REQUIRED - existing pipeline from mlops/08]
MODEL_PACKAGE_GROUP:    [REQUIRED - from mlops/10]

TRAINING_STRATEGY:      [OPTIONAL: full | incremental | warm-start]
                        full: retrain from scratch on all data
                        incremental: train on new data only, merge with existing model
                        warm-start: start from previous model weights

DATA_VALIDATION_ENABLED:[OPTIONAL: true]
MIN_DATA_ROWS:          [OPTIONAL: 1000 - minimum rows to trigger retrain]
MAX_DATA_STALENESS_DAYS:[OPTIONAL: 30 - alert if no new data in N days]

CHAMPION_CHALLENGER:    [OPTIONAL: true - compare new model vs current before promotion]
PROMOTION_METRIC:       [OPTIONAL: accuracy]
PROMOTION_THRESHOLD:    [OPTIONAL: 0.01 - new model must beat current by at least this]

MAX_RETRAIN_FREQUENCY:  [OPTIONAL: rate(1 day) - prevent runaway retraining loops]
RETRAIN_BUDGET_MONTHLY: [OPTIONAL: 500 - USD budget cap for retraining]
```

---

## Task

Generate complete continuous training system:

```
{PROJECT_NAME}-continuous-training/
├── data_versioning/
│   ├── version_dataset.py             # Create versioned dataset snapshot
│   ├── dataset_registry.py            # DynamoDB registry of all dataset versions
│   ├── s3_versioning_setup.py         # Enable S3 versioning + lifecycle
│   ├── lineage_registration.py        # Register dataset as SageMaker Lineage Artifact
│   └── compare_datasets.py            # Diff between dataset versions (schema, stats)
├── data_validation/
│   ├── validate_schema.py             # Validate column names, types, constraints
│   ├── validate_statistics.py         # Statistical validation (distribution, nulls, ranges)
│   ├── validate_freshness.py          # Check data is recent enough
│   └── validation_report.py           # Generate data quality report
├── triggers/
│   ├── schedule_trigger.py            # EventBridge scheduled rule → Step Functions
│   ├── drift_trigger.py               # SNS subscription → Lambda → Step Functions
│   ├── s3_event_trigger.py            # S3 PutObject event → Lambda → Step Functions
│   └── manual_trigger.py              # API Gateway → Lambda → Step Functions
├── orchestration/
│   ├── retrain_state_machine.py       # Step Functions: validate → train → compare → promote
│   ├── champion_challenger.py         # Compare new model vs current production model
│   ├── promotion_decision.py          # Auto-promote or request manual approval
│   └── rollback.py                    # Rollback to previous model version
├── training/
│   ├── full_retrain.py                # Full retraining on complete dataset
│   ├── incremental_train.py           # Incremental training on new data only
│   └── warm_start_train.py            # Warm-start from previous model checkpoint
├── safeguards/
│   ├── frequency_limiter.py           # Prevent retraining more than MAX_RETRAIN_FREQUENCY
│   ├── budget_checker.py              # Check retrain cost against RETRAIN_BUDGET_MONTHLY
│   └── circuit_breaker.py             # Stop retraining if 3 consecutive failures
├── config/
│   └── ct_config.py
└── run_setup.py                       # CLI: setup all triggers and state machine
```

### data_versioning/version_dataset.py

Create immutable dataset snapshots:

```python
def version_dataset(data_s3_path: str, version_label: str):
    # 1. Compute dataset fingerprint (hash of content)
    dataset_hash = compute_s3_hash(data_s3_path)

    # 2. Copy to versioned path
    versioned_path = f"s3://{artifact_bucket}/datasets/{version_label}/{timestamp}/"
    copy_s3_prefix(data_s3_path, versioned_path)

    # 3. Record in DynamoDB registry
    dataset_table.put_item(Item={
        "dataset_id": f"{version_label}-{timestamp}",
        "version": version_label,
        "s3_path": versioned_path,
        "hash": dataset_hash,
        "row_count": count_rows(data_s3_path),
        "schema_hash": compute_schema_hash(data_s3_path),
        "created_at": timestamp,
        "created_by": "continuous-training-pipeline"
    })

    # 4. Register as SageMaker Lineage Artifact
    Artifact.create(
        artifact_name=f"dataset-{version_label}-{timestamp}",
        source_uri=versioned_path,
        artifact_type="DataSet",
        properties={"hash": dataset_hash, "rows": str(row_count)}
    )
    return versioned_path
```

### orchestration/retrain_state_machine.py

Step Functions state machine for retraining workflow:

```
State Machine: {PROJECT_NAME}-retrain-{ENV}

┌───────────────┐
│ Check Budget   │ → Over budget? → STOP + notify
└───────┬───────┘
        ▼
┌───────────────┐
│ Check Frequency│ → Too frequent? → SKIP + log
└───────┬───────┘
        ▼
┌───────────────┐
│ Version Data   │ → Snapshot current dataset
└───────┬───────┘
        ▼
┌───────────────┐
│ Validate Data  │ → Fails validation? → STOP + notify
└───────┬───────┘
        ▼
┌───────────────┐
│ Start Training │ → Invoke SageMaker Pipeline (mlops/08)
│ (Task Token)   │ → Wait for callback (async)
└───────┬───────┘
        ▼
┌───────────────┐
│ Champion vs    │ → New model vs current production model
│ Challenger     │ → Compare on held-out test set
└───────┬───────┘
        ▼
┌───────────────┐
│ Promotion      │ → Beats current by PROMOTION_THRESHOLD?
│ Decision       │ → YES: register + approve (dev) or request approval (prod)
│                │ → NO: log, keep current champion
└───────┬───────┘
        ▼
┌───────────────┐
│ Update Lineage │ → Record retrain event in lineage graph
│ + Notify       │ → SNS/Slack notification
└───────────────┘
```

### orchestration/champion_challenger.py

Compare new model against current production model:

```python
def compare_models(challenger_model_arn: str, champion_endpoint: str, test_data_s3: str):
    # 1. Deploy challenger to shadow endpoint (no live traffic)
    challenger_endpoint = deploy_shadow_endpoint(challenger_model_arn)

    # 2. Run test predictions on both
    champion_predictions = invoke_endpoint(champion_endpoint, test_data)
    challenger_predictions = invoke_endpoint(challenger_endpoint, test_data)

    # 3. Compute metrics
    champion_metrics = compute_metrics(champion_predictions, ground_truth)
    challenger_metrics = compute_metrics(challenger_predictions, ground_truth)

    # 4. Statistical significance test
    is_significantly_better = run_significance_test(champion_metrics, challenger_metrics)

    # 5. Decision
    metric_improvement = challenger_metrics[PROMOTION_METRIC] - champion_metrics[PROMOTION_METRIC]
    should_promote = is_significantly_better and metric_improvement >= PROMOTION_THRESHOLD

    # 6. Cleanup shadow endpoint
    delete_endpoint(challenger_endpoint)

    return {
        "champion_metric": champion_metrics[PROMOTION_METRIC],
        "challenger_metric": challenger_metrics[PROMOTION_METRIC],
        "improvement": metric_improvement,
        "statistically_significant": is_significantly_better,
        "promote": should_promote
    }
```

### triggers/drift_trigger.py

Lambda triggered by monitoring SNS:

```python
def handler(event, context):
    # Parse monitoring violation from SNS
    message = json.loads(event["Records"][0]["Sns"]["Message"])
    violation_type = message.get("violation_type")
    severity = message.get("severity")

    # Only retrain on sustained data drift (not transient spikes)
    if violation_type == "DATA_DRIFT" and severity in ["HIGH", "CRITICAL"]:
        # Check if drift sustained for 3+ monitoring periods
        if is_sustained_drift(endpoint_name, periods=3):
            # Trigger Step Functions retrain state machine
            sfn.start_execution(
                stateMachineArn=retrain_state_machine_arn,
                input=json.dumps({
                    "trigger": "drift_detection",
                    "violation": message,
                    "timestamp": datetime.utcnow().isoformat()
                })
            )
```

### safeguards/circuit_breaker.py

Stop retraining if repeated failures:

```python
def check_circuit_breaker():
    # Query last 3 retrain attempts from DynamoDB
    recent = get_recent_retrains(limit=3)
    failures = [r for r in recent if r["status"] == "FAILED"]

    if len(failures) >= 3:
        # Circuit open — stop retraining, alert on-call
        notify_pagerduty("Circuit breaker OPEN: 3 consecutive retrain failures")
        return False  # Do not proceed

    return True  # Circuit closed, proceed
```

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**Data Versioning:** Every retrain uses a versioned dataset snapshot (immutable). Never train on mutable data paths. DynamoDB registry tracks all versions with hash, row count, schema.

**Validation Gates:** Data must pass schema + statistical + freshness checks before training starts. Block retrain if: row count < MIN_DATA_ROWS, schema changed unexpectedly, data older than MAX_DATA_STALENESS_DAYS.

**Champion-Challenger:** New model must beat current model by PROMOTION_THRESHOLD on held-out test set with statistical significance. Never auto-promote to prod without human approval.

**Safeguards:** Frequency limiter prevents runaway loops (drift → retrain → deploy → drift → retrain...). Budget checker caps monthly retraining spend. Circuit breaker stops after 3 consecutive failures.

**Cost:** Estimate retrain cost before starting (instance type x estimated hours). Log actual cost after completion. Monthly budget alert at 80% threshold.

---

## Integration Points

- **Upstream**: `mlops/05` → drift detection triggers retraining
- **Upstream**: `mlops/08` → SageMaker Pipeline is the training engine
- **Upstream**: `mlops/07` → Feature Store provides training data
- **Downstream**: `mlops/10` → new model registered in registry
- **Downstream**: `mlops/11` → retrain event logged in governance audit
- **Downstream**: `cicd/03` → CodePipeline deploys promoted model
- **Downstream**: `data/05` → EventBridge triggers for retraining events and data change notifications
