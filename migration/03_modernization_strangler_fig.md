<!-- Template Version: 2.0 | F369 Wave 13 (composite) | Composes: MIGRATION_HUB_STRATEGY + SERVERLESS_HTTP_API_COGNITO + LAYER_BACKEND_LAMBDA + LAYER_BACKEND_ECS + ENTERPRISE_NETWORK_HUB_TGW -->

# Template 03 — Modernization via Strangler Fig (Refactor Spaces · monolith decomposition · path-by-path microservice extraction)

## Purpose

Modernize a **legacy monolithic application into microservices** using the **Strangler Fig pattern**, with AWS Migration Hub Refactor Spaces orchestrating the routing. Stand up the umbrella + first 2 microservices in **4-6 weeks**; extract remaining endpoints over **6-18 months** at the customer's pace.

This is the **canonical "modernize monolith" engagement** — the "REFACTOR" leg of the 6R framework. Replaces 6-month rewrites with incremental, low-risk extractions.

Output: Refactor Spaces environment, first microservices (Lambda + ECS), routing config, observability, runbook for ongoing extractions.

---

## Role Definition

You are an expert AWS application modernization architect with deep expertise in:
- AWS Migration Hub Refactor Spaces (Environment + Application + Service + Route)
- Strangler Fig pattern for incremental monolith decomposition
- API Gateway as proxy + Transit Gateway as network fabric
- Microservice architecture (bounded contexts, sync vs async, anti-corruption layer)
- Q Developer / CodeWhisperer for assisted code refactor
- Distributed tracing (X-Ray) for split-monolith observability
- Database decomposition (shared DB → per-service DB) — the hardest part

Generate complete, production-deployable code. No TODOs.

---

## Context and Inputs

```
PROJECT_NAME:                [REQUIRED]
AWS_REGION:                  [REQUIRED]
ENV:                         [REQUIRED]

# --- MONOLITH ---
MONOLITH_LANGUAGE:           [REQUIRED — java | dotnet | nodejs | python | ruby | php]
MONOLITH_DEPLOYMENT:         [REQUIRED — ec2 | ecs | lambda | rds_for_db | mixed]
MONOLITH_URL:                [REQUIRED — public-facing URL of monolith]
MONOLITH_VPC_ID:             [REQUIRED]

# --- TARGET MICROSERVICES ---
FIRST_EXTRACTION_PATHS:      [REQUIRED — comma-separated; e.g. /api/users,/api/auth]
FIRST_SERVICE_RUNTIMES:      [comma-separated matching paths; e.g. lambda,ecs_fargate]
EXTRACTION_BACKLOG:          [REQUIRED — list of paths planned for future extraction]

# --- DATABASE STRATEGY ---
DB_DECOMPOSITION:            [REQUIRED — shared_db_anticorruption | event_streaming | snapshot_replicate | monolith_owns_until_last]

# --- INFRA ---
TRANSIT_GATEWAY_ID:          [REQUIRED — Refactor Spaces uses TGW for network fabric]
COGNITO_USER_POOL_ID:        [REQUIRED — re-use existing for auth]

# --- COMPLIANCE ---
KMS_KEY_ARN:                 [REQUIRED]
ENABLE_WAF:                  [true default]
ENABLE_XRAY:                 [true default — critical for split-monolith debugging]

# --- OBSERVABILITY ---
SNS_ALARM_TOPIC_ARN:         [REQUIRED]
EXISTING_DASHBOARD_ARNS:     [comma-separated — for monolith metrics already monitored]
```

---

## Partial Library (Claude MUST load)

| Partial | Why |
|---|---|
| `MIGRATION_HUB_STRATEGY` | Refactor Spaces sub-section + 6R framework |
| `SERVERLESS_HTTP_API_COGNITO` | (if Lambda extraction path) HTTP API + Cognito auth |
| `SERVERLESS_LAMBDA_POWERTOOLS` | (if Lambda) logger + tracer + idempotency |
| `LAYER_BACKEND_ECS` | (if ECS extraction path) ECS + Fargate base |
| `LAYER_BACKEND_LAMBDA` | Lambda 5 non-negotiables |
| `ENTERPRISE_NETWORK_HUB_TGW` | TGW network fabric (Refactor Spaces requires it) |
| `LAYER_API` | API Gateway base patterns |
| `LAYER_OBSERVABILITY` | X-Ray tracing across monolith + microservices |
| `EVENT_DRIVEN_PATTERNS` | EventBridge for monolith ↔ microservice async events |

---

## Architecture (initial state vs intermediate state vs end state)

### Initial — Monolith handles all
```
   Client ──► Old URL (e.g., app.example.com)
                     │
                     ▼
              Monolith on EC2 (java-app.jar, .NET assembly)
                     │
                     ▼
              Shared RDS (Oracle / SQL Server)
```

