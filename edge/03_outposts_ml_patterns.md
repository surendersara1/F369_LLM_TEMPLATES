<!-- Template Version: 1.0 | boto3: 1.35+ -->

# Template Edge 03 — AWS Outposts Hybrid ML Patterns

## Purpose
Generate a production-ready hybrid ML architecture on AWS Outposts: SageMaker endpoint deployment targeting Outposts subnets for low-latency local inference, hybrid training patterns with preprocessing on Outposts ECS/EC2 and full training in the parent AWS Region, S3 on Outposts bucket configuration for data-resident workloads, DataSync and S3 replication for bidirectional model artifact and training data synchronization, local inference continuity patterns for degraded or disconnected connectivity scenarios, and CloudWatch agent configuration on Outposts for local metric collection and forwarding.

---

## Role Definition

You are an expert AWS hybrid cloud and edge ML architect specializing in AWS Outposts with expertise in:
- AWS Outposts: rack deployment topology, local subnet configuration, Outposts-supported service instances (EC2, ECS, S3, EBS), and local gateway routing
- SageMaker on Outposts: deploying real-time inference endpoints to Outposts subnets using customer-managed VPC configurations and local instance types
- S3 on Outposts: bucket creation via S3 Outposts access points, data residency enforcement, lifecycle policies, and access point policies
- AWS DataSync: agent-based and agentless transfer tasks between Outposts local storage and regional S3, scheduling, bandwidth throttling, and transfer verification
- Hybrid training patterns: local data preprocessing on Outposts ECS tasks or EC2 instances with regional SageMaker training job submission
- Network resilience: local inference continuity during WAN degradation, health-check-based failover, local result queuing, and automatic resynchronization on reconnect
- CloudWatch agent on Outposts: local metric collection, StatsD/collectd integration, log forwarding, and custom metric namespaces for hybrid fleet monitoring
- IAM policies for cross-service access spanning Outposts local resources and regional AWS services

Generate complete, production-deployable code.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

OUTPOST_ID:             [REQUIRED - Outposts rack ID]
                        Example: op-0123456789abcdef0

OUTPOST_ARN:            [REQUIRED - Outposts rack ARN]
                        Example: arn:aws:outposts:us-east-1:123456789012:outpost/op-0123456789abcdef0

OUTPOST_SUBNET_IDS:     [REQUIRED - list of VPC subnet IDs on the Outpost]
                        Example: ["subnet-0aaa111", "subnet-0bbb222"]

S3_OUTPOSTS_BUCKET:     [REQUIRED - name for the S3 on Outposts bucket]
                        Example: my-project-outpost-data

S3_OUTPOSTS_ACCESS_POINT:
                        [REQUIRED - access point name for S3 on Outposts]
                        Example: my-project-outpost-ap

REGIONAL_S3_BUCKET:     [REQUIRED - regional S3 bucket for model artifacts and synced data]
                        Example: my-project-regional-models

DATA_SYNC_DIRECTION:    [REQUIRED - direction of data synchronization]
                        Options:
                        - outpost_to_region (upload local data to regional S3)
                        - region_to_outpost (download models/data from region to Outpost)
                        - bidirectional (sync in both directions)

LOCAL_INFERENCE_ENABLED:[REQUIRED - deploy SageMaker endpoint on Outposts subnet]
                        Options: true | false

DATA_RESIDENCY_REGION:  [REQUIRED - geographic region for data residency compliance]
                        Example: eu-west-1 (data must remain in this region/Outpost)

MODEL_S3_URI:           [OPTIONAL: "" - S3 URI of the model artifact for local inference]
                        Example: s3://my-bucket/models/my-model.tar.gz

INFERENCE_INSTANCE_TYPE:[OPTIONAL: ml.c5.xlarge - SageMaker instance type available on the Outpost]

ECS_CLUSTER_NAME:       [OPTIONAL: "" - ECS cluster on Outposts for preprocessing tasks]
                        Example: my-project-outpost-ecs

PREPROCESSING_IMAGE_URI:[OPTIONAL: "" - ECR image URI for preprocessing container]
                        Example: 123456789012.dkr.ecr.us-east-1.amazonaws.com/preprocess:latest

DATASYNC_SCHEDULE:      [OPTIONAL: "cron(0 */6 * * ? *)" - DataSync transfer schedule]

DATASYNC_BANDWIDTH_LIMIT_MBPS:
                        [OPTIONAL: 100 - bandwidth throttle for DataSync transfers in Mbps]

CLOUDWATCH_NAMESPACE:   [OPTIONAL: "{PROJECT_NAME}/OutpostsML/{ENV}" - CloudWatch metric namespace]

VPC_ID:                 [OPTIONAL: "" - VPC ID containing the Outposts subnets]

