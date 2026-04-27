<!-- Template Version: 2.0 | F369 Wave 10 (composite) | Composes: SERVERLESS_HTTP_API_COGNITO + SERVERLESS_LAMBDA_POWERTOOLS + SERVERLESS_DYNAMODB_PATTERNS + LAYER_BACKEND_LAMBDA + LAYER_DATA + LAYER_OBSERVABILITY -->

# Template 01 — Serverless API Starter (HTTP API · Cognito JWT · Lambda Powertools · DynamoDB single-table)

## Purpose

Stand up a production-grade serverless REST API in **2 days**. The output is a portfolio-ready API with auth, structured logging, traces, metrics, idempotency, and a single-table DynamoDB backend — what a senior team would build given a week, but in 2 days because the patterns are pre-codified.

This is **the canonical "serverless backend" engagement** — every consulting client with a new app build asks for this shape. Generates production-deployable CDK + Lambda code + Pytest suite + frontend integration snippet.

---

## Role Definition

You are an expert AWS serverless architect with deep expertise in:
- API Gateway HTTP API v2 (preferred over REST API for new builds)
- Cognito User Pools + JWT authorizer + advanced security mode + MFA
- Lambda Powertools v3 (Python) — logger, tracer, metrics, idempotency, parameters, batch, event parser, feature flags
- DynamoDB single-table design + GSI patterns + transactions + Streams + TTL
- Lambda + Lambda Layers + provisioned concurrency for cold-start sensitive paths
- Custom domain via Route 53 + ACM regional certs
- WAF v2 with managed rules + per-IP rate limit
- CloudWatch Logs Insights + dashboards + alarms

Generate complete, production-deployable code. No TODOs.

---

## Context and Inputs

```
PROJECT_NAME:                [REQUIRED]
AWS_REGION:                  [REQUIRED]
AWS_ACCOUNT_ID:              [REQUIRED]
ENV:                         [REQUIRED — dev | stage | prod]

# --- DOMAIN ---
HOSTED_ZONE_ID:              [REQUIRED]
HOSTED_ZONE_NAME:            [REQUIRED — e.g. example.com]
API_SUBDOMAIN:               [api default — final URL: api.{HOSTED_ZONE_NAME}]

# --- AUTH ---
COGNITO_MFA_MODE:            [OPTIONAL default — REQUIRED for prod]
COGNITO_DOMAIN_PREFIX:       [REQUIRED — globally unique; e.g. {project}-{env}]
ALLOWED_CALLBACK_URLS:       [comma-separated — e.g. https://app.example.com/callback]
ALLOWED_LOGOUT_URLS:         [comma-separated]
USER_GROUPS:                 [comma-separated — e.g. admin,editor,viewer]

# --- API ROUTES ---
RESOURCE_ENTITIES:           [REQUIRED — comma-separated singular nouns; e.g. order,product,customer]
ROUTE_PATTERNS:              [crud default — generates GET/POST/PUT/DELETE per entity]

# --- DATA ---
DDB_TABLE_NAME:              [REQUIRED]
ENABLE_STREAMS:              [true default — for downstream CDC]
ENABLE_GLOBAL_TABLES:        [false default — true if multi-region]
SECONDARY_REGIONS:           [comma-separated if global tables]

# --- COMPLIANCE ---
KMS_KEY_ARN:                 [REQUIRED — DDB + log group + Cognito encryption]
LOG_RETENTION_DAYS:          [30 default; 365 for regulated]
ENABLE_WAF:                  [true default for non-dev]
WAF_RATE_LIMIT_PER_IP_5MIN:  [2000 default]

# --- OBSERVABILITY ---
SNS_ALARM_TOPIC_ARN:         [REQUIRED]
ENABLE_XRAY:                 [true default]

# --- IDEMPOTENCY ---
ENABLE_IDEMPOTENCY:          [true default for mutating routes]
IDEMPOTENCY_TTL_SECONDS:     [3600 default]
```

---

## Partial Library (Claude MUST load)