### Intermediate — Refactor Spaces routing 2 paths to new services
```
   Client ──► API Gateway (Refactor Space proxy at app.example.com)
                     │
        ┌────────────┼────────────────┬──────────────────────┐
        ▼            ▼                ▼                      ▼
   /api/users    /api/auth        /api/orders          /* (default)
   → Lambda     → ECS Fargate     → Monolith           → Monolith
   (new)        (new)             (still here)          (everything else)
                                       │                       │
                                       ▼                       ▼
   Per-service DBs (Aurora PG)              Shared monolith RDS

   Communication between new services + monolith:
     - Sync: HTTP via internal endpoints
     - Async: EventBridge (events.users-changed, events.auth-validated)
     - Anti-corruption layer: Lambda translates monolith events → microservice schemas
```

### End state — Monolith has no traffic
```
   Client ──► API Gateway (Refactor Space proxy)
                     │
        ┌────────────┼─────────────┬───────────────┬─────────┐
        ▼            ▼             ▼               ▼         ▼
   /api/users   /api/auth     /api/orders   /api/cart    /api/...
   (Lambda)     (ECS)         (ECS)          (Lambda)    (per service)
                     │            │              │            │
                     ▼            ▼              ▼            ▼
   Per-service Aurora / DynamoDB tables (each owns its data)

   Monolith → DECOMMISSIONED
```

---

## Day-by-day execution (4-6 week initial setup)

### Week 1 — Discovery + decomposition plan
- Map monolith to bounded contexts (DDD analysis)
- Identify extraction order: leaf nodes first (services with fewest dependencies)
- DB decomposition strategy decision (shared vs per-service)
- Stakeholder + dev team alignment on first 2 extractions

### Week 2 — Refactor Spaces setup + monolith as default service
- Refactor Spaces Environment (TGW network fabric)
- Refactor Spaces Application (API Gateway proxy)
- Default Service = monolith URL → all traffic still goes to monolith
- DNS swap: app.example.com now points to API Gateway proxy
- Validate: zero behavior change (proxy is transparent)

### Week 3 — First microservice extraction (e.g., /api/users)
- Build new users microservice (Lambda or ECS based on `FIRST_SERVICE_RUNTIMES`)
- Per-service database (Aurora PG or DynamoDB)
- Initial data sync from monolith DB (one-time DMS load OR snapshot replication)
- Add Refactor Spaces Service for new microservice
- Add Route: `/api/users/*` → new service (initially in `INACTIVE` state)
- Stage validation: hit `/api/users/...` directly on new service → behavior parity check

### Week 4 — Activate route for first extraction
- Activate `/api/users/*` route → 100% traffic now goes to new service (or canary 10% then 100%)
- Monitor X-Ray traces for trace coverage across split path
- Monitor CloudWatch alarms for error rate / latency increase
- App-side behavior parity tests (golden suite of 100 user flows)
- If GO: keep route active; if NO-GO: deactivate route → traffic returns to monolith

### Week 5 — Anti-corruption layer + events
- For monolith → microservice events (e.g., monolith updates user → notify users-svc cache):
  - Monolith publishes domain event (CDC from monolith DB or app-emitted to EventBridge)
  - Anti-corruption layer Lambda transforms monolith schema → microservice schema
  - Microservice consumes
- For microservice → monolith events: same pattern reverse
- Test event flows end-to-end

### Week 6 — Second extraction (e.g., /api/auth) + observability + runbook
- Repeat extraction process for second path
- Build extraction runbook for future endpoints (parameterized template):
  1. Build new microservice
  2. Replicate initial data
  3. Add Refactor Space Service + Route (INACTIVE)
  4. Stage validation
  5. Activate route (canary 10% → 50% → 100%)
  6. Monitor 1 week
  7. Confirm GO; remove monolith path code
- Set up extraction backlog tracking in Jira / linear
- **Deliverable:** Working production setup; team self-sufficient for next extractions.

---

## Validation criteria (per extraction)

- [ ] **Refactor Spaces Service status: ACTIVE**, Route status: ACTIVE
- [ ] **API Gateway proxy receives traffic** matching the route pattern
- [ ] **Behavior parity tests pass** (golden suite of N requests)
- [ ] **Latency p99 within 1.5× of monolith baseline** (microservice cold start counted)
- [ ] **Error rate within 0.1% of monolith baseline**
- [ ] **X-Ray trace shows full path** through Refactor Space → microservice → DB → response
- [ ] **CloudWatch alarms set** on new microservice (5xx, p99, throttles)
- [ ] **Per-service IAM scoped** (no shared role across services)
- [ ] **Per-service DB encrypted** with CMK
- [ ] **Monolith code removed** for this path (after 1 week stable)

