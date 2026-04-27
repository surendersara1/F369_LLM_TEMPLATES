<!-- Template Version: 2.0 | F369 Wave 16 (composite) | Composes: ECS_CLUSTER_FOUNDATION + ECS_DEPLOYMENT_PATTERNS + ECS_PRODUCTION_HARDENING + LAYER_API + LAYER_OBSERVABILITY -->

# Template 04 — ECS Production Baseline (Fargate · Service Connect · Blue/Green · auto-scaling · GuardDuty Runtime · 2-3 day POC)

## Purpose

Stand up a **production-grade ECS cluster + first service in 2-3 days** for clients who want containers without Kubernetes complexity. Output: cluster, Service Connect mesh, ALB ingress, blue/green deploys, auto-scaling, security hardening, observability — ready for additional services.

This is the **canonical "ECS engagement" alternative to EKS**. Fits clients with < 50 microservices, simpler ops teams, AWS-native preference.

---

## Role Definition

You are an expert AWS ECS architect with deep expertise in:
- ECS cluster + capacity providers (Fargate + Fargate Spot + EC2)
- Service Connect (replaces App Mesh; GA 2023)
- ECS task definition (multi-container, ARM64, ulimits, healthCheck)
- CodeDeploy ECS blue/green with canary configurations + Lambda hooks
- Service auto-scaling (target tracking + step + scheduled)
- ECR + Inspector + GuardDuty Runtime Monitoring for ECS
- Container Insights v2 + ECS Exec for debugging
- Fargate Spot economics + ECS Anywhere (on-prem)

Generate complete, production-deployable code. No TODOs.

---

## Context and Inputs

```
PROJECT_NAME:                [REQUIRED]
AWS_REGION:                  [REQUIRED]
AWS_ACCOUNT_ID:              [REQUIRED]
ENV:                         [REQUIRED — dev | stage | prod]

# --- VPC ---
VPC_ID:                      [REQUIRED — or to-create]
PRIVATE_SUBNET_IDS:          [REQUIRED — comma-separated, ≥ 3 AZs]
PUBLIC_SUBNET_IDS:           [REQUIRED for internet-facing ALB]

# --- CLUSTER ---
CLUSTER_NAME:                [REQUIRED — e.g. f369-prod-cluster]
SERVICE_CONNECT_NAMESPACE:   [{ENV}.local default]
ENABLE_FARGATE_SPOT:         [true default — 70% discount on fault-tolerant]
ENABLE_EC2_CAPACITY:         [false default — true if GPU/heavy workloads]

# --- FIRST SERVICE ---
SERVICE_NAME:                [REQUIRED — e.g. api]
CONTAINER_IMAGE_REPO:        [REQUIRED — ECR repo name]
CONTAINER_PORT:              [8080 default]
DESIRED_COUNT:               [3 default]
TASK_CPU:                    [1024 default — 1 vCPU]
TASK_MEMORY:                 [2048 default — 2 GB]
ARCHITECTURE:                [ARM64 default — Graviton, 20% cheaper]

# --- DEPLOYMENT ---
DEPLOYMENT_TYPE:             [REQUIRED — rolling | blue_green]
CANARY_CONFIG:               [CANARY_10_PERCENT_5_MINUTES default if blue_green]
TERMINATION_WAIT_HOURS:      [1 default for blue/green prod]

# --- INGRESS ---
HOSTED_ZONE_NAME:            [REQUIRED]
DOMAIN_NAME:                 [REQUIRED — e.g. api.example.com]
ACM_CERT_ARN:                [REQUIRED — regional]
ENABLE_WAF:                  [true default for prod]
COGNITO_USER_POOL_ID:        [optional — for ALB Cognito auth]

# --- AUTO-SCALING ---
MIN_CAPACITY:                [3 default — HA baseline]
MAX_CAPACITY:                [30 default]
SCALING_METRIC:              [cpu default — also: memory, alb_request_count, sqs_depth]
TARGET_CPU_PCT:              [50 default]

# --- COMPLIANCE ---
KMS_KEY_ARN:                 [REQUIRED]
LOG_RETENTION_DAYS:          [30 default; 365 for regulated]
ENABLE_GUARDDUTY_RUNTIME:    [true default for prod]

# --- OBSERVABILITY ---
SNS_ALARM_TOPIC_ARN:         [REQUIRED]
ENABLE_XRAY:                 [true default]
```

---

## Partial Library (Claude MUST load)

