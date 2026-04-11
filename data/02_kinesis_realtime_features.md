<!-- Template Version: 1.0 | boto3: 1.35+ | Managed Flink: 1.19+ -->

# Template Data 02 — Kinesis Real-Time Feature Computation

## Purpose
Generate a production-ready real-time feature computation pipeline using Amazon Kinesis Data Streams, AWS Lambda, Amazon Data Firehose, and Amazon Managed Service for Apache Flink: a Kinesis Data Stream with configurable shard count, a Lambda consumer that computes ML features from incoming records with dead-letter queue routing, a Firehose delivery stream writing processed features to S3 in Parquet format and optionally to OpenSearch for real-time serving, a Flink application for windowed streaming aggregations (sliding window averages, counts, percentiles), a feature schema definition mapping raw fields to computed features with type validation, and integration with SageMaker Feature Store online store for real-time inference.

---

## Role Definition

You are an expert AWS streaming data engineer and ML feature platform specialist with expertise in:
- Amazon Kinesis Data Streams: shard management, enhanced fan-out, producer/consumer patterns, KPL/KCL
- AWS Lambda: Kinesis event source mapping, batch processing, bisect-on-error, parallelization factor, tumbling windows
- Amazon Data Firehose: delivery streams to S3 (Parquet format conversion via Glue schema) and OpenSearch, buffering hints, error handling
- Amazon Managed Service for Apache Flink: Flink SQL for streaming aggregations, sliding/tumbling/session windows, watermarks, late data handling
- SageMaker Feature Store: online store `put_record()` for real-time feature serving, feature group schema alignment
- Dead-letter queue patterns: SQS DLQ for malformed records, CloudWatch error metric emission, retry strategies
- Feature engineering: real-time numerical aggregations, windowed statistics, sessionization, event counting
- IAM policies for Kinesis, Lambda, Firehose, Flink, SQS, and Feature Store cross-service permissions

Generate complete, production-deployable code.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

STREAM_NAME:            [REQUIRED - name suffix for the Kinesis Data Stream]
                         Example: "clickstream-events"
                         Full name becomes: {PROJECT_NAME}-{STREAM_NAME}-{ENV}

SHARD_COUNT:            [REQUIRED - number of shards for the Kinesis Data Stream]
                         Example: 4
                         Each shard supports 1 MB/s write, 2 MB/s read.
                         Size based on expected throughput.

FEATURE_SCHEMA:         [REQUIRED - JSON mapping raw stream fields to computed features]
                         Example:
                         {
                           "raw_fields": ["user_id", "event_type", "amount", "timestamp"],
                           "computed_features": [
                             {"name": "amount_log", "type": "double", "transform": "log1p(amount)"},
                             {"name": "event_count_1h", "type": "int", "transform": "count(event_type, window=1h)"},
                             {"name": "amount_avg_15m", "type": "double", "transform": "avg(amount, window=15m)"}
                           ],
                           "record_identifier": "user_id",
                           "event_time_field": "timestamp"
                         }

WINDOW_SIZES:           [REQUIRED - comma-separated window durations for Flink aggregations]
                         Example: "1m,5m,15m,1h"
                         Used for sliding window averages, counts, and percentiles.

FIREHOSE_DESTINATION:   [REQUIRED - s3 | opensearch | both]
                         - s3: Firehose delivers to S3 in Parquet format (via Glue schema)
                         - opensearch: Firehose delivers to OpenSearch for real-time serving
                         - both: two Firehose streams — one to S3 Parquet, one to OpenSearch

DLQ_QUEUE_NAME:         [REQUIRED - name suffix for the SQS dead-letter queue]
                         Example: "feature-dlq"
                         Full name becomes: {PROJECT_NAME}-{DLQ_QUEUE_NAME}-{ENV}

FEATURE_GROUP_NAME:     [OPTIONAL - SageMaker Feature Store feature group name]
                         If provided, computed features are written to the online store
                         via sagemaker_featurestore_runtime.put_record().

OPENSEARCH_DOMAIN:      [OPTIONAL - OpenSearch domain endpoint]
                         Required if FIREHOSE_DESTINATION includes opensearch.
                         Example: "https://search-myai-features-abc123.us-east-1.es.amazonaws.com"

OPENSEARCH_INDEX:       [OPTIONAL: features]
                         OpenSearch index name for feature documents.

BATCH_SIZE:             [OPTIONAL: 100]
                         Lambda event source mapping batch size (1-10000).

PARALLELIZATION_FACTOR: [OPTIONAL: 1]
                         Lambda parallelization factor per shard (1-10).

RETENTION_HOURS:        [OPTIONAL: 24]
                         Kinesis Data Stream retention period in hours (24-8760).

FLINK_PARALLELISM:      [OPTIONAL: 2]
                         Flink application parallelism for streaming aggregations.

SNS_TOPIC_ARN:          [OPTIONAL - SNS topic for DLQ and error alerting]
                         If not provided, a new topic is created.
