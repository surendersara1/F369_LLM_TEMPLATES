<!-- Template Version: 1.1 | boto3: 1.35+ | QuickSight: Enterprise Edition | Model IDs: 2026-04-22 refresh -->

# Template DevOps 13 — Cost-per-Inference Dashboards and Per-Customer Cost Allocation

## Purpose
Generate production-ready cost-per-inference analytics infrastructure: CloudWatch dashboards with per-model cost-per-inference metric math expressions joining invocation counts with token-based pricing, a DynamoDB per-customer cost allocation table with a Lambda processor that attributes inference costs to individual customers or business units, QuickSight analysis for cost trend visualization (daily/weekly/monthly cost trends, model cost comparison, per-customer breakdown, margin analysis), cost alerting CloudWatch alarms that fire when per-inference cost exceeds configurable thresholds, automatic endpoint registration that discovers new SageMaker endpoints and Bedrock models for tracking, and a monthly CSV cost report generator that produces chargeback-ready reports by customer and model.

---

## Role Definition

You are an expert AWS FinOps engineer and ML inference cost analytics specialist with expertise in:
- CloudWatch Metric Math: arithmetic expressions for cost-per-inference computation (`cost / invocations`), `SEARCH` expressions for auto-discovering endpoint metrics, `FILL()` for handling missing data, cross-metric ratio calculations
- CloudWatch Dashboards: `put_dashboard()` with JSON widget definitions, metric math widgets, number widgets for KPI summaries, annotation lines for cost thresholds, automatic widget generation for dynamically discovered endpoints
- DynamoDB cost allocation: table design for per-customer inference cost tracking with partition key on customer ID and sort key on date, TTL for data retention, atomic counter updates via `UpdateExpression ADD`, and GSI for model-level aggregation
- Lambda cost processors: event-driven Lambda functions that consume CloudWatch Logs subscription filters or Bedrock invocation logs, extract customer identifiers from request metadata, compute per-request cost using token-based pricing, and write cost records to DynamoDB
- Amazon QuickSight: `quicksight.create_data_set()` from DynamoDB or Athena sources, `create_analysis()` with visual types (line charts, bar charts, pie charts, KPI widgets, pivot tables), calculated fields for margin analysis, and scheduled email reports
- Bedrock and SageMaker pricing: token-based pricing models (per 1K input/output tokens for Bedrock), instance-hour pricing for SageMaker endpoints, cost attribution across multiple models and endpoints
- Cost alerting: CloudWatch alarms on metric math expressions for cost thresholds, composite alarms for multi-model cost monitoring, SNS notifications with cost context
- CSV report generation: Lambda-based monthly report generator that queries DynamoDB, aggregates costs by customer/model/date, and writes CSV to S3 for finance team consumption
- IAM for cost analytics: least-privilege roles for CloudWatch, DynamoDB, QuickSight, Lambda, S3, and SSM access

Generate complete, production-deployable cost-per-inference analytics code.

---

## Context & Inputs

```
PROJECT_NAME:               [REQUIRED]
AWS_REGION:                 [REQUIRED]
AWS_ACCOUNT_ID:             [REQUIRED]
ENV:                        [REQUIRED - dev | stage | prod]

ENDPOINTS_TO_TRACK:         [REQUIRED - JSON list of SageMaker endpoint names to track]
                            List of SageMaker endpoint names for cost-per-inference tracking.
                            Each endpoint gets its own cost metrics and dashboard widgets.
                            Example: ["text-classifier-prod", "embedding-service-prod"]

BEDROCK_MODELS_TO_TRACK:    [REQUIRED - JSON list of Bedrock model IDs to track]
                            List of Bedrock model IDs for cost-per-inference tracking.
                            Pricing is computed from token counts in invocation logs.
                            Example: ["us.anthropic.claude-sonnet-4-7-20260109-v1:0", "amazon.titan-text-express-v1"]

CUSTOMER_TRACKING_ENABLED:  [OPTIONAL: false]
                            Enable per-customer cost allocation tracking.
                            When true, generates a DynamoDB table and Lambda processor
                            that attributes inference costs to individual customers or
                            business units based on request metadata (API key, tenant ID,
                            or custom header).
                            Requires inference requests to include a customer identifier.

QUICKSIGHT_ENABLED:         [OPTIONAL: false]
                            Enable Amazon QuickSight analysis and dashboard creation.
                            Requires QuickSight Enterprise Edition in the account.
                            When true, generates QuickSight datasets, analyses, and
                            scheduled email reports for cost visualization.

COST_ALERT_THRESHOLD:       [OPTIONAL: 100]
                            Daily cost threshold in USD that triggers a CloudWatch alarm.
                            When the estimated daily inference cost across all tracked
                            endpoints/models exceeds this value, an SNS notification is sent.
                            Example: 100 means alarm fires when daily cost > $100.

REPORT_FREQUENCY:           [OPTIONAL: monthly]
                            Frequency for CSV cost report generation.
                            Options: daily, weekly, monthly
                            Reports are written to S3 and optionally emailed via SNS.

CUSTOMER_ID_FIELD:          [OPTIONAL: customer_id]
                            JSON field name in inference request metadata that contains
                            the customer or tenant identifier for cost attribution.
                            Example: "customer_id", "tenant_id", "api_key_id"

BEDROCK_LOG_GROUP:          [OPTIONAL: /aws/bedrock/{PROJECT_NAME}/{ENV}/invocations]
                            CloudWatch Logs log group containing Bedrock invocation logs.
                            Typically created by devops/12 (Bedrock Invocation Logging).
                            Read from SSM: /mlops/{PROJECT_NAME}/{ENV}/bedrock-log-group

SAGEMAKER_METRIC_NAMESPACE: [OPTIONAL: AWS/SageMaker]
                            CloudWatch metric namespace for SageMaker endpoint metrics.
                            Default is the built-in AWS/SageMaker namespace.

CUSTOM_METRIC_NAMESPACE:    [OPTIONAL: {PROJECT_NAME}/MLModelQuality]
                            Custom CloudWatch metric namespace from devops/11.
                            Used for token count metrics (InputTokens, OutputTokens).
                            Read from SSM: /mlops/{PROJECT_NAME}/{ENV}/cw-metric-namespace

SNS_TOPIC_ARN:              [OPTIONAL: none]
                            SNS topic ARN for cost alert notifications and report delivery.
                            If not provided, creates a new SNS topic.

REPORT_S3_BUCKET:           [OPTIONAL: {PROJECT_NAME}-cost-reports-{ENV}]
                            S3 bucket for storing generated CSV cost reports.

PRICING_TABLE:              [OPTIONAL: embedded]
                            Source for Bedrock model pricing data.
                            Options:
                            - embedded: Use a hardcoded pricing dictionary in the code
                            - athena: Query pricing from Athena table created by devops/12
                            - ssm: Read pricing from SSM Parameter Store
```

