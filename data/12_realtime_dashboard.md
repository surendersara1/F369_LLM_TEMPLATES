<!-- Template Version: 2.0 | F369 Wave 12 (composite) | Composes: DATA_KINESIS_STREAMS_FIREHOSE + DATA_MANAGED_FLINK + LAYER_API_APPSYNC + SERVERLESS_DYNAMODB_PATTERNS + LAYER_FRONTEND -->

# Template 12 — Real-Time Dashboard (Kinesis · Flink · DDB Streams · AppSync subscriptions · React UI)

## Purpose

Stand up a **collaborative real-time UI in 3-4 days** — events flow Kinesis → Flink (aggregate per-1s windows) → DDB → AppSync subscriptions → React dashboard updates within 2 seconds.

Use cases this template fits:
- Live ops dashboards (orders/min, active users, system health)
- Trading floor / market data UI
- Multiplayer game leaderboards / live scores
- Auction live bid feed
- Live election / sports tracking
- Collaborative editing (operational transform variant)

Generates production-deployable CDK + Flink job + AppSync schema + React component.

---

## Role Definition

You are an expert AWS real-time UX architect with deep expertise in:
- Kinesis Data Streams + Managed Flink for sub-second aggregation
- DynamoDB Streams as canonical CDC for downstream subscriptions
- AppSync GraphQL subscriptions over WebSocket
- React 19 + Amplify v6 GraphQL client + useSubscription hooks
- Optimistic UI updates with React Query / TanStack Query
- WebSocket scaling considerations (AppSync 100k connection limit)

Generate complete, production-deployable code. No TODOs.

---

## Context and Inputs

```
PROJECT_NAME:                [REQUIRED]
AWS_REGION:                  [REQUIRED]
ENV:                         [REQUIRED — dev | stage | prod]

# --- DASHBOARD SHAPE ---
DASHBOARD_TYPE:              [REQUIRED — ops_metrics | trading | leaderboard | live_feed | auction | custom]
METRICS:                     [REQUIRED — comma-separated; e.g. orders_per_min,active_users,p95_latency]
AGGREGATION_WINDOW_SEC:      [1 default — 1s tumbling window]
SUBSCRIPTION_FILTER:         [optional — e.g. by user_id, by region, by symbol]

# --- INGEST ---
EVENT_SOURCE:                [REQUIRED — kinesis | http_api | iot_core | webhook]
PEAK_EVENTS_PER_SECOND:      [REQUIRED]

# --- UI ---
EXPECTED_CONCURRENT_VIEWERS: [REQUIRED — drives AppSync sizing]
HOSTED_ZONE_NAME:            [REQUIRED]
APP_SUBDOMAIN:               [dashboard default]
COGNITO_USER_POOL_ID:        [REQUIRED]
COGNITO_USER_POOL_CLIENT_ID: [REQUIRED]

# --- COMPLIANCE ---
KMS_KEY_ARN:                 [REQUIRED]
ENABLE_WAF:                  [true default]

# --- OBSERVABILITY ---
SNS_ALARM_TOPIC_ARN:         [REQUIRED]
```

---

## Partial Library (Claude MUST load)

| Partial | Why |
|---|---|
| `DATA_KINESIS_STREAMS_FIREHOSE` | Event ingestion (no Firehose needed; just KDS) |
| `DATA_MANAGED_FLINK` | Per-second aggregation windows |
| `SERVERLESS_DYNAMODB_PATTERNS` | DDB single-table for live state + Streams enabled |
| `LAYER_API_APPSYNC` | GraphQL API + subscriptions |
| `LAYER_FRONTEND` | React + CloudFront + OAC for static hosting |
| `SERVERLESS_HTTP_API_COGNITO` | Cognito User Pool re-use for AppSync auth |
| `SERVERLESS_LAMBDA_POWERTOOLS` | Lambda fanout (DDB Stream → AppSync mutation) |

---

## Architecture

