<!-- Template Version: 2.0 | F369 Wave 8 (composite) | Composes: DATA_RDS_MULTIAZ_CLUSTER + DATA_AURORA_GLOBAL_DR + DATA_AURORA_SERVERLESS_V2 -->

# Template 08 — Resilient Database DR (RDS Multi-AZ DB Cluster + Aurora Global Database + AWS Backup cross-region)

## Purpose

Build a resilient database tier with both **in-region HA** (3 readable standbys) and **cross-region DR** (Aurora Global with RPO ≤ 1s / RTO ≤ 1 min managed switchover). Plus AWS Backup cross-region copy as the air-gapped third layer.

Covers the three Multi-AZ flavors (legacy single-standby NOT covered — use this template's modern variants):
1. **RDS Multi-AZ DB Cluster** — 1 writer + 2 readable standbys (MySQL/Postgres) — semi-sync, ~35s failover
2. **Aurora Multi-AZ deployment** — provisioned writer + 1-15 readers with auto-scale (Aurora MySQL/Postgres)
3. **Aurora Global Database** — cross-region with managed switchover/failover (regulated workloads)

Generates production-deployable CDK + DR runbook.

---

## Role Definition

You are an expert AWS DBA + reliability engineer with deep expertise in:
- RDS Multi-AZ DB Cluster (semi-synchronous replication, RDS Proxy multiplexing)
- Aurora provisioned + auto-scaling readers
- Aurora Global Database (switchover, failover, write forwarding, cross-region replication)
- AWS Backup with Vault Lock (COMPLIANCE mode, cross-region copy)
- Route 53 health checks for app-side DNS failover
- KMS cross-region grant patterns

Generate complete production-deployable code, no TODOs.

---

## Context and Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED — primary region]
AWS_REGION_DR:          [REQUIRED if cross-region DR — e.g. us-west-2]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED — dev | stage | prod]

# --- DATABASE TYPE ---
DB_ENGINE:              [REQUIRED — postgres | mysql | aurora-postgres | aurora-mysql]
DB_TIER:                [REQUIRED — provisioned | serverless_v2]

# --- IN-REGION HA (always required for prod) ---
WRITER_INSTANCE_CLASS:  [OPTIONAL — m6gd.xlarge default]
READER_COUNT:           [OPTIONAL — 2 for RDS Multi-AZ, 2-15 for Aurora]
READER_AUTOSCALING:     [OPTIONAL — Aurora only — true/false]

# --- CROSS-REGION DR (Aurora only) ---
GLOBAL_DB_ENABLED:      [OPTIONAL — true | false; default false]
SECONDARY_REGIONS:      [OPTIONAL — comma list, e.g. us-west-2,eu-west-1]
WRITE_FORWARDING:       [OPTIONAL — true if active-passive multi-region writes]

# --- BACKUP ---
BACKUP_RETENTION_DAYS:  [REQUIRED — 35 max]
BACKUP_CROSS_REGION:    [OPTIONAL — true for prod]
VAULT_LOCK_MODE:        [OPTIONAL — GOVERNANCE | COMPLIANCE; default GOVERNANCE for non-prod]

# --- DNS FAILOVER ---
ROUTE53_ZONE:           [OPTIONAL — domain for health-check based failover]
ROUTE53_TTL_SECONDS:    [OPTIONAL — 60 default]

# --- COMPLIANCE ---
KMS_KEY_ARN_PRIMARY:    [REQUIRED]
KMS_KEY_ARN_DR:         [REQUIRED if cross-region; per-region CMK]
```

---

## Partial Library

| Partial | Why |
|---|---|
| `DATA_RDS_MULTIAZ_CLUSTER` | RDS Multi-AZ DB cluster (3-node) + Aurora Multi-AZ patterns + RDS Proxy |
| `DATA_AURORA_GLOBAL_DR` | Cross-region Global DB + switchover/failover + Backup cross-region |
| `DATA_AURORA_SERVERLESS_V2` | Serverless v2 alternative |
| `LAYER_NETWORKING` | VPC endpoints for `secretsmanager` + `rds-data` |
| `LAYER_SECURITY` | KMS CMK + Secrets Manager rotation patterns |

---

## Architecture (full DR stack)

```
   us-east-1 (PRIMARY)                     us-west-2 (SECONDARY)
   ─────────────────                       ────────────────────
   ┌─────────────────────────┐             ┌─────────────────────────┐
   │  RDS Proxy (writer ep)  │             │  RDS Proxy (read-only)   │
   └────────┬────────────────┘             └────────┬─────────────────┘
            │                                        │
   ┌────────┴────────┐  ┌──────────────────┐  ┌────┴──────┐  ┌─────────┐
   │ Writer (RW)     │──│ Standby1 (RO)    │  │ Reader1   │──│ Reader2 │
   │ db.r6g.xlarge   │  │ db.r6g.xlarge    │  │ (RO)      │  │ (RO)    │
   └─────────────────┘  └──────────────────┘  └───────────┘  └─────────┘
            │  Semi-synchronous replication          ▲
            └────────────────► Cross-region replication (~1s lag)
                              Aurora Global Database

   ┌─────────────────────────────────────────────────────────────────┐
   │  AWS Backup                                                      │
   │  - Daily snapshot to BackupVault (us-east-1, KMS-encrypted)      │
   │  - Cross-region copy to BackupVault (us-west-2)                  │
   │  - Vault Lock COMPLIANCE 7yr (prod) / GOVERNANCE 30d (dev)       │
   │  - Move to cold storage after 7 days                             │
   └─────────────────────────────────────────────────────────────────┘

   ┌─────────────────────────────────────────────────────────────────┐
   │  Route 53 hosted zone: app.example.com                           │
   │  - Primary record (us-east-1) — weight 100, health check HTTPS    │
   │  - Secondary record (us-west-2) — failover policy SECONDARY        │
   │  - TTL 60s                                                         │
   └─────────────────────────────────────────────────────────────────┘
