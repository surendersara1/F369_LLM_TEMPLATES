<!-- Template Version: 1.0 | boto3: 1.35+ | sagemaker: 2.232+ -->

# Template FinOps 02 — Spot Instance Strategies for ML Training

## Purpose
Generate a production-ready SageMaker managed spot training framework: spot training job configuration with checkpointing for interruption recovery, a spot savings calculator using the AWS Pricing API to compare on-demand vs spot costs across instance types, a fallback-to-on-demand pattern with CloudWatch interruption detection and automatic retry, instance type diversification recommendations for maximizing spot capacity availability, and CloudWatch alarms with SNS notifications for spot interruption events.

---

## Role Definition

You are an expert AWS ML cost optimization engineer and SageMaker spot training specialist with expertise in:
- SageMaker Managed Spot Training: `EnableManagedSpotTraining`, `MaxWaitTimeInSeconds`, `MaxRuntimeInSeconds`, checkpoint configuration, spot capacity handling
- S3 Checkpointing: checkpoint save/restore patterns, `CheckpointConfig` with S3 URI, framework-specific checkpoint integration (PyTorch, TensorFlow, XGBoost)
- AWS Pricing API: `pricing.get_products()` for on-demand and spot price comparison, service code filters, attribute filters for instance types and regions
- SageMaker Training Jobs: `sagemaker.create_training_job()`, `describe_training_job()`, training job lifecycle, `SecondaryStatus` monitoring, `BillableTimeInSeconds` for savings calculation
- Spot interruption handling: CloudWatch metrics for `SpotInterruption`, automatic retry logic, fallback to on-demand instances
- Instance type diversification: GPU instance families (ml.p3, ml.p4d, ml.p5, ml.g5, ml.g6), CPU instance families (ml.m5, ml.m6i, ml.c5, ml.c6i), capacity pool strategies
- CloudWatch Alarms: metric filters for spot interruptions, composite alarms, SNS notification integration
- Cost analysis: spot savings percentage calculation, training cost attribution, Cost Explorer integration for historical spot usage

Generate complete, production-deployable cost optimization code.

---

## Context & Inputs

```
PROJECT_NAME:               [REQUIRED]
AWS_REGION:                 [REQUIRED]
AWS_ACCOUNT_ID:             [REQUIRED]
ENV:                        [REQUIRED - dev | stage | prod]

TRAINING_INSTANCE_TYPES:    [REQUIRED - JSON list of instance types for spot diversification]
                            Multiple instance types increase spot capacity availability.
                            Example:
                            ["ml.p3.2xlarge", "ml.p3.8xlarge", "ml.g5.2xlarge", "ml.g5.4xlarge"]
                            - GPU training: ml.p3, ml.p4d, ml.p5, ml.g5, ml.g6 families
                            - CPU training: ml.m5, ml.m6i, ml.c5, ml.c6i families

CHECKPOINT_S3_PATH:         [REQUIRED - S3 path for training checkpoints]
                            Checkpoints enable training resumption after spot interruptions.
                            Example: "s3://my-project-checkpoints-prod/training/"

MAX_WAIT_TIME:              [OPTIONAL: 86400 - maximum seconds to wait for spot capacity]
                            Total time (training + waiting) before the job fails.
                            Default 86400 = 24 hours. Must be >= MAX_RUNTIME.

MAX_RUNTIME:                [OPTIONAL: 43200 - maximum training runtime in seconds]
                            Maximum seconds the training job can run.
                            Default 43200 = 12 hours.

FALLBACK_TO_ON_DEMAND:      [OPTIONAL: true - automatically retry with on-demand on repeated interruptions]
                            When true, retries the training job with on-demand instances
                            after a configurable number of spot interruptions.

MAX_SPOT_RETRIES:           [OPTIONAL: 3 - number of spot attempts before falling back to on-demand]
                            Number of spot training attempts before switching to on-demand.

SPOT_SAVINGS_COMPARISON:    [OPTIONAL: true - run spot vs on-demand cost comparison]
                            When true, queries the Pricing API to calculate potential savings
                            for each instance type in TRAINING_INSTANCE_TYPES.

SNS_TOPIC_ARN:              [OPTIONAL - SNS topic for spot interruption notifications]
                            If not provided, a new topic is created:
                            {PROJECT_NAME}-spot-interruptions-{ENV}

TRAINING_ROLE_ARN:          [OPTIONAL - SageMaker execution role ARN]
                            If not provided, references role from devops/04.

TRAINING_IMAGE_URI:         [OPTIONAL - training container image URI]
                            SageMaker built-in algorithm or custom ECR image.
                            Example: "763104351884.dkr.ecr.us-east-1.amazonaws.com/pytorch-training:2.1-gpu-py310"

TRAINING_HYPERPARAMETERS:   [OPTIONAL: {} - JSON object of training hyperparameters]
                            Example: {"epochs": "50", "batch_size": "32", "learning_rate": "0.001"}

INPUT_DATA_S3:              [OPTIONAL - S3 path to training data]
                            Example: "s3://my-project-data-prod/training/"

OUTPUT_DATA_S3:             [OPTIONAL - S3 path for model artifacts]
                            Example: "s3://my-project-models-prod/output/"
```

