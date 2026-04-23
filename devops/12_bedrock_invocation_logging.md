<!-- Template Version: 1.1 | boto3: 1.35+ | Model IDs: 2026-04-22 refresh -->

# Template DevOps 12 — Bedrock Invocation Logging and Analysis

## Purpose
Generate production-ready Bedrock invocation logging infrastructure: enable Bedrock model invocation logging to CloudWatch Logs and/or S3, create Athena table DDL and named queries for analyzing invocation logs in S3 (invocations by model, average latency, token distribution, error rate), build cost-per-invocation tracking by joining invocation logs with Bedrock pricing data, provide a CloudWatch Logs Insights query library for real-time log analysis, configure error rate CloudWatch alarms with an optional model failover Lambda that switches to a backup model when error thresholds are breached, and set up S3 lifecycle rules for log archival transitioning from Standard to Glacier.

---

## Role Definition

You are an expert AWS observability engineer and Bedrock operations specialist with expertise in:
- Amazon Bedrock invocation logging: `bedrock.put_model_invocation_logging_configuration()` for enabling logging to CloudWatch Logs and S3 destinations
- Bedrock log schema: understanding the JSON structure of invocation logs including `modelId`, `inputTokenCount`, `outputTokenCount`, `invocationLatency`, `statusCode`, `errorCode`, `timestamp`, and request/response payloads
- Amazon Athena: creating external tables over S3-stored JSON logs, writing performant Presto/Trino SQL queries, named queries for reusable analytics, and partition projection for time-based partitioning
- CloudWatch Logs Insights: writing structured query syntax (`fields`, `filter`, `stats`, `sort`, `parse`) for real-time log analysis across Bedrock invocation log groups
- CloudWatch Alarms: metric filters on log groups, alarm math expressions for error rate computation, composite alarms, and SNS notification targets
- S3 lifecycle management: lifecycle rules for transitioning log data through storage classes (Standard → Intelligent-Tiering → Glacier Instant Retrieval → Glacier Deep Archive) and expiration policies
- Cost analysis: joining invocation logs with Bedrock pricing data to compute per-invocation and per-model cost breakdowns, token-based cost attribution
- Lambda-based failover: automated model switching when error rate thresholds are breached, using DynamoDB for active model configuration and CloudWatch alarm actions
- IAM for logging: service-linked roles and resource policies for Bedrock to write to CloudWatch Logs and S3

Generate complete, production-deployable Bedrock invocation logging and analysis code.

---

## Context & Inputs

```
PROJECT_NAME:               [REQUIRED]
AWS_REGION:                 [REQUIRED]
AWS_ACCOUNT_ID:             [REQUIRED]
ENV:                        [REQUIRED - dev | stage | prod]

LOG_DESTINATIONS:           [REQUIRED - comma-separated list of log destinations]
                            Options: cloudwatch_logs, s3
                            Example: "cloudwatch_logs,s3"
                            - cloudwatch_logs: Stream invocation logs to a CloudWatch Logs log group
                              for real-time Logs Insights queries and metric filters.
                            - s3: Deliver invocation logs to an S3 bucket for long-term storage
                              and Athena-based batch analytics.
                            Both destinations can be enabled simultaneously.

S3_LOG_BUCKET:              [OPTIONAL: {PROJECT_NAME}-bedrock-logs-{ENV}]
                            S3 bucket name for storing Bedrock invocation logs.
                            Required if "s3" is in LOG_DESTINATIONS.
                            Logs are written as JSON files partitioned by date.
                            Example: "myai-bedrock-logs-prod"

CLOUDWATCH_LOG_GROUP:       [OPTIONAL: /aws/bedrock/{PROJECT_NAME}/{ENV}/invocations]
                            CloudWatch Logs log group name for invocation logs.
                            Required if "cloudwatch_logs" is in LOG_DESTINATIONS.

ATHENA_DATABASE:            [OPTIONAL: {PROJECT_NAME}_bedrock_analytics_{ENV}]
                            Athena database name for querying S3-stored invocation logs.
                            Only used if "s3" is in LOG_DESTINATIONS.
                            Example: "myai_bedrock_analytics_prod"

ATHENA_RESULTS_BUCKET:      [OPTIONAL: {PROJECT_NAME}-athena-results-{ENV}]
                            S3 bucket for Athena query results.

LOG_RETENTION_DAYS:         [OPTIONAL: 90]
                            CloudWatch Logs retention period in days.
                            Common values: 30 (dev), 90 (stage), 365 (prod).

ALERT_ERROR_RATE_THRESHOLD: [OPTIONAL: 5]
                            Error rate percentage threshold for CloudWatch alarm.
                            When the Bedrock invocation error rate exceeds this value
                            over a 5-minute window, the alarm triggers.
                            Example: 5 means alarm fires when >5% of invocations fail.

FAILOVER_ENABLED:           [OPTIONAL: false]
                            Enable automatic model failover when error rate alarm fires.
                            When true, generates a Lambda function that switches the
                            active model ID in a DynamoDB configuration table.
                            Requires application code to read the active model from DynamoDB.

FAILOVER_MODEL_ID:          [OPTIONAL: none]
                            Backup model ID to failover to when the primary model errors.
                            Example: "amazon.titan-text-express-v1"
                            Required if FAILOVER_ENABLED is true.

PRIMARY_MODEL_ID:           [OPTIONAL: none]
                            Primary model ID being monitored.
                            Example: "us.anthropic.claude-sonnet-4-7-20260109-v1:0"
                            Used for cost tracking and failover configuration.

LOG_MODEL_INPUT_OUTPUT:     [OPTIONAL: false]
                            Whether to include full request/response payloads in logs.
                            WARNING: Enabling this increases log volume and may capture
                            sensitive data. Ensure compliance with data handling policies.

S3_LOG_PREFIX:              [OPTIONAL: bedrock-invocation-logs/]
                            S3 key prefix for invocation log delivery.
                            Logs are stored as: s3://{S3_LOG_BUCKET}/{S3_LOG_PREFIX}{YYYY/MM/DD}/

LIFECYCLE_GLACIER_DAYS:     [OPTIONAL: 90]
                            Days after which S3 logs transition to Glacier Instant Retrieval.

LIFECYCLE_DEEP_ARCHIVE_DAYS: [OPTIONAL: 365]
                            Days after which S3 logs transition to Glacier Deep Archive.

LIFECYCLE_EXPIRATION_DAYS:  [OPTIONAL: 730]
                            Days after which S3 logs are permanently deleted.
```

---

## Task

Generate complete Bedrock invocation logging and analysis infrastructure:

```
{PROJECT_NAME}-bedrock-logging/
├── logging/
│   ├── enable_invocation_logging.py      # Enable Bedrock invocation logging (CW Logs + S3)
│   ├── create_log_resources.py           # Create S3 bucket, CW log group, IAM policies
│   └── config.py                         # Central configuration
├── athena/
│   ├── create_database.py                # Create Athena database and workgroup
│   ├── create_table.py                   # Athena CREATE TABLE DDL for Bedrock log schema
│   └── named_queries.py                  # Named queries: by model, latency, tokens, errors
├── cost_tracking/
│   ├── pricing_table.py                  # Bedrock pricing reference data
│   └── cost_per_invocation_query.py      # Athena query joining logs with pricing
├── insights/
│   ├── logs_insights_queries.py          # CloudWatch Logs Insights query library
│   └── deploy_metric_filters.py         # Metric filters for error rate, latency, tokens
├── alerting/
│   ├── create_alarms.py                  # Error rate alarm + SNS notification
│   └── failover_lambda.py               # Optional model failover Lambda function
├── lifecycle/
│   ├── s3_lifecycle_rules.py             # S3 lifecycle rules for log archival
│   └── log_retention.py                  # CloudWatch Logs retention configuration
├── run_setup.py                          # CLI orchestrator
└── requirements.txt
```

**config.py**: Central configuration dataclass with all parameters. Load from environment variables or CLI args. Validate required fields. Construct resource names using `{PROJECT_NAME}-{component}-{ENV}` convention. Parse LOG_DESTINATIONS into a list. Validate that S3_LOG_BUCKET is provided when "s3" is in LOG_DESTINATIONS. Validate that CLOUDWATCH_LOG_GROUP is provided when "cloudwatch_logs" is in LOG_DESTINATIONS. Validate that FAILOVER_MODEL_ID is provided when FAILOVER_ENABLED is true.

**enable_invocation_logging.py**: Enable Bedrock model invocation logging:
- Call `bedrock.put_model_invocation_logging_configuration()` with:
  - `loggingConfig.cloudWatchConfig.logGroupName`: CLOUDWATCH_LOG_GROUP (if cloudwatch_logs destination enabled)
  - `loggingConfig.cloudWatchConfig.roleArn`: IAM role ARN that grants Bedrock permission to write to CloudWatch Logs
  - `loggingConfig.cloudWatchConfig.largeDataDelivery.s3Config`: optional S3 config for large payloads
  - `loggingConfig.s3Config.bucketName`: S3_LOG_BUCKET (if s3 destination enabled)
  - `loggingConfig.s3Config.keyPrefix`: S3_LOG_PREFIX
  - `loggingConfig.textDataDeliveryEnabled`: true (always log text data)
  - `loggingConfig.imageDataDeliveryEnabled`: LOG_MODEL_INPUT_OUTPUT
  - `loggingConfig.embeddingDataDeliveryEnabled`: LOG_MODEL_INPUT_OUTPUT
- Verify configuration with `bedrock.get_model_invocation_logging_configuration()`
- Print summary of enabled destinations

**create_log_resources.py**: Create prerequisite resources for logging:
- If "s3" in LOG_DESTINATIONS:
  - Create S3 bucket `{S3_LOG_BUCKET}` with:
    - Server-side encryption (SSE-S3 or SSE-KMS)
    - Bucket policy allowing `bedrock.amazonaws.com` to write logs via `s3:PutObject`
    - Block public access enabled
    - Versioning enabled
    - Tags: `Project={PROJECT_NAME}`, `Environment={ENV}`, `Purpose=bedrock-logs`
- If "cloudwatch_logs" in LOG_DESTINATIONS:
  - Create CloudWatch Logs log group `{CLOUDWATCH_LOG_GROUP}` with:
    - Retention period: LOG_RETENTION_DAYS
    - KMS encryption (optional)
    - Tags: `Project={PROJECT_NAME}`, `Environment={ENV}`
  - Create resource policy allowing `bedrock.amazonaws.com` to write to the log group
- Create IAM role `{PROJECT_NAME}-bedrock-logging-role-{ENV}` with:
  - Trust policy for `bedrock.amazonaws.com`
  - Permissions: `logs:CreateLogStream`, `logs:PutLogEvents` on the log group
  - Permissions: `s3:PutObject` on the log bucket
- Write role ARN and resource identifiers to SSM Parameter Store

**create_database.py**: Create Athena database and workgroup:
- Create Athena database using `athena.start_query_execution()` with DDL: `CREATE DATABASE IF NOT EXISTS {ATHENA_DATABASE}`
- Create Athena workgroup `{PROJECT_NAME}-bedrock-analytics-{ENV}` with:
  - Result configuration pointing to ATHENA_RESULTS_BUCKET
  - Enforce workgroup configuration enabled
  - Bytes scanned cutoff per query (cost control)
  - Tags: `Project={PROJECT_NAME}`, `Environment={ENV}`

**create_table.py**: Create Athena external table for Bedrock invocation logs:
- Execute CREATE EXTERNAL TABLE DDL matching the Bedrock invocation log JSON schema:
  - `schemaType` (STRING), `schemaVersion` (STRING)
  - `timestamp` (STRING), `accountId` (STRING), `region` (STRING)
  - `requestId` (STRING), `operation` (STRING)
  - `modelId` (STRING)
  - `input.inputContentType` (STRING), `input.inputTokenCount` (INT)
  - `input.inputBodyJson` (STRING) — present only if LOG_MODEL_INPUT_OUTPUT is true
  - `output.outputContentType` (STRING), `output.outputTokenCount` (INT)
  - `output.outputBodyJson` (STRING) — present only if LOG_MODEL_INPUT_OUTPUT is true
  - `status` (STRING) — success or error
  - `errorCode` (STRING) — null on success
  - `errorMessage` (STRING) — null on success
  - `requestLatency` (INT) — milliseconds
- ROW FORMAT SERDE `org.openx.data.jsonserde.JsonSerDe`
- LOCATION `s3://{S3_LOG_BUCKET}/{S3_LOG_PREFIX}`
- TBLPROPERTIES for partition projection on date (type=date, range=NOW-1YEARS/NOW, format=yyyy/MM/dd)

**named_queries.py**: Create Athena named queries for common analytics:
- `invocations_by_model`: Count invocations grouped by `modelId` over a date range, ordered by count descending
- `avg_latency_by_model`: Average, p50, p90, p99 `requestLatency` grouped by `modelId` using `approx_percentile()`
- `token_distribution`: Distribution of `inputTokenCount` and `outputTokenCount` by model with min, max, avg, and histogram buckets
- `error_rate_by_model`: Error rate percentage per model: `COUNT(CASE WHEN status='error') / COUNT(*) * 100` grouped by `modelId` and date
- `hourly_invocation_volume`: Invocation count per hour per model for capacity planning
- `top_errors`: Most frequent `errorCode` values with count and example `errorMessage`
- Each named query is created using `athena.create_named_query()` with the workgroup

**pricing_table.py**: Bedrock pricing reference data:
- Define a Python dictionary mapping `modelId` to pricing per 1K input tokens and per 1K output tokens
- Include pricing for common models: Claude Sonnet 4.7, Claude Haiku 4.5, Titan Text Express, Titan Text Lite, Llama models, Mistral models
- Create an Athena table `bedrock_pricing` with columns: `model_id`, `input_price_per_1k_tokens`, `output_price_per_1k_tokens`, `provider`
- Populate the pricing table using INSERT INTO from a VALUES clause
- Include a utility function to refresh pricing data