```

---

## Task

Generate complete real-time feature computation pipeline:

```
{PROJECT_NAME}-kinesis-features/
├── stream/
│   ├── create_stream.py               # Create Kinesis Data Stream
│   └── producer_example.py            # Example producer with put_record()
├── lambda_consumer/
│   ├── handler.py                     # Lambda function: compute features from Kinesis records
│   ├── feature_transforms.py          # Feature computation logic (numerical, windowed counters)
│   ├── feature_schema.py              # Feature schema definition and validation
│   ├── feature_store_writer.py        # Write computed features to SageMaker Feature Store online store
│   └── dlq_handler.py                 # DLQ routing for malformed records
├── firehose/
│   ├── create_s3_firehose.py          # Firehose delivery stream → S3 Parquet
│   ├── create_opensearch_firehose.py  # Firehose delivery stream → OpenSearch
│   └── transform_lambda.py           # Optional Firehose transform Lambda
├── flink/
│   ├── create_flink_app.py            # Create Managed Flink application
│   ├── flink_sql_aggregations.sql     # Flink SQL for windowed aggregations
│   └── flink_app.py                   # PyFlink application code
├── dlq/
│   ├── create_dlq.py                  # Create SQS dead-letter queue
│   └── dlq_processor.py              # Lambda to process DLQ messages (retry or alert)
├── infrastructure/
│   ├── create_event_source_mapping.py # Lambda ↔ Kinesis event source mapping
│   ├── config.py                      # Central configuration
│   └── setup_monitoring.py            # CloudWatch alarms and dashboards
├── run_setup.py                       # CLI orchestrator
└── requirements.txt
```

**config.py**: Central configuration dataclass with all parameters. Load from environment variables or CLI args. Validate required fields. Construct resource names using `{PROJECT_NAME}-{component}-{ENV}` convention.

**create_stream.py**: Create Kinesis Data Stream using `kinesis.create_stream()`:
- Set `StreamName` to `{PROJECT_NAME}-{STREAM_NAME}-{ENV}`
- Set `ShardCount` to SHARD_COUNT (use `StreamModeDetails` with `StreamMode='PROVISIONED'`)
- Configure `RetentionPeriodHours` to RETENTION_HOURS
- Enable server-side encryption with `StreamEncryption` using KMS key
- Tag with Project and Environment
- Wait for stream to become ACTIVE using `kinesis.describe_stream()` polling

**producer_example.py**: Example Kinesis producer for testing:
- Use `kinesis.put_record()` with `StreamName`, `Data` (JSON-encoded record), `PartitionKey` (use record_identifier field for even distribution)
- Include batch producer using `kinesis.put_records()` for high-throughput ingestion
- Demonstrate proper partition key selection for even shard distribution
- Include error handling for `ProvisionedThroughputExceededException` with exponential backoff

**handler.py**: Lambda consumer function triggered by Kinesis event source mapping:
- Parse `event['Records']` — each record has `kinesis.data` (base64-encoded)
- Decode and parse JSON payload from each record
- Validate record against FEATURE_SCHEMA using `feature_schema.py`
- On validation failure: route to DLQ via `dlq_handler.py`, emit CloudWatch metric `MalformedRecords`
- On success: compute features using `feature_transforms.py`
- Write computed features to Feature Store online store via `feature_store_writer.py` (if FEATURE_GROUP_NAME set)
- Return `batchItemFailures` for partial batch failure reporting (bisect-on-error pattern)

**feature_transforms.py**: Feature computation logic:
- `compute_features(raw_record, schema)`: Apply transforms defined in FEATURE_SCHEMA
- `log_transform(value)`: log1p for skewed numerical fields
- `ratio_transform(numerator, denominator)`: safe division with zero handling
- `bin_transform(value, boundaries)`: bucket numerical values into bins
- `time_features(timestamp)`: extract hour, day_of_week, is_weekend from event timestamp
- All transforms operate on single records (point-in-time features); windowed aggregations are handled by Flink

**feature_schema.py**: Feature schema definition and validation:
- Load FEATURE_SCHEMA JSON and validate structure
- `validate_record(record, schema)`: check required raw_fields exist, types match, values within expected ranges
- `get_output_schema()`: return the computed feature schema for downstream consumers
- Raise `SchemaValidationError` with details on which fields failed

**feature_store_writer.py**: Write to SageMaker Feature Store online store:
- Use `sagemaker_featurestore_runtime.put_record()` with `FeatureGroupName`, `Record` (list of `FeatureValue` dicts)
- Map computed features to Feature Store record format: `[{"FeatureName": name, "ValueAsString": str(value)}]`
- Include `record_identifier` and `event_time` fields
- Handle `ValidationError` and `ResourceNotFound` exceptions with CloudWatch metric emission

**dlq_handler.py**: Dead-letter queue routing:
- Send malformed records to SQS DLQ using `sqs.send_message()` with original record, error details, and timestamp
- Emit CloudWatch custom metric `{PROJECT_NAME}/DLQMessages` with dimensions for error type
- Log error details to CloudWatch Logs for debugging

**create_s3_firehose.py**: Create Firehose delivery stream to S3 with Parquet format conversion:
- Use `firehose.create_delivery_stream()` with `DeliveryStreamType='KinesisStreamAsSource'`
- Configure `KinesisStreamSourceConfiguration` pointing to the Kinesis Data Stream
- Configure `ExtendedS3DestinationConfiguration` with:
  - `BucketARN` for feature output bucket
  - `Prefix` with partitioning: `features/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/`
  - `ErrorOutputPrefix` for failed records
  - `BufferingHints` with `SizeInMBs=128` and `IntervalInSeconds=300`
  - `DataFormatConversionConfiguration` with Parquet serializer and Glue table schema reference
- Note: format conversion to Parquet requires S3-only destination (cannot combine with OpenSearch on same stream)

**create_opensearch_firehose.py**: Create Firehose delivery stream to OpenSearch:
- Use `firehose.create_delivery_stream()` with `DeliveryStreamType='KinesisStreamAsSource'`
- Configure `AmazonopensearchserviceDestinationConfiguration` with:
  - `DomainARN` or `ClusterEndpoint` for the OpenSearch domain
  - `IndexName` set to OPENSEARCH_INDEX
  - `IndexRotationPeriod` set to `OneDay` for time-series feature data
  - `BufferingHints` with `IntervalInSeconds=60` and `SizeInMBs=5`
  - `RetryOptions` with `DurationInSeconds=300`
  - `S3BackupMode='FailedDocumentsOnly'` with S3 backup configuration
- This is a separate Firehose stream from the S3 Parquet stream (Firehose does not support Parquet conversion + OpenSearch on the same stream)

**transform_lambda.py**: Optional Firehose data transformation Lambda:
- Process `event['records']` — each record has `recordId`, `data` (base64-encoded)
- Decode, transform (add computed fields, flatten nested JSON), re-encode
- Return records with `result='Ok'`, `result='Dropped'`, or `result='ProcessingFailed'`

**create_flink_app.py**: Create Managed Service for Apache Flink application:
- Use `kinesisanalyticsv2.create_application()` with `RuntimeEnvironment='FLINK-1_19'`
- Configure `ApplicationConfiguration` with:
  - `FlinkApplicationConfiguration` with `ParallelismConfiguration` (parallelism=FLINK_PARALLELISM)
  - `ApplicationCodeConfiguration` pointing to S3 location of Flink app code
  - `EnvironmentProperties` with Kinesis stream name, region, and window sizes
- Start application using `kinesisanalyticsv2.start_application()`

**flink_sql_aggregations.sql**: Flink SQL for windowed streaming aggregations:
- Define source table connected to Kinesis Data Stream with `WATERMARK` for event time
- Define sliding window aggregations for each WINDOW_SIZE:
  - `COUNT(*)` — event count per window
  - `AVG(amount)` — average per window
  - `APPROX_PERCENTILE(amount, 0.95)` — p95 per window
- Define tumbling window aggregations for session-level features
- Output to Kinesis Data Stream or direct to S3 via Firehose connector

**flink_app.py**: PyFlink application code:
- Initialize `StreamExecutionEnvironment` and `StreamTableEnvironment`
- Register Kinesis source table with watermark strategy
- Execute SQL aggregations from `flink_sql_aggregations.sql`
- Register sink table (Kinesis output stream or Firehose)
- Execute the Flink job

**create_dlq.py**: Create SQS dead-letter queue:
- Use `sqs.create_queue()` with `QueueName` set to `{PROJECT_NAME}-{DLQ_QUEUE_NAME}-{ENV}`
- Configure `MessageRetentionPeriod` to 14 days (1209600 seconds)
- Configure `VisibilityTimeout` to 300 seconds
- Enable server-side encryption with KMS
- Create CloudWatch alarm on `ApproximateNumberOfMessagesVisible` > 0 for alerting

**dlq_processor.py**: Lambda function to process DLQ messages:
- Poll SQS queue for failed records
- Attempt re-processing with relaxed validation (log warnings instead of errors)
- If re-processing fails, publish to SNS for manual investigation
- Delete successfully re-processed messages from queue

**create_event_source_mapping.py**: Create Lambda event source mapping for Kinesis:
- Use `lambda_client.create_event_source_mapping()` with:
  - `EventSourceArn` pointing to the Kinesis Data Stream ARN
  - `FunctionName` for the Lambda consumer
  - `StartingPosition='LATEST'` (or `'TRIM_HORIZON'` for backfill)
  - `BatchSize` set to BATCH_SIZE
  - `ParallelizationFactor` set to PARALLELIZATION_FACTOR
  - `MaximumBatchingWindowInSeconds=5` for micro-batching
  - `BisectBatchOnFunctionError=True` for partial failure handling
  - `MaximumRetryAttempts=3`
  - `DestinationConfig` with `OnFailure` pointing to DLQ ARN
  - `FunctionResponseTypes=['ReportBatchItemFailures']` for granular error reporting

**setup_monitoring.py**: CloudWatch alarms and dashboard:
- Alarm on `GetRecords.IteratorAgeMilliseconds` > threshold (consumer falling behind)
- Alarm on `WriteProvisionedThroughputExceeded` > 0 (producer throttling)
- Alarm on `ReadProvisionedThroughputExceeded` > 0 (consumer throttling)
- Alarm on Lambda `Errors` metric > threshold
- Alarm on DLQ `ApproximateNumberOfMessagesVisible` > 0
- Dashboard with stream throughput, Lambda duration/errors, Firehose delivery metrics, DLQ depth

**run_setup.py**: CLI orchestrator that runs setup steps in order:
1. Create Kinesis Data Stream and wait for ACTIVE
2. Create SQS dead-letter queue
3. Create Firehose delivery stream(s) based on FIREHOSE_DESTINATION
4. Create Lambda consumer function (assumes code is packaged and uploaded)
5. Create Lambda ↔ Kinesis event source mapping
6. Create Flink application (if windowed aggregations needed)
7. Set up CloudWatch monitoring
8. Print summary with stream ARN, Lambda function name, Firehose names, DLQ URL

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**Kinesis Data Stream:** Use provisioned mode with explicit shard count for predictable throughput and cost. Each shard supports 1 MB/s (1,000 records/s) write and 2 MB/s read. Enable server-side encryption using KMS. Set retention to 24 hours minimum; increase for replay scenarios. Use partition keys that distribute evenly across shards (e.g., hash of user_id).

**Lambda Consumer:** Configure event source mapping with `BisectBatchOnFunctionError=True` and `ReportBatchItemFailures` to handle partial failures without reprocessing the entire batch. Set `MaximumRetryAttempts=3` before routing to DLQ. Use `ParallelizationFactor` up to 10 for high-throughput streams. Keep Lambda timeout under 5 minutes for streaming workloads. Process records idempotently — the same record may be delivered more than once.

**Firehose:** For S3 Parquet delivery, use `DataFormatConversionConfiguration` with a Glue table schema reference. Parquet format conversion requires `BufferingHints.SizeInMBs` >= 64 (default 128 when format conversion is enabled). Set `CompressionFormat` to `UNCOMPRESSED` when format conversion is enabled (Snappy compression is applied automatically by the Parquet serializer). For OpenSearch delivery, use a separate Firehose stream since format conversion is S3-only. Set `S3BackupMode='FailedDocumentsOnly'` for OpenSearch to capture delivery failures.

**Flink:** Use `FLINK-1_19` runtime for latest features. Define watermarks on event time columns to handle late-arriving data. Use sliding windows for continuous aggregations and tumbling windows for periodic snapshots. Set checkpointing interval to 60 seconds for fault tolerance. Monitor `lastCheckpointDuration` and `numberOfFailedCheckpoints` CloudWatch metrics.

**Dead-Letter Queue:** Route all malformed, schema-invalid, and repeatedly-failing records to SQS DLQ. Set message retention to 14 days for investigation. Emit CloudWatch custom metrics on every DLQ send with error type dimensions. Create a CloudWatch alarm on DLQ depth > 0 for immediate alerting.

**Feature Store Integration:** Use `sagemaker_featurestore_runtime.put_record()` for online store writes. Map computed features to `FeatureValue` format: `[{"FeatureName": "amount_log", "ValueAsString": "2.345"}]`. Include `record_identifier` and `event_time` fields. Handle throttling with exponential backoff. Batch writes are not supported for online store — each record requires a separate `put_record()` call.

**Security:** Encrypt Kinesis stream with KMS. Encrypt SQS DLQ with KMS. Firehose delivery role needs `kinesis:GetRecords`, `kinesis:GetShardIterator`, `kinesis:DescribeStream` on the source stream, `s3:PutObject` on the destination bucket, and `es:ESHttpPut` on the OpenSearch domain. Lambda execution role needs `kinesis:GetRecords`, `kinesis:GetShardIterator`, `kinesis:DescribeStreamSummary`, `kinesis:ListShards`, `sqs:SendMessage` on DLQ, and `sagemaker:PutRecord` on the Feature Store. Deploy Lambda in VPC if accessing OpenSearch or Feature Store via VPC endpoints.

**Cost:** Kinesis pricing is per-shard-hour plus per-PUT-payload-unit. Right-size shard count to actual throughput. Use on-demand mode for unpredictable workloads. Firehose pricing is per-GB ingested. Flink pricing is per-KPU-hour — start with minimal parallelism and scale based on `millisBehindLatest`. Lambda pricing is per-invocation plus duration — larger batch sizes reduce invocation count.

**Naming:** All resources follow `{PROJECT_NAME}-{component}-{ENV}` convention:
- Kinesis stream: `{PROJECT_NAME}-{STREAM_NAME}-{ENV}`
- Lambda consumer: `{PROJECT_NAME}-feature-consumer-{ENV}`
- Firehose (S3): `{PROJECT_NAME}-firehose-s3-features-{ENV}`
- Firehose (OpenSearch): `{PROJECT_NAME}-firehose-os-features-{ENV}`
- Flink app: `{PROJECT_NAME}-flink-aggregations-{ENV}`
- DLQ: `{PROJECT_NAME}-{DLQ_QUEUE_NAME}-{ENV}`
- CloudWatch dashboard: `{PROJECT_NAME}-streaming-features-{ENV}`

---

## Code Scaffolding Hints

**Create Kinesis Data Stream:**
```python
import boto3
import time

