<!-- Template Version: 1.0 | boto3: 1.35+ | sagemaker-python-sdk: 2.230+ -->

# Template 07 — SageMaker Feature Store

## Purpose
Generate a production-ready SageMaker Feature Store setup with online (low-latency serving) and offline (training dataset generation) stores, feature ingestion pipelines, and real-time feature serving for inference.

---

## Role Definition

You are an expert AWS MLOps engineer and feature engineering specialist with expertise in:
- SageMaker Feature Store: FeatureGroup, Online/Offline Store, Feature Ingestion
- Feature engineering pipelines with SageMaker Processing, Glue, Lambda
- Real-time feature serving for inference endpoints
- Training dataset generation from offline store (Athena queries)
- Feature monitoring and data quality validation

Generate complete, production-deployable code.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

FEATURE_GROUP_NAME:     [REQUIRED - e.g. customer-features]
RECORD_IDENTIFIER:      [REQUIRED - primary key column, e.g. customer_id]
EVENT_TIME_FEATURE:     [REQUIRED - timestamp column, e.g. event_time]

FEATURE_DEFINITIONS:    [REQUIRED - JSON list of features]
    Example:
    [
        {"name": "customer_id", "type": "String"},
        {"name": "event_time", "type": "Fractional"},
        {"name": "total_purchases", "type": "Integral"},
        {"name": "avg_order_value", "type": "Fractional"},
        {"name": "days_since_last_purchase", "type": "Integral"},
        {"name": "customer_segment", "type": "String"}
    ]

ENABLE_ONLINE_STORE:    [OPTIONAL: true]
ENABLE_OFFLINE_STORE:   [OPTIONAL: true]
OFFLINE_STORE_S3:       [OPTIONAL: s3://{PROJECT_NAME}-{AWS_ACCOUNT_ID}-{ENV}-features/offline/]

INGESTION_MODE:         [OPTIONAL: batch | streaming | both]
SOURCE_DATA_S3:         [OPTIONAL - for batch: s3://bucket/path/to/source/data/]
KINESIS_STREAM_NAME:    [OPTIONAL - for streaming: {PROJECT_NAME}-{ENV}-feature-stream]

GLUE_DATABASE:          [OPTIONAL: {PROJECT_NAME}_{ENV}_feature_catalog]
```

---

## Task

Generate complete Feature Store infrastructure:

```
feature_store/
├── setup/
│   ├── create_feature_group.py    # Create FeatureGroup with schema
│   ├── feature_definitions.py     # Feature schema definitions
│   └── glue_catalog.py            # Glue Data Catalog for Athena queries
├── ingestion/
│   ├── batch_ingestion.py         # Batch ingest from S3/DataFrame
│   ├── streaming_ingestion.py     # Kinesis → Lambda → Feature Store
│   ├── processing_job.py          # SageMaker Processing for feature engineering
│   └── lambda_ingester/
│       └── handler.py             # Lambda for real-time feature ingestion
├── serving/
│   ├── online_lookup.py           # Real-time feature lookup (GetRecord)
│   ├── batch_retrieval.py         # Offline store query via Athena
│   └── training_dataset.py        # Generate training dataset from offline store
├── monitoring/
│   └── feature_freshness.py       # Monitor feature staleness and quality
├── config/
│   └── feature_store_config.py
└── run_setup.py
```

**create_feature_group.py**: `FeatureGroup.create()` with online+offline config, KMS encryption, IAM role, S3 offline path, Glue catalog.

**batch_ingestion.py**: Load DataFrame from S3, validate schema, `feature_group.ingest(df, max_workers=5)` with progress tracking.

**streaming_ingestion.py**: Lambda consuming Kinesis records, transforming features, calling `feature_group.put_record()`.

**online_lookup.py**: `featurestore_runtime.get_record(FeatureGroupName, RecordIdentifierValueAsString)` with caching layer.

**training_dataset.py**: Athena query on offline store to generate point-in-time correct training datasets.

---

## Output Format

Output ALL files with headers: `### FILE: [path/filename.py]`

---

## Requirements & Constraints

**Schema:** All features must have explicit types (String, Integral, Fractional). Record identifier and event time are mandatory. Max 2500 features per group.

**Online Store:** Single-digit ms latency. Use for inference-time feature lookup. Records auto-expire after 24 months.

**Offline Store:** Append-only in S3 (Parquet). Queryable via Athena through Glue catalog. Use for training dataset generation with point-in-time correctness.

**Ingestion:** Batch: max 500 records per `put_record` batch. Streaming: Lambda with Kinesis trigger, concurrency 10 for dev, 100 for prod.

---

## Code Scaffolding Hints

```python
from sagemaker.feature_store.feature_group import FeatureGroup
feature_group = FeatureGroup(name=FEATURE_GROUP_NAME, sagemaker_session=session)
feature_group.load_feature_definitions(data_frame=df)
feature_group.create(
    s3_uri=OFFLINE_STORE_S3, record_identifier_name=RECORD_IDENTIFIER,
    event_time_feature_name=EVENT_TIME_FEATURE, role_arn=role_arn,
    enable_online_store=True, online_store_kms_key_id=kms_key_id
)
```

---

## Integration Points

- **Upstream**: `devops/04` → IAM roles for Feature Store access
- **Downstream**: `mlops/01` → training pipeline reads from offline store
- **Downstream**: `mlops/03` → inference endpoint reads from online store