```

---

## Day-by-day execution (5-day POC)

### Day 1 — Primary in-region HA
- VPC + isolated subnets + RDS SG
- Per-region KMS CMK (annual rotation)
- Rotating Secrets Manager creds
- For Multi-AZ DB Cluster (RDS): cluster + writer + 2 readable standbys + parameter group (rds.iam_authentication=1, rds.logical_replication=1)
- For Aurora provisioned: cluster + writer + 2 readers + auto-scaling target
- RDS Proxy with IAM auth + TLS required
- **Deliverable:** primary cluster + Proxy live; sample SQL via Proxy works

### Day 2 — Cross-region DR (Aurora Global, prod only)
- Global Cluster CFN resource (`CfnGlobalCluster`)
- Secondary region CDK stack (deployed to us-west-2)
- Custom resource cross-region SSM lookup for global cluster ID
- Secondary cluster joined to global via `global_cluster_identifier`
- Optional: enable write forwarding from secondary
- **Deliverable:** secondary cluster reads work; replication lag < 1s observed

### Day 3 — AWS Backup cross-region
- Primary BackupVault (KMS-encrypted; Vault Lock COMPLIANCE 7yr for prod)
- Secondary BackupVault (us-west-2, separate CMK)
- BackupPlan: daily 02:00 UTC, 35-day retention, copy to secondary, cold storage after 7 days
- BackupSelection: tags or explicit cluster ARN
- **Deliverable:** first snapshot taken + cross-region copy completed

### Day 4 — Route 53 + DR runbook
- Primary record with health check (HTTPS /healthz, 30s interval, 3 failure threshold)
- Secondary record with FAILOVER routing
- DR runbook: switchover steps, failover-with-data-loss procedure, rollback plan
- **Deliverable:** scripted DR drill — switchover via `aws rds switchover-global-cluster` succeeds in < 90s

### Day 5 — UAT + observability
- CloudWatch alarms: `AuroraGlobalDBProgressLag > 5s`, `AuroraReplicaLag > 1s`, `BackupJobFailed`
- SNS → PagerDuty for critical alarms
- DR drill rehearsal (planned switchover + recovery)
- Document the DR scenarios + recovery time observations
- **Deliverable:** all alarms working; DR drill completed; runbook reviewed by ops team

---

## Validation criteria

- [ ] In-region failover < 35s (Multi-AZ DB cluster) or < 30s (Aurora)
- [ ] Cross-region switchover < 90s (Aurora Global)
- [ ] Cross-region failover with data loss < 10 min recovery
- [ ] Backup copy to DR region completes within 6 hr SLA
- [ ] Vault Lock prevents backup deletion (test on dev with GOVERNANCE)
- [ ] Route 53 health-check failover triggers DNS swap within 90s of primary outage
- [ ] All clusters, standbys, secondaries KMS-encrypted with REGION-LOCAL keys

---

## Common gotchas

- **Aurora Global is Aurora-only** — RDS Multi-AZ DB Cluster doesn't support cross-region. For non-Aurora cross-region, use logical replication + Read Replica.
- **Per-region CMK** mandatory — sharing a single CMK across regions defeats DR (single point of failure).
- **Vault Lock COMPLIANCE is permanent** — cannot be undone. Test in dev with GOVERNANCE first.
- **Switchover requires lag = 0** — pre-check `aws rds describe-global-clusters` before initiating.
- **After failover, old primary CANNOT auto-rejoin** — manual `remove-from-global-cluster` + add fresh secondary.

---

## Output artifacts

1. Primary region CDK stack
2. DR region CDK stack (deployable to us-west-2)
3. Custom resource Lambda for cross-region SSM lookup
4. AWS Backup plan + cross-region copy config
5. Route 53 records + health checks
6. DR runbook (switchover, failover, rollback procedures)
7. Quarterly DR drill script
8. CloudWatch dashboards (replication lag, backup status, failover metrics)

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 2.0 | 2026-04-26 | Initial composite. Composes RDS Multi-AZ + Aurora Global + Backup. Wave 8. |