kinesis = boto3.client("kinesis", region_name=AWS_REGION)

kinesis.create_stream(
    StreamName=f"{PROJECT_NAME}-{STREAM_NAME}-{ENV}",
    ShardCount=SHARD_COUNT,
    StreamModeDetails={"StreamMode": "PROVISIONED"},
)

# Wait for stream to become ACTIVE
while True:
    desc = kinesis.describe_stream(
        StreamName=f"{PROJECT_NAME}-{STREAM_NAME}-{ENV}"
    )
    status = desc["StreamDescription"]["StreamStatus"]
    if status == "ACTIVE":
        break
    time.sleep(5)

# Enable server-side encryption
kinesis.start_stream_encryption(
    StreamName=f"{PROJECT_NAME}-{STREAM_NAME}-{ENV}",
    EncryptionType="KMS",
    KeyId=kms_key_id,
)

# Set retention period
kinesis.increase_stream_retention_period(
    StreamName=f"{PROJECT_NAME}-{STREAM_NAME}-{ENV}",
    RetentionPeriodHours=RETENTION_HOURS,
)

# Tag the stream
kinesis.add_tags_to_stream(
    StreamName=f"{PROJECT_NAME}-{STREAM_NAME}-{ENV}",
    Tags={"Project": PROJECT_NAME, "Environment": ENV},
)

