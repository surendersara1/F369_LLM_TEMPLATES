<!-- Template Version: 1.1 | boto3: 1.35+ | numpy: 1.26+ | Model IDs: 2026-04-22 refresh -->

# Template DevOps 11 — Custom CloudWatch Metrics for ML Model Quality Monitoring

## Purpose
Generate production-ready custom CloudWatch metrics infrastructure for ML model quality monitoring: custom metric publishing code for tokens per second, latency percentiles (p50/p95/p99), error rates, and embedding cosine similarity, metric math expressions for cost per 1K tokens, error rate percentage, and throughput efficiency, CloudWatch anomaly detection models on key inference metrics, a Lambda function that parses CloudWatch Logs from inference endpoints and publishes derived metrics, anomaly breach alarms with SNS notifications, and a CloudWatch dashboard displaying all custom metrics by endpoint name and model version.

---

## Role Definition

You are an expert AWS observability engineer and ML model quality monitoring specialist with expertise in:
- CloudWatch Custom Metrics: `put_metric_data()` with statistic sets, high-resolution metrics, dimensions, units, and metric math expressions
- CloudWatch Anomaly Detection: `put_anomaly_detector()` with configurable bands, metric math anomaly detectors, and `ANOMALY_DETECTION_BAND()` alarm functions
- CloudWatch Metric Math: arithmetic expressions across metrics (cost per 1K tokens, error rate %, throughput efficiency), `SEARCH` expressions, cross-account/cross-region aggregation
- CloudWatch Dashboards: `put_dashboard()` with JSON widget definitions, metric widgets, number widgets, text widgets, alarm status widgets, and automatic layout
- CloudWatch Logs Insights: Lambda subscription filters that parse structured inference logs and publish derived metrics
- ML inference metrics: tokens per second throughput, latency percentile distributions (p50/p95/p99), invocation error rates, embedding cosine similarity for drift detection, cost-per-inference calculations
- SageMaker endpoint metrics: extending built-in `Invocations`, `ModelLatency`, `OverheadLatency` with custom business metrics
- Bedrock invocation metrics: parsing Bedrock CloudWatch metrics and invocation logs for token usage, latency, and throttle tracking
- Alarm design: composite alarms combining anomaly detection with static thresholds, alarm actions for SNS and Lambda remediation

Generate complete, production-deployable custom CloudWatch metrics infrastructure code.

---

## Context & Inputs

```
PROJECT_NAME:               [REQUIRED]
AWS_REGION:                 [REQUIRED]
AWS_ACCOUNT_ID:             [REQUIRED]
ENV:                        [REQUIRED - dev | stage | prod]

ENDPOINT_NAMES:             [REQUIRED - JSON list of endpoint/model identifiers]
                            List of SageMaker endpoint names or Bedrock model IDs to monitor.
                            Each endpoint gets its own dimension in custom metrics.
                            Example: ["my-llm-endpoint-prod", "us.anthropic.claude-sonnet-4-7-20260109-v1:0"]

METRIC_NAMESPACE:           [OPTIONAL: {PROJECT_NAME}/MLModelQuality]
                            CloudWatch custom metric namespace for all published metrics.
                            All custom metrics are published under this namespace.

METRICS_TO_PUBLISH:         [OPTIONAL: all]
                            Comma-separated list of metric groups to publish:
                            - tokens_per_second: throughput metric (total tokens / latency)
                            - latency_percentiles: p50, p95, p99 invocation latency
                            - error_rate: invocation error rate as percentage
                            - embedding_similarity: cosine similarity between request
                              embeddings and a baseline for drift detection
                            - all: publish all metric groups (default)

ANOMALY_DETECTION_ENABLED:  [OPTIONAL: true]
                            Enable CloudWatch anomaly detection models on key metrics
                            (tokens_per_second, p95 latency, error_rate). Anomaly detectors
                            learn normal patterns and trigger alarms on deviations.

DASHBOARD_NAME:             [OPTIONAL: {PROJECT_NAME}-model-quality-{ENV}]
                            CloudWatch dashboard name. The dashboard displays all custom
                            metrics organized by endpoint and model version.

LOG_GROUP_NAME:             [OPTIONAL: /aws/sagemaker/Endpoints/{ENDPOINT_NAMES[0]}]
                            CloudWatch Logs log group containing inference logs to parse.
                            The metric publishing Lambda subscribes to this log group.

BASELINE_EMBEDDINGS_S3:     [OPTIONAL: none]
                            S3 path to baseline embedding vectors (NumPy .npy file) for
                            cosine similarity drift detection. If not provided, the first
                            batch of embeddings becomes the baseline.
                            Example: s3://{PROJECT_NAME}-ml-artifacts-{ENV}/baselines/embeddings.npy

COST_PER_1K_INPUT_TOKENS:   [OPTIONAL: 0.003]
                            Cost in USD per 1,000 input tokens for the monitored model.
                            Used in the cost-per-1K-tokens metric math expression.

COST_PER_1K_OUTPUT_TOKENS:  [OPTIONAL: 0.015]
                            Cost in USD per 1,000 output tokens for the monitored model.
                            Used in the cost-per-1K-tokens metric math expression.

SNS_TOPIC_ARN:              [OPTIONAL: none]
                            SNS topic ARN for anomaly breach and threshold alarm notifications.
                            If not provided, creates a new SNS topic.
                            Typically read from SSM: /mlops/{PROJECT_NAME}/{ENV}/sns-alerts-arn
```

---

## Task

Generate complete custom CloudWatch metrics infrastructure for ML model quality monitoring:

```
{PROJECT_NAME}-cw-model-quality/
├── metrics/
│   ├── metric_publisher.py               # Core metric publishing with put_metric_data()
│   ├── tokens_per_second.py              # Tokens/sec throughput metric computation
│   ├── latency_percentiles.py            # p50/p95/p99 latency from statistic sets
│   ├── error_rate.py                     # Error rate percentage metric
│   └── embedding_similarity.py           # Cosine similarity drift metric
├── metric_math/
│   ├── cost_per_1k_tokens.py             # Metric math: cost per 1K tokens
│   ├── error_rate_pct.py                 # Metric math: error rate as percentage
│   └── throughput_efficiency.py          # Metric math: throughput efficiency ratio
├── anomaly/
│   ├── anomaly_detectors.py              # Create anomaly detection models
│   └── anomaly_alarms.py                 # Anomaly breach alarms + SNS
├── log_parser/
│   ├── log_metric_lambda.py              # Lambda: parse logs → publish metrics
│   └── subscription_filter.py            # CloudWatch Logs subscription filter setup
├── dashboard/
│   └── dashboard_builder.py              # CloudWatch dashboard with all metrics
├── infrastructure/
│   ├── ssm_outputs.py                    # Write metric namespace and dashboard to SSM
│   └── config.py                         # Central configuration
├── run_setup.py                          # CLI orchestrator
└── requirements.txt
```

