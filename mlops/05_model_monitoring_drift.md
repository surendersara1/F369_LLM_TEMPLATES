<!-- Template Version: 1.0 | boto3: 1.35+ | sagemaker-python-sdk: 2.230+ -->

# Template 05 — Model Monitoring & Drift Detection

## 🎯 Purpose
Generate a production-ready model monitoring system using SageMaker Model Monitor for data quality, model quality, bias drift, and feature attribution drift detection, with CloudWatch alarms, SNS notifications, and automated retraining triggers.

---

## 📋 Role Definition

You are an expert AWS MLOps engineer specializing in ML observability with deep expertise in:
- SageMaker Model Monitor: DataQualityMonitor, ModelQualityMonitor, BiasMonitor, ExplainabilityMonitor
- SageMaker Clarify for bias detection and feature attribution (SHAP)
- CloudWatch Metrics, Alarms, Dashboards, and Log Insights
- SNS, Lambda, and EventBridge for alert routing and automated remediation
- Statistical drift detection: PSI, KS test, Chi-squared, Jensen-Shannon divergence
- Production ML system reliability engineering

Generate complete, production-deployable code.

---

## 🧩 Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

ENDPOINT_NAME:          [REQUIRED - the SageMaker endpoint to monitor]
MODEL_TYPE:             [REQUIRED - binary_classification | multiclass | regression | llm]