SECURITY_GROUP_IDS:     [OPTIONAL: [] - security group IDs for Outposts resources]
```

---

## Task

Generate complete AWS Outposts hybrid ML architecture:

```
{PROJECT_NAME}-outposts-ml/
├── infrastructure/
│   ├── s3_outposts.py                 # S3 on Outposts bucket + access point setup
│   ├── datasync_tasks.py              # DataSync transfer tasks (Outpost ↔ Region)
│   └── config.py                      # Central configuration
├── inference/
│   ├── deploy_endpoint.py             # SageMaker endpoint on Outposts subnet
│   ├── invoke_endpoint.py             # Invoke local inference endpoint
│   └── inference_continuity.py        # Local inference during degraded connectivity
├── training/
│   ├── preprocess_ecs.py              # ECS task on Outposts for local preprocessing
│   ├── submit_training.py             # Submit SageMaker training job in Region
│   └── hybrid_pipeline.py            # Orchestrate preprocess-on-Outpost → train-in-Region
├── monitoring/
│   ├── cloudwatch_agent_config.json   # CloudWatch agent config for Outposts instances
│   ├── outposts_dashboard.py          # CloudWatch dashboard for hybrid fleet
│   └── connectivity_monitor.py        # WAN connectivity health check + alerting
├── cleanup/
│   └── delete_resources.py            # Delete all Outposts ML resources
├── run_setup.py                       # CLI orchestrator
└── requirements.txt
```

**config.py**: Central configuration dataclass with all parameters. Load from environment variables or CLI args. Validate required fields. Construct resource names using `{PROJECT_NAME}-{component}-{ENV}` convention. Validate that OUTPOST_SUBNET_IDS are non-empty and belong to the specified VPC. Build S3 on Outposts ARN from OUTPOST_ARN and bucket name. Set default CLOUDWATCH_NAMESPACE to `{PROJECT_NAME}/OutpostsML/{ENV}`.

**s3_outposts.py**: Create and configure S3 on Outposts bucket:
- Call `s3outposts.create_bucket()` with `Bucket` (S3_OUTPOSTS_BUCKET), `OutpostId` (OUTPOST_ID) to create a local S3 bucket on the Outpost
- Create an S3 on Outposts access point using `s3control.create_access_point()` with `Name` (S3_OUTPOSTS_ACCESS_POINT), `Bucket` (S3 on Outposts bucket ARN), `VpcConfiguration` restricting access to VPC_ID
- Apply access point policy that grants the SageMaker execution role and ECS task role `s3-outposts:GetObject`, `s3-outposts:PutObject`, `s3-outposts:ListBucket`
- Configure lifecycle rules on the bucket for automatic cleanup of processed data (e.g., delete objects older than 90 days)
- Tag bucket with `Project`, `Environment`, `DataResidency` (DATA_RESIDENCY_REGION)
- Log the bucket ARN and access point ARN for use by other scripts

**datasync_tasks.py**: Configure DataSync transfer tasks between Outposts and Region:
- Create DataSync source and destination locations:
  - For `outpost_to_region`: source = S3 on Outposts (via access point ARN), destination = regional S3 bucket (REGIONAL_S3_BUCKET)
  - For `region_to_outpost`: source = regional S3 bucket, destination = S3 on Outposts
  - For `bidirectional`: create two tasks (one in each direction)
- Call `datasync.create_location_s3()` for regional S3 with `S3BucketArn`, `S3Config` (IAM role ARN for DataSync)
- Call `datasync.create_location_s3()` for S3 on Outposts with `S3BucketArn` (Outposts bucket ARN), `S3Config`, and `S3StorageClass` set to `OUTPOSTS`
- Call `datasync.create_task()` with `SourceLocationArn`, `DestinationLocationArn`, `Name` (`{PROJECT_NAME}-sync-{direction}-{ENV}`), `Schedule` (DATASYNC_SCHEDULE), `Options` including:
  - `BytesPerSecond` set from DATASYNC_BANDWIDTH_LIMIT_MBPS (converted to bytes)
  - `VerifyMode` set to `POINT_IN_TIME_CONSISTENT` for data integrity
  - `OverwriteMode` set to `ALWAYS` for model artifact updates
  - `PreserveDeletedFiles` set to `PRESERVE`
  - `TransferMode` set to `CHANGED` for incremental sync
- Include a manual trigger function using `datasync.start_task_execution()` for on-demand sync
- Include a status check function using `datasync.describe_task_execution()` to monitor transfer progress

**deploy_endpoint.py** (when LOCAL_INFERENCE_ENABLED is true): Deploy SageMaker endpoint on Outposts subnet:
- Create SageMaker model using `sagemaker.create_model()` with `PrimaryContainer` pointing to MODEL_S3_URI and `VpcConfig` with `Subnets` (OUTPOST_SUBNET_IDS) and `SecurityGroupIds` (SECURITY_GROUP_IDS)
- Create endpoint configuration using `sagemaker.create_endpoint_config()` with `ProductionVariants` specifying:
  - `InstanceType` set to INFERENCE_INSTANCE_TYPE (must be an Outposts-supported instance type)
  - `InitialInstanceCount` set to 1 (or configurable)
  - `ModelName` referencing the created model
  - `RoutingConfig` for local traffic routing
- The endpoint is deployed to the Outposts subnet by specifying the Outposts subnet IDs in the model's VpcConfig — SageMaker places the endpoint instances on the Outpost
- Create endpoint using `sagemaker.create_endpoint()` with `EndpointName` (`{PROJECT_NAME}-outpost-endpoint-{ENV}`) and `EndpointConfigName`
- Poll with `sagemaker.describe_endpoint()` until status is `InService`
- Log endpoint name and creation time

**invoke_endpoint.py**: Invoke the local SageMaker endpoint:
- Call `sagemaker_runtime.invoke_endpoint()` with `EndpointName`, `Body` (JSON payload), `ContentType` (`application/json`)
- Parse response body and return prediction results
- Measure and log inference latency
- Include batch invocation helper that processes a list of inputs sequentially
- Publish invocation metrics to CloudWatch: `LocalInferenceLatency`, `LocalInferenceCount`, `LocalInferenceErrors`

**inference_continuity.py**: Local inference continuity during degraded connectivity:
- Implement a health check loop that tests WAN connectivity by attempting to reach the regional SageMaker API endpoint (`sagemaker.{AWS_REGION}.amazonaws.com`) and CloudWatch endpoint
- When connectivity is healthy: invoke the local SageMaker endpoint normally and forward results to regional services
- When connectivity is degraded or lost:
  - Continue invoking the local SageMaker endpoint (it runs on the Outpost and does not require WAN)
  - Queue inference results to a local file-based buffer (`/tmp/{PROJECT_NAME}/continuity_queue/`) or a local DynamoDB table if available
  - Track queue depth and oldest queued result
- When connectivity is restored:
  - Drain the local queue by forwarding buffered results to regional S3 or CloudWatch in chronological order
  - Log resynchronization metrics: queue depth drained, time offline, oldest result age
- Configurable health check interval (default 30 seconds) and max queue size (default 50,000 results)

**preprocess_ecs.py**: Run preprocessing on Outposts ECS cluster:
- Register an ECS task definition using `ecs.register_task_definition()` with:
  - `containerDefinitions` specifying PREPROCESSING_IMAGE_URI, memory/CPU limits, environment variables for S3 on Outposts access point ARN and input/output prefixes
  - `networkMode` set to `awsvpc` for Outposts subnet placement
  - `requiresCompatibilities` set to `["EC2"]` (Fargate is not available on Outposts)
  - `executionRoleArn` and `taskRoleArn` with permissions for S3 on Outposts and CloudWatch
- Run the ECS task using `ecs.run_task()` with:
  - `cluster` set to ECS_CLUSTER_NAME
  - `taskDefinition` referencing the registered definition
  - `networkConfiguration` with `awsvpcConfiguration` specifying OUTPOST_SUBNET_IDS and SECURITY_GROUP_IDS
  - `placementConstraints` with `memberOf` expression targeting Outposts instances (e.g., `attribute:ecs.outpost-arn == {OUTPOST_ARN}`)
- Poll with `ecs.describe_tasks()` until task status is `STOPPED`
- Check exit code and log preprocessing results

**submit_training.py**: Submit SageMaker training job in the parent Region:
- Call `sagemaker.create_training_job()` with:
  - `TrainingJobName` set to `{PROJECT_NAME}-train-{ENV}-{timestamp}`
  - `AlgorithmSpecification` with training image URI and input mode
  - `InputDataConfig` pointing to the regional S3 bucket where preprocessed data was synced from Outposts
  - `OutputDataConfig` pointing to regional S3 for model artifacts
  - `ResourceConfig` with regional instance type and volume size
  - `RoleArn` for SageMaker execution role
  - `StoppingCondition` with `MaxRuntimeInSeconds`
- Poll with `sagemaker.describe_training_job()` until status is `Completed` or `Failed`
- On success, log the model artifact S3 URI for subsequent sync back to Outposts

**hybrid_pipeline.py**: Orchestrate the full hybrid workflow:
1. Run preprocessing on Outposts ECS (calls `preprocess_ecs.py`)
2. Trigger DataSync to upload preprocessed data from Outposts to Region (calls `datasync_tasks.py` manual trigger)
3. Wait for DataSync transfer to complete
4. Submit SageMaker training job in Region (calls `submit_training.py`)
5. Wait for training to complete
6. Trigger DataSync to download trained model from Region to Outposts
7. Optionally update the local SageMaker endpoint with the new model (calls `deploy_endpoint.py`)
- Include error handling at each step with rollback logging
- Publish pipeline stage metrics to CloudWatch

**cloudwatch_agent_config.json**: CloudWatch agent configuration for Outposts EC2/ECS instances:
- Configure `metrics` section with:
  - `namespace` set to CLOUDWATCH_NAMESPACE
  - `metrics_collected`: CPU utilization, memory usage, disk I/O, network throughput
  - `append_dimensions`: `InstanceId`, `OutpostId`, `Environment`
- Configure `logs` section to forward application logs from `/var/log/{PROJECT_NAME}/` to CloudWatch Logs group `/{PROJECT_NAME}/outposts/{ENV}`
- Configure `statsd` listener on port 8125 for custom application metrics (inference latency, queue depth)
- Set `force_flush_interval` to 15 seconds for near-real-time metric delivery
- Include `endpoint_override` for CloudWatch if using VPC endpoints

**outposts_dashboard.py**: Create CloudWatch dashboard for hybrid ML fleet:
- Widget 1: Local inference latency (p50/p95/p99) from the Outposts endpoint
- Widget 2: Local inference throughput (invocations per minute)
- Widget 3: DataSync transfer status (bytes transferred, task duration, success/failure)
- Widget 4: WAN connectivity health (healthy/degraded/disconnected timeline)
- Widget 5: Continuity queue depth (buffered results during disconnection)
- Widget 6: ECS preprocessing task duration and success rate
- Widget 7: Outposts instance CPU/memory utilization
- Widget 8: S3 on Outposts storage utilization
- Dashboard name: `{PROJECT_NAME}-outposts-ml-{ENV}`

**connectivity_monitor.py**: WAN connectivity health check and alerting:
- Periodically test connectivity to regional AWS endpoints (SageMaker, S3, CloudWatch)
- Classify connectivity state: `HEALTHY` (all endpoints reachable, latency < 200ms), `DEGRADED` (reachable but latency > 500ms or packet loss > 5%), `DISCONNECTED` (endpoints unreachable)
- Publish `WANConnectivityStatus` metric to CloudWatch (1=healthy, 0.5=degraded, 0=disconnected)
- Create CloudWatch alarm that triggers SNS notification when connectivity is DEGRADED or DISCONNECTED for more than 5 minutes
- Log connectivity state transitions with timestamps

**delete_resources.py**: Clean up all Outposts ML resources:
- Delete SageMaker endpoint, endpoint config, and model
- Cancel active DataSync task executions and delete tasks and locations
- Delete S3 on Outposts access point and bucket (after emptying)
- Deregister ECS task definitions
- Delete CloudWatch dashboard and alarms
- Optionally delete CloudWatch log groups

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Code Scaffolding Hints

**S3 on Outposts bucket and access point:**
```python
import boto3

