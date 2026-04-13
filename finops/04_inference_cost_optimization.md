<!-- Template Version: 1.0 | boto3: 1.35+ | SageMaker: Application Auto Scaling -->

# Template FinOps 04 — Inference Cost Optimization

## Purpose
Generate a production-ready ML inference cost optimization framework: SageMaker endpoint auto-scaling policies (target tracking on InvocationsPerInstance and step scaling on ModelLatency), scale-to-zero pattern for dev/stage endpoints using scheduled scaling actions with CloudWatch alarm-based scale-up, instance type benchmarking across Inferentia2 (ml.inf2), GPU (ml.g5), and Graviton (ml.c7g) instance families, batch transform job configuration for latency-tolerant workloads, multi-model endpoint setup to consolidate per-model infrastructure cost, and a utilization-based downsizing Lambda that reduces instance count or switches instance types when utilization drops below a configurable threshold.

---

## Role Definition

You are an expert AWS ML platform engineer and inference cost optimization specialist with expertise in:
- SageMaker endpoint auto-scaling: Application Auto Scaling integration, `register_scalable_target()`, `put_scaling_policy()`, target tracking and step scaling policies
- Scale-to-zero patterns: scheduled scaling actions for off-hours, CloudWatch alarm-based scale-up on `Invocations` metric, `MinCapacity=0` configuration
- Instance type optimization: Inferentia2 (ml.inf2) for cost-efficient inference, GPU (ml.g5) for general-purpose ML, Graviton (ml.c7g) for CPU-based inference, latency/cost benchmarking
- SageMaker batch transform: `create_transform_job()` for offline batch inference, S3 input/output, instance selection, max concurrent transforms
- Multi-model endpoints: `MultiModelConfig` for hosting multiple models on a single endpoint, model loading/unloading, shared infrastructure cost reduction
- CloudWatch metrics: `InvocationsPerInstance`, `ModelLatency`, `CPUUtilization`, `GPUUtilization`, `MemoryUtilization`, custom metric math
- Lambda-based automation: utilization monitoring, instance downsizing, instance type switching, SNS notifications
- Cost analysis: per-inference cost calculation, instance hour optimization, spot vs on-demand for batch workloads
- IAM policies for Application Auto Scaling, SageMaker, CloudWatch, and Lambda permissions

Generate complete, production-deployable code.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

ENDPOINT_NAME:          [REQUIRED - SageMaker endpoint name suffix]
                         Example: "text-classifier"
                         Full name becomes: {PROJECT_NAME}-{ENDPOINT_NAME}-{ENV}

SCALING_METRIC:         [OPTIONAL: InvocationsPerInstance - primary metric for auto-scaling]
                         Options: InvocationsPerInstance, ModelLatency
                         - InvocationsPerInstance: scale based on request volume (recommended for most)
                         - ModelLatency: scale based on response latency (for latency-sensitive workloads)

MIN_CAPACITY:           [OPTIONAL: 1 - minimum instance count for auto-scaling]
                         Set to 0 for scale-to-zero pattern (requires SCALE_TO_ZERO_SCHEDULE).

MAX_CAPACITY:           [OPTIONAL: 4 - maximum instance count for auto-scaling]
                         Upper bound for scaling. Set based on peak traffic expectations.

SCALE_TO_ZERO_SCHEDULE: [OPTIONAL: none - cron expressions for scale-to-zero windows]
                         JSON with scale-down and scale-up schedules.
                         Example:
                         {
                           "scale_down": "cron(0 22 ? * MON-FRI *)",
                           "scale_up": "cron(0 7 ? * MON-FRI *)"
                         }
                         scale_down sets MinCapacity=0, MaxCapacity=0.
                         scale_up restores MIN_CAPACITY and MAX_CAPACITY.
                         A CloudWatch alarm on Invocations > 0 provides emergency scale-up.

INSTANCE_FAMILIES:      [OPTIONAL: g5 - comma-separated instance families to benchmark]
                         Options: inf2, g5, c7g
                         - inf2: AWS Inferentia2 chips — best cost/performance for supported models
                         - g5: NVIDIA A10G GPUs — general-purpose GPU inference
                         - c7g: AWS Graviton3 — CPU-based inference for lightweight models
                         Example: "inf2,g5,c7g"

BATCH_TRANSFORM_ENABLED: [OPTIONAL: false - enable batch transform job configuration]
                          Set to true to generate batch transform infrastructure as an
                          alternative to real-time endpoints for latency-tolerant workloads.

MULTI_MODEL_ENABLED:    [OPTIONAL: false - enable multi-model endpoint configuration]
                         Set to true to generate multi-model endpoint setup that hosts
                         multiple models on a single endpoint for cost consolidation.

TARGET_VALUE:           [OPTIONAL: 750 - target value for target tracking scaling policy]
                         For InvocationsPerInstance: target invocations per instance (e.g., 750).
                         For ModelLatency: target latency in microseconds (e.g., 100000 = 100ms).

UTILIZATION_THRESHOLD:  [OPTIONAL: 20 - CPU/GPU utilization % below which downsizing triggers]
                         When average utilization stays below this threshold for
                         EVALUATION_PERIODS consecutive periods, the downsizing Lambda fires.

EVALUATION_PERIODS:     [OPTIONAL: 6 - number of consecutive low-utilization periods before action]
                         Each period is 5 minutes. Default 6 = 30 minutes sustained low utilization.

NOTIFICATION_EMAIL:     [OPTIONAL: none - email for scaling and downsizing notifications]
                         Example: "ml-platform@example.com"

SNS_TOPIC_ARN:          [OPTIONAL - existing SNS topic for notifications]
                         If not provided, a new topic is created:
                         {PROJECT_NAME}-inference-scaling-{ENV}