**config.py**: Central configuration dataclass with all parameters. Load from environment variables or CLI args. Validate required fields. Construct resource names using `{PROJECT_NAME}-{component}-{ENV}` convention. Parse ENDPOINT_NAMES from JSON string. Parse METRICS_TO_PUBLISH into a set of metric group names. Set defaults for METRIC_NAMESPACE, DASHBOARD_NAME, ANOMALY_DETECTION_ENABLED, COST_PER_1K_INPUT_TOKENS, COST_PER_1K_OUTPUT_TOKENS. Validate that ENDPOINT_NAMES is a non-empty list.

**metric_publisher.py**: Core metric publishing module:
- `publish_metric(metric_name, value, unit, dimensions, timestamp=None)`: wrapper around `cloudwatch.put_metric_data()` that publishes a single metric value to the configured METRIC_NAMESPACE. Accepts dimensions as a list of `{"Name": ..., "Value": ...}` dicts. Handles batching (max 1000 metric data points per `put_metric_data()` call).
- `publish_statistic_set(metric_name, stats, unit, dimensions)`: publish a `StatisticSet` with `SampleCount`, `Sum`, `Minimum`, `Maximum` for pre-aggregated data. Used for latency percentile computation from batched inference results.
- `publish_batch(metric_data_list)`: batch-publish multiple metrics in a single `put_metric_data()` call, splitting into chunks of 1000 if needed.
- All metrics include dimensions: `EndpointName` (from ENDPOINT_NAMES), `Environment` (ENV), `ModelVersion` (extracted from endpoint config or passed as parameter).

**tokens_per_second.py**: Tokens per second throughput metric:
- `compute_tokens_per_second(input_tokens, output_tokens, latency_ms)`: calculate `(input_tokens + output_tokens) / (latency_ms / 1000)` to get tokens per second throughput.
- `publish_tokens_per_second(endpoint_name, input_tokens, output_tokens, latency_ms, model_version="latest")`: compute and publish `TokensPerSecond` metric with dimensions `EndpointName`, `Environment`, `ModelVersion`. Also publish `InputTokens` and `OutputTokens` as separate count metrics for downstream metric math.
- Include a batch variant `publish_tokens_per_second_batch(records)` that accepts a list of `(endpoint_name, input_tokens, output_tokens, latency_ms)` tuples and publishes as a statistic set.

**latency_percentiles.py**: Latency percentile metrics:
- `publish_latency_sample(endpoint_name, latency_ms, model_version="latest")`: publish a single `InvocationLatency` metric value in milliseconds. CloudWatch computes percentiles from individual data points when using `ExtendedStatistics` in `get_metric_statistics()`.
- `publish_latency_statistic_set(endpoint_name, latencies_ms, model_version="latest")`: given a list of latency values, compute `SampleCount`, `Sum`, `Minimum`, `Maximum` and publish as a `StatisticSet`. This is more efficient for batch publishing.
- `compute_and_publish_percentiles(endpoint_name, latencies_ms, model_version="latest")`: compute p50, p95, p99 locally from the latency list using `numpy.percentile()` and publish as separate metrics `LatencyP50`, `LatencyP95`, `LatencyP99`. This provides exact percentiles (CloudWatch statistic sets only support approximate percentiles).
- All latency metrics use `Unit=Milliseconds` and dimensions `EndpointName`, `Environment`, `ModelVersion`.

**error_rate.py**: Error rate metric:
- `publish_invocation_result(endpoint_name, is_error, error_type=None, model_version="latest")`: publish `InvocationCount` (value=1) and `ErrorCount` (value=1 if is_error, 0 otherwise) as separate metrics. If is_error, also publish `ErrorByType` with an additional `ErrorType` dimension (e.g., `ThrottlingException`, `ModelError`, `ValidationError`).
- `publish_error_rate_batch(endpoint_name, total_count, error_count, model_version="latest")`: publish pre-aggregated counts for a batch period. Downstream metric math computes the percentage.
- Error rate percentage is computed via metric math (see `error_rate_pct.py`) rather than published directly, to allow CloudWatch to compute it over any time period.

**embedding_similarity.py**: Cosine similarity drift metric:
- `load_baseline_embeddings(s3_path)`: load baseline embedding vectors from S3 (NumPy .npy format). Cache locally after first load.
- `compute_cosine_similarity(embedding_a, embedding_b)`: compute cosine similarity between two embedding vectors using `numpy.dot(a, b) / (numpy.linalg.norm(a) * numpy.linalg.norm(b))`.
- `publish_embedding_similarity(endpoint_name, current_embedding, model_version="latest")`: compute cosine similarity between the current embedding and the baseline mean embedding. Publish `EmbeddingSimilarity` metric (value 0.0–1.0, Unit=None). Values below a threshold (e.g., 0.85) indicate potential drift.
- `update_baseline(embeddings_list, s3_path)`: compute mean embedding from a list of embeddings and save to S3 as the new baseline. Called periodically or when a new model version is deployed.
- If BASELINE_EMBEDDINGS_S3 is not provided, the first batch of 100 embeddings is used to create the initial baseline.

**cost_per_1k_tokens.py**: Metric math expression for cost per 1K tokens:
- `get_cost_metric_math_expression()`: return the CloudWatch metric math expression that computes cost per 1K tokens: `(input_tokens * COST_PER_1K_INPUT_TOKENS / 1000) + (output_tokens * COST_PER_1K_OUTPUT_TOKENS / 1000)`. Uses metric IDs `m1` (InputTokens) and `m2` (OutputTokens) from the custom namespace.
- `create_cost_metric_widget(endpoint_name)`: return a CloudWatch dashboard widget JSON definition that displays the cost-per-1K-tokens metric math expression as a line graph over time, with dimensions filtered to the specified endpoint.
- Expression: `e1 = (m1 * {COST_PER_1K_INPUT_TOKENS} / 1000) + (m2 * {COST_PER_1K_OUTPUT_TOKENS} / 1000)` where `m1 = InputTokens SUM`, `m2 = OutputTokens SUM`.