---

## Task

Generate complete cost-per-inference analytics infrastructure:

```
{PROJECT_NAME}-cost-per-inference/
├── config/
│   └── config.py                         # Central configuration
├── pricing/
│   ├── pricing_data.py                   # Bedrock + SageMaker pricing reference
│   └── pricing_updater.py                # Refresh pricing from AWS Pricing API
├── metrics/
│   ├── cost_metric_math.py               # CloudWatch metric math for cost-per-inference
│   ├── endpoint_discovery.py             # Auto-discover new endpoints for tracking
│   └── sagemaker_cost_calculator.py      # SageMaker instance-hour cost computation
├── customer_tracking/
│   ├── dynamodb_table.py                 # DynamoDB per-customer cost allocation table
│   ├── cost_processor_lambda.py          # Lambda: parse logs → attribute cost to customer
│   └── subscription_setup.py            # CloudWatch Logs subscription filter
├── dashboard/
│   ├── cloudwatch_dashboard.py           # CloudWatch dashboard with cost widgets
│   └── quicksight_analysis.py            # QuickSight dataset + analysis (optional)
├── alerting/
│   ├── cost_alarms.py                    # Cost threshold alarms + SNS
│   └── anomaly_cost_detector.py          # Cost anomaly detection
├── reporting/
│   ├── csv_report_generator.py           # Monthly CSV cost report Lambda
│   ├── report_scheduler.py               # EventBridge schedule for report generation
│   └── report_s3_setup.py                # S3 bucket for reports
├── infrastructure/
│   ├── iam_roles.py                      # IAM roles for Lambda, QuickSight
│   └── ssm_outputs.py                    # Write outputs to SSM
├── run_setup.py                          # CLI orchestrator
└── requirements.txt
```

**config.py**: Central configuration dataclass with all parameters. Load from environment variables or CLI args. Validate required fields. Construct resource names using `{PROJECT_NAME}-{component}-{ENV}` convention. Parse ENDPOINTS_TO_TRACK and BEDROCK_MODELS_TO_TRACK from JSON strings. Validate that at least one endpoint or model is specified. Set defaults for CUSTOMER_TRACKING_ENABLED, QUICKSIGHT_ENABLED, COST_ALERT_THRESHOLD, REPORT_FREQUENCY. Validate that CUSTOMER_ID_FIELD is provided when CUSTOMER_TRACKING_ENABLED is true.

**pricing_data.py**: Bedrock and SageMaker pricing reference:
- Define a Python dictionary `BEDROCK_PRICING` mapping `modelId` to `{"input_per_1k_tokens": float, "output_per_1k_tokens": float, "provider": str}` for common models: Claude Sonnet 4.7, Claude Haiku 4.5, Titan Text Express, Titan Text Lite, Llama 3 models, Mistral models.
- Define a Python dictionary `SAGEMAKER_PRICING` mapping instance type to hourly cost in USD for common inference instances: ml.m5.xlarge, ml.g5.xlarge, ml.g5.2xlarge, ml.inf2.xlarge, ml.c7g.xlarge.
- `get_bedrock_cost(model_id, input_tokens, output_tokens)`: compute total cost for a single Bedrock invocation: `(input_tokens / 1000) * input_price + (output_tokens / 1000) * output_price`.
- `get_sagemaker_cost_per_invocation(instance_type, invocations_per_hour)`: compute per-invocation cost for a SageMaker endpoint: `hourly_cost / invocations_per_hour`.
- `get_pricing_for_model(model_id)`: return pricing dict for a model ID, with fallback to a default pricing if model not found.

**pricing_updater.py**: Refresh pricing from AWS Pricing API:
- `fetch_bedrock_pricing()`: call `pricing.get_products(ServiceCode='AmazonBedrock')` to retrieve current Bedrock pricing. Parse the JSON response to extract per-model input/output token prices. Update the `BEDROCK_PRICING` dictionary.
- `fetch_sagemaker_pricing(region)`: call `pricing.get_products(ServiceCode='AmazonSageMaker')` filtered by region and instance type to retrieve current SageMaker instance pricing.
- `save_pricing_to_ssm()`: write pricing data to SSM Parameter Store at `/mlops/{PROJECT_NAME}/{ENV}/bedrock-pricing` as a JSON string for cross-template consumption.
- `load_pricing_from_ssm()`: read pricing data from SSM as an alternative to the embedded dictionary.

**cost_metric_math.py**: CloudWatch metric math expressions for cost-per-inference:
- `get_bedrock_cost_per_invocation_expression(model_id)`: return metric math expression that computes cost per Bedrock invocation:
  - `m1`: `{CUSTOM_METRIC_NAMESPACE}/InputTokens` SUM with dimension `EndpointName={model_id}`
  - `m2`: `{CUSTOM_METRIC_NAMESPACE}/OutputTokens` SUM with dimension `EndpointName={model_id}`
  - `m3`: `{CUSTOM_METRIC_NAMESPACE}/InvocationCount` SUM with dimension `EndpointName={model_id}`
  - `e1`: `((m1 * {input_price} / 1000) + (m2 * {output_price} / 1000)) / m3` — cost per invocation in USD
- `get_sagemaker_cost_per_invocation_expression(endpoint_name)`: return metric math expression:
  - `m1`: `AWS/SageMaker/Invocations` SUM with dimension `EndpointName={endpoint_name}`
  - `e1`: `{hourly_cost} / 3600 * PERIOD / m1` — instance cost per invocation based on time slice
- `get_daily_cost_expression(model_id)`: return metric math expression for total daily cost:
  - `e1`: `(m1 * {input_price} / 1000) + (m2 * {output_price} / 1000)` — total token cost per period
- `create_cost_per_invocation_widget(name, expression_type)`: return a CloudWatch dashboard widget JSON definition displaying cost-per-invocation as a line graph with threshold annotation.

**endpoint_discovery.py**: Auto-discover new endpoints for tracking:
- `discover_sagemaker_endpoints()`: call `sagemaker.list_endpoints(StatusEquals='InService')` to find all active SageMaker endpoints. Filter by tag `Project={PROJECT_NAME}` using `sagemaker.list_tags()`. Return list of endpoint names not already in ENDPOINTS_TO_TRACK.
- `discover_bedrock_models()`: call `bedrock.list_foundation_models()` and cross-reference with CloudWatch metrics to find models with recent invocations. Return list of model IDs not already in BEDROCK_MODELS_TO_TRACK.
- `register_new_endpoints(new_endpoints)`: update the CloudWatch dashboard to include widgets for newly discovered endpoints. Update DynamoDB tracking configuration. Send SNS notification about new endpoints added to tracking.
- `create_discovery_lambda()`: generate a Lambda function triggered by EventBridge scheduled rule (daily) that runs endpoint discovery and auto-registers new endpoints.

