<!-- Template Version: 2.0 | F369 Wave 16 (composite) | Composes: ECS_CLUSTER_FOUNDATION + ECS_DEPLOYMENT_PATTERNS + ECS_PRODUCTION_HARDENING + EVENT_DRIVEN_PATTERNS + SERVERLESS_DYNAMODB_PATTERNS -->

# Template 05 — ECS Blue/Green Microservices Platform (5+ services · Service Connect mesh · per-service Aurora/DDB · CodeDeploy · GuardDuty Runtime · 5-7 day deploy)

## Purpose

Stand up an **ECS-based microservices platform with 5+ services in 5-7 days** — service mesh via Service Connect, per-service blue/green CodeDeploy, per-service IAM/data isolation, observability across the mesh, security baselined.

This is the **canonical "ECS at scale" engagement** — a step beyond the Single Service POC. Shows how to land 5-15 services with proper isolation + deployment hygiene + cost optimization. For larger fleets (50+ services), recommend EKS instead.

Output: cluster, multi-service deployment + per-service pipelines, dashboards, runbooks.

---

## Role Definition

You are an expert AWS containers architect with deep expertise in:
- ECS multi-service architecture + Service Connect mesh
- Per-service ALB target groups + listener rule routing
- CodeDeploy ECS blue/green per service (independent pipelines)
- Per-service IAM roles + per-service secrets + per-service data
- Service-to-service auth (mTLS via Service Connect, JWT propagation, IAM SigV4)
- Cross-service observability (X-Ray service map, Container Insights v2)
- Bulkhead patterns (per-service capacity provider strategies)
- Anti-patterns: shared task role across services, single ALB without listener rules

Generate complete, production-deployable code. No TODOs.

---

## Context and Inputs

```
PROJECT_NAME:                [REQUIRED]
AWS_REGION:                  [REQUIRED]
AWS_ACCOUNT_ID:              [REQUIRED]
ENV:                         [REQUIRED — dev | stage | prod]

# --- SERVICES ---
SERVICES:                    [REQUIRED — JSON array of service specs:
                              [
                                {"name":"users","port":8080,"path":"/api/users","db":"ddb",
                                 "spot_eligible":true},
                                {"name":"orders","port":8080,"path":"/api/orders","db":"aurora",
                                 "spot_eligible":false},
                                {"name":"notifications","port":8080,"path":"/api/notifications",
                                 "db":"none","spot_eligible":true},
                                {"name":"worker","port":0,"path":"","db":"none",
                                 "spot_eligible":true,"sqs_source":"jobs-queue"},
                                {"name":"admin","port":8080,"path":"/api/admin","db":"aurora",
                                 "spot_eligible":false}
                              ]
                             ]

# --- VPC ---
VPC_ID:                      [REQUIRED]
PRIVATE_SUBNET_IDS:          [REQUIRED]
PUBLIC_SUBNET_IDS:           [REQUIRED]

# --- INGRESS ---
HOSTED_ZONE_NAME:            [REQUIRED]
DOMAIN_NAME:                 [REQUIRED — single domain; path-based routing]
ACM_CERT_ARN:                [REQUIRED]
ENABLE_WAF:                  [true default]
COGNITO_USER_POOL_ID:        [optional]

# --- DEPLOYMENT ---
DEPLOY_PER_SERVICE:          [true default — independent pipelines]
CANARY_CONFIG:               [CANARY_10_PERCENT_5_MINUTES default]

# --- DATA ---
SHARED_AURORA:               [false default — per-service Aurora preferred]
SHARED_DDB_TABLE:            [false default]

# --- COMPLIANCE ---
KMS_KEY_ARN:                 [REQUIRED]
LOG_RETENTION_DAYS:          [30 default]
ENABLE_GUARDDUTY_RUNTIME:    [true default]

# --- OBSERVABILITY ---
SNS_ALARM_TOPIC_ARN:         [REQUIRED]
ENABLE_XRAY:                 [true default — critical for multi-service tracing]

# --- COST ---
DAILY_FARGATE_BUDGET_USD:    [REQUIRED]
TARGET_SPOT_PERCENTAGE:      [40 default — for spot-eligible services]
```

---

## Partial Library (Claude MUST load)

| Partial | Why |
|---|---|
| `ECS_CLUSTER_FOUNDATION` | Cluster + Service Connect namespace |
| `ECS_DEPLOYMENT_PATTERNS` | CodeDeploy blue/green per service |
| `ECS_PRODUCTION_HARDENING` | Per-service IAM + GuardDuty + Inspector |
| `EVENT_DRIVEN_PATTERNS` | EventBridge for service-to-service async |
| `SERVERLESS_DYNAMODB_PATTERNS` | Per-service DDB single-table design |
| `LAYER_API` | ALB + listener rules |
| `LAYER_OBSERVABILITY` | X-Ray service map + dashboards |
| `LAYER_NETWORKING` | VPC + endpoints |
| `LAYER_SECURITY` | KMS + Secrets Manager |
| `SERVERLESS_HTTP_API_COGNITO` | Cognito JWT for inbound auth (optional) |