stream_arn = desc["StreamDescription"]["StreamARN"]
```

**Producer example with put_record() and put_records():**
```python
import json
import hashlib

kinesis = boto3.client("kinesis", region_name=AWS_REGION)

# Single record put
def put_single_record(stream_name, record):
    """Put a single record to Kinesis with partition key from record identifier."""
    partition_key = str(record.get("user_id", "default"))
    response = kinesis.put_record(
        StreamName=stream_name,
        Data=json.dumps(record).encode("utf-8"),
        PartitionKey=partition_key,
    )
    return response["ShardId"], response["SequenceNumber"]

# Batch put with error handling
def put_batch_records(stream_name, records, max_retries=3):
    """Put batch of records with retry on ProvisionedThroughputExceededException."""
    kinesis_records = [
        {
            "Data": json.dumps(r).encode("utf-8"),
            "PartitionKey": str(r.get("user_id", "default")),
        }
        for r in records
    ]

    retry_count = 0
    while kinesis_records and retry_count < max_retries:
        response = kinesis.put_records(
            StreamName=stream_name,
            Records=kinesis_records,
        )
        failed_count = response["FailedRecordCount"]
        if failed_count == 0:
            break

        # Retry only failed records with exponential backoff
        kinesis_records = [
            rec
            for rec, res in zip(kinesis_records, response["Records"])
            if "ErrorCode" in res
        ]
        retry_count += 1
        time.sleep(2 ** retry_count)

    return response