```
   Producers (server backends, IoT, etc.)
        │
        ▼
   ┌─────────────────────────────────────┐
   │ Kinesis Data Stream (events)         │
   └────────────────┬─────────────────────┘
                    │
                    ▼
   ┌─────────────────────────────────────┐
   │ Managed Flink Application            │
   │  - 1s tumbling windows               │
   │  - Per-metric aggregation             │
   │  - Output stream → DDB writer Lambda  │
   │     OR direct DDB sink via async I/O  │
   └────────────────┬─────────────────────┘
                    │
                    ▼
   ┌─────────────────────────────────────┐
   │ DynamoDB single-table                │
   │   PK = "DASHBOARD#{type}"            │
   │   SK = "METRIC#{name}#{window_end}"  │
   │   Streams: NEW_AND_OLD_IMAGES         │
   └────────────────┬─────────────────────┘
                    │ DDB Stream → Lambda
                    ▼
   ┌─────────────────────────────────────┐
   │ Fanout Lambda (Powertools batch)    │
   │  For each new metric record:         │
   │   - Call AppSync mutation            │
   │     publishMetric(...)               │
   │   - This triggers @aws_subscribe to  │
   │     fan out to all subscribers        │
   └────────────────┬─────────────────────┘
                    │ AppSync API call
                    ▼
   ┌─────────────────────────────────────┐
   │ AppSync GraphQL API                  │
   │   - Cognito User Pool auth            │
   │   - Schema:                            │
   │     type Metric {                      │
   │       name: String!                    │
   │       value: Float!                    │
   │       windowEnd: AWSDateTime!          │
   │     }                                   │
   │     type Mutation {                    │
   │       publishMetric(...) — IAM only    │
   │     }                                   │
   │     type Subscription {                │
   │       onMetricUpdate(name: String!)    │
   │         @aws_subscribe(mutations:      │
   │           ["publishMetric"])           │
   │     }                                   │
   │   - WAF v2 attached                    │
   │   - Custom domain                      │
   └────────────────┬─────────────────────┘
                    │ WebSocket
                    ▼
   ┌─────────────────────────────────────┐
   │ React 19 SPA (Amplify v6)            │
   │   - Cognito Hosted UI sign-in         │
   │   - useSubscription per metric        │
   │   - Recharts / Visx for live charts   │
   │   - Optimistic UI for user actions    │
   │   - Cached at CloudFront edge          │
   └─────────────────────────────────────┘
```

---

## Day-by-day execution (3-4 day POC, 1 dev)

### Day 1 — Kinesis + Flink + DDB
- KMS CMK + Cognito User Pool (re-use from existing if present)
- Kinesis Data Stream (on-demand)
- DDB single-table with Streams (NEW_AND_OLD_IMAGES) + KMS encrypted
- Managed Flink Application with Flink SQL job:
  - Source: KDS events
  - 1s tumbling windows per (metric_name)
  - Aggregation: SUM/AVG/MAX/MIN/COUNT_DISTINCT per `METRICS`
  - Sink: KDS aggregates stream
- DDB writer Lambda — consume Flink output stream → write to DDB
  - PK: `DASHBOARD#{type}`, SK: `METRIC#{name}#{window_end}`
- **Deliverable:** End of Day 1: producer publishes raw events → DDB shows aggregated metrics within 2-3s.

### Day 2 — AppSync + subscription fanout
- AppSync GraphQL API + Cognito auth + WAF + custom domain
- Schema: Metric type + publishMetric mutation (IAM-only) + onMetricUpdate subscription with filter
- DDB Stream → Lambda (Powertools batch processor)
  - For each new metric: call `mutation publishMetric(...)` via AppSync API (IAM-signed)
- Subscription resolver with filter expression: `name == $name`
- Test: subscribe via Postman → publish event → see push within 2s
- **Deliverable:** End of Day 2: subscription receives live updates within 2s of producer event.

### Day 3 — React UI + CloudFront
- React 19 SPA scaffolded with Vite + Amplify v6
- Cognito Hosted UI sign-in flow
- Custom hooks: `useMetricSubscription(metricName)` returning latest value + chart history
- Components per `DASHBOARD_TYPE`:
  - **ops_metrics**: KPI tiles + sparklines + alarm bell on threshold cross
  - **trading**: candlestick chart + ladder + order book
  - **leaderboard**: animated rank changes
  - **live_feed**: virtualized infinite scroll feed
  - **auction**: bid timer + live bid history + you-leading indicator
- S3 bucket + CloudFront distribution + OAC + ACM cert for `APP_SUBDOMAIN.HOSTED_ZONE_NAME`
- **Deliverable:** End of Day 3: dashboard live at `https://dashboard.example.com` showing real-time metrics; opening 2 browser tabs → both update simultaneously within 2s.