**cost_per_invocation_query.py**: Athena query joining invocation logs with pricing:
- Named query `cost_per_invocation`: JOIN invocation logs with pricing table on `modelId` to compute:
  - `input_cost = (inputTokenCount / 1000.0) * input_price_per_1k_tokens`
  - `output_cost = (outputTokenCount / 1000.0) * output_price_per_1k_tokens`
  - `total_cost = input_cost + output_cost`
- Named query `daily_cost_by_model`: Aggregate total cost per model per day
- Named query `cost_trend`: 7-day rolling average cost per model for trend analysis
- Named query `top_cost_requests`: Top 100 most expensive individual invocations with model, tokens, and cost

**logs_insights_queries.py**: CloudWatch Logs Insights query library:
- `error_rate_query`: Calculate error rate over the last hour:
  ```
  fields @timestamp, modelId, status
  | stats count(*) as total,
          sum(status="error") as errors,
          (sum(status="error") / count(*)) * 100 as error_rate
    by bin(5m)
  | sort @timestamp desc
  ```
- `latency_percentiles_query`: P50, P90, P99 latency by model:
  ```
  fields @timestamp, modelId, requestLatency
  | stats pct(requestLatency, 50) as p50,
          pct(requestLatency, 90) as p90,
          pct(requestLatency, 99) as p99,
          avg(requestLatency) as avg_latency
    by modelId
  ```
- `token_usage_query`: Token consumption by model over time:
  ```
  fields @timestamp, modelId, input.inputTokenCount, output.outputTokenCount
  | stats sum(input.inputTokenCount) as total_input_tokens,
          sum(output.outputTokenCount) as total_output_tokens
    by modelId, bin(1h)
  | sort @timestamp desc
  ```
- `error_details_query`: Recent errors with details:
  ```
  fields @timestamp, modelId, errorCode, errorMessage, requestId
  | filter status = "error"
  | sort @timestamp desc
  | limit 50
  ```
- `invocations_per_model_query`: Invocation volume by model over time:
  ```
  fields @timestamp, modelId
  | stats count(*) as invocations by modelId, bin(1h)
  | sort invocations desc
  ```
- `slow_requests_query`: Requests exceeding a latency threshold:
  ```
  fields @timestamp, modelId, requestLatency, requestId
  | filter requestLatency > 5000
  | sort requestLatency desc
  | limit 100
  ```
- Store each query as a Python string constant and provide a function to execute any query via `logs.start_query()` and `logs.get_query_results()`

**deploy_metric_filters.py**: Create CloudWatch metric filters on the invocation log group:
- `ErrorCount` metric filter: pattern `{ $.status = "error" }` → metric `{PROJECT_NAME}/Bedrock/ErrorCount` with value 1
- `InvocationCount` metric filter: pattern `{ $.modelId = * }` → metric `{PROJECT_NAME}/Bedrock/InvocationCount` with value 1
- `Latency` metric filter: pattern `{ $.requestLatency = * }` → metric `{PROJECT_NAME}/Bedrock/Latency` with metric value `$.requestLatency`
- `InputTokens` metric filter: pattern `{ $.input.inputTokenCount = * }` → metric `{PROJECT_NAME}/Bedrock/InputTokens` with metric value `$.input.inputTokenCount`
- `OutputTokens` metric filter: pattern `{ $.output.outputTokenCount = * }` → metric `{PROJECT_NAME}/Bedrock/OutputTokens` with metric value `$.output.outputTokenCount`
- Each metric filter uses `logs.put_metric_filter()` with the log group name and metric namespace

**create_alarms.py**: Create CloudWatch alarms for Bedrock invocation monitoring:
- Error rate alarm using metric math:
  - `m1`: `{PROJECT_NAME}/Bedrock/ErrorCount` SUM over 5 minutes
  - `m2`: `{PROJECT_NAME}/Bedrock/InvocationCount` SUM over 5 minutes
  - `e1`: expression `(m1 / m2) * 100` — error rate percentage
  - Threshold: ALERT_ERROR_RATE_THRESHOLD
  - Comparison: `GreaterThanThreshold`
  - Evaluation periods: 3 (15 minutes of sustained errors)
  - Alarm actions: SNS topic ARN `{PROJECT_NAME}-bedrock-alerts-{ENV}`
  - If FAILOVER_ENABLED: add Lambda function ARN as alarm action
- Create SNS topic `{PROJECT_NAME}-bedrock-alerts-{ENV}` for alarm notifications
- High latency alarm: P99 latency exceeding 10 seconds
- Token budget alarm: daily token consumption exceeding a configurable threshold

**failover_lambda.py**: Optional model failover Lambda function (generated only if FAILOVER_ENABLED is true):
- Lambda function `{PROJECT_NAME}-bedrock-failover-{ENV}` triggered by CloudWatch alarm action
- On ALARM state:
  - Read current active model from DynamoDB table `{PROJECT_NAME}-model-config-{ENV}`
  - Update active model to FAILOVER_MODEL_ID
  - Publish SNS notification: "Failover activated: switched from {PRIMARY_MODEL_ID} to {FAILOVER_MODEL_ID}"
  - Log failover event with timestamp, reason, and alarm details
- On OK state:
  - Restore active model to PRIMARY_MODEL_ID
  - Publish SNS notification: "Failover restored: switched back to {PRIMARY_MODEL_ID}"
- Create DynamoDB table `{PROJECT_NAME}-model-config-{ENV}` with:
  - Partition key: `config_key` (String)
  - Item: `{ config_key: "active_model", model_id: PRIMARY_MODEL_ID, updated_at: timestamp }`
- Create Lambda execution role with permissions for DynamoDB read/write, SNS publish, and CloudWatch Logs
- Application code reads active model from DynamoDB before each Bedrock invocation

**s3_lifecycle_rules.py**: Configure S3 lifecycle rules for log archival:
- Rule 1: Transition to Intelligent-Tiering after 30 days
- Rule 2: Transition to Glacier Instant Retrieval after LIFECYCLE_GLACIER_DAYS days
- Rule 3: Transition to Glacier Deep Archive after LIFECYCLE_DEEP_ARCHIVE_DAYS days
- Rule 4: Expire (delete) objects after LIFECYCLE_EXPIRATION_DAYS days
- Rule 5: Abort incomplete multipart uploads after 7 days
- Apply rules to the S3_LOG_PREFIX within the S3_LOG_BUCKET
- Use `s3.put_bucket_lifecycle_configuration()` with the lifecycle rules
- Tag lifecycle rules with `Project={PROJECT_NAME}`, `Environment={ENV}`

**log_retention.py**: Configure CloudWatch Logs retention:
- Set retention policy on CLOUDWATCH_LOG_GROUP using `logs.put_retention_policy()` with LOG_RETENTION_DAYS
- Verify retention is applied using `logs.describe_log_groups()`