```

**Lambda consumer handler with batch item failure reporting:**
```python
import json
import base64
import os
import boto3
from datetime import datetime

sqs = boto3.client("sqs")
cloudwatch = boto3.client("cloudwatch")

DLQ_URL = os.environ["DLQ_URL"]
FEATURE_GROUP_NAME = os.environ.get("FEATURE_GROUP_NAME")
METRIC_NAMESPACE = os.environ.get("METRIC_NAMESPACE", f"{os.environ['PROJECT_NAME']}/StreamingFeatures")

def handler(event, context):
    """Process Kinesis records and compute ML features."""
    batch_item_failures = []

    for record in event["Records"]:
        try:
            # Decode Kinesis record
            payload = base64.b64decode(record["kinesis"]["data"])
            raw_record = json.loads(payload)

            # Validate against feature schema
            validated = validate_record(raw_record, FEATURE_SCHEMA)
            if not validated["valid"]:
                route_to_dlq(raw_record, validated["errors"], record["kinesis"]["sequenceNumber"])
                emit_metric("MalformedRecords", 1)
                continue

            # Compute features
            computed = compute_features(raw_record, FEATURE_SCHEMA)

            # Write to Feature Store online store (if configured)
            if FEATURE_GROUP_NAME:
                write_to_feature_store(computed)

            emit_metric("ProcessedRecords", 1)

        except Exception as e:
            # Report this record as failed for bisect-on-error
            batch_item_failures.append({
                "itemIdentifier": record["kinesis"]["sequenceNumber"]
            })
            emit_metric("ProcessingErrors", 1)

    return {"batchItemFailures": batch_item_failures}

