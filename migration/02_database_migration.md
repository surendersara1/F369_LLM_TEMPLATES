<!-- Template Version: 2.0 | F369 Wave 13 (composite) | Composes: MIGRATION_SCHEMA_CONVERSION + DATA_DMS_REPLICATION + MIGRATION_HUB_STRATEGY + DATA_RDS_MULTIAZ_CLUSTER + DATA_AURORA_GLOBAL_DR -->

# Template 02 — Database Migration (Oracle/SQL Server → Aurora · DMS Schema Conversion · DMS classic data load · Babelfish · validation)

## Purpose

Migrate **a fleet of heterogeneous databases (Oracle, SQL Server, Db2, Sybase) → Aurora PostgreSQL / Aurora MySQL / Redshift** in **8-16 weeks**, including: schema conversion, data migration, application code refactor coordination, and cutover with minimal downtime.

Output: assessment reports, converted schemas, DMS replication tasks, validation results, cutover runbook, post-cutover monitoring.

This is the **canonical "exit Oracle" / "modernize databases" engagement** — escapes proprietary licensing, cuts costs 40-70%.

---

## Role Definition

You are an expert AWS database migration architect with deep expertise in:
- AWS DMS Schema Conversion (in-DMS replacement for SCT) for heterogeneous DB conversion
- AWS DMS Fleet Advisor for portfolio assessment
- AWS DMS classic for data migration (full load + CDC) + data validation
- Babelfish for Aurora PostgreSQL (T-SQL passthrough — 80% reduction in app code change)
- Aurora PostgreSQL + Aurora MySQL + RDS Multi-AZ DB cluster
- Application code refactor (PL/SQL, T-SQL stored procedures → PostgreSQL functions)
- Q Developer / CodeWhisperer for stored procedure translation
- Cutover orchestration with minimal app downtime (1-30 min typical)

Generate complete, production-deployable code. No TODOs.

---

## Context and Inputs

```
PROJECT_NAME:                [REQUIRED]
AWS_REGION:                  [REQUIRED]
ENV:                         [REQUIRED]

# --- SOURCE FLEET ---
SOURCE_DBS:                  [REQUIRED — JSON array of {name, engine, version, size_gb, app_owner}]
SOURCE_NETWORK:              [REQUIRED — vpn | dx_1g | dx_10g]

# --- TARGET STRATEGY ---
ORACLE_TARGETS:              [aurora_pg default — also: rds_pg | redshift]
SQL_SERVER_TARGETS:          [REQUIRED — aurora_pg | aurora_pg_babelfish | rds_sql_server (lift)]
DB2_TARGETS:                 [aurora_pg default]
SYBASE_TARGETS:              [aurora_pg default]
USE_BABELFISH:               [true if SQL Server fleet > 10 DBs and ANSI-mostly]

# --- WAVE PLAN ---
WAVE_COUNT:                  [REQUIRED — 4 typical for 50 DBs]
WAVE_CADENCE_WEEKS:          [4 default]
COMPLEXITY_BREAKDOWN:        [from Fleet Advisor: LOW/MED/HIGH counts]

# --- CUTOVER ---
MAX_DOWNTIME_MINUTES:        [REQUIRED — drives cutover strategy]
                             # < 5 min  → continuous CDC + DNS swap
                             # 5-30 min → CDC + brief read-only window
                             # > 30 min → batch full-load with weekend window

# --- DATA VALIDATION ---
VALIDATION_LEVEL:            [STRICT default — full row hash compare]
                             # STRICT | TYPED | LIGHT (count only)

# --- COMPLIANCE ---
KMS_KEY_ARN:                 [REQUIRED]
ENCRYPT_AT_REST_DMS:         [true required]
ENCRYPT_IN_TRANSIT:          [SSL required for all endpoints]

# --- OBSERVABILITY ---
SNS_ALARM_TOPIC_ARN:         [REQUIRED]
SLACK_WEBHOOK_URL:           [for migration progress updates]
```

---

## Partial Library (Claude MUST load)