**run_setup.py**: CLI orchestrator that runs setup steps in order:
1. Load and validate configuration
2. Create log resources (S3 bucket, CloudWatch log group, IAM role)
3. Enable Bedrock invocation logging
4. Set CloudWatch Logs retention
5. Deploy metric filters (if cloudwatch_logs destination)
6. Create CloudWatch alarms and SNS topic
7. Deploy failover Lambda (if FAILOVER_ENABLED)
8. Configure S3 lifecycle rules (if s3 destination)
9. Create Athena database and table (if s3 destination)
10. Create Athena named queries and cost tracking queries
11. Write resource identifiers to SSM Parameter Store
12. Print summary with log group, bucket, Athena database, alarm ARNs, and query names

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints


**Logging Configuration:** Bedrock invocation logging is configured at the account level per region using `bedrock.put_model_invocation_logging_configuration()`. There is one logging configuration per account per region — it applies to all Bedrock model invocations in that account/region. You cannot configure per-model logging; all models are logged when logging is enabled. Both CloudWatch Logs and S3 destinations can be enabled simultaneously. CloudWatch Logs provides real-time query capability via Logs Insights; S3 provides cost-effective long-term storage for Athena analytics.

**Log Schema:** Bedrock invocation logs are JSON documents with a well-defined schema. Key fields include: `schemaType`, `schemaVersion`, `timestamp`, `accountId`, `region`, `requestId`, `operation` (InvokeModel, Converse, etc.), `modelId`, `input` (inputContentType, inputTokenCount, optionally inputBodyJson), `output` (outputContentType, outputTokenCount, optionally outputBodyJson), `status` (success/error), `errorCode`, `errorMessage`, and `requestLatency` (milliseconds). The schema may evolve — use `schemaVersion` for forward compatibility.

**Input/Output Logging:** By default, Bedrock logs metadata only (model ID, token counts, latency, status). Enabling `textDataDeliveryEnabled`, `imageDataDeliveryEnabled`, or `embeddingDataDeliveryEnabled` includes full request/response payloads. WARNING: Full payload logging significantly increases log volume and may capture sensitive or PII data. Enable only when needed for debugging, and ensure compliance with data handling policies. Consider using Bedrock Guardrails (mlops/12) to redact sensitive content before logging.

**IAM for Logging:** Bedrock requires an IAM role with permissions to write to the configured log destinations. The role must trust `bedrock.amazonaws.com` as a service principal. For CloudWatch Logs, the role needs `logs:CreateLogStream` and `logs:PutLogEvents`. For S3, the role needs `s3:PutObject` on the log bucket. Additionally, a CloudWatch Logs resource policy must allow the Bedrock service to create log streams in the target log group.

**Athena Partitioning:** Use Athena partition projection for time-based partitioning of S3 logs. Partition projection eliminates the need for `MSCK REPAIR TABLE` by dynamically computing partitions based on the S3 key prefix pattern. Configure projection on a `date` column with type `date`, range `NOW-1YEARS,NOW`, format `yyyy/MM/dd`, and interval `1` day. This enables efficient time-range queries without partition management overhead.

**Cost Tracking:** Bedrock pricing varies by model and is charged per 1K input/output tokens. To compute cost per invocation, join the invocation log table with a pricing reference table on `modelId`. Maintain the pricing table manually or via a refresh script that queries the AWS Pricing API. Cost tracking enables per-model, per-day, and per-request cost attribution for chargeback and optimization.

**Error Rate Monitoring:** Use CloudWatch metric filters to extract error counts and invocation counts from the log group. Compute error rate using metric math: `(ErrorCount / InvocationCount) * 100`. Set an alarm threshold at ALERT_ERROR_RATE_THRESHOLD percent with 3 evaluation periods (15 minutes) to avoid false positives from transient errors. Alarm actions notify via SNS and optionally trigger the failover Lambda.

**Model Failover:** The optional failover pattern uses a DynamoDB table to store the active model ID. Application code reads the active model before each Bedrock invocation. When the error rate alarm fires, the failover Lambda updates the DynamoDB entry to the backup model. When the alarm returns to OK, the Lambda restores the primary model. This pattern requires application-level integration — the application must read from DynamoDB rather than hardcoding the model ID.

**S3 Lifecycle:** Configure lifecycle rules to manage log storage costs. Transition logs to Intelligent-Tiering after 30 days (automatic cost optimization), to Glacier Instant Retrieval after LIFECYCLE_GLACIER_DAYS (millisecond retrieval, lower cost), to Glacier Deep Archive after LIFECYCLE_DEEP_ARCHIVE_DAYS (lowest cost, 12-hour retrieval), and expire after LIFECYCLE_EXPIRATION_DAYS. Adjust thresholds based on compliance retention requirements.

**CloudWatch Logs Retention:** Set retention on the invocation log group to LOG_RETENTION_DAYS. CloudWatch Logs charges $0.50/GB ingested and $0.03/GB/month stored. For high-volume Bedrock usage, S3 is more cost-effective for long-term storage. Use CloudWatch Logs for real-time analysis (last 7-30 days) and S3+Athena for historical analytics.

**Naming:** All resources follow `{PROJECT_NAME}-{component}-{ENV}` convention:
- S3 bucket: `{PROJECT_NAME}-bedrock-logs-{ENV}`
- CloudWatch log group: `/aws/bedrock/{PROJECT_NAME}/{ENV}/invocations`
- Athena database: `{PROJECT_NAME}_bedrock_analytics_{ENV}`
- Athena table: `bedrock_invocation_logs`
- CloudWatch alarm: `{PROJECT_NAME}-bedrock-error-rate-{ENV}`
- SNS topic: `{PROJECT_NAME}-bedrock-alerts-{ENV}`
- Lambda function: `{PROJECT_NAME}-bedrock-failover-{ENV}`
- DynamoDB table: `{PROJECT_NAME}-model-config-{ENV}`
- IAM role: `{PROJECT_NAME}-bedrock-logging-role-{ENV}`
- SSM parameters: `/mlops/{PROJECT_NAME}/{ENV}/bedrock-log-group`, `/mlops/{PROJECT_NAME}/{ENV}/bedrock-log-bucket`

---

## Code Scaffolding Hints

