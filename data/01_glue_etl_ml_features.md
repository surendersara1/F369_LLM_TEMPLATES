<!-- Template Version: 1.0 | boto3: 1.35+ | pyspark: 3.3+ (Glue 4.0) -->

# Template Data 01 — Glue ETL for ML Feature Engineering

## Purpose
Generate a production-ready AWS Glue ETL pipeline for ML feature engineering: PySpark-based transform scripts (numerical encoding, categorical one-hot encoding, timestamp feature extraction, text tokenization), Glue Data Quality rulesets with DQDL, Crawler configurations for automatic schema discovery, job bookmark incremental processing, dual output to Parquet (for SageMaker training) and Feature Store ingestion format, and SNS notification with Step Functions halt on quality failure.

---

## Role Definition

You are an expert AWS data engineer and ML feature engineering specialist with expertise in:
- AWS Glue ETL: PySpark jobs, GlueContext, DynamicFrame, ResolveChoice, job bookmarks
- Glue Data Quality: DQDL rulesets, completeness/uniqueness/freshness/custom SQL checks
- Glue Crawlers: S3 schema discovery, scheduling, schema change policies, Data Catalog integration
- ML feature transforms: numerical scaling/encoding, categorical one-hot encoding, timestamp decomposition, text tokenization/TF-IDF
- Apache Spark: DataFrame operations, UDFs, window functions, partitioning strategies
- Output formats: Parquet with Snappy compression for SageMaker, Feature Store record format
- Incremental processing: job bookmarks, partition-based filtering, change data capture
- Data quality monitoring: SNS alerting, Step Functions integration for pipeline gating
- IAM policies for Glue service roles, S3 access, and cross-service permissions

Generate complete, production-deployable code.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

GLUE_JOB_NAME:          [REQUIRED - name suffix for the Glue ETL job]
                         Example: "feature-engineering"
                         Full name becomes: {PROJECT_NAME}-{GLUE_JOB_NAME}-{ENV}

S3_SOURCE_PATH:          [REQUIRED - S3 path to raw data]
                         Example: "s3://my-data-lake-raw/events/"

S3_OUTPUT_PATH:          [REQUIRED - S3 path for processed features]
                         Example: "s3://my-data-lake-features/output/"

TRANSFORM_TYPES:         [REQUIRED - comma-separated list of transform categories]
                         Options: numerical, categorical, timestamp, text
                         Example: "numerical,categorical,timestamp"
                         - numerical: StandardScaler, MinMaxScaler, log transforms, binning
                         - categorical: one-hot encoding, label encoding, frequency encoding
                         - timestamp: year/month/day/hour/dayofweek extraction, cyclical encoding
                         - text: tokenization, TF-IDF vectorization, word count features

DATA_QUALITY_RULES:      [OPTIONAL: default completeness + uniqueness checks]
                         DQDL rules to apply. Example:
                         "Rules = [
                             Completeness 'user_id' > 0.99,
                             Uniqueness 'transaction_id' > 0.99,
                             ColumnValues 'amount' > 0,
                             Freshness 'event_timestamp' <= 24 hours
                         ]"

CRAWLER_SCHEDULE:        [OPTIONAL: cron(0 6 * * ? *) - daily at 6 AM UTC]
                         Cron expression for crawler schedule.

OUTPUT_FORMAT:           [REQUIRED - parquet | feature_store | both]
                         - parquet: Snappy-compressed Parquet partitioned by date for SageMaker
                         - feature_store: JSON records compatible with mlops/07 Feature Store ingestion
                         - both: generate both output formats

GLUE_VERSION:            [OPTIONAL: 4.0]
                         Glue version. 4.0 = Spark 3.3, Python 3.10.

WORKER_TYPE:             [OPTIONAL: G.1X]
                         Options: G.1X (standard), G.2X (memory-intensive), G.4X (large datasets)

NUM_WORKERS:             [OPTIONAL: 10]
                         Number of Glue workers for the ETL job.