| Partial | Why |
|---|---|
| `MIGRATION_SCHEMA_CONVERSION` | DMS Schema Conversion + Fleet Advisor + Babelfish |
| `DATA_DMS_REPLICATION` | DMS classic for data movement (full load + CDC) |
| `MIGRATION_HUB_STRATEGY` | Per-DB strategy + wave plan + tracking |
| `DATA_RDS_MULTIAZ_CLUSTER` | Target RDS Multi-AZ deployment patterns |
| `DATA_AURORA_GLOBAL_DR` | Aurora Global if multi-region target |
| `LAYER_SECURITY` | KMS + Secrets Manager for DB credentials |
| `LAYER_OBSERVABILITY` | Performance Insights + Enhanced Monitoring |

---

## Architecture (program-level)

```
   Source DBs (on-prem or other cloud)
   ┌─────────────────────────────────────────────────────────────────┐
   │ Oracle 12c/19c (12 DBs, 2-20 TB each)                            │
   │ SQL Server 2016/2019 (8 DBs, 500GB-5TB each)                     │
   │ PostgreSQL 11/13 (30 DBs, varying — homogeneous, no SCT)         │
   │ Db2 LUW (3 DBs, legacy)                                          │
   └─────────────────────────────────┬───────────────────────────────┘
                                     │ JDBC over DX/VPN with SSL
                                     ▼
   ┌─────────────────────────────────────────────────────────────────┐
   │ Phase 1 — Discovery (Week 1-2)                                   │
   │   - DMS Fleet Advisor agent (Docker container in DC)              │
   │   - 24h-7day scan of all DBs                                       │
   │   - Per-DB: complexity score (LOW/MED/HIGH), recommended target   │
   │   - Per-DB: license cost analysis (Oracle BYOL vs PG free)         │
   └─────────────────────────────────────────────────────────────────┘
   ┌─────────────────────────────────────────────────────────────────┐
   │ Phase 2 — Schema Conversion (Week 3-N per wave)                  │
   │   - DMS Migration Project per source DB                            │
   │   - Conversion: 80%+ auto for typical Oracle/SQL Server            │
   │   - Manual: PL/SQL packages, hierarchical queries, OUTER JOIN (+) │
   │   - Tools: DMS-SC editor, Q Developer for stored proc translate   │
   │   - Output: SQL DDL files → apply to Aurora target                 │
   └─────────────────────────────────────────────────────────────────┘
   ┌─────────────────────────────────────────────────────────────────┐
   │ Phase 3 — Data Migration (overlap with Phase 2)                  │
   │   - DMS Classic replication instance (dms.r6i.4xlarge multi-AZ)   │
   │   - Replication tasks: full-load + CDC                              │
   │   - Source endpoint: Oracle/SQL Server (read-only user)            │
   │   - Target endpoint: Aurora PG (DDL pre-applied)                    │
   │   - Validation: row-level hash compare                              │
   │   - Bandwidth: monitor, throttle if needed                          │
   └─────────────────────────────────────────────────────────────────┘
   ┌─────────────────────────────────────────────────────────────────┐
   │ Phase 4 — Application Refactor (parallel to Phase 2-3)           │
   │   - Translate PL/SQL packages → PG functions                        │
   │   - Update connection strings (port 1521→5432, JDBC URL)            │
   │   - Update ORM dialects (Hibernate, EntityFramework)                │
   │   - For Babelfish targets: keep T-SQL + TDS port 1433               │
   │   - For full PG targets: app refactor via Q Developer               │
   │   - Test end-to-end in stage with Aurora target                     │
   └─────────────────────────────────────────────────────────────────┘
   ┌─────────────────────────────────────────────────────────────────┐
   │ Phase 5 — Cutover (4-week per wave, 30 min downtime typical)     │
   │   T-1d: Stage cutover dry-run                                       │
   │   T-1h: App enters read-only mode (writes blocked)                  │
   │   T-0: Wait for DMS CDC lag → 0                                     │
   │   T+5min: Stop source DB                                            │
   │   T+10min: Update app config to target DB                           │
   │   T+15min: Validate writes succeed on Aurora                        │
   │   T+30min: GO/NO-GO; release read-only                              │
   │   T+1d: Source DB on read-only (rollback safety net)                │
   │   T+7d: Source DB decommission                                      │
   └─────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
   ┌─────────────────────────────────────────────────────────────────┐
   │ Aurora target fleet                                              │
   │   - 12 Aurora PostgreSQL clusters (Oracle replacements)           │
   │   - 8 Aurora PG with Babelfish (SQL Server replacements)          │
   │   - 30 Aurora PG (homogeneous PG migrations)                      │
   │   - 3 Aurora PG (Db2 replacements)                                │
   │   - Performance Insights + Enhanced Monitoring                     │
   │   - Cross-region read replica (for tier-1 DBs)                    │
   └─────────────────────────────────────────────────────────────────┘
```

