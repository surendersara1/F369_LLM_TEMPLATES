<!-- Template Version: 2.0 | F369 Wave 14 (composite) | Composes: DR_MULTI_REGION_PATTERNS + DR_ROUTE53_ARC + DR_BACKUP_VAULT_LOCK + DATA_AURORA_GLOBAL_DR + ENTERPRISE_NETWORK_HUB_TGW -->

# Template 11 — Multi-Region Disaster Recovery (Pilot Light · Warm Standby · Active-Active · Aurora Global · DDB Global · ARC failover · Vault Lock backups)

## Purpose

Stand up a **production-grade multi-region DR plan in 3-5 weeks** that hits explicit RPO/RTO targets, has tested failover via Route 53 ARC, immutable backups via Vault Lock, and quarterly game-day validation. This template fits the canonical "make our app survive regional failure" engagement.

Output: replicated workload across 2 regions, ARC routing controls, runbook, validation tests.

---

## Role Definition

You are an expert AWS resilience architect with deep expertise in:
- 4 DR patterns (Backup-Restore / Pilot Light / Warm Standby / Active-Active) + RPO/RTO trade-offs
- Aurora Global Database + DynamoDB Global Tables v2 + S3 CRR with RTC + multi-region KMS
- Route 53 Application Recovery Controller (ARC) — cluster + routing controls + safety rules + readiness checks
- AWS Backup centralized + Vault Lock COMPLIANCE for ransomware-proof backups
- Resilience Hub assessments + FIS chaos engineering for DR validation
- DR runbook authoring + game day facilitation

Generate complete, production-deployable code. No TODOs.

---

## Context and Inputs

```
PROJECT_NAME:                [REQUIRED]
PRIMARY_REGION:              [REQUIRED — e.g. us-east-1]
DR_REGION:                   [REQUIRED — e.g. us-west-2; geographically separate]

# --- DR PATTERN PER WORKLOAD ---
WORKLOADS:                   [REQUIRED — JSON array of {name, dr_pattern, rpo_target, rto_target}]
                             # Example:
                             # [
                             #   {"name":"customer-api", "dr_pattern":"warm_standby",
                             #    "rpo_target":"60s", "rto_target":"5min"},
                             #   {"name":"admin-tools",  "dr_pattern":"pilot_light",
                             #    "rpo_target":"5min",   "rto_target":"30min"},
                             #   {"name":"reports",      "dr_pattern":"backup_restore",
                             #    "rpo_target":"4h",     "rto_target":"4h"},
                             # ]

# --- DATA SERVICES ---
HAS_AURORA:                  [true if any workload uses Aurora]
HAS_DYNAMODB:                [true]
HAS_S3:                      [true]
S3_CRR_RTC_ENABLED:          [true required for prod]
HAS_OPENSEARCH:              [false default; manual snapshot replication if true]

# --- FAILOVER ORCHESTRATION ---
FAILOVER_TRIGGER:             [REQUIRED — manual_only | r53_health_check | arc_routing_control]
ENABLE_ZONAL_AUTOSHIFT:      [true default — for AZ-level resilience]

# --- BACKUPS ---
ENABLE_VAULT_LOCK:           [COMPLIANCE default for prod]
BACKUP_RETENTION_YEARS:      [7 default for compliance; 1 for non-regulated]
ENABLE_CROSS_ACCOUNT_VAULT:  [true for prod]
BACKUP_ACCOUNT_ID:           [REQUIRED if cross-account]

# --- COMPLIANCE ---
KMS_KEY_ARN_PRIMARY:         [REQUIRED]
KMS_MULTI_REGION:            [true required]
COMPLIANCE_FRAMEWORK:        [pci | hipaa | soc2 | nist | combined]

# --- GAME DAYS ---
GAME_DAY_CADENCE:            [quarterly default — also: monthly | weekly]
SLO_ERROR_RATE_THRESHOLD:    [1 default — % above which FIS auto-stops]

# --- OBSERVABILITY ---
SNS_ALARM_TOPIC_ARN:         [REQUIRED]
PAGERDUTY_INTEGRATION_KEY:   [REQUIRED]
```

