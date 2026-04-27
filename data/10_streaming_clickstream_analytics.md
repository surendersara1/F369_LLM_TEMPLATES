<!-- Template Version: 2.0 | F369 Wave 12 (composite) | Composes: DATA_KINESIS_STREAMS_FIREHOSE + DATA_OPENSEARCH_SERVERLESS + DATA_QUICKSIGHT_REALTIME + DATA_GLUE_CATALOG + DATA_ATHENA -->

# Template 10 — Streaming Clickstream Analytics (Kinesis · Firehose · OpenSearch + Athena · QuickSight)

## Purpose

Stand up a **production-grade clickstream / event analytics pipeline in 3-4 days** — from web/mobile/IoT producer events to live operational dashboards (OpenSearch) + cost-effective historical analytics (S3 Iceberg + Athena) + executive BI (QuickSight).

Use cases this template fits:
- Web/mobile clickstream → real-time dashboards + retention cohorts
- App telemetry / log analytics (replace Splunk/Elastic stack)
- IoT device events → real-time alerts + historical trends
- Marketing event tracking (page views, conversions, attribution)

Generates production-deployable CDK + producer SDK snippet + dashboards.

---

## Role Definition

You are an expert AWS streaming architect with deep expertise in:
- Kinesis Data Streams (on-demand auto-scale + provisioned + Enhanced Fan-Out)
- Kinesis Data Firehose (Lambda transform, dynamic partitioning, Parquet conversion, dual S3+OS sinks)
- OpenSearch Serverless TIMESERIES collections + ISM lifecycle + IAM SigV4 auth
- Glue Catalog + Athena workgroups + partition projection
- QuickSight datasets (SPICE) + Q topics + embedded dashboards
- Producer SDKs (KPL/KCL Java, boto3 Python, AWS SDK JS)

Generate complete, production-deployable code. No TODOs.

---

## Context and Inputs

```
PROJECT_NAME:                [REQUIRED]
AWS_REGION:                  [REQUIRED]
AWS_ACCOUNT_ID:              [REQUIRED]
ENV:                         [REQUIRED — dev | stage | prod]

# --- EVENT SHAPE ---
EVENT_TYPES:                 [REQUIRED — comma-separated; e.g. page_view,click,purchase,signup]
EVENT_SCHEMA:                [JSON example with required fields]
PEAK_EVENTS_PER_SECOND:      [REQUIRED — drives KDS provisioning]
DAILY_EVENT_VOLUME_GB:       [REQUIRED — drives Firehose + S3 sizing]

# --- INGEST ---
PRODUCER_TYPE:               [REQUIRED — sdk_browser | mobile | server_lambda | iot_core | api_gateway]
PRODUCER_AUTH:               [cognito_jwt | iam | api_key]

# --- ANALYTICS DESTINATIONS ---
ENABLE_OPENSEARCH:           [true default — for real-time dashboards]
ENABLE_S3_ICEBERG:           [true default — historical queries via Athena]
ENABLE_QUICKSIGHT:           [true default — executive dashboards]
OS_RETENTION_DAYS:           [30 default; rolls to S3 after]
S3_RETENTION_YEARS:          [7 default]

# --- COMPLIANCE ---
KMS_KEY_ARN:                 [REQUIRED]
PII_FIELDS:                  [comma-separated event fields containing PII; e.g. email,user_id]
PII_HANDLING:                [hash default — also: redact, tokenize, drop]

# --- OBSERVABILITY ---
SNS_ALARM_TOPIC_ARN:         [REQUIRED]
DASHBOARD_USER_ARNS:         [comma-separated IAM ARNs allowed to view OS Dashboards + QS]
```

---

## Partial Library (Claude MUST load)

| Partial | Why |
|---|---|
| `DATA_KINESIS_STREAMS_FIREHOSE` | KDS + Firehose + Lambda transform + dynamic partitioning + dual sink |
| `DATA_OPENSEARCH_SERVERLESS` | TIMESERIES collection + ISM + Firehose sink |
| `DATA_QUICKSIGHT_REALTIME` | SPICE datasets + Q topics + dashboards |
| `DATA_GLUE_CATALOG` | Glue table for events (Parquet) + partition projection |
| `DATA_ATHENA` | Workgroup + scan limits + result bucket KMS |
| `LAYER_BACKEND_LAMBDA` | Lambda transform 5 non-negotiables |
| `LAYER_OBSERVABILITY` | CW dashboards + alarms |