s3outposts = boto3.client("s3outposts", region_name=AWS_REGION)
s3control = boto3.client("s3control", region_name=AWS_REGION)

# Create S3 on Outposts bucket
bucket_response = s3outposts.create_bucket(
    Bucket=S3_OUTPOSTS_BUCKET,
    OutpostId=OUTPOST_ID,
)
bucket_arn = bucket_response["BucketArn"]
print(f"S3 on Outposts bucket created: {bucket_arn}")

# Create access point for VPC-restricted access
ap_response = s3control.create_access_point(
    AccountId=AWS_ACCOUNT_ID,
    Name=S3_OUTPOSTS_ACCESS_POINT,
    Bucket=bucket_arn,
    VpcConfiguration={
        "VpcId": VPC_ID,
    },
)
access_point_arn = ap_response["AccessPointArn"]
print(f"Access point created: {access_point_arn}")

# Apply access point policy
import json

access_point_policy = {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowSageMakerAndECS",
            "Effect": "Allow",
            "Principal": {
                "AWS": [
                    f"arn:aws:iam::{AWS_ACCOUNT_ID}:role/{PROJECT_NAME}-sagemaker-role-{ENV}",
                    f"arn:aws:iam::{AWS_ACCOUNT_ID}:role/{PROJECT_NAME}-ecs-task-role-{ENV}",
                ]
            },
            "Action": [
                "s3-outposts:GetObject",
                "s3-outposts:PutObject",
                "s3-outposts:ListBucket",
                "s3-outposts:DeleteObject",
            ],
            "Resource": [
                access_point_arn,
                f"{access_point_arn}/object/*",
            ],
        }
    ],
}