---

## Task

Generate complete SageMaker spot training cost optimization framework:

```
{PROJECT_NAME}-spot-training/
├── spot_training/
│   ├── create_spot_training_job.py       # SageMaker managed spot training job configuration
│   ├── checkpoint_config.py              # S3 checkpointing for interruption recovery
│   └── fallback_handler.py              # Fallback-to-on-demand pattern with retry logic
├── cost_analysis/
│   ├── spot_savings_calculator.py        # Spot vs on-demand pricing comparison using Pricing API
│   ├── training_cost_tracker.py          # Track actual spot savings from completed training jobs
│   └── instance_diversification.py       # Instance type diversification recommendations
├── monitoring/
│   ├── spot_interruption_alarm.py        # CloudWatch alarm for spot interruptions
│   ├── spot_metrics_dashboard.py         # CloudWatch dashboard for spot training metrics
│   └── interruption_notifier.py          # SNS notification with interruption details and checkpoint status
├── infrastructure/
│   ├── config.py                         # Central configuration
│   └── sns_topics.py                     # SNS topics for spot interruption alerts
├── run_setup.py                          # CLI orchestrator
└── requirements.txt
```

**config.py**: Central configuration dataclass with all parameters. Load from environment variables or CLI args. Validate required fields. Parse TRAINING_INSTANCE_TYPES from JSON string. Validate MAX_WAIT_TIME >= MAX_RUNTIME. Construct resource names using `{PROJECT_NAME}-{component}-{ENV}` convention.

**create_spot_training_job.py**: Create SageMaker managed spot training job:
- Call `sagemaker.create_training_job()` with `EnableManagedSpotTraining=True`
- Set `StoppingCondition` with `MaxRuntimeInSeconds` (MAX_RUNTIME) and `MaxWaitTimeInSeconds` (MAX_WAIT_TIME)
- Configure `CheckpointConfig` with `S3Uri` pointing to CHECKPOINT_S3_PATH and `LocalPath` set to `/opt/ml/checkpoints`
- Set `ResourceConfig` with instance type from TRAINING_INSTANCE_TYPES (first entry as primary)
- Set `InputDataConfig` with S3 training data channel
- Set `OutputDataConfig` with S3 model artifact path
- Set `HyperParameters` from TRAINING_HYPERPARAMETERS
- Tag with Project, Environment, SpotTraining=true
- Training job name: `{PROJECT_NAME}-spot-train-{timestamp}-{ENV}`
- Poll job status using `sagemaker.describe_training_job()` and report `SecondaryStatus` transitions
- On completion, calculate spot savings: `1 - (BillableTimeInSeconds / TrainingTimeInSeconds)`

**checkpoint_config.py**: S3 checkpointing configuration and utilities:
- `get_checkpoint_config(checkpoint_s3_path)`: Return `CheckpointConfig` dict for `create_training_job()`
- `list_checkpoints(checkpoint_s3_path)`: List existing checkpoints in S3 using `s3.list_objects_v2()` to verify checkpoint availability before resuming
- `cleanup_old_checkpoints(checkpoint_s3_path, keep_latest_n)`: Delete old checkpoints to manage S3 storage costs, keeping only the N most recent
- `validate_checkpoint_path(checkpoint_s3_path)`: Verify S3 path exists and IAM role has read/write access
- Include framework-specific checkpoint guidance:
  - PyTorch: `torch.save()` to `/opt/ml/checkpoints/` with epoch number in filename
  - TensorFlow: `tf.train.Checkpoint` with `CheckpointManager` pointing to `/opt/ml/checkpoints/`
  - XGBoost: `save_model()` to `/opt/ml/checkpoints/` after each boosting round

**fallback_handler.py**: Fallback-to-on-demand pattern with retry logic:
- `run_with_spot_fallback(training_config, max_retries)`: Attempt spot training up to MAX_SPOT_RETRIES times
- On each attempt, call `create_spot_training_job()` and monitor for completion
- If training job fails with `InternalServerError` or status shows spot interruption (check `SecondaryStatus` for `Interrupted`), increment retry counter
- After MAX_SPOT_RETRIES spot failures, retry with `EnableManagedSpotTraining=False` (on-demand)
- Log each attempt with instance type, duration, and outcome
- If FALLBACK_TO_ON_DEMAND is false, raise an exception after max retries instead of falling back
- Publish retry metrics to CloudWatch: `SpotRetryCount`, `FallbackToOnDemand` (0 or 1)
- Return final training job ARN and cost summary