**error_rate_pct.py**: Metric math expression for error rate percentage:
- `get_error_rate_math_expression()`: return the metric math expression `(error_count / invocation_count) * 100`. Uses metric IDs `m1` (ErrorCount SUM) and `m2` (InvocationCount SUM).
- `create_error_rate_widget(endpoint_name)`: return a dashboard widget JSON definition displaying error rate percentage as a line graph with a horizontal annotation at 5% (warning) and 10% (critical).
- Expression: `e1 = (m1 / m2) * 100` where `m1 = ErrorCount SUM`, `m2 = InvocationCount SUM`.

**throughput_efficiency.py**: Metric math expression for throughput efficiency:
- `get_throughput_efficiency_expression()`: return the metric math expression that computes throughput efficiency as `tokens_per_second / max_tokens_per_second_baseline`. This ratio (0.0–1.0+) indicates how close the endpoint is to its theoretical maximum throughput.
- `create_throughput_widget(endpoint_name)`: return a dashboard widget JSON definition displaying throughput efficiency as a gauge or number widget.
- Expression: `e1 = m1 / {MAX_TOKENS_PER_SECOND}` where `m1 = TokensPerSecond AVG`.

**anomaly_detectors.py**: CloudWatch anomaly detection models:
- `create_anomaly_detector(metric_name, dimensions, stat="Average")`: call `cloudwatch.put_anomaly_detector()` to create an anomaly detection model for the specified metric. The detector learns normal patterns over 2 weeks and creates an anomaly detection band.
- `create_all_anomaly_detectors(endpoint_name)`: create anomaly detectors for: `TokensPerSecond` (Average), `LatencyP95` (Maximum), `ErrorCount` (Sum), and `EmbeddingSimilarity` (Average) for the given endpoint. Each detector uses dimensions `EndpointName` and `Environment`.
- `delete_anomaly_detector(metric_name, dimensions, stat)`: remove an anomaly detector using `cloudwatch.delete_anomaly_detector()`.
- Only create detectors if ANOMALY_DETECTION_ENABLED is true.

**anomaly_alarms.py**: Anomaly breach alarms with SNS:
- `create_anomaly_alarm(metric_name, dimensions, stat, threshold_band=2, alarm_suffix="")`: create a CloudWatch alarm using `ANOMALY_DETECTION_BAND(m1, {threshold_band})` as the threshold metric. The alarm fires when the metric value falls outside the anomaly detection band. Action: send to SNS_TOPIC_ARN.
- `create_static_threshold_alarm(metric_name, dimensions, stat, threshold, comparison, alarm_suffix="")`: create a standard threshold alarm for metrics where static thresholds are appropriate (e.g., error rate > 10%).
- `create_all_alarms(endpoint_name)`: create alarms for:
  - `TokensPerSecond` anomaly alarm (fires on throughput drop)
  - `LatencyP95` anomaly alarm (fires on latency spike)
  - `ErrorCount` static alarm (fires when error count > 0 for 3 consecutive periods in prod)
  - `EmbeddingSimilarity` static alarm (fires when similarity < 0.85, indicating drift)
- `create_composite_alarm(endpoint_name)`: create a composite alarm that fires when ANY of the individual alarms are in ALARM state, providing a single "model quality degraded" signal.
- All alarms follow naming: `{PROJECT_NAME}-{metric}-{endpoint}-{ENV}`.

**log_metric_lambda.py**: Lambda function that parses CloudWatch Logs and publishes metrics:
- Triggered by CloudWatch Logs subscription filter on the inference log group.
- Parse log events for structured JSON log lines containing: `input_tokens`, `output_tokens`, `latency_ms`, `is_error`, `error_type`, `model_id`, `endpoint_name`.
- For each parsed log event, call the appropriate metric publishing functions: `publish_tokens_per_second()`, `publish_latency_sample()`, `publish_invocation_result()`.
- Batch metric data points and publish in a single `put_metric_data()` call per invocation for efficiency.
- Handle malformed log lines gracefully (log warning, skip, continue).
- Include CloudWatch Embedded Metric Format (EMF) as an alternative: emit metrics directly from the inference function using `aws_embedded_metrics` library, avoiding the need for log parsing.

**subscription_filter.py**: CloudWatch Logs subscription filter setup:
- `create_subscription_filter(log_group_name, lambda_arn)`: call `logs.put_subscription_filter()` to subscribe the metric publishing Lambda to the inference log group.
- Filter pattern: `{ $.input_tokens = * }` to match structured JSON log lines containing token metrics.
- Grant the log group permission to invoke the Lambda using `lambda_client.add_permission()`.
- Include a `delete_subscription_filter()` for cleanup.

**dashboard_builder.py**: CloudWatch dashboard with all custom metrics:
- `build_dashboard(endpoint_names)`: construct a CloudWatch dashboard JSON body with widgets organized in rows:
  - Row 1: Title text widget + alarm status widget showing all model quality alarms
  - Row 2: Tokens per second line graph (one line per endpoint), cost per 1K tokens metric math graph
  - Row 3: Latency percentiles (p50/p95/p99) line graph per endpoint, latency anomaly detection band overlay
  - Row 4: Error rate percentage metric math graph, error count by type stacked area chart
  - Row 5: Embedding similarity line graph with threshold annotation at 0.85, throughput efficiency gauge
  - Row 6: Number widgets showing current values: avg tokens/sec, p95 latency, error rate %, similarity score
- Each metric widget includes dimensions for `EndpointName` and `Environment`, with one line per endpoint from ENDPOINT_NAMES.
- `create_dashboard()`: call `cloudwatch.put_dashboard()` with the built JSON body.
- `update_dashboard()`: re-create the dashboard (put_dashboard is idempotent) when endpoints are added or removed.
- Dashboard name follows: `{PROJECT_NAME}-model-quality-{ENV}` (or DASHBOARD_NAME if provided).

**ssm_outputs.py**: Write metric configuration to SSM Parameter Store:
- Path: `/mlops/{PROJECT_NAME}/{ENV}/cw-metric-namespace` — custom metric namespace
- Path: `/mlops/{PROJECT_NAME}/{ENV}/cw-model-quality-dashboard` — dashboard name
- Path: `/mlops/{PROJECT_NAME}/{ENV}/cw-anomaly-alarms` — JSON list of alarm ARNs
- Other templates read from these SSM paths for dashboard embedding and alarm aggregation.