s3control.put_access_point_policy(
    AccountId=AWS_ACCOUNT_ID,
    Name=S3_OUTPOSTS_ACCESS_POINT,
    Policy=json.dumps(access_point_policy),
)
print("Access point policy applied")
```

**S3 on Outposts lifecycle rule:**
```python
s3control.put_bucket_lifecycle_configuration(
    AccountId=AWS_ACCOUNT_ID,
    Bucket=bucket_arn,
    LifecycleConfiguration={
        "Rules": [
            {
                "ID": "CleanupProcessedData",
                "Filter": {
                    "Prefix": "processed/",
                },
                "Status": "Enabled",
                "Expiration": {
                    "Days": 90,
                },
            },
            {
                "ID": "CleanupTempFiles",
                "Filter": {
                    "Prefix": "tmp/",
                },
                "Status": "Enabled",
                "Expiration": {
                    "Days": 7,
                },
            },
        ],
    },
)
print("Lifecycle rules configured")
```

**SageMaker endpoint on Outposts subnet:**
```python
import time

sagemaker = boto3.client("sagemaker", region_name=AWS_REGION)

model_name = f"{PROJECT_NAME}-outpost-model-{ENV}"
endpoint_config_name = f"{PROJECT_NAME}-outpost-epc-{ENV}"
endpoint_name = f"{PROJECT_NAME}-outpost-endpoint-{ENV}"

# Create model with VPC config targeting Outposts subnets
sagemaker.create_model(
    ModelName=model_name,
    PrimaryContainer={
        "Image": f"763104351884.dkr.ecr.{AWS_REGION}.amazonaws.com/pytorch-inference:2.1-cpu-py310",
        "ModelDataUrl": MODEL_S3_URI,
        "Environment": {
            "SAGEMAKER_PROGRAM": "inference.py",
            "SAGEMAKER_SUBMIT_DIRECTORY": MODEL_S3_URI,
        },
    },
    VpcConfig={
        "Subnets": OUTPOST_SUBNET_IDS,          # Outposts subnets force placement on Outpost
        "SecurityGroupIds": SECURITY_GROUP_IDS,
    },
    ExecutionRoleArn=f"arn:aws:iam::{AWS_ACCOUNT_ID}:role/{PROJECT_NAME}-sagemaker-role-{ENV}",
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
        {"Key": "OutpostId", "Value": OUTPOST_ID},
    ],
)
print(f"Model created: {model_name}")

# Create endpoint configuration
sagemaker.create_endpoint_config(
    EndpointConfigName=endpoint_config_name,
    ProductionVariants=[
        {
            "VariantName": "primary",
            "ModelName": model_name,
            "InstanceType": INFERENCE_INSTANCE_TYPE,  # Must be Outposts-supported
            "InitialInstanceCount": 1,
        },
    ],
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)
print(f"Endpoint config created: {endpoint_config_name}")

# Create endpoint (deployed to Outposts via subnet placement)
sagemaker.create_endpoint(
    EndpointName=endpoint_name,
    EndpointConfigName=endpoint_config_name,
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)

# Poll until InService
while True:
    status = sagemaker.describe_endpoint(EndpointName=endpoint_name)
    endpoint_status = status["EndpointStatus"]
    if endpoint_status == "InService":
        print(f"Endpoint InService: {endpoint_name}")
        break
    elif endpoint_status == "Failed":
        raise RuntimeError(f"Endpoint failed: {status.get('FailureReason', 'Unknown')}")
    print(f"  Status: {endpoint_status}...")
    time.sleep(30)