BOOKMARK_ENABLED:        [OPTIONAL: true]
                         Enable job bookmarks for incremental processing.

SNS_TOPIC_ARN:           [OPTIONAL - SNS topic for quality failure notifications]
                         If not provided, a new topic is created.

STEP_FUNCTIONS_ARN:      [OPTIONAL - Step Functions state machine ARN to halt on quality failure]
                         Used to gate downstream ML pipeline execution.
```

---

## Task

Generate complete Glue ETL feature engineering pipeline:

```
{PROJECT_NAME}-glue-features/
├── etl/
│   ├── feature_engineering_job.py     # Main PySpark ETL script with ML transforms
│   ├── transforms/
│   │   ├── numerical.py               # Numerical feature transforms
│   │   ├── categorical.py             # Categorical encoding transforms
│   │   ├── timestamp_features.py      # Timestamp feature extraction
│   │   └── text_features.py           # Text tokenization and vectorization
│   └── utils/
│       ├── spark_helpers.py           # GlueContext, DynamicFrame utilities
│       └── output_writers.py          # Parquet and Feature Store output writers
├── quality/
│   ├── create_ruleset.py              # Create Glue Data Quality ruleset
│   ├── run_quality_check.py           # Run quality evaluation and handle results
│   └── notify_failure.py             # SNS notification + Step Functions halt
├── crawler/
│   ├── create_crawler.py              # Create Glue Crawler for source data
│   └── create_output_crawler.py       # Create Glue Crawler for output features
├── infrastructure/
│   ├── create_glue_job.py             # Create Glue ETL job with bookmarks
│   ├── create_database.py             # Create Glue Data Catalog database
│   └── config.py                      # Central configuration
├── run_setup.py                       # CLI orchestrator
└── requirements.txt
```

**config.py**: Central configuration dataclass with all parameters. Load from environment variables or CLI args. Validate required fields. Construct resource names using `{PROJECT_NAME}-{component}-{ENV}` convention.

**feature_engineering_job.py**: Main PySpark ETL script that runs as a Glue job:
- Initialize `GlueContext` from `SparkContext` with job bookmark enabled (if BOOKMARK_ENABLED)
- Read source data from S3_SOURCE_PATH using `glueContext.create_dynamic_frame.from_options()` with `format="parquet"` or `"json"` (auto-detect)
- Convert `DynamicFrame` to Spark `DataFrame` for complex transforms
- Apply `ResolveChoice` to handle ambiguous column types (cast to most specific type)
- Apply transform modules based on TRANSFORM_TYPES:
  - `numerical.py`: StandardScaler, MinMaxScaler, log1p, quantile binning
  - `categorical.py`: StringIndexer + OneHotEncoder, frequency encoding
  - `timestamp_features.py`: extract year, month, day, hour, day_of_week, is_weekend; cyclical sin/cos encoding for hour and month
  - `text_features.py`: Tokenizer, StopWordsRemover, CountVectorizer, IDF
- Write output based on OUTPUT_FORMAT:
  - Parquet: partition by `year/month/day`, Snappy compression, coalesce to optimal file count
  - Feature Store: convert to JSON records with `feature_group_name`, `record_identifier`, `event_time`
- Commit job bookmark via `job.commit()`

**numerical.py**: Numerical feature transforms using PySpark ML:
- `apply_standard_scaler(df, columns)`: VectorAssembler + StandardScaler pipeline
- `apply_minmax_scaler(df, columns)`: VectorAssembler + MinMaxScaler pipeline
- `apply_log_transform(df, columns)`: log1p transform for skewed distributions
- `apply_quantile_binning(df, column, num_buckets)`: QuantileDiscretizer for binning

**categorical.py**: Categorical encoding transforms:
- `apply_one_hot_encoding(df, columns)`: StringIndexer + OneHotEncoder pipeline
- `apply_label_encoding(df, columns)`: StringIndexer only
- `apply_frequency_encoding(df, columns)`: replace category with its frequency count

**timestamp_features.py**: Timestamp feature extraction:
- `extract_datetime_features(df, timestamp_col)`: year, month, day, hour, minute, day_of_week, day_of_year, is_weekend
- `apply_cyclical_encoding(df, column, period)`: sin/cos encoding for cyclical features (hour of day, month of year)

**text_features.py**: Text feature engineering:
- `apply_tokenization(df, text_col)`: Tokenizer + StopWordsRemover
- `apply_tfidf(df, text_col, num_features)`: CountVectorizer + IDF pipeline
- `apply_word_count(df, text_col)`: simple word count feature

**spark_helpers.py**: Utility functions:
- `init_glue_context(job_name, bookmark_enabled)`: Initialize SparkContext, GlueContext, Job with bookmark option
- `read_dynamic_frame(glue_context, s3_path, format_options)`: Read from S3 with format auto-detection
- `resolve_types(dynamic_frame, specs)`: Apply ResolveChoice for ambiguous types

**output_writers.py**: Output format writers:
- `write_parquet(df, s3_path, partition_cols)`: Write partitioned Parquet with Snappy compression
- `write_feature_store_format(df, s3_path, feature_group_name, record_id_col, event_time_col)`: Write JSON records compatible with SageMaker Feature Store batch ingestion

**create_ruleset.py**: Create Glue Data Quality ruleset using `glue.create_data_quality_ruleset()`:
- Define DQDL rules from DATA_QUALITY_RULES parameter
- Associate ruleset with the Glue Data Catalog table for the source data
- Tag with Project and Environment

**run_quality_check.py**: Run data quality evaluation:
- Start quality evaluation run using `glue.start_data_quality_ruleset_evaluation_run()`
- Poll for completion using `glue.get_data_quality_ruleset_evaluation_run()`
- Parse results: check each rule's pass/fail status
- On failure: invoke `notify_failure.py` to send SNS notification and halt Step Functions

**notify_failure.py**: Quality failure notification:
- Publish SNS message with failed rule details, dataset path, timestamp
- If STEP_FUNCTIONS_ARN provided, send `SendTaskFailure` to halt downstream ML pipeline
- Log failure details to CloudWatch

**create_crawler.py**: Create Glue Crawler for source data:
- Call `glue.create_crawler()` with S3 target path, IAM role, database name, schedule
- Configure `SchemaChangePolicy` with `UpdateBehavior='UPDATE_IN_DATABASE'`
- Configure `RecrawlPolicy` with `RecrawlBehavior='CRAWL_NEW_FOLDERS_ONLY'` for incremental crawling
- Set `TablePrefix` to `{PROJECT_NAME}_{ENV}_`

**create_output_crawler.py**: Create Glue Crawler for output features:
- Same pattern as source crawler but targeting S3_OUTPUT_PATH
- Enables downstream consumers (SageMaker, Athena) to query features via Data Catalog

**create_glue_job.py**: Create Glue ETL job using `glue.create_job()`:
- Set `GlueVersion` to GLUE_VERSION, `WorkerType`, `NumberOfWorkers`
- Enable job bookmarks via `DefaultArguments` with `--job-bookmark-option=job-bookmark-enable`
- Set `--additional-python-modules` for any extra dependencies
- Configure `MaxRetries`, `Timeout`, and `Tags`

**create_database.py**: Create Glue Data Catalog database for source and feature tables using `glue.create_database()`.

**run_setup.py**: CLI orchestrator that runs setup steps in order:
1. Create Glue Data Catalog database
2. Create source and output crawlers
3. Create Glue ETL job
4. Create Data Quality ruleset
5. Run initial crawler to populate catalog
6. Print summary with job name, crawler names, and ruleset name

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**Glue Job:** Use Glue 4.0 (Spark 3.3, Python 3.10) for latest PySpark ML features. Enable job bookmarks for incremental processing — bookmarks track previously processed data so only new records are transformed. Set `MaxRetries` to 1 for transient failures. Set `Timeout` to 2880 minutes (48 hours) as a safety limit.

**Data Quality:** Define DQDL rulesets with at minimum: Completeness checks on key columns (>99%), Uniqueness on identifier columns, ColumnValues range checks on numerical features, and Freshness checks on timestamp columns. Run quality evaluation before writing output — never write bad data downstream.

**Crawlers:** Schedule crawlers to run after ETL job completion, not on a fixed cron (unless data arrives on a known schedule). Use `CRAWL_NEW_FOLDERS_ONLY` recrawl policy for partitioned data to avoid re-crawling historical partitions. Set `SchemaChangePolicy` to `UPDATE_IN_DATABASE` so schema evolution is captured automatically.

**Incremental Processing:** Job bookmarks track the last processed data. For S3 sources, bookmarks work with file-based formats (Parquet, JSON, CSV). For partitioned data, combine bookmarks with partition predicates for efficient reads. Always call `job.commit()` at the end of successful runs.

**Output Formats:** Parquet output should use Snappy compression, partition by date columns (year/month/day), and coalesce to files between 128MB–512MB for optimal SageMaker training reads. Feature Store output must include `record_identifier_feature_name` and `event_time_feature_name` columns matching the Feature Group schema from `mlops/07`.

**Security:** Glue service role needs `s3:GetObject` on source bucket, `s3:PutObject` on output bucket, `glue:*` on catalog resources, and `logs:*` for CloudWatch. Use VPC endpoints for S3 access if data is sensitive. Enable encryption at rest for the Glue Data Catalog and S3 output using KMS keys from `devops/08`.

**Cost:** Use `G.1X` workers for standard transforms, `G.2X` for memory-intensive operations (large one-hot encoding, TF-IDF). Start with 10 workers and enable auto-scaling. Monitor `glue.driver.aggregate.bytesRead` and `glue.driver.aggregate.elapsedTime` CloudWatch metrics to right-size.

**Naming:** All resources follow `{PROJECT_NAME}-{component}-{ENV}` convention:
- Glue job: `{PROJECT_NAME}-{GLUE_JOB_NAME}-{ENV}`
- Crawler: `{PROJECT_NAME}-crawler-{source|output}-{ENV}`
- Database: `{PROJECT_NAME}_features_{ENV}`
- Ruleset: `{PROJECT_NAME}-dq-ruleset-{ENV}`
- SNS topic: `{PROJECT_NAME}-dq-failure-{ENV}`

---

## Code Scaffolding Hints

**Initialize GlueContext with job bookmarks:**
```python
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from awsglue.context import GlueContext
from awsglue.job import Job
from pyspark.context import SparkContext