| Partial | Why |
|---|---|
| `ECS_CLUSTER_FOUNDATION` | Cluster + capacity providers + Service Connect + Container Insights v2 |
| `ECS_DEPLOYMENT_PATTERNS` | Rolling + Blue/Green CodeDeploy + canary + circuit breaker |
| `ECS_PRODUCTION_HARDENING` | Task IAM + auto-scaling + GuardDuty Runtime + ECR/Inspector + network |
| `LAYER_API` | (if API GW in front) REST/HTTP API + custom domain |
| `LAYER_OBSERVABILITY` | CW dashboards + alarms + X-Ray |
| `LAYER_NETWORKING` | VPC + endpoints baseline |
| `LAYER_SECURITY` | KMS + Secrets Manager rotation |
| `SERVERLESS_HTTP_API_COGNITO` | (if HTTP API → ECS) Cognito auth at edge |

---

## Architecture

```
   Browser ──► Route 53 (api.example.com)
                     │
                     ▼
              ┌────────────────────────────┐
              │ WAF v2                      │
              └────────┬───────────────────┘
                       │
                       ▼
              ┌────────────────────────────┐
              │ ALB (HTTPS only, TLS 1.3)   │
              │ Cognito auth (optional)      │
              │ Listener:443                  │
              │   90% → TG-Blue (current)    │
              │   10% → TG-Green (canary)    │   ← during deploy only
              │ Listener:9090 (test)          │
              │   100% → TG-Green (validate)  │
              └────────┬───────────────────┘
                       │
                       ▼
              ┌────────────────────────────┐
              │ ECS Cluster (prod-cluster)  │
              │   Capacity providers:        │
              │     FARGATE (60%, base 1)    │
              │     FARGATE_SPOT (40%)       │
              │   Container Insights v2      │
              │   Service Connect: prod.local│
              │                               │
              │   Service: api               │
              │     Tasks: 3-30 (auto-scale) │
              │     ARM64 Graviton            │
              │     1 vCPU + 2 GB             │
              │     ECS Exec enabled          │
              │     GuardDuty Runtime agent   │
              │     CodeDeploy blue/green     │
              │     Per-task IAM role         │
              │     Private subnets only      │
              └────────┬───────────────────┘
                       │ Service Connect
              ┌────────┼─────────────────────┐
              ▼        ▼                      ▼
         api → other.prod.local:8080 → other tasks
                       │
                       ▼
              ┌────────────────────────────┐
              │ AWS APIs via VPC endpoints  │
              │   (Secrets Manager, S3,      │
              │    DDB, KMS, Logs, etc.)     │
              └────────────────────────────┘
                       │
                       ▼
              ┌────────────────────────────┐
              │ Aurora PG / DDB              │
              │ (private subnets, KMS, PITR) │
              └────────────────────────────┘
```

---

## Day-by-day execution (2-3 day POC, 1 dev)

### Day 1 — Cluster + first service running
- KMS CMK
- ECS Cluster with Container Insights v2 + ECS Exec
- Capacity providers: FARGATE + FARGATE_SPOT (base 1 on FARGATE)
- Service Connect Cloud Map namespace (`{ENV}.local`)
- ECR repo for `CONTAINER_IMAGE_REPO` (KMS-encrypted, IMMUTABLE, scan-on-push)
- Per-service task role + execution role (least-privilege)
- Task definition (Fargate, ARM64, Powertools layer, healthCheck, Service Connect port mapping)
- ALB + 1-2 target groups (depending on rolling vs blue/green) + ACM cert + WAF
- ECS service with `desired_count=3` + Service Connect registration
- Route 53 alias to ALB
- **Deliverable:** End of Day 1: `curl https://api.example.com/healthz` returns 200; service registered in Service Connect.

### Day 2 — Deployment pattern + auto-scaling + observability
- (If `DEPLOYMENT_TYPE=blue_green`) CodeDeploy app + deployment group + pre/post-traffic Lambda hooks + CW alarms wired to auto-rollback
- (If `DEPLOYMENT_TYPE=rolling`) Circuit breaker + deployment_alarms config
- Auto-scaling: target tracking on `SCALING_METRIC` (CPU, memory, ALB req count); min/max capacity per inputs
- VPC endpoints: ecr.api, ecr.dkr, logs, secretsmanager, ssm, kms, monitoring, s3 (gateway)
- (If `ENABLE_GUARDDUTY_RUNTIME`) Enable GuardDuty Runtime Monitoring for ECS Fargate (auto-deploy agent)
- Inspector v2 enabled for ECR
- CW dashboard: cluster overview (CPU/mem/req/p99/5xx), per-service breakdown
- 6 CW alarms wired to SNS:
  - Service running count < desired (capacity loss)
  - Service CPU > 80% sustained 5 min
  - Service memory > 85% sustained 5 min
  - ALB 5xx rate > 1%
  - ALB p99 latency > SLO
  - GuardDuty CRITICAL finding for cluster
- **Deliverable:** End of Day 2: deploy v2 of image → blue/green canary 10% → 100% completes successfully with zero errors; auto-scale tested by load injection.