**Enable Bedrock invocation logging:**
```python
import boto3
import json

bedrock = boto3.client("bedrock", region_name=AWS_REGION)

def enable_invocation_logging(log_group_name, s3_bucket, s3_prefix, role_arn,
                               enable_cloudwatch=True, enable_s3=True,
                               log_model_input_output=False):
    """Enable Bedrock model invocation logging to CloudWatch Logs and/or S3."""
    logging_config = {
        "textDataDeliveryEnabled": True,
        "imageDataDeliveryEnabled": log_model_input_output,
        "embeddingDataDeliveryEnabled": log_model_input_output,
    }

    if enable_cloudwatch:
        logging_config["cloudWatchConfig"] = {
            "logGroupName": log_group_name,
            "roleArn": role_arn,
            "largeDataDelivery": {
                "s3Config": {
                    "bucketName": s3_bucket,
                    "keyPrefix": f"{s3_prefix}large-data/",
                }
            },
        }

    if enable_s3:
        logging_config["s3Config"] = {
            "bucketName": s3_bucket,
            "keyPrefix": s3_prefix,
        }

    bedrock.put_model_invocation_logging_configuration(
        loggingConfig=logging_config
    )

    # Verify configuration
    response = bedrock.get_model_invocation_logging_configuration()
    config = response.get("loggingConfig", {})
    print(f"Bedrock invocation logging enabled:")
    print(f"  CloudWatch Logs: {bool(config.get('cloudWatchConfig'))}")
    print(f"  S3: {bool(config.get('s3Config'))}")
    print(f"  Text data delivery: {config.get('textDataDeliveryEnabled')}")
    print(f"  Image data delivery: {config.get('imageDataDeliveryEnabled')}")
    return config
```


**Create S3 bucket for Bedrock logs with bucket policy:**
```python
s3 = boto3.client("s3", region_name=AWS_REGION)

def create_log_bucket(bucket_name):
    """Create S3 bucket for Bedrock invocation logs with service access policy."""
    # Create bucket
    if AWS_REGION == "us-east-1":
        s3.create_bucket(Bucket=bucket_name)
    else:
        s3.create_bucket(
            Bucket=bucket_name,
            CreateBucketConfiguration={"LocationConstraint": AWS_REGION},
        )

    # Block public access
    s3.put_public_access_block(
        Bucket=bucket_name,
        PublicAccessBlockConfiguration={
            "BlockPublicAcls": True,
            "IgnorePublicAcls": True,
            "BlockPublicPolicy": True,
            "RestrictPublicBuckets": True,
        },
    )

    # Enable versioning
    s3.put_bucket_versioning(
        Bucket=bucket_name,
        VersioningConfiguration={"Status": "Enabled"},
    )

    # Enable default encryption
    s3.put_bucket_encryption(
        Bucket=bucket_name,
        ServerSideEncryptionConfiguration={
            "Rules": [
                {"ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "AES256"}}
            ]
        },
    )

    # Bucket policy allowing Bedrock to write logs
    bucket_policy = {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "AllowBedrockLogging",
                "Effect": "Allow",
                "Principal": {"Service": "bedrock.amazonaws.com"},
                "Action": "s3:PutObject",
                "Resource": f"arn:aws:s3:::{bucket_name}/*",
                "Condition": {
                    "StringEquals": {
                        "aws:SourceAccount": AWS_ACCOUNT_ID
                    }
                },
            }
        ],
    }
    s3.put_bucket_policy(Bucket=bucket_name, Policy=json.dumps(bucket_policy))

    # Tag bucket
    s3.put_bucket_tagging(
        Bucket=bucket_name,
        Tagging={
            "TagSet": [
                {"Key": "Project", "Value": PROJECT_NAME},
                {"Key": "Environment", "Value": ENV},
                {"Key": "Purpose", "Value": "bedrock-invocation-logs"},
            ]
        },
    )
    print(f"S3 log bucket created: {bucket_name}")
    return bucket_name
```

**Create CloudWatch Logs log group with resource policy:**
```python
logs = boto3.client("logs", region_name=AWS_REGION)

def create_log_group(log_group_name, retention_days):
    """Create CloudWatch Logs log group for Bedrock invocation logs."""
    try:
        logs.create_log_group(
            logGroupName=log_group_name,
            tags={
                "Project": PROJECT_NAME,
                "Environment": ENV,
                "Purpose": "bedrock-invocation-logs",
            },
        )
    except logs.exceptions.ResourceAlreadyExistsException:
        print(f"Log group already exists: {log_group_name}")

    # Set retention
    logs.put_retention_policy(
        logGroupName=log_group_name,
        retentionInDays=retention_days,
    )

    # Resource policy allowing Bedrock to write logs
    resource_policy = {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "AllowBedrockLogging",
                "Effect": "Allow",
                "Principal": {"Service": "bedrock.amazonaws.com"},
                "Action": ["logs:CreateLogStream", "logs:PutLogEvents"],
                "Resource": f"arn:aws:logs:{AWS_REGION}:{AWS_ACCOUNT_ID}:log-group:{log_group_name}:*",
                "Condition": {
                    "StringEquals": {
                        "aws:SourceAccount": AWS_ACCOUNT_ID
                    }
                },
            }
        ],
    }
    logs.put_resource_policy(
        policyName=f"{PROJECT_NAME}-bedrock-logging-{ENV}",
        policyDocument=json.dumps(resource_policy),
    )
    print(f"Log group created: {log_group_name} (retention: {retention_days} days)")
    return log_group_name
```

**Athena CREATE TABLE DDL for Bedrock invocation log schema:**
```python
athena = boto3.client("athena", region_name=AWS_REGION)

BEDROCK_LOG_TABLE_DDL = f"""
CREATE EXTERNAL TABLE IF NOT EXISTS {ATHENA_DATABASE}.bedrock_invocation_logs (
    schemaType        STRING,
    schemaVersion     STRING,
    timestamp         STRING,
    accountId         STRING,
    region            STRING,
    requestId         STRING,
    operation         STRING,
    modelId           STRING,
    input             STRUCT<
        inputContentType:  STRING,
        inputTokenCount:   INT,
        inputBodyJson:     STRING
    >,
    output            STRUCT<
        outputContentType: STRING,
        outputTokenCount:  INT,
        outputBodyJson:    STRING
    >,
    status            STRING,
    errorCode         STRING,
    errorMessage      STRING,
    requestLatency    INT
)
PARTITIONED BY (date_partition STRING)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
WITH SERDEPROPERTIES ('ignore.malformed.json' = 'true')
LOCATION 's3://{S3_LOG_BUCKET}/{S3_LOG_PREFIX}'
TBLPROPERTIES (
    'projection.enabled'                = 'true',
    'projection.date_partition.type'    = 'date',
    'projection.date_partition.range'   = 'NOW-1YEARS,NOW',
    'projection.date_partition.format'  = 'yyyy/MM/dd',
    'projection.date_partition.interval'= '1',
    'projection.date_partition.interval.unit' = 'DAYS',
    'storage.location.template'         = 's3://{S3_LOG_BUCKET}/{S3_LOG_PREFIX}${{date_partition}}'
)
"""

def create_invocation_log_table(database, workgroup):
    """Create Athena external table for Bedrock invocation logs."""
    response = athena.start_query_execution(
        QueryString=BEDROCK_LOG_TABLE_DDL,
        QueryExecutionContext={"Database": database},
        WorkGroup=workgroup,
    )
    query_id = response["QueryExecutionId"]
    print(f"Creating Athena table: query execution ID = {query_id}")
    return query_id
```