**run_setup.py**: CLI orchestrator that runs setup steps in order:
1. Load and validate configuration
2. Create anomaly detection models for each endpoint (if ANOMALY_DETECTION_ENABLED)
3. Create anomaly breach alarms and static threshold alarms for each endpoint
4. Create composite alarm per endpoint
5. Deploy metric publishing Lambda and subscription filter
6. Create CloudWatch dashboard
7. Write outputs to SSM
8. Print summary with metric namespace, dashboard URL, alarm names, and Lambda ARN

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**CloudWatch Custom Metrics:** Use `cloudwatch.put_metric_data()` for all custom metric publishing. Each `put_metric_data()` call supports up to 1000 metric data points — batch metrics to minimize API calls. Use `StatisticSets` (`SampleCount`, `Sum`, `Minimum`, `Maximum`) for pre-aggregated data to reduce costs (one API call per batch instead of one per data point). All metrics must include `Unit` (Milliseconds, Count, None, etc.) for correct CloudWatch aggregation. Use high-resolution metrics (StorageResolution=1) only for latency metrics in prod where sub-minute granularity is needed; use standard resolution (60s) for all other metrics to control costs.

**Metric Dimensions:** Every custom metric must include at minimum two dimensions: `EndpointName` (SageMaker endpoint name or Bedrock model ID) and `Environment` (dev/stage/prod). Add `ModelVersion` as a third dimension when available to enable per-version comparison. CloudWatch charges per unique dimension combination — limit to 3 dimensions per metric to control costs. Do not use high-cardinality dimensions (e.g., request ID, customer ID) as metric dimensions; use CloudWatch Logs Insights for high-cardinality analysis instead.

**Metric Math Expressions:** Use CloudWatch metric math for derived metrics (cost per 1K tokens, error rate %, throughput efficiency) rather than publishing computed values. This allows CloudWatch to compute the expression over any time period and avoids double-counting. Metric math expressions use metric IDs (`m1`, `m2`, etc.) and arithmetic operators (`+`, `-`, `*`, `/`). Use `METRICS()` function in dashboard widgets to auto-discover metrics matching a search pattern. Use `FILL(m1, 0)` to handle missing data points in ratio calculations.

**Anomaly Detection:** CloudWatch anomaly detection uses machine learning to model expected metric behavior and creates a band of expected values. Use `put_anomaly_detector()` to create detectors — they require 2 weeks of data to train. Use `ANOMALY_DETECTION_BAND(m1, 2)` in alarm threshold expressions where `2` is the number of standard deviations. Anomaly detection is most effective for metrics with regular patterns (e.g., daily traffic cycles). For metrics without patterns (e.g., error rate that should always be near zero), use static threshold alarms instead.

**Lambda Log Parser:** The metric publishing Lambda is triggered by CloudWatch Logs subscription filters. Each invocation receives a batch of log events (up to 10,000 events or 1MB). Parse structured JSON log lines and batch-publish metrics. The Lambda must complete within 5 minutes (default timeout). Use `aws_embedded_metrics` as an alternative to log parsing — it publishes metrics directly from the inference function without a separate Lambda. EMF is simpler but requires modifying the inference function code.

**Statistic Sets:** When publishing pre-aggregated metrics, use `StatisticSet` instead of individual `Value` entries. A `StatisticSet` contains `SampleCount`, `Sum`, `Minimum`, `Maximum` and allows CloudWatch to compute `Average` (Sum/SampleCount). This is more cost-effective for high-throughput endpoints: instead of publishing 1000 individual latency values, aggregate into one `StatisticSet` per minute. Note: `StatisticSet` does not support percentile computation — publish individual values or use `ExtendedStatistics` for p50/p95/p99.

**Embedding Similarity:** Cosine similarity is computed between current inference embeddings and a stored baseline. Values range from -1.0 to 1.0 (1.0 = identical, 0.0 = orthogonal). For drift detection, a similarity below 0.85 typically indicates meaningful distribution shift. Store baseline embeddings in S3 as NumPy .npy files. Update baselines when deploying new model versions. This metric is optional and only applicable when the inference pipeline produces embeddings (e.g., RAG retrieval, semantic search).

**Cost:** CloudWatch custom metrics cost $0.30 per metric per month (first 10,000 metrics). Each unique combination of metric name + dimensions = one metric. With 5 endpoints × 10 metric names × 3 dimensions = 150 metrics ≈ $45/month. `put_metric_data()` API calls cost $0.01 per 1,000 requests. Anomaly detection adds $3.00 per detector per month. Dashboard is free (up to 3 dashboards, then $3.00/month each). Budget for CloudWatch costs in high-throughput scenarios.

**Naming:** All resources follow `{PROJECT_NAME}-{component}-{ENV}` convention:
- Metric namespace: `{PROJECT_NAME}/MLModelQuality`
- Dashboard: `{PROJECT_NAME}-model-quality-{ENV}`
- Anomaly alarm: `{PROJECT_NAME}-anomaly-{metric}-{endpoint}-{ENV}`
- Static alarm: `{PROJECT_NAME}-{metric}-threshold-{endpoint}-{ENV}`
- Composite alarm: `{PROJECT_NAME}-model-quality-{endpoint}-{ENV}`
- Lambda function: `{PROJECT_NAME}-metric-publisher-{ENV}`
- SSM parameter: `/mlops/{PROJECT_NAME}/{ENV}/cw-metric-namespace`
- SNS topic: `{PROJECT_NAME}-model-quality-alerts-{ENV}`

---

## Code Scaffolding Hints

**Publishing custom metrics with put_metric_data() and statistic sets:**
```python
import boto3
import time
from datetime import datetime, timezone

cloudwatch = boto3.client("cloudwatch", region_name=AWS_REGION)

METRIC_NAMESPACE = f"{PROJECT_NAME}/MLModelQuality"


def publish_metric(metric_name: str, value: float, unit: str, dimensions: list, timestamp: datetime = None):
    """Publish a single custom metric value to CloudWatch."""
    metric_data = {
        "MetricName": metric_name,
        "Value": value,
        "Unit": unit,
        "Dimensions": dimensions,
        "Timestamp": timestamp or datetime.now(timezone.utc),
    }
    cloudwatch.put_metric_data(Namespace=METRIC_NAMESPACE, MetricData=[metric_data])


def publish_statistic_set(metric_name: str, values: list, unit: str, dimensions: list):
    """Publish a pre-aggregated statistic set to CloudWatch.

    More cost-effective than publishing individual values for high-throughput endpoints.
    CloudWatch computes Average = Sum / SampleCount from the statistic set.
    """
    if not values:
        return
    metric_data = {
        "MetricName": metric_name,
        "StatisticValues": {
            "SampleCount": len(values),
            "Sum": sum(values),
            "Minimum": min(values),
            "Maximum": max(values),
        },
        "Unit": unit,
        "Dimensions": dimensions,
        "Timestamp": datetime.now(timezone.utc),
    }
    cloudwatch.put_metric_data(Namespace=METRIC_NAMESPACE, MetricData=[metric_data])


def publish_batch(metric_data_list: list):
    """Batch-publish multiple metrics, splitting into chunks of 1000 (API limit)."""
    for i in range(0, len(metric_data_list), 1000):
        chunk = metric_data_list[i : i + 1000]
        cloudwatch.put_metric_data(Namespace=METRIC_NAMESPACE, MetricData=chunk)


def build_dimensions(endpoint_name: str, model_version: str = "latest") -> list:
    """Build standard dimensions for ML model quality metrics."""
    return [
        {"Name": "EndpointName", "Value": endpoint_name},
        {"Name": "Environment", "Value": ENV},
        {"Name": "ModelVersion", "Value": model_version},
    ]
```