MODEL_DATA_URL:         [OPTIONAL - S3 path to model artifacts for batch transform / multi-model]
                         Example: "s3://my-models/text-classifier/model.tar.gz"
                         For multi-model: "s3://my-models/multi-model/" (prefix containing multiple model.tar.gz)

BATCH_INPUT_S3:         [OPTIONAL - S3 path for batch transform input data]
                         Example: "s3://my-data/batch-input/"

BATCH_OUTPUT_S3:        [OPTIONAL - S3 path for batch transform output]
                         Example: "s3://my-data/batch-output/"
```

---

## Task

Generate complete inference cost optimization framework:

```
{PROJECT_NAME}-inference-cost/
├── autoscaling/
│   ├── register_scalable_target.py       # Register endpoint as scalable target
│   ├── target_tracking_policy.py         # Target tracking scaling policy
│   ├── step_scaling_policy.py            # Step scaling policy for latency
│   └── scaling_status.py                 # Check current scaling configuration
├── scale_to_zero/
│   ├── scheduled_actions.py              # Scheduled scale-down and scale-up actions
│   ├── alarm_scale_up.py                 # CloudWatch alarm for emergency scale-up
│   └── manage_schedule.py                # Enable/disable scale-to-zero schedules
├── benchmarking/
│   ├── instance_benchmark.py             # Benchmark latency/cost across instance families
│   └── benchmark_report.py              # Generate comparison report
├── batch_transform/
│   ├── create_transform_job.py           # Create batch transform job
│   └── batch_vs_realtime.py              # Cost comparison: batch vs real-time
├── multi_model/
│   ├── create_multi_model_endpoint.py    # Create multi-model endpoint
│   └── model_management.py              # Load/unload models, list loaded models
├── downsizing/
│   ├── downsizing_lambda.py              # Lambda: reduce instances on low utilization
│   ├── downsizing_alarm.py               # CloudWatch alarm triggering downsizing Lambda
│   └── deploy_lambda.py                  # Deploy the downsizing Lambda function
├── infrastructure/
│   ├── config.py                         # Central configuration
│   └── sns_topics.py                     # SNS topics for scaling notifications
├── run_setup.py                          # CLI orchestrator
└── requirements.txt
```

**config.py**: Central configuration dataclass with all parameters. Load from environment variables or CLI args. Validate required fields. Parse SCALE_TO_ZERO_SCHEDULE from JSON string. Validate INSTANCE_FAMILIES against allowed values (inf2, g5, c7g). Validate UTILIZATION_THRESHOLD is between 1 and 100. Construct resource names using `{PROJECT_NAME}-{component}-{ENV}` convention. Construct full endpoint name as `{PROJECT_NAME}-{ENDPOINT_NAME}-{ENV}`.

**register_scalable_target.py**: Register SageMaker endpoint variant as a scalable target:
- Call `application_autoscaling.register_scalable_target()` with:
  - `ServiceNamespace="sagemaker"`
  - `ResourceId=f"endpoint/{endpoint_name}/variant/AllTraffic"`
  - `ScalableDimension="sagemaker:variant:DesiredInstanceCount"`
  - `MinCapacity=MIN_CAPACITY`, `MaxCapacity=MAX_CAPACITY`
- Verify endpoint exists before registration using `sagemaker.describe_endpoint()`
- Print current and configured scaling bounds
- Handle `ObjectNotFoundException` if endpoint variant doesn't exist

**target_tracking_policy.py**: Create target tracking auto-scaling policy:
- Call `application_autoscaling.put_scaling_policy()` with:
  - `PolicyType="TargetTrackingScaling"`
  - `TargetTrackingScalingPolicyConfiguration` using:
    - `TargetValue=TARGET_VALUE`
    - `PredefinedMetricSpecification` with `PredefinedMetricType` set to `SageMakerVariantInvocationsPerInstance` (if SCALING_METRIC is InvocationsPerInstance) or `CustomizedMetricSpecification` for ModelLatency
    - `ScaleInCooldown=300` (5 minutes), `ScaleOutCooldown=60` (1 minute)
- Policy name: `{PROJECT_NAME}-{ENDPOINT_NAME}-target-tracking-{ENV}`
- Print policy ARN and configuration summary

**step_scaling_policy.py**: Create step scaling policy for latency-based scaling:
- Call `application_autoscaling.put_scaling_policy()` with:
  - `PolicyType="StepScaling"`
  - `StepScalingPolicyConfiguration` with step adjustments:
    - ModelLatency 100-200ms above target → add 1 instance
    - ModelLatency 200-500ms above target → add 2 instances
    - ModelLatency >500ms above target → add 3 instances
  - `AdjustmentType="ChangeInCapacity"`, `Cooldown=120`
- Create companion CloudWatch alarm on `ModelLatency` metric that triggers the step policy
- Policy name: `{PROJECT_NAME}-{ENDPOINT_NAME}-step-scaling-{ENV}`

**scaling_status.py**: Check current scaling configuration:
- Call `application_autoscaling.describe_scalable_targets()` for the endpoint
- Call `application_autoscaling.describe_scaling_policies()` for all policies
- Call `application_autoscaling.describe_scheduled_actions()` for scheduled actions
- Display current min/max capacity, active policies, and scheduled actions
- Query CloudWatch for recent `InvocationsPerInstance` and `ModelLatency` metrics

**scheduled_actions.py**: Create scheduled scaling actions for scale-to-zero:
- Parse SCALE_TO_ZERO_SCHEDULE JSON for scale_down and scale_up cron expressions
- Create scale-down action using `application_autoscaling.put_scheduled_action()`:
  - Set `MinCapacity=0`, `MaxCapacity=0` during off-hours
  - Schedule: scale_down cron expression
  - Action name: `{PROJECT_NAME}-{ENDPOINT_NAME}-scale-down-{ENV}`
- Create scale-up action:
  - Restore `MinCapacity=MIN_CAPACITY`, `MaxCapacity=MAX_CAPACITY` during business hours
  - Schedule: scale_up cron expression
  - Action name: `{PROJECT_NAME}-{ENDPOINT_NAME}-scale-up-{ENV}`
- Print schedule summary with next execution times

**alarm_scale_up.py**: Create CloudWatch alarm for emergency scale-up when at zero:
- Create alarm using `cloudwatch.put_metric_alarm()`:
  - Monitor `Invocations` metric for the endpoint (sum over 1 minute)
  - Threshold: `Invocations > 0` (any incoming request triggers scale-up)
  - Action: call Application Auto Scaling to set `MinCapacity=1`
  - Alarm name: `{PROJECT_NAME}-{ENDPOINT_NAME}-scale-from-zero-{ENV}`
- This ensures the endpoint scales up immediately when traffic arrives during off-hours
- Companion Lambda function that the alarm invokes to call `register_scalable_target()` with `MinCapacity=1`

**manage_schedule.py**: Enable/disable scale-to-zero schedules:
- `enable_schedule()`: activate both scheduled actions
- `disable_schedule()`: deactivate scheduled actions (useful during high-traffic periods)
- `status()`: show current schedule state and next execution times
- Uses `application_autoscaling.describe_scheduled_actions()` and `delete_scheduled_action()` / `put_scheduled_action()`

**instance_benchmark.py**: Benchmark inference across instance families:
- For each instance family in INSTANCE_FAMILIES:
  - Map to specific instance types: inf2→ml.inf2.xlarge, g5→ml.g5.xlarge, c7g→ml.c7g.xlarge
  - Create temporary endpoint configuration with the instance type
  - Deploy model to temporary endpoint using `sagemaker.create_endpoint()`
  - Send N benchmark requests using `sagemaker_runtime.invoke_endpoint()`
  - Measure: p50/p95/p99 latency, throughput (requests/sec), cold start time
  - Get on-demand hourly rate from Pricing API
  - Calculate cost-per-1000-inferences = (hourly_rate / throughput_per_hour) × 1000
  - Clean up temporary endpoint after benchmarking
- Output comparison table: instance type, latency (p50/p95/p99), throughput, hourly cost, cost-per-1000-inferences
- Recommend optimal instance type based on cost/latency trade-off

**benchmark_report.py**: Generate formatted benchmark comparison report:
- Read benchmark results from instance_benchmark.py
- Generate markdown table with all metrics
- Highlight recommended instance type (lowest cost-per-inference meeting latency SLA)
- Include cost projection: monthly cost at current traffic volume for each instance type
- Output JSON and markdown report files

**create_transform_job.py**: Create SageMaker batch transform job:
- Call `sagemaker.create_transform_job()` with:
  - `TransformJobName=f"{PROJECT_NAME}-batch-{ENDPOINT_NAME}-{ENV}-{timestamp}"`
  - `ModelName` referencing the deployed model
  - `TransformInput` with S3 data source (BATCH_INPUT_S3), content type, split type
  - `TransformOutput` with S3 output path (BATCH_OUTPUT_S3), accept type, assembler
  - `TransformResources` with instance type and count
  - `MaxConcurrentTransforms` for parallelism
  - `BatchStrategy="MultiRecord"` for throughput optimization
- Poll for completion using `sagemaker.describe_transform_job()`
- Print job status, duration, and records processed

**batch_vs_realtime.py**: Cost comparison between batch transform and real-time endpoints:
- Calculate real-time endpoint cost: instance_hours × hourly_rate (24/7 running)
- Calculate batch transform cost: job_instance_hours × hourly_rate (only during job)
- Compare for different traffic patterns: continuous, periodic (daily batch), sporadic
- Output recommendation: use real-time for >N requests/hour, batch for less
- Include break-even analysis showing the traffic threshold

**create_multi_model_endpoint.py**: Create multi-model endpoint:
- Create model using `sagemaker.create_model()` with:
  - `PrimaryContainer` including `MultiModelConfig={"ModelDataUrl": MODEL_DATA_URL}`
  - Container image for multi-model server (e.g., SageMaker built-in multi-model container)
- Create endpoint configuration with the multi-model model
- Create endpoint using `sagemaker.create_endpoint()`
- Endpoint name: `{PROJECT_NAME}-multi-model-{ENV}`
- Print endpoint details and model data URL prefix

**model_management.py**: Manage models on multi-model endpoint:
- `list_models()`: call `sagemaker_runtime.invoke_endpoint()` with target model listing
- `load_model(model_name)`: invoke endpoint with `TargetModel` parameter to trigger loading
- `unload_model(model_name)`: remove model from memory (via endpoint update)
- `invoke_model(model_name, payload)`: call `sagemaker_runtime.invoke_endpoint()` with `TargetModel=model_name`
- Track model loading latency and cache hit rates

**downsizing_lambda.py**: Lambda function for utilization-based downsizing:
- Triggered by CloudWatch alarm when utilization drops below UTILIZATION_THRESHOLD
- Query current endpoint configuration using `sagemaker.describe_endpoint()`
- Query CloudWatch for average CPU/GPU utilization over the evaluation window
- If utilization is below threshold:
  - Option 1: Reduce `DesiredInstanceCount` by 1 (if current count > MIN_CAPACITY)
  - Option 2: Switch to a smaller instance type within the same family
- Call `application_autoscaling.register_scalable_target()` to update capacity
- Send SNS notification with: endpoint name, previous/new capacity, utilization metrics, action taken
- Log all actions to CloudWatch Logs for audit trail

**downsizing_alarm.py**: Create CloudWatch alarm that triggers the downsizing Lambda:
- Create alarm using `cloudwatch.put_metric_alarm()`:
  - Metric: `CPUUtilization` or `GPUUtilization` for the endpoint
  - Threshold: UTILIZATION_THRESHOLD (default 20%)
  - Comparison: `LessThanThreshold`
  - Evaluation periods: EVALUATION_PERIODS (default 6 × 5min = 30 minutes)
  - Action: invoke downsizing Lambda ARN
  - Alarm name: `{PROJECT_NAME}-{ENDPOINT_NAME}-low-util-{ENV}`

**deploy_lambda.py**: Deploy the downsizing Lambda function:
- Create Lambda function using `lambda_client.create_function()`:
  - Function name: `{PROJECT_NAME}-downsizing-{ENV}`
  - Runtime: `python3.12`
  - Handler: `downsizing_lambda.lambda_handler`
  - IAM role with permissions for SageMaker, Application Auto Scaling, CloudWatch, SNS
  - Environment variables: ENDPOINT_NAME, MIN_CAPACITY, SNS_TOPIC_ARN
  - Timeout: 60 seconds
- Add CloudWatch Alarms as trigger via `lambda_client.add_permission()` and alarm action

**sns_topics.py**: Create SNS topics for scaling notifications:
- Topic name: `{PROJECT_NAME}-inference-scaling-{ENV}`
- Subscribe NOTIFICATION_EMAIL if provided
- Set topic policy allowing CloudWatch Alarms and Lambda to publish

**run_setup.py**: CLI orchestrator that runs setup steps in order:
1. Load and validate configuration
2. Create SNS topics for notifications
3. Register endpoint as scalable target
4. Create target tracking scaling policy
5. Create step scaling policy (if SCALING_METRIC includes latency)
6. Create scale-to-zero scheduled actions (if SCALE_TO_ZERO_SCHEDULE provided)
7. Create emergency scale-up alarm (if scale-to-zero enabled)
8. Deploy downsizing Lambda and alarm
9. Run instance benchmarking (if INSTANCE_FAMILIES specified)
10. Create batch transform job configuration (if BATCH_TRANSFORM_ENABLED)
11. Create multi-model endpoint (if MULTI_MODEL_ENABLED)
12. Print summary with scaling policies, schedules, alarms, and recommendations

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints


**Auto-Scaling:** Register every production SageMaker endpoint as a scalable target with Application Auto Scaling. Use target tracking on `SageMakerVariantInvocationsPerInstance` as the primary policy — it automatically adjusts instance count to maintain the target invocations per instance. Add a step scaling policy on `ModelLatency` as a secondary policy for latency-sensitive workloads — step scaling provides more aggressive scale-out when latency spikes. Set `ScaleOutCooldown` to 60 seconds (fast scale-out) and `ScaleInCooldown` to 300 seconds (conservative scale-in to avoid flapping).

**Scale-to-Zero:** For dev/stage endpoints, implement scale-to-zero using scheduled scaling actions that set `MinCapacity=0` and `MaxCapacity=0` during off-hours. Pair with a CloudWatch alarm on `Invocations > 0` that triggers a Lambda to restore `MinCapacity=1`. Cold start latency when scaling from zero is 5-10 minutes for SageMaker endpoints — this pattern is only suitable for non-production environments where cold start is acceptable. Never use scale-to-zero for production endpoints serving real-time traffic.

**Instance Type Selection:** Benchmark across instance families before committing to a type. Inferentia2 (ml.inf2) offers up to 4x better price-performance for supported model architectures (transformers, CNNs) — always test inf2 first. GPU instances (ml.g5) are the general-purpose choice for models requiring CUDA. Graviton (ml.c7g) offers 20% better price-performance than x86 for CPU-based inference (sklearn, XGBoost, lightweight NLP). Run benchmarks with production-representative payloads, not synthetic data.

**Batch Transform:** Use batch transform instead of real-time endpoints when: latency tolerance is >minutes, traffic is periodic (daily/weekly batch), or request volume doesn't justify 24/7 endpoint cost. Batch transform charges only for the compute time during the job. Configure `MaxConcurrentTransforms` to match the number of vCPUs for optimal throughput. Use `BatchStrategy="MultiRecord"` with `MaxPayloadInMB` to batch multiple records per request for higher throughput.

**Multi-Model Endpoints:** Multi-model endpoints host multiple models on a single endpoint, sharing the instance cost across models. Best for: many models with low individual traffic, A/B testing variants, per-customer models. Models are loaded into memory on first invocation and unloaded when memory pressure requires it. Monitor `ModelLoadingWaitTime` and `ModelCacheHit` metrics to ensure adequate instance sizing. Multi-model endpoints do not support auto-scaling on a per-model basis — scaling applies to the entire endpoint.

**Downsizing Automation:** The downsizing Lambda monitors CPU/GPU utilization and reduces capacity when utilization is consistently low. Use a sustained low-utilization threshold (default: 20% for 30 minutes) to avoid reacting to transient dips. The Lambda should only reduce by 1 instance at a time to avoid aggressive downsizing. Never downsize below `MIN_CAPACITY`. Send SNS notifications for all downsizing actions so the team can review and adjust thresholds.

**Security:** Application Auto Scaling requires `application-autoscaling:RegisterScalableTarget`, `application-autoscaling:PutScalingPolicy`, `application-autoscaling:PutScheduledAction` permissions. The auto-scaling service-linked role `AWSServiceRoleForApplicationAutoScaling_SageMakerEndpoint` must exist. The downsizing Lambda needs `sagemaker:DescribeEndpoint`, `sagemaker:UpdateEndpointWeightsAndCapacities`, `application-autoscaling:RegisterScalableTarget`, `cloudwatch:GetMetricData`, and `sns:Publish` permissions. Restrict scaling policy modifications to platform engineers — data scientists should not be able to change scaling bounds.

**Cost:** Real-time endpoints charge per instance-hour. Batch transform charges per instance-hour during the job only. Multi-model endpoints charge for the single endpoint's instances regardless of how many models are loaded. CloudWatch alarms: $0.10/month per standard alarm. Lambda invocations for downsizing: negligible cost. The primary cost savings come from: right-sizing instance types (20-60% savings), scale-to-zero for non-prod (up to 70% savings on dev/stage), and batch transform for periodic workloads (80-95% savings vs 24/7 endpoints).

**Naming:** All resources follow `{PROJECT_NAME}-{component}-{ENV}` convention:
- Scalable target: `endpoint/{PROJECT_NAME}-{ENDPOINT_NAME}-{ENV}/variant/AllTraffic`
- Target tracking policy: `{PROJECT_NAME}-{ENDPOINT_NAME}-target-tracking-{ENV}`
- Step scaling policy: `{PROJECT_NAME}-{ENDPOINT_NAME}-step-scaling-{ENV}`
- Scheduled actions: `{PROJECT_NAME}-{ENDPOINT_NAME}-scale-down-{ENV}`, `{PROJECT_NAME}-{ENDPOINT_NAME}-scale-up-{ENV}`
- Scale-from-zero alarm: `{PROJECT_NAME}-{ENDPOINT_NAME}-scale-from-zero-{ENV}`
- Low utilization alarm: `{PROJECT_NAME}-{ENDPOINT_NAME}-low-util-{ENV}`
- Downsizing Lambda: `{PROJECT_NAME}-downsizing-{ENV}`
- Multi-model endpoint: `{PROJECT_NAME}-multi-model-{ENV}`
- Batch transform job: `{PROJECT_NAME}-batch-{ENDPOINT_NAME}-{ENV}-{timestamp}`
- SNS topic: `{PROJECT_NAME}-inference-scaling-{ENV}`

---

## Code Scaffolding Hints

**Register SageMaker endpoint as scalable target:**
```python
import boto3