**spot_savings_calculator.py**: Spot vs on-demand pricing comparison:
- Query AWS Pricing API using `pricing.get_products()` with service code `AmazonSageMaker`
- For each instance type in TRAINING_INSTANCE_TYPES:
  - Filter by `instanceType`, `regionCode`, and `usagetype` containing `SpotUsage` vs `Usage`
  - Extract on-demand price per hour and spot price per hour
  - Calculate savings percentage: `(1 - spot_price / on_demand_price) * 100`
  - Calculate estimated savings for a given training duration
- Output comparison table: instance type, on-demand $/hr, spot $/hr, savings %, estimated savings for MAX_RUNTIME
- Recommend the instance type with the best savings-to-capacity ratio
- Cache pricing data locally to avoid repeated API calls (pricing changes infrequently)

**training_cost_tracker.py**: Track actual spot savings from completed training jobs:
- Query completed training jobs using `sagemaker.list_training_jobs()` filtered by tag `SpotTraining=true`
- For each job, call `sagemaker.describe_training_job()` and extract:
  - `TrainingTimeInSeconds`: wall-clock training time
  - `BillableTimeInSeconds`: actual billed time (spot discount applied)
  - `ResourceConfig.InstanceType` and `InstanceCount`
- Calculate actual savings: `(TrainingTimeInSeconds - BillableTimeInSeconds) / TrainingTimeInSeconds * 100`
- Aggregate savings over configurable time period (7/30/90 days)
- Publish savings metrics to CloudWatch namespace `{PROJECT_NAME}/SpotSavings`:
  - `ActualSavingsPercent`, `TotalSpotHours`, `TotalOnDemandEquivalentCost`, `TotalSpotCost`
- Output savings report as JSON

**instance_diversification.py**: Instance type diversification recommendations:
- For each instance type in TRAINING_INSTANCE_TYPES, check spot capacity availability patterns
- Group instance types by family (p3, p4d, p5, g5, g6, m5, c5) and size
- Recommend diversification strategy:
  - Include at least 2-3 instance families for GPU training
  - Include multiple sizes within each family (e.g., 2xlarge and 4xlarge)
  - Prefer newer generation instances (g5 over p3) for better spot availability
- Query historical spot interruption rates using CloudWatch metrics (if available)
- Output diversification score: higher score = better diversification across capacity pools
- Recommend additional instance types if diversification is low

**spot_interruption_alarm.py**: CloudWatch alarm for spot interruptions:
- Create CloudWatch alarm using `cloudwatch.put_metric_alarm()` that monitors SageMaker training job metrics
- Alarm triggers when a training job's `SecondaryStatus` transitions to spot interruption
- Create metric filter on CloudWatch Logs (if training job logging is enabled) for interruption events
- Alarm name: `{PROJECT_NAME}-spot-interruption-{ENV}`
- Alarm action: SNS topic for immediate notification
- Include composite alarm that triggers when interruption count exceeds threshold within a time window

**spot_metrics_dashboard.py**: CloudWatch dashboard for spot training metrics:
- Create dashboard using `cloudwatch.put_dashboard()` with widgets:
  - Spot savings percentage over time (line chart)
  - Spot vs on-demand training hours (stacked bar chart)
  - Spot interruption count by instance type (bar chart)
  - Cumulative cost savings (number widget)
  - Active spot training jobs (number widget)
  - Fallback-to-on-demand events (number widget)
- Dashboard name: `{PROJECT_NAME}-spot-training-{ENV}`
- Use metric math for derived metrics (savings %, cost per training hour)

**interruption_notifier.py**: SNS notification with interruption details:
- Publish SNS message when spot interruption is detected containing:
  - Training job name and ARN
  - Instance type that was interrupted
  - Training duration before interruption
  - Checkpoint status (last checkpoint timestamp and S3 path)
  - Retry attempt number
  - Whether fallback to on-demand will be attempted
- Message format: structured JSON for programmatic consumption
- Subject line: `[{ENV}] Spot Interruption — {training_job_name}`

**sns_topics.py**: Create SNS topics for spot interruption alerts:
- Topic name: `{PROJECT_NAME}-spot-interruptions-{ENV}`
- Subscribe notification endpoints (email, Lambda, etc.)
- Set topic policy allowing CloudWatch Alarms to publish

**run_setup.py**: CLI orchestrator that runs setup steps in order:
1. Load and validate configuration
2. Create SNS topics for spot interruption notifications
3. Run spot savings calculator for all instance types
4. Create CloudWatch alarms for spot interruptions
5. Create CloudWatch dashboard for spot training metrics
6. Print summary with recommended instance types, estimated savings, alarm names, and dashboard URL

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**Managed Spot Training:** Use SageMaker managed spot training (`EnableManagedSpotTraining=True`) which handles spot capacity requests and interruptions automatically. SageMaker manages the spot lifecycle — you do not need to manage EC2 spot instances directly. Spot training can save up to 70-90% compared to on-demand pricing. The `MaxWaitTimeInSeconds` must be greater than or equal to `MaxRuntimeInSeconds` — it represents the total time budget including waiting for spot capacity. If spot capacity is not available within `MaxWaitTimeInSeconds`, the job fails.

