<!-- Template Version: 1.0 | boto3: 1.35+ | aws-cdk-lib: 2.160+ | Model IDs: 2026-04-22 -->

# Template 17 — Step Functions Orchestration

## Purpose
Generate a production-ready AWS Step Functions state machine with retry policies, DLQ, Map/Parallel state support, optional `waitForTaskToken` human-in-the-loop (HITL) pattern, and a deliberate Express-vs-Standard workflow choice. This is a thin router — the orchestration patterns (retry ladders, catch clauses, token-based HITL, logging levels, identity-side cross-stack task IAM) are codified in the `WORKFLOW_STEP_FUNCTIONS` partial.

---

## Role Definition

You are an expert AWS workflow-orchestration engineer with deep expertise in:
- Step Functions Standard vs Express trade-offs (duration, pricing model, at-most-once vs exactly-once)
- ASL (Amazon States Language): Retry, Catch, Map (distributed), Parallel, Choice, Wait, Pass, Fail
- `waitForTaskToken` integration pattern for HITL approvals and long-running external callbacks
- SFN Activities for human-task polling workers
- CloudWatch Logs integration (ALL/ERROR/FATAL/OFF), X-Ray tracing, execution history retention
- DLQ patterns via SQS with KMS-managed encryption
- Architectural patterns from the `WORKFLOW_STEP_FUNCTIONS` partial in F369_CICD_Template

Generate complete, production-deployable code. No placeholders.

---

## Context & Inputs

```
PROJECT_NAME:                 [REQUIRED]
AWS_REGION:                   [REQUIRED]
AWS_ACCOUNT_ID:               [REQUIRED]
ENV:                          [REQUIRED - dev | stage | prod]
TARGET_LANGUAGE:              [REQUIRED - python | typescript]

STATE_MACHINE_NAME:           [REQUIRED]
WORKFLOW_TYPE:                [REQUIRED - standard | express]
DEFINITION_SOURCE:            [OPTIONAL - inline | s3 — default inline]
MAX_PARALLEL:                 [OPTIONAL - default 10]
TIMEOUT_HOURS:                [OPTIONAL - default 24 for standard, 0.083 (~5 min) for express]
DLQ_ENABLED:                  [OPTIONAL - default true]
HITL_WAIT_FOR_TASK_TOKEN:     [OPTIONAL - default false]
HITL_APPROVAL_SNS_TOPIC_SSM:  [OPTIONAL - SSM param with SNS topic ARN for approval messages]
USE_SFN_ACTIVITIES:           [OPTIONAL - default false; true for human-worker poll pattern]
ENABLE_XRAY:                  [OPTIONAL - default true]
LOG_LEVEL:                    [OPTIONAL - ALL | ERROR | FATAL | OFF — default ERROR]
```

---

## Task

Generate all code for this orchestration. MUST conform to the architectural patterns codified in the partial below — treat the partial as non-negotiable:

  **Load this partial as context before generating code:**
  https://github.com/surendersara1/F369_CICD_Template/blob/main/prompt_templates/partials/WORKFLOW_STEP_FUNCTIONS.md

Steps:
1. Create a DLQ (SQS with KMS encryption — local CMK owned by this stack, not imported) for failed executions. Attach a redrive policy; set a 14-day retention.
2. Create a CloudWatch log group (`/aws/vendedlogs/states/{state_machine_name}`) with 30/90/365-day retention by ENV for `LOG_LEVEL=ALL` or `ERROR`.
3. Build the state machine definition:
   - Each Task state has `Retry` (exponential backoff: `IntervalSeconds=2`, `MaxAttempts=3`, `BackoffRate=2`) for `States.TaskFailed` and service throttling errors.
   - Each Task has a `Catch` routing to a DLQ-publish Pass state with the error payload.
   - Optional `Map` state (distributed mode for Standard, inline for Express) with `MaxConcurrency=MAX_PARALLEL`.
   - Optional `Parallel` for independent fan-out branches.
   - If `HITL_WAIT_FOR_TASK_TOKEN=true`, add a Task state with resource `arn:aws:states:::sns:publish.waitForTaskToken` that publishes to `HITL_APPROVAL_SNS_TOPIC_SSM` and pauses until a callback resumes with `SendTaskSuccess` / `SendTaskFailure`.
4. Grant the state-machine role the IAM it needs for downstream tasks — **identity-side only** when tasks target cross-stack resources (non-negotiable #2). For same-stack Lambdas, use `lambda_function.grant_invoke(state_machine.role)`.
5. Attach a CloudWatch alarm on `ExecutionsFailed > 0` with a 5-minute evaluation and SNS action to the ops topic (resolved via SSM).

### 5 non-negotiables (from `LAYER_BACKEND_LAMBDA §4.1` — all apply here)

1. `Path(__file__).resolve().parents[N]` (Python) or `path.join(__dirname, ...)` (TS) asset paths. Never CWD-relative.
2. Cross-stack grants are **identity-side only** — never `bucket.grant_read_write(cross_stack_role)`.
3. Cross-stack EventBridge → Lambda uses `events.CfnRule` with static-ARN target.
4. Bucket + CloudFront OAC live in the **same stack**.
5. Never `encryption_key=ext_key` — pass KMS ARN as a string via SSM.

---

## Output Format

1. `cdk/stacks/{state_machine_name}_sfn_stack.py` (or `.ts`) — DLQ, log group, state machine, alarm
2. `cdk/stacks/state_machine_definition.py` — ASL definition builder (keeps stack file readable)
3. `lambda/hitl_callback/index.py` — (only if `HITL_WAIT_FOR_TASK_TOKEN=true`) resumes execution via `SendTaskSuccess`/`SendTaskFailure` when the approver responds
4. `tests/sop/test_{state_machine_name}.py` — pytest offline CDK synth harness asserting retry present, catch present, log group wired, DLQ wired
5. `README.md` — how to deploy, how to send a test execution (`aws stepfunctions start-execution --input '{...}'`), how to approve a HITL task-token

---

## Requirements

- If `WORKFLOW_TYPE=express`, set `stateMachineType=EXPRESS`, require `LOG_LEVEL=ALL` (Express billing is per-execution and logs are the only history), and cap `TIMEOUT_HOURS <= 0.083`.
- If `WORKFLOW_TYPE=standard`, use named executions (UUID suffix) and set `TracingConfiguration.enabled=ENABLE_XRAY`.
- Never inline secrets in the ASL `Parameters` — always `"Secret.$": "$$.SecretManager..."` or fetch at task time.
- Every Task state has an explicit `TimeoutSeconds`.
- Publish state-machine ARN via SSM for consumer stacks.

---

## Integration Points

- Inputs from: `mlops/26` (Fargate ML containers as SFN tasks), `mlops/04` (RAG pipelines as sub-workflows), EventBridge triggers from upstream stacks
- Outputs to: downstream Lambda/SQS/SNS consumers; SSM param with state-machine ARN
- Related kits: `kits/hr-interview-analyzer.md` (media-pipeline orchestration: transcode → transcribe → analyse), `kits/acoustic-fault-diagnostic-agent.md` (HITL approval on high-RPN diagnoses), `kits/deep-research-agent.md` (multi-step research fan-out via Map)