autoscaling = boto3.client("application-autoscaling", region_name=AWS_REGION)
sagemaker = boto3.client("sagemaker", region_name=AWS_REGION)

endpoint_name = f"{PROJECT_NAME}-{ENDPOINT_NAME}-{ENV}"
resource_id = f"endpoint/{endpoint_name}/variant/AllTraffic"

# Verify endpoint exists
try:
    ep = sagemaker.describe_endpoint(EndpointName=endpoint_name)
    print(f"Endpoint status: {ep['EndpointStatus']}")
except sagemaker.exceptions.ClientError:
    raise ValueError(f"Endpoint {endpoint_name} not found")

# Register scalable target
autoscaling.register_scalable_target(
    ServiceNamespace="sagemaker",
    ResourceId=resource_id,
    ScalableDimension="sagemaker:variant:DesiredInstanceCount",
    MinCapacity=MIN_CAPACITY,
    MaxCapacity=MAX_CAPACITY,
)
print(f"Registered scalable target: {resource_id}")
print(f"  MinCapacity: {MIN_CAPACITY}, MaxCapacity: {MAX_CAPACITY}")
```

**Create target tracking scaling policy (InvocationsPerInstance):**
```python
policy_name = f"{PROJECT_NAME}-{ENDPOINT_NAME}-target-tracking-{ENV}"

