<!-- Template Version: 2.0 | F369 Wave 8 (composite) | Composes: DATA_DMS_REPLICATION + DATA_EVENTBRIDGE_PIPES + DATA_APPFLOW_SAAS_INGEST + DATA_ICEBERG_S3_TABLES + DATA_GLUE_CATALOG -->

# Template 06 — Operational DB → Lakehouse Landing (DMS · EventBridge Pipes · AppFlow · Iceberg)

## Purpose

Build a complete data-ingestion pipeline that lands operational data (RDBMS, NoSQL, SaaS) into the lakehouse curated zone as governed Iceberg tables. Cover ALL three ingestion paths in one template:

- **Path 1 — Database CDC** (Oracle / SQL Server / on-prem Postgres / on-prem MySQL): DMS classic replication → S3 Parquet → Glue ETL MERGE → Iceberg
- **Path 2 — Operational stream** (DDB / Kinesis): EventBridge Pipes → Firehose → S3 → Iceberg via Athena CTAS
- **Path 3 — SaaS ingest** (Salesforce / Slack / ServiceNow / 60+ sources): AppFlow → S3 → Glue Crawler → Iceberg

Generates production-deployable CDK + supporting Lambda code, no placeholders.

---

## Role Definition

You are an expert AWS data engineer with deep expertise in:
- AWS DMS (homogeneous + heterogeneous + S3 target endpoints + CDC start positions)
- EventBridge Pipes (DDB Streams / Kinesis sources, filter expressions, Lambda enrichment)
- AppFlow (Salesforce / Slack / ServiceNow + 60+ SaaS connectors, OAuth + PrivateLink)
- Apache Iceberg on S3 (S3 Tables managed + self-managed via Glue Catalog)
- Glue ETL (Spark MERGE-INTO patterns for CDC replay)
- Athena CTAS + Iceberg DML
- KMS-per-zone encryption + Lake Formation governance

Generate complete, production-deployable code. No TODOs, no placeholder comments.

---

## Context and Inputs

```
PROJECT_NAME:           [REQUIRED — e.g. acme-lakehouse]
AWS_REGION:             [REQUIRED — e.g. us-east-1]
AWS_ACCOUNT_ID:         [REQUIRED — 12-digit]
ENV:                    [REQUIRED — dev | stage | prod]

# --- INGESTION PATHS (pick 1+) ---
INGEST_PATH_1_DB_CDC:        [OPTIONAL — true | false; default false]
INGEST_PATH_2_DDB_STREAM:    [OPTIONAL — true | false; default false]
INGEST_PATH_3_SAAS_APPFLOW:  [OPTIONAL — true | false; default false]

# --- PATH 1 (DB CDC via DMS) ---
DMS_SOURCE_ENGINE:           [oracle | sqlserver | postgres | mysql | mongodb]
DMS_SOURCE_HOST:             [hostname or VPC endpoint]
DMS_SOURCE_PORT:             [5432 | 1521 | 1433 | 3306 | 27017]
DMS_SOURCE_DB:               [database name]
DMS_REPLICATION_INSTANCE_CLASS: [OPTIONAL — dms.r5.large default; dms.r5.xlarge prod]
DMS_HOMOGENEOUS_PATH:        [OPTIONAL — true if source+target same engine; uses DMS Serverless]
DMS_TABLES_INCLUDE:          [REQUIRED — list of tables/schemas; e.g. "ORDERS.%" "CUSTOMERS.%"]
DMS_PII_COLUMNS_REMOVE:      [OPTIONAL — list of columns to drop in transformation rules]

# --- PATH 2 (DDB / Kinesis Stream via EventBridge Pipes) ---
PIPE_SOURCE_TYPE:            [ddb_stream | kinesis_stream | msk_topic]
PIPE_SOURCE_ARN:             [ARN of source stream]
PIPE_FILTER_PATTERN:         [JSON pattern; default {} = pass-all]
PIPE_ENRICHMENT_LAMBDA:      [OPTIONAL — ARN if you want enrichment]
PIPE_BATCH_SIZE:             [OPTIONAL — 100 default for DDB, 500 for Kinesis]

# --- PATH 3 (SaaS via AppFlow) ---
APPFLOW_CONNECTOR:           [salesforce | slack | servicenow | marketo | datadog | snowflake]
APPFLOW_SOURCE_OBJECT:       [e.g. Account, Lead, Channel, Incident]
APPFLOW_SCHEDULE_EXPRESSION: [OPTIONAL — rate(15 minutes) default]
APPFLOW_INCREMENTAL_FIELD:   [OPTIONAL — LastModifiedDate, updated_at, etc.]
APPFLOW_PRIVATELINK:         [OPTIONAL — true if Salesforce Shield + PL]

# --- LAKEHOUSE TARGET ---
RAW_BUCKET_NAME:             [e.g. {project}-raw-{env}; auto-generated if blank]
CURATED_BUCKET_NAME:         [e.g. {project}-curated-{env}]
ICEBERG_DATABASE:            [Glue database for Iceberg tables; e.g. lakehouse_curated]
ICEBERG_TABLE_FORMAT:        [OPTIONAL — s3_tables (managed) | self_managed; default s3_tables]
PARTITION_STRATEGY:          [OPTIONAL — date | tenant_date | none; default date]

# --- COMPLIANCE / SECURITY ---
KMS_KEY_ARN:                 [REQUIRED — per-zone CMK]
LAKE_FORMATION_ENABLED:      [OPTIONAL — true; default true for prod]
LF_TBAC_TAGS:                [OPTIONAL — domain/sensitivity/access_tier]
```