**Athena named queries for invocation analytics:**
```python
def create_named_queries(database, workgroup):
    """Create Athena named queries for Bedrock invocation log analysis."""
    queries = {
        "invocations_by_model": {
            "description": "Count invocations grouped by model over a date range",
            "query": f"""
                SELECT modelId,
                       COUNT(*) AS invocation_count,
                       COUNT(CASE WHEN status = 'error' THEN 1 END) AS error_count,
                       MIN(timestamp) AS first_invocation,
                       MAX(timestamp) AS last_invocation
                FROM {database}.bedrock_invocation_logs
                WHERE date_partition >= date_format(current_date - interval '7' day, '%Y/%m/%d')
                GROUP BY modelId
                ORDER BY invocation_count DESC
            """,
        },
        "avg_latency_by_model": {
            "description": "Average and percentile latency by model",
            "query": f"""
                SELECT modelId,
                       COUNT(*) AS invocations,
                       AVG(requestLatency) AS avg_latency_ms,
                       approx_percentile(requestLatency, 0.50) AS p50_ms,
                       approx_percentile(requestLatency, 0.90) AS p90_ms,
                       approx_percentile(requestLatency, 0.99) AS p99_ms,
                       MIN(requestLatency) AS min_ms,
                       MAX(requestLatency) AS max_ms
                FROM {database}.bedrock_invocation_logs
                WHERE date_partition >= date_format(current_date - interval '7' day, '%Y/%m/%d')
                  AND status = 'success'
                GROUP BY modelId
                ORDER BY avg_latency_ms DESC
            """,
        },
        "token_distribution": {
            "description": "Token usage distribution by model",
            "query": f"""
                SELECT modelId,
                       COUNT(*) AS invocations,
                       AVG(input.inputTokenCount) AS avg_input_tokens,
                       AVG(output.outputTokenCount) AS avg_output_tokens,
                       SUM(input.inputTokenCount) AS total_input_tokens,
                       SUM(output.outputTokenCount) AS total_output_tokens,
                       MIN(input.inputTokenCount) AS min_input_tokens,
                       MAX(input.inputTokenCount) AS max_input_tokens,
                       approx_percentile(input.inputTokenCount, 0.95) AS p95_input_tokens
                FROM {database}.bedrock_invocation_logs
                WHERE date_partition >= date_format(current_date - interval '7' day, '%Y/%m/%d')
                  AND status = 'success'
                GROUP BY modelId
                ORDER BY total_input_tokens DESC
            """,
        },
        "error_rate_by_model": {
            "description": "Error rate percentage per model per day",
            "query": f"""
                SELECT date_partition,
                       modelId,
                       COUNT(*) AS total,
                       COUNT(CASE WHEN status = 'error' THEN 1 END) AS errors,
                       ROUND(CAST(COUNT(CASE WHEN status = 'error' THEN 1 END) AS DOUBLE)
                             / COUNT(*) * 100, 2) AS error_rate_pct
                FROM {database}.bedrock_invocation_logs
                WHERE date_partition >= date_format(current_date - interval '30' day, '%Y/%m/%d')
                GROUP BY date_partition, modelId
                HAVING COUNT(CASE WHEN status = 'error' THEN 1 END) > 0
                ORDER BY date_partition DESC, error_rate_pct DESC
            """,
        },
        "top_errors": {
            "description": "Most frequent error codes with example messages",
            "query": f"""
                SELECT errorCode,
                       COUNT(*) AS occurrences,
                       modelId,
                       arbitrary(errorMessage) AS example_message,
                       MIN(timestamp) AS first_seen,
                       MAX(timestamp) AS last_seen
                FROM {database}.bedrock_invocation_logs
                WHERE date_partition >= date_format(current_date - interval '7' day, '%Y/%m/%d')
                  AND status = 'error'
                GROUP BY errorCode, modelId
                ORDER BY occurrences DESC
                LIMIT 20
            """,
        },
    }

    created_ids = {}
    for name, q in queries.items():
        response = athena.create_named_query(
            Name=f"{PROJECT_NAME}-{name}-{ENV}",
            Description=q["description"],
            Database=database,
            QueryString=q["query"].strip(),
            WorkGroup=workgroup,
        )
        created_ids[name] = response["NamedQueryId"]
        print(f"Named query created: {name} -> {response['NamedQueryId']}")

    return created_ids
```

**Cost-per-invocation Athena query joining logs with pricing:**
```python
COST_PER_INVOCATION_QUERY = f"""
SELECT l.date_partition,
       l.modelId,
       COUNT(*) AS invocations,
       SUM(l.input.inputTokenCount) AS total_input_tokens,
       SUM(l.output.outputTokenCount) AS total_output_tokens,
       ROUND(SUM(CAST(l.input.inputTokenCount AS DOUBLE) / 1000.0
                 * p.input_price_per_1k_tokens), 4) AS total_input_cost,
       ROUND(SUM(CAST(l.output.outputTokenCount AS DOUBLE) / 1000.0
                 * p.output_price_per_1k_tokens), 4) AS total_output_cost,
       ROUND(SUM(CAST(l.input.inputTokenCount AS DOUBLE) / 1000.0
                 * p.input_price_per_1k_tokens
               + CAST(l.output.outputTokenCount AS DOUBLE) / 1000.0
                 * p.output_price_per_1k_tokens), 4) AS total_cost,
       ROUND(AVG(CAST(l.input.inputTokenCount AS DOUBLE) / 1000.0
                 * p.input_price_per_1k_tokens
               + CAST(l.output.outputTokenCount AS DOUBLE) / 1000.0
                 * p.output_price_per_1k_tokens), 6) AS avg_cost_per_invocation
FROM {ATHENA_DATABASE}.bedrock_invocation_logs l
JOIN {ATHENA_DATABASE}.bedrock_pricing p
  ON l.modelId = p.model_id
WHERE l.date_partition >= date_format(current_date - interval '7' day, '%Y/%m/%d')
  AND l.status = 'success'
GROUP BY l.date_partition, l.modelId
ORDER BY total_cost DESC
"""

# Pricing reference table DDL
PRICING_TABLE_DDL = f"""
CREATE TABLE IF NOT EXISTS {ATHENA_DATABASE}.bedrock_pricing (
    model_id                    VARCHAR,
    provider                    VARCHAR,
    input_price_per_1k_tokens   DOUBLE,
    output_price_per_1k_tokens  DOUBLE
)
"""

# Populate pricing data (update values as pricing changes)
PRICING_INSERT = f"""
INSERT INTO {ATHENA_DATABASE}.bedrock_pricing VALUES
    ('us.anthropic.claude-sonnet-4-7-20260109-v1:0', 'Anthropic', 0.003, 0.015),
    ('us.anthropic.claude-haiku-4-5-20251001-v1:0',  'Anthropic', 0.001, 0.005),
    ('us.anthropic.claude-sonnet-4-7-20260109-v1:0',   'Anthropic', 0.003, 0.015),
    ('us.anthropic.claude-haiku-4-5-20251001-v1:0',    'Anthropic', 0.00025, 0.00125),
    ('amazon.titan-text-express-v1',              'Amazon',    0.0002, 0.0006),
    ('amazon.titan-text-lite-v1',                 'Amazon',    0.00015, 0.0002),
    ('meta.llama3-1-70b-instruct-v1:0',           'Meta',      0.00099, 0.00099),
    ('meta.llama3-1-8b-instruct-v1:0',            'Meta',      0.00022, 0.00022),
    ('mistral.mistral-large-2407-v1:0',           'Mistral',   0.002, 0.006)
"""
```

