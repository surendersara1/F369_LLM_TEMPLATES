<!-- Template Version: 1.0 | boto3: 1.35+ | sagemaker-python-sdk: 2.230+ -->

# Template DevOps 14 — SageMaker Clarify Real-Time Bias Monitoring and Explainability

## Purpose
Generate production-ready SageMaker Clarify infrastructure for real-time bias detection and model explainability: an endpoint configuration with `ClarifyExplainerConfig` for online SHAP-based feature attribution on every inference request, a `ModelBiasMonitor` schedule that periodically evaluates bias metrics (Demographic Parity Difference, Disparate Impact, KL Divergence) against a training-data baseline, bias baseline creation from the training dataset, CloudWatch alarms with SNS and EventBridge rules that trigger when bias metrics breach configurable thresholds, a bias report visualization utility that parses Clarify JSON outputs into human-readable summaries, and integration with `mlops/11` (ML Governance) for Model Cards audit trail logging of bias evaluation results.

---

## Role Definition

You are an expert AWS ML responsible AI engineer and SageMaker Clarify specialist with expertise in:
- SageMaker Clarify: `ClarifyExplainerConfig` for real-time SHAP explanations, `ModelBiasJobDefinition` for scheduled bias drift detection, `ModelBiasMonitor` monitoring schedules, bias baseline creation from training datasets
- Bias metrics: Demographic Parity Difference (DPL), Disparate Impact (DI), KL Divergence (KL), Jensen-Shannon Divergence (JS), Conditional Demographic Disparity (CDD), Class Imbalance (CI) — understanding when each metric applies and how to interpret threshold violations
- SHAP explainability: Kernel SHAP for tabular data, configurable baseline (zero, mean, median), `NumberOfSamples` tuning for latency vs accuracy tradeoff, feature attribution parsing from Clarify JSON responses
- SageMaker endpoint configuration: `create_endpoint_config()` with `ClarifyExplainerConfig` including `InferenceConfig`, `ShapConfig`, and `ShapBaselineConfig` for real-time explainability
- Bias monitoring pipelines: `create_model_bias_job_definition()` with `ModelBiasAppSpecification`, `ModelBiasJobInput`, `ModelBiasJobOutputConfig`, and `ModelBiasBaselineConfig` for scheduled bias evaluation
- CloudWatch integration: custom metrics for bias metric values, alarms on bias threshold breaches, composite alarms for multi-metric monitoring
- EventBridge automation: rules triggered by bias threshold violations that invoke remediation workflows (model rollback, traffic shifting, retraining triggers)
- Responsible AI governance: Model Cards integration for audit trail, bias evaluation history tracking, compliance reporting for fairness requirements
- IAM for Clarify: least-privilege roles for SageMaker monitoring jobs, S3 access for baseline and output data, CloudWatch metric publishing, SNS notifications

Generate complete, production-deployable bias monitoring and explainability code.

---

## Context & Inputs

```
PROJECT_NAME:               [REQUIRED]
AWS_REGION:                 [REQUIRED]
AWS_ACCOUNT_ID:             [REQUIRED]
ENV:                        [REQUIRED - dev | stage | prod]

ENDPOINT_NAME:              [REQUIRED - SageMaker endpoint to monitor for bias]
                            The inference endpoint that serves predictions.
                            Must have data capture enabled (see mlops/05).
                            Example: "credit-scoring-endpoint-prod"

BIAS_METRICS:               [REQUIRED - comma-separated list of bias metrics to evaluate]
                            Supported metrics:
                            - DPL: Demographic Parity Difference (difference in positive prediction rates)
                            - DI: Disparate Impact (ratio of positive prediction rates)
                            - KL: KL Divergence (distribution divergence between groups)
                            - JS: Jensen-Shannon Divergence (symmetric distribution divergence)
                            - CDD: Conditional Demographic Disparity
                            - CI: Class Imbalance
                            Example: "DPL,DI,KL"

BASELINE_DATASET_S3:        [REQUIRED - s3://bucket/path/to/training_data.csv]
                            Training dataset used to compute the bias baseline.
                            Must include the label column and sensitive attribute column.
                            CSV format with headers.

SENSITIVE_ATTRIBUTE:        [REQUIRED - column name of the protected attribute]
                            The feature column representing the sensitive/protected group.
                            Example: "gender", "age_group", "ethnicity"

SENSITIVE_ATTRIBUTE_VALUES: [REQUIRED - JSON list of values considered disadvantaged]
                            Values of the sensitive attribute for the disadvantaged group.
                            Example: ["female"] or ["18-25", "65+"]

LABEL_COLUMN:               [REQUIRED - column name of the target/label]
                            The ground truth label column in the baseline dataset.
                            Example: "loan_approved"

FAVORABLE_LABEL_VALUES:     [REQUIRED - JSON list of favorable outcome values]
                            Values of the label column considered favorable outcomes.
                            Example: [1] or ["approved"]

MONITORING_SCHEDULE:        [OPTIONAL: cron(0 */6 ? * * *)]
                            Schedule expression for bias monitoring job execution.
                            Default runs every 6 hours.
                            Options: cron expression or rate expression.
                            Example: "rate(1 day)" for daily, "cron(0 0 ? * MON *)" for weekly

BIAS_THRESHOLDS:            [OPTIONAL: {"DPL": 0.1, "DI": 0.8, "KL": 0.1}]
                            JSON object mapping bias metric names to threshold values.
                            DPL: alarm if |DPL| > threshold (default 0.1)
                            DI: alarm if DI < threshold (default 0.8, where 1.0 = parity)
                            KL: alarm if KL > threshold (default 0.1)
                            JS: alarm if JS > threshold (default 0.1)

EXPLAINABILITY_ENABLED:     [OPTIONAL: true]
                            Enable real-time SHAP explainability on the endpoint.
                            When true, generates ClarifyExplainerConfig for the endpoint
                            configuration. Adds latency to inference requests (~100-500ms
                            depending on NumberOfSamples).

SHAP_NUM_SAMPLES:           [OPTIONAL: 100]
                            Number of SHAP samples for Kernel SHAP computation.
                            Higher values = more accurate attributions but higher latency.
                            Recommended: 50-200 for production, 500+ for detailed analysis.

SHAP_BASELINE_METHOD:       [OPTIONAL: mean]
                            Method for computing SHAP baseline values.
                            Options: zero, mean, median, custom
                            - zero: all-zeros baseline (simplest)
                            - mean: mean of training data features (recommended)
                            - median: median of training data features
                            - custom: provide SHAP_BASELINE_S3 path

SHAP_BASELINE_S3:           [OPTIONAL: none]
                            S3 path to custom SHAP baseline CSV.
                            Required only when SHAP_BASELINE_METHOD=custom.

DATA_CAPTURE_S3:            [OPTIONAL: s3://{PROJECT_NAME}-{AWS_ACCOUNT_ID}-{ENV}-artifacts/data-capture/{ENDPOINT_NAME}]
                            S3 path where endpoint data capture files are stored.
                            Typically configured by mlops/05 (Model Monitoring).

MONITORING_OUTPUT_S3:       [OPTIONAL: s3://{PROJECT_NAME}-{AWS_ACCOUNT_ID}-{ENV}-artifacts/bias-monitoring/]
                            S3 path for bias monitoring job outputs (constraint violations,
                            statistics, and analysis results).

MODEL_CARDS_ENABLED:        [OPTIONAL: false]
                            Enable Model Cards integration for audit trail.
                            When true, bias evaluation results are logged to the
                            SageMaker Model Card associated with the endpoint's model.

SNS_TOPIC_ARN:              [OPTIONAL: none]
                            SNS topic ARN for bias alert notifications.
                            If not provided, creates a new SNS topic.

INSTANCE_TYPE:              [OPTIONAL: ml.m5.xlarge]
                            Instance type for bias monitoring processing jobs.

INSTANCE_COUNT:             [OPTIONAL: 1]
                            Number of instances for bias monitoring processing jobs.
```