response = autoscaling.put_scaling_policy(
    PolicyName=policy_name,
    ServiceNamespace="sagemaker",
    ResourceId=resource_id,
    ScalableDimension="sagemaker:variant:DesiredInstanceCount",
    PolicyType="TargetTrackingScaling",
    TargetTrackingScalingPolicyConfiguration={
        "TargetValue": TARGET_VALUE,  # e.g., 750 invocations per instance
        "PredefinedMetricSpecification": {
            "PredefinedMetricType": "SageMakerVariantInvocationsPerInstance",
        },
        "ScaleInCooldown": 300,   # 5 minutes — conservative scale-in
        "ScaleOutCooldown": 60,   # 1 minute — fast scale-out
    },
)
print(f"Target tracking policy created: {policy_name}")
print(f"  Target value: {TARGET_VALUE} invocations/instance")
print(f"  Policy ARN: {response['PolicyARN']}")
```

**Create target tracking scaling policy (ModelLatency):**
```python
policy_name = f"{PROJECT_NAME}-{ENDPOINT_NAME}-latency-tracking-{ENV}"

response = autoscaling.put_scaling_policy(
    PolicyName=policy_name,
    ServiceNamespace="sagemaker",
    ResourceId=resource_id,
    ScalableDimension="sagemaker:variant:DesiredInstanceCount",
    PolicyType="TargetTrackingScaling",
    TargetTrackingScalingPolicyConfiguration={
        "TargetValue": TARGET_VALUE,  # e.g., 100000 microseconds = 100ms
        "CustomizedMetricSpecification": {
            "MetricName": "ModelLatency",
            "Namespace": "AWS/SageMaker",
            "Dimensions": [
                {"Name": "EndpointName", "Value": endpoint_name},
                {"Name": "VariantName", "Value": "AllTraffic"},
            ],
            "Statistic": "Average",
            "Unit": "Microseconds",
        },
        "ScaleInCooldown": 300,
        "ScaleOutCooldown": 60,
    },
)
print(f"Latency tracking policy created: {policy_name}")
print(f"  Target latency: {TARGET_VALUE} microseconds")
```

**Create step scaling policy for latency spikes:**
```python
step_policy_name = f"{PROJECT_NAME}-{ENDPOINT_NAME}-step-scaling-{ENV}"

