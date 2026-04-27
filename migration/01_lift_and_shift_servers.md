<!-- Template Version: 2.0 | F369 Wave 13 (composite) | Composes: MIGRATION_MGN + MIGRATION_HUB_STRATEGY + MIGRATION_DATASYNC + ENTERPRISE_NETWORK_HUB_TGW -->

# Template 01 — Lift-and-Shift Server Migration (MGN · waves · post-launch automation · vCenter agentless · cutover runbook)

## Purpose

Stand up a **production-grade server migration program** delivering 50-500 servers from on-prem (or other cloud) to AWS EC2 in **6-12 weeks**. Output: discovery, wave plan, MGN replication active, post-launch automation, cutover runbook, day-2 operations.

This is the **canonical "data center exit" engagement** — every enterprise client adopting AWS at scale needs this.

Generates production-deployable CDK + CLI runbooks + cutover checklists.

---

## Role Definition

You are an expert AWS migration architect with deep expertise in:
- AWS Application Migration Service (MGN) agent + agentless vCenter Source Server Connector
- Application Discovery Service (ADS) + Migration Hub Strategy Recommendations
- 6R framework + wave-based migration program management
- Post-launch automation via SSM (CW Agent install, AD domain-join, app-level smoke tests)
- DataSync for non-server data (file shares, object storage)
- Network design for migration (Direct Connect, VPN, Transit Gateway hub)
- Cutover orchestration + runbook authoring

Generate complete, production-deployable code. No TODOs.

---

## Context and Inputs

```
PROJECT_NAME:                [REQUIRED]
AWS_REGION:                  [REQUIRED]
AWS_ACCOUNT_ID:              [REQUIRED]
ENV:                         [REQUIRED — typically prod/migration]

# --- SOURCE ENVIRONMENT ---
SOURCE_TYPE:                 [REQUIRED — vmware | hyperv | physical | azure | gcp | mixed]
SOURCE_DATACENTER_LOCATION:  [REQUIRED — for latency planning]
SOURCE_VPN_OR_DX:            [REQUIRED — vpn | dx_1g | dx_10g | public_internet]
SERVER_COUNT:                [REQUIRED]
WINDOWS_PCT:                 [REQUIRED — % of fleet on Windows]
LINUX_PCT:                   [REQUIRED]

# --- TARGET ---
TARGET_VPC_ID:               [REQUIRED — or to-create]
STAGING_SUBNET_CIDR:         [REQUIRED — sized /22 for > 250 servers]
TARGET_INSTANCE_RIGHT_SIZING:[BASIC default — also: NONE | CUSTOM]
ACTIVE_DIRECTORY:            [REQUIRED if Windows — Managed AD ARN OR self-managed AD config]

# --- WAVE PLAN ---
WAVE_COUNT:                  [REQUIRED — 3 typical for 100 servers; 8 for 500]
WAVE_CADENCE_WEEKS:          [4 default — 4-week wave cycle]
CUTOVER_WINDOW:              [REQUIRED — typically Sat 10pm - Sun 8am ET]

# --- POST-LAUNCH AUTOMATION ---
POST_LAUNCH_DOC_NAMES:       [comma-separated SSM doc names — e.g. install-agents,domain-join,smoke-test]
APP_SPECIFIC_DOCS:           [comma-separated per-app SSM docs]

# --- COMPLIANCE ---
KMS_KEY_ARN:                 [REQUIRED]
ENCRYPT_REPLICATION:         [true required]
LOG_RETENTION_DAYS:          [365 default for regulated]

# --- OBSERVABILITY ---
SNS_ALARM_TOPIC_ARN:         [REQUIRED]
PAGERDUTY_INTEGRATION_KEY:   [REQUIRED for cutover events]
SLACK_WEBHOOK_URL:           [for wave status updates]

# --- BUDGET ---
DAILY_REPLICATION_COST_BUDGET_USD: [REQUIRED]
COMPLETE_BY_DATE:            [REQUIRED — DC contract termination]
```

---

## Partial Library (Claude MUST load)

| Partial | Why |
|---|---|
| `MIGRATION_MGN` | MGN agent + agentless + post-launch SSM + cutover |
| `MIGRATION_HUB_STRATEGY` | Migration Hub + ADS + 6R + wave plan + TCO |
| `MIGRATION_DATASYNC` | (if file shares present) NFS/SMB → S3/EFS/FSx |
| `ENTERPRISE_NETWORK_HUB_TGW` | (if multi-account migration) TGW + Egress + RAM |
| `LAYER_NETWORKING` | VPC + endpoints baseline |
| `LAYER_SECURITY` | KMS + IAM least-privilege |
| `LAYER_OBSERVABILITY` | CW dashboards + alarms baseline |

---

## Architecture (program-level)