**sagemaker_cost_calculator.py**: SageMaker instance-hour cost computation:
- `get_endpoint_config(endpoint_name)`: call `sagemaker.describe_endpoint()` and `sagemaker.describe_endpoint_config()` to retrieve instance type, instance count, and variant weights.
- `compute_hourly_cost(endpoint_name)`: compute total hourly cost: `instance_count * hourly_price_per_instance`. Support multi-variant endpoints by summing across variants.
- `compute_cost_per_invocation(endpoint_name, period_seconds=300)`: query CloudWatch for `Invocations` metric over the period, compute `hourly_cost * (period_seconds / 3600) / invocation_count`.
- `publish_sagemaker_cost_metrics(endpoint_name)`: publish `CostPerInvocation` and `HourlyCost` as custom CloudWatch metrics under `{PROJECT_NAME}/InferenceCost` namespace.

**dynamodb_table.py**: DynamoDB per-customer cost allocation table (generated only if CUSTOMER_TRACKING_ENABLED is true):
- Create DynamoDB table `{PROJECT_NAME}-customer-inference-costs-{ENV}` with:
  - Partition key: `customer_id` (String) — customer or tenant identifier
  - Sort key: `date_model` (String) — composite key `{YYYY-MM-DD}#{model_id}` for per-day per-model granularity
  - Attributes: `input_tokens` (Number), `output_tokens` (Number), `invocation_count` (Number), `total_cost_usd` (Number), `updated_at` (String)
  - TTL attribute: `ttl_epoch` — set to 90 days after creation for automatic data expiration
  - GSI `model-date-index`: partition key `model_id`, sort key `date` for model-level aggregation queries
  - GSI `date-index`: partition key `date`, sort key `customer_id` for daily reporting queries
  - Billing mode: PAY_PER_REQUEST
  - Tags: `Project={PROJECT_NAME}`, `Environment={ENV}`, `Purpose=cost-allocation`
- `update_customer_cost(customer_id, model_id, date, input_tokens, output_tokens, cost)`: use `UpdateExpression` with `ADD` to atomically increment token counts, invocation count, and cost for the customer/date/model combination.
- `get_customer_costs(customer_id, start_date, end_date)`: query costs for a customer over a date range.
- `get_model_costs(model_id, start_date, end_date)`: query GSI for model-level cost aggregation.
- `get_daily_summary(date)`: query date-index GSI for all customers on a given date.

**cost_processor_lambda.py**: Lambda function that parses inference logs and attributes cost to customers (generated only if CUSTOMER_TRACKING_ENABLED is true):
- Triggered by CloudWatch Logs subscription filter on BEDROCK_LOG_GROUP.
- For each log event:
  1. Parse JSON log line to extract: `modelId`, `inputTokenCount`, `outputTokenCount`, `requestId`, and custom metadata field `CUSTOMER_ID_FIELD` (from request headers or session attributes).
  2. Look up pricing for `modelId` using `pricing_data.get_bedrock_cost()`.
  3. Compute per-request cost: `(inputTokenCount / 1000) * input_price + (outputTokenCount / 1000) * output_price`.
  4. Call `dynamodb_table.update_customer_cost()` to atomically increment the customer's cost record.
  5. Publish `CustomerCost` custom metric with dimensions `CustomerId` and `ModelId`.
- Batch DynamoDB writes using `batch_writer()` for efficiency when processing multiple log events.
- Handle missing customer ID gracefully: attribute to `unknown` customer and log a warning.
- Handle pricing lookup failures: use a default fallback price and log a warning.
- Lambda execution role needs: DynamoDB read/write, CloudWatch PutMetricData, CloudWatch Logs read.

**subscription_setup.py**: CloudWatch Logs subscription filter for cost processor:
- `create_subscription_filter(log_group_name, lambda_arn)`: call `logs.put_subscription_filter()` to subscribe the cost processor Lambda to the Bedrock invocation log group.
- Filter pattern: `{ $.modelId = * }` to match all Bedrock invocation log entries.
- Grant the log group permission to invoke the Lambda using `lambda_client.add_permission()`.
- `delete_subscription_filter()` for cleanup.

**cloudwatch_dashboard.py**: CloudWatch dashboard with cost-per-inference widgets:
- `build_dashboard(endpoints, bedrock_models)`: construct a CloudWatch dashboard JSON body with widgets:
  - Row 1: Title text widget "Cost-per-Inference Dashboard — {PROJECT_NAME} ({ENV})" + alarm status widget showing all cost alarms
  - Row 2: Per-model cost-per-invocation line graph (one line per Bedrock model) using metric math, daily total cost bar chart across all models
  - Row 3: Per-endpoint cost-per-invocation line graph (one line per SageMaker endpoint) using metric math, endpoint hourly cost line graph
  - Row 4: Token usage stacked area chart (input vs output tokens by model), invocation volume line graph by model
  - Row 5 (if CUSTOMER_TRACKING_ENABLED): Per-customer daily cost bar chart (top 10 customers), customer cost distribution pie chart
  - Row 6: Number widgets showing current KPIs: avg cost per invocation (all models), total daily cost estimate, most expensive model, total invocations today
  - Row 7: Cost trend line graph (7-day rolling average) with threshold annotation at COST_ALERT_THRESHOLD
- Each metric widget uses appropriate dimensions and metric math expressions from `cost_metric_math.py`.
- `create_dashboard()`: call `cloudwatch.put_dashboard()` with the built JSON body.
- `update_dashboard(new_endpoints=None, new_models=None)`: rebuild and re-publish the dashboard when endpoints or models are added.
- Dashboard name: `{PROJECT_NAME}-cost-per-inference-{ENV}`.

**quicksight_analysis.py**: QuickSight dataset and analysis (generated only if QUICKSIGHT_ENABLED is true):
- `create_data_source()`: create a QuickSight data source connected to the DynamoDB customer cost table using `quicksight.create_data_source()` with `DynamoDBParameters`.
- `create_data_set()`: create a QuickSight dataset using `quicksight.create_data_set()` with:
  - Physical table map pointing to the DynamoDB table
  - Logical table map with calculated fields:
    - `cost_per_token`: `total_cost_usd / (input_tokens + output_tokens)`
    - `avg_cost_per_invocation`: `total_cost_usd / invocation_count`
    - `month`: extracted from `date_model` for monthly aggregation
  - SPICE import mode for fast query performance
  - Refresh schedule: daily
- `create_analysis()`: create a QuickSight analysis using `quicksight.create_analysis()` with visuals:
  - Line chart: daily cost trend by model over time
  - Stacked bar chart: monthly cost by customer (top 20)
  - Pie chart: cost distribution by model
  - KPI widget: total monthly cost, month-over-month change percentage
  - Pivot table: customer × model cost matrix
  - Scatter plot: invocation count vs total cost per customer (identify high-cost customers)
