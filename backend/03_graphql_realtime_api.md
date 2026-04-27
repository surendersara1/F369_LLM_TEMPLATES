<!-- Template Version: 2.0 | F369 Wave 10 (composite) | Composes: LAYER_API_APPSYNC + SERVERLESS_DYNAMODB_PATTERNS + SERVERLESS_LAMBDA_POWERTOOLS + SERVERLESS_HTTP_API_COGNITO -->

# Template 03 — GraphQL Real-Time API (AppSync · Cognito · DynamoDB · Pipeline Resolvers · Subscriptions)

## Purpose

Stand up a production-grade GraphQL API with **real-time subscriptions** in **2-3 days**. AppSync handles the WebSocket transport, schema validation, query batching, and per-field authorization — replaces "REST + custom WebSocket server + custom auth" with a managed service.

Output: a GraphQL API with Cognito auth, DDB direct resolvers (no Lambda for trivial CRUD), pipeline resolvers for multi-step business logic, real-time subscriptions for collaborative UI, and an offline-capable schema (Amplify-compatible).

Generates production-deployable CDK + GraphQL schema + JS resolvers + Pytest suite.

---

## Role Definition

You are an expert AWS GraphQL architect with deep expertise in:
- AppSync (Merged APIs, JavaScript resolvers preferred over VTL since 2023)
- GraphQL schema design (connections, edges, pagination, field-level auth)
- AppSync resolvers — DDB direct, Lambda, HTTP, Pipeline (multi-step)
- AppSync caching (per-resolver TTL)
- Subscriptions over WebSocket — invalidation patterns + auth
- Cognito User Pool authorization mode + IAM mode + API key (for public reads only)
- DynamoDB single-table design (composes with `SERVERLESS_DYNAMODB_PATTERNS`)
- Amplify CLI / Amplify Gen 2 frontend integration

Generate complete, production-deployable code. No TODOs.

---

## Context and Inputs

```
PROJECT_NAME:                [REQUIRED]
AWS_REGION:                  [REQUIRED]
AWS_ACCOUNT_ID:              [REQUIRED]
ENV:                         [REQUIRED — dev | stage | prod]

# --- API ---
API_NAME:                    [REQUIRED — e.g. {project}-api]
HOSTED_ZONE_NAME:            [REQUIRED]
API_SUBDOMAIN:               [graphql default — final URL: graphql.{HOSTED_ZONE_NAME}]

# --- AUTH ---
DEFAULT_AUTH_MODE:           [USER_POOL default — Cognito JWT]
COGNITO_USER_POOL_ID:        [REQUIRED — created or existing from SERVERLESS_HTTP_API_COGNITO]
ADDITIONAL_AUTH_MODES:       [comma-separated optional — IAM, API_KEY, OIDC]
ENABLE_FIELD_AUTH:           [true default — @aws_auth(cognito_groups: [...]) directives]

# --- SCHEMA ---
ENTITIES:                    [REQUIRED — comma-separated; e.g. User,Project,Task,Comment]
RELATIONSHIPS:               [list of "Entity1 has_many Entity2" descriptions]
ENABLE_SUBSCRIPTIONS:        [true default — generate onCreated/onUpdated/onDeleted per entity]

# --- DATA ---
DDB_TABLE_NAME:              [REQUIRED — single-table from SERVERLESS_DYNAMODB_PATTERNS]
ENABLE_DDB_DIRECT_RESOLVERS: [true default — bypasses Lambda for simple CRUD]
ENABLE_PIPELINE_RESOLVERS:   [true default — for multi-step business logic]

# --- CACHING ---
ENABLE_CACHING:              [false default — true for read-heavy APIs]
CACHE_TTL_SECONDS:           [60 default if enabled]

# --- COMPLIANCE ---
KMS_KEY_ARN:                 [REQUIRED]
LOG_RETENTION_DAYS:          [30 default]
ENABLE_WAF:                  [true default for non-dev]

# --- OBSERVABILITY ---
SNS_ALARM_TOPIC_ARN:         [REQUIRED]
ENABLE_XRAY:                 [true default]
ENABLE_LOGGING:              [ALL default; ERROR for prod-quiet]
```

---

## Partial Library (Claude MUST load)

| Partial | Why |
|---|---|
| `LAYER_API_APPSYNC` | Canonical AppSync setup (data sources, resolvers, schema base) |
| `SERVERLESS_DYNAMODB_PATTERNS` | Single-table design — AppSync DDB resolvers compose with this |
| `SERVERLESS_LAMBDA_POWERTOOLS` | For Lambda data sources — Powertools layer + idempotency |
| `SERVERLESS_HTTP_API_COGNITO` | Cognito User Pool — re-use for AppSync auth mode |
| `LAYER_OBSERVABILITY` | CW dashboards + alarms |
| `LAYER_FRONTEND` | (optional) React + Amplify GraphQL client integration |