**Checkpointing:** Always configure checkpointing for spot training jobs. Without checkpoints, a spot interruption means restarting training from scratch. Set `CheckpointConfig.S3Uri` to a dedicated S3 path and `CheckpointConfig.LocalPath` to `/opt/ml/checkpoints`. Training code must save checkpoints periodically (every N epochs or N steps) and restore from the latest checkpoint on startup. SageMaker automatically syncs `/opt/ml/checkpoints` to S3. Checkpoint frequency is a tradeoff: too frequent wastes I/O time, too infrequent loses more progress on interruption. Recommended: checkpoint every 10-15 minutes of training time.

**Fallback Pattern:** When spot capacity is repeatedly unavailable or interrupted, fall back to on-demand instances to ensure training completes. Track spot retry count and switch to on-demand after MAX_SPOT_RETRIES failures. Log the fallback event for cost analysis. In production, always enable fallback to guarantee training completion for critical model updates.

**Instance Diversification:** Use multiple instance types from different families and sizes to maximize spot capacity availability. Each instance type draws from a separate capacity pool. Diversifying across pools reduces the probability of simultaneous unavailability. For GPU training, combine instances from at least 2 families (e.g., ml.g5 + ml.p3). For CPU training, combine at least 2 families (e.g., ml.m5 + ml.c5). Newer generation instances (g5, g6) often have better spot availability than older generations (p3).

**Pricing API:** The AWS Pricing API (`pricing.get_products()`) is available only in `us-east-1` and `ap-south-1` regions. Always create the pricing client in `us-east-1` regardless of the training region. Pricing data is updated periodically — cache results for at least 1 hour. SageMaker spot pricing is typically 60-90% off on-demand, but varies by instance type and region.

**Cost Tracking:** After each spot training job completes, calculate actual savings using `BillableTimeInSeconds` vs `TrainingTimeInSeconds` from `describe_training_job()`. `BillableTimeInSeconds` reflects the spot-discounted billing. Publish savings metrics to CloudWatch for dashboard visualization and trend analysis. Track cumulative savings over time to demonstrate ROI of spot training adoption.

**Security:** SageMaker execution role needs `s3:GetObject` and `s3:PutObject` on the checkpoint S3 path. Checkpoint data may contain model weights — encrypt the S3 path using KMS keys from `devops/08`. The Pricing API requires `pricing:GetProducts` permission (read-only). CloudWatch alarm actions require `sns:Publish` permission on the notification topic.

**Naming:** All resources follow `{PROJECT_NAME}-{component}-{ENV}` convention:
- Training job: `{PROJECT_NAME}-spot-train-{timestamp}-{ENV}`
- SNS topic: `{PROJECT_NAME}-spot-interruptions-{ENV}`
- CloudWatch alarm: `{PROJECT_NAME}-spot-interruption-{ENV}`
- CloudWatch dashboard: `{PROJECT_NAME}-spot-training-{ENV}`
- CloudWatch namespace: `{PROJECT_NAME}/SpotSavings`

---

## Code Scaffolding Hints

**Create SageMaker managed spot training job with checkpointing:**
```python
import boto3
import json
from datetime import datetime

sagemaker = boto3.client("sagemaker", region_name=AWS_REGION)

def create_spot_training_job(
    project_name, env, instance_type, checkpoint_s3_path,
    max_wait_time, max_runtime, training_image_uri, role_arn,
    input_data_s3, output_data_s3, hyperparameters=None,
):
    """Create a SageMaker managed spot training job with checkpointing."""
    timestamp = datetime.utcnow().strftime("%Y%m%d-%H%M%S")
    job_name = f"{project_name}-spot-train-{timestamp}-{env}"

    training_params = {
        "TrainingJobName": job_name,
        "AlgorithmSpecification": {
            "TrainingImage": training_image_uri,
            "TrainingInputMode": "File",
        },
        "RoleArn": role_arn,
        "InputDataConfig": [
            {
                "ChannelName": "training",
                "DataSource": {
                    "S3DataSource": {
                        "S3DataType": "S3Prefix",
                        "S3Uri": input_data_s3,
                        "S3DataDistributionType": "FullyReplicated",
                    }
                },
                "ContentType": "application/x-parquet",
            }
        ],
        "OutputDataConfig": {
            "S3OutputPath": output_data_s3,
        },
        "ResourceConfig": {
            "InstanceType": instance_type,
            "InstanceCount": 1,
            "VolumeSizeInGB": 50,
        },
        "StoppingCondition": {
            "MaxRuntimeInSeconds": max_runtime,
            "MaxWaitTimeInSeconds": max_wait_time,
        },
        # Enable managed spot training
        "EnableManagedSpotTraining": True,
        # Checkpoint configuration for interruption recovery
        "CheckpointConfig": {
            "S3Uri": f"{checkpoint_s3_path}{job_name}/",
            "LocalPath": "/opt/ml/checkpoints",
        },
        "HyperParameters": hyperparameters or {},
        "Tags": [
            {"Key": "Project", "Value": project_name},
            {"Key": "Environment", "Value": env},
            {"Key": "SpotTraining", "Value": "true"},
            {"Key": "InstanceType", "Value": instance_type},
        ],
    }

    response = sagemaker.create_training_job(**training_params)
    print(f"Spot training job created: {job_name}")
    print(f"  Instance type: {instance_type}")
    print(f"  Max runtime: {max_runtime}s, Max wait: {max_wait_time}s")
    print(f"  Checkpoints: {checkpoint_s3_path}{job_name}/")
    return job_name


def wait_for_training_job(job_name):
    """Poll training job status and return results with savings info."""
    import time

    while True:
        response = sagemaker.describe_training_job(TrainingJobName=job_name)
        status = response["TrainingJobStatus"]
        secondary = response.get("SecondaryStatus", "")
        print(f"  Status: {status} | Secondary: {secondary}")

        if status in ("Completed", "Failed", "Stopped"):
            break
        time.sleep(30)

    if status == "Completed":
        training_time = response.get("TrainingTimeInSeconds", 0)
        billable_time = response.get("BillableTimeInSeconds", 0)
        if training_time > 0:
            savings_pct = (1 - billable_time / training_time) * 100
            print(f"  Training time: {training_time}s")
            print(f"  Billable time: {billable_time}s")
            print(f"  Spot savings: {savings_pct:.1f}%")
        return {
            "job_name": job_name,
            "status": status,
            "training_time": training_time,
            "billable_time": billable_time,
            "savings_pct": round(savings_pct, 1) if training_time > 0 else 0,
        }
    else:
        failure_reason = response.get("FailureReason", "Unknown")
        print(f"  Training failed: {failure_reason}")
        return {"job_name": job_name, "status": status, "failure_reason": failure_reason}
```