---

## Partial Library (Claude MUST load)

| Partial | Why |
|---|---|
| `DR_MULTI_REGION_PATTERNS` | 4 DR patterns + multi-region replication primitives |
| `DR_ROUTE53_ARC` | Failover control plane (routing controls, readiness, safety rules) |
| `DR_BACKUP_VAULT_LOCK` | Immutable backups with Vault Lock + cross-region + cross-account |
| `DR_RESILIENCE_HUB_FIS` | (validation) Resilience Hub assessments + FIS chaos engineering |
| `DATA_AURORA_GLOBAL_DR` | Aurora Global Database — primary + secondary cluster |
| `ENTERPRISE_NETWORK_HUB_TGW` | (if multi-VPC) cross-region TGW peering |
| `LAYER_SECURITY` | Multi-region KMS key foundation |
| `LAYER_OBSERVABILITY` | CW alarms + dashboards baseline |

---

## Architecture (canonical Warm Standby for tier-1 workloads)

```
   ┌──────────────────────────────────────────────────────────────────┐
   │                       Route 53                                    │
   │   Hosted zone: example.com                                        │
   │   ARC Cluster (5-region data plane)                                │
   │   ARC Control Panel: prod-failover                                 │
   │   ARC Routing Controls:                                            │
   │     - prod-primary-us-east-1 (ON)                                  │
   │     - prod-secondary-us-west-2 (OFF)                                │
   │   Safety Rules:                                                    │
   │     - At least one ON (assertion)                                   │
   │     - Cannot turn primary OFF unless secondary ON (gating)         │
   │   ARC Readiness Checks (6 resource sets per workload)              │
   └──────────────────────────────────────────────────────────────────┘
                                  │
              ┌───────────────────┴────────────────────┐
              ▼                                          ▼
   ┌─────────────────────────────┐      ┌──────────────────────────────┐
   │ PRIMARY (us-east-1)         │      │ DR (us-west-2) — WARM STANDBY│
   │                              │      │                                │
   │ ALB → ECS Fargate (10 tasks) │      │ ALB → ECS Fargate (2 tasks)   │
   │   + auto-scaling (2-50)      │      │   + auto-scaling (1-50)        │
   │ ↓                            │      │ ↓                              │
   │ Aurora PG (writer + reader)  │◄────►│ Aurora PG secondary (reader)  │
   │ (Aurora Global; RPO < 1s)    │      │ (read-only; promotable)        │
   │ DynamoDB (multi-region)      │ ───► │ DynamoDB (active-active)       │
   │ S3 (KMS multi-region key)    │ CRR ►│ S3 replica (RTC 15-min RPO)    │
   │ Secrets Manager               │ ───► │ Secrets Manager replicated     │
   │ ECR repo                      │ ───► │ ECR replicated                  │
   └──────────────────────────────┘      └──────────────────────────────┘
              ▲                                          ▲
              │  AWS Backup centralized                  │
              │  ┌────────────────────────────────────┐  │
              ├──┤ Backup Vault (us-east-1)            ├──┤
              │  │   Vault Lock COMPLIANCE             │  │
              │  │   30d daily / 1y weekly / 7y monthly│  │
              │  └────────────┬────────────────────────┘  │
              │               │ cross-region + cross-account copy
              │               ▼
              │  ┌────────────────────────────────────┐
              │  │ Backup Vault (us-west-2 + Backup   │
              │  │   Account)                          │
              │  │   Vault Lock COMPLIANCE             │
              │  │   IMMUTABLE; ransomware-proof       │
              │  └────────────────────────────────────┘
              │
              │  Resilience Hub
              │   - Resiliency policies per workload
              │   - Quarterly assessment
              │
              │  FIS
              │   - Weekly: kill 30% ECS tasks
              │   - Monthly: Aurora failover test
              │   - Quarterly: full region partition (game day)
```

---

## Day-by-day execution (3-5 week deployment, 1-2 engineers)