---

## Architecture

```
   React SPA (Amplify GraphQL client)
        │
        │ 1. Cognito Hosted UI sign-in → JWT
        │ 2. WebSocket: wss://graphql.example.com/realtime
        │    + Authorization: Bearer <JWT>
        ▼
   ┌──────────────────────────────────────────────────┐
   │  AppSync GraphQL API                             │
   │   - Auth: USER_POOL (default) + IAM (admin)       │
   │   - Schema with @aws_auth directives              │
   │   - WAF v2 attached                                │
   │   - X-Ray active                                   │
   │   - CloudWatch logs (FIELD_LEVEL ERROR)            │
   └────────────────┬─────────────────────────────────┘
                    │
       ┌────────────┼────────────────┬────────────────┐
       ▼            ▼                ▼                ▼
   ┌────────┐  ┌──────────────┐  ┌──────────┐   ┌─────────────┐
   │ DDB    │  │ Lambda       │  │ HTTP     │   │ Pipeline    │
   │ direct │  │ (complex     │  │ (REST    │   │ resolvers   │
   │ (CRUD) │  │  business    │  │ proxy)   │   │ (multi-step)│
   │        │  │  logic)      │  │          │   │             │
   └────────┘  └──────────────┘  └──────────┘   └─────────────┘
       │            │
       ▼            ▼
   ┌──────────────────────────┐
   │  DDB single-table         │
   │   - Streams enabled       │
   │   - Mutation publishes    │
   │     subscription event    │
   │     via DDB → Lambda →    │
   │     AppSync mutation API  │
   └──────────────────────────┘
                    │
                    ▼
   ┌──────────────────────────────────────────────────┐
   │  Subscriptions (WebSocket fan-out)               │
   │   - onProjectUpdated(id: ID!) — per-id push       │
   │   - onCommentCreated(taskId: ID!) — collab UI     │
   │   - Filter at subscription level (server-side)   │
   └──────────────────────────────────────────────────┘
```

---

## Day-by-day execution (3-day POC, 1 dev)

### Day 1 — Schema + auth + DDB direct resolvers
- KMS CMK
- AppSync API with Cognito User Pool default auth, IAM additional auth (admin operations)
- GraphQL schema (`schema.graphql`) covering all `ENTITIES` + relationships
- `@aws_auth` field-level directives (admin-only mutations, group-based read scopes)
- DDB data source bound to single table
- DDB direct resolvers (JavaScript runtime since 2023) for simple CRUD per entity:
  - `getEntity(id)` → GetItem
  - `listEntities(limit, nextToken)` → Query on PK
  - `createEntity(input)` → PutItem
  - `updateEntity(id, input)` → UpdateItem
  - `deleteEntity(id)` → DeleteItem
- Custom domain via Route 53 + ACM cert
- WAF v2 ACL with rate limit + AWSManagedRulesCommonRuleSet
- **Deliverable:** End of Day 1: Amplify Studio / Postman GraphQL queries hit `getEntity`/`listEntities` with valid JWT.

### Day 2 — Pipeline resolvers + Lambda data sources + subscriptions
- For each multi-step mutation (e.g., `createOrderWithLineItems`):
  - Pipeline resolver with N functions: validate → enrich → write → publish
- Lambda data source for entities requiring complex auth or external calls
  - Powertools layer + `@idempotent` for safe retries
- Subscriptions:
  - `onProjectUpdated(id: ID!)` — push to clients subscribed to that project
  - `onCommentCreated(taskId: ID!)` — collaborative UI updates
  - Filter expressions in resolver to enforce per-user visibility
- DDB Stream → Lambda → `appsync.graphql()` to publish to subscribers (when DDB direct mutation needs to fan-out)
- **Deliverable:** End of Day 2: Subscribe to `onProjectUpdated`, then mutate the project from another client → first client sees update push within 1s.

### Day 3 — Caching, observability, tests, frontend snippet
- (If `ENABLE_CACHING`) per-resolver caching at field level (60s TTL on `listEntities`)
- CloudWatch dashboard:
  - Operations/sec (queries, mutations, subscriptions)
  - Latency p50/p99 per operation
  - Subscription connection count
  - Error rate by resolver
  - DDB throttled requests