**Spot savings calculator using Pricing API:**
```python
# Pricing API is only available in us-east-1 and ap-south-1
pricing = boto3.client("pricing", region_name="us-east-1")

def get_sagemaker_pricing(instance_type, region):
    """Get on-demand and spot pricing for a SageMaker instance type."""
    # Map region name to Pricing API location format
    region_map = {
        "us-east-1": "US East (N. Virginia)",
        "us-west-2": "US West (Oregon)",
        "eu-west-1": "EU (Ireland)",
        "ap-northeast-1": "Asia Pacific (Tokyo)",
    }
    location = region_map.get(region, region)

    # Query on-demand pricing
    on_demand_response = pricing.get_products(
        ServiceCode="AmazonSageMaker",
        Filters=[
            {"Type": "TERM_MATCH", "Field": "instanceType", "Value": instance_type},
            {"Type": "TERM_MATCH", "Field": "location", "Value": location},
            {"Type": "TERM_MATCH", "Field": "component", "Value": "Training Instance"},
        ],
        MaxResults=10,
    )

    on_demand_price = None
    for price_item in on_demand_response.get("PriceList", []):
        item = json.loads(price_item)
        terms = item.get("terms", {}).get("OnDemand", {})
        for term in terms.values():
            for dimension in term.get("priceDimensions", {}).values():
                price = float(dimension["pricePerUnit"]["USD"])
                if price > 0:
                    on_demand_price = price
                    break

    return {"instance_type": instance_type, "on_demand_price_per_hour": on_demand_price}


def calculate_spot_savings(training_instance_types, region, training_hours):
    """Compare spot vs on-demand costs for multiple instance types."""
    results = []
    for instance_type in training_instance_types:
        pricing_info = get_sagemaker_pricing(instance_type, region)
        on_demand = pricing_info.get("on_demand_price_per_hour")
        if on_demand is None:
            print(f"  {instance_type}: Pricing not available")
            continue

        # SageMaker managed spot typically saves 60-90%
        # Exact spot price varies; estimate using 70% average savings
        estimated_spot_savings_pct = 70.0
        estimated_spot_price = on_demand * (1 - estimated_spot_savings_pct / 100)
        on_demand_total = on_demand * training_hours
        spot_total = estimated_spot_price * training_hours
        estimated_savings = on_demand_total - spot_total

        result = {
            "instance_type": instance_type,
            "on_demand_per_hour": round(on_demand, 4),
            "estimated_spot_per_hour": round(estimated_spot_price, 4),
            "estimated_savings_pct": estimated_spot_savings_pct,
            "on_demand_total": round(on_demand_total, 2),
            "spot_total": round(spot_total, 2),
            "estimated_savings": round(estimated_savings, 2),
        }
        results.append(result)
        print(f"  {instance_type}: ${on_demand:.4f}/hr on-demand, "
              f"~${estimated_spot_price:.4f}/hr spot ({estimated_spot_savings_pct}% savings)")

    # Sort by estimated savings descending
    results.sort(key=lambda x: x["estimated_savings"], reverse=True)
    if results:
        best = results[0]
        print(f"\n  Recommended: {best['instance_type']} "
              f"(~${best['estimated_savings']:.2f} savings for {training_hours}hr)")
    return results
```