### Week 1 — Foundation: multi-region KMS + AWS Backup + Vault Lock
- Multi-region KMS CMK in primary; replica in DR
- AWS Backup vault in primary + DR; Vault Lock COMPLIANCE (3-day grace, then sealed)
- Backup plan with daily/weekly/monthly rules + cross-region copy + cross-account copy
- Tag tier-1 resources (`backup: daily-30day`)
- Run first backup; validate success
- (If `ENABLE_CROSS_ACCOUNT_VAULT`) configure cross-account vault in dedicated backup account
- **Deliverable:** Vault Lock active; first cross-region backup completes; cannot delete recovery point even as root.

### Week 2 — Replication: Aurora Global + DDB Global + S3 CRR + Secrets Manager
- Aurora Global Database — primary cluster (existing) + secondary cluster in DR
- DynamoDB tables → Global Tables v2 (add DR region as replica)
- S3 buckets → enable Cross-Region Replication with RTC (15-min RPO)
- Secrets Manager secrets → replicated to DR region
- ECR cross-region replication (registry-level)
- Validate replication lag — Aurora < 1s, DDB < 1s, S3 < 15 min
- **Deliverable:** All data replicated; RPO targets met.

### Week 3 — Compute standby: ECS / Lambda in DR + Route 53 ARC
- Deploy CDK app stacks to DR region (same code, different regional resources)
- ECS service in DR with `desired_count=2` (warm standby) — running but minimal
- ALB in DR with target group registered
- Set up Route 53 ARC:
  - Cluster (5-region data plane)
  - Control Panel + 2 Routing Controls (primary ON, secondary OFF)
  - Safety rules (at least one ON, gating on primary OFF)
  - Readiness checks per resource set (Aurora, DDB, S3, ECS, ALB)
  - R53 records: PRIMARY + SECONDARY failover with RECOVERY_CONTROL health checks
- Validate: bringing primary down via routing control flip → traffic to DR
- **Deliverable:** Manual failover via ARC works end-to-end.

### Week 4 — Validation: Resilience Hub + FIS chaos engineering
- Resilience Hub: import app, define resiliency policy (RPO/RTO per workload)
- Run baseline assessment — capture initial score (typical 70-85 for first run)
- Address top 5 recommendations (e.g., add health check, add cross-region copy)
- Deploy FIS experiment templates:
  - Kill 30% ECS tasks (auto-recovery validation)
  - Aurora failover (DB reconnect validation)
  - S3 throttle (retry behavior)
  - Region partition (full DR drill)
- Run experiments in stage; validate expected behavior
- **Deliverable:** Resilience score ≥ 80; 4 FIS experiments running successfully in stage.

### Week 5 — Game day + runbook + handoff
- Author DR runbook (Markdown) covering:
  - Failover decision matrix (when to fail over, who decides, escalation)
  - Pre-failover checklist (data lag verification, readiness check, capacity)
  - Cutover steps (ARC routing control flip, capacity scale-up, monitoring)
  - Validation steps (synthetic check, error rate, latency)
  - Failback procedure (when primary recovers)
  - Communication plan (status page, Slack, customer comms templates)
- Run **first game day** with full multi-team observation:
  - 1h tabletop scenario walk-through
  - 30-min simulated regional failure (FIS region partition + ARC failover)
  - Retrospective: timing, blockers, gaps
- Schedule recurring game days per `GAME_DAY_CADENCE`
- **Deliverable:** Working DR end-to-end; runbook validated; team confident in failover.

---

## Validation criteria