- `create_dashboard_from_analysis()`: publish the analysis as a shareable QuickSight dashboard.
- `schedule_email_report()`: configure QuickSight email report delivery on REPORT_FREQUENCY schedule.

**cost_alarms.py**: Cost threshold alarms with SNS:
- `create_daily_cost_alarm()`: create a CloudWatch alarm using metric math:
  - Sum all model costs over a 24-hour period using `METRICS("SEARCH('{CUSTOM_METRIC_NAMESPACE}', 'Sum', 86400)")` or explicit metric references
  - Threshold: COST_ALERT_THRESHOLD USD
  - Comparison: `GreaterThanThreshold`
  - Evaluation periods: 1 (24-hour period)
  - Alarm actions: SNS_TOPIC_ARN
  - Alarm name: `{PROJECT_NAME}-daily-cost-threshold-{ENV}`
- `create_per_model_cost_alarm(model_id, threshold)`: create per-model cost alarm for individual model cost spikes.
- `create_cost_spike_alarm()`: create an alarm that fires when cost increases by more than 50% compared to the previous day (using metric math with `PERIOD` offset).
- Create SNS topic `{PROJECT_NAME}-cost-alerts-{ENV}` if SNS_TOPIC_ARN not provided.
- SNS message includes: alarm name, current cost, threshold, top contributing model, and recommended actions.

**anomaly_cost_detector.py**: Cost anomaly detection:
- `create_cost_anomaly_detector()`: call `cloudwatch.put_anomaly_detector()` on the daily cost metric to learn normal spending patterns. The detector identifies unusual cost spikes that may not exceed the static threshold but deviate from historical patterns.
- `create_anomaly_alarm()`: create a CloudWatch alarm using `ANOMALY_DETECTION_BAND(m1, 2)` on the daily cost metric. Fires when cost falls outside the expected band.
- Anomaly detection requires 2 weeks of data to train — include a note in setup output.

**csv_report_generator.py**: Lambda function for CSV cost report generation:
- Lambda function `{PROJECT_NAME}-cost-report-generator-{ENV}` triggered by EventBridge scheduled rule.
- Query DynamoDB customer cost table for the reporting period (daily/weekly/monthly based on REPORT_FREQUENCY).
- Aggregate costs by:
  - Customer: total cost, invocation count, avg cost per invocation, top model by cost
  - Model: total cost, total tokens, invocation count, avg cost per invocation
  - Date: daily cost breakdown for trend analysis
- Generate CSV files:
  - `{YYYY-MM}/customer_costs_{period}.csv`: per-customer cost breakdown
  - `{YYYY-MM}/model_costs_{period}.csv`: per-model cost breakdown
  - `{YYYY-MM}/daily_summary_{period}.csv`: daily cost summary
- Upload CSV files to S3: `s3://{REPORT_S3_BUCKET}/reports/{YYYY-MM}/`
- Send SNS notification with report summary and S3 links.
- Include a `generate_chargeback_report(month)` function that produces a finance-ready chargeback CSV with columns: `customer_id`, `model_id`, `total_invocations`, `total_input_tokens`, `total_output_tokens`, `total_cost_usd`, `period`.

**report_scheduler.py**: EventBridge schedule for report generation:
- Create EventBridge scheduled rule `{PROJECT_NAME}-cost-report-schedule-{ENV}` with:
  - `daily`: `rate(1 day)`
  - `weekly`: `cron(0 8 ? * MON *)`  — every Monday at 8 AM UTC
  - `monthly`: `cron(0 8 1 * ? *)`  — first day of each month at 8 AM UTC
- Target: cost report generator Lambda ARN.
- Include input payload with reporting period parameters.

**report_s3_setup.py**: S3 bucket for cost reports:
- Create S3 bucket `{REPORT_S3_BUCKET}` with:
  - Server-side encryption (SSE-S3)
  - Block public access enabled
  - Lifecycle rule: transition reports to Glacier after 365 days, expire after 2555 days (7 years for financial records)
  - Bucket policy restricting access to the report generator Lambda role and finance team IAM roles
  - Tags: `Project={PROJECT_NAME}`, `Environment={ENV}`, `Purpose=cost-reports`

**iam_roles.py**: IAM roles for Lambda functions and QuickSight:
- Cost processor Lambda role `{PROJECT_NAME}-cost-processor-role-{ENV}`:
  - Trust: `lambda.amazonaws.com`
  - Permissions: DynamoDB read/write on customer cost table, CloudWatch PutMetricData, CloudWatch Logs (read source log group, write own logs), SSM GetParameter
- Report generator Lambda role `{PROJECT_NAME}-cost-report-role-{ENV}`:
  - Trust: `lambda.amazonaws.com`
  - Permissions: DynamoDB read on customer cost table, S3 PutObject on report bucket, SNS Publish, CloudWatch Logs
- QuickSight service role (if QUICKSIGHT_ENABLED):
  - Trust: `quicksight.amazonaws.com`
  - Permissions: DynamoDB read on customer cost table, S3 read on report bucket
- Endpoint discovery Lambda role:
  - Trust: `lambda.amazonaws.com`
  - Permissions: SageMaker ListEndpoints/DescribeEndpoint/ListTags, Bedrock ListFoundationModels, CloudWatch PutDashboard, SNS Publish

**ssm_outputs.py**: Write outputs to SSM Parameter Store:
- `/mlops/{PROJECT_NAME}/{ENV}/cost-dashboard-name` — CloudWatch dashboard name
- `/mlops/{PROJECT_NAME}/{ENV}/cost-alert-topic-arn` — SNS topic ARN for cost alerts
- `/mlops/{PROJECT_NAME}/{ENV}/customer-cost-table` — DynamoDB table name (if CUSTOMER_TRACKING_ENABLED)
- `/mlops/{PROJECT_NAME}/{ENV}/cost-report-bucket` — S3 bucket for reports

**run_setup.py**: CLI orchestrator that runs setup steps in order:
1. Load and validate configuration
2. Create IAM roles
3. Set up pricing data (embedded or from SSM/Athena)
4. Create DynamoDB customer cost table (if CUSTOMER_TRACKING_ENABLED)
5. Deploy cost processor Lambda and subscription filter (if CUSTOMER_TRACKING_ENABLED)
6. Create S3 report bucket
7. Deploy report generator Lambda and EventBridge schedule
8. Create CloudWatch cost-per-inference dashboard
9. Create cost threshold alarms and anomaly detector
10. Deploy endpoint discovery Lambda and schedule
11. Set up QuickSight data source, dataset, and analysis (if QUICKSIGHT_ENABLED)
12. Write outputs to SSM
13. Print summary with dashboard URL, alarm names, DynamoDB table, report bucket, and QuickSight dashboard URL

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**Cost-per-Inference Computation:** Cost per inference varies by model type. For Bedrock models, cost is token-based: `(input_tokens / 1000) * input_price + (output_tokens / 1000) * output_price`. For SageMaker endpoints, cost is instance-hour-based: `hourly_instance_cost / invocations_per_hour`. These are fundamentally different pricing models — the dashboard must handle both and normalize to a comparable "cost per invocation" metric. Use CloudWatch metric math to compute these ratios dynamically over any time period rather than publishing pre-computed values.