**Tokens per second throughput metric:**
```python
def compute_tokens_per_second(input_tokens: int, output_tokens: int, latency_ms: float) -> float:
    """Compute tokens per second throughput."""
    if latency_ms <= 0:
        return 0.0
    total_tokens = input_tokens + output_tokens
    return total_tokens / (latency_ms / 1000.0)


def publish_tokens_per_second(endpoint_name: str, input_tokens: int, output_tokens: int,
                               latency_ms: float, model_version: str = "latest"):
    """Publish tokens/sec throughput and individual token count metrics."""
    dims = build_dimensions(endpoint_name, model_version)
    tps = compute_tokens_per_second(input_tokens, output_tokens, latency_ms)

    metric_data = [
        {
            "MetricName": "TokensPerSecond",
            "Value": tps,
            "Unit": "Count/Second",
            "Dimensions": dims,
            "Timestamp": datetime.now(timezone.utc),
        },
        {
            "MetricName": "InputTokens",
            "Value": input_tokens,
            "Unit": "Count",
            "Dimensions": dims,
            "Timestamp": datetime.now(timezone.utc),
        },
        {
            "MetricName": "OutputTokens",
            "Value": output_tokens,
            "Unit": "Count",
            "Dimensions": dims,
            "Timestamp": datetime.now(timezone.utc),
        },
    ]
    publish_batch(metric_data)
```

**Latency percentile metrics with numpy:**
```python
import numpy as np


def publish_latency_statistic_set(endpoint_name: str, latencies_ms: list, model_version: str = "latest"):
    """Publish latency as a statistic set for efficient aggregation."""
    dims = build_dimensions(endpoint_name, model_version)
    publish_statistic_set("InvocationLatency", latencies_ms, "Milliseconds", dims)


def compute_and_publish_percentiles(endpoint_name: str, latencies_ms: list, model_version: str = "latest"):
    """Compute exact p50/p95/p99 percentiles and publish as separate metrics."""
    if not latencies_ms:
        return
    dims = build_dimensions(endpoint_name, model_version)
    arr = np.array(latencies_ms)
    p50 = float(np.percentile(arr, 50))
    p95 = float(np.percentile(arr, 95))
    p99 = float(np.percentile(arr, 99))

    metric_data = [
        {"MetricName": "LatencyP50", "Value": p50, "Unit": "Milliseconds",
         "Dimensions": dims, "Timestamp": datetime.now(timezone.utc)},
        {"MetricName": "LatencyP95", "Value": p95, "Unit": "Milliseconds",
         "Dimensions": dims, "Timestamp": datetime.now(timezone.utc)},
        {"MetricName": "LatencyP99", "Value": p99, "Unit": "Milliseconds",
         "Dimensions": dims, "Timestamp": datetime.now(timezone.utc)},
    ]
    publish_batch(metric_data)
    return {"p50": p50, "p95": p95, "p99": p99}
```

**Error rate metric publishing:**
```python
def publish_invocation_result(endpoint_name: str, is_error: bool, error_type: str = None,
                               model_version: str = "latest"):
    """Publish invocation count and error count metrics."""
    dims = build_dimensions(endpoint_name, model_version)
    now = datetime.now(timezone.utc)

    metric_data = [
        {"MetricName": "InvocationCount", "Value": 1, "Unit": "Count",
         "Dimensions": dims, "Timestamp": now},
        {"MetricName": "ErrorCount", "Value": 1 if is_error else 0, "Unit": "Count",
         "Dimensions": dims, "Timestamp": now},
    ]

    if is_error and error_type:
        error_dims = dims + [{"Name": "ErrorType", "Value": error_type}]
        metric_data.append(
            {"MetricName": "ErrorByType", "Value": 1, "Unit": "Count",
             "Dimensions": error_dims, "Timestamp": now}
        )

    publish_batch(metric_data)
```

**Embedding cosine similarity for drift detection:**
```python
import numpy as np
import boto3
import io

s3 = boto3.client("s3", region_name=AWS_REGION)

_baseline_cache = {}


def load_baseline_embeddings(s3_path: str) -> np.ndarray:
    """Load baseline embedding vectors from S3 (NumPy .npy format)."""
    if s3_path in _baseline_cache:
        return _baseline_cache[s3_path]

    # Parse s3://bucket/key
    parts = s3_path.replace("s3://", "").split("/", 1)
    bucket, key = parts[0], parts[1]

    response = s3.get_object(Bucket=bucket, Key=key)
    data = np.load(io.BytesIO(response["Body"].read()))
    _baseline_cache[s3_path] = data
    return data


def compute_cosine_similarity(embedding_a: np.ndarray, embedding_b: np.ndarray) -> float:
    """Compute cosine similarity between two embedding vectors."""
    norm_a = np.linalg.norm(embedding_a)
    norm_b = np.linalg.norm(embedding_b)
    if norm_a == 0 or norm_b == 0:
        return 0.0
    return float(np.dot(embedding_a, embedding_b) / (norm_a * norm_b))


def publish_embedding_similarity(endpoint_name: str, current_embedding: np.ndarray,
                                  baseline_s3_path: str, model_version: str = "latest"):
    """Compute and publish cosine similarity between current and baseline embeddings."""
    baseline = load_baseline_embeddings(baseline_s3_path)
    # Use mean of baseline embeddings as reference
    baseline_mean = np.mean(baseline, axis=0) if baseline.ndim > 1 else baseline
    similarity = compute_cosine_similarity(current_embedding, baseline_mean)

    dims = build_dimensions(endpoint_name, model_version)
    publish_metric("EmbeddingSimilarity", similarity, "None", dims)
    return similarity
```