args = getResolvedOptions(sys.argv, [
    "JOB_NAME", "S3_SOURCE_PATH", "S3_OUTPUT_PATH",
    "TRANSFORM_TYPES", "OUTPUT_FORMAT",
])

sc = SparkContext()
glue_context = GlueContext(sc)
spark = glue_context.spark_session
job = Job(glue_context)
job.init(args["JOB_NAME"], args)
```

**Read source data with DynamicFrame:**
```python
# Read from S3 with job bookmark support
source_dyf = glue_context.create_dynamic_frame.from_options(
    connection_type="s3",
    connection_options={
        "paths": [args["S3_SOURCE_PATH"]],
        "recurse": True,
        "jobBookmarkKeys": ["event_timestamp"],
        "jobBookmarkKeysSortOrder": "asc",
    },
    format="parquet",
    transformation_ctx="source_dyf",  # Required for job bookmarks
)

# Resolve ambiguous types
resolved_dyf = ResolveChoice.apply(
    frame=source_dyf,
    choice="cast:double",  # Cast ambiguous numeric types to double
    transformation_ctx="resolved_dyf",
)

# Convert to Spark DataFrame for ML transforms
df = resolved_dyf.toDF()
```

**Numerical transforms with PySpark ML:**
```python
from pyspark.ml.feature import StandardScaler, MinMaxScaler, VectorAssembler, QuantileDiscretizer
from pyspark.ml import Pipeline
from pyspark.sql.functions import log1p, col