**CloudWatch Metric Math:** Use metric math expressions for all derived cost metrics. Key expressions: `e1 = ((m1 * input_price / 1000) + (m2 * output_price / 1000)) / m3` for Bedrock cost per invocation (where m1=InputTokens, m2=OutputTokens, m3=InvocationCount). Use `FILL(metric, 0)` to handle periods with zero invocations and avoid division-by-zero errors. Use `METRICS("SEARCH()")` expressions in dashboard widgets to auto-discover new metrics matching a pattern, enabling automatic tracking of new endpoints without dashboard updates.

**DynamoDB Per-Customer Tracking:** The customer cost allocation table uses atomic counter updates (`UpdateExpression: ADD`) to handle concurrent Lambda invocations writing to the same customer/date/model record. The composite sort key `{date}#{model_id}` enables efficient range queries for a customer's costs over a date range. GSIs on `model_id` and `date` support model-level and daily aggregation queries. TTL automatically expires records after 90 days to control table size — increase for longer retention requirements. Use PAY_PER_REQUEST billing to handle variable write volumes from inference traffic.

**Customer Identification:** The cost processor Lambda extracts customer identifiers from inference request metadata. For Bedrock, the customer ID can be passed in session attributes or custom request headers. For SageMaker, the customer ID can be included in the inference request payload or extracted from API Gateway request context (API key, Cognito identity). The CUSTOMER_ID_FIELD parameter specifies which JSON field contains the customer identifier. Requests without a customer identifier are attributed to an `unknown` customer.

**QuickSight Integration:** QuickSight Enterprise Edition is required for scheduled email reports and row-level security. The QuickSight data source connects to DynamoDB via the QuickSight DynamoDB connector. SPICE (Super-fast, Parallel, In-memory Calculation Engine) import mode provides fast query performance but requires periodic refresh. Configure daily SPICE refresh to keep the analysis current. QuickSight costs $18/month per author and $0.30/session per reader (up to $5/month cap). Consider whether the cost of QuickSight justifies the visualization benefits versus CloudWatch dashboards alone.

**Cost Alerting:** The daily cost threshold alarm uses a 24-hour evaluation period to compute total daily cost. The cost spike alarm compares current-period cost to the previous period using metric math with `PERIOD` offset. Anomaly detection learns normal spending patterns over 2 weeks and identifies unusual deviations. All three alarm types complement each other: static threshold catches absolute cost overruns, spike detection catches relative increases, and anomaly detection catches pattern deviations. Alarm actions send to SNS with structured messages including cost context.

**CSV Report Generation:** The report generator Lambda queries DynamoDB for the reporting period and produces CSV files suitable for finance team consumption. Reports include per-customer chargeback data, per-model cost breakdown, and daily summaries. CSV files are stored in S3 with a lifecycle policy that transitions to Glacier after 1 year and expires after 7 years (common financial record retention). Reports can be delivered via SNS email notification with S3 pre-signed URLs.

**Endpoint Discovery:** The auto-discovery Lambda runs on a daily EventBridge schedule and finds new SageMaker endpoints (filtered by project tag) and Bedrock models with recent invocations. New endpoints are automatically added to the CloudWatch dashboard and cost tracking configuration. This eliminates the need to manually update ENDPOINTS_TO_TRACK when new endpoints are deployed. Discovery results are published as SNS notifications for visibility.

**Pricing Data Management:** Bedrock and SageMaker pricing changes over time. The embedded pricing dictionary provides a starting point but should be refreshed periodically using the AWS Pricing API (`pricing.get_products()`). The pricing updater can store refreshed prices in SSM Parameter Store for cross-template consumption. Alternatively, the Athena pricing table created by devops/12 can be used as the pricing source. Always validate pricing data before computing costs — stale pricing leads to inaccurate cost attribution.

**Naming:** All resources follow `{PROJECT_NAME}-{component}-{ENV}` convention:
- CloudWatch dashboard: `{PROJECT_NAME}-cost-per-inference-{ENV}`
- DynamoDB table: `{PROJECT_NAME}-customer-inference-costs-{ENV}`
- Cost processor Lambda: `{PROJECT_NAME}-cost-processor-{ENV}`
- Report generator Lambda: `{PROJECT_NAME}-cost-report-generator-{ENV}`
- Discovery Lambda: `{PROJECT_NAME}-endpoint-discovery-{ENV}`
- CloudWatch alarm: `{PROJECT_NAME}-daily-cost-threshold-{ENV}`
- SNS topic: `{PROJECT_NAME}-cost-alerts-{ENV}`
- S3 report bucket: `{PROJECT_NAME}-cost-reports-{ENV}`
- SSM parameters: `/mlops/{PROJECT_NAME}/{ENV}/cost-dashboard-name`, etc.
- Metric namespace: `{PROJECT_NAME}/InferenceCost`

---

## Code Scaffolding Hints