**Fallback-to-on-demand pattern with retry logic:**
```python
def run_with_spot_fallback(
    project_name, env, training_instance_types, checkpoint_s3_path,
    max_wait_time, max_runtime, training_image_uri, role_arn,
    input_data_s3, output_data_s3, hyperparameters=None,
    max_spot_retries=3, fallback_to_on_demand=True,
):
    """Run training with spot instances, falling back to on-demand on repeated failures."""
    cloudwatch = boto3.client("cloudwatch", region_name=AWS_REGION)
    attempt = 0
    instance_idx = 0

    while attempt < max_spot_retries:
        # Rotate through instance types for diversification
        instance_type = training_instance_types[instance_idx % len(training_instance_types)]
        attempt += 1
        print(f"\nSpot training attempt {attempt}/{max_spot_retries} "
              f"with {instance_type}")

        job_name = create_spot_training_job(
            project_name=project_name,
            env=env,
            instance_type=instance_type,
            checkpoint_s3_path=checkpoint_s3_path,
            max_wait_time=max_wait_time,
            max_runtime=max_runtime,
            training_image_uri=training_image_uri,
            role_arn=role_arn,
            input_data_s3=input_data_s3,
            output_data_s3=output_data_s3,
            hyperparameters=hyperparameters,
        )

        result = wait_for_training_job(job_name)

        if result["status"] == "Completed":
            # Publish success metric
            cloudwatch.put_metric_data(
                Namespace=f"{project_name}/SpotSavings",
                MetricData=[{
                    "MetricName": "SpotTrainingSuccess",
                    "Dimensions": [
                        {"Name": "Environment", "Value": env},
                        {"Name": "InstanceType", "Value": instance_type},
                    ],
                    "Value": 1,
                    "Unit": "Count",
                }],
            )
            return result

        # Spot interruption or capacity issue — try next instance type
        print(f"  Spot attempt {attempt} failed: {result.get('failure_reason', 'Unknown')}")
        instance_idx += 1

        # Publish retry metric
        cloudwatch.put_metric_data(
            Namespace=f"{project_name}/SpotSavings",
            MetricData=[{
                "MetricName": "SpotRetryCount",
                "Dimensions": [{"Name": "Environment", "Value": env}],
                "Value": 1,
                "Unit": "Count",
            }],
        )

    # All spot attempts exhausted — fallback to on-demand
    if fallback_to_on_demand:
        print(f"\nFalling back to on-demand after {max_spot_retries} spot failures")
        instance_type = training_instance_types[0]

        # Publish fallback metric
        cloudwatch.put_metric_data(
            Namespace=f"{project_name}/SpotSavings",
            MetricData=[{
                "MetricName": "FallbackToOnDemand",
                "Dimensions": [{"Name": "Environment", "Value": env}],
                "Value": 1,
                "Unit": "Count",
            }],
        )

        timestamp = datetime.utcnow().strftime("%Y%m%d-%H%M%S")
        job_name = f"{project_name}-ondemand-train-{timestamp}-{env}"

        sagemaker.create_training_job(
            TrainingJobName=job_name,
            AlgorithmSpecification={
                "TrainingImage": training_image_uri,
                "TrainingInputMode": "File",
            },
            RoleArn=role_arn,
            InputDataConfig=[{
                "ChannelName": "training",
                "DataSource": {
                    "S3DataSource": {
                        "S3DataType": "S3Prefix",
                        "S3Uri": input_data_s3,
                        "S3DataDistributionType": "FullyReplicated",
                    }
                },
            }],
            OutputDataConfig={"S3OutputPath": output_data_s3},
            ResourceConfig={
                "InstanceType": instance_type,
                "InstanceCount": 1,
                "VolumeSizeInGB": 50,
            },
            StoppingCondition={"MaxRuntimeInSeconds": max_runtime},
            EnableManagedSpotTraining=False,  # On-demand fallback
            CheckpointConfig={
                "S3Uri": f"{checkpoint_s3_path}{job_name}/",
                "LocalPath": "/opt/ml/checkpoints",
            },
            HyperParameters=hyperparameters or {},
            Tags=[
                {"Key": "Project", "Value": project_name},
                {"Key": "Environment", "Value": env},
                {"Key": "SpotTraining", "Value": "false"},
                {"Key": "SpotFallback", "Value": "true"},
            ],
        )
        print(f"On-demand training job created: {job_name}")
        return wait_for_training_job(job_name)
    else:
        raise RuntimeError(
            f"Spot training failed after {max_spot_retries} attempts. "
            "Fallback to on-demand is disabled."
        )
```