def route_to_dlq(record, errors, sequence_number):
    """Send malformed record to SQS dead-letter queue."""
    sqs.send_message(
        QueueUrl=DLQ_URL,
        MessageBody=json.dumps({
            "original_record": record,
            "errors": errors,
            "sequence_number": sequence_number,
            "timestamp": datetime.utcnow().isoformat(),
        }),
        MessageAttributes={
            "ErrorType": {"StringValue": "SchemaValidation", "DataType": "String"},
        },
    )

def emit_metric(metric_name, value):
    """Emit CloudWatch custom metric."""
    cloudwatch.put_metric_data(
        Namespace=METRIC_NAMESPACE,
        MetricData=[{
            "MetricName": metric_name,
            "Value": value,
            "Unit": "Count",
            "Dimensions": [
                {"Name": "Environment", "Value": os.environ.get("ENV", "dev")},
            ],
        }],
    )
```

**Feature Store online store integration:**
```python
import boto3
from datetime import datetime

featurestore_runtime = boto3.client("sagemaker-featurestore-runtime", region_name=AWS_REGION)

def write_to_feature_store(computed_features):
    """Write computed features to SageMaker Feature Store online store."""
    record = [
        {"FeatureName": name, "ValueAsString": str(value)}
        for name, value in computed_features.items()
    ]

    # Ensure event_time is ISO 8601
    if not any(f["FeatureName"] == "event_time" for f in record):
        record.append({
            "FeatureName": "event_time",
            "ValueAsString": datetime.utcnow().isoformat() + "Z",
        })

    featurestore_runtime.put_record(
        FeatureGroupName=FEATURE_GROUP_NAME,
        Record=record,
    )
```

**Lambda event source mapping for Kinesis:**
```python
lambda_client = boto3.client("lambda", region_name=AWS_REGION)

lambda_client.create_event_source_mapping(
    EventSourceArn=stream_arn,
    FunctionName=f"{PROJECT_NAME}-feature-consumer-{ENV}",
    Enabled=True,
    BatchSize=BATCH_SIZE,
    ParallelizationFactor=PARALLELIZATION_FACTOR,
    StartingPosition="LATEST",
    MaximumBatchingWindowInSeconds=5,
    MaximumRetryAttempts=3,
    BisectBatchOnFunctionError=True,
    DestinationConfig={
        "OnFailure": {
            "Destination": dlq_arn,
        }
    },
    FunctionResponseTypes=["ReportBatchItemFailures"],
)
```

**Firehose delivery stream to S3 with Parquet format conversion:**
```python
firehose = boto3.client("firehose", region_name=AWS_REGION)

firehose.create_delivery_stream(
    DeliveryStreamName=f"{PROJECT_NAME}-firehose-s3-features-{ENV}",
    DeliveryStreamType="KinesisStreamAsSource",
    KinesisStreamSourceConfiguration={
        "KinesisStreamARN": stream_arn,
        "RoleARN": firehose_role_arn,
    },
    ExtendedS3DestinationConfiguration={
        "RoleARN": firehose_role_arn,
        "BucketARN": f"arn:aws:s3:::{PROJECT_NAME}-features-{ENV}",
        "Prefix": "features/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/",
        "ErrorOutputPrefix": "errors/!{firehose:error-output-type}/year=!{timestamp:yyyy}/",
        "BufferingHints": {
            "SizeInMBs": 128,
            "IntervalInSeconds": 300,
        },
        "CompressionFormat": "UNCOMPRESSED",  # Required when format conversion is enabled
        "DataFormatConversionConfiguration": {
            "Enabled": True,
            "SchemaConfiguration": {
                "RoleARN": firehose_role_arn,
                "DatabaseName": f"{PROJECT_NAME}_features_{ENV}",
                "TableName": f"{PROJECT_NAME}_stream_features",
                "Region": AWS_REGION,
                "VersionId": "LATEST",
            },
            "InputFormatConfiguration": {
                "Deserializer": {
                    "OpenXJsonSerDe": {},
                },
            },
            "OutputFormatConfiguration": {
                "Serializer": {
                    "ParquetSerDe": {
                        "Compression": "SNAPPY",
                    },
                },
            },
        },
        "CloudWatchLoggingOptions": {
            "Enabled": True,
            "LogGroupName": f"/aws/firehose/{PROJECT_NAME}-firehose-s3-features-{ENV}",
            "LogStreamName": "S3Delivery",
        },
    },
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)
```

**Firehose delivery stream to OpenSearch:**
```python
firehose.create_delivery_stream(
    DeliveryStreamName=f"{PROJECT_NAME}-firehose-os-features-{ENV}",
    DeliveryStreamType="KinesisStreamAsSource",
    KinesisStreamSourceConfiguration={
        "KinesisStreamARN": stream_arn,
        "RoleARN": firehose_role_arn,
    },
    AmazonopensearchserviceDestinationConfiguration={
        "RoleARN": firehose_role_arn,
        "DomainARN": opensearch_domain_arn,
        "IndexName": OPENSEARCH_INDEX,
        "IndexRotationPeriod": "OneDay",
        "BufferingHints": {
            "IntervalInSeconds": 60,
            "SizeInMBs": 5,
        },
        "RetryOptions": {
            "DurationInSeconds": 300,
        },
        "S3BackupMode": "FailedDocumentsOnly",
        "S3Configuration": {
            "RoleARN": firehose_role_arn,
            "BucketARN": f"arn:aws:s3:::{PROJECT_NAME}-firehose-backup-{ENV}",
            "Prefix": "opensearch-failures/",
            "BufferingHints": {"SizeInMBs": 5, "IntervalInSeconds": 300},
            "CompressionFormat": "GZIP",
        },
        "CloudWatchLoggingOptions": {
            "Enabled": True,
            "LogGroupName": f"/aws/firehose/{PROJECT_NAME}-firehose-os-features-{ENV}",
            "LogStreamName": "OpenSearchDelivery",
        },
    },
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)
```

**Flink SQL for windowed aggregations:**
```sql
-- Source table connected to Kinesis Data Stream
CREATE TABLE kinesis_source (
    user_id       VARCHAR,
    event_type    VARCHAR,
    amount        DOUBLE,
    event_time    TIMESTAMP(3),
    WATERMARK FOR event_time AS event_time - INTERVAL '5' SECOND
) WITH (
    'connector' = 'kinesis',
    'stream'    = '{PROJECT_NAME}-{STREAM_NAME}-{ENV}',
    'aws.region' = '{AWS_REGION}',
    'scan.stream.initpos' = 'LATEST',
    'format'    = 'json'
);