**CloudWatch metric math for cost-per-invocation (Bedrock):**
```python
import boto3
import json

cloudwatch = boto3.client("cloudwatch", region_name=AWS_REGION)

COST_NAMESPACE = f"{PROJECT_NAME}/InferenceCost"
CUSTOM_NAMESPACE = f"{PROJECT_NAME}/MLModelQuality"


def get_bedrock_cost_per_invocation_widget(model_id, input_price, output_price):
    """Build a CloudWatch dashboard widget with metric math for Bedrock cost per invocation."""
    return {
        "type": "metric",
        "width": 12,
        "height": 6,
        "properties": {
            "title": f"Cost per Invocation — {model_id}",
            "view": "timeSeries",
            "stacked": False,
            "metrics": [
                [
                    {"expression": f"((m1 * {input_price} / 1000) + (m2 * {output_price} / 1000)) / m3",
                     "label": "Cost per Invocation (USD)", "id": "e1", "region": AWS_REGION}
                ],
                [CUSTOM_NAMESPACE, "InputTokens", "EndpointName", model_id,
                 {"id": "m1", "visible": False, "stat": "Sum"}],
                [CUSTOM_NAMESPACE, "OutputTokens", "EndpointName", model_id,
                 {"id": "m2", "visible": False, "stat": "Sum"}],
                [CUSTOM_NAMESPACE, "InvocationCount", "EndpointName", model_id,
                 {"id": "m3", "visible": False, "stat": "Sum"}],
            ],
            "period": 300,
            "region": AWS_REGION,
            "yAxis": {"left": {"label": "USD", "showUnits": False}},
            "annotations": {
                "horizontal": [
                    {"label": "Cost Alert", "value": 0.05, "color": "#d62728"}
                ]
            },
        },
    }


def get_daily_cost_widget(models_config):
    """Build a CloudWatch dashboard widget showing total daily cost across all models."""
    metrics = []
    expressions = []
    for i, (model_id, pricing) in enumerate(models_config.items()):
        input_price = pricing["input_per_1k_tokens"]
        output_price = pricing["output_per_1k_tokens"]
        m_in = f"m{i * 2 + 1}"
        m_out = f"m{i * 2 + 2}"
        expr_id = f"e{i + 1}"
        metrics.append(
            [CUSTOM_NAMESPACE, "InputTokens", "EndpointName", model_id,
             {"id": m_in, "visible": False, "stat": "Sum"}]
        )
        metrics.append(
            [CUSTOM_NAMESPACE, "OutputTokens", "EndpointName", model_id,
             {"id": m_out, "visible": False, "stat": "Sum"}]
        )
        expressions.append(
            {"expression": f"({m_in} * {input_price} / 1000) + ({m_out} * {output_price} / 1000)",
             "label": model_id, "id": expr_id, "region": AWS_REGION}
        )
    all_metrics = [[expr] for expr in expressions] + metrics
    return {
        "type": "metric",
        "width": 12,
        "height": 6,
        "properties": {
            "title": "Daily Cost by Model (USD)",
            "view": "timeSeries",
            "stacked": True,
            "metrics": all_metrics,
            "period": 86400,
            "region": AWS_REGION,
            "yAxis": {"left": {"label": "USD", "showUnits": False}},
        },
    }
```

**DynamoDB per-customer cost allocation table and update function:**
```python
import boto3
from datetime import datetime, timezone
from decimal import Decimal
import time

dynamodb = boto3.resource("dynamodb", region_name=AWS_REGION)
dynamodb_client = boto3.client("dynamodb", region_name=AWS_REGION)

TABLE_NAME = f"{PROJECT_NAME}-customer-inference-costs-{ENV}"


def create_customer_cost_table():
    """Create DynamoDB table for per-customer inference cost tracking."""
    table = dynamodb.create_table(
        TableName=TABLE_NAME,
        KeySchema=[
            {"AttributeName": "customer_id", "KeyType": "HASH"},
            {"AttributeName": "date_model", "KeyType": "RANGE"},
        ],
        AttributeDefinitions=[
            {"AttributeName": "customer_id", "AttributeType": "S"},
            {"AttributeName": "date_model", "AttributeType": "S"},
            {"AttributeName": "model_id", "AttributeType": "S"},
            {"AttributeName": "date", "AttributeType": "S"},
        ],
        GlobalSecondaryIndexes=[
            {
                "IndexName": "model-date-index",
                "KeySchema": [
                    {"AttributeName": "model_id", "KeyType": "HASH"},
                    {"AttributeName": "date", "KeyType": "RANGE"},
                ],
                "Projection": {"ProjectionType": "ALL"},
            },
            {
                "IndexName": "date-index",
                "KeySchema": [
                    {"AttributeName": "date", "KeyType": "HASH"},
                    {"AttributeName": "customer_id", "KeyType": "RANGE"},
                ],
                "Projection": {"ProjectionType": "ALL"},
            },
        ],
        BillingMode="PAY_PER_REQUEST",
        Tags=[
            {"Key": "Project", "Value": PROJECT_NAME},
            {"Key": "Environment", "Value": ENV},
            {"Key": "Purpose", "Value": "cost-allocation"},
        ],
    )
    table.wait_until_exists()

    # Enable TTL
    dynamodb_client.update_time_to_live(
        TableName=TABLE_NAME,
        TimeToLiveSpecification={"Enabled": True, "AttributeName": "ttl_epoch"},
    )
    print(f"Created DynamoDB table: {TABLE_NAME}")
    return table


def update_customer_cost(customer_id, model_id, input_tokens, output_tokens, cost_usd):
    """Atomically increment customer cost record for the current date and model."""
    table = dynamodb.Table(TABLE_NAME)
    today = datetime.now(timezone.utc).strftime("%Y-%m-%d")
    date_model = f"{today}#{model_id}"
    ttl_epoch = int(time.time()) + (90 * 86400)  # 90-day TTL

    table.update_item(
        Key={"customer_id": customer_id, "date_model": date_model},
        UpdateExpression=(
            "ADD input_tokens :inp, output_tokens :out, "
            "invocation_count :one, total_cost_usd :cost "
            "SET model_id = :mid, #d = :dt, updated_at = :now, ttl_epoch = :ttl"
        ),
        ExpressionAttributeNames={"#d": "date"},
        ExpressionAttributeValues={
            ":inp": input_tokens,
            ":out": output_tokens,
            ":one": 1,
            ":cost": Decimal(str(round(cost_usd, 8))),
            ":mid": model_id,
            ":dt": today,
            ":now": datetime.now(timezone.utc).isoformat(),
            ":ttl": ttl_epoch,
        },
    )
```