BASELINE_DATA_S3_PATH:  [REQUIRED - s3://bucket/path/to/training/data/baseline.csv]
                        Training data used to create the monitoring baseline

MONITORING_S3_OUTPUT:   [OPTIONAL: s3://{PROJECT_NAME}-{AWS_ACCOUNT_ID}-{ENV}-artifacts/monitoring/]
CAPTURE_S3_PATH:        [OPTIONAL: {MONITORING_S3_OUTPUT}data-capture/]

MONITORS_TO_ENABLE:     [OPTIONAL: data_quality,model_quality,bias,explainability]
                        Comma-separated list. Start with data_quality for most use cases.

SCHEDULE_EXPRESSION:    [OPTIONAL: cron(0 * ? * * *) for hourly, rate(1 day) for daily]

ALERT_EMAIL:            [OPTIONAL - email for SNS notifications]
SLACK_WEBHOOK_URL_SECRET:[OPTIONAL - Secrets Manager secret name with Slack webhook URL]
AUTO_RETRAIN_ENABLED:   [OPTIONAL: false]
AUTO_RETRAIN_THRESHOLD: [OPTIONAL: 0.1 for 10% drift threshold]

PROBLEM_TYPE:           [OPTIONAL: BinaryClassification | MulticlassClassification | Regression]
GROUND_TRUTH_S3_PATH:   [OPTIONAL - s3://bucket/path/to/ground-truth/ - needed for ModelQuality]
INFERENCE_ATTRIBUTE:    [OPTIONAL: prediction - column name in capture data]
PROBABILITY_ATTRIBUTE:  [OPTIONAL: probability - for classification endpoints]
LABEL_ATTRIBUTE:        [OPTIONAL: label - column name for ground truth]

DRIFT_THRESHOLD_PSI:    [OPTIONAL: 0.2 - PSI > 0.2 = significant drift]
ACCURACY_DROP_THRESHOLD:[OPTIONAL: 0.05 - alert if accuracy drops by > 5%]
```

---

## 📐 Task

Generate a complete model monitoring system:

### 1. Project Structure
```
{PROJECT_NAME}-monitoring/
├── monitoring/
│   ├── setup_monitoring.py          # Main setup orchestrator
│   ├── data_quality_monitor.py      # DataQualityMonitor setup
│   ├── model_quality_monitor.py     # ModelQualityMonitor setup
│   ├── bias_monitor.py              # BiasMonitor (Clarify) setup
│   ├── explainability_monitor.py    # ExplainabilityMonitor setup
│   └── data_capture.py              # Enable data capture on endpoint
├── baselines/
│   ├── create_data_baseline.py      # Run baseline job for data quality
│   ├── create_model_baseline.py     # Run baseline job for model quality
│   └── create_bias_baseline.py      # Run Clarify baseline job
├── alerts/
│   ├── cloudwatch_alarms.py         # CloudWatch alarms setup
│   ├── sns_notifications.py         # SNS topic + subscriptions
│   └── lambda_remediation/
│       ├── handler.py               # Lambda: route alerts + trigger retrain
│       └── requirements.txt
├── dashboard/
│   └── create_dashboard.py          # CloudWatch dashboard for monitoring
├── ground_truth/
│   └── merge_ground_truth.py        # Script to merge ground truth labels with captures
├── reports/
│   └── generate_report.py           # Weekly monitoring summary report
├── config/
│   └── monitoring_config.py         # All config dataclass
└── run_setup.py                      # CLI: python run_setup.py --env prod
```

### 2. monitoring/data_capture.py
Enable data capture on existing endpoint:
- `DataCaptureConfig` with capture 100% (prod) or 10% (dev) of requests
- Capture: `INPUT` and `OUTPUT` in JSON Lines format
- Destination: `CAPTURE_S3_PATH`
- Update endpoint with `UpdateEndpointInput`

### 3. baselines/create_data_baseline.py
Create statistical baseline from training data:
- `DefaultModelMonitor` for data quality
- `suggest_baseline()` to run baseline processing job
- Baseline job: ml.m5.xlarge, output to S3
- Wait for completion, log statistics and constraints file paths

### 4. monitoring/data_quality_monitor.py
Set up continuous data quality monitoring:
- `DefaultModelMonitor` with hourly/daily schedule
- Compare captured inference data against baseline
- Violations logged to S3 + CloudWatch
- Custom monitoring script option for advanced checks

### 5. monitoring/model_quality_monitor.py
Set up model quality monitoring (requires ground truth labels):
- `ModelQualityMonitor` with `PROBLEM_TYPE` config
- Monitors: accuracy, F1, AUC, RMSE (based on problem type)
- Ground truth merge job setup using `merge_ground_truth.py`
- Threshold: alert if PRIMARY_METRIC drops by > `ACCURACY_DROP_THRESHOLD`

### 6. monitoring/bias_monitor.py
Clarify bias monitoring:
- `ClarifyModelMonitor` with sensitive feature configuration
- Bias metrics: DPPL, DI, DCO, RD (Disparate Impact, etc.)
- Facet definition for protected attributes (age, gender, etc.)

### 7. alerts/cloudwatch_alarms.py
Create CloudWatch alarms:
- `monitoring/{ENDPOINT_NAME}/data_quality/violations` > 0 → ALARM
- `monitoring/{ENDPOINT_NAME}/model_quality/accuracy` < threshold → ALARM
- `SageMaker/EndpointInvocationErrors` > 10 per 5min → ALARM
- `SageMaker/ModelLatency` p99 > 2000ms → ALARM
- All alarms → SNS topic

### 8. alerts/lambda_remediation/handler.py
Lambda function triggered by SNS/EventBridge:
```python
def handler(event, context):
    # Parse monitoring violation type
    violation = parse_violation(event)

    if violation.type == "DATA_DRIFT" and violation.psi > AUTO_RETRAIN_THRESHOLD:
        trigger_retraining_pipeline()  # Start SageMaker Pipeline
        notify_slack(f"Data drift detected (PSI={violation.psi:.3f}). Retraining triggered.")

    elif violation.type == "MODEL_QUALITY":
        notify_slack(f"Model accuracy dropped to {violation.current_value:.3f}.")
        create_jira_ticket_or_sns(violation)  # Depending on config

    elif violation.type == "ENDPOINT_ERROR":
        attempt_endpoint_rollback()
        notify_pagerduty_urgent(violation)
```

### 9. dashboard/create_dashboard.py
CloudWatch dashboard with widgets:
- Model accuracy trend (7-day, 30-day)
- Data drift score (PSI) over time per feature
- Endpoint latency (p50, p95, p99)
- Invocation count and error rate
- Monitoring job status (pass/fail)

### 10. reports/generate_report.py
Weekly automated report:
- Summary of monitoring violations
- Feature drift rankings (top 5 drifted features)
- Model quality trend
- Recommended actions
- Outputs JSON + HTML report to S3
- Send via SNS email digest

---

## 📦 Output Format

Output ALL files listed above.

---

## ✅ Requirements & Constraints

**Monitoring Coverage:**
- Always enable data_quality monitoring as minimum
- model_quality requires ground truth pipeline (may have delay for batch labels)
- bias monitoring: only if model makes decisions affecting protected groups
- explainability: high compute cost, enable only for compliance-sensitive use cases

**Data Capture:**
- Prod: capture 100% of requests (full observability)
- Stage: capture 100%
- Dev: capture 10% (cost saving)
- Retention: 90 days in S3, then transition to Glacier

**Alert Fatigue Prevention:**
- Use composite alarms: trigger SNS only if 3 consecutive monitoring runs show violations
- Severity levels: INFO (log only), WARNING (Slack), CRITICAL (PagerDuty + auto-action)
- Deduplication: Lambda checks if same violation was reported in last 24h before alerting

**Auto-Retraining Safety:**
- Never auto-deploy to prod — only trigger pipeline, require human approval for prod
- Auto-retrain only if drift sustained for > 3 monitoring periods
- Log all auto-retrain triggers to DynamoDB audit table

---

## 💡 Code Scaffolding Hints

**Data Capture config:**
```python
from sagemaker.model_monitor import DataCaptureConfig
data_capture_config = DataCaptureConfig(
    enable_capture=True,
    sampling_percentage=100,
    destination_s3_uri=capture_s3_path,
    capture_options=["REQUEST", "RESPONSE"],
    csv_content_types=["text/csv"],
    json_content_types=["application/json"]
)
```

**Data Quality Monitor:**
```python
from sagemaker.model_monitor import DefaultModelMonitor
from sagemaker.model_monitor.dataset_format import DatasetFormat

monitor = DefaultModelMonitor(
    role=execution_role,
    instance_count=1,
    instance_type="ml.m5.xlarge",
    volume_size_in_gb=20,
    max_runtime_in_seconds=3600,
)
monitor.create_monitoring_schedule(
    monitor_schedule_name=f"{PROJECT_NAME}-data-quality-{ENV}",
    endpoint_input=EndpointInput(endpoint_name=ENDPOINT_NAME, destination="/opt/ml/processing/input/endpoint"),
    record_preprocessor_script=None,
    post_analytics_processor_script=None,
    output_s3_uri=monitoring_output_s3,
    statistics=monitor.baseline_statistics(),
    constraints=monitor.suggested_constraints(),
    schedule_cron_expression=SCHEDULE_EXPRESSION,
)
```

---

## 🔗 Integration Points

- **Upstream**: `mlops/03_llm_inference_deployment.md` → endpoint name to monitor
- **Upstream**: `mlops/01_sagemaker_training_pipeline.md` → baseline training data path
- **Upstream**: `devops/03_cloudwatch_monitoring_alerting.md` → shared CloudWatch dashboard
- **Downstream**: `mlops/01_sagemaker_training_pipeline.md` → trigger retraining pipeline
- **Downstream**: `cicd/03_codepipeline_multistage_ml.md` → monitoring violations feed into pipeline gates