```

**Invoke local endpoint:**
```python
import json

sagemaker_runtime = boto3.client("sagemaker-runtime", region_name=AWS_REGION)

def invoke_local_endpoint(payload, endpoint_name=endpoint_name):
    """Invoke SageMaker endpoint running on Outposts."""
    start = time.time()
    response = sagemaker_runtime.invoke_endpoint(
        EndpointName=endpoint_name,
        ContentType="application/json",
        Body=json.dumps(payload),
    )
    latency_ms = (time.time() - start) * 1000
    result = json.loads(response["Body"].read().decode("utf-8"))

    # Publish latency metric
    cloudwatch = boto3.client("cloudwatch", region_name=AWS_REGION)
    cloudwatch.put_metric_data(
        Namespace=CLOUDWATCH_NAMESPACE,
        MetricData=[
            {
                "MetricName": "LocalInferenceLatency",
                "Value": latency_ms,
                "Unit": "Milliseconds",
                "Dimensions": [
                    {"Name": "EndpointName", "Value": endpoint_name},
                    {"Name": "Environment", "Value": ENV},
                ],
            },
            {
                "MetricName": "LocalInferenceCount",
                "Value": 1,
                "Unit": "Count",
                "Dimensions": [
                    {"Name": "EndpointName", "Value": endpoint_name},
                ],
            },
        ],
    )
    return result, latency_ms
```

**DataSync task configuration:**
```python
datasync = boto3.client("datasync", region_name=AWS_REGION)

# Create regional S3 location
regional_location = datasync.create_location_s3(
    S3BucketArn=f"arn:aws:s3:::{REGIONAL_S3_BUCKET}",
    S3Config={
        "BucketAccessRoleArn": f"arn:aws:iam::{AWS_ACCOUNT_ID}:role/{PROJECT_NAME}-datasync-role-{ENV}",
    },
    Subdirectory=f"/{PROJECT_NAME}/{ENV}/",
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)
regional_location_arn = regional_location["LocationArn"]
print(f"Regional S3 location: {regional_location_arn}")

# Create S3 on Outposts location
outpost_location = datasync.create_location_s3(
    S3BucketArn=bucket_arn,  # S3 on Outposts bucket ARN
    S3Config={
        "BucketAccessRoleArn": f"arn:aws:iam::{AWS_ACCOUNT_ID}:role/{PROJECT_NAME}-datasync-role-{ENV}",
    },
    S3StorageClass="OUTPOSTS",
    Subdirectory=f"/{PROJECT_NAME}/{ENV}/",
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)
outpost_location_arn = outpost_location["LocationArn"]
print(f"Outposts S3 location: {outpost_location_arn}")

# Create transfer task (region → outpost for model sync)
bandwidth_bytes = DATASYNC_BANDWIDTH_LIMIT_MBPS * 1024 * 1024 if DATASYNC_BANDWIDTH_LIMIT_MBPS else -1

model_sync_task = datasync.create_task(
    SourceLocationArn=regional_location_arn,
    DestinationLocationArn=outpost_location_arn,
    Name=f"{PROJECT_NAME}-model-sync-to-outpost-{ENV}",
    Schedule={
        "ScheduleExpression": DATASYNC_SCHEDULE,
    },
    Options={
        "VerifyMode": "POINT_IN_TIME_CONSISTENT",
        "OverwriteMode": "ALWAYS",
        "PreserveDeletedFiles": "PRESERVE",
        "TransferMode": "CHANGED",
        "BytesPerSecond": bandwidth_bytes,
        "LogLevel": "TRANSFER",
    },
    Includes=[
        {
            "FilterType": "SIMPLE_PATTERN",
            "Value": "/models/*",
        },
    ],
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
        {"Key": "Direction", "Value": "region_to_outpost"},
    ],
)
model_sync_task_arn = model_sync_task["TaskArn"]
print(f"Model sync task created: {model_sync_task_arn}")

# Create transfer task (outpost → region for training data)
data_sync_task = datasync.create_task(
    SourceLocationArn=outpost_location_arn,
    DestinationLocationArn=regional_location_arn,
    Name=f"{PROJECT_NAME}-data-sync-to-region-{ENV}",
    Schedule={
        "ScheduleExpression": DATASYNC_SCHEDULE,
    },
    Options={
        "VerifyMode": "POINT_IN_TIME_CONSISTENT",
        "OverwriteMode": "ALWAYS",
        "PreserveDeletedFiles": "PRESERVE",
        "TransferMode": "CHANGED",
        "BytesPerSecond": bandwidth_bytes,
        "LogLevel": "TRANSFER",
    },
    Includes=[
        {
            "FilterType": "SIMPLE_PATTERN",
            "Value": "/training-data/*",
        },
    ],
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
        {"Key": "Direction", "Value": "outpost_to_region"},
    ],
)
data_sync_task_arn = data_sync_task["TaskArn"]
print(f"Data sync task created: {data_sync_task_arn}")


def trigger_sync(task_arn):
    """Manually trigger a DataSync task execution."""
    execution = datasync.start_task_execution(TaskArn=task_arn)
    execution_arn = execution["TaskExecutionArn"]
    print(f"Task execution started: {execution_arn}")
    return execution_arn


def check_sync_status(execution_arn):
    """Check DataSync task execution status."""
    result = datasync.describe_task_execution(TaskExecutionArn=execution_arn)
    status = result["Status"]
    print(f"Execution {execution_arn}: {status}")
    if status == "SUCCESS":
        print(f"  Files transferred: {result.get('FilesTransferred', 0)}")
        print(f"  Bytes transferred: {result.get('BytesTransferred', 0)}")
    elif status == "ERROR":
        print(f"  Error: {result.get('Result', {}).get('ErrorCode', 'Unknown')}")
    return status