**Cost processor Lambda function:**
```python
import boto3
import json
import base64
import gzip
from datetime import datetime, timezone
from decimal import Decimal

dynamodb = boto3.resource("dynamodb")
cloudwatch = boto3.client("cloudwatch")

TABLE_NAME = f"{PROJECT_NAME}-customer-inference-costs-{ENV}"
COST_NAMESPACE = f"{PROJECT_NAME}/InferenceCost"
CUSTOMER_ID_FIELD = "customer_id"  # configurable

BEDROCK_PRICING = {
    "us.anthropic.claude-sonnet-4-7-20260109-v1:0": {"input": 0.003, "output": 0.015},
    "us.anthropic.claude-haiku-4-5-20251001-v1:0": {"input": 0.00025, "output": 0.00125},
    "us.anthropic.claude-sonnet-4-7-20260109-v1:0": {"input": 0.003, "output": 0.015},
    "amazon.titan-text-express-v1": {"input": 0.0002, "output": 0.0006},
}


def lambda_handler(event, context):
    """Process Bedrock invocation logs and attribute costs to customers."""
    # Decode and decompress CloudWatch Logs event
    payload = base64.b64decode(event["awslogs"]["data"])
    log_data = json.loads(gzip.decompress(payload))

    table = dynamodb.Table(TABLE_NAME)
    metric_data = []

    with table.batch_writer() as batch:
        for log_event in log_data.get("logEvents", []):
            try:
                record = json.loads(log_event["message"])
                model_id = record.get("modelId", "unknown")
                input_tokens = record.get("input", {}).get("inputTokenCount", 0)
                output_tokens = record.get("output", {}).get("outputTokenCount", 0)

                # Extract customer ID from request metadata
                customer_id = (
                    record.get("metadata", {}).get(CUSTOMER_ID_FIELD)
                    or record.get("sessionAttributes", {}).get(CUSTOMER_ID_FIELD)
                    or "unknown"
                )

                # Compute cost
                pricing = BEDROCK_PRICING.get(model_id, {"input": 0.003, "output": 0.015})
                cost = (input_tokens / 1000) * pricing["input"] + (output_tokens / 1000) * pricing["output"]

                # Update DynamoDB
                today = datetime.now(timezone.utc).strftime("%Y-%m-%d")
                batch.put_item(Item={
                    "customer_id": customer_id,
                    "date_model": f"{today}#{model_id}",
                    "model_id": model_id,
                    "date": today,
                    "input_tokens": input_tokens,
                    "output_tokens": output_tokens,
                    "invocation_count": 1,
                    "total_cost_usd": Decimal(str(round(cost, 8))),
                    "updated_at": datetime.now(timezone.utc).isoformat(),
                })

                # Collect metric data
                metric_data.append({
                    "MetricName": "CustomerInvocationCost",
                    "Value": cost,
                    "Unit": "None",
                    "Dimensions": [
                        {"Name": "CustomerId", "Value": customer_id},
                        {"Name": "ModelId", "Value": model_id},
                        {"Name": "Environment", "Value": ENV},
                    ],
                    "Timestamp": datetime.now(timezone.utc),
                })
            except (json.JSONDecodeError, KeyError) as e:
                print(f"Warning: skipping malformed log event: {e}")
                continue

    # Batch publish metrics
    for i in range(0, len(metric_data), 1000):
        cloudwatch.put_metric_data(
            Namespace=COST_NAMESPACE,
            MetricData=metric_data[i:i + 1000],
        )

    return {"processed": len(log_data.get("logEvents", [])), "metrics_published": len(metric_data)}
```

**QuickSight analysis creation:**
```python
import boto3

quicksight = boto3.client("quicksight", region_name=AWS_REGION)


def create_quicksight_data_source(account_id):
    """Create QuickSight data source connected to DynamoDB cost table."""
    quicksight.create_data_source(
        AwsAccountId=account_id,
        DataSourceId=f"{PROJECT_NAME}-cost-data-{ENV}",
        Name=f"{PROJECT_NAME} Inference Cost Data ({ENV})",
        Type="DYNAMODB",
        DataSourceParameters={
            "DynamoDBParameters": {
                "TableName": f"{PROJECT_NAME}-customer-inference-costs-{ENV}",
            }
        },
        Permissions=[
            {
                "Principal": f"arn:aws:quicksight:{AWS_REGION}:{account_id}:user/default/admin",
                "Actions": [
                    "quicksight:DescribeDataSource",
                    "quicksight:DescribeDataSourcePermissions",
                    "quicksight:PassDataSource",
                    "quicksight:UpdateDataSource",
                    "quicksight:DeleteDataSource",
                    "quicksight:UpdateDataSourcePermissions",
                ],
            }
        ],
        Tags=[
            {"Key": "Project", "Value": PROJECT_NAME},
            {"Key": "Environment", "Value": ENV},
        ],
    )
    print(f"Created QuickSight data source: {PROJECT_NAME}-cost-data-{ENV}")


def create_quicksight_analysis(account_id, data_set_id):
    """Create QuickSight analysis with cost visualization visuals."""
    quicksight.create_analysis(
        AwsAccountId=account_id,
        AnalysisId=f"{PROJECT_NAME}-cost-analysis-{ENV}",
        Name=f"{PROJECT_NAME} Inference Cost Analysis ({ENV})",
        SourceEntity={
            "SourceTemplate": {
                "DataSetReferences": [
                    {
                        "DataSetPlaceholder": "cost_data",
                        "DataSetArn": f"arn:aws:quicksight:{AWS_REGION}:{account_id}:dataset/{data_set_id}",
                    }
                ],
                "Arn": f"arn:aws:quicksight:{AWS_REGION}:{account_id}:template/cost-analysis-template",
            }
        },
        Tags=[
            {"Key": "Project", "Value": PROJECT_NAME},
            {"Key": "Environment", "Value": ENV},
        ],
    )
    print(f"Created QuickSight analysis: {PROJECT_NAME}-cost-analysis-{ENV}")
```

**CSV report generation:**
```python
import boto3
import csv
import io
from datetime import datetime, timezone, timedelta
from boto3.dynamodb.conditions import Key

dynamodb = boto3.resource("dynamodb", region_name=AWS_REGION)
s3 = boto3.client("s3", region_name=AWS_REGION)
sns = boto3.client("sns", region_name=AWS_REGION)

TABLE_NAME = f"{PROJECT_NAME}-customer-inference-costs-{ENV}"
REPORT_BUCKET = f"{PROJECT_NAME}-cost-reports-{ENV}"


def generate_monthly_chargeback_report(year, month):
    """Generate a chargeback-ready CSV report for a given month."""
    table = dynamodb.Table(TABLE_NAME)
    start_date = f"{year}-{month:02d}-01"
    if month == 12:
        end_date = f"{year + 1}-01-01"
    else:
        end_date = f"{year}-{month + 1:02d}-01"

    # Scan with date filter (for large datasets, use GSI query instead)
    response = table.scan(
        FilterExpression=Key("date").between(start_date, end_date),
        ProjectionExpression="customer_id, model_id, #d, input_tokens, output_tokens, invocation_count, total_cost_usd",
        ExpressionAttributeNames={"#d": "date"},
    )
    items = response.get("Items", [])
    while "LastEvaluatedKey" in response:
        response = table.scan(
            FilterExpression=Key("date").between(start_date, end_date),
            ExclusiveStartKey=response["LastEvaluatedKey"],
            ProjectionExpression="customer_id, model_id, #d, input_tokens, output_tokens, invocation_count, total_cost_usd",
            ExpressionAttributeNames={"#d": "date"},
        )
        items.extend(response.get("Items", []))

    # Aggregate by customer and model
    aggregated = {}
    for item in items:
        key = (item["customer_id"], item["model_id"])
        if key not in aggregated:
            aggregated[key] = {
                "customer_id": item["customer_id"],
                "model_id": item["model_id"],
                "total_invocations": 0,
                "total_input_tokens": 0,
                "total_output_tokens": 0,
                "total_cost_usd": 0,
            }
        aggregated[key]["total_invocations"] += int(item.get("invocation_count", 0))
        aggregated[key]["total_input_tokens"] += int(item.get("input_tokens", 0))
        aggregated[key]["total_output_tokens"] += int(item.get("output_tokens", 0))
        aggregated[key]["total_cost_usd"] += float(item.get("total_cost_usd", 0))

    # Write CSV
    output = io.StringIO()
    writer = csv.DictWriter(output, fieldnames=[
        "customer_id", "model_id", "total_invocations",
        "total_input_tokens", "total_output_tokens", "total_cost_usd", "period",
    ])
    writer.writeheader()
    for record in sorted(aggregated.values(), key=lambda x: x["total_cost_usd"], reverse=True):
        record["total_cost_usd"] = round(record["total_cost_usd"], 4)
        record["period"] = f"{year}-{month:02d}"
        writer.writerow(record)

    # Upload to S3
    csv_key = f"reports/{year}-{month:02d}/chargeback_{year}_{month:02d}.csv"
    s3.put_object(
        Bucket=REPORT_BUCKET,
        Key=csv_key,
        Body=output.getvalue().encode("utf-8"),
        ContentType="text/csv",
        ServerSideEncryption="AES256",
    )
    print(f"Uploaded chargeback report: s3://{REPORT_BUCKET}/{csv_key}")
    return csv_key
```