---

## Task

Generate a complete Clarify real-time bias monitoring and explainability system:

```
{PROJECT_NAME}-clarify-bias-monitoring/
├── config/
│   └── config.py                         # Central configuration
├── baseline/
│   ├── create_bias_baseline.py           # Generate bias baseline from training data
│   └── baseline_config.py                # Baseline job configuration builder
├── explainability/
│   ├── clarify_endpoint_config.py        # ClarifyExplainerConfig for real-time SHAP
│   ├── shap_baseline_generator.py        # Compute SHAP baseline values
│   └── explanation_parser.py             # Parse SHAP responses from endpoint
├── monitoring/
│   ├── bias_job_definition.py            # ModelBiasJobDefinition creation
│   ├── monitoring_schedule.py            # ModelBiasMonitor schedule management
│   └── constraint_violations.py          # Parse and evaluate constraint violations
├── alerting/
│   ├── bias_alarms.py                    # CloudWatch alarms for bias thresholds
│   ├── eventbridge_rules.py              # EventBridge rules for bias breach events
│   └── sns_notifications.py              # SNS topic and notification formatting
├── reporting/
│   ├── bias_report_parser.py             # Parse Clarify JSON output into summaries
│   ├── bias_visualization.py             # Generate bias report visualizations
│   └── model_cards_integration.py        # Log bias results to Model Cards
├── infrastructure/
│   ├── iam_roles.py                      # IAM roles for monitoring jobs
│   └── ssm_outputs.py                    # Write outputs to SSM
├── run_setup.py                          # CLI orchestrator
└── requirements.txt
```

**config.py**: Central configuration dataclass with all parameters. Load from environment variables or CLI args. Validate required fields. Construct resource names using `{PROJECT_NAME}-{component}-{ENV}` convention. Parse BIAS_METRICS from comma-separated string into list. Parse BIAS_THRESHOLDS, SENSITIVE_ATTRIBUTE_VALUES, and FAVORABLE_LABEL_VALUES from JSON strings. Validate that BIAS_METRICS contains only supported values (DPL, DI, KL, JS, CDD, CI). Validate that SHAP_BASELINE_S3 is provided when SHAP_BASELINE_METHOD is "custom". Set defaults for MONITORING_SCHEDULE, BIAS_THRESHOLDS, EXPLAINABILITY_ENABLED, SHAP_NUM_SAMPLES, SHAP_BASELINE_METHOD.

**create_bias_baseline.py**: Generate bias baseline from training data:
- `create_bias_baseline_config()`: build a `ModelBiasBaselineConfig` with:
  - `BaseliningJobName`: `{PROJECT_NAME}-bias-baseline-{ENV}`
  - `ConstraintsResource`: S3 path to the generated bias constraints file