```
   Source Data Center (on-prem)
   ┌──────────────────────────────────────────────────────────────┐
   │ vCenter / Hyper-V / Physical                                 │
   │   - 250 servers (mixed Windows/Linux)                         │
   │   - Source Server Connector OVA (agentless for VMware)        │
   │   - Replication Agent on physical / Hyper-V hosts              │
   │   - File shares: NetApp NFS (50 TB), Windows SMB (10 TB)       │
   │   - DBs: 12 Oracle, 8 SQL Server, 30 PostgreSQL                │
   └──────────────────────────────┬───────────────────────────────┘
                                  │ DX 10 Gbps OR VPN
                                  ▼
   AWS — Migration Account
   ┌──────────────────────────────────────────────────────────────┐
   │ Migration Hub (home region us-east-1)                         │
   │   - Aggregates: MGN + DMS + DataSync + App2Container          │
   │   - Wave dashboard                                              │
   │                                                                 │
   │ Application Discovery Service                                  │
   │   - Agent inventory (250 servers)                              │
   │   - Network dependency map                                      │
   │                                                                 │
   │ Migration Hub Strategy Recommendations                         │
   │   - Per-app 6R recommendation                                  │
   │   - 138 REHOST, 35 REPLATFORM, 12 REFACTOR, 25 REPURCHASE,    │
   │     20 RETIRE, 20 RETAIN                                        │
   │                                                                 │
   │ Target VPC (10.50.0.0/16)                                      │
   │   ├── Staging subnets (/22, replication servers)                │
   │   ├── Production subnets (migrated workloads)                   │
   │   ├── Active Directory (Managed AD)                              │
   │   └── DataSync agent EC2 instance for file shares               │
   │                                                                 │
   │ MGN (per-region, per-account)                                  │
   │   - Replication settings template (CMK, dedicated repl srv)    │
   │   - Launch template (instance type right-sizing, SSM docs)      │
   │   - Source server inventory: 250 servers                        │
   │                                                                 │
   │ Post-launch automation                                          │
   │   - SSM doc 1: Install CW Agent + SSM Agent                     │
   │   - SSM doc 2: Domain-join (Windows)                             │
   │   - SSM doc 3: Per-app smoke test                                │
   │   - SSM doc 4: Tag for cost-allocation                            │
   └────────────────────────────────────────────────────────────────┘
```

---

## Wave-by-wave execution (4-week wave cycle, repeated)

### Wave Cycle (per wave, 4 weeks)

#### Week 1 — Discovery + Plan refinement
- ADS agent inventory for wave servers
- Confirm dependencies (which app talks to which)
- Validate target instance sizes via Strategy Recommendations
- Stakeholder sign-off on wave scope + cutover window

#### Week 2 — Replication setup
- Install MGN replication agent on each wave server (or vCenter Source Server Connector if VMware)
- Configure launch templates per server (override instance type if needed)
- Configure post-launch action chain
- Wait for replication: NOT_READY → READY_FOR_TESTING (typically 2-7 days for full disk sync)

#### Week 3 — Test + iteration
- Run test launches for sample 5-10 servers per app
- Validate post-launch SSM docs execute correctly
- Run app smoke tests — connect, log in, basic flow
- Fix issues; re-test
- Mark all wave servers READY_FOR_CUTOVER

#### Week 4 — Cutover
- T-2 days: dry-run cutover for 1 server end-to-end
- T-1 day: stakeholder GO/NO-GO meeting
- T-0 (cutover window — typically Sat 10pm to Sun 8am):
  1. App freeze (DB read-only / app maintenance mode)
  2. Wait for MGN replication lag → 0
  3. Stop source apps (or quiesce)
  4. `start-cutover` for all wave servers in dependency order
  5. Wait for instances to launch + post-launch actions to complete
  6. Run app-level cutover (DNS swap, LB target re-point, cert rotation)
  7. Smoke test (automated + manual)
  8. **GO**: finalize-cutover + monitor for 48h
  9. **NO-GO**: revert (start sources back up + reset replication)
- T+2 days: hyper-care monitoring; address P1/P2 issues
- T+7 days: source decommission + replication cleanup

### Wave Summary Schedule (example for 200-server program)

| Wave | Apps | Servers | Risk | Cutover Date |
|---|---|---|---|---|
| W1 | Dev environment | 30 | LOW | 2026-05-09 |
| W2 | Internal tools (HR, ITSM) | 35 | LOW | 2026-06-06 |
| W3 | Stage / Pre-prod | 45 | MED | 2026-07-04 |
| W4 | Prod tier 1 (web/app) | 60 | HIGH | 2026-08-01 |
| W5 | Prod tier 2 (DB-fronts) | 30 | HIGH | 2026-08-29 |

Total: 200 servers, 5 waves, 5 months end-to-end.

---