**Cost threshold alarm with metric math:**
```python
cloudwatch = boto3.client("cloudwatch", region_name=AWS_REGION)
sns = boto3.client("sns", region_name=AWS_REGION)

COST_NAMESPACE = f"{PROJECT_NAME}/InferenceCost"


def create_cost_alert_topic():
    """Create SNS topic for cost alert notifications."""
    response = sns.create_topic(
        Name=f"{PROJECT_NAME}-cost-alerts-{ENV}",
        Tags=[
            {"Key": "Project", "Value": PROJECT_NAME},
            {"Key": "Environment", "Value": ENV},
        ],
    )
    topic_arn = response["TopicArn"]
    print(f"Created SNS topic: {topic_arn}")
    return topic_arn


def create_daily_cost_alarm(sns_topic_arn, threshold_usd):
    """Create CloudWatch alarm for daily inference cost threshold."""
    cloudwatch.put_metric_alarm(
        AlarmName=f"{PROJECT_NAME}-daily-cost-threshold-{ENV}",
        AlarmDescription=f"Daily inference cost exceeds ${threshold_usd} USD",
        Metrics=[
            {
                "Id": "m1",
                "MetricStat": {
                    "Metric": {
                        "Namespace": COST_NAMESPACE,
                        "MetricName": "CustomerInvocationCost",
                    },
                    "Period": 86400,
                    "Stat": "Sum",
                },
                "ReturnData": False,
            },
            {
                "Id": "e1",
                "Expression": "m1",
                "Label": "Daily Total Cost (USD)",
                "ReturnData": True,
            },
        ],
        ComparisonOperator="GreaterThanThreshold",
        Threshold=threshold_usd,
        EvaluationPeriods=1,
        DatapointsToAlarm=1,
        TreatMissingData="notBreaching",
        AlarmActions=[sns_topic_arn],
        OKActions=[sns_topic_arn],
        Tags=[
            {"Key": "Project", "Value": PROJECT_NAME},
            {"Key": "Environment", "Value": ENV},
        ],
    )
    print(f"Created daily cost alarm: threshold=${threshold_usd}")


def create_cost_spike_alarm(sns_topic_arn):
    """Create alarm for cost spikes (>50% increase vs previous period)."""
    cloudwatch.put_metric_alarm(
        AlarmName=f"{PROJECT_NAME}-cost-spike-{ENV}",
        AlarmDescription="Inference cost spiked >50% compared to previous period",
        Metrics=[
            {
                "Id": "m1",
                "MetricStat": {
                    "Metric": {
                        "Namespace": COST_NAMESPACE,
                        "MetricName": "CustomerInvocationCost",
                    },
                    "Period": 86400,
                    "Stat": "Sum",
                },
                "ReturnData": False,
            },
            {
                "Id": "e1",
                "Expression": "RATE(m1)",
                "Label": "Cost Rate of Change",
                "ReturnData": True,
            },
        ],
        ComparisonOperator="GreaterThanThreshold",
        Threshold=0.5,
        EvaluationPeriods=1,
        DatapointsToAlarm=1,
        TreatMissingData="notBreaching",
        AlarmActions=[sns_topic_arn],
        Tags=[
            {"Key": "Project", "Value": PROJECT_NAME},
            {"Key": "Environment", "Value": ENV},
        ],
    )
    print(f"Created cost spike alarm: >50% increase detection")
```

**Write cost analytics outputs to SSM:**
```python
ssm = boto3.client("ssm", region_name=AWS_REGION)


def write_cost_outputs_to_ssm(dashboard_name, sns_topic_arn, dynamodb_table=None, report_bucket=None):
    """Write cost analytics resource identifiers to SSM for cross-template consumption."""
    params = {
        f"/mlops/{PROJECT_NAME}/{ENV}/cost-dashboard-name": dashboard_name,
        f"/mlops/{PROJECT_NAME}/{ENV}/cost-alert-topic-arn": sns_topic_arn,
    }
    if dynamodb_table:
        params[f"/mlops/{PROJECT_NAME}/{ENV}/customer-cost-table"] = dynamodb_table
    if report_bucket:
        params[f"/mlops/{PROJECT_NAME}/{ENV}/cost-report-bucket"] = report_bucket

    for name, value in params.items():
        ssm.put_parameter(
            Name=name,
            Description=f"Cost analytics output for {PROJECT_NAME} {ENV}",
            Value=value,
            Type="String",
            Overwrite=True,
            Tags=[
                {"Key": "Project", "Value": PROJECT_NAME},
                {"Key": "Environment", "Value": ENV},
            ],
        )
        print(f"SSM: {name} = {value}")
```

---

## Integration Points

- **Upstream**: `devops/11` → Custom CloudWatch metrics (InputTokens, OutputTokens, InvocationCount, TokensPerSecond) published by the model quality monitoring template provide the raw metric data for cost-per-inference metric math expressions
- **Upstream**: `devops/12` → Bedrock invocation logging provides the CloudWatch Logs log group and S3-stored invocation logs that the cost processor Lambda consumes; the Athena pricing table and cost-per-invocation queries from devops/12 can be reused as the pricing data source
- **Upstream**: `finops/04` → Inference cost optimization template provides SageMaker endpoint configuration data (instance types, instance counts, scaling policies) needed to compute per-invocation cost for SageMaker endpoints
- **Downstream**: `finops/05` → FinOps dashboards consume the CloudWatch cost-per-inference metrics, DynamoDB customer cost data, and CSV reports produced by this template for executive-level cost visualization
- **Downstream**: `finops/01` → Cost allocation template consumes the per-customer and per-model cost data from DynamoDB and the cost alert SNS topic for integration into organization-wide cost tracking and budget enforcement