- `run_bias_baseline_job()`: use `sagemaker.create_processing_job()` to run a Clarify bias analysis on the training dataset:
  - Container image: SageMaker Clarify container for the region
  - Input: BASELINE_DATASET_S3 (training data CSV)
  - Output: `{MONITORING_OUTPUT_S3}baseline/` — produces `analysis.json` (bias metrics) and `constraints.json` (threshold constraints)
  - Processing configuration:
    - `label`: LABEL_COLUMN
    - `facet_name`: SENSITIVE_ATTRIBUTE
    - `facet_values_or_threshold`: SENSITIVE_ATTRIBUTE_VALUES
    - `favorable_label_values`: FAVORABLE_LABEL_VALUES
    - `methods`: map BIAS_METRICS to Clarify method names (DPL→`post_training_bias.dppl`, DI→`post_training_bias.di`, KL→`post_training_bias.kl`, etc.)
- `wait_for_baseline_completion()`: poll `sagemaker.describe_processing_job()` until status is `Completed` or `Failed`. Return S3 paths to output files.
- `parse_baseline_constraints(s3_path)`: download and parse `constraints.json` to extract baseline metric values. Return dict mapping metric name to baseline value.

**baseline_config.py**: Baseline job configuration builder:
- `build_analysis_config()`: construct the Clarify analysis configuration JSON:
  ```json
  {
    "label": "{LABEL_COLUMN}",
    "label_values_or_threshold": {FAVORABLE_LABEL_VALUES},
    "facet": [{"name_or_index": "{SENSITIVE_ATTRIBUTE}", "value_or_threshold": {SENSITIVE_ATTRIBUTE_VALUES}}],
    "methods": {"post_training_bias": {"methods": ["dppl", "di", "kl_divergence", ...]}}
  }
  ```
- `build_data_config()`: construct the data configuration for the baseline job specifying S3 input path, content type (CSV), label column index, and features.
- `build_model_config()`: construct the model configuration with endpoint name, content type, accept type, and instance count for shadow inference during baseline computation.

**clarify_endpoint_config.py**: ClarifyExplainerConfig for real-time SHAP (generated only if EXPLAINABILITY_ENABLED is true):
- `build_clarify_explainer_config()`: construct the `ClarifyExplainerConfig` dict:
  ```python
  {
      "InferenceConfig": {
          "FeaturesAttribute": "features",
          "ContentTemplate": '{"features": $features}',
          "MaxRecordCount": 1,
          "MaxPayloadInMB": 6
      },
      "ShapConfig": {
          "ShapBaselineConfig": {
              "MimeType": "text/csv",
              "ShapBaseline": shap_baseline_string  # or ShapBaselineUri for S3
          },
          "NumberOfSamples": SHAP_NUM_SAMPLES,
          "UseLogit": False,
          "Seed": 42
      },
      "EnableExplanations": "`true`"
  }
  ```
- `create_explainability_endpoint_config(model_name)`: call `sagemaker.create_endpoint_config()` with:
  - `EndpointConfigName`: `{PROJECT_NAME}-clarify-{ENV}`
  - `ProductionVariants`: single variant with model_name, instance type, initial instance count
  - `ExplainerConfig`: `{"ClarifyExplainerConfig": build_clarify_explainer_config()}`
- `update_endpoint_with_explainability(endpoint_name)`: call `sagemaker.update_endpoint()` to switch the endpoint to the new config with ClarifyExplainerConfig. Include a rollback function that reverts to the previous endpoint config if the update fails.

**shap_baseline_generator.py**: Compute SHAP baseline values:
- `compute_mean_baseline(dataset_s3_path)`: download training data, compute column means for numerical features and mode for categorical features. Return as CSV string.
- `compute_median_baseline(dataset_s3_path)`: download training data, compute column medians. Return as CSV string.
- `compute_zero_baseline(num_features)`: return a CSV string of zeros with the correct number of features.
- `upload_baseline_to_s3(baseline_csv, s3_path)`: upload the computed baseline to S3 for use in ShapBaselineUri.
- `get_shap_baseline()`: dispatch to the appropriate method based on SHAP_BASELINE_METHOD. If "custom", validate that SHAP_BASELINE_S3 exists.

**explanation_parser.py**: Parse SHAP responses from endpoint:
- `parse_shap_response(response_body)`: parse the JSON response from a Clarify-enabled endpoint. Extract:
  - `explanations.kernel_shap`: list of per-feature SHAP values
  - `predictions`: the model prediction
  - Feature attribution ranking (sorted by absolute SHAP value)
- `format_explanation_summary(shap_values, feature_names)`: produce a human-readable summary: "Top contributing features: feature_A (+0.32), feature_B (-0.18), feature_C (+0.11)".
- `invoke_with_explanation(endpoint_name, payload)`: call `sagemaker_runtime.invoke_endpoint()` with `EnableExplanations='`true`'` header. Parse and return both prediction and explanation.
- `batch_explain(endpoint_name, payloads)`: invoke endpoint for multiple payloads and aggregate SHAP values for population-level feature importance analysis.

**bias_job_definition.py**: ModelBiasJobDefinition creation:
- `create_bias_job_definition()`: call `sagemaker.create_model_bias_job_definition()` with:
  - `JobDefinitionName`: `{PROJECT_NAME}-bias-job-def-{ENV}`
  - `ModelBiasAppSpecification`:
    - `ImageUri`: SageMaker Clarify container URI for the region
    - `ConfigUri`: S3 path to the analysis configuration JSON
    - `Environment`: `{"analysis_type": "bias"}`
  - `ModelBiasJobInput`:
    - `EndpointInput`:
      - `EndpointName`: ENDPOINT_NAME
      - `LocalPath`: `/opt/ml/processing/input`
      - `S3InputMode`: `File`
      - `S3DataDistributionType`: `FullyReplicated`
      - `FeaturesAttribute`: `features`
      - `InferenceAttribute`: `prediction`
      - `ProbabilityAttribute`: `probability` (if classification)
    - `GroundTruthS3Input`: `{"S3Uri": ground_truth_s3_path}` (if available)
  - `ModelBiasJobOutputConfig`:
    - `MonitoringOutputs`: S3 path `{MONITORING_OUTPUT_S3}results/`
  - `ModelBiasBaselineConfig`:
    - `ConstraintsResource`: `{"S3Uri": baseline_constraints_s3_path}`
  - `RoleArn`: monitoring job IAM role ARN
  - `JobResources`: `{"ClusterConfig": {"InstanceCount": INSTANCE_COUNT, "InstanceType": INSTANCE_TYPE, "VolumeSizeInGB": 30}}`