response = autoscaling.put_scaling_policy(
    PolicyName=step_policy_name,
    ServiceNamespace="sagemaker",
    ResourceId=resource_id,
    ScalableDimension="sagemaker:variant:DesiredInstanceCount",
    PolicyType="StepScaling",
    StepScalingPolicyConfiguration={
        "AdjustmentType": "ChangeInCapacity",
        "StepAdjustments": [
            {
                "MetricIntervalLowerBound": 0,       # 0-100ms above threshold
                "MetricIntervalUpperBound": 100000,
                "ScalingAdjustment": 1,
            },
            {
                "MetricIntervalLowerBound": 100000,   # 100-400ms above threshold
                "MetricIntervalUpperBound": 400000,
                "ScalingAdjustment": 2,
            },
            {
                "MetricIntervalLowerBound": 400000,   # >400ms above threshold
                "ScalingAdjustment": 3,
            },
        ],
        "Cooldown": 120,
    },
)
step_policy_arn = response["PolicyARN"]

# Create companion CloudWatch alarm to trigger step policy
cloudwatch = boto3.client("cloudwatch", region_name=AWS_REGION)

cloudwatch.put_metric_alarm(
    AlarmName=f"{PROJECT_NAME}-{ENDPOINT_NAME}-latency-high-{ENV}",
    Namespace="AWS/SageMaker",
    MetricName="ModelLatency",
    Dimensions=[
        {"Name": "EndpointName", "Value": endpoint_name},
        {"Name": "VariantName", "Value": "AllTraffic"},
    ],
    Statistic="Average",
    Period=60,
    EvaluationPeriods=3,
    Threshold=float(TARGET_VALUE),  # Latency threshold in microseconds
    ComparisonOperator="GreaterThanThreshold",
    AlarmActions=[step_policy_arn],
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)
print(f"Step scaling policy created: {step_policy_name}")
```

**Create scheduled scaling actions for scale-to-zero:**
```python
import json

