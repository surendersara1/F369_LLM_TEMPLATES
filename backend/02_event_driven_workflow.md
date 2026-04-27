<!-- Template Version: 2.0 | F369 Wave 10 (composite) | Composes: WORKFLOW_STEP_FUNCTIONS + EVENT_DRIVEN_PATTERNS + EVENT_DRIVEN_FAN_IN_AGGREGATOR + SERVERLESS_LAMBDA_POWERTOOLS + SERVERLESS_DYNAMODB_PATTERNS + DATA_EVENTBRIDGE_PIPES -->

# Template 02 — Event-Driven Workflow (EventBridge · Step Functions · SQS · Lambda · DynamoDB)

## Purpose

Stand up a durable, reactive event-driven backend in **2-3 days**. Replaces brittle "tightly coupled microservices" with EventBridge as the spine + Step Functions for stateful workflows + SQS for buffering. The output handles failure gracefully, retries with backoff, dead-letters poison messages, and emits events for downstream consumers.

Common use cases this template fits:
- Order processing (order placed → inventory check → payment → fulfilment → notification)
- Document processing (upload → OCR → classify → index → notify)
- User onboarding (sign-up → email verify → KYC → provision → welcome)
- Webhook fan-out (vendor webhook → enrich → fan-out to 5 consumers)

Generates production-deployable CDK + Step Function ASL + Lambda code.

---

## Role Definition

You are an expert AWS event-driven architect with deep expertise in:
- EventBridge custom event buses + rules + cross-account targets + schema discovery
- EventBridge Pipes (DDB Streams / Kinesis / SQS / MSK source → enrich → target)
- Step Functions (Standard vs Express; ASL constructs Map/Choice/Parallel/Retry/Catch; .sync vs callback patterns)
- SQS (standard vs FIFO; long polling; redrive policy + DLQ; SQS-Lambda partial-batch failure)
- Lambda Powertools batch processor for poison-pill handling
- DynamoDB Streams as canonical CDC mechanism
- Idempotency at every step boundary

Generate complete, production-deployable code. No TODOs.

---

## Context and Inputs

```
PROJECT_NAME:                [REQUIRED]
AWS_REGION:                  [REQUIRED]
AWS_ACCOUNT_ID:              [REQUIRED]
ENV:                         [REQUIRED — dev | stage | prod]

# --- WORKFLOW SHAPE ---
WORKFLOW_TYPE:               [REQUIRED — order_processing | doc_processing | user_onboarding | webhook_fanout | custom]
WORKFLOW_STAGES:             [REQUIRED — comma-separated; e.g. validate,enrich,price,reserve,charge,fulfill,notify]
WORKFLOW_TIMEOUT_HOURS:      [1 default; 720 max for Standard; 5 min for Express]
SF_TYPE:                     [STANDARD default — for long-running with audit; EXPRESS for high-throughput short]

# --- EVENT BUS ---
CUSTOM_BUS_NAME:             [REQUIRED — e.g. {project}-{env}-events]
EXTERNAL_PUBLISHERS:         [comma-separated AWS account IDs allowed to PutEvents]
EVENT_SCHEMAS_REGISTRY:      [true default — auto-register event schemas via Discoverer]

# --- TRIGGERS ---
TRIGGER_TYPE:                [REQUIRED — http_api | sqs | s3_event | ddb_stream | scheduled | webhook]
TRIGGER_DETAILS:             [varies per type]

# --- COMPLIANCE ---
KMS_KEY_ARN:                 [REQUIRED]
LOG_RETENTION_DAYS:          [30 default]

# --- OBSERVABILITY ---
SNS_ALARM_TOPIC_ARN:         [REQUIRED]
ENABLE_XRAY:                 [true default]

# --- ERROR HANDLING ---
RETRY_MAX_ATTEMPTS:          [3 default per step]
DLQ_RETENTION_DAYS:          [14 default — max for SQS]
ALARM_ON_DLQ_DEPTH:          [1 default — alarm on first message]
```

---

## Partial Library (Claude MUST load)

| Partial | Why |
|---|---|
| `WORKFLOW_STEP_FUNCTIONS` | Standard vs Express, ASL constructs, .sync vs callback, retry/catch |
| `EVENT_DRIVEN_PATTERNS` | Custom bus + rules + cross-account + cross-stack ARN pattern |
| `EVENT_DRIVEN_FAN_IN_AGGREGATOR` | Fan-in aggregation when steps run in parallel |
| `DATA_EVENTBRIDGE_PIPES` | DDB Stream / SQS source → enrich Lambda → target |
| `SERVERLESS_LAMBDA_POWERTOOLS` | Batch processor, idempotency at each step |
| `SERVERLESS_DYNAMODB_PATTERNS` | Workflow state in DDB single-table; TransactWrite for ACID across steps |
| `LAYER_BACKEND_LAMBDA` | Lambda 5 non-negotiables |
| `LAYER_OBSERVABILITY` | CW alarms on DLQ depth, SF execution failures, Lambda errors |

---

## Architecture (order processing example)

```
   Trigger (HTTP API / SQS / S3 / Schedule)
        │
        ▼
   ┌──────────────────────────────────────────────┐
   │  EventBridge custom bus: events              │
   │   Rule: order.placed → SF target              │
   └────────────┬─────────────────────────────────┘
                │
                ▼
   ┌──────────────────────────────────────────────┐
   │  Step Function (Standard, audit retained)    │
   │                                               │
   │  ┌──────────────┐   Choice (validation pass?)│
   │  │ ValidateOrder│──┬──── No → SendRejection  │
   │  └──────────────┘  │                          │
   │                    └──── Yes                  │
   │                          ▼                    │
   │                   ┌──────────────┐            │
   │                   │ EnrichCustomer│           │
   │                   └──────┬───────┘            │
   │                          ▼                    │
   │                   Parallel:                   │
   │                   ┌─────────────┐ ┌─────────┐ │
   │                   │ReserveStock │ │PriceCalc│ │
   │                   └──────┬──────┘ └────┬────┘ │
   │                          └────┬────────┘      │
   │                               ▼                │
   │                       ┌──────────────┐         │
   │                       │ ChargePayment│         │
   │                       └──────┬───────┘         │
   │                              ▼                 │
   │                       ┌──────────────┐         │
   │                       │ Fulfill (.sync)│       │
   │                       │ → ECS task    │        │
   │                       └──────┬───────┘         │
   │                              ▼                 │
   │                       ┌──────────────┐         │
   │                       │ PutEvent:    │         │
   │                       │ order.completed│       │
   │                       └──────────────┘         │
   │                                                │
   │  Catch (any step) → SendRollback + DLQ        │
   └──────────────────────────────────────────────┘
                │
                ▼
   ┌──────────────────────────────────────────────┐
   │  EventBridge: order.completed                 │
   │   ├── Rule → SES email notification           │
   │   ├── Rule → SNS warehouse                    │
   │   ├── Rule → Firehose → S3 analytics          │
   │   └── Rule → Cross-account target (CRM)       │
   └──────────────────────────────────────────────┘
```

---

## Day-by-day execution (3-day POC, 1 dev)

### Day 1 — Event bus + state store + first 2 steps
- KMS CMK
- Custom EventBridge bus + archive (7d) + schema discoverer
- DDB single-table for workflow state
- DLQ (SQS) per Lambda + per Step Function (with KMS, redrive 14d)
- Powertools Lambda Layer
- First 2 step Lambdas (ValidateOrder, EnrichCustomer) with `@idempotent`, `@event_parser`
- Step Function skeleton (Standard) with first 2 states + Catch → DLQ
- EventBridge rule: `source=app.orders, detail-type=order.placed → SF execute`
- **Deliverable:** End of Day 1: PutEvent → SF execution starts, runs 2 steps, success/failure visible in console.

### Day 2 — Remaining steps + parallel + .sync integration
- Remaining step Lambdas per `WORKFLOW_STAGES`
- Parallel state for steps that can run concurrently (e.g., ReserveStock + PriceCalc)
- `.sync` task pattern for ECS/Batch jobs (long-running)
- Callback pattern for human-approval steps (`.waitForTaskToken`)
- Catch handlers per step + Choice for retry-vs-rollback
- Per-step retry policy with exponential backoff (max 3 attempts default)
- Idempotency keys derived from input business keys (NOT execution ID — execution can be replayed)
- **Deliverable:** Full happy-path workflow executes end-to-end in < 5 min for sample input.

### Day 3 — Outputs, observability, failure tests
- Final step PutEvent (`order.completed` or per workflow type)
- Downstream rules: SES/SNS/Firehose/cross-account targets per `WORKFLOW_TYPE`
- CloudWatch dashboard:
  - SF execution count (started, succeeded, failed, timed-out)
  - SF state-level duration histograms
  - Lambda concurrent executions, throttles, errors
  - DLQ depth per queue
  - EventBridge rule invocation count + failed invocations
- Alarms:
  - DLQ depth ≥ 1 (any DLQ)
  - SF failure rate > 5% over 15 min
  - Step duration p99 > stage SLA
  - EventBridge rule failed invocations > 0
- Failure tests:
  - Inject malformed event → DLQ + alarm
  - Inject downstream failure (e.g., payment 500) → retry → eventual rollback
  - Long-running step timeout → Catch → rollback
- **Deliverable:** End of Day 3: workflow handles 5 failure modes correctly + alarm fires + downstream events reach all consumers.

---

## Validation criteria

- [ ] **Bus event accepted**: `aws events put-events --entries '[{"Source":"app.orders","DetailType":"order.placed","EventBusName":"<bus>","Detail":"{...}"}]'` → SF execution starts within 5s
- [ ] **Happy path**: full workflow completes successfully for known-good input
- [ ] **Idempotency**: replay same event 3× → 1 SF execution + 1 DDB write per step (verified via DDB query)
- [ ] **Retry on transient**: inject 503 from payment mock → SF retries 3× → eventually succeeds OR clean rollback
- [ ] **DLQ on poison-pill**: malformed input → after retries exhausted → DLQ + CW alarm fires
- [ ] **Parallel steps**: ReserveStock + PriceCalc execute concurrently (verified via SF execution timeline)
- [ ] **Audit trail**: SF execution history retained (Standard only); CloudTrail records every PutEvent + SF action
- [ ] **EventBridge archive replays**: `aws events start-replay` from archive successfully reprocesses events
- [ ] **Cross-account target receives event** (if `EXTERNAL_PUBLISHERS` non-empty): partner account confirms receipt
- [ ] **All Lambda functions use Powertools layer + idempotency table**

---

## Common gotchas (claude must address proactively)

- **SF Standard has 25,000 history events per execution.** For long-loop workflows, restart with continuation token instead of looping inside one execution.
- **SF Express logs to CW Logs only — no execution history in console.** Use Standard if you need audit trail.
- **`.sync` task can run up to 365 days** but Lambda timeout is 15 min. For long Lambda → use callback pattern.
- **EventBridge cross-account target requires resource policy on target bus + `aws events put-permission` on source.** Both. Common miss.
- **EventBridge schema registry costs $0.10/M events** for discovery. Disable in dev to avoid surprise bill.
- **SQS-Lambda partial batch failure requires `report_batch_item_failures` config in event source mapping.** Without it, single failure poisons whole batch.
- **DLQ message reception requires permissions on the DLQ from the Lambda role / SF role.** Cross-account DLQ — RAM share + SQS resource policy.
- **EventBridge rule input transformer** is the right way to reshape event payload before sending to target. Don't reshape in Lambda.
- **SF Catch ordering matters** — `States.ALL` should be LAST; specific error names go first.
- **Step Function service-linked role doesn't auto-grant Lambda invoke** — pass an explicit role with `lambda:InvokeFunction` on each task Lambda's ARN.

---

## Output artifacts

1. **CDK Python app** — stacks: `EventBusStack` → `DataStack` → `WorkflowStack` (SF + all step Lambdas) → `DownstreamStack` (rules + targets)
2. **Step Function definition** — ASL JSON or CDK ChainBuilder; one per `WORKFLOW_TYPE`
3. **Lambda source code** — one file per step in `src/steps/`; all use Powertools + idempotency
4. **Pydantic event schemas** — `src/schemas/` per event type; auto-registered in EventBridge schema registry
5. **DLQ + redrive Lambda** — replays DLQ messages back to source after fix deployed
6. **Pytest suite** — happy path, retry, idempotency, DLQ, cross-account
7. **CloudWatch dashboard JSON** — workflow overview
8. **6+ alarms YAML** — wired to SNS
9. **Failure simulation runbook** — `inject-failure.sh` script that produces 5 known failure modes
10. **EventBridge schema registry export** — for downstream consumers to generate types

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 2.0 | 2026-04-26 | Initial. EventBridge bus + Step Functions + SQS DLQ + Lambda step pattern. Wave 10. |