## Validation criteria (per wave)

- [ ] **All wave servers in MGN** with `state: READY_FOR_CUTOVER`
- [ ] **Replication lag < 5 min** for all wave servers at cutover-1
- [ ] **Test launch success rate ≥ 95%** before cutover go-decision
- [ ] **Post-launch SSM docs executed successfully** on test launches (verified via SSM execution history)
- [ ] **App smoke tests pass** on test launches
- [ ] **Cutover runbook signed off** by app owner + ops + security
- [ ] **PagerDuty escalation chain confirmed** for cutover window
- [ ] **Backup of source taken** within 24h of cutover (rollback safety)
- [ ] **Migration Hub progress streams updating** (MGN auto-reports)
- [ ] **Cost-allocation tags** applied to all migrated EC2 (Wave, AppGroup, Owner)

## Validation criteria (program-level)

- [ ] **All 6R classifications signed off** by app owners (RACI matrix complete)
- [ ] **Source data center contract termination date documented**; program timeline backsolved
- [ ] **Daily replication cost < `DAILY_REPLICATION_COST_BUDGET_USD`**
- [ ] **Migration Hub dashboard reflects all in-flight + completed waves**
- [ ] **Source decommission verified** within 7 days post-cutover (no extra replication $$$)

---

## Common gotchas (claude must address proactively)

- **VMware Tools must be current** for Source Server Connector to work. Older versions cause silent enumeration failures.
- **MGN replication burns WAN bandwidth.** Throttle during business hours; full-throttle nights/weekends. Plan for ~10× compressed source delta over migration period.
- **Domain-join post-launch action requires AD reachability** from migrated EC2. If AD on-prem, need DX/VPN routing; if Managed AD, ensure VPC route exists.
- **Boot mode mismatch** = unbootable EC2. UEFI sources → UEFI EC2 instance type only; pre-Nitro instance types don't support UEFI.
- **Drivers and license activation** — Windows post-migration may show "Activate Windows" if KMS server unreachable. Use AWS Provided KMS or VPN to corporate KMS.
- **Cutover NO-GO scenarios**: replication stuck > 1 day, post-launch SSM docs failing, app smoke test fails. Always have go-back plan.
- **MGN re-test after agent reinstall** — if agent replaced/restarted, replication may regress. Validate state before relying on cutover window.
- **MGN cost = staging instances + EBS replication** — about $1-2/server/day. Cumulative across 12 weeks = $90-180 per server.
- **Cross-account migration** — staging area in target account; source accounts use Source Server Connector with cross-account access keys. Use AFT (`ENTERPRISE_CONTROL_TOWER`) for the target account governance.
- **Wave ordering** — DBs migrate AFTER their dependent apps OR via DMS classic CDC sync (use `MIGRATION_SCHEMA_CONVERSION` + `DATA_DMS_REPLICATION` partial).

---

## Output artifacts

1. **CDK stacks**:
   - `MgnInfrastructureStack` (target VPC + staging subnet + post-launch SSM docs + IAM)
   - `MigrationHubStack` (home region + progress streams + ADS access)
   - `WaveTrackingStack` (CW dashboards + alarms per wave)

2. **MGN configuration**:
   - Replication settings template JSON
   - Launch template per Wave with right-sizing rules
   - 4 post-launch SSM documents (install-agents, domain-join, smoke-test, tag-resources)

3. **Discovery + planning**:
   - ADS agent install scripts (Linux + Windows + Ansible playbook)
   - Wave plan spreadsheet (servers × waves × cutover dates)
   - 6R classification per app (CSV)
   - Network dependency map (graphviz / mermaid diagram)

4. **Cutover runbooks** (Markdown):
   - Master cutover runbook (wave-agnostic)
   - Per-wave cutover runbook (with wave-specific app cutover steps)
   - Rollback / NO-GO runbook
   - Post-cutover hyper-care runbook (T+0 to T+7 days)

5. **Pytest validation suite**:
   - Pre-cutover gate tests (replication state, lag, post-launch validation)
   - Post-cutover smoke tests (app endpoint reachable, AD-joined, agents running)
   - Daily program health checks

6. **Dashboards**:
   - Migration Hub master dashboard
   - CloudWatch wave-level dashboard (replication progress, lag, cost)
   - Cost dashboard (daily replication spend vs budget)

7. **Cost models**:
   - TCO calculator output (source vs target)
   - Migration program cost (replication + parallel-run + labor)
   - Per-wave cost projection

8. **Communication artifacts**:
   - Stakeholder weekly status template
   - Wave kickoff deck
   - Cutover GO/NO-GO checklist

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 2.0 | 2026-04-27 | Initial composite template. Composes MIGRATION_MGN + HUB_STRATEGY + DATASYNC. 6-12 week server migration program. Wave 13. |