```

**ECS preprocessing task on Outposts:**
```python
ecs = boto3.client("ecs", region_name=AWS_REGION)

task_def_name = f"{PROJECT_NAME}-preprocess-{ENV}"

# Register task definition
ecs.register_task_definition(
    family=task_def_name,
    networkMode="awsvpc",
    requiresCompatibilities=["EC2"],  # Fargate not available on Outposts
    cpu="2048",
    memory="4096",
    executionRoleArn=f"arn:aws:iam::{AWS_ACCOUNT_ID}:role/{PROJECT_NAME}-ecs-execution-role-{ENV}",
    taskRoleArn=f"arn:aws:iam::{AWS_ACCOUNT_ID}:role/{PROJECT_NAME}-ecs-task-role-{ENV}",
    containerDefinitions=[
        {
            "name": "preprocessor",
            "image": PREPROCESSING_IMAGE_URI,
            "cpu": 2048,
            "memory": 4096,
            "essential": True,
            "environment": [
                {"name": "S3_ACCESS_POINT_ARN", "value": access_point_arn},
                {"name": "INPUT_PREFIX", "value": "raw-data/"},
                {"name": "OUTPUT_PREFIX", "value": "training-data/"},
                {"name": "PROJECT_NAME", "value": PROJECT_NAME},
                {"name": "ENV", "value": ENV},
            ],
            "logConfiguration": {
                "logDriver": "awslogs",
                "options": {
                    "awslogs-group": f"/{PROJECT_NAME}/ecs-preprocess/{ENV}",
                    "awslogs-region": AWS_REGION,
                    "awslogs-stream-prefix": "preprocess",
                },
            },
        },
    ],
    tags=[
        {"key": "Project", "value": PROJECT_NAME},
        {"key": "Environment", "value": ENV},
    ],
)
print(f"Task definition registered: {task_def_name}")

# Run task on Outposts
run_response = ecs.run_task(
    cluster=ECS_CLUSTER_NAME,
    taskDefinition=task_def_name,
    count=1,
    networkConfiguration={
        "awsvpcConfiguration": {
            "subnets": OUTPOST_SUBNET_IDS,
            "securityGroups": SECURITY_GROUP_IDS,
            "assignPublicIp": "DISABLED",
        },
    },
    placementConstraints=[
        {
            "type": "memberOf",
            "expression": f"attribute:ecs.outpost-arn == {OUTPOST_ARN}",
        },
    ],
    tags=[
        {"key": "Project", "value": PROJECT_NAME},
        {"key": "Environment", "value": ENV},
    ],
)

task_arn = run_response["tasks"][0]["taskArn"]
print(f"ECS task started: {task_arn}")

# Poll until complete
while True:
    desc = ecs.describe_tasks(cluster=ECS_CLUSTER_NAME, tasks=[task_arn])
    task_status = desc["tasks"][0]["lastStatus"]
    if task_status == "STOPPED":
        exit_code = desc["tasks"][0]["containers"][0].get("exitCode", -1)
        if exit_code == 0:
            print("Preprocessing completed successfully")
        else:
            reason = desc["tasks"][0].get("stoppedReason", "Unknown")
            raise RuntimeError(f"Preprocessing failed (exit {exit_code}): {reason}")
        break
    print(f"  Task status: {task_status}...")
    time.sleep(15)
```

**Inference continuity pattern:**
```python
import socket
import os
import json
import time
from pathlib import Path
from enum import Enum

class ConnectivityState(Enum):
    HEALTHY = "HEALTHY"
    DEGRADED = "DEGRADED"
    DISCONNECTED = "DISCONNECTED"


QUEUE_DIR = Path(f"/tmp/{PROJECT_NAME}/continuity_queue")
MAX_QUEUE_SIZE = 50000


def check_wan_connectivity():
    """Test WAN connectivity to regional AWS endpoints."""
    endpoints = [
        (f"sagemaker.{AWS_REGION}.amazonaws.com", 443),
        (f"monitoring.{AWS_REGION}.amazonaws.com", 443),
    ]
    reachable = 0
    total_latency = 0

    for host, port in endpoints:
        try:
            start = time.time()
            sock = socket.create_connection((host, port), timeout=5)
            latency = (time.time() - start) * 1000
            sock.close()
            reachable += 1
            total_latency += latency
        except (socket.timeout, socket.error, OSError):
            pass

    if reachable == 0:
        return ConnectivityState.DISCONNECTED, 0
    avg_latency = total_latency / reachable
    if reachable < len(endpoints) or avg_latency > 500:
        return ConnectivityState.DEGRADED, avg_latency
    return ConnectivityState.HEALTHY, avg_latency


def queue_result(result_dict):
    """Queue inference result locally during connectivity loss."""
    QUEUE_DIR.mkdir(parents=True, exist_ok=True)
    queue_files = sorted(QUEUE_DIR.glob("*.json"))
    if len(queue_files) >= MAX_QUEUE_SIZE:
        # Evict oldest
        queue_files[0].unlink()
    filename = f"{int(time.time() * 1000)}_{os.getpid()}.json"
    (QUEUE_DIR / filename).write_text(json.dumps(result_dict))