**CloudWatch anomaly detection models:**
```python
import boto3

cloudwatch = boto3.client("cloudwatch", region_name=AWS_REGION)


def create_anomaly_detector(metric_name: str, dimensions: list, stat: str = "Average"):
    """Create a CloudWatch anomaly detection model for the specified metric."""
    cloudwatch.put_anomaly_detector(
        Namespace=METRIC_NAMESPACE,
        MetricName=metric_name,
        Dimensions=dimensions,
        Stat=stat,
        Configuration={
            "ExcludedTimeRanges": [],  # No exclusions by default
            "MetricTimezone": "UTC",
        },
    )
    print(f"Anomaly detector created: {metric_name} ({stat}) for {dimensions}")


def create_all_anomaly_detectors(endpoint_name: str):
    """Create anomaly detectors for key model quality metrics."""
    dims = [
        {"Name": "EndpointName", "Value": endpoint_name},
        {"Name": "Environment", "Value": ENV},
    ]
    detectors = [
        ("TokensPerSecond", "Average"),
        ("LatencyP95", "Maximum"),
        ("ErrorCount", "Sum"),
        ("EmbeddingSimilarity", "Average"),
    ]
    for metric_name, stat in detectors:
        create_anomaly_detector(metric_name, dims, stat)
    print(f"All anomaly detectors created for endpoint: {endpoint_name}")


def delete_anomaly_detector(metric_name: str, dimensions: list, stat: str):
    """Remove an anomaly detector."""
    cloudwatch.delete_anomaly_detector(
        Namespace=METRIC_NAMESPACE,
        MetricName=metric_name,
        Dimensions=dimensions,
        Stat=stat,
    )
```

**Anomaly breach alarms with ANOMALY_DETECTION_BAND:**
```python
def create_anomaly_alarm(metric_name: str, dimensions: list, stat: str,
                          threshold_band: int = 2, endpoint_name: str = ""):
    """Create a CloudWatch alarm using anomaly detection band."""
    alarm_name = f"{PROJECT_NAME}-anomaly-{metric_name.lower()}-{endpoint_name}-{ENV}"

    cloudwatch.put_metric_alarm(
        AlarmName=alarm_name,
        AlarmDescription=f"Anomaly detected in {metric_name} for {endpoint_name}",
        Metrics=[
            {
                "Id": "m1",
                "MetricStat": {
                    "Metric": {
                        "Namespace": METRIC_NAMESPACE,
                        "MetricName": metric_name,
                        "Dimensions": dimensions,
                    },
                    "Period": 300,
                    "Stat": stat,
                },
                "ReturnData": True,
            },
            {
                "Id": "ad1",
                "Expression": f"ANOMALY_DETECTION_BAND(m1, {threshold_band})",
                "Label": f"{metric_name} anomaly band",
                "ReturnData": True,
            },
        ],
        ComparisonOperator="LessThanLowerOrGreaterThanUpperThreshold",
        ThresholdMetricId="ad1",
        EvaluationPeriods=3,
        DatapointsToAlarm=2,
        TreatMissingData="missing",
        AlarmActions=[SNS_TOPIC_ARN],
        OKActions=[SNS_TOPIC_ARN],
        Tags=[
            {"Key": "Project", "Value": PROJECT_NAME},
            {"Key": "Environment", "Value": ENV},
        ],
    )
    print(f"Anomaly alarm created: {alarm_name}")
    return alarm_name


def create_static_threshold_alarm(metric_name: str, dimensions: list, stat: str,
                                    threshold: float, comparison: str, endpoint_name: str = ""):
    """Create a standard threshold alarm."""
    alarm_name = f"{PROJECT_NAME}-{metric_name.lower()}-threshold-{endpoint_name}-{ENV}"

    cloudwatch.put_metric_alarm(
        AlarmName=alarm_name,
        AlarmDescription=f"{metric_name} threshold breach for {endpoint_name}",
        Namespace=METRIC_NAMESPACE,
        MetricName=metric_name,
        Dimensions=dimensions,
        Statistic=stat,
        Period=300,
        EvaluationPeriods=3,
        DatapointsToAlarm=2,
        Threshold=threshold,
        ComparisonOperator=comparison,
        TreatMissingData="missing",
        AlarmActions=[SNS_TOPIC_ARN],
        OKActions=[SNS_TOPIC_ARN],
        Tags=[
            {"Key": "Project", "Value": PROJECT_NAME},
            {"Key": "Environment", "Value": ENV},
        ],
    )
    print(f"Static alarm created: {alarm_name}")
    return alarm_name


def create_composite_alarm(endpoint_name: str, child_alarm_names: list):
    """Create a composite alarm that fires when ANY child alarm is in ALARM state."""
    alarm_name = f"{PROJECT_NAME}-model-quality-{endpoint_name}-{ENV}"
    alarm_rule = " OR ".join([f'ALARM("{name}")' for name in child_alarm_names])

    cloudwatch.put_composite_alarm(
        AlarmName=alarm_name,
        AlarmDescription=f"Model quality degraded for {endpoint_name}",
        AlarmRule=alarm_rule,
        AlarmActions=[SNS_TOPIC_ARN],
        OKActions=[SNS_TOPIC_ARN],
        Tags=[
            {"Key": "Project", "Value": PROJECT_NAME},
            {"Key": "Environment", "Value": ENV},
        ],
    )
    print(f"Composite alarm created: {alarm_name}")
    return alarm_name
```