---

## Architecture

```
   ┌──────────────────────────────────────────────────────────────────┐
   │ Producers                                                        │
   │   Browser SDK (analytics.js with Cognito JWT)                     │
   │   Mobile SDK (Amplify with Cognito JWT)                           │
   │   Server Lambda (boto3 PutRecords)                                │
   │   IoT devices (IoT Core → IoT Rules → KDS)                        │
   └────────────────────────────────┬─────────────────────────────────┘
                                    │
                                    ▼
                   ┌─────────────────────────────────┐
                   │ API Gateway HTTP API (optional)  │
                   │   - Cognito JWT                  │
                   │   - WAF (rate limit, bot block)  │
                   │   - Lambda authorizer for sample │
                   └────────────────┬─────────────────┘
                                    │
                                    ▼
                   ┌─────────────────────────────────┐
                   │ Kinesis Data Stream (on-demand)  │
                   │   - KMS encryption                │
                   │   - 24h retention (extend to 7d)  │
                   └────────────────┬─────────────────┘
                                    │
                  ┌─────────────────┼─────────────────┐
                  ▼                                   ▼
       ┌─────────────────┐                ┌──────────────────────┐
       │ Firehose #1     │                │ Firehose #2          │
       │ → S3 Parquet    │                │ → OpenSearch (live)  │
       │   (historical)  │                │                      │
       │   Lambda xform: │                │   Lambda xform:      │
       │    PII hashing, │                │    PII hashing,      │
       │    enrichment   │                │    add geoip/ua      │
       │   Dyn partition:│                │   Index per-day      │
       │    yyyy/mm/dd/  │                │    events-2026-04-26 │
       │    event_type   │                │   ISM: 30d hot → del │
       └────────┬────────┘                └──────────┬───────────┘
                │                                     │
                ▼                                     ▼
       ┌─────────────────┐                ┌──────────────────────┐
       │ S3 Iceberg      │                │ OpenSearch Dashboards │
       │ + Glue Catalog  │                │ - Live last-hour view │
       │ + Athena        │                │ - Anomaly detection   │
       │   workgroup     │                │ - Per-event-type      │
       │   scan limit 1GB│                │   panels              │
       └────────┬────────┘                └──────────────────────┘
                │
                ▼
       ┌─────────────────┐
       │ QuickSight      │
       │ - SPICE dataset │
       │   refresh hourly│
       │ - Q topic (NL → │
       │   SQL via       │
       │   Bedrock)      │
       │ - Embedded in   │
       │   product UI    │
       └─────────────────┘
```

---

## Day-by-day execution (3-4 day POC, 1 dev)

### Day 1 — Kinesis + Firehose → S3 Iceberg
- KMS CMK setup (or import)
- Kinesis Data Stream (on-demand, 24h retention) + producer IAM role
- S3 raw bucket (KMS, partitioned lifecycle)
- Glue Catalog database + table (Parquet schema for `EVENT_TYPES`)
- Lambda transform function — parse JSON, validate, hash `PII_FIELDS`, add `ingested_at`
- Firehose #1 → S3 with Parquet conversion + dynamic partitioning + Lambda transform
- Athena workgroup with 1 GB scan cutoff
- Producer SDK / sample script — generate test events
- **Deliverable:** End of Day 1: 100 test events → KDS → Firehose → S3 Parquet within 90s; Athena query returns rows.

### Day 2 — OpenSearch Serverless + Firehose #2 + Dashboards
- OpenSearch Serverless TIMESERIES collection (CMK encrypted, public for POC, VPC for prod)
- Encryption + Network + Data Access policies
- Index template for events (auto-rolling daily indices: `events-YYYY-MM-DD`)
- ISM policy: hot 7d → warm 23d → delete 30d
- Firehose #2 → OS sink (Lambda transform same as #1 OR shared)
- OS Dashboards: 4 default panels — events/min, top event types, geo map, p95 latency
- **Deliverable:** End of Day 2: dashboards show live events; lag from event time to dashboard ≤ 90s.