def drain_queue(s3_client, bucket, prefix):
    """Drain queued results to S3 when connectivity is restored."""
    queue_files = sorted(QUEUE_DIR.glob("*.json"))
    drained = 0
    for f in queue_files:
        try:
            data = f.read_text()
            key = f"{prefix}{f.stem}.json"
            s3_client.put_object(Bucket=bucket, Key=key, Body=data)
            f.unlink()
            drained += 1
        except Exception as e:
            print(f"Failed to drain {f.name}: {e}")
            break  # Stop on first failure — connectivity may be lost again
    print(f"Drained {drained}/{len(queue_files)} queued results")
    return drained


def publish_connectivity_metric(state, latency_ms):
    """Publish WAN connectivity status to CloudWatch."""
    cloudwatch = boto3.client("cloudwatch", region_name=AWS_REGION)
    state_value = {"HEALTHY": 1.0, "DEGRADED": 0.5, "DISCONNECTED": 0.0}
    try:
        cloudwatch.put_metric_data(
            Namespace=CLOUDWATCH_NAMESPACE,
            MetricData=[
                {
                    "MetricName": "WANConnectivityStatus",
                    "Value": state_value[state.value],
                    "Unit": "None",
                    "Dimensions": [
                        {"Name": "OutpostId", "Value": OUTPOST_ID},
                        {"Name": "Environment", "Value": ENV},
                    ],
                },
                {
                    "MetricName": "WANLatency",
                    "Value": latency_ms,
                    "Unit": "Milliseconds",
                    "Dimensions": [
                        {"Name": "OutpostId", "Value": OUTPOST_ID},
                    ],
                },
            ],
        )
    except Exception:
        pass  # Cannot publish if disconnected — expected
```

**CloudWatch agent configuration for Outposts:**
```json
{
  "agent": {
    "metrics_collection_interval": 60,
    "run_as_user": "cwagent"
  },
  "metrics": {
    "namespace": "{PROJECT_NAME}/OutpostsML/{ENV}",
    "append_dimensions": {
      "InstanceId": "${aws:InstanceId}",
      "AutoScalingGroupName": "${aws:AutoScalingGroupName}"
    },
    "metrics_collected": {
      "cpu": {
        "measurement": ["cpu_usage_idle", "cpu_usage_user", "cpu_usage_system"],
        "totalcpu": true
      },
      "mem": {
        "measurement": ["mem_used_percent", "mem_available"]
      },
      "disk": {
        "measurement": ["disk_used_percent", "disk_inodes_free"],
        "resources": ["/", "/opt/ml"]
      },
      "net": {
        "measurement": ["bytes_sent", "bytes_recv", "packets_sent", "packets_recv"],
        "resources": ["eth0"]
      },
      "statsd": {
        "service_address": ":8125",
        "metrics_collection_interval": 15,
        "metrics_aggregation_interval": 60
      }
    },
    "force_flush_interval": 15
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/{PROJECT_NAME}/inference.log",
            "log_group_name": "/{PROJECT_NAME}/outposts/{ENV}",
            "log_stream_name": "{instance_id}/inference",
            "retention_in_days": 30
          },
          {
            "file_path": "/var/log/{PROJECT_NAME}/connectivity.log",
            "log_group_name": "/{PROJECT_NAME}/outposts/{ENV}",
            "log_stream_name": "{instance_id}/connectivity",
            "retention_in_days": 30
          }
        ]
      }
    },
    "force_flush_interval": 15
  }
}
```

**CloudWatch dashboard for hybrid ML:**
```python
import json

cloudwatch = boto3.client("cloudwatch", region_name=AWS_REGION)

dashboard_name = f"{PROJECT_NAME}-outposts-ml-{ENV}"
namespace = CLOUDWATCH_NAMESPACE

dashboard_body = {
    "widgets": [
        {
            "type": "metric",
            "x": 0, "y": 0, "width": 12, "height": 6,
            "properties": {
                "title": "Local Inference Latency (p50/p95/p99)",
                "metrics": [
                    [namespace, "LocalInferenceLatency", "EndpointName",
                     f"{PROJECT_NAME}-outpost-endpoint-{ENV}", {"stat": "p50", "label": "p50"}],
                    ["...", {"stat": "p95", "label": "p95"}],
                    ["...", {"stat": "p99", "label": "p99"}],
                ],
                "period": 300,
                "view": "timeSeries",
            },
        },
        {
            "type": "metric",
            "x": 12, "y": 0, "width": 12, "height": 6,
            "properties": {
                "title": "Local Inference Throughput",
                "metrics": [
                    [namespace, "LocalInferenceCount", {"stat": "Sum"}],
                ],
                "period": 60,
                "view": "timeSeries",
            },
        },
        {
            "type": "metric",
            "x": 0, "y": 6, "width": 12, "height": 6,
            "properties": {
                "title": "WAN Connectivity Status",
                "metrics": [
                    [namespace, "WANConnectivityStatus", "OutpostId", OUTPOST_ID,
                     {"stat": "Minimum", "label": "Connectivity (1=OK, 0=Down)"}],
                ],
                "period": 60,
                "view": "timeSeries",
                "yAxis": {"left": {"min": 0, "max": 1}},
            },
        },
        {
            "type": "metric",
            "x": 12, "y": 6, "width": 12, "height": 6,
            "properties": {
                "title": "WAN Latency",
                "metrics": [
                    [namespace, "WANLatency", "OutpostId", OUTPOST_ID,
                     {"stat": "Average", "label": "Avg Latency (ms)"}],
                    ["...", {"stat": "p99", "label": "p99 Latency (ms)"}],
                ],
                "period": 60,
                "view": "timeSeries",
            },
        },
        {
            "type": "metric",
            "x": 0, "y": 12, "width": 12, "height": 6,
            "properties": {
                "title": "Outposts Instance CPU Utilization",
                "metrics": [
                    [namespace, "cpu_usage_user", {"stat": "Average"}],
                    [namespace, "cpu_usage_system", {"stat": "Average"}],
                ],
                "period": 300,
                "view": "timeSeries",
            },
        },
        {
            "type": "metric",
            "x": 12, "y": 12, "width": 12, "height": 6,
            "properties": {
                "title": "Outposts Instance Memory Usage",
                "metrics": [
                    [namespace, "mem_used_percent", {"stat": "Average"}],
                ],
                "period": 300,
                "view": "timeSeries",
                "yAxis": {"left": {"min": 0, "max": 100}},
            },
        },
    ],
}