---

## Wave-by-wave execution (4-week cycle per wave)

### Wave Cycle (per wave, 4 weeks)

#### Week 1 — Pick wave, deep-dive each DB
- Select 4-6 DBs for the wave (mix of complexity)
- Run DMS Schema Conversion assessment per DB
- Review report — auto-conversion %, manual items, complexity per object
- App owner sign-off on target DB choice (PG full vs Babelfish vs Redshift)

#### Week 2 — Schema conversion + manual fixes
- Apply auto-conversions
- Fix manual items in DMS-SC editor (PL/SQL packages, CONNECT BY, etc.)
- Use Q Developer for difficult translations
- Apply to dev Aurora target → run schema diff validation
- Iterate until schema diff = 0

#### Week 3 — Data migration + validation + app refactor
- DMS classic replication task (full-load + CDC) per DB
- Monitor `FullLoadProgressPercent` + `CdcLatencyTarget`
- After full-load: enable validation; review `ValidationFailedRecords`
- App team refactors connection strings + ORM dialects
- Stage cutover dry-run: app team points to Aurora target → smoke test

#### Week 4 — Cutover + hyper-care
- T-2 days: stage cutover dry-run with full app stack
- T-1 day: GO/NO-GO meeting
- Cutover (downtime per `MAX_DOWNTIME_MINUTES`):
  - App enters read-only mode
  - Wait for CDC lag → 0
  - Stop source DB
  - Update app config (ConnString → Aurora endpoint)
  - Validate writes
  - Release read-only mode
- T+1d: source DB read-only safety net
- T+7d: source decommission + DMS task cleanup

### Cutover strategies by `MAX_DOWNTIME_MINUTES`

| Downtime allowed | Strategy |
|---|---|
| < 5 min | Continuous CDC + DNS-based cutover; DB connection pool drains old + reconnects to new |
| 5-30 min | Brief read-only window + CDC final catch-up + app config update |
| > 30 min | Weekend window; freeze app, full data load (no CDC), cutover |
| Zero (active-active) | Aurora Global Database + bidirectional CDC during transition (advanced) |

---

## Validation criteria (per wave)

- [ ] **Schema conversion ≥ 80% auto** for each DB; manual items signed off by DBA
- [ ] **Schema diff = 0** between converted DDL applied to target vs DMS-SC output (object count, column count)
- [ ] **DMS replication task running** with `FullLoadProgressPercent: 100` + CDC active
- [ ] **DMS validation: 0 failed rows** for STRICT level (or < 0.01% for TYPED)
- [ ] **CDC lag < 5 min** for 24h continuously before cutover
- [ ] **App stage smoke test passes** end-to-end with Aurora target
- [ ] **Performance Insights baseline** captured on Aurora target before cutover
- [ ] **Backup of source taken** within 24h pre-cutover
- [ ] **Rollback plan tested** in stage (restart source → repoint app → app works)
- [ ] **Cutover dry-run successful** with full timing: App freeze + DMS catch-up + app config update + smoke test all within `MAX_DOWNTIME_MINUTES`

## Validation criteria (program-level)

- [ ] **Fleet Advisor assessment complete** for all `SOURCE_DBS`
- [ ] **License cost savings tracked** — Oracle/SQL Server BYOL → PG (no license)
- [ ] **All Aurora targets Multi-AZ + KMS-encrypted + PITR enabled**
- [ ] **Migration Hub progress streams** updating per wave
- [ ] **Daily DMS replication cost < budget**
- [ ] **Source DBs decommissioned within 7 days post-cutover**

---

## Common gotchas (claude must address proactively)