- [ ] **Multi-region KMS key** with replica in DR region; encryption-at-rest in both
- [ ] **Aurora Global Database** secondary lag < 1s p99
- [ ] **DDB Global Tables** replicating; latest item replicated < 1s
- [ ] **S3 CRR with RTC** active; sample object replicates < 15 min
- [ ] **Secrets Manager replicated** to DR region; status `InSync`
- [ ] **ECR cross-region replication** active; new image push appears in DR within 5 min
- [ ] **AWS Backup vault locked COMPLIANCE** in primary + DR; even root cannot delete
- [ ] **Backup recovery points** successfully copied cross-region (verified in DR vault)
- [ ] **Cross-account backup vault** receiving copies (if enabled)
- [ ] **ARC cluster ACTIVE** with 5 endpoints; routing controls reachable from all 5
- [ ] **ARC routing control flip** changes R53 DNS resolution within 30s (TTL bound)
- [ ] **ARC readiness check** PASSES for all resource sets in both regions
- [ ] **Resilience Hub score ≥ 80** for tier-1 workloads
- [ ] **FIS experiments executable + auto-stop on SLO breach** (verified by injection test)
- [ ] **First game day run** with multi-team observation + retrospective documented
- [ ] **DR runbook signed off** by app owners + ops + security
- [ ] **DR drill scheduled quarterly** in calendar; PagerDuty escalation pre-configured

---

## Common gotchas (claude must address proactively)

- **Vault Lock COMPLIANCE is permanent.** Test in stage with COMPLIANCE first, validate, then promote to prod. Cannot undo.
- **Aurora Global write forwarding adds 100-300ms latency** — use sparingly. Apps should write to primary region directly.
- **DDB Global Tables conflict resolution = last-writer-wins by item-level timestamp.** Apps that update same item from both regions can lose data; design app-level idempotency.
- **S3 CRR doesn't replicate existing objects** — only new puts. Use S3 Batch Replication for backfill.
- **ARC cluster cost ~$250/mo + $2.50/control/mo.** Justify vs basic R53 health checks (which are $0.50/check).
- **Failover orchestration**: ARC routing control flips DNS within 30s, but client-side DNS caches can take longer (TTL respect). Set R53 TTL to 30s on failover records.
- **Cross-region data transfer** — replicating 10 TB monthly = ~$200/mo. Plan budget.
- **FIS experiments can damage production.** Always require CW alarm stop conditions; tag-based IAM scoping; practice in stage 30+ days.
- **Resilience Hub recommendations are best-effort guidance**, not blocking — engineering judgment required.
- **Game days surface real bugs** — first game day typically reveals 5-10 gaps. Budget time to fix before next.
- **Failback is harder than failover** — primary recovers but data on DR may have drifted. Design failback process before failover ever needed.

---

## Output artifacts

1. **CDK stacks** (per-region):
   - `DrPrimaryStack` — primary region resources (KMS, S3, DDB, Aurora, Secrets, ECR)
   - `DrSecondaryStack` — DR region resources (replicas + Aurora secondary + ECS desired_count=2)
   - `BackupStack` — AWS Backup vault + plan + Vault Lock + cross-region copy
   - `ArcStack` — Cluster + Control Panel + Routing Controls + Safety Rules + R53 records
   - `ResilienceHubStack` — App + Resiliency Policy + assessments
   - `FisStack` — 4+ experiment templates with stop conditions

2. **Multi-region KMS key replica** + key policy
3. **Aurora Global Database** primary + secondary clusters
4. **DDB Global Tables v2** with DR region replica
5. **S3 buckets** with CRR + RTC + KMS encryption
6. **Failover orchestration script** (ARC routing control flip with retry across 5 cluster endpoints)
7. **DR runbook** (Markdown):
   - Decision matrix
   - Pre-failover checklist
   - Cutover steps
   - Validation
   - Failback
   - Communication plan
   - Customer comms templates
8. **Game day playbook** + 4 scenarios (region failure, AZ failure, DB failure, ALB failure)
9. **Pytest validation suite** — covers all validation criteria
10. **CloudWatch dashboards**:
    - DR posture overview (replication lag, vault status, ARC status)
    - Per-workload RPO/RTO observed vs target
11. **Quarterly game day calendar** + PagerDuty schedule
12. **Cost projection** — DR costs (vault, replication, ARC, FIS) per month

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 2.0 | 2026-04-27 | Initial composite template. Composes 5 DR + supporting partials. 3-5 week multi-region DR program. Wave 14. |