cloudwatch.put_dashboard(
    DashboardName=dashboard_name,
    DashboardBody=json.dumps(dashboard_body),
)
print(f"Dashboard created: {dashboard_name}")
```

---

## Requirements & Constraints

**Outposts Prerequisites:** An AWS Outposts rack must be installed, connected, and operational at the customer site. The Outpost must have sufficient compute capacity for the requested instance types. S3 on Outposts requires available S3 capacity on the rack. Verify Outpost capacity with `outposts.get_outpost_instance_types()` before deploying.

**Instance Types:** Only Outposts-supported instance types can be used for SageMaker endpoints and ECS tasks. Common ML-capable types on Outposts include `ml.c5.xlarge`, `ml.c5.2xlarge`, `ml.m5.xlarge`, `ml.m5.2xlarge`. GPU instances (e.g., `ml.g4dn`) are available only on Outposts racks with GPU capacity. Verify availability before deployment.

**S3 on Outposts:** S3 on Outposts has a maximum object size of 5 TB (same as regional S3) but total bucket capacity is limited by the Outposts S3 storage allocation. S3 on Outposts does not support S3 Transfer Acceleration, S3 Select, or cross-region replication natively — use DataSync for data movement. Access is restricted to VPC access points.

**Networking:** Outposts subnets must have a local gateway route for on-premises traffic and a service link connection to the parent AWS Region for API calls. SageMaker endpoints on Outposts require the service link to be operational for initial deployment but can serve inference locally once InService. DataSync transfers use the service link bandwidth.

**Data Residency:** When DATA_RESIDENCY_REGION is set, ensure that training data stored on S3 on Outposts is not replicated to regions outside the specified residency boundary. Use S3 on Outposts lifecycle rules and DataSync task filters to enforce data locality. Tag all resources with `DataResidency` for audit compliance.

**Connectivity Resilience:** The local inference continuity pattern assumes the SageMaker endpoint is already deployed and InService on the Outpost. During WAN disconnection, the endpoint continues serving inference but new deployments, updates, and CloudWatch metric publishing are unavailable. Queue depth should be monitored to prevent local disk exhaustion.

**DataSync:** DataSync tasks between S3 on Outposts and regional S3 require a DataSync agent deployed on the Outpost (or agentless if using the service link). Bandwidth throttling is recommended for production to avoid saturating the service link. Transfer verification adds overhead but ensures data integrity.

**Security:** Use VPC endpoints for SageMaker and S3 API calls where possible. Encrypt S3 on Outposts data using SSE-S3 (SSE-KMS is supported with regional KMS keys via the service link). IAM roles for ECS tasks and SageMaker must include permissions for both regional and Outposts S3 operations. Reference `devops/04` for IAM role patterns and `devops/08` for KMS key configuration.

**Cost:** Outposts capacity is billed as a fixed monthly rack fee — there are no per-instance charges for compute on the Outpost. However, DataSync transfer costs apply for data movement between Outposts and the Region. S3 on Outposts storage is included in the rack fee up to the provisioned capacity. Regional SageMaker training jobs are billed at standard rates. Monitor DataSync transfer volumes to control costs.

**Naming:** All resources follow `{PROJECT_NAME}-{component}-{ENV}` convention:
- S3 on Outposts bucket: `{S3_OUTPOSTS_BUCKET}`
- Access point: `{S3_OUTPOSTS_ACCESS_POINT}`
- SageMaker model: `{PROJECT_NAME}-outpost-model-{ENV}`
- SageMaker endpoint: `{PROJECT_NAME}-outpost-endpoint-{ENV}`
- DataSync tasks: `{PROJECT_NAME}-sync-{direction}-{ENV}`
- ECS task definition: `{PROJECT_NAME}-preprocess-{ENV}`
- Dashboard: `{PROJECT_NAME}-outposts-ml-{ENV}`

---

## Integration Points

- **Upstream**: `devops/04` → IAM roles for SageMaker execution, ECS task roles, DataSync service role, and S3 on Outposts access policies
- **Upstream**: `devops/02` → VPC and Outposts subnet configuration, local gateway routing, security groups, and service link connectivity
- **Upstream**: `mlops/03` → Model artifact in S3 (MODEL_S3_URI) from regional inference deployment pipeline; endpoint deployment patterns reused for Outposts placement
- **Upstream**: `devops/08` → KMS keys for encrypting S3 on Outposts data and SageMaker model artifacts
- **Downstream**: `devops/03` → CloudWatch dashboards consume metrics published by the CloudWatch agent and inference continuity monitor
- **Downstream**: `devops/11` → Custom CloudWatch metrics (LocalInferenceLatency, WANConnectivityStatus) feed into model quality monitoring
- **Downstream**: `finops/01` → Cost tracking for DataSync transfer volumes and regional SageMaker training job costs
- **Downstream**: `edge/01` → SageMaker Edge Manager can complement Outposts inference for lighter-weight edge devices connected to the same on-premises network