**CloudWatch Logs Insights query execution:**
```python
import time

logs = boto3.client("logs", region_name=AWS_REGION)

# Query library
INSIGHTS_QUERIES = {
    "error_rate": """
        fields @timestamp, modelId, status
        | stats count(*) as total,
                sum(status="error") as errors,
                (sum(status="error") / count(*)) * 100 as error_rate_pct
          by bin(5m)
        | sort @timestamp desc
    """,
    "latency_percentiles": """
        fields @timestamp, modelId, requestLatency
        | stats pct(requestLatency, 50) as p50_ms,
                pct(requestLatency, 90) as p90_ms,
                pct(requestLatency, 99) as p99_ms,
                avg(requestLatency) as avg_ms
          by modelId
    """,
    "token_usage": """
        fields @timestamp, modelId, input.inputTokenCount, output.outputTokenCount
        | stats sum(input.inputTokenCount) as total_input_tokens,
                sum(output.outputTokenCount) as total_output_tokens,
                avg(input.inputTokenCount) as avg_input,
                avg(output.outputTokenCount) as avg_output
          by modelId, bin(1h)
        | sort @timestamp desc
    """,
    "recent_errors": """
        fields @timestamp, modelId, errorCode, errorMessage, requestId
        | filter status = "error"
        | sort @timestamp desc
        | limit 50
    """,
    "slow_requests": """
        fields @timestamp, modelId, requestLatency, requestId,
               input.inputTokenCount, output.outputTokenCount
        | filter requestLatency > 5000
        | sort requestLatency desc
        | limit 100
    """,
    "invocations_per_model": """
        fields @timestamp, modelId
        | stats count(*) as invocations by modelId, bin(1h)
        | sort invocations desc
    """,
}

def run_insights_query(query_name, log_group, start_time, end_time):
    """Execute a CloudWatch Logs Insights query and return results."""
    query_string = INSIGHTS_QUERIES[query_name]
    response = logs.start_query(
        logGroupName=log_group,
        startTime=int(start_time.timestamp()),
        endTime=int(end_time.timestamp()),
        queryString=query_string.strip(),
    )
    query_id = response["queryId"]

    # Poll for results
    while True:
        result = logs.get_query_results(queryId=query_id)
        if result["status"] in ("Complete", "Failed", "Cancelled"):
            break
        time.sleep(1)

    if result["status"] == "Complete":
        print(f"Query '{query_name}' returned {len(result['results'])} rows")
        return result["results"]
    else:
        print(f"Query '{query_name}' {result['status']}")
        return []
```

**Deploy CloudWatch metric filters:**
```python
def deploy_metric_filters(log_group_name, namespace):
    """Create metric filters on the Bedrock invocation log group."""
    filters = [
        {
            "name": f"{PROJECT_NAME}-bedrock-errors-{ENV}",
            "pattern": '{ $.status = "error" }',
            "metric_name": "ErrorCount",
            "metric_value": "1",
            "default_value": 0,
        },
        {
            "name": f"{PROJECT_NAME}-bedrock-invocations-{ENV}",
            "pattern": '{ $.modelId = * }',
            "metric_name": "InvocationCount",
            "metric_value": "1",
            "default_value": 0,
        },
        {
            "name": f"{PROJECT_NAME}-bedrock-latency-{ENV}",
            "pattern": "{ $.requestLatency = * }",
            "metric_name": "Latency",
            "metric_value": "$.requestLatency",
            "default_value": 0,
        },
        {
            "name": f"{PROJECT_NAME}-bedrock-input-tokens-{ENV}",
            "pattern": "{ $.input.inputTokenCount = * }",
            "metric_name": "InputTokens",
            "metric_value": "$.input.inputTokenCount",
            "default_value": 0,
        },
        {
            "name": f"{PROJECT_NAME}-bedrock-output-tokens-{ENV}",
            "pattern": "{ $.output.outputTokenCount = * }",
            "metric_name": "OutputTokens",
            "metric_value": "$.output.outputTokenCount",
            "default_value": 0,
        },
    ]

    for f in filters:
        logs.put_metric_filter(
            logGroupName=log_group_name,
            filterName=f["name"],
            filterPattern=f["pattern"],
            metricTransformations=[
                {
                    "metricNamespace": namespace,
                    "metricName": f["metric_name"],
                    "metricValue": f["metric_value"],
                    "defaultValue": f["default_value"],
                }
            ],
        )
        print(f"Metric filter created: {f['name']} -> {namespace}/{f['metric_name']}")
```


**Error rate alarm with metric math:**
```python
cloudwatch = boto3.client("cloudwatch", region_name=AWS_REGION)
sns = boto3.client("sns", region_name=AWS_REGION)

def create_error_rate_alarm(namespace, threshold, sns_topic_arn, failover_lambda_arn=None):
    """Create CloudWatch alarm for Bedrock invocation error rate using metric math."""
    alarm_actions = [sns_topic_arn]
    if failover_lambda_arn:
        alarm_actions.append(failover_lambda_arn)

    cloudwatch.put_metric_alarm(
        AlarmName=f"{PROJECT_NAME}-bedrock-error-rate-{ENV}",
        AlarmDescription=(
            f"Bedrock invocation error rate exceeds {threshold}% "
            f"over 15 minutes (3 x 5-min periods)"
        ),
        Metrics=[
            {
                "Id": "errors",
                "MetricStat": {
                    "Metric": {
                        "Namespace": namespace,
                        "MetricName": "ErrorCount",
                    },
                    "Period": 300,
                    "Stat": "Sum",
                },
                "ReturnData": False,
            },
            {
                "Id": "invocations",
                "MetricStat": {
                    "Metric": {
                        "Namespace": namespace,
                        "MetricName": "InvocationCount",
                    },
                    "Period": 300,
                    "Stat": "Sum",
                },
                "ReturnData": False,
            },
            {
                "Id": "error_rate",
                "Expression": "(errors / invocations) * 100",
                "Label": "Error Rate %",
                "ReturnData": True,
            },
        ],
        EvaluationPeriods=3,
        DatapointsToAlarm=3,
        Threshold=threshold,
        ComparisonOperator="GreaterThanThreshold",
        TreatMissingData="notBreaching",
        AlarmActions=alarm_actions,
        OKActions=alarm_actions if failover_lambda_arn else [],
        Tags=[
            {"Key": "Project", "Value": PROJECT_NAME},
            {"Key": "Environment", "Value": ENV},
        ],
    )
    print(f"Error rate alarm created: {PROJECT_NAME}-bedrock-error-rate-{ENV} "
          f"(threshold: {threshold}%)")
```

