<!-- Template Version: 2.0 | F369 Wave 8 (composite) | Composes: DATA_ATHENA_FEDERATED_QUERY + DATA_ATHENA + DATA_LAKE_FORMATION + DATA_GLUE_CATALOG -->

# Template 07 — Multi-DB Federation Query (single SQL across Aurora · Postgres · DynamoDB · Snowflake · Redshift · S3)

## Purpose

Set up Athena Federated Query so analysts can JOIN across multiple operational databases + the lakehouse in a single SQL statement. Replaces "ETL everything into S3 first" pattern with on-demand federation. Two paths covered:

1. **Lambda connectors via SAR** — 30+ pre-built connectors for any RDBMS / NoSQL / SaaS source
2. **Glue Catalog Federation** (newer 2024+) — register external metastore (Snowflake, BigQuery, Iceberg REST) AS A CATALOG inside Glue, queryable natively

Plus Lake Formation governance over federated queries (LF-TBAC tags applied to federated catalog enforce row/column filters).

Generates production-deployable CDK + connector deployment scripts.

---

## Role Definition

You are an expert AWS data engineer with deep expertise in:
- Athena engine v3 + workgroup configuration
- Athena Federated Query (Lambda connectors + Glue Catalog Federation)
- 30+ pre-built connectors via SAR (Postgres, MySQL, Oracle, SQL Server, DynamoDB, DocumentDB, Snowflake, BigQuery, Redshift, OpenSearch, MSK, etc.)
- Glue Catalog with federation type catalogs (CfnCatalog with FederatedCatalog)
- Lake Formation TBAC enforcement on federated catalogs
- Cross-VPC + cross-account connector deployment patterns
- Pushdown optimization (which connectors push WHERE, IN, LIMIT, ORDER BY)

Generate complete, production-deployable code. No TODOs, no placeholder comments.

---

## Context and Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED — dev | stage | prod]

# --- FEDERATED SOURCES (pick 1+) ---
SOURCE_AURORA_POSTGRES:    [OPTIONAL — true/false; uses athena-postgresql connector]
SOURCE_DYNAMODB:           [OPTIONAL — true/false; uses athena-dynamodb connector]
SOURCE_SNOWFLAKE:          [OPTIONAL — true/false; choose path: lambda OR glue_federation]
SOURCE_REDSHIFT:           [OPTIONAL — true/false]
SOURCE_OPENSEARCH:         [OPTIONAL — true/false]
SOURCE_DOCUMENTDB:         [OPTIONAL — true/false]
SOURCE_BIGQUERY:           [OPTIONAL — true/false; prefers glue_federation]
SOURCE_MSK_KAFKA:          [OPTIONAL — true/false; ad-hoc only]

# --- PER-SOURCE SETUP (fill for each enabled source) ---
AURORA_PG_HOST:            [Aurora writer endpoint]
AURORA_PG_DB:              [database name]
AURORA_PG_SECRET_ARN:      [Secrets Manager arn for username/password]
AURORA_PG_SG_ID:           [security group allowing connector ingress]

DDB_TABLES_INCLUDE:        [comma-separated table names; empty = all]

SNOWFLAKE_ACCOUNT:         [acme.snowflakecomputing.com]
SNOWFLAKE_WAREHOUSE:       [warehouse name]
SNOWFLAKE_DB:              [database name]
SNOWFLAKE_SECRET_ARN:      [Secrets Manager arn]
SNOWFLAKE_PATH:            [lambda | glue_federation; default glue_federation]

# --- ATHENA WORKGROUP ---
WORKGROUP_NAME:            [e.g. analyst-federated-prod]
WORKGROUP_BYTES_SCANNED_CUTOFF: [REQUIRED — e.g. 1000000000 (1 GB)]
WORKGROUP_ENGINE:          [v3 always]

# --- COMPLIANCE ---
KMS_KEY_ARN:               [REQUIRED]
LAKE_FORMATION_TBAC:       [REQUIRED — true for prod]
LF_TAGS_TO_APPLY:          [domain/sensitivity tags to apply to each federated catalog]