-- Sliding window: 15-minute average amount per user
CREATE TABLE features_15m AS
SELECT
    user_id,
    TUMBLE_START(event_time, INTERVAL '15' MINUTE) AS window_start,
    COUNT(*)                                        AS event_count_15m,
    AVG(amount)                                     AS amount_avg_15m,
    MAX(amount)                                     AS amount_max_15m,
    MIN(amount)                                     AS amount_min_15m
FROM kinesis_source
GROUP BY user_id, TUMBLE(event_time, INTERVAL '15' MINUTE);

-- Sliding window: 1-hour aggregations per user
CREATE TABLE features_1h AS
SELECT
    user_id,
    HOP_START(event_time, INTERVAL '5' MINUTE, INTERVAL '1' HOUR) AS window_start,
    COUNT(*)                                                        AS event_count_1h,
    AVG(amount)                                                     AS amount_avg_1h,
    STDDEV_POP(amount)                                              AS amount_stddev_1h
FROM kinesis_source
GROUP BY user_id, HOP(event_time, INTERVAL '5' MINUTE, INTERVAL '1' HOUR);

-- Sink table to output Kinesis stream for downstream consumption
CREATE TABLE feature_sink (
    user_id           VARCHAR,
    window_start      TIMESTAMP(3),
    event_count_15m   BIGINT,
    amount_avg_15m    DOUBLE,
    event_count_1h    BIGINT,
    amount_avg_1h     DOUBLE
) WITH (
    'connector' = 'kinesis',
    'stream'    = '{PROJECT_NAME}-aggregated-features-{ENV}',
    'aws.region' = '{AWS_REGION}',
    'format'    = 'json'
);
```

**Create Managed Flink application:**
```python
kinesisanalyticsv2 = boto3.client("kinesisanalyticsv2", region_name=AWS_REGION)

kinesisanalyticsv2.create_application(
    ApplicationName=f"{PROJECT_NAME}-flink-aggregations-{ENV}",
    RuntimeEnvironment="FLINK-1_19",
    ServiceExecutionRole=flink_role_arn,
    ApplicationConfiguration={
        "FlinkApplicationConfiguration": {
            "CheckpointConfiguration": {
                "ConfigurationType": "CUSTOM",
                "CheckpointingEnabled": True,
                "CheckpointInterval": 60000,
                "MinPauseBetweenCheckpoints": 5000,
            },
            "ParallelismConfiguration": {
                "ConfigurationType": "CUSTOM",
                "Parallelism": FLINK_PARALLELISM,
                "ParallelismPerKPU": 1,
                "AutoScalingEnabled": True,
            },
            "MonitoringConfiguration": {
                "ConfigurationType": "CUSTOM",
                "MetricsLevel": "APPLICATION",
                "LogLevel": "INFO",
            },
        },
        "ApplicationCodeConfiguration": {
            "CodeContent": {
                "S3ContentLocation": {
                    "BucketARN": f"arn:aws:s3:::{PROJECT_NAME}-code-{ENV}",
                    "FileKey": "flink/flink_app.zip",
                },
            },
            "CodeContentType": "ZIPFILE",
        },
        "EnvironmentProperties": {
            "PropertyGroups": [
                {
                    "PropertyGroupId": "KinesisSource",
                    "PropertyMap": {
                        "stream.name": f"{PROJECT_NAME}-{STREAM_NAME}-{ENV}",
                        "aws.region": AWS_REGION,
                    },
                },
                {
                    "PropertyGroupId": "WindowConfig",
                    "PropertyMap": {
                        "window.sizes": WINDOW_SIZES,
                    },
                },
            ],
        },
    },
    CloudWatchLoggingOptions=[
        {
            "LogStreamARN": f"arn:aws:logs:{AWS_REGION}:{AWS_ACCOUNT_ID}:log-group:/aws/kinesis-analytics/{PROJECT_NAME}-flink-aggregations-{ENV}:log-stream:kinesis-analytics-log-stream",
        }
    ],
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)