**Model failover Lambda function:**
```python
import json
import os
import boto3
from datetime import datetime, timezone

dynamodb = boto3.resource("dynamodb")
sns_client = boto3.client("sns")

TABLE_NAME = os.environ["MODEL_CONFIG_TABLE"]
SNS_TOPIC_ARN = os.environ["SNS_TOPIC_ARN"]
PRIMARY_MODEL = os.environ["PRIMARY_MODEL_ID"]
FAILOVER_MODEL = os.environ["FAILOVER_MODEL_ID"]


def handler(event, context):
    """Handle CloudWatch alarm state change for model failover."""
    table = dynamodb.Table(TABLE_NAME)
    alarm_state = event.get("detail", {}).get("state", {}).get("value", "UNKNOWN")
    alarm_name = event.get("detail", {}).get("alarmName", "unknown")
    timestamp = datetime.now(timezone.utc).isoformat()

    if alarm_state == "ALARM":
        # Failover: switch to backup model
        new_model = FAILOVER_MODEL
        action = "FAILOVER_ACTIVATED"
        message = (
            f"Bedrock model failover activated.\n"
            f"Alarm: {alarm_name}\n"
            f"Switched from: {PRIMARY_MODEL}\n"
            f"Switched to: {FAILOVER_MODEL}\n"
            f"Time: {timestamp}"
        )
    elif alarm_state == "OK":
        # Restore: switch back to primary model
        new_model = PRIMARY_MODEL
        action = "FAILOVER_RESTORED"
        message = (
            f"Bedrock model failover restored.\n"
            f"Alarm: {alarm_name}\n"
            f"Restored to: {PRIMARY_MODEL}\n"
            f"Time: {timestamp}"
        )
    else:
        print(f"Ignoring alarm state: {alarm_state}")
        return {"statusCode": 200, "body": "No action taken"}

    # Update DynamoDB with new active model
    table.update_item(
        Key={"config_key": "active_model"},
        UpdateExpression="SET model_id = :m, updated_at = :t, #a = :a, previous_model = :p",
        ExpressionAttributeNames={"#a": "action"},
        ExpressionAttributeValues={
            ":m": new_model,
            ":t": timestamp,
            ":a": action,
            ":p": PRIMARY_MODEL if alarm_state == "ALARM" else FAILOVER_MODEL,
        },
    )

    # Send SNS notification
    sns_client.publish(
        TopicArn=SNS_TOPIC_ARN,
        Subject=f"Bedrock Model {action}",
        Message=message,
    )

    print(f"{action}: active model set to {new_model}")
    return {"statusCode": 200, "body": json.dumps({"action": action, "model": new_model})}
```

**S3 lifecycle rules for log archival:**
```python
def configure_lifecycle_rules(bucket_name, prefix, glacier_days, deep_archive_days, expiration_days):
    """Configure S3 lifecycle rules for Bedrock log archival."""
    lifecycle_config = {
        "Rules": [
            {
                "ID": f"{PROJECT_NAME}-logs-intelligent-tiering",
                "Status": "Enabled",
                "Filter": {"Prefix": prefix},
                "Transitions": [
                    {
                        "Days": 30,
                        "StorageClass": "INTELLIGENT_TIERING",
                    },
                ],
            },
            {
                "ID": f"{PROJECT_NAME}-logs-glacier",
                "Status": "Enabled",
                "Filter": {"Prefix": prefix},
                "Transitions": [
                    {
                        "Days": glacier_days,
                        "StorageClass": "GLACIER_IR",
                    },
                ],
            },
            {
                "ID": f"{PROJECT_NAME}-logs-deep-archive",
                "Status": "Enabled",
                "Filter": {"Prefix": prefix},
                "Transitions": [
                    {
                        "Days": deep_archive_days,
                        "StorageClass": "DEEP_ARCHIVE",
                    },
                ],
            },
            {
                "ID": f"{PROJECT_NAME}-logs-expiration",
                "Status": "Enabled",
                "Filter": {"Prefix": prefix},
                "Expiration": {"Days": expiration_days},
            },
            {
                "ID": f"{PROJECT_NAME}-logs-abort-multipart",
                "Status": "Enabled",
                "Filter": {"Prefix": prefix},
                "AbortIncompleteMultipartUpload": {"DaysAfterInitiation": 7},
            },
        ]
    }

    s3.put_bucket_lifecycle_configuration(
        Bucket=bucket_name,
        LifecycleConfiguration=lifecycle_config,
    )
    print(f"Lifecycle rules configured on {bucket_name}:")
    print(f"  Intelligent-Tiering: 30 days")
    print(f"  Glacier IR: {glacier_days} days")
    print(f"  Deep Archive: {deep_archive_days} days")
    print(f"  Expiration: {expiration_days} days")
```

**Write logging outputs to SSM:**
```python
ssm = boto3.client("ssm", region_name=AWS_REGION)

def write_logging_outputs_to_ssm(log_group_name, bucket_name, athena_database, alarm_arn):
    """Write Bedrock logging resource identifiers to SSM for cross-template consumption."""
    params = {
        f"/mlops/{PROJECT_NAME}/{ENV}/bedrock-log-group": log_group_name,
        f"/mlops/{PROJECT_NAME}/{ENV}/bedrock-log-bucket": bucket_name or "none",
        f"/mlops/{PROJECT_NAME}/{ENV}/bedrock-athena-database": athena_database or "none",
        f"/mlops/{PROJECT_NAME}/{ENV}/bedrock-error-alarm-arn": alarm_arn,
    }
    for name, value in params.items():
        ssm.put_parameter(
            Name=name,
            Description=f"Bedrock logging output for {PROJECT_NAME} {ENV}",
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

- **Upstream**: `devops/04` → IAM roles (Bedrock execution role, Lambda execution role) that perform Bedrock invocations; the logging role must be separate from execution roles and trusted by `bedrock.amazonaws.com`
- **Upstream**: `mlops/12` → Bedrock Guardrails logging; guardrail evaluation events appear in invocation logs alongside model responses, enabling correlation of guardrail blocks with invocation errors
- **Downstream**: `devops/13` → Cost-per-inference dashboards consume the Athena cost-per-invocation queries and CloudWatch custom metrics (token counts, latency) produced by this template
- **Downstream**: `finops/01` → Cost allocation tags on the S3 log bucket and CloudWatch log group enable FinOps cost tracking; the Athena cost queries feed into cost allocation reports
- **Downstream**: `finops/05` → FinOps dashboards consume the CloudWatch metrics (error rate, latency percentiles, token usage) and Athena query results for Bedrock spend visualization
- **Downstream**: `devops/11` → Custom CloudWatch metrics published by this template's metric filters (ErrorCount, InvocationCount, Latency, InputTokens, OutputTokens) are consumed by the custom model quality monitoring template for anomaly detection
- **Downstream**: `mlops/19` → Marketplace model invocations are captured in the same invocation logs; the named queries and cost tracking apply to marketplace models alongside first-party models