---

## Partial Library (Claude MUST load these into context)

For Claude to generate correct code, load these F369 canonical partials at session start:

| Partial | Why |
|---|---|
| `DATA_DMS_REPLICATION` | Source-of-truth for DMS Serverless + classic + S3 target endpoint settings |
| `DATA_EVENTBRIDGE_PIPES` | Pipe service-role IAM patterns + filter syntax + Firehose target |
| `DATA_APPFLOW_SAAS_INGEST` | Connector profile creation + flow tasks + EventBridge integration |
| `DATA_ICEBERG_S3_TABLES` | S3 Tables managed Iceberg patterns |
| `DATA_LAKEHOUSE_ICEBERG` | Self-managed Iceberg + MERGE replay pattern |
| `DATA_GLUE_CATALOG` | Glue database + table + crawler config |
| `DATA_LAKE_FORMATION` | LF-TBAC tag application to ingested tables |
| `LAYER_BACKEND_LAMBDA` | 5 non-negotiables (asset paths, cross-stack patterns) |
| `LAYER_NETWORKING` | VPC + PrivateLink for DMS source connectivity |
| `SECURITY_DATALAKE_CHECKLIST` | KMS-per-zone + Object Lock + LF strict mode |

---

## Architecture (composite — generate diagram per active paths)

```
   Sources                              Ingest layer                       Lakehouse
   ───────                              ────────────                       ─────────
   On-prem Oracle 12c ─┐                ┌─► DMS classic ──┐
   On-prem SQL Server ─┤                │   replication   │
   On-prem Postgres ───┼──► PrivateLink ┤                 │
   Aurora MySQL CDC ───┤                │   Path 1 (CDC)  │
                                        │                 │     ┌── S3 raw zone ───────┐
   DynamoDB Streams ───┐                                  ├──► │  s3://raw/dms/...     │
   Kinesis stream ─────┼──► EventBridge ─► Firehose       │     │  s3://raw/ddb/...     │
                       │   Pipes (filter   (Parquet,      ├──► │  s3://raw/saas/...    │
                       │   + enrich)       partition)     │     │  KMS-encrypted        │
                       │                                   │     └──┬───────────────────┘
                       │   Path 2 (stream)                │        │
                                                          │        │  Glue Crawler (hourly)
   Salesforce Account ─┐                                  │        │  → Glue Catalog tables
   Slack messages ─────┼──► AppFlow flow                  │        │
   ServiceNow tickets ─┘   (Filter + Mask                 │        ▼
                            + Validate)                   │     ┌── Glue ETL job ───────┐
                                                          │     │  Iceberg MERGE-INTO   │
                            Path 3 (SaaS)                 │     │  scheduled hourly     │
                                                          │     │  via Step Functions   │
                                                          │     └──┬───────────────────┘
                                                          │        │
                                                          ▼        ▼
                                                          ┌── S3 curated zone (Iceberg) ─┐
                                                          │  s3://curated/orders/         │
                                                          │  s3://curated/customers/      │
                                                          │  Lake Formation governed      │
                                                          │  Athena + Redshift + EMR ✓   │
                                                          └───────────────────────────────┘
```

---

