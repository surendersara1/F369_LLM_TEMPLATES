<!-- Template Version: 1.0 | boto3: 1.35+ | aws-cdk-lib: 2.160+ | Model IDs: 2026-04-22 -->

# Template 26 — Fargate ML Custom Container

## Purpose
Generate an ECS Fargate task definition + service for ML workloads that don't fit a SageMaker endpoint: custom native libraries (Librosa, OpenSMILE, PRAAT, FFmpeg), deterministic CPU-only processing, cost-controlled always-on inference, or long-running batch. This is a thin router — the ECS/Fargate primitives (task-role vs execution-role split, ARM64-first choice, SQS-depth autoscaling, Container Insights) are codified in `LAYER_BACKEND_ECS`; the serving contract (health check, input/output envelope, CloudWatch metrics) is codified in `MLOPS_SAGEMAKER_SERVING`.

---

## Role Definition

You are an expert AWS ML-platform engineer with deep expertise in:
- ECS Fargate task definitions: execution role vs task role split, sidecars, task-scoped ephemeral storage
- `DockerImageAsset` asset-based image builds (Path(__file__)-anchored, never CWD-relative)
- ARM64 (Graviton) vs x86_64 trade-offs for ML containers — Graviton ~20% cheaper when libraries compile cleanly
- SQS-driven autoscaling (`ApproximateNumberOfMessagesVisible` target-tracking)
- ALB + Fargate service patterns, service discovery, Container Insights
- When Fargate is the wrong answer (GPU-required → SageMaker Processing / EC2 with NVIDIA runtime)
- Architectural patterns from `LAYER_BACKEND_ECS` and the serving contract from `MLOPS_SAGEMAKER_SERVING` in F369_CICD_Template

Generate complete, production-deployable code. No placeholders.

---

## Context & Inputs

```
PROJECT_NAME:             [REQUIRED]
AWS_REGION:               [REQUIRED]
AWS_ACCOUNT_ID:           [REQUIRED]
ENV:                      [REQUIRED - dev | stage | prod]
TARGET_LANGUAGE:          [REQUIRED - python | typescript]

SERVICE_NAME:             [REQUIRED - e.g. audio-feature-extractor]
IMAGE_ARCHITECTURE:       [OPTIONAL - arm64 (default — cheaper) | x86_64]
CPU:                      [OPTIONAL - 1024 default]
MEMORY_MIB:               [OPTIONAL - 4096 default]
DESIRED_COUNT:            [OPTIONAL - 2]
MAX_COUNT:                [OPTIONAL - 10]
AUTOSCALING_METRIC:       [OPTIONAL - cpu | memory | sqs_queue_depth — default sqs_queue_depth]
DOCKERFILE_PATH:          [REQUIRED - e.g. containers/audio-extractor/Dockerfile]
SQS_SOURCE_ARN_SSM:       [OPTIONAL - SSM param with source SQS ARN if event-driven]
S3_INPUT_BUCKET_SSM:      [OPTIONAL - SSM for raw inputs]
S3_OUTPUT_BUCKET_SSM:     [OPTIONAL - SSM for outputs]
ENABLE_SPOT:              [OPTIONAL - default true in non-prod, false in prod]
GPU_REQUIRED:             [OPTIONAL - default false; true triggers Fargate-rejection message — route to SageMaker Processing]
```

---

## Task

Generate all code for this Fargate ML service. MUST conform to the architectural patterns codified in the partials below — treat them as non-negotiable:

  **Load these partials as context before generating code:**
  - https://github.com/surendersara1/F369_CICD_Template/blob/main/prompt_templates/partials/LAYER_BACKEND_ECS.md (infra shape)
  - https://github.com/surendersara1/F369_CICD_Template/blob/main/prompt_templates/partials/MLOPS_SAGEMAKER_SERVING.md (serving contract: `/healthz`, `/invocations`, CloudWatch custom metrics, input/output JSON envelope)

Steps:
1. If `GPU_REQUIRED=true`, STOP: emit a README note directing the user to SageMaker Processing (`mlops/03`) or EC2-backed ECS with the NVIDIA runtime — Fargate does not support GPU. Do not generate a Fargate stack.
2. Build the container image via `DockerImageAsset` anchored with `Path(__file__).resolve().parents[N] / DOCKERFILE_PATH` (non-negotiable #1). Set `platform=Platform.LINUX_ARM64` when `IMAGE_ARCHITECTURE=arm64`.
3. Resolve or create the ECS cluster (`enableFargateCapacityProviders=true`, `containerInsights=ENHANCED`).
4. Task definition:
   - Execution role: `AmazonECSTaskExecutionRolePolicy` + KMS decrypt for the log group CMK.
   - Task role: identity-side grants only (non-negotiable #2) for cross-stack S3 read on `S3_INPUT_BUCKET_SSM`, S3 write on `S3_OUTPUT_BUCKET_SSM`, SQS receive/delete on `SQS_SOURCE_ARN_SSM`.
   - Container: CPU/MEMORY per params, port 8080 (health `/healthz` per serving partial), log driver `awslogs` with CMK-encrypted log group.
5. Service:
   - If `SQS_SOURCE_ARN_SSM` is set, event-driven Fargate service (no ALB); autoscale on `ApproximateNumberOfMessagesVisible` target (target `MAX_PARALLEL_PER_TASK * DESIRED_COUNT`, e.g. messages-per-task=5).
   - Else, ALB-fronted service with `/healthz` target group health check.
6. Auto-scaling: min=`DESIRED_COUNT`, max=`MAX_COUNT`, target-tracking on `AUTOSCALING_METRIC`, scale-in cooldown 300s, scale-out 60s.
7. If `ENABLE_SPOT=true` and `ENV != prod`, set capacity-provider strategy `FARGATE_SPOT` weight 4 + `FARGATE` weight 1.
8. Publish service ARN + task-definition family via SSM for consumer stacks.

### 5 non-negotiables (from `LAYER_BACKEND_LAMBDA §4.1` — all apply here)

1. `Path(__file__).resolve().parents[N]` (Python) or `path.join(__dirname, ...)` (TS) asset paths. Never CWD-relative.
2. Cross-stack grants are **identity-side only** — never `bucket.grant_read_write(cross_stack_role)`.
3. Cross-stack EventBridge → Lambda uses `events.CfnRule` with static-ARN target.
4. Bucket + CloudFront OAC live in the **same stack**.
5. Never `encryption_key=ext_key` — pass KMS ARN as a string via SSM.

---

## Output Format

1. `cdk/stacks/{service_name}_fargate_stack.py` (or `.ts`) — cluster reference, task def, service, autoscaling, optional ALB
2. `containers/{service_name}/Dockerfile` — minimal ML container (multi-stage, ARM64-ready, non-root user)
3. `containers/{service_name}/app/server.py` — `/healthz` + `/invocations` conforming to `MLOPS_SAGEMAKER_SERVING` envelope
4. `containers/{service_name}/app/worker.py` — (only if `SQS_SOURCE_ARN_SSM` set) long-poll SQS → process → write S3 output
5. `tests/sop/test_{service_name}_fargate.py` — pytest offline CDK synth harness asserting task role has identity-side grants only, log group CMK present, autoscaling wired
6. `README.md` — local Docker test, cost estimate, GPU-routing notice

---

## Requirements

- Container runs as non-root UID. Read-only root filesystem with `/tmp` tmpfs writable.
- `/invocations` emits CloudWatch custom metrics (`InferenceLatencyMs`, `InferenceErrors`) per `MLOPS_SAGEMAKER_SERVING`.
- Log group retention: 30d dev, 90d stage, 365d prod. KMS via local CMK (non-negotiable #5 — not imported).
- Task-definition `runtimePlatform.cpuArchitecture` matches `IMAGE_ARCHITECTURE`.
- If event-driven, the worker uses SQS long-poll (20s `WaitTimeSeconds`) and visibility-timeout > max-processing-time.

---

## Integration Points

- Inputs from: `devops/17` (Step Functions invoking this service as an `ecs:RunTask.sync` task), `enterprise/06` (compliance CMK + audit bucket via SSM)
- Outputs to: S3 output bucket (consumer stacks read via SSM), CloudWatch custom metrics
- Related kits: `kits/acoustic-fault-diagnostic-agent.md` (Librosa/OpenSMILE feature extraction — the canonical use case), `kits/hr-interview-analyzer.md` (FFmpeg media transcode worker)