**CloudWatch alarm for spot interruptions:**
```python
cloudwatch = boto3.client("cloudwatch", region_name=AWS_REGION)
sns = boto3.client("sns", region_name=AWS_REGION)

def create_spot_interruption_alarm(project_name, env, sns_topic_arn):
    """Create CloudWatch alarm for spot training interruptions."""
    alarm_name = f"{project_name}-spot-interruption-{env}"

    cloudwatch.put_metric_alarm(
        AlarmName=alarm_name,
        AlarmDescription=(
            f"Spot training interruption detected for {project_name} ({env}). "
            "Check checkpoint status and consider instance diversification."
        ),
        Namespace=f"{project_name}/SpotSavings",
        MetricName="SpotRetryCount",
        Dimensions=[{"Name": "Environment", "Value": env}],
        Statistic="Sum",
        Period=300,  # 5 minutes
        EvaluationPeriods=1,
        Threshold=1.0,
        ComparisonOperator="GreaterThanOrEqualToThreshold",
        TreatMissingData="notBreaching",
        AlarmActions=[sns_topic_arn],
        OKActions=[sns_topic_arn],
        Tags=[
            {"Key": "Project", "Value": project_name},
            {"Key": "Environment", "Value": env},
        ],
    )
    print(f"Spot interruption alarm created: {alarm_name}")

    # Create composite alarm for repeated interruptions
    composite_alarm_name = f"{project_name}-spot-repeated-interruptions-{env}"
    cloudwatch.put_metric_alarm(
        AlarmName=composite_alarm_name,
        AlarmDescription=(
            f"Multiple spot interruptions detected for {project_name} ({env}). "
            "Consider adding more instance types or switching to on-demand."
        ),
        Namespace=f"{project_name}/SpotSavings",
        MetricName="SpotRetryCount",
        Dimensions=[{"Name": "Environment", "Value": env}],
        Statistic="Sum",
        Period=3600,  # 1 hour
        EvaluationPeriods=1,
        Threshold=3.0,  # 3+ interruptions in 1 hour
        ComparisonOperator="GreaterThanOrEqualToThreshold",
        TreatMissingData="notBreaching",
        AlarmActions=[sns_topic_arn],
        Tags=[
            {"Key": "Project", "Value": project_name},
            {"Key": "Environment", "Value": env},
        ],
    )
    print(f"Repeated interruption alarm created: {composite_alarm_name}")
    return alarm_name
```

**Track actual spot savings from completed training jobs:**
```python
from datetime import datetime, timedelta

def track_spot_savings(project_name, env, days=30):
    """Track actual spot savings from completed training jobs."""
    cloudwatch = boto3.client("cloudwatch", region_name=AWS_REGION)

    # List recent training jobs with SpotTraining tag
    paginator = sagemaker.get_paginator("list_training_jobs")
    creation_after = datetime.utcnow() - timedelta(days=days)

    total_training_seconds = 0
    total_billable_seconds = 0
    job_count = 0

    for page in paginator.paginate(
        StatusEquals="Completed",
        CreationTimeAfter=creation_after,
        SortBy="CreationTime",
        SortOrder="Descending",
    ):
        for job_summary in page["TrainingJobSummaries"]:
            job = sagemaker.describe_training_job(
                TrainingJobName=job_summary["TrainingJobName"]
            )
            # Check if this was a spot training job
            if job.get("EnableManagedSpotTraining", False):
                training_time = job.get("TrainingTimeInSeconds", 0)
                billable_time = job.get("BillableTimeInSeconds", 0)
                total_training_seconds += training_time
                total_billable_seconds += billable_time
                job_count += 1

    if total_training_seconds > 0:
        overall_savings_pct = (
            (1 - total_billable_seconds / total_training_seconds) * 100
        )
    else:
        overall_savings_pct = 0

    # Publish aggregate savings metrics
    cloudwatch.put_metric_data(
        Namespace=f"{project_name}/SpotSavings",
        MetricData=[
            {
                "MetricName": "ActualSavingsPercent",
                "Dimensions": [{"Name": "Environment", "Value": env}],
                "Value": overall_savings_pct,
                "Unit": "Percent",
            },
            {
                "MetricName": "TotalSpotTrainingHours",
                "Dimensions": [{"Name": "Environment", "Value": env}],
                "Value": total_billable_seconds / 3600,
                "Unit": "None",
            },
        ],
    )

    report = {
        "period_days": days,
        "spot_training_jobs": job_count,
        "total_training_hours": round(total_training_seconds / 3600, 2),
        "total_billable_hours": round(total_billable_seconds / 3600, 2),
        "overall_savings_pct": round(overall_savings_pct, 1),
    }
    print(json.dumps(report, indent=2))
    return report
```