## Day-by-day execution (5-day POC, 1 dev)

### Day 1 — Foundation
- Provision VPC + PrivateLink endpoints (DMS, Bedrock, S3, KMS, Glue)
- 3 KMS CMKs (raw, curated, audit zones)
- S3 buckets (raw, curated, audit) with KMS + Block Public + lifecycle
- Lake Formation strict mode + LF-Tags
- Glue database `lakehouse_raw` + `lakehouse_curated`
- **Deliverable:** all infra synth-clean; LF strict mode active

### Day 2 — Path 1 (DMS) — only if `INGEST_PATH_1_DB_CDC=true`
- DMS replication subnet group + replication instance (or Serverless data provider for homogeneous)
- Source endpoint (with Secrets Manager creds, KMS, PrivateLink)
- S3 target endpoint (Parquet output, partition by date, KMS-encrypted)
- Replication task with table mappings + transformation rules + validation
- **Deliverable:** sample full-load + CDC running; `LOAD0001.parquet` files appearing in `s3://raw/dms/`

### Day 3 — Path 2 (Pipes) — only if `INGEST_PATH_2_DDB_STREAM=true`
- DDB table with Stream enabled (or reference existing stream ARN)
- Pipe service IAM role
- Pipe with filter + (optional) enrichment Lambda + Firehose target
- Firehose with Parquet conversion + dynamic partitioning + KMS
- **Deliverable:** sample DDB write triggers Pipe → Firehose → S3 within 60 seconds

### Day 4 — Path 3 (AppFlow) — only if `INGEST_PATH_3_SAAS_APPFLOW=true`
- AppFlow connector profile (OAuth or static creds via Secrets Manager)
- Flow with filter + mask + validate + Map_all tasks
- Schedule trigger + S3 destination with date partitioning
- **Deliverable:** scheduled flow runs every 15 min; new records land in `s3://raw/saas/`

### Day 5 — Curated layer + observability
- Glue Crawlers (per source path) → catalog tables
- Iceberg target tables (S3 Tables managed OR Glue Catalog self-managed)
- Glue ETL job: MERGE-INTO from raw CDC → Iceberg curated, scheduled hourly via Step Functions
- LF-TBAC tags on curated tables
- CloudWatch dashboards: ingest latency, DMS CDCLatencySource, Pipes throughput, AppFlow records-processed
- **Deliverable:** end-to-end query in Athena: `SELECT * FROM lakehouse_curated.orders WHERE updated_at > NOW() - INTERVAL '1' HOUR LIMIT 100;` returns results

---

## Validation criteria

After Day 5, all of the following must pass:

- [ ] **Latency:** raw → curated within 60 min of source change (per active path)
- [ ] **Lineage:** every Iceberg table row traces back to source via `source_system` + `_dms_op` columns
- [ ] **PII:** flagged columns dropped at ingest (DMS transformation rules / AppFlow Mask tasks)
- [ ] **LF governance:** Athena query succeeds for `analyst-role`, fails for `unauthorized-role`
- [ ] **Cost:** ingestion < $50/day for the POC volume (sample 10K records/day)
- [ ] **Failure modes:** simulated source disconnect → CDC resumes from last checkpoint without data loss

---

## Phase-2 expansions (catalog separately)

Items NOT in this template but in adjacent ones:
- Real-time streaming (sub-second SLA) → Kinesis Data Streams + Flink (separate template)
- Cross-region DR for the lakehouse → `data/08_resilient_db_dr` template
- Multi-DB federation queries → `data/07_multi_db_federation_query` template
- Heavy compaction / Hudi / Delta → `data/09_emr_serverless_spark_iceberg` template
- Full data lake security audit → `enterprise/07_datalake_security_baseline` template

---

## Output artifacts

Generate:
1. CDK Python app (10-13 stacks) — synth-clean
2. DMS table mappings JSON (per source schema) — Path 1 only
3. Pipe filter expressions + enrichment Lambda code — Path 2 only
4. AppFlow flow tasks JSON — Path 3 only
5. Glue ETL Spark MERGE script (PySpark)
6. Athena CTAS for initial Iceberg table provisioning
7. Step Functions workflow JSON for hourly MERGE
8. CloudWatch dashboard JSON
9. UAT script with sample queries
10. Runbook (incident response, common ops)

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 2.0 | 2026-04-26 | Initial composite template — 3 ingestion paths in one. Composes 10 partials. Wave 8. |
