<!-- Template Version: 2.0 | F369 Wave 8 (composite) | Composes: DATA_EMR_SERVERLESS_SPARK + DATA_ICEBERG_S3_TABLES + DATA_LAKEHOUSE_ICEBERG + DATA_LAKE_FORMATION + DATA_GLUE_CATALOG -->

# Template 09 — EMR Serverless Spark on Iceberg (heavy transforms · compaction · ML feature engineering)

## Purpose

Build heavy-transform Spark workloads on the lakehouse where Athena's compute model is insufficient: complex UDFs, ML feature engineering, scheduled Iceberg compaction, MERGE-INTO at scale, and multi-hop ETL. Uses EMR Serverless 7.12+ with Iceberg 1.10 + Glue Catalog metastore + Lake Formation enforcement.

Generates production-deployable CDK + Spark job scripts + Step Functions orchestration.

---

## Role Definition

You are an expert AWS data engineer + Spark practitioner with deep expertise in:
- EMR Serverless 7.12 (release label, ARM64 architecture, pre-init capacity, auto-stop)
- Apache Spark 3.5 + Iceberg 1.10 + Hudi 0.15 + Delta 3.x
- Glue Data Catalog as Iceberg metastore (universal across Athena, Redshift Spectrum, EMR)
- Lake Formation enforcement on Spark jobs (`AWS_LAKEFORMATION_ENABLED`)
- Iceberg maintenance (`rewrite_data_files`, `expire_snapshots`, `remove_orphan_files`)
- Spark ML feature engineering pipelines

Generate complete production-deployable code, no TODOs.

---

## Context and Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED — dev | stage | prod]

# --- WORKLOAD TYPE (pick 1+) ---
WORKLOAD_COMPACTION:    [OPTIONAL — true: nightly Iceberg compaction]
WORKLOAD_MERGE_CDC:     [OPTIONAL — true: hourly MERGE-INTO from raw CDC]
WORKLOAD_FEATURE_ENG:   [OPTIONAL — true: ML feature engineering pipeline]
WORKLOAD_CUSTOM_UDF:    [OPTIONAL — true: bring-your-own Spark job]

# --- TABLE FORMAT ---
TABLE_FORMAT:           [REQUIRED — iceberg (default) | hudi | delta]
TABLES_TO_COMPACT:      [comma list — e.g. lakehouse_curated.orders, lakehouse_curated.customers]

# --- COMPUTE ---
EMR_RELEASE_LABEL:      [OPTIONAL — emr-7.12.0 default (latest LTS)]
ARCHITECTURE:           [OPTIONAL — ARM64 default (20% cheaper) | X86_64]
INIT_CAPACITY_DRIVERS:  [OPTIONAL — 2 default]
INIT_CAPACITY_EXEC:     [OPTIONAL — 4 default]
MAX_CPU:                [OPTIONAL — 200 default]
AUTO_STOP_MIN:          [OPTIONAL — 60 default]

# --- ORCHESTRATION ---
SCHEDULE_EXPRESSION:    [REQUIRED for compaction — cron(0 2 * * ? *) nightly default]
USE_STEP_FUNCTIONS:     [REQUIRED — true for production]

# --- COMPLIANCE ---
KMS_KEY_ARN:            [REQUIRED]
LAKE_FORMATION_ENABLED: [REQUIRED — true for prod]
```

---

## Partial Library

| Partial | Why |
|---|---|
| `DATA_EMR_SERVERLESS_SPARK` | Application config, IAM, Iceberg config, runtime overrides |
| `DATA_ICEBERG_S3_TABLES` | S3 Tables managed Iceberg (preferred for new builds) |
| `DATA_LAKEHOUSE_ICEBERG` | Self-managed Iceberg + MERGE patterns |
| `DATA_LAKE_FORMATION` | LF enforcement on EMR jobs |
| `DATA_GLUE_CATALOG` | Iceberg metastore |
| `LAYER_OBSERVABILITY` | CloudWatch metrics + alarms for job duration / cost |

---

## Architecture

```
   Step Functions State Machine: lakehouse-maintenance
   ─────────────────────────────────────────────────────
        │
        ▼ (scheduled nightly via EventBridge cron)
   ┌──────────────────────────────────────────────────────────────┐
   │  EMR Serverless Application: spark-iceberg-prod              │
   │     - Release: emr-7.12.0                                     │
   │     - Architecture: ARM64                                     │
   │     - Pre-init: 2 driver + 4 executor (warm pool)             │
   │     - Auto-stop: 60 min idle                                  │
   │     - Glue Catalog as Iceberg metastore                       │
   │     - LF enforcement enabled                                  │
   └──────────────────────────────────────────────────────────────┘
        │
        ├──► Job 1: iceberg_compact.py (nightly)
        │      For each table: rewrite_data_files + expire_snapshots + remove_orphan
        │
        ├──► Job 2: cdc_merge.py (hourly)
        │      Read s3://raw/cdc/<table>/<date>/*.parquet
        │      MERGE INTO lakehouse_curated.<table>
        │
        ├──► Job 3: feature_engineering.py (daily)
        │      Read curated → join + window functions + UDFs → write feature_store
        │
        └──► (optional) Custom UDF jobs

   ┌──────────────────────────────────────────────────────────────┐
   │  Outputs                                                      │
   │     - Logs: s3://emr-logs/spark-logs/<job-id>/                │
   │     - Job metrics: CloudWatch + Spark History Server           │
   │     - Iceberg tables: 5x faster reads after nightly compact    │
   └──────────────────────────────────────────────────────────────┘