## Validation criteria (program-level)

- [ ] **Refactor Spaces Environment ACTIVE** with TGW attachment
- [ ] **API Gateway proxy** in front of monolith URL
- [ ] **Zero downtime cutover** at proxy installation (DNS swap traffic not lost)
- [ ] **Extraction runbook documented + parameterized**
- [ ] **Q Developer / CodeWhisperer integrated** for ongoing refactor work
- [ ] **DB decomposition strategy in execution** (per-service DBs growing as extractions land)
- [ ] **Monolith retirement projection** — N months until last endpoint extracted

---

## Common gotchas (claude must address proactively)

- **Refactor Spaces requires Transit Gateway** as network fabric. If your VPC isn't TGW-attached, RS creates a TGW. Plan VPC IPs accordingly.
- **Stateful sessions break on path-split** — if monolith stores session in JSESSIONID cookie, requests to `/api/users` (now on Lambda) won't see it. Migrate session to Redis (ElastiCache) BEFORE first extraction.
- **Database decomposition is the hardest part.** Two strategies:
  - "Shared DB anti-corruption layer" — both monolith + new svc read same DB; new svc has its own connection + its own ORM, treats DB as legacy boundary. Pragmatic, technical-debt-laden.
  - "Per-service DB with event streaming" — each new svc gets new DB; events from monolith DB (via CDC + DMS) populate new DB. Cleaner long-term, more upfront work.
- **API Gateway proxy adds 50-100ms latency** vs direct monolith call. Critical paths may need direct routing (skip RS) — but loses RS observability.
- **Refactor Spaces Service `endpoint_type: URL`** for monolith — RS proxies HTTPS to the URL. SSL cert must be valid + reachable from RS.
- **Refactor Spaces Service `endpoint_type: LAMBDA` or `ECS`** for new services — RS handles auth + invocation directly.
- **Q Developer for stored proc translation** is best-effort. Always have human DBA review business logic translation.
- **Anti-corruption layer is mandatory** when domain models differ between monolith + new service. Don't share schemas across the boundary.
- **Monitoring during transition** — set up duplicate dashboards: monolith view (legacy metrics) + new service view (CloudWatch / X-Ray). Keep both for 30 days post-extraction.
- **Rollback strategy** — Route activation can be flipped INACTIVE in seconds. Practice this; have on-call ready during cutover.
- **Compliance drift** — new services in different runtimes may inherit different compliance posture. Re-validate PCI/HIPAA scope per service.

---

## Output artifacts

1. **CDK stacks**:
   - `RefactorSpacesStack` — Environment + Application + Default Service (monolith)
   - `MicroserviceStack` (parameterized) — Lambda OR ECS service + DB + Route
   - `AntiCorruptionLayerStack` — Lambda transformers for cross-boundary events
   - `EventBusStack` — EventBridge custom bus for monolith ↔ microservice events

2. **Refactor Spaces config**:
   - Environment (TGW network fabric, KMS encryption)
   - Application (API Gateway proxy, regional endpoint)
   - Per-service registration (URL, Lambda, or ECS endpoint)
   - Per-route activation control script

3. **First microservice scaffolds**:
   - Lambda Powertools-based microservice template
   - ECS Fargate task definition + service definition
   - Per-service Aurora PG or DynamoDB

4. **Anti-corruption layer**:
   - EventBridge rules for monolith → microservice events
   - Lambda transformers (Pydantic schemas for monolith + microservice domains)
   - DDB Streams + Pipes for monolith DB CDC (if shared DB strategy)

5. **Extraction runbook** (parameterized Markdown):
   - Pre-extraction checklist (behavior parity baseline, ownership, rollback plan)
   - Build microservice (using F369 partials)
   - Refactor Space registration steps
   - Route activation steps (canary → 100%)
   - Monolith code removal procedure

6. **Pytest validation suite**:
   - Behavior parity tests (golden suite running against monolith vs new service)
   - Refactor Space Service health checks
   - Route activation tests (INACTIVE → ACTIVE → traffic flow)
   - X-Ray trace coverage tests

7. **Dashboards**:
   - Monolith vs microservice traffic split (via Refactor Space metrics)
   - Latency comparison chart (monolith baseline vs microservice)
   - Error rate per route
   - Cost comparison (monolith hours vs microservice usage)

8. **Decomposition tracker**:
   - Mermaid diagram of current state (monolith + extracted services)
   - Backlog of remaining extractions with size estimates
   - Monolith retirement timeline projection

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 2.0 | 2026-04-27 | Initial composite template. Composes MIGRATION_HUB_STRATEGY (Refactor Spaces) + microservice partials. Strangler Fig pattern. Wave 13. |