### Day 3 — Hardening + tests + handoff
- Pytest suite covering all validation criteria
- Runbook for: deploy, rollback, scale up/down manually, exec into task, view logs
- README with API URL, sample requests, dashboard URL
- Cost projection (Fargate hourly, Spot %, ALB, CW Logs)
- **Deliverable:** End of Day 3: full pytest pass; runbook signed off; handoff to ops.

---

## Validation criteria

- [ ] **Cluster ACTIVE** with Container Insights v2 = ENHANCED
- [ ] **Capacity provider strategy**: FARGATE base ≥ 1 + FARGATE_SPOT for cost
- [ ] **Service running count == desired count** (3+)
- [ ] **Service Connect** registered (Cloud Map shows `{service}.{env}.local`)
- [ ] **ALB health check** passing for all targets
- [ ] **HTTPS only**: HTTP redirects to HTTPS; TLS 1.3 SSL policy
- [ ] **WAF associated** with ALB; managed common rule set + rate limit
- [ ] **Per-service task role** scoped (no `Resource: *` for sensitive actions)
- [ ] **Task in private subnet** (no public IP)
- [ ] **VPC endpoints active** for ECR, Logs, Secrets Manager, KMS
- [ ] **CodeDeploy blue/green completes successfully** (test deploy)
- [ ] **Circuit breaker rolls back failed deploy** (test by deploying broken image)
- [ ] **Auto-scaling triggers** under load (verified via load test)
- [ ] **GuardDuty Runtime Monitoring** active; sample finding observed during simulated event
- [ ] **ECR Inspector** scanning images; zero CRITICAL CVEs in deployed image
- [ ] **ECS Exec** works: `aws ecs execute-command --cluster ... --task ... --container ... --command "/bin/sh"`
- [ ] **CW Logs** structured + KMS-encrypted + retention configured
- [ ] **6 alarms in OK state** baseline; firing on injected failures

---

## Common gotchas (claude must address proactively)

- **`max_healthy_percent: 200`** during deploy means temporary 2× cost — budget accordingly.
- **Fargate Spot 2-min interruption** — apps must SIGTERM gracefully + ALB deregistration delay must accommodate.
- **Service Connect adds Envoy sidecar** to each task (~50 MB image, ~256 MB RAM). Plan task memory.
- **NAT Gateway vs VPC endpoints** — VPC endpoints save 70% for AWS API calls; NAT Gateway needed for non-AWS internet.
- **CodeDeploy lifecycle hooks Lambda must be named `CodeDeployHook_*`** — case-sensitive.
- **Container Insights v2 enhanced costs more** at scale — disable per-container metrics if cluster has > 500 containers.
- **GuardDuty Runtime agent in Fargate** = ~50 MB image extension + 256 MB RAM. Plan task memory.
- **ARM64 (Graviton) Fargate** requires multi-arch image: `docker buildx build --platform linux/arm64,linux/amd64`.
- **Task role vs execution role** — never share. Execution role = ECS agent's role for image pull / log write. Task role = container's role for runtime AWS calls.
- **NON_BLOCKING log mode** is preferred — old default could hang container on logging backpressure.

---

## Output artifacts

1. **CDK stacks**:
   - `EcsFoundationStack` — cluster + capacity providers + ECR + Service Connect namespace
   - `EcsServiceStack` — task definition + service + ALB + target groups + auto-scaling
   - `EcsDeploymentStack` — CodeDeploy app + DG + alarms + Lambda hooks (if blue/green)
   - `EcsObservabilityStack` — dashboards + alarms + GuardDuty config
2. **Container Dockerfile** — Powertools layer + ARM64 multi-arch + non-root user + healthCheck
3. **CodeDeploy lifecycle hook Lambdas** (pre-traffic + post-traffic) — smoke tests
4. **Pytest suite** covering all validation criteria
5. **CloudWatch dashboard JSON** — cluster + service overview
6. **6 alarms YAML** — wired to SNS
7. **Runbook** (Markdown):
   - Deploy procedure (CodePipeline → CodeDeploy → CodeBuild)
   - Rollback procedure (manual + auto)
   - Scale up/down (UpdateService desired-count)
   - Exec into task for debugging
   - Force new deployment (e.g., to refresh secrets)
8. **Cost projection** — Fargate × spot mix + ALB + CW Logs + Inspector + GuardDuty Runtime
9. **Load test script** (k6 or Artillery) — validate auto-scaling
10. **README** — URL + sample curl + Cognito flow (if enabled) + dashboard URL

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 2.0 | 2026-04-27 | Initial composite template. Composes ECS_CLUSTER_FOUNDATION + DEPLOYMENT_PATTERNS + PRODUCTION_HARDENING. 2-3 day POC. Wave 16. |