**Metric math expressions for dashboard widgets:**
```python
# Cost per 1K tokens metric math expression
cost_per_1k_tokens_widget = {
    "type": "metric",
    "properties": {
        "title": "Cost per 1K Tokens",
        "metrics": [
            [METRIC_NAMESPACE, "InputTokens", "EndpointName", ENDPOINT_NAME,
             "Environment", ENV, {"id": "m1", "visible": False}],
            [METRIC_NAMESPACE, "OutputTokens", "EndpointName", ENDPOINT_NAME,
             "Environment", ENV, {"id": "m2", "visible": False}],
            [{"expression": f"(m1 * {COST_PER_1K_INPUT_TOKENS} / 1000) + (m2 * {COST_PER_1K_OUTPUT_TOKENS} / 1000)",
              "label": "Cost per 1K tokens (USD)", "id": "e1"}],
        ],
        "period": 300,
        "stat": "Sum",
        "region": AWS_REGION,
        "yAxis": {"left": {"label": "USD", "showUnits": False}},
    },
}

# Error rate percentage metric math expression
error_rate_pct_widget = {
    "type": "metric",
    "properties": {
        "title": "Error Rate %",
        "metrics": [
            [METRIC_NAMESPACE, "ErrorCount", "EndpointName", ENDPOINT_NAME,
             "Environment", ENV, {"id": "m1", "visible": False, "stat": "Sum"}],
            [METRIC_NAMESPACE, "InvocationCount", "EndpointName", ENDPOINT_NAME,
             "Environment", ENV, {"id": "m2", "visible": False, "stat": "Sum"}],
            [{"expression": "IF(m2 > 0, (m1 / m2) * 100, 0)",
              "label": "Error Rate %", "id": "e1"}],
        ],
        "period": 300,
        "region": AWS_REGION,
        "yAxis": {"left": {"label": "%", "showUnits": False}},
        "annotations": {
            "horizontal": [
                {"label": "Warning", "value": 5, "color": "#ff9900"},
                {"label": "Critical", "value": 10, "color": "#d13212"},
            ]
        },
    },
}

# Throughput efficiency metric math expression
throughput_efficiency_widget = {
    "type": "metric",
    "properties": {
        "title": "Throughput Efficiency",
        "metrics": [
            [METRIC_NAMESPACE, "TokensPerSecond", "EndpointName", ENDPOINT_NAME,
             "Environment", ENV, {"id": "m1", "visible": False, "stat": "Average"}],
            [{"expression": f"m1 / {MAX_TOKENS_PER_SECOND}",
              "label": "Efficiency Ratio", "id": "e1"}],
        ],
        "period": 300,
        "region": AWS_REGION,
        "yAxis": {"left": {"label": "Ratio", "min": 0, "max": 1.2}},
    },
}
```

**Lambda log parser for metric publishing:**
```python
import json
import base64
import gzip
from datetime import datetime, timezone

cloudwatch = boto3.client("cloudwatch", region_name=AWS_REGION)


def lambda_handler(event, context):
    """Parse CloudWatch Logs events and publish custom metrics.

    Triggered by CloudWatch Logs subscription filter on inference log group.
    """
    # Decode and decompress the CloudWatch Logs payload
    payload = base64.b64decode(event["awslogs"]["data"])
    log_data = json.loads(gzip.decompress(payload))

    metric_data = []
    for log_event in log_data.get("logEvents", []):
        try:
            record = json.loads(log_event["message"])
        except (json.JSONDecodeError, KeyError):
            continue  # Skip non-JSON log lines

        endpoint_name = record.get("endpoint_name", "unknown")
        model_version = record.get("model_version", "latest")
        dims = [
            {"Name": "EndpointName", "Value": endpoint_name},
            {"Name": "Environment", "Value": ENV},
            {"Name": "ModelVersion", "Value": model_version},
        ]
        now = datetime.now(timezone.utc)

        # Token metrics
        input_tokens = record.get("input_tokens", 0)
        output_tokens = record.get("output_tokens", 0)
        latency_ms = record.get("latency_ms", 0)

        if latency_ms > 0:
            tps = (input_tokens + output_tokens) / (latency_ms / 1000.0)
            metric_data.append(
                {"MetricName": "TokensPerSecond", "Value": tps,
                 "Unit": "Count/Second", "Dimensions": dims, "Timestamp": now}
            )

        metric_data.append(
            {"MetricName": "InputTokens", "Value": input_tokens,
             "Unit": "Count", "Dimensions": dims, "Timestamp": now}
        )
        metric_data.append(
            {"MetricName": "OutputTokens", "Value": output_tokens,
             "Unit": "Count", "Dimensions": dims, "Timestamp": now}
        )

        # Latency
        if latency_ms > 0:
            metric_data.append(
                {"MetricName": "InvocationLatency", "Value": latency_ms,
                 "Unit": "Milliseconds", "Dimensions": dims, "Timestamp": now}
            )

        # Error tracking
        is_error = record.get("is_error", False)
        metric_data.append(
            {"MetricName": "InvocationCount", "Value": 1,
             "Unit": "Count", "Dimensions": dims, "Timestamp": now}
        )
        metric_data.append(
            {"MetricName": "ErrorCount", "Value": 1 if is_error else 0,
             "Unit": "Count", "Dimensions": dims, "Timestamp": now}
        )

    # Batch publish (max 1000 per call)
    for i in range(0, len(metric_data), 1000):
        chunk = metric_data[i : i + 1000]
        cloudwatch.put_metric_data(Namespace=METRIC_NAMESPACE, MetricData=chunk)

    return {"statusCode": 200, "metricsPublished": len(metric_data)}
```

**CloudWatch Logs subscription filter setup:**
```python
import boto3

logs = boto3.client("logs", region_name=AWS_REGION)
lambda_client = boto3.client("lambda", region_name=AWS_REGION)


def create_subscription_filter(log_group_name: str, lambda_arn: str):
    """Subscribe the metric publishing Lambda to the inference log group."""
    filter_name = f"{PROJECT_NAME}-metric-publisher-{ENV}"

    # Grant CloudWatch Logs permission to invoke the Lambda
    try:
        lambda_client.add_permission(
            FunctionName=lambda_arn,
            StatementId=f"cloudwatch-logs-{filter_name}",
            Action="lambda:InvokeFunction",
            Principal="logs.amazonaws.com",
            SourceArn=f"arn:aws:logs:{AWS_REGION}:{AWS_ACCOUNT_ID}:log-group:{log_group_name}:*",
        )
    except lambda_client.exceptions.ResourceConflictException:
        pass  # Permission already exists

    # Create subscription filter matching structured JSON log lines with token data
    logs.put_subscription_filter(
        logGroupName=log_group_name,
        filterName=filter_name,
        filterPattern='{ $.input_tokens = * }',
        destinationArn=lambda_arn,
    )
    print(f"Subscription filter created: {filter_name} on {log_group_name}")
```