schedule = json.loads(SCALE_TO_ZERO_SCHEDULE)

# Scale-down action: set capacity to 0 during off-hours
autoscaling.put_scheduled_action(
    ServiceNamespace="sagemaker",
    ScheduledActionName=f"{PROJECT_NAME}-{ENDPOINT_NAME}-scale-down-{ENV}",
    ResourceId=resource_id,
    ScalableDimension="sagemaker:variant:DesiredInstanceCount",
    Schedule=schedule["scale_down"],  # e.g., "cron(0 22 ? * MON-FRI *)"
    ScalableTargetAction={
        "MinCapacity": 0,
        "MaxCapacity": 0,
    },
)
print(f"Scale-down scheduled: {schedule['scale_down']}")

# Scale-up action: restore capacity during business hours
autoscaling.put_scheduled_action(
    ServiceNamespace="sagemaker",
    ScheduledActionName=f"{PROJECT_NAME}-{ENDPOINT_NAME}-scale-up-{ENV}",
    ResourceId=resource_id,
    ScalableDimension="sagemaker:variant:DesiredInstanceCount",
    Schedule=schedule["scale_up"],  # e.g., "cron(0 7 ? * MON-FRI *)"
    ScalableTargetAction={
        "MinCapacity": MIN_CAPACITY,
        "MaxCapacity": MAX_CAPACITY,
    },
)
print(f"Scale-up scheduled: {schedule['scale_up']}")
```

**CloudWatch alarm for emergency scale-up from zero:**
```python
# Lambda function that restores MinCapacity when invocations arrive at zero-capacity endpoint
scale_from_zero_lambda_code = """
import boto3
import os

def lambda_handler(event, context):
    autoscaling = boto3.client("application-autoscaling")
    sns = boto3.client("sns")

    endpoint_name = os.environ["ENDPOINT_NAME"]
    min_capacity = int(os.environ["MIN_CAPACITY"])
    max_capacity = int(os.environ["MAX_CAPACITY"])
    resource_id = f"endpoint/{endpoint_name}/variant/AllTraffic"

    # Restore scaling capacity
    autoscaling.register_scalable_target(
        ServiceNamespace="sagemaker",
        ResourceId=resource_id,
        ScalableDimension="sagemaker:variant:DesiredInstanceCount",
        MinCapacity=min_capacity,
        MaxCapacity=max_capacity,
    )

    # Notify team
    topic_arn = os.environ.get("SNS_TOPIC_ARN")
    if topic_arn:
        sns.publish(
            TopicArn=topic_arn,
            Subject=f"Scale-from-zero triggered: {endpoint_name}",
            Message=f"Endpoint {endpoint_name} received traffic while at zero capacity. "
                    f"Restored MinCapacity={min_capacity}, MaxCapacity={max_capacity}.",
        )

    return {"statusCode": 200, "body": f"Scaled up {endpoint_name}"}
"""

# CloudWatch alarm: trigger when invocations > 0 while at zero capacity
cloudwatch.put_metric_alarm(
    AlarmName=f"{PROJECT_NAME}-{ENDPOINT_NAME}-scale-from-zero-{ENV}",
    AlarmDescription="Scale endpoint from zero when invocations arrive",
    Namespace="AWS/SageMaker",
    MetricName="Invocations",
    Dimensions=[
        {"Name": "EndpointName", "Value": endpoint_name},
        {"Name": "VariantName", "Value": "AllTraffic"},
    ],
    Statistic="Sum",
    Period=60,
    EvaluationPeriods=1,
    Threshold=0,
    ComparisonOperator="GreaterThanThreshold",
    AlarmActions=[scale_from_zero_lambda_arn],  # Lambda ARN
    TreatMissingData="notBreaching",
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)
print(f"Scale-from-zero alarm created for {endpoint_name}")
```

**Create batch transform job:**
```python
from datetime import datetime

sagemaker = boto3.client("sagemaker", region_name=AWS_REGION)

timestamp = datetime.utcnow().strftime("%Y%m%d-%H%M%S")
job_name = f"{PROJECT_NAME}-batch-{ENDPOINT_NAME}-{ENV}-{timestamp}"