**SNS notification for spot interruption with checkpoint status:**
```python
def notify_spot_interruption(
    project_name, env, sns_topic_arn, job_name, instance_type,
    training_duration_seconds, checkpoint_s3_path, retry_attempt, will_retry,
):
    """Send SNS notification with spot interruption details."""
    sns = boto3.client("sns", region_name=AWS_REGION)
    s3 = boto3.client("s3", region_name=AWS_REGION)

    # Check latest checkpoint
    bucket, prefix = checkpoint_s3_path.replace("s3://", "").split("/", 1)
    checkpoint_prefix = f"{prefix}{job_name}/"
    checkpoint_objects = s3.list_objects_v2(
        Bucket=bucket, Prefix=checkpoint_prefix, MaxKeys=5
    )
    latest_checkpoint = None
    if checkpoint_objects.get("Contents"):
        latest = sorted(
            checkpoint_objects["Contents"],
            key=lambda x: x["LastModified"],
            reverse=True,
        )[0]
        latest_checkpoint = {
            "key": latest["Key"],
            "last_modified": latest["LastModified"].isoformat(),
            "size_bytes": latest["Size"],
        }

    message = {
        "source": f"{project_name}-spot-training-{env}",
        "event": "SpotInterruption",
        "training_job": job_name,
        "instance_type": instance_type,
        "training_duration_seconds": training_duration_seconds,
        "checkpoint_status": "available" if latest_checkpoint else "none",
        "latest_checkpoint": latest_checkpoint,
        "retry_attempt": retry_attempt,
        "will_retry": will_retry,
        "timestamp": datetime.utcnow().isoformat(),
    }

    sns.publish(
        TopicArn=sns_topic_arn,
        Subject=f"[{env.upper()}] Spot Interruption — {job_name}",
        Message=json.dumps(message, indent=2),
    )
    print(f"Interruption notification sent to {sns_topic_arn}")
```

**CloudWatch dashboard for spot training metrics:**
```python
def create_spot_training_dashboard(project_name, env, region):
    """Create CloudWatch dashboard for spot training metrics."""
    dashboard_name = f"{project_name}-spot-training-{env}"
    namespace = f"{project_name}/SpotSavings"

    dashboard_body = {
        "widgets": [
            {
                "type": "metric",
                "x": 0, "y": 0, "width": 12, "height": 6,
                "properties": {
                    "title": "Spot Savings % Over Time",
                    "metrics": [
                        [namespace, "ActualSavingsPercent",
                         "Environment", env, {"stat": "Average", "period": 86400}],
                    ],
                    "view": "timeSeries",
                    "region": region,
                    "yAxis": {"left": {"min": 0, "max": 100, "label": "Savings %"}},
                },
            },
            {
                "type": "metric",
                "x": 12, "y": 0, "width": 6, "height": 6,
                "properties": {
                    "title": "Cumulative Spot Training Hours",
                    "metrics": [
                        [namespace, "TotalSpotTrainingHours",
                         "Environment", env, {"stat": "Maximum", "period": 86400}],
                    ],
                    "view": "singleValue",
                    "region": region,
                },
            },
            {
                "type": "metric",
                "x": 18, "y": 0, "width": 6, "height": 6,
                "properties": {
                    "title": "Fallback to On-Demand Events",
                    "metrics": [
                        [namespace, "FallbackToOnDemand",
                         "Environment", env, {"stat": "Sum", "period": 86400}],
                    ],
                    "view": "singleValue",
                    "region": region,
                },
            },
            {
                "type": "metric",
                "x": 0, "y": 6, "width": 12, "height": 6,
                "properties": {
                    "title": "Spot Retry Count (Interruptions)",
                    "metrics": [
                        [namespace, "SpotRetryCount",
                         "Environment", env, {"stat": "Sum", "period": 3600}],
                    ],
                    "view": "timeSeries",
                    "region": region,
                },
            },
            {
                "type": "metric",
                "x": 12, "y": 6, "width": 12, "height": 6,
                "properties": {
                    "title": "Spot Training Success vs Failure",
                    "metrics": [
                        [namespace, "SpotTrainingSuccess",
                         "Environment", env, {"stat": "Sum", "period": 86400}],
                        [namespace, "SpotRetryCount",
                         "Environment", env, {"stat": "Sum", "period": 86400}],
                    ],
                    "view": "timeSeries",
                    "region": region,
                },
            },
        ],
    }

    cloudwatch.put_dashboard(
        DashboardName=dashboard_name,
        DashboardBody=json.dumps(dashboard_body),
    )
    print(f"Dashboard created: {dashboard_name}")
    print(f"  URL: https://{region}.console.aws.amazon.com/cloudwatch/"
          f"home?region={region}#dashboards:name={dashboard_name}")
    return dashboard_name
```

---

## Integration Points

- **Upstream**: `devops/04` → IAM roles for SageMaker execution role (S3 checkpoint access, training permissions), Pricing API read permissions, CloudWatch publish permissions
- **Upstream**: `mlops/01` → SageMaker training pipeline provides the base training job configuration that this template enhances with spot training, checkpointing, and fallback patterns
- **Downstream**: `finops/01` → Cost allocation and tracking consumes spot savings metrics and per-job cost data for budget monitoring and experiment cost attribution
- **Downstream**: `finops/05` → FinOps dashboards integrate spot training metrics (savings %, interruption count, fallback events) into the unified ML cost dashboard
- **Related**: `devops/08` → KMS encryption keys for encrypting checkpoint data in S3 at rest
- **Related**: `devops/03` → CloudWatch monitoring and alerting patterns used for spot interruption alarms and dashboard creation