```

---

## Day-by-day execution (3-day POC)

### Day 1 — EMR Serverless application + IAM
- EMR Serverless application (Spark, emr-7.12.0, ARM64, VPC-config, IAM execution role)
- Glue Catalog read+write IAM
- Lake Formation `GetDataAccess` IAM
- S3 logs bucket (90-day lifecycle)
- KMS encryption end-to-end
- **Deliverable:** application status `STARTED`; sample `pyspark` job runs via `aws emr-serverless start-job-run`

### Day 2 — Iceberg compaction + MERGE-INTO scripts
- `scripts/iceberg_compact.py` — calls `rewrite_data_files` + `rewrite_manifests` + `expire_snapshots` + `remove_orphan_files`
- `scripts/cdc_merge.py` — Athena MERGE replay from raw CDC into curated Iceberg
- Step Functions state machine that takes table name list, fans out one job-run per table
- EventBridge cron schedule (nightly 02:00 UTC compaction; hourly :05 CDC merge)
- **Deliverable:** compaction reduces small-file count by 90%+; MERGE replay produces Iceberg snapshot

### Day 3 — Feature engineering pipeline (optional, if `WORKLOAD_FEATURE_ENG=true`)
- `scripts/feature_engineering.py` — reads curated tables, computes features (window functions, custom UDFs), writes to `feature_store_offline` Iceberg table
- SageMaker Feature Store Iceberg-mode integration (writes auto-register in feature group)
- CloudWatch dashboard: job runtime, cost-per-run, executor utilization, S3 IO
- **Deliverable:** end-to-end pipeline runs; features queryable from Athena

---

## Spark job patterns (Claude generates these)

### Iceberg compaction
```python
spark.sql(f"""
  CALL glue.system.rewrite_data_files(
    table => '{db}.{tbl}',
    options => map(
      'target-file-size-bytes', '268435456',
      'min-input-files', '5'
    )
  )
""")
spark.sql(f"CALL glue.system.expire_snapshots(table => '{db}.{tbl}', older_than => TIMESTAMP '{cutoff}')")
spark.sql(f"CALL glue.system.remove_orphan_files(table => '{db}.{tbl}')")
```

### CDC MERGE-INTO replay
```python
spark.sql(f"""
  MERGE INTO glue.{db}.{tbl} AS target
  USING (SELECT * FROM glue.raw_cdc.{tbl} WHERE _dms_timestamp > '{watermark}') AS src
  ON target.id = src.id
  WHEN MATCHED AND src._dms_op = 'D' THEN DELETE
  WHEN MATCHED AND src._dms_op = 'U' THEN UPDATE SET *
  WHEN NOT MATCHED AND src._dms_op = 'I' THEN INSERT *
""")
```

---

## Validation criteria

- [ ] Compaction job: small-file count reduced 90%+ vs pre-run
- [ ] MERGE replay: row counts match source DB at watermark
- [ ] LF governance: unauthorized role's Spark job fails with permission error
- [ ] Cost: nightly compaction < $5/table for 100 GB-class tables
- [ ] Recovery: failed job auto-retries via Step Functions; alarm fires after 3 retries
- [ ] Spark History Server accessible via `aws emr-serverless get-dashboard-for-job-run`

---

## Output artifacts

1. CDK app (EmrServerlessStack)
2. Spark scripts: `iceberg_compact.py`, `cdc_merge.py`, `feature_engineering.py`
3. Step Functions definition (parallel fan-out per table)
4. EventBridge cron schedule
5. CloudWatch dashboard JSON
6. Cost projection spreadsheet (per-table $/run estimates)
7. Spark UI access guide

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 2.0 | 2026-04-26 | Initial composite. Composes EMR Serverless + Iceberg + LF. Wave 8. |