sagemaker.create_transform_job(
    TransformJobName=job_name,
    ModelName=f"{PROJECT_NAME}-{ENDPOINT_NAME}-{ENV}",
    TransformInput={
        "DataSource": {
            "S3DataSource": {
                "S3DataType": "S3Prefix",
                "S3Uri": BATCH_INPUT_S3,
            }
        },
        "ContentType": "application/json",
        "SplitType": "Line",
    },
    TransformOutput={
        "S3OutputPath": BATCH_OUTPUT_S3,
        "Accept": "application/json",
        "AssembleWith": "Line",
    },
    TransformResources={
        "InstanceType": "ml.g5.xlarge",
        "InstanceCount": 1,
    },
    MaxConcurrentTransforms=4,
    MaxPayloadInMB=6,
    BatchStrategy="MultiRecord",
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)
print(f"Batch transform job created: {job_name}")

# Poll for completion
import time
while True:
    desc = sagemaker.describe_transform_job(TransformJobName=job_name)
    status = desc["TransformJobStatus"]
    if status in ("Completed", "Failed", "Stopped"):
        break
    print(f"  Status: {status}...")
    time.sleep(30)

print(f"Transform job {status}: {job_name}")
if status == "Completed":
    print(f"  Output: {BATCH_OUTPUT_S3}")
```

**Create multi-model endpoint:**
```python
import time

sagemaker = boto3.client("sagemaker", region_name=AWS_REGION)

multi_model_name = f"{PROJECT_NAME}-multi-model-{ENV}"
endpoint_config_name = f"{multi_model_name}-config"

# Create model with MultiModelConfig
sagemaker.create_model(
    ModelName=multi_model_name,
    PrimaryContainer={
        "Image": f"763104351884.dkr.ecr.{AWS_REGION}.amazonaws.com/pytorch-inference:2.0-cpu-py310",
        "Mode": "MultiModel",
        "ModelDataUrl": MODEL_DATA_URL,  # S3 prefix containing multiple model.tar.gz files
        "MultiModelConfig": {
            "ModelCacheSetting": "Enabled",  # Cache models in memory
        },
    },
    ExecutionRoleArn=sagemaker_role_arn,
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)
print(f"Multi-model created: {multi_model_name}")

# Create endpoint configuration
sagemaker.create_endpoint_config(
    EndpointConfigName=endpoint_config_name,
    ProductionVariants=[
        {
            "VariantName": "AllTraffic",
            "ModelName": multi_model_name,
            "InstanceType": "ml.g5.xlarge",
            "InitialInstanceCount": 1,
            "InitialVariantWeight": 1.0,
        }
    ],
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)

# Create endpoint
sagemaker.create_endpoint(
    EndpointName=multi_model_name,
    EndpointConfigName=endpoint_config_name,
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)
print(f"Multi-model endpoint creating: {multi_model_name}")

# Wait for endpoint to be in service
while True:
    desc = sagemaker.describe_endpoint(EndpointName=multi_model_name)
    status = desc["EndpointStatus"]
    if status == "InService":
        break
    elif status == "Failed":
        raise RuntimeError(f"Endpoint creation failed: {desc.get('FailureReason')}")
    print(f"  Status: {status}...")
    time.sleep(30)

print(f"Multi-model endpoint in service: {multi_model_name}")
```

**Invoke multi-model endpoint with target model:**
```python
sagemaker_runtime = boto3.client("sagemaker-runtime", region_name=AWS_REGION)

def invoke_multi_model(endpoint_name, target_model, payload):
    """Invoke a specific model on a multi-model endpoint."""
    response = sagemaker_runtime.invoke_endpoint(
        EndpointName=endpoint_name,
        TargetModel=target_model,  # e.g., "model-a.tar.gz"
        ContentType="application/json",
        Body=json.dumps(payload),
    )
    result = json.loads(response["Body"].read().decode("utf-8"))
    return result

# Example: invoke different models on the same endpoint
result_a = invoke_multi_model(
    f"{PROJECT_NAME}-multi-model-{ENV}",
    "model-a.tar.gz",
    {"inputs": "classify this text"},
)
result_b = invoke_multi_model(
    f"{PROJECT_NAME}-multi-model-{ENV}",
    "model-b.tar.gz",
    {"inputs": "classify this text"},
)
```

**Utilization-based downsizing Lambda:**
```python
import boto3
import os
import json
from datetime import datetime, timedelta

def lambda_handler(event, context):
    """Reduce endpoint capacity when utilization is consistently low."""
    sagemaker = boto3.client("sagemaker")
    autoscaling = boto3.client("application-autoscaling")
    cloudwatch = boto3.client("cloudwatch")
    sns = boto3.client("sns")

    endpoint_name = os.environ["ENDPOINT_NAME"]
    min_capacity = int(os.environ["MIN_CAPACITY"])
    sns_topic_arn = os.environ.get("SNS_TOPIC_ARN")
    resource_id = f"endpoint/{endpoint_name}/variant/AllTraffic"

    # Get current endpoint details
    ep = sagemaker.describe_endpoint(EndpointName=endpoint_name)
    current_count = ep["ProductionVariants"][0]["CurrentInstanceCount"]

    if current_count <= min_capacity:
        print(f"Already at minimum capacity ({min_capacity}). No action.")
        return {"action": "none", "reason": "at_minimum"}

    # Query average utilization over the last 30 minutes
    end_time = datetime.utcnow()
    start_time = end_time - timedelta(minutes=30)

    metrics = cloudwatch.get_metric_data(
        MetricDataQueries=[
            {
                "Id": "cpu_util",
                "MetricStat": {
                    "Metric": {
                        "Namespace": "/aws/sagemaker/Endpoints",
                        "MetricName": "CPUUtilization",
                        "Dimensions": [
                            {"Name": "EndpointName", "Value": endpoint_name},
                            {"Name": "VariantName", "Value": "AllTraffic"},
                        ],
                    },
                    "Period": 300,
                    "Stat": "Average",
                },
            },
        ],
        StartTime=start_time,
        EndTime=end_time,
    )

    values = metrics["MetricDataResults"][0].get("Values", [])
    avg_util = sum(values) / len(values) if values else 0

    threshold = float(os.environ.get("UTILIZATION_THRESHOLD", "20"))
    if avg_util >= threshold:
        print(f"Utilization {avg_util:.1f}% >= threshold {threshold}%. No action.")
        return {"action": "none", "utilization": avg_util}

    # Reduce capacity by 1
    new_count = current_count - 1
    autoscaling.register_scalable_target(
        ServiceNamespace="sagemaker",
        ResourceId=resource_id,
        ScalableDimension="sagemaker:variant:DesiredInstanceCount",
        MinCapacity=min(min_capacity, new_count),
        MaxCapacity=max(new_count, min_capacity),
    )

    action_summary = {
        "action": "downsize",
        "endpoint": endpoint_name,
        "previous_count": current_count,
        "new_count": new_count,
        "avg_utilization": round(avg_util, 1),
        "threshold": threshold,
        "timestamp": datetime.utcnow().isoformat(),
    }

    # Send notification
    if sns_topic_arn:
        sns.publish(
            TopicArn=sns_topic_arn,
            Subject=f"Endpoint downsized: {endpoint_name}",
            Message=json.dumps(action_summary, indent=2),
        )

    print(f"Downsized {endpoint_name}: {current_count} → {new_count} instances")
    return action_summary