| Partial | Why |
|---|---|
| `SERVERLESS_HTTP_API_COGNITO` | HTTP API + Cognito JWT + custom domain + WAF + throttling |
| `SERVERLESS_LAMBDA_POWERTOOLS` | Logger + Tracer + Metrics + Idempotency + Event Parser; Lambda Layer pattern |
| `SERVERLESS_DYNAMODB_PATTERNS` | Single-table design + GSI + transactions + PITR |
| `LAYER_BACKEND_LAMBDA` | 5 non-negotiables (KMS env, X-Ray, log retention, IAM least-priv, dead-letter) |
| `LAYER_DATA` | DDB base patterns (re-verify alignment) |
| `LAYER_OBSERVABILITY` | CW dashboards + alarms + correlation IDs |
| `LAYER_SECURITY` | KMS CMK + Secrets Manager rotation foundation |
| `LAYER_FRONTEND` | (optional) React + Cognito hosted UI integration if frontend in scope |

---

## Architecture

```
   Browser (React SPA)
        │
        │  1. OAuth2 Auth Code grant via Cognito Hosted UI
        ▼
   Cognito User Pool (advanced security ENFORCED, MFA optional/required)
        │
        │  2. JWT (id + access tokens, 1h TTL)
        ▼
   ┌────────────────────────────────────┐
   │  Route 53: api.example.com          │
   │  ACM cert (regional)                 │
   └────────────────┬───────────────────┘
                    │
                    ▼
   ┌────────────────────────────────────┐
   │  WAF v2 (rate limit + Common Rules) │
   └────────────────┬───────────────────┘
                    │
                    ▼
   ┌────────────────────────────────────┐
   │  HTTP API v2                       │
   │   - JWT authorizer (Cognito)        │
   │   - CORS preflight                  │
   │   - Stage access logs (JSON)        │
   │   - /healthz (no auth)              │
   │   - /{entity} (CRUD, JWT required)  │
   │   - /admin/* (admin group only)     │
   └────────────────┬───────────────────┘
                    │  Lambda proxy integration
                    ▼
   ┌────────────────────────────────────┐
   │  Lambda function (Python 3.12)     │
   │   - Powertools layer (managed AWS)  │
   │   - APIGatewayHttpResolver routes   │
   │   - @idempotent on POST/PUT/DELETE  │
   │   - @event_parser Pydantic validate │
   │   - X-Ray active tracing            │
   │   - log_retention 30d, KMS-encrypted│
   └────────────────┬───────────────────┘
                    │
                    ▼
   ┌────────────────────────────────────┐
   │  DynamoDB (single-table)            │
   │   - PK/SK + GSI1 (inverted index)   │
   │   - PITR + KMS CMK                  │
   │   - Streams → Lambda CDC fanout     │
   │   - TTL on session/cache items      │
   │   - Idempotency table (separate)    │
   └────────────────────────────────────┘
                    │
                    ▼
   ┌────────────────────────────────────┐
   │  CloudWatch                         │
   │   - Logs Insights (correlation IDs) │
   │   - Dashboard: req/s, p99, 4xx/5xx, │
   │     Cognito sign-in failures        │
   │   - Alarms: 5xx rate, p99 > 1s,     │
   │     WAF blocked rate spike          │
   └────────────────────────────────────┘
```

---

## Day-by-day execution (2-day POC, 1 dev)

### Day 1 — Foundation + Cognito + DDB + Lambda + first route
- VPC review (no VPC needed for HTTP API + Lambda + DDB unless private subnets required)
- KMS CMK created (or imported)
- Cognito User Pool: advanced security ENFORCED, MFA optional, hosted UI domain, OAuth2 client (SRP, no secret)
- Cognito groups: admin, editor, viewer (or per `USER_GROUPS`)
- DDB single table: PK/SK, GSI1, PITR on, Streams enabled, KMS-encrypted, deletion protection on
- Idempotency DDB table (separate, TTL attribute)
- Powertools Lambda Layer (managed AWS ARN per region)
- Lambda function with `APIGatewayHttpResolver` + 3 routes (`/healthz`, `/{first_entity}` GET, POST)
- HTTP API + JWT authorizer + CORS preflight + custom domain + Route 53 alias + ACM cert
- WAF v2 ACL with AWSManagedRulesCommonRuleSet + rate limit
- **Deliverable:** End of Day 1: `curl https://api.example.com/healthz` returns 200; `POST /orders` with valid JWT creates a record in DDB; `GET /orders` returns list.

### Day 2 — Remaining CRUD + idempotency + admin + observability + tests
- For each entity in `RESOURCE_ENTITIES`:
  - Generate GET (collection), POST, GET (item), PUT, DELETE handlers
  - Apply `@idempotent` on POST/PUT/DELETE with key derived from request body
  - Apply `@event_parser` Pydantic models per route