### Day 4 — Hardening + load test + tests
- Throttle the source to verify Flink scaling
- 6 CloudWatch alarms:
  - KDS `IncomingRecords` drop
  - Flink restarts > 0
  - DDB Streams iterator age > 60s
  - Fanout Lambda errors > 1%
  - AppSync `5xx` > 1%
  - WebSocket disconnects spike
- Load test: simulate 1000 concurrent viewers + 10× peak event rate
- Pytest + Cypress E2E:
  - Producer event → metric appears in subscription within 3s
  - Subscription auto-reconnects on network blip
  - Optimistic UI rollback on mutation failure
- **Deliverable:** Pass load test; document ops runbook (scaling, debugging slow updates).

---

## Validation criteria

- [ ] **End-to-end latency ≤ 3s** from producer PutRecord to React UI update
- [ ] **Multiple subscribers update simultaneously** (test with 2 browsers)
- [ ] **Subscription auto-reconnects** after network blip (≤ 10s)
- [ ] **Filter works** — subscribing to `metric=orders_per_min` doesn't receive `metric=cpu`
- [ ] **Load test sustains target** — 1000 concurrent viewers, 10× peak events
- [ ] **All 6 alarms in OK state** baseline
- [ ] **Cognito sign-in flow works** end-to-end
- [ ] **CloudFront cache headers correct** — HTML no-cache, JS/CSS 1y cache
- [ ] **AppSync logs show subscription registrations** + dispatches
- [ ] **Flink restart from snapshot < 30s** (recovery test)

---

## Common gotchas (claude must address proactively)

- **AppSync subscription connection limit = 100,000 per API.** For high-fanout, partition across multiple APIs or use one-to-many fan-out via SNS Fan-out + WebSocket API Gateway as alternative.
- **DDB Stream → Lambda has 1-3s lag** + Lambda cold start + AppSync publish latency. End-to-end "real-time" ≈ 2-5s, not sub-second. Set expectations.
- **Subscription filter happens server-side via VTL/JS resolver** — naive `onMetricUpdate` without filter pushes EVERY metric to EVERY client → bandwidth + auth issues.
- **AppSync mutation called from Lambda** requires IAM auth + IAM as additional auth mode on the API. Easy to misconfigure.
- **WebSocket connections cost** = $0.08/M connection minutes ($0.0048/connection/hr). 1000 viewers × 8h × 30 days = ~$1150/mo.
- **Flink 1s windows are aggressive** — overhead from per-window state mgmt. For < 1s, use stateless transformation only.
- **DDB write capacity for high-frequency updates** — 1 write/sec per metric × 100 metrics = 100 writes/sec. On-demand handles fine; provisioned needs auto-scaling.
- **React subscription cleanup is critical** — unmounted component subscriptions accumulate → memory leaks. Use Amplify v6 `subscription.unsubscribe()` in useEffect cleanup.
- **CloudFront + AppSync subscription requires custom origin**, not S3 origin — different setup from static site.
- **Optimistic UI rollback on mutation failure** is non-trivial — use TanStack Query's onError + rollback or write a custom hook.
- **Cognito session expires after 1h** — subscription disconnects. Implement token refresh + reconnect logic.

---

## Output artifacts

1. **CDK stacks** — `EventStreamStack`, `FlinkStack`, `DataStack` (DDB), `AppSyncStack`, `WebStack` (S3+CF)
2. **Flink job** — SQL aggregation per `METRICS`
3. **AppSync GraphQL schema** + JS resolvers + filter expression
4. **DDB Stream → Fanout Lambda** with Powertools batch
5. **React 19 SPA** — Vite scaffold, Amplify v6, Cognito Hosted UI integration, useSubscription custom hook, dashboard components per `DASHBOARD_TYPE`
6. **Recharts / Visx components** — KPI tile, sparkline, candlestick, leaderboard
7. **Cypress E2E tests** — sign-in + subscribe + update + reconnect
8. **Load test script** — k6 or Artillery — 1000 viewers
9. **CloudWatch dashboard** — pipeline + AppSync overview
10. **6 alarms YAML** — wired to SNS
11. **README** — quick start + Cognito setup + sample event payloads

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 2.0 | 2026-04-26 | Initial. KDS + Flink 1s windows + DDB Streams + AppSync subscriptions + React 19 + Amplify v6. Wave 12. |