- `delete_bias_job_definition()`: call `sagemaker.delete_model_bias_job_definition()` for cleanup.

**monitoring_schedule.py**: ModelBiasMonitor schedule management:
- `create_monitoring_schedule()`: call `sagemaker.create_monitoring_schedule()` with:
  - `MonitoringScheduleName`: `{PROJECT_NAME}-bias-monitor-{ENV}`
  - `MonitoringScheduleConfig`:
    - `ScheduleConfig`: `{"ScheduleExpression": MONITORING_SCHEDULE}`
    - `MonitoringJobDefinitionName`: bias job definition name
    - `MonitoringType`: `ModelBiasMonitor`
- `describe_monitoring_schedule()`: call `sagemaker.describe_monitoring_schedule()` to check schedule status and last execution details.
- `list_monitoring_executions()`: call `sagemaker.list_monitoring_executions()` filtered by schedule name. Return execution history with status, start time, end time, and failure reason.
- `stop_monitoring_schedule()`: call `sagemaker.stop_monitoring_schedule()` to pause monitoring.
- `start_monitoring_schedule()`: call `sagemaker.start_monitoring_schedule()` to resume monitoring.
- `delete_monitoring_schedule()`: call `sagemaker.delete_monitoring_schedule()` for cleanup.

**constraint_violations.py**: Parse and evaluate constraint violations:
- `download_latest_violations(schedule_name)`: find the most recent monitoring execution, download the `constraint_violations.json` from S3.
- `parse_violations(violations_json)`: parse the Clarify constraint violations JSON. Extract:
  - `violations`: list of `{"metric_name": str, "constraint_check_type": str, "description": str, "metric_value": float, "threshold": float}`
- `evaluate_thresholds(violations, thresholds)`: compare each violation's metric value against BIAS_THRESHOLDS. Return list of breached metrics with severity (warning if within 80% of threshold, critical if exceeded).
- `format_violation_report(violations)`: produce a formatted text report of all violations with metric name, observed value, threshold, and breach severity.
- `publish_violation_metrics(violations)`: publish each bias metric value as a CloudWatch custom metric under `{PROJECT_NAME}/BiasMonitoring` namespace with dimensions `EndpointName` and `MetricName`.

**bias_alarms.py**: CloudWatch alarms for bias thresholds:
- `create_bias_metric_alarm(metric_name, threshold, comparison)`: create a CloudWatch alarm:
  - Alarm name: `{PROJECT_NAME}-bias-{metric_name}-{ENV}`
  - Namespace: `{PROJECT_NAME}/BiasMonitoring`
  - MetricName: `{metric_name}` (e.g., `DPL`, `DI`, `KL`)
  - Dimensions: `EndpointName={ENDPOINT_NAME}`
  - For DPL/KL/JS: `ComparisonOperator=GreaterThanThreshold`, threshold from BIAS_THRESHOLDS
  - For DI: `ComparisonOperator=LessThanThreshold` (DI < 0.8 indicates bias)
  - `EvaluationPeriods`: 1
  - `Period`: derived from MONITORING_SCHEDULE frequency
  - `AlarmActions`: [SNS_TOPIC_ARN]
  - `TreatMissingData`: `notBreaching`
- `create_all_bias_alarms()`: iterate over BIAS_METRICS and create an alarm for each metric using the corresponding threshold from BIAS_THRESHOLDS.
- `create_composite_bias_alarm()`: create a composite alarm `{PROJECT_NAME}-bias-any-violation-{ENV}` that fires when ANY individual bias metric alarm is in ALARM state. Use `AlarmRule` with `OR` logic across all individual alarms.
- `delete_all_alarms()`: cleanup function to delete all bias-related alarms.

**eventbridge_rules.py**: EventBridge rules for bias breach events:
- `create_bias_breach_rule()`: create an EventBridge rule that matches CloudWatch alarm state changes for bias alarms:
  - Rule name: `{PROJECT_NAME}-bias-breach-{ENV}`
  - Event pattern:
    ```json
    {
      "source": ["aws.cloudwatch"],
      "detail-type": ["CloudWatch Alarm State Change"],
      "detail": {
        "alarmName": [{"prefix": "{PROJECT_NAME}-bias-"}],
        "state": {"value": ["ALARM"]}
      }
    }
    ```
  - Targets: SNS topic for notifications, optionally a Lambda function for automated remediation (model rollback, traffic shifting)
- `create_monitoring_completion_rule()`: create an EventBridge rule that matches SageMaker monitoring job completion events:
  - Event pattern:
    ```json
    {
      "source": ["aws.sagemaker"],
      "detail-type": ["SageMaker Model Monitor Execution Status Change"],
      "detail": {
        "MonitoringScheduleName": ["{PROJECT_NAME}-bias-monitor-{ENV}"],
        "MonitoringExecutionStatus": ["CompletedWithViolations", "Failed"]
      }
    }
    ```
  - Target: Lambda function that downloads violations, evaluates thresholds, publishes metrics, and triggers alerts.
- `create_remediation_lambda()`: generate a Lambda function that:
  1. Receives bias breach event from EventBridge
  2. Downloads the latest constraint violations
  3. Evaluates severity (warning vs critical)
  4. For critical violations: optionally trigger model rollback via `sagemaker.update_endpoint()` to a previous endpoint config
  5. Publishes detailed bias breach event to custom event bus for downstream consumers

**sns_notifications.py**: SNS topic and notification formatting:
- `create_sns_topic()`: create SNS topic `{PROJECT_NAME}-bias-alerts-{ENV}` if SNS_TOPIC_ARN not provided. Return topic ARN.
- `format_bias_alert(alarm_name, metric_name, metric_value, threshold)`: format a human-readable SNS message:
  ```
  ⚠️ Bias Alert — {PROJECT_NAME} ({ENV})
  Endpoint: {ENDPOINT_NAME}
  Metric: {metric_name}
  Value: {metric_value}
  Threshold: {threshold}
  Status: BREACHED
  Time: {timestamp}
  Action Required: Review bias monitoring results at {MONITORING_OUTPUT_S3}
  Dashboard: https://{AWS_REGION}.console.aws.amazon.com/cloudwatch/...
  ```
- `subscribe_email(topic_arn, email)`: add email subscription to the SNS topic.

**bias_report_parser.py**: Parse Clarify JSON output into summaries:
- `download_analysis_results(execution_s3_path)`: download `analysis.json` from the monitoring execution output path.
- `parse_bias_analysis(analysis_json)`: parse the Clarify analysis JSON output. Extract:
  - Per-facet bias metrics: `{metric_name: {value: float, description: str, threshold: float, violated: bool}}`
  - Dataset statistics: total records, group distribution, label distribution
  - Comparison with baseline values
- `generate_summary_report(analysis_results)`: produce a structured summary:
  - Overall bias status: PASS / WARNING / FAIL
  - Per-metric breakdown with values, thresholds, and pass/fail status
  - Group distribution comparison (training vs production)
  - Trend analysis comparing current values to previous N executions
- `generate_csv_report(analysis_results, output_path)`: write a CSV report with columns: `timestamp`, `metric_name`, `metric_value`, `threshold`, `status`, `endpoint_name`, `sensitive_attribute`.

**bias_visualization.py**: Generate bias report visualizations:
- `plot_bias_metrics_over_time(execution_history)`: query past monitoring executions, extract bias metric values, and generate a time-series plot (using matplotlib) showing metric trends with threshold lines. Save as PNG to S3.
- `plot_group_distribution(analysis_results)`: generate a bar chart comparing prediction distributions across sensitive attribute groups. Highlight disparities.
- `plot_shap_feature_importance(shap_values, feature_names)`: generate a horizontal bar chart of mean absolute SHAP values across a sample of predictions. Show top 15 features.
- `generate_html_report(analysis_results, plots_s3_paths)`: generate an HTML report combining all visualizations, metric tables, and trend analysis into a single shareable document. Upload to S3.