---

## Architecture (5-service example)

```
   Browser ──► Route 53: api.example.com
                     │
                     ▼
              ┌────────────────────────────┐
              │ WAF v2                      │
              └────────┬───────────────────┘
                       │
                       ▼
              ┌────────────────────────────┐
              │ ALB (single, shared)         │
              │ Listener:443 with rules:     │
              │   /api/users/*    → TG_users  │
              │   /api/orders/*   → TG_orders │
              │   /api/notifs/*   → TG_notif  │
              │   /api/admin/*    → TG_admin  │
              │   /healthz        → fixed-200 │
              │   default         → 404       │
              └────────┬───────────────────┘
                       │
                       ▼
   ┌─────────────────────────────────────────────────────────────────┐
   │ ECS Cluster (single, shared across all services)                │
   │   Capacity providers: FARGATE + FARGATE_SPOT                     │
   │   Container Insights v2 enhanced                                  │
   │   Service Connect namespace: prod.local                           │
   │                                                                    │
   │   Services (each registered in Service Connect):                  │
   │                                                                    │
   │   ┌─────────────┐  ┌──────────────┐  ┌────────────────────┐     │
   │   │ users-svc   │  │ orders-svc   │  │ notifications-svc  │     │
   │   │ FARGATE_SPOT│  │ FARGATE      │  │ FARGATE_SPOT        │     │
   │   │ 3-30 tasks  │  │ 3-30 tasks   │  │ 2-20 tasks          │     │
   │   │ DDB single  │  │ Aurora PG    │  │ no DB                │     │
   │   │ Pre-traffic │  │ Pre-traffic  │  │ Pre-traffic          │     │
   │   │ + canary    │  │ + canary     │  │ + canary             │     │
   │   └─────────────┘  └──────────────┘  └────────────────────┘     │
   │                                                                    │
   │   ┌─────────────┐  ┌──────────────┐                               │
   │   │ admin-svc   │  │ worker-svc   │                               │
   │   │ FARGATE     │  │ FARGATE_SPOT │                               │
   │   │ 2-10 tasks  │  │ 2-50 tasks   │                               │
   │   │ Aurora PG   │  │ no inbound   │                               │
   │   │             │  │ SQS-driven   │                               │
   │   └─────────────┘  └──────────────┘                               │
   │                                                                    │
   │   Service Connect mesh:                                            │
   │     orders-svc.prod.local:8080 → orders task replicas              │
   │     users-svc.prod.local:8080  → users task replicas               │
   │     ... (Envoy sidecar in each task handles routing + retries)    │
   │                                                                    │
   │   GuardDuty Runtime Monitoring on all tasks                        │
   │   ECR repos per service (KMS, IMMUTABLE, scan-on-push)             │
   │   Per-service task role (least-priv)                                │
   │   Per-service execution role                                         │
   └─────────────────────────────────────────────────────────────────┘
                       │
                       ▼
   ┌─────────────────────────────────────────────────────────────────┐
   │ Data layer                                                        │
   │   users-svc      → DDB (users-table, single-table)                │
   │   orders-svc     → Aurora PG (orders DB; PITR + KMS)               │
   │   admin-svc      → Aurora PG (shared with orders OR separate)     │
   │   worker-svc     → SQS (jobs-queue) → Aurora PG (writes)            │
   │   notifications-svc → SES + SNS (no persistence)                    │
   └─────────────────────────────────────────────────────────────────┘
                       │
                       ▼
   ┌─────────────────────────────────────────────────────────────────┐
   │ Async events (EventBridge custom bus)                            │
   │   order.placed → notifications-svc (send email)                   │
   │   order.placed → users-svc (update purchase history)               │
   │   user.signed-up → notifications-svc (welcome email)               │
   └─────────────────────────────────────────────────────────────────┘
                       │
                       ▼
   ┌─────────────────────────────────────────────────────────────────┐
   │ CI/CD per service (5 independent pipelines)                      │
   │   GitHub → CodeBuild → ECR push → CodeDeploy ECS blue/green       │
   │     - Pre-traffic Lambda smoke test                                │
   │     - Canary 10% for 5 min                                          │
   │     - Auto-rollback on alarm                                         │
   │     - Post-traffic verification                                       │
   └─────────────────────────────────────────────────────────────────┘
```