# Start the application
kinesisanalyticsv2.start_application(
    ApplicationName=f"{PROJECT_NAME}-flink-aggregations-{ENV}",
    RunConfiguration={
        "FlinkRunConfiguration": {
            "AllowNonRestoredState": True,
        },
    },
)
```

**Create SQS dead-letter queue:**
```python
sqs = boto3.client("sqs", region_name=AWS_REGION)

dlq_response = sqs.create_queue(
    QueueName=f"{PROJECT_NAME}-{DLQ_QUEUE_NAME}-{ENV}",
    Attributes={
        "MessageRetentionPeriod": "1209600",  # 14 days
        "VisibilityTimeout": "300",
        "KmsMasterKeyId": kms_key_id,
        "KmsDataKeyReusePeriodSeconds": "300",
    },
    tags={"Project": PROJECT_NAME, "Environment": ENV},
)
dlq_url = dlq_response["QueueUrl"]

# Get DLQ ARN for event source mapping
dlq_attrs = sqs.get_queue_attributes(
    QueueUrl=dlq_url,
    AttributeNames=["QueueArn"],
)
dlq_arn = dlq_attrs["Attributes"]["QueueArn"]
```

**CloudWatch monitoring setup:**
```python
cloudwatch = boto3.client("cloudwatch", region_name=AWS_REGION)

# Alarm: consumer falling behind (iterator age > 1 minute)
cloudwatch.put_metric_alarm(
    AlarmName=f"{PROJECT_NAME}-iterator-age-{ENV}",
    Namespace="AWS/Kinesis",
    MetricName="GetRecords.IteratorAgeMilliseconds",
    Dimensions=[
        {"Name": "StreamName", "Value": f"{PROJECT_NAME}-{STREAM_NAME}-{ENV}"},
    ],
    Statistic="Maximum",
    Period=60,
    EvaluationPeriods=5,
    Threshold=60000,  # 1 minute
    ComparisonOperator="GreaterThanThreshold",
    AlarmActions=[SNS_TOPIC_ARN],
    Tags=[{"Key": "Project", "Value": PROJECT_NAME}],
)

# Alarm: DLQ has messages (malformed records detected)
cloudwatch.put_metric_alarm(
    AlarmName=f"{PROJECT_NAME}-dlq-depth-{ENV}",
    Namespace="AWS/SQS",
    MetricName="ApproximateNumberOfMessagesVisible",
    Dimensions=[
        {"Name": "QueueName", "Value": f"{PROJECT_NAME}-{DLQ_QUEUE_NAME}-{ENV}"},
    ],
    Statistic="Sum",
    Period=60,
    EvaluationPeriods=1,
    Threshold=0,
    ComparisonOperator="GreaterThanThreshold",
    AlarmActions=[SNS_TOPIC_ARN],
    Tags=[{"Key": "Project", "Value": PROJECT_NAME}],
)

# Alarm: Lambda consumer errors
cloudwatch.put_metric_alarm(
    AlarmName=f"{PROJECT_NAME}-consumer-errors-{ENV}",
    Namespace="AWS/Lambda",
    MetricName="Errors",
    Dimensions=[
        {"Name": "FunctionName", "Value": f"{PROJECT_NAME}-feature-consumer-{ENV}"},
    ],
    Statistic="Sum",
    Period=300,
    EvaluationPeriods=1,
    Threshold=5,
    ComparisonOperator="GreaterThanThreshold",
    AlarmActions=[SNS_TOPIC_ARN],
    Tags=[{"Key": "Project", "Value": PROJECT_NAME}],
)
```

---

## Integration Points

- **Upstream**: `devops/04` → IAM roles for Kinesis, Lambda, Firehose, Flink, SQS, and Feature Store cross-service permissions
- **Upstream**: `devops/02` → VPC configuration for Lambda consumer if accessing OpenSearch or Feature Store via VPC endpoints
- **Downstream**: `mlops/07` → Feature Store online store receives computed features via `put_record()` for real-time inference
- **Downstream**: `data/01` → Glue ETL can backfill batch features from the same source data for consistency between streaming and batch feature sets
- **Downstream**: `devops/10` → OpenTelemetry tracing for Lambda consumer and Flink application to trace feature computation latency end-to-end
- **Downstream**: `data/05` → EventBridge rules can trigger on Kinesis stream scaling events or Flink application state changes