def apply_standard_scaler(df, columns):
    """Apply StandardScaler to numerical columns."""
    assembler = VectorAssembler(inputCols=columns, outputCol="num_features_vec")
    scaler = StandardScaler(
        inputCol="num_features_vec",
        outputCol="scaled_features",
        withStd=True,
        withMean=True,
    )
    pipeline = Pipeline(stages=[assembler, scaler])
    model = pipeline.fit(df)
    return model.transform(df)

def apply_log_transform(df, columns):
    """Apply log1p transform for skewed distributions."""
    for col_name in columns:
        df = df.withColumn(f"{col_name}_log", log1p(col(col_name)))
    return df

def apply_quantile_binning(df, column, num_buckets=10):
    """Bin numerical column into quantile-based buckets."""
    discretizer = QuantileDiscretizer(
        numBuckets=num_buckets,
        inputCol=column,
        outputCol=f"{column}_binned",
    )
    return discretizer.fit(df).transform(df)
```

**Categorical encoding transforms:**
```python
from pyspark.ml.feature import StringIndexer, OneHotEncoder
from pyspark.sql.functions import count, col
from pyspark.ml import Pipeline

def apply_one_hot_encoding(df, columns):
    """Apply one-hot encoding to categorical columns."""
    stages = []
    for col_name in columns:
        indexer = StringIndexer(
            inputCol=col_name,
            outputCol=f"{col_name}_index",
            handleInvalid="keep",
        )
        encoder = OneHotEncoder(
            inputCol=f"{col_name}_index",
            outputCol=f"{col_name}_ohe",
        )
        stages.extend([indexer, encoder])
    pipeline = Pipeline(stages=stages)
    return pipeline.fit(df).transform(df)