# --- VPC ---
VPC_ID:                    [REQUIRED — connector Lambdas must be in VPC]
PRIVATE_SUBNET_IDS:        [REQUIRED — comma-separated]
LAMBDA_SG_ID:              [REQUIRED — connector Lambda's SG]
```

---

## Partial Library (Claude MUST load)

| Partial | Why |
|---|---|
| `DATA_ATHENA_FEDERATED_QUERY` | Decision tree, full connector matrix, pushdown semantics, CDK for SAR + Glue Federation |
| `DATA_ATHENA` | Workgroup foundation, scan cutoffs, engine v3, result bucket |
| `DATA_LAKE_FORMATION` | Apply LF-Tags to federated catalogs |
| `DATA_GLUE_CATALOG` | Native Glue Catalog underlying federation |
| `LAYER_BACKEND_LAMBDA` | 5 non-negotiables |
| `LAYER_NETWORKING` | VPC + connector Lambda subnets + source SG ingress |
| `LAYER_SECURITY` | KMS + Secrets Manager rotation |

---

## Architecture

```
   Athena query (workgroup: analyst-federated-prod, scan cap 1GB):
   ───────────────────────────────────────────────────────────
   SELECT o.order_id, o.amount, c.tier, i.season
   FROM aurora_pg_prod.public.orders o            ← Path 1: Lambda connector
   JOIN dynamodb.customers c                      ← Path 1: Lambda connector
     ON o.customer_id = c.customer_id
   JOIN snowflake_prod.analytics.products p       ← Path 2: Glue Federation
     ON o.product_id = p.product_id
   JOIN lakehouse_curated.product_inventory i     ← Native Glue Catalog (Iceberg)
     ON p.product_id = i.product_id
   WHERE o.created_at >= DATE '2026-04-01'
     AND c.tier IN ('platinum', 'gold')
     AND i.season = 'spring'
   LIMIT 1000;

   ─── Athena engine v3 ───
        │
        ├─► aurora_pg_prod ──► Lambda(athena-postgresql)
        │                       │
        │                       ▼
        │                       Aurora Postgres ─── PrivateLink ─── source DB
        │
        ├─► dynamodb ───────► Lambda(athena-dynamodb) ──► DynamoDB
        │
        ├─► snowflake_prod ─► Glue Catalog Federation ──► Snowflake (no Lambda!)
        │                       (uses Glue Connection w/ JDBC URL)
        │
        └─► lakehouse_curated → Native Glue Catalog ─► S3 Iceberg

   Lake Formation governs ALL catalogs uniformly:
     - Tags on aurora_pg_prod  → row/column filters apply
     - Tags on snowflake_prod  → same
     - Tags on lakehouse_curated → same

   Spill (large intermediate results > 4 MB) → s3://athena-spill-{env}/ (7d expiry)
```

---

## Day-by-day execution (3-day POC, 1 dev)

### Day 1 — Foundation
- Athena workgroup (engine v3, scan cap, result bucket KMS-encrypted)
- Spill bucket (KMS, 7-day lifecycle)
- IAM analyst role with workgroup-scoped permissions
- Lake Formation strict mode (if `LAKE_FORMATION_TBAC=true`)
- VPC endpoints for Athena + Glue + KMS + Secrets Manager
- **Deliverable:** native Glue Catalog query works in workgroup

### Day 2 — Lambda connectors (Path 1 — for each enabled SAR source)
- For each source in (aurora_postgres, dynamodb, redshift, opensearch, docdb, msk):
  - Deploy SAR application (`AthenaPostgreSQLConnector`, etc.) — `sam.CfnApplication`
  - Configure parameters (LambdaFunctionName, DefaultConnectionString, SecretNamePrefix, SpillBucket, VPC config)
  - Register Athena DataCatalog (`CfnDataCatalog` type=LAMBDA)
- **Deliverable:** sample federated query runs against each connector's source — `SELECT COUNT(*) FROM aurora_pg_prod.public.orders` returns

### Day 3 — Glue Catalog Federation (Path 2 — for Snowflake / BigQuery / Iceberg REST)
- Glue Connection (JDBC for Snowflake, gRPC for BigQuery)
- Driver JAR upload to S3 (Snowflake JDBC 3.16+, BigQuery driver)
- `glue.CfnCatalog` with `FederatedCatalog` configuration
- LF-Tags applied at catalog level → row/column filters enforced
- Test cross-DB JOIN spanning ≥3 catalogs (one per path)
- **Deliverable:** the architecture-diagram query above runs successfully under analyst-role with LF enforcement

---

## Validation criteria

- [ ] **Pushdown verified:** EXPLAIN on `WHERE created_at >= DATE '2026-04-01'` shows predicate pushed to source (not full scan)
- [ ] **Latency:** simple federated query returns in < 30 sec for moderate datasets
- [ ] **Cost cap holds:** workgroup `BytesScannedCutoffPerQuery` rejects runaway queries
- [ ] **LF enforcement:** unauthorized role blocked at table/column level on federated catalog
- [ ] **Spill cleanup:** 7-day-old objects expired by lifecycle
- [ ] **Connector resilience:** source DB restart → connector reconnects within 60s

---

## Common gotchas (claude must address proactively)

- **Connector Lambda VPC config required** for any source DB not on public internet. Include VPC + subnets + SG.
- **PassRole for connector Lambda** — connector role must be passable from Athena to Lambda
- **Spill bucket lifecycle = 7 days max.** Forgetting this leaks $$$ in spill files.
- **Glue Federation doesn't pushdown all WHERE clauses** — for Snowflake-Glue-Federation, complex WHERE may revert to full table read
- **DDB connector pushdown only on partition key** — `WHERE customer_id = ...` works; `WHERE name LIKE '%foo%'` doesn't pushdown

---

## Output artifacts

1. CDK Python app (FederationStack)
2. SAR connector deployment YAML + CDK wrappers (one per Lambda source)
3. Glue Connection JDBC config (for Snowflake)
4. Sample query suite (10 queries hitting all federated sources + lakehouse)
5. Pushdown verification script (runs EXPLAIN, parses result)
6. CloudWatch dashboard for connector invocation count + duration

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 2.0 | 2026-04-26 | Initial composite template. Composes Athena Federated + Lake Formation. Wave 8. |