### Day 3 — QuickSight + Q topic + executive dashboard
- QuickSight enabled (console — pre-req)
- Athena data source registered
- SPICE dataset with custom SQL (last 30d events) + hourly incremental refresh
- Analysis with: KPI cards (events today, conversion rate, MAU), time-series of events, geo map, funnel
- Publish as dashboard
- Q topic on dataset with column synonyms (`amount` = `revenue` = `$`)
- (Optional) embed analytics Lambda for SaaS scenario — `GenerateEmbedUrlForRegisteredUser`
- **Deliverable:** End of Day 3: exec dashboard live in QuickSight; Q topic answers "what was revenue last week?"

### Day 4 — Hardening + observability + tests
- 6 CloudWatch alarms wired to SNS:
  - KDS `IncomingRecords` drops 50% (ingest broken)
  - Firehose #1 `DeliveryToS3.Records` drops 50%
  - Firehose #2 `DeliveryToOpenSearch.Records` drops 50%
  - Lambda transform errors > 1%
  - OS collection storage > 80% of OCU
  - QuickSight dataset refresh failure
- Pytest suite: smoke, end-to-end, PII handling, dashboard URL accessible
- README with producer SDK code samples (JS, Python, Swift)
- **Deliverable:** Pass all tests + alarm matrix tested by inducing 6 failure scenarios.

---

## Validation criteria

- [ ] **Producer can publish event** via `aws kinesis put-record --stream-name $S --data ... --partition-key user-1`
- [ ] **Event reaches S3 Parquet within 90s** of put
- [ ] **Event reaches OpenSearch Dashboards within 90s** of put
- [ ] **Athena query** `SELECT COUNT(*) FROM events WHERE day=$today` returns ≥ test event count
- [ ] **PII fields hashed/redacted** in S3 + OS (verified via SELECT/_search)
- [ ] **OS ISM rolls indices** — `events-{30+ days ago}` is gone
- [ ] **QS SPICE dataset shows event data** within 1h of latest events
- [ ] **Q topic** answers a sample NL query with reasonable SQL
- [ ] **All 6 alarms in OK state** initially; firing on injected failures
- [ ] **CMK encryption** on KDS + S3 + OS (verified via describe APIs)

---

## Common gotchas (claude must address proactively)

- **PII hashing in Lambda transform must be deterministic** (HMAC-SHA256 with KMS-derived key) so same user_id → same hash for cohort analysis. Random hash = analytical chaos.
- **Glue table schema must match Parquet output** exactly. Adding a producer field = pipeline error until Glue updated. Use Glue Schema Registry for evolution.
- **OS Serverless has 2 OCU minimum** = ~$350/mo per collection. Share collections across event_types if budget tight (one TIMESERIES collection, multiple indices).
- **Firehose buffer interval 60-900s** — minimum end-to-end lag = 60s. Real-time means "near real-time" here.
- **KDS on-demand has 5-min scale-up latency** — first burst may throttle. For known traffic spikes, pre-scale to provisioned.
- **OS Dashboards public access (`AllowFromPublic: true`) is convenient for POC** but DANGEROUS for prod. Switch to VPC endpoints + IAM data access policy.
- **QuickSight SPICE incremental refresh requires monotonic date column.** Random updates to old rows = silent SPICE drift.
- **Athena partition projection** vs MSCK REPAIR — projection is preferred for `year/month/day` patterns; eliminates partition catalog overhead.

---

## Output artifacts

1. **CDK stacks** — `StreamingStack`, `OssStack`, `GlueAthenaStack`, `QuicksightStack`
2. **Lambda transform code** — `src/firehose_transform/transform.py` with PII hashing
3. **Producer SDK snippets** — JavaScript (browser), Python (server), Swift (iOS)
4. **OS Dashboards JSON** — 4 default panels exportable
5. **Athena saved queries** — 10 (events/day, conversion funnel, retention cohorts, geo distribution, etc.)
6. **QuickSight analysis JSON** — exportable via `describe-dashboard-definition`
7. **Q topic JSON** — column synonyms + named entities for NL queries
8. **CloudWatch dashboard JSON** — pipeline health overview
9. **Pytest suite** — covers all validation criteria
10. **README** — producer integration + dashboard URLs + KQL/SQL example queries

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 2.0 | 2026-04-26 | Initial. KDS + Firehose × 2 sinks (S3 Iceberg + OS Serverless) + QuickSight + Q topic. Wave 12. |