def apply_frequency_encoding(df, columns):
    """Replace categorical values with their frequency counts."""
    for col_name in columns:
        freq_df = df.groupBy(col_name).agg(count("*").alias(f"{col_name}_freq"))
        df = df.join(freq_df, on=col_name, how="left")
    return df
```

**Timestamp feature extraction:**
```python
from pyspark.sql.functions import (
    year, month, dayofmonth, hour, minute, dayofweek,
    dayofyear, when, sin, cos, lit, col,
)
import math

def extract_datetime_features(df, timestamp_col):
    """Extract datetime components as ML features."""
    return (
        df.withColumn("feat_year", year(col(timestamp_col)))
        .withColumn("feat_month", month(col(timestamp_col)))
        .withColumn("feat_day", dayofmonth(col(timestamp_col)))
        .withColumn("feat_hour", hour(col(timestamp_col)))
        .withColumn("feat_minute", minute(col(timestamp_col)))
        .withColumn("feat_day_of_week", dayofweek(col(timestamp_col)))
        .withColumn("feat_day_of_year", dayofyear(col(timestamp_col)))
        .withColumn("feat_is_weekend", when(dayofweek(col(timestamp_col)).isin(1, 7), 1).otherwise(0))
    )

def apply_cyclical_encoding(df, column, period):
    """Encode cyclical features using sin/cos (e.g., hour=24, month=12)."""
    return (
        df.withColumn(f"{column}_sin", sin(2 * math.pi * col(column) / lit(period)))
        .withColumn(f"{column}_cos", cos(2 * math.pi * col(column) / lit(period)))
    )
```

**Write Parquet output with partitioning:**
```python
from awsglue.dynamicframe import DynamicFrame