**model_cards_integration.py**: Log bias results to Model Cards (generated only if MODEL_CARDS_ENABLED is true):
- `get_or_create_model_card(model_name)`: call `sagemaker.describe_model_card()` to check if a Model Card exists. If not, create one using `sagemaker.create_model_card()` with initial content including model name, description, and intended uses.
- `update_model_card_with_bias_results(model_card_name, bias_results)`: call `sagemaker.update_model_card()` to append bias evaluation results to the Model Card's `EvaluationDetails` section:
  - Add evaluation entry with: evaluation name, dataset (baseline S3 path), metrics (bias metric values), timestamp, and overall status
  - Preserve existing evaluation entries (append, don't overwrite)
- `create_model_card_export(model_card_name)`: call `sagemaker.create_model_card_export_job()` to export the Model Card as a PDF for compliance documentation.
- `log_bias_audit_event(bias_results)`: write a structured audit log entry to CloudWatch Logs with bias evaluation results, endpoint name, timestamp, and pass/fail status for compliance trail.

**iam_roles.py**: IAM roles for monitoring jobs:
- Bias monitoring job role `{PROJECT_NAME}-bias-monitor-role-{ENV}`:
  - Trust: `sagemaker.amazonaws.com`
  - Permissions:
    - S3: read BASELINE_DATASET_S3, DATA_CAPTURE_S3; write MONITORING_OUTPUT_S3
    - CloudWatch Logs: create log group and write logs
    - CloudWatch: PutMetricData for bias metrics
    - SageMaker: InvokeEndpoint on ENDPOINT_NAME (for shadow inference during bias evaluation)
- Remediation Lambda role `{PROJECT_NAME}-bias-remediation-role-{ENV}`:
  - Trust: `lambda.amazonaws.com`
  - Permissions:
    - S3: read MONITORING_OUTPUT_S3
    - SageMaker: DescribeEndpoint, UpdateEndpoint, DescribeEndpointConfig
    - CloudWatch: PutMetricData
    - SNS: Publish to bias alerts topic
    - CloudWatch Logs: write own logs
- Model Cards role (if MODEL_CARDS_ENABLED):
  - Additional permissions: `sagemaker:DescribeModelCard`, `sagemaker:UpdateModelCard`, `sagemaker:CreateModelCardExportJob`

**ssm_outputs.py**: Write outputs to SSM Parameter Store:
- `/mlops/{PROJECT_NAME}/{ENV}/bias-monitor-schedule` — monitoring schedule name
- `/mlops/{PROJECT_NAME}/{ENV}/bias-baseline-constraints-s3` — S3 path to baseline constraints
- `/mlops/{PROJECT_NAME}/{ENV}/bias-alert-topic-arn` — SNS topic ARN for bias alerts
- `/mlops/{PROJECT_NAME}/{ENV}/bias-monitoring-output-s3` — S3 path for monitoring outputs
- `/mlops/{PROJECT_NAME}/{ENV}/clarify-endpoint-config` — endpoint config name with ClarifyExplainerConfig (if EXPLAINABILITY_ENABLED)

**run_setup.py**: CLI orchestrator that runs setup steps in order:
1. Load and validate configuration
2. Create IAM roles
3. Compute SHAP baseline values (if EXPLAINABILITY_ENABLED)
4. Create endpoint config with ClarifyExplainerConfig (if EXPLAINABILITY_ENABLED)
5. Update endpoint to use Clarify-enabled config (if EXPLAINABILITY_ENABLED)
6. Run bias baseline job on training dataset
7. Wait for baseline completion and parse results
8. Create model bias job definition
9. Create monitoring schedule
10. Create SNS topic (if needed)
11. Create CloudWatch alarms for each bias metric
12. Create composite bias alarm
13. Create EventBridge rules for bias breach events
14. Deploy remediation Lambda
15. Set up Model Cards integration (if MODEL_CARDS_ENABLED)
16. Write outputs to SSM
17. Print summary with monitoring schedule name, baseline metrics, alarm names, SNS topic, and dashboard link

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**Bias Metrics Interpretation:** Bias metrics have different scales and interpretations. DPL (Demographic Parity Difference) ranges from -1 to 1, where 0 indicates parity; values > 0.1 or < -0.1 typically indicate meaningful bias. DI (Disparate Impact) is a ratio where 1.0 indicates parity; the "four-fifths rule" considers DI < 0.8 as evidence of adverse impact. KL Divergence and JS Divergence are non-negative, where 0 indicates identical distributions; values > 0.1 suggest meaningful divergence. Thresholds must be configured per metric type — a single threshold value does not apply uniformly across all metrics.

**Real-Time Explainability Latency:** Enabling `ClarifyExplainerConfig` on an endpoint adds latency to every inference request. Kernel SHAP computation requires multiple model invocations (controlled by `NumberOfSamples`). With 100 samples, expect 100-500ms additional latency per request. For latency-sensitive applications, consider: (1) reducing `NumberOfSamples` to 25-50, (2) enabling explanations only for a sample of requests using the `EnableExplanations` parameter with a conditional expression, or (3) running explanations asynchronously via a separate endpoint. The SHAP baseline significantly affects attribution quality — use mean or median of training data rather than zeros for more meaningful explanations.

**Monitoring Schedule and Data Requirements:** The `ModelBiasMonitor` requires sufficient data capture volume between monitoring executions to produce statistically meaningful bias metrics. If the endpoint receives fewer than ~200 requests between monitoring runs, bias metric estimates will have high variance. Adjust MONITORING_SCHEDULE frequency based on traffic volume: high-traffic endpoints (>1000 req/hr) can use hourly monitoring; low-traffic endpoints should use daily or weekly schedules. The monitoring job processes data capture files from S3 — ensure data capture is enabled on the endpoint (see `mlops/05`).

**Baseline Drift vs Absolute Bias:** The `ModelBiasMonitor` compares current bias metrics against the baseline computed from training data. This detects bias *drift* (change from training-time bias levels), not absolute bias. If the training data itself contains bias, the baseline will reflect that bias, and the monitor will only alert on *changes* from that biased baseline. For absolute bias detection, set BIAS_THRESHOLDS to absolute fairness targets (e.g., DPL threshold of 0.05) rather than relying solely on baseline comparison. Consider running a one-time bias audit on the training data using `create_bias_baseline.py` and reviewing the baseline metrics before deploying monitoring.

**Ground Truth Availability:** Full bias monitoring accuracy requires ground truth labels. If ground truth is available (e.g., loan default outcomes after 6 months), configure the monitoring job with `GroundTruthS3Input` for post-training bias metrics. If ground truth is not available, the monitor can still compute pre-training bias metrics based on prediction distributions across groups, but these are less informative. The `constraint_violations.py` module handles both cases.

**Model Cards Compliance:** When MODEL_CARDS_ENABLED is true, bias evaluation results are appended to the SageMaker Model Card's `EvaluationDetails` section. This creates an auditable history of bias evaluations over time. Model Cards support export to PDF for compliance documentation. The integration preserves existing Model Card content and appends new evaluation entries — it never overwrites previous results. This supports regulatory requirements (EU AI Act, NIST AI RMF) that mandate ongoing bias monitoring documentation.

**Resource Naming:** All generated AWS resources follow the `{PROJECT_NAME}-{component}-{ENV}` convention. Monitoring schedule: `{PROJECT_NAME}-bias-monitor-{ENV}`. Job definition: `{PROJECT_NAME}-bias-job-def-{ENV}`. Alarms: `{PROJECT_NAME}-bias-{metric}-{ENV}`. SNS topic: `{PROJECT_NAME}-bias-alerts-{ENV}`. IAM roles: `{PROJECT_NAME}-bias-monitor-role-{ENV}`.

**Cost Considerations:** Bias monitoring jobs run SageMaker Processing instances (INSTANCE_TYPE × INSTANCE_COUNT) for each scheduled execution. With ml.m5.xlarge at ~$0.23/hr and a typical 10-15 minute job, each execution costs ~$0.04-0.06. Hourly monitoring = ~$1/day; daily monitoring = ~$0.05/day. Real-time SHAP explanations do not incur additional instance costs but increase inference latency and may require larger endpoint instances to maintain throughput. CloudWatch custom metrics cost $0.30/metric/month. Budget accordingly based on monitoring frequency and number of bias metrics tracked.

---

## Code Scaffolding Hints

### Endpoint Config with ClarifyExplainerConfig

```python
import boto3

sagemaker = boto3.client("sagemaker")

# Create endpoint config with real-time SHAP explainability
endpoint_config_response = sagemaker.create_endpoint_config(
    EndpointConfigName=f"{PROJECT_NAME}-clarify-{ENV}",
    ProductionVariants=[
        {
            "VariantName": "AllTraffic",
            "ModelName": model_name,
            "InstanceType": "ml.m5.xlarge",
            "InitialInstanceCount": 1,
            "InitialVariantWeight": 1.0,
        }
    ],
    ExplainerConfig={
        "ClarifyExplainerConfig": {
            "InferenceConfig": {
                "FeaturesAttribute": "features",
                "ContentTemplate": '{"features": $features}',
                "MaxRecordCount": 1,
                "MaxPayloadInMB": 6,
            },
            "ShapConfig": {
                "ShapBaselineConfig": {
                    "MimeType": "text/csv",
                    "ShapBaseline": "0.0,0.0,0.0,0.0,0.0",  # or ShapBaselineUri for S3
                },
                "NumberOfSamples": 100,
                "UseLogit": False,
                "Seed": 42,
            },
            "EnableExplanations": "`true`",  # backtick-wrapped JMESPath expression
        }
    },
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)

# Invoke endpoint with explanations enabled
sagemaker_runtime = boto3.client("sagemaker-runtime")
response = sagemaker_runtime.invoke_endpoint(
    EndpointName=ENDPOINT_NAME,
    ContentType="application/json",
    Accept="application/json",
    Body='{"features": [0.5, 1.2, 0.0, 3.4, 0.8]}',
    EnableExplanations="`true`",
)

import json
result = json.loads(response["Body"].read())
predictions = result["predictions"]
explanations = result["explanations"]["kernel_shap"]
# explanations[0]["feature_attributions"] = [0.12, -0.08, 0.01, 0.25, -0.03]
```

### Model Bias Job Definition

```python
# Create model bias job definition for scheduled monitoring
bias_job_response = sagemaker.create_model_bias_job_definition(
    JobDefinitionName=f"{PROJECT_NAME}-bias-job-def-{ENV}",
    ModelBiasAppSpecification={
        "ImageUri": f"306415355426.dkr.ecr.{AWS_REGION}.amazonaws.com/sagemaker-clarify-processing:1.0",
        "ConfigUri": f"s3://{PROJECT_NAME}-artifacts-{ENV}/bias-config/analysis_config.json",
        "Environment": {"analysis_type": "bias"},
    },
    ModelBiasJobInput={
        "EndpointInput": {
            "EndpointName": ENDPOINT_NAME,
            "LocalPath": "/opt/ml/processing/input",
            "S3InputMode": "File",
            "S3DataDistributionType": "FullyReplicated",
            "FeaturesAttribute": "features",
            "InferenceAttribute": "prediction",
            "ProbabilityAttribute": "probability",
        }
    },
    ModelBiasJobOutputConfig={
        "MonitoringOutputs": [
            {
                "S3Output": {
                    "S3Uri": f"s3://{PROJECT_NAME}-artifacts-{ENV}/bias-monitoring/results/",
                    "LocalPath": "/opt/ml/processing/output",
                    "S3UploadMode": "EndOfJob",
                }
            }
        ]
    },
    ModelBiasBaselineConfig={
        "BaseliningJobName": f"{PROJECT_NAME}-bias-baseline-{ENV}",
        "ConstraintsResource": {
            "S3Uri": f"s3://{PROJECT_NAME}-artifacts-{ENV}/bias-monitoring/baseline/constraints.json"
        },
    },
    JobResources={
        "ClusterConfig": {
            "InstanceCount": 1,
            "InstanceType": "ml.m5.xlarge",
            "VolumeSizeInGB": 30,
        }
    },
    RoleArn=f"arn:aws:iam::{AWS_ACCOUNT_ID}:role/{PROJECT_NAME}-bias-monitor-role-{ENV}",
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)
```

### Monitoring Schedule with ModelBiasMonitor

```python
# Create monitoring schedule for periodic bias evaluation
schedule_response = sagemaker.create_monitoring_schedule(
    MonitoringScheduleName=f"{PROJECT_NAME}-bias-monitor-{ENV}",
    MonitoringScheduleConfig={
        "ScheduleConfig": {
            "ScheduleExpression": "cron(0 */6 ? * * *)",  # every 6 hours
        },
        "MonitoringJobDefinitionName": f"{PROJECT_NAME}-bias-job-def-{ENV}",
        "MonitoringType": "ModelBiasMonitor",
    },
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)

# Check monitoring execution history
executions = sagemaker.list_monitoring_executions(
    MonitoringScheduleName=f"{PROJECT_NAME}-bias-monitor-{ENV}",
    SortBy="CreationTime",
    SortOrder="Descending",
    MaxResults=5,
)
for execution in executions["MonitoringExecutionSummaries"]:
    print(f"Status: {execution['MonitoringExecutionStatus']}, "
          f"Time: {execution['CreationTime']}, "
          f"Failure: {execution.get('FailureReason', 'N/A')}")
```

### Clarify JSON Output Parsing

```python
import json

# Parse bias analysis output from monitoring job
def parse_bias_analysis(analysis_json_path):
    with open(analysis_json_path, "r") as f:
        analysis = json.load(f)

    results = {}
    # Post-training bias metrics are under "post_training_bias_metrics"
    bias_metrics = analysis.get("post_training_bias_metrics", {})
    facets = bias_metrics.get("facets", {})

    for facet_name, facet_data in facets.items():
        for facet_value_group in facet_data:
            metrics = facet_value_group.get("metrics", [])
            for metric in metrics:
                metric_name = metric["name"]  # e.g., "DPL", "DI"
                metric_value = metric["value"]
                metric_desc = metric["description"]
                results[metric_name] = {
                    "value": metric_value,
                    "description": metric_desc,
                    "facet": facet_name,
                }
    return results


# Parse constraint violations from monitoring execution
def parse_constraint_violations(violations_json_path):
    with open(violations_json_path, "r") as f:
        violations = json.load(f)

    breaches = []
    for violation in violations.get("violations", []):
        breaches.append({
            "metric_name": violation["metric_name"],
            "constraint_check_type": violation["constraint_check_type"],
            "description": violation["description"],
        })
    return breaches


# Publish bias metrics to CloudWatch
cloudwatch = boto3.client("cloudwatch")

def publish_bias_metrics(bias_results, endpoint_name, namespace):
    metric_data = []
    for metric_name, metric_info in bias_results.items():
        metric_data.append({
            "MetricName": metric_name,
            "Dimensions": [
                {"Name": "EndpointName", "Value": endpoint_name},
                {"Name": "SensitiveAttribute", "Value": metric_info["facet"]},
            ],
            "Value": metric_info["value"],
            "Unit": "None",
        })

    cloudwatch.put_metric_data(
        Namespace=namespace,
        MetricData=metric_data,
    )
```

### CloudWatch Alarm for Bias Threshold

```python
# Create alarm for Demographic Parity Difference exceeding threshold
cloudwatch.put_metric_alarm(
    AlarmName=f"{PROJECT_NAME}-bias-DPL-{ENV}",
    AlarmDescription="Demographic Parity Difference exceeds acceptable threshold",
    Namespace=f"{PROJECT_NAME}/BiasMonitoring",
    MetricName="DPL",
    Dimensions=[
        {"Name": "EndpointName", "Value": ENDPOINT_NAME},
    ],
    Statistic="Maximum",
    Period=21600,  # 6 hours (matches monitoring schedule)
    EvaluationPeriods=1,
    Threshold=0.1,
    ComparisonOperator="GreaterThanThreshold",
    TreatMissingData="notBreaching",
    AlarmActions=[sns_topic_arn],
    OKActions=[sns_topic_arn],
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)

# Create alarm for Disparate Impact below threshold (DI < 0.8 = bias)
cloudwatch.put_metric_alarm(
    AlarmName=f"{PROJECT_NAME}-bias-DI-{ENV}",
    AlarmDescription="Disparate Impact ratio below four-fifths rule threshold",
    Namespace=f"{PROJECT_NAME}/BiasMonitoring",
    MetricName="DI",
    Dimensions=[
        {"Name": "EndpointName", "Value": ENDPOINT_NAME},
    ],
    Statistic="Minimum",
    Period=21600,
    EvaluationPeriods=1,
    Threshold=0.8,
    ComparisonOperator="LessThanThreshold",
    TreatMissingData="notBreaching",
    AlarmActions=[sns_topic_arn],
    OKActions=[sns_topic_arn],
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)

# Composite alarm: ANY bias metric breached
cloudwatch.put_composite_alarm(
    AlarmName=f"{PROJECT_NAME}-bias-any-violation-{ENV}",
    AlarmDescription="One or more bias metrics have breached their thresholds",
    AlarmRule=(
        f'ALARM("{PROJECT_NAME}-bias-DPL-{ENV}") OR '
        f'ALARM("{PROJECT_NAME}-bias-DI-{ENV}") OR '
        f'ALARM("{PROJECT_NAME}-bias-KL-{ENV}")'
    ),
    AlarmActions=[sns_topic_arn],
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)
```

---

## Integration Points

- **Upstream**: `devops/04` (IAM Roles & Policies) → Provides the base IAM role patterns and permission boundaries for bias monitoring job roles, remediation Lambda roles, and Model Cards access roles
- **Upstream**: `mlops/03` (LLM Inference Deployment) → Provides the SageMaker endpoint (ENDPOINT_NAME) that this template monitors for bias; endpoint must have data capture enabled
- **Upstream**: `mlops/05` (Model Monitoring & Drift Detection) → Provides the data capture configuration on the endpoint and the base monitoring infrastructure; this template extends monitoring with bias-specific capabilities
- **Downstream**: `mlops/11` (ML Governance & Responsible AI) → Consumes bias evaluation results for Model Cards audit trail; bias metrics and violation history are logged to Model Cards for compliance documentation
- **Downstream**: `devops/03` (CloudWatch Monitoring & Alerting) → Publishes bias metrics to CloudWatch custom namespace for inclusion in operational dashboards; bias alarms integrate with existing alerting infrastructure
- **Downstream**: `devops/11` (Custom CloudWatch Model Quality) → Bias metrics published to CloudWatch can be consumed by custom model quality dashboards for unified model health monitoring
- **Downstream**: `data/05` (EventBridge ML Orchestration) → Bias breach events published via EventBridge can trigger downstream ML workflows such as model retraining or traffic shifting