---

## Day-by-day execution (5-7 day deploy, 1-2 engineers)

### Day 1 — Foundation (cluster + ALB + ECR per service)
- KMS CMK
- ECS cluster + Service Connect namespace + Container Insights v2
- ALB + WAF + Cognito (if enabled) + ACM cert
- Per-service ECR repos (5 repos)
- Per-service IAM roles (5 task roles, 1 shared execution role per cluster, 5 codeDeploy roles)
- Per-service ALB target groups + listener rules
- VPC endpoints (ecr.api, ecr.dkr, logs, secretsmanager, kms, ssm, monitoring, s3 gateway)
- **Deliverable:** End of Day 1: cluster up; ALB routes test traffic to fixed-200 default; per-service ECRs ready.

### Day 2 — First 2 services deployed (users + orders)
- Build + push container images for `users` and `orders` services
- Per-service task definitions (Fargate + ARM64 + Powertools layer)
- Per-service Aurora PG cluster (orders) + DDB table (users)
- Service Connect registration per service
- ECS services with rolling deploys initially
- Smoke test path-based routing: `/api/users` → users-svc, `/api/orders` → orders-svc
- **Deliverable:** End of Day 2: 2 services live, accessible via path-based ALB routing.

### Day 3 — Remaining 3 services (notifications, admin, worker)
- Same pattern as Day 2 for each
- `worker` service has no inbound port (consumes from SQS)
- `notifications` integrates with SES + SNS
- `admin` shares Aurora cluster with orders OR has separate cluster
- EventBridge custom bus + rules for async event routing
- **Deliverable:** End of Day 3: 5 services running; async events flowing via EventBridge.

### Day 4 — CodeDeploy blue/green per service
- Per service: CodeDeploy app + deployment group + 2 target groups (blue + green)
- Per service: Pre-traffic Lambda smoke test
- Per service: CW alarms (5xx rate, latency p99, container restart rate)
- Per service: Auto-rollback config (deployment_in_alarm + failed_deployment + stopped_deployment)
- Per service: Canary 10% for 5 min then 100%
- Test deployment: deploy v2 of users-svc → canary → 100% successful
- Test rollback: deploy intentionally broken v3 → alarm fires → auto-rollback to v2
- **Deliverable:** End of Day 4: all 5 services have blue/green; rollback tested.

### Day 5 — Auto-scaling + GuardDuty Runtime + Inspector
- Per service: target tracking auto-scaling (CPU 50% + memory 70%)
- Worker service: step scaling on SQS depth
- GuardDuty Runtime Monitoring with `ECS_FARGATE_AGENT_MANAGEMENT: ENABLED`
- Inspector v2 enabled for ECR; CI gate to block CRITICAL CVE deploys
- Cross-service X-Ray instrumentation (Powertools tracer auto-traces; Service Connect adds spans)
- **Deliverable:** End of Day 5: load test triggers auto-scale; GuardDuty agent visible in tasks; X-Ray service map shows full mesh.

### Day 6-7 — Observability + load test + tests + handoff
- CloudWatch dashboard with per-service tiles + cross-service mesh diagram
- 30+ alarms across 5 services + cluster (capacity loss, error rates, deploy failures, GuardDuty findings)
- X-Ray service map shows latency + error rate per edge
- Pytest suite covering all validation criteria
- Load test (k6) — 1000 RPS sustained for 1h; verify auto-scale + no errors
- Runbook per service: deploy, rollback, scale, exec, debug
- Cost projection: Fargate Spot % achieved + Aurora + ALB + Inspector + GuardDuty
- **Deliverable:** End of Day 7: all tests pass; runbooks signed off; handoff complete.

---

## Validation criteria (program-level)

- [ ] **All 5 services running** with `running == desired` count
- [ ] **All 5 services in Service Connect** (Cloud Map shows entries)
- [ ] **Path-based ALB routing** works for each service
- [ ] **Per-service task roles unique** (no sharing — verified via boto3)
- [ ] **All deployments are blue/green CodeDeploy** with canary config
- [ ] **Auto-rollback tested** for each service (intentional broken deploy → reverts)
- [ ] **GuardDuty Runtime** active for all 5 services (verified via task agent)
- [ ] **Inspector** scanning ECR; zero CRITICAL CVEs across deployed images
- [ ] **Auto-scaling triggers** under load test for each service
- [ ] **EventBridge async events** delivered (test order.placed → notifications-svc)
- [ ] **X-Ray service map** complete; latency per edge < SLO
- [ ] **Spot percentage ≥ `TARGET_SPOT_PERCENTAGE`** for spot-eligible services
- [ ] **Daily Fargate cost ≤ `DAILY_FARGATE_BUDGET_USD`** (verified via Cost Explorer after 7 days)
- [ ] **All Aurora clusters Multi-AZ + KMS-encrypted + PITR**
- [ ] **All DDB tables** Streams + KMS + PITR + deletion_protection
- [ ] **VPC endpoints active** for required AWS APIs
- [ ] **Cross-service X-Ray traces** show mesh edges with sub-second p99 latency

## Validation criteria (per-service)

- [ ] Service running count == desired
- [ ] Health check passing on ALB target group
- [ ] CloudWatch alarms firing for known failure modes
- [ ] Pre-traffic Lambda smoke test passes during deploy
- [ ] Per-service Cognito JWT enforcement (if applicable)
- [ ] Per-service task IAM scoped to only its data resources

---

## Common gotchas (claude must address proactively)

- **Shared ALB with listener rules** is cheaper than per-service ALB ($16/mo each saved). But all services share rate-limit pool — bullies can affect others.
- **Per-service ECR repo** is mandatory for IMMUTABLE tags + per-service scan results in Inspector.
- **Service Connect mesh latency** adds ~5-15ms per hop (Envoy sidecar). For latency-sensitive paths, measure end-to-end.
- **Cross-service Aurora** = bottleneck. Per-service Aurora is more $$$ but isolates failure blast radius.
- **EventBridge custom bus archive** retains events for replay — useful for replaying after a downstream bug. Enable from day 1.
- **Per-service CodeDeploy = 5 separate apps + DGs** — automate via CDK loop. CDK pipeline can deploy all in single pipeline run.
- **Worker service has no ALB target group** — service auto-scales on SQS depth via step scaling instead.
- **Service Connect TLS** is opt-in — for inter-service mTLS, configure `tls.kms_key`. Without TLS, traffic is plain HTTP within VPC.
- **GuardDuty Runtime agent in Fargate** = ~50 MB image extension + 256 MB RAM extra per task. Account for this in task memory sizing.
- **Path-based routing precedence** — most-specific path first. `/api/admin/*` must come before `/api/*`.
- **Cognito on ALB** triggers a redirect on every unauthenticated request. Internal microservices should bypass via SG-only access.
- **X-Ray sampling**: default 1 RPS + 5% rate. For low-traffic services, increase to 100% sampling for visibility.
- **Per-service task definitions are versioned** — keep last 5-10 revisions for rollback flexibility.
- **Service quotas** — default 1000 services per cluster, 5000 tasks per service. For larger fleets, request quota increase or split clusters.

---

## Output artifacts

1. **CDK stacks**:
   - `EcsClusterStack` — cluster + namespace + ECRs + ALB + WAF + listener rules
   - `EcsServiceStack` (parameterized) — task def + service + target group + per-service IAM + Aurora/DDB
   - `EcsCodeDeployStack` (parameterized) — CodeDeploy app + DG + Lambda hooks + alarms per service
   - `EcsObservabilityStack` — dashboards + alarms + GuardDuty + X-Ray
   - `EventBusStack` — EventBridge custom bus + rules + targets

2. **Per-service container Dockerfiles** — multi-arch (ARM64 + amd64) + Powertools layer + non-root user

3. **CodeDeploy lifecycle hook Lambdas** — pre-traffic + post-traffic per service (smoke + integration tests)

4. **Pytest validation suite** — per-service + program-level

5. **CloudWatch dashboards**:
   - Cluster overview
   - Per-service tile (CPU, memory, deployments, errors)
   - Cross-service mesh latency map (Service Connect metrics)
   - Cost dashboard (Fargate $/service/day)

6. **30+ alarms YAML** — per-service + cluster

7. **CI/CD pipelines per service**:
   - GitHub Actions workflow OR CodePipeline
   - Build (Buildkit multi-arch) → push → Inspector wait → CodeDeploy

8. **Runbooks** (Markdown):
   - Per-service deploy / rollback procedures
   - Cluster-wide ops (cordon a service, scale all)
   - Incident response (high error rate, capacity loss, GuardDuty alert)

9. **Load test scripts** (k6) — per service + cluster-wide

10. **Cost projection**:
    - Per-service Fargate hourly + Spot %
    - Aurora cluster sizing
    - DDB on-demand vs provisioned
    - ALB + Cognito + WAF
    - GuardDuty + Inspector + Container Insights

11. **Architecture diagram** (Mermaid or graphviz) — service mesh + data flow

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 2.0 | 2026-04-27 | Initial composite template. Composes 3 ECS partials + EventBridge + DDB. 5-service ECS platform in 5-7 days. Wave 16. |