- `/admin/{proxy+}` route — JWT claim group check
- CloudWatch dashboard + 5 alarms (4xx rate, 5xx rate, p99 > 1s, Cognito failed sign-ins, WAF blocked count spike)
- Pytest suite: smoke, auth, CRUD, idempotency replay, CORS, rate limit
- README with `npm run dev` + Cognito Hosted UI URL + sample fetch snippet for frontend
- **Deliverable:** End of Day 2: all routes working, dashboard live, ≥ 90% test pass, frontend can call API end-to-end.

---

## Validation criteria

- [ ] **Custom domain reachable**: `curl https://api.example.com/healthz` → 200
- [ ] **Auto endpoint disabled**: `curl https://*.execute-api.region.amazonaws.com/...` → 403 or 404
- [ ] **JWT required**: `curl https://api.example.com/orders` (no auth) → 401
- [ ] **JWT works**: `curl -H 'Authorization: Bearer <token>' .../orders` → 200
- [ ] **CORS**: OPTIONS preflight returns correct `Access-Control-*` headers
- [ ] **Admin guard**: non-admin user → 403 on `/admin/*`
- [ ] **Idempotency**: two identical POSTs in idempotency window → same response, ONE DDB write
- [ ] **WAF active**: 2000+ req/IP in 5 min → 429 returned
- [ ] **Logs structured**: CW Logs Insights query `fields @timestamp, level, correlation_id, message | sort @timestamp desc` returns JSON-parsed rows
- [ ] **X-Ray traces present**: Trace map shows API GW → Lambda → DDB
- [ ] **EMF metrics in dashboard**: `Checkout/OrderProcessed`, `Checkout/ColdStart` visible
- [ ] **Cognito advanced security ENFORCED**: `aws cognito-idp describe-user-pool --query 'UserPool.UserPoolAddOns.AdvancedSecurityMode'` → `ENFORCED`
- [ ] **DDB PITR ON + deletion_protection ON** (verified via `aws dynamodb describe-table`)

---

## Common gotchas (claude must address proactively)

- **Cognito MFA cannot be enabled retroactively** without forcing all users to re-enroll. Decide upfront for prod.
- **HTTP API JWT cache = 1 hour** — token revoke not instant. For instant revoke, fall back to Lambda authorizer + DDB session check.
- **Idempotency key derivation must be deterministic.** Don't use `timestamp` or auto-generated `request_id` — use a business key (`order_id`).
- **`disable_execute_api_endpoint=false` (default) leaks API past WAF + custom domain.** Set `true` in production.
- **CORS `allow_credentials=true` requires explicit origins**. Browser silently rejects credentials-mode XHR with `*`.
- **DDB on-demand can hit hot-partition throttling** even on PAY_PER_REQUEST during massive bursts. Mitigate with shard prefixes in PK.
- **Lambda Layer ARN is region-specific.** Use the region map in `SERVERLESS_LAMBDA_POWERTOOLS` §4.1.
- **Log retention defaults to "Never expire"** ($$$). Always set `log_retention=Duration.days(N)`.

---

## Output artifacts

1. **CDK Python app** — stacks: `NetworkStack` (optional) → `DataStack` (DDB + idempotency table) → `AuthStack` (Cognito) → `ApiStack` (HTTP API + Lambda + WAF + custom domain)
2. **Lambda source code** — `src/api/handler.py` with APIGatewayHttpResolver, Pydantic models per route, `@idempotent` decorators
3. **Pydantic models** — `src/api/models.py` (one per `RESOURCE_ENTITIES`)
4. **DDB repository layer** — `src/api/repositories.py` with single-table CRUD per entity
5. **Pytest suite** (`tests/test_api.py`) — covers all validation criteria
6. **CloudWatch dashboard JSON** — req/s, latency p50/p99, 4xx/5xx, Cognito events, WAF blocked
7. **5 CloudWatch alarms** YAML — wired to `SNS_ALARM_TOPIC_ARN`
8. **Frontend integration snippet** — JavaScript fetch with Cognito JWT (sample React hook)
9. **README** — quickstart + curl examples + Hosted UI sign-up flow + dashboard URL
10. **Cognito user-import CSV template** — bulk import format

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 2.0 | 2026-04-26 | Initial. HTTP API + Cognito + Powertools + DDB single-table + WAF + idempotency. 2-day POC. Wave 10. |