**CloudWatch dashboard builder:**
```python
import json

cloudwatch = boto3.client("cloudwatch", region_name=AWS_REGION)


def build_dashboard(endpoint_names: list, dashboard_name: str) -> str:
    """Build CloudWatch dashboard JSON with all model quality metrics."""
    widgets = []
    y = 0

    # Row 1: Title + alarm status
    widgets.append({
        "type": "text", "x": 0, "y": y, "width": 12, "height": 2,
        "properties": {"markdown": f"# ML Model Quality — {PROJECT_NAME} ({ENV})"}
    })
    widgets.append({
        "type": "alarm", "x": 12, "y": y, "width": 12, "height": 2,
        "properties": {
            "title": "Alarm Status",
            "alarms": [
                f"arn:aws:cloudwatch:{AWS_REGION}:{AWS_ACCOUNT_ID}:alarm:{PROJECT_NAME}-model-quality-{ep}-{ENV}"
                for ep in endpoint_names
            ],
        }
    })
    y += 2

    # Row 2: Tokens/sec + cost per 1K tokens
    tokens_metrics = []
    for ep in endpoint_names:
        tokens_metrics.append(
            [METRIC_NAMESPACE, "TokensPerSecond", "EndpointName", ep, "Environment", ENV]
        )
    widgets.append({
        "type": "metric", "x": 0, "y": y, "width": 12, "height": 6,
        "properties": {
            "title": "Tokens per Second by Endpoint",
            "metrics": tokens_metrics,
            "period": 300, "stat": "Average", "region": AWS_REGION,
            "yAxis": {"left": {"label": "Tokens/sec"}},
        }
    })
    # Cost per 1K tokens (first endpoint as example)
    widgets.append({
        "type": "metric", "x": 12, "y": y, "width": 12, "height": 6,
        "properties": {
            "title": "Cost per 1K Tokens (USD)",
            "metrics": [
                [METRIC_NAMESPACE, "InputTokens", "EndpointName", endpoint_names[0],
                 "Environment", ENV, {"id": "m1", "visible": False}],
                [METRIC_NAMESPACE, "OutputTokens", "EndpointName", endpoint_names[0],
                 "Environment", ENV, {"id": "m2", "visible": False}],
                [{"expression": f"(m1 * {COST_PER_1K_INPUT_TOKENS} / 1000) + (m2 * {COST_PER_1K_OUTPUT_TOKENS} / 1000)",
                  "label": "Cost/1K tokens", "id": "e1"}],
            ],
            "period": 300, "stat": "Sum", "region": AWS_REGION,
        }
    })
    y += 6

    # Row 3: Latency percentiles
    latency_metrics = []
    for ep in endpoint_names:
        latency_metrics.extend([
            [METRIC_NAMESPACE, "LatencyP50", "EndpointName", ep, "Environment", ENV, {"label": f"{ep} p50"}],
            [METRIC_NAMESPACE, "LatencyP95", "EndpointName", ep, "Environment", ENV, {"label": f"{ep} p95"}],
            [METRIC_NAMESPACE, "LatencyP99", "EndpointName", ep, "Environment", ENV, {"label": f"{ep} p99"}],
        ])
    widgets.append({
        "type": "metric", "x": 0, "y": y, "width": 24, "height": 6,
        "properties": {
            "title": "Latency Percentiles (ms)",
            "metrics": latency_metrics,
            "period": 300, "stat": "Maximum", "region": AWS_REGION,
            "yAxis": {"left": {"label": "ms"}},
        }
    })
    y += 6

    # Row 4: Error rate % + error by type
    widgets.append({
        "type": "metric", "x": 0, "y": y, "width": 12, "height": 6,
        "properties": {
            "title": "Error Rate %",
            "metrics": [
                [METRIC_NAMESPACE, "ErrorCount", "EndpointName", endpoint_names[0],
                 "Environment", ENV, {"id": "m1", "visible": False, "stat": "Sum"}],
                [METRIC_NAMESPACE, "InvocationCount", "EndpointName", endpoint_names[0],
                 "Environment", ENV, {"id": "m2", "visible": False, "stat": "Sum"}],
                [{"expression": "IF(m2 > 0, (m1 / m2) * 100, 0)",
                  "label": "Error Rate %", "id": "e1"}],
            ],
            "period": 300, "region": AWS_REGION,
            "annotations": {"horizontal": [
                {"label": "Warning", "value": 5, "color": "#ff9900"},
                {"label": "Critical", "value": 10, "color": "#d13212"},
            ]},
        }
    })
    widgets.append({
        "type": "metric", "x": 12, "y": y, "width": 12, "height": 6,
        "properties": {
            "title": "Embedding Similarity (Drift Detection)",
            "metrics": [
                [METRIC_NAMESPACE, "EmbeddingSimilarity", "EndpointName", ep, "Environment", ENV]
                for ep in endpoint_names
            ],
            "period": 300, "stat": "Average", "region": AWS_REGION,
            "yAxis": {"left": {"min": 0, "max": 1}},
            "annotations": {"horizontal": [
                {"label": "Drift Threshold", "value": 0.85, "color": "#d13212"},
            ]},
        }
    })

    dashboard_body = json.dumps({"widgets": widgets})
    return dashboard_body


def create_dashboard(endpoint_names: list, dashboard_name: str):
    """Create or update the CloudWatch dashboard."""
    body = build_dashboard(endpoint_names, dashboard_name)
    cloudwatch.put_dashboard(DashboardName=dashboard_name, DashboardBody=body)
    print(f"Dashboard created: {dashboard_name}")
    print(f"URL: https://{AWS_REGION}.console.aws.amazon.com/cloudwatch/home?region={AWS_REGION}#dashboards:name={dashboard_name}")
```

---

## Integration Points

- **Upstream**: `devops/04` → IAM roles for the metric publishing Lambda and any service that calls `cloudwatch:PutMetricData`; the Lambda execution role must include `cloudwatch:PutMetricData`, `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents`, and `s3:GetObject` (for baseline embeddings)
- **Upstream**: `mlops/03` → SageMaker inference endpoints whose invocation logs are parsed by the metric publishing Lambda; endpoint names from this template populate the ENDPOINT_NAMES parameter
- **Upstream**: `devops/03` → CloudWatch base monitoring configuration; this template extends the base CloudWatch setup with custom ML-specific metrics and dashboards
- **Upstream**: `devops/10` → OpenTelemetry tracing publishes `tokens.input`, `tokens.output`, and `latency_ms` span attributes to CloudWatch; this template consumes those metrics for percentile computation and anomaly detection
- **Downstream**: `devops/13` → Cost-per-inference dashboards consume the `InputTokens`, `OutputTokens`, and `InvocationCount` metrics from this template's namespace to compute per-invocation cost breakdowns
- **Downstream**: `finops/05` → FinOps dashboards embed widgets from this template's dashboard or reference the custom metric namespace for ML cost reporting
- **Downstream**: `devops/14` → Clarify real-time bias monitoring uses the `EmbeddingSimilarity` metric as a drift signal to trigger bias re-evaluation when similarity drops below threshold