```

**Instance type benchmarking:**
```python
import time
import statistics

sagemaker_runtime = boto3.client("sagemaker-runtime", region_name=AWS_REGION)
pricing = boto3.client("pricing", region_name="us-east-1")

INSTANCE_MAP = {
    "inf2": "ml.inf2.xlarge",
    "g5": "ml.g5.xlarge",
    "c7g": "ml.c7g.xlarge",
}

def benchmark_instance(instance_type, model_name, sample_payload, num_requests=100):
    """Benchmark inference latency and throughput for an instance type."""
    # Create temporary endpoint
    temp_config = f"bench-{instance_type.replace('.', '-')}-{int(time.time())}"
    temp_endpoint = f"bench-{instance_type.replace('.', '-')}-{int(time.time())}"

    sagemaker.create_endpoint_config(
        EndpointConfigName=temp_config,
        ProductionVariants=[{
            "VariantName": "AllTraffic",
            "ModelName": model_name,
            "InstanceType": instance_type,
            "InitialInstanceCount": 1,
            "InitialVariantWeight": 1.0,
        }],
    )
    sagemaker.create_endpoint(
        EndpointName=temp_endpoint,
        EndpointConfigName=temp_config,
    )

    # Wait for endpoint
    while True:
        desc = sagemaker.describe_endpoint(EndpointName=temp_endpoint)
        if desc["EndpointStatus"] == "InService":
            break
        time.sleep(30)

    # Run benchmark
    latencies = []
    for _ in range(num_requests):
        start = time.perf_counter()
        sagemaker_runtime.invoke_endpoint(
            EndpointName=temp_endpoint,
            ContentType="application/json",
            Body=json.dumps(sample_payload),
        )
        elapsed_ms = (time.perf_counter() - start) * 1000
        latencies.append(elapsed_ms)

    # Get hourly rate
    hourly_rate = get_sagemaker_on_demand_rate(instance_type) or 0.0

    # Calculate metrics
    throughput_per_sec = 1000 / statistics.mean(latencies)
    cost_per_1k = (hourly_rate / (throughput_per_sec * 3600)) * 1000

    result = {
        "instance_type": instance_type,
        "p50_ms": round(statistics.median(latencies), 1),
        "p95_ms": round(sorted(latencies)[int(0.95 * len(latencies))], 1),
        "p99_ms": round(sorted(latencies)[int(0.99 * len(latencies))], 1),
        "throughput_rps": round(throughput_per_sec, 2),
        "hourly_cost": hourly_rate,
        "cost_per_1k_inferences": round(cost_per_1k, 4),
    }

    # Cleanup
    sagemaker.delete_endpoint(EndpointName=temp_endpoint)
    sagemaker.delete_endpoint_config(EndpointConfigName=temp_config)

    return result


def get_sagemaker_on_demand_rate(instance_type):
    """Get SageMaker hosting on-demand hourly rate from Pricing API."""
    try:
        response = pricing.get_products(
            ServiceCode="AmazonSageMaker",
            Filters=[
                {"Type": "TERM_MATCH", "Field": "instanceType", "Value": instance_type},
                {"Type": "TERM_MATCH", "Field": "location", "Value": "US East (N. Virginia)"},
                {"Type": "TERM_MATCH", "Field": "component", "Value": "Hosting Instance"},
            ],
            MaxResults=5,
        )
        for price_item in response.get("PriceList", []):
            item = json.loads(price_item)
            terms = item.get("terms", {}).get("OnDemand", {})
            for term in terms.values():
                for dimension in term.get("priceDimensions", {}).values():
                    price = float(dimension["pricePerUnit"]["USD"])
                    if price > 0:
                        return price
    except Exception as e:
        print(f"  Warning: Could not get pricing for {instance_type}: {e}")
    return None
```

---

## Integration Points

- **Upstream**: `devops/04` → IAM roles for Application Auto Scaling service-linked role, SageMaker endpoint management, Lambda execution role, CloudWatch permissions
- **Upstream**: `mlops/03` → SageMaker inference endpoints that this template optimizes with auto-scaling, scale-to-zero, and instance type selection
- **Downstream**: `devops/13` → Cost-per-inference dashboards consume the scaling metrics and per-endpoint cost data generated by this template
- **Downstream**: `finops/01` → Cost allocation and tracking integrates with the per-endpoint cost metrics and batch transform cost data
- **Downstream**: `finops/05` → FinOps dashboards display inference cost optimization metrics: scaling events, utilization trends, cost-per-inference, and savings from scale-to-zero