- Alarms:
  - Error rate > 1% over 15 min
  - p99 latency > 1s
  - WAF blocked count spike
  - Subscription disconnects spike
- Pytest suite with `gql` Python client:
  - All queries + mutations (happy path)
  - Auth — non-admin blocked from admin mutations
  - Subscription — receives event within 2s
  - Pagination — nextToken cycles correctly
  - Field-level auth — restricted fields hidden
- Frontend snippet (React + Amplify v6 GraphQL client) showing query, mutation, subscription
- **Deliverable:** Full suite passing + dashboard live + frontend can drive end-to-end.

---

## Validation criteria

- [ ] **Custom domain reachable**: `curl https://graphql.example.com/graphql` returns AppSync schema introspection (with auth)
- [ ] **JWT required**: query without `Authorization` header → 401
- [ ] **Field-level auth enforced**: non-admin user → cannot mutate admin-only fields (resolver returns AuthError)
- [ ] **DDB direct resolver works**: `getProject(id: "p1")` returns item from single-table
- [ ] **Subscription delivers**: subscribe to `onProjectUpdated(id: "p1")`, mutate from another session → push received within 2s
- [ ] **Pipeline resolver succeeds**: `createOrderWithLineItems` runs all steps in order, atomic on failure
- [ ] **Caching reduces DDB calls** (if enabled): same `listProjects` query within TTL → no DDB read in CW metrics
- [ ] **WAF active**: rate limit triggers 429 above threshold
- [ ] **CW logs FIELD_LEVEL ERROR**: errors logged with resolver name + variables
- [ ] **X-Ray traces show resolver chain** for pipeline resolvers
- [ ] **Schema introspection disabled in prod** (or restricted to admins)

---

## Common gotchas (claude must address proactively)

- **JavaScript resolvers (APPSYNC_JS) replaced VTL since 2023.** New work should use JS — don't author VTL unless maintaining legacy.
- **Subscription filter happens server-side via resolver, NOT client-side.** Naive `onProjectUpdated` without filter pushes every project update to every client → bandwidth + auth issues.
- **AppSync `@aws_subscribe(mutations: ["mutateProject"])` directive only fires when that mutation is invoked through AppSync.** External writes (DDB direct, Lambda) won't trigger subscriptions — must call `appsync.graphql()` mutation explicitly.
- **DDB `Streams` events arrive 1-3 sec after DDB write.** Subscription latency = DDB stream latency + Lambda + AppSync publish ≈ 2-5s. Real-time-but-eventual.
- **Pipeline resolvers limited to 10 functions.** Push extra steps into Lambda data source.
- **AppSync caching is per-resolver, not per-query.** Same field with different args = different cache entries. Ensure cache keys exclude PII.
- **WAF on AppSync requires the API to be `RequestType: GraphQLApi`** in the WAF association. Different from API Gateway.
- **Custom domain on AppSync uses CloudFront + regional cert** — cert must be in same region as AppSync (NOT us-east-1 unless deploying to us-east-1).
- **Field-level `@aws_auth` only works with Cognito User Pools as default auth.** With IAM as default, use `@aws_iam`.
- **Subscription connection limit per AppSync API: 100,000.** For high-fanout scenarios, partition across multiple APIs.
- **Lambda data source response must match GraphQL field type.** Mismatch = silent error in CW logs, GraphQL returns null.
- **AppSync log retention defaults to "Never expire"** — set explicitly via `aws logs put-retention-policy`.

---

## Output artifacts

1. **CDK Python app** — stacks: `DataStack` (DDB) → `AuthStack` (Cognito reuse) → `AppSyncStack` (API + schema + resolvers + WAF + custom domain)
2. **GraphQL schema** (`schema.graphql`) — entities + relationships + connections + auth directives + subscriptions
3. **JavaScript resolver files** (`resolvers/*.js`) — one per (Type, Field); pipeline resolvers split into functions
4. **Lambda source code** — one file per Lambda data source; Powertools + idempotency
5. **DDB → AppSync subscription publisher Lambda** — for off-AppSync mutations that need subscription fan-out
6. **Pytest suite** with `gql` client — covers all validation criteria
7. **CloudWatch dashboard JSON** — GraphQL operations overview
8. **5+ alarms YAML** — wired to SNS
9. **Frontend snippet** — React + Amplify v6 GraphQL with subscription hooks (sample useSubscription)
10. **README** — sample queries + sub examples + Amplify codegen instructions

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 2.0 | 2026-04-26 | Initial. AppSync + Cognito + DDB direct resolvers + JS pipeline resolvers + subscriptions + WAF. Wave 10. |