- **Stored procedure refactor is the #1 timeline risk.** Oracle PL/SQL packages with AUTONOMOUS_TRANSACTION, hierarchical queries, OPENROWSET — auto-conversion ≤ 30%. Plan extra weeks per complex DB.
- **Empty string vs NULL semantics differ** — Oracle treats `''` as NULL; PostgreSQL treats `''` as empty string. App-level NULL checks expose bugs that didn't exist on Oracle.
- **Identity / sequence semantics differ** — Oracle preallocates; PG doesn't. Apps that rely on sequential IDs without gaps WILL break.
- **Date arithmetic** — Oracle `DATE - DATE` returns days; PG returns `interval`. Wrap with `EXTRACT(DAY FROM ...)`.
- **CHAR padding** — Oracle pads CHAR; PG pads CHAR but VARCHAR doesn't. Use VARCHAR everywhere for portability.
- **DMS source supplemental logging** — Oracle requires `ALTER DATABASE ADD SUPPLEMENTAL LOG DATA` + per-table for CDC. ~10% redo log overhead.
- **DMS source change tracking on SQL Server** requires CDC enabled at DB + table level (`sys.sp_cdc_enable_db`).
- **DMS target Aurora — set `WriteBufferSize` task setting** for high-volume writes; default 1MB is small for 100K rows/sec.
- **Babelfish single-db vs multi-db mode** — choose at cluster create; cannot change later. Multi-db = SQL Server-like multi-database; single-db = single PG database.
- **LOB column data not validated by default** — DMS validation skips LOBs unless `ValidateLOBOnly: true`. Manually compare LOB columns post-cutover.
- **Application connection pool tuning** — Aurora PG has different `max_connections` defaults than Oracle. Tune `max_connections` parameter group + app pool sizes.
- **Partition tables** — Oracle range partitioning maps to PG declarative partitioning since v11. DMS-SC handles. Verify partition pruning works on target.
- **`SELECT ... FOR UPDATE`** semantics differ. Oracle: row-locks until commit. PG: same, but with `NOWAIT` / `SKIP LOCKED` extensions. Test ORM behavior.

---

## Output artifacts

1. **CDK stacks**:
   - `DmsInfrastructureStack` (replication instance Multi-AZ, subnet group, KMS)
   - `AuroraTargetStack` (per-source Aurora cluster, parameter groups, secrets)
   - `BabelfishStack` (if `USE_BABELFISH`)
   - `DiscoveryStack` (Fleet Advisor agent EC2 + IAM)

2. **DMS configuration**:
   - Migration Project per source DB (Schema Conversion)
   - Endpoint per source + target
   - Replication task per source DB (full-load + CDC + validation)
   - Table mappings + transformation rules

3. **Schema conversion artifacts** (per DB):
   - Assessment report PDF + JSON + CSV
   - Auto-converted DDL SQL
   - Manual fix log + comments
   - Final apply script (idempotent psql script)

4. **Application refactor**:
   - Connection string templates (Oracle → PG, SQL Server → Babelfish)
   - ORM dialect changes (Hibernate, EF, Sequelize, SQLAlchemy)
   - Stored procedure translation (Oracle → PG) via Q Developer

5. **Cutover runbooks**:
   - Master cutover runbook
   - Per-DB cutover runbook (with app cutover steps)
   - Rollback runbook
   - Hyper-care runbook (T+0 to T+7 days)

6. **Pytest validation suite**:
   - Schema diff tests (object count per source vs target)
   - Data validation tests (DMS validation reports parsed)
   - App smoke tests against target (CRUD + key business queries)
   - Cutover gate tests (lag, validation status, app smoke)

7. **Dashboards**:
   - DMS task progress dashboard
   - Aurora Performance Insights dashboards
   - Cost dashboard (DMS + replication target Aurora)

8. **Cost models**:
   - Per-DB license savings (Oracle BYOL @ $X/core → PG = $0)
   - Aurora vs source RDS sizing TCO
   - Migration program cost (DMS replication + parallel-run)

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 2.0 | 2026-04-27 | Initial composite template. Composes MIGRATION_SCHEMA_CONVERSION + DATA_DMS_REPLICATION + Aurora targets. 8-16 week DB migration program. Wave 13. |