def write_parquet(glue_context, df, s3_path, partition_cols=None):
    """Write partitioned Parquet with Snappy compression."""
    # Coalesce for optimal file sizes (128-512MB)
    df = df.coalesce(max(1, df.rdd.getNumPartitions() // 4))

    output_dyf = DynamicFrame.fromDF(df, glue_context, "output_dyf")

    sink = glue_context.getSink(
        connection_type="s3",
        path=s3_path,
        enableUpdateCatalog=True,
        partitionKeys=partition_cols or ["feat_year", "feat_month", "feat_day"],
    )
    sink.setFormat("glueparquet", compression="snappy")
    sink.setCatalogInfo(
        catalogDatabase=f"{PROJECT_NAME}_features_{ENV}",
        catalogTableName=f"{PROJECT_NAME}_features",
    )
    sink.writeFrame(output_dyf)
```

**Write Feature Store ingestion format:**
```python
import json
from datetime import datetime

def write_feature_store_format(df, s3_path, feature_group_name, record_id_col, event_time_col):
    """Write JSON records compatible with SageMaker Feature Store batch ingestion."""
    # Ensure event_time is ISO 8601 format
    from pyspark.sql.functions import date_format
    df = df.withColumn(
        event_time_col,
        date_format(col(event_time_col), "yyyy-MM-dd'T'HH:mm:ss'Z'"),
    )

    # Write as JSON lines for Feature Store batch ingestion
    df.write.mode("overwrite").json(f"{s3_path}/{feature_group_name}/")
```

**Create Glue Crawler:**
```python
import boto3

glue = boto3.client("glue", region_name=AWS_REGION)

glue.create_crawler(
    Name=f"{PROJECT_NAME}-crawler-source-{ENV}",
    Role=crawler_role_arn,
    DatabaseName=f"{PROJECT_NAME}_features_{ENV}",
    Description=f"Crawl source data for {PROJECT_NAME} feature engineering",
    Targets={
        "S3Targets": [
            {
                "Path": S3_SOURCE_PATH,
                "Exclusions": ["_temporary/**", "_spark_metadata/**"],
            }
        ]
    },
    Schedule=f"cron({CRAWLER_SCHEDULE})" if CRAWLER_SCHEDULE else "",
    SchemaChangePolicy={
        "UpdateBehavior": "UPDATE_IN_DATABASE",
        "DeleteBehavior": "LOG",
    },
    RecrawlPolicy={
        "RecrawlBehavior": "CRAWL_NEW_FOLDERS_ONLY",
    },
    TablePrefix=f"{PROJECT_NAME}_{ENV}_",
    Tags={"Project": PROJECT_NAME, "Environment": ENV},
)
```

**Create Data Quality ruleset:**
```python
glue.create_data_quality_ruleset(
    Name=f"{PROJECT_NAME}-dq-ruleset-{ENV}",
    Description=f"Data quality rules for {PROJECT_NAME} ML features",
    Ruleset=DATA_QUALITY_RULES or """
        Rules = [
            Completeness "user_id" > 0.99,
            Uniqueness "transaction_id" > 0.99,
            ColumnValues "amount" > 0,
            Freshness "event_timestamp" <= 24 hours
        ]
    """,
    TargetTable={
        "TableName": f"{PROJECT_NAME}_{ENV}_source",
        "DatabaseName": f"{PROJECT_NAME}_features_{ENV}",
    },
    Tags={"Project": PROJECT_NAME, "Environment": ENV},
)
```

**Run quality evaluation and handle failure:**
```python
import time

# Start evaluation run
run_response = glue.start_data_quality_ruleset_evaluation_run(
    DataSource={
        "GlueTable": {
            "DatabaseName": f"{PROJECT_NAME}_features_{ENV}",
            "TableName": f"{PROJECT_NAME}_{ENV}_source",
        }
    },
    Role=glue_role_arn,
    RulesetNames=[f"{PROJECT_NAME}-dq-ruleset-{ENV}"],
)
run_id = run_response["RunId"]

# Poll for completion
while True:
    result = glue.get_data_quality_ruleset_evaluation_run(RunId=run_id)
    status = result["Status"]
    if status in ("SUCCEEDED", "FAILED", "STOPPED"):
        break
    time.sleep(30)

# Check results
if status == "SUCCEEDED":
    results = glue.get_data_quality_result(ResultId=result["ResultIds"][0])
    overall_score = results["Score"]
    failed_rules = [
        r for r in results.get("RuleResults", [])
        if r["Result"] == "FAIL"
    ]
    if failed_rules:
        notify_quality_failure(failed_rules, overall_score)
```

**SNS notification on quality failure:**
```python
import json

sns = boto3.client("sns", region_name=AWS_REGION)
sfn = boto3.client("stepfunctions", region_name=AWS_REGION)

def notify_quality_failure(failed_rules, overall_score):
    """Send SNS notification and halt Step Functions on quality failure."""
    message = {
        "source": f"{PROJECT_NAME}-data-quality-{ENV}",
        "status": "FAILED",
        "overall_score": overall_score,
        "failed_rules": [
            {"name": r["Name"], "description": r.get("Description", ""),
             "result": r["Result"], "evaluation_message": r.get("EvaluationMessage", "")}
            for r in failed_rules
        ],
        "timestamp": datetime.utcnow().isoformat(),
    }

    # Send SNS notification
    sns.publish(
        TopicArn=SNS_TOPIC_ARN,
        Subject=f"[{ENV.upper()}] Data Quality Check FAILED — {PROJECT_NAME}",
        Message=json.dumps(message, indent=2),
    )

    # Halt downstream ML pipeline via Step Functions callback
    if STEP_FUNCTIONS_ARN:
        sfn.send_task_failure(
            taskToken=task_token,  # Passed from upstream Step Functions state
            error="DataQualityCheckFailed",
            cause=json.dumps({"failed_rules": len(failed_rules), "score": overall_score}),
        )
```

**Create Glue ETL job with bookmarks:**
```python
glue.create_job(
    Name=f"{PROJECT_NAME}-{GLUE_JOB_NAME}-{ENV}",
    Role=glue_role_arn,
    Command={
        "Name": "glueetl",
        "ScriptLocation": f"s3://{PROJECT_NAME}-scripts-{ENV}/etl/feature_engineering_job.py",
        "PythonVersion": "3",
    },
    DefaultArguments={
        "--job-bookmark-option": "job-bookmark-enable" if BOOKMARK_ENABLED else "job-bookmark-disable",
        "--S3_SOURCE_PATH": S3_SOURCE_PATH,
        "--S3_OUTPUT_PATH": S3_OUTPUT_PATH,
        "--TRANSFORM_TYPES": TRANSFORM_TYPES,
        "--OUTPUT_FORMAT": OUTPUT_FORMAT,
        "--enable-metrics": "true",
        "--enable-continuous-cloudwatch-log": "true",
        "--enable-spark-ui": "true",
        "--spark-event-logs-path": f"s3://{PROJECT_NAME}-logs-{ENV}/spark-ui/",
        "--additional-python-modules": "scikit-learn==1.3.2,nltk==3.8.1",
        "--TempDir": f"s3://{PROJECT_NAME}-temp-{ENV}/glue/",
    },
    GlueVersion=GLUE_VERSION,
    WorkerType=WORKER_TYPE,
    NumberOfWorkers=NUM_WORKERS,
    MaxRetries=1,
    Timeout=2880,
    Tags={"Project": PROJECT_NAME, "Environment": ENV},
)
```

**Job commit for bookmark tracking:**
```python
# At the end of the ETL script, after all transforms and writes
job.commit()
# This records the bookmark state so the next run processes only new data
```

---

## Integration Points

- **Upstream**: `devops/04` → IAM roles for Glue service role (S3 access, Catalog access, CloudWatch logging)
- **Upstream**: `data/03` → Lake Formation permissions for governed access to source data in the Data Catalog
- **Downstream**: `mlops/07` → Feature Store ingestion from the Feature Store format output (JSON records in S3)
- **Downstream**: `mlops/01` → SageMaker training pipeline reads Parquet features from S3_OUTPUT_PATH
- **Downstream**: `data/05` → EventBridge rule triggers on Glue job state change for orchestration
- **Downstream**: `data/02` → Kinesis real-time features can backfill from Glue batch features for consistency
