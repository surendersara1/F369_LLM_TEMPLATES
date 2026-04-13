<!-- Template Version: 1.0 | boto3: 1.35+ -->

# Template Edge 01 — SageMaker Edge Deployment (Neo + IoT)

> **⚠️ DEPRECATION NOTICE:** SageMaker Edge Manager reached end of life on April 26, 2024. The Edge Manager APIs (`create_device_fleet`, `register_devices`, `create_edge_packaging_job`, `describe_device`) are no longer functional. AWS recommends migrating to **AWS IoT Greengrass V2** (see `edge/02`) for edge deployment and monitoring, or **ONNX Runtime** for cross-platform edge inference. The **SageMaker Neo compilation** APIs (`create_compilation_job`) remain fully supported and are still the recommended approach for model optimization.
>
> This template retains Neo compilation as its core value and uses IoT Jobs for OTA delivery. For device fleet management and data capture, use `edge/02` (IoT Greengrass V2) instead of the deprecated Edge Manager fleet APIs.

## Purpose
Generate a production-ready edge ML deployment pipeline: SageMaker Neo model compilation for target edge platforms, OTA model update workflows via IoT Jobs, Lambda-based edge inference metric aggregation, and CloudWatch fleet monitoring dashboards. **Note:** Device fleet management and edge packaging use IoT Greengrass V2 patterns (see `edge/02`) as SageMaker Edge Manager is no longer available.

---

## Role Definition

You are an expert AWS edge ML engineer specializing in SageMaker Neo and IoT-based edge deployment with expertise in:
- SageMaker Neo: model compilation for ARM64, x86_64, Android, and iOS target platforms across PyTorch, TensorFlow, and ONNX frameworks
- IoT Core: role aliases for device authentication, IoT Jobs for OTA model deployment, thing registry management
- IoT Greengrass V2: component-based edge deployment (preferred over deprecated Edge Manager — see `edge/02`)
- Edge inference monitoring: S3 event-driven Lambda aggregation, CloudWatch custom metrics for fleet health
- IAM policies for SageMaker execution roles, IoT device roles, and Lambda execution roles
- Model optimization: quantization, pruning, and compilation trade-offs for resource-constrained edge devices
- **Note:** SageMaker Edge Manager (device fleets, edge packaging) reached EOL April 2024. This template focuses on Neo compilation + IoT Jobs OTA. For full device fleet management, use `edge/02` (IoT Greengrass V2).

Generate complete, production-deployable code.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

TARGET_PLATFORM:        [REQUIRED - target device platform for Neo compilation]
                        Options:
                        - linux_arm64 (Raspberry Pi, NVIDIA Jetson, AWS Graviton)
                        - linux_x86_64 (Intel/AMD edge servers)
                        - android (Android mobile devices)
                        - ios (Apple iOS devices)

FRAMEWORK:              [REQUIRED - ML framework of the source model]
                        Options:
                        - pytorch (PyTorch .pt or TorchScript)
                        - tensorflow (TensorFlow SavedModel or frozen graph)
                        - onnx (ONNX .onnx format)

MODEL_S3_URI:           [REQUIRED - S3 URI of the trained model artifact (tar.gz)]
                        Example: s3://my-bucket/models/my-model.tar.gz

MODEL_INPUT_SHAPE:      [REQUIRED - JSON dict of input tensor name to shape]
                        Example: {"input": [1, 3, 224, 224]}

DEVICE_FLEET_NAME:      [REQUIRED - name for the edge device fleet]
                        Example: factory-floor-fleet

S3_OUTPUT_PATH:         [REQUIRED - S3 path for compiled models and edge data capture]
                        Example: s3://my-bucket/edge-output/

IOT_ROLE_ALIAS:         [REQUIRED - IoT role alias for device authentication]
                        Example: SageMakerEdgeDeviceRole

DEVICE_NAMES:           [OPTIONAL: [] - list of device names to register]
                        Example: ["device-001", "device-002", "device-003"]

OTA_ENABLED:            [OPTIONAL: true - enable OTA model update via IoT Jobs]

OTA_THING_GROUP:        [OPTIONAL: "" - IoT thing group for OTA targeting]
                        Example: edge-ml-devices

COMPILATION_INSTANCE:   [OPTIONAL: ml.c5.4xlarge - instance type for Neo compilation]

DATA_CAPTURE_PERCENTAGE:[OPTIONAL: 100 - percentage of edge inferences to capture to S3]
```

---

## Task

Generate complete SageMaker Edge Manager deployment pipeline:

```
{PROJECT_NAME}-edge-deployment/
├── compilation/
│   ├── compile_model.py               # SageMaker Neo compilation job
│   └── config.py                      # Central configuration
├── fleet/
│   ├── create_fleet.py                # Device fleet creation with S3 data capture
│   ├── register_devices.py            # Register edge devices to fleet
│   └── manage_devices.py              # Describe, list, deregister devices
├── packaging/
│   └── package_model.py               # Edge packaging job for deployment
├── ota/
│   ├── create_iot_job.py              # IoT Jobs OTA model update workflow
│   └── ota_status.py                  # Track OTA deployment status per device
├── monitoring/
│   ├── metric_aggregator_lambda.py    # Lambda: aggregate S3 edge inference data
│   ├── deploy_lambda.py               # Deploy the aggregator Lambda + S3 trigger
│   └── fleet_dashboard.py             # CloudWatch fleet monitoring dashboard
├── cleanup/
│   └── delete_resources.py            # Delete fleet, devices, compilation artifacts
├── run_setup.py                       # CLI orchestrator
└── requirements.txt
```

**config.py**: Central configuration dataclass with all parameters. Load from environment variables or CLI args. Validate required fields. Construct resource names using `{PROJECT_NAME}-{component}-{ENV}` convention. Map TARGET_PLATFORM to Neo `TargetPlatform` dict (`Os`, `Arch`, `Accelerator`). Map FRAMEWORK to Neo `Framework` enum.

**compile_model.py**: Create SageMaker Neo compilation job using `sagemaker.create_compilation_job()`:
- Set `InputConfig` with `S3Uri` (MODEL_S3_URI), `DataInputConfig` (MODEL_INPUT_SHAPE JSON), `Framework` (FRAMEWORK)
- Set `OutputConfig` with `S3OutputLocation` (S3_OUTPUT_PATH/compiled/), `TargetPlatform` dict built from TARGET_PLATFORM (e.g., `{"Os": "LINUX", "Arch": "ARM64"}`)
- Set `RoleArn` for SageMaker execution role with S3 read/write access
- Set `StoppingCondition` with `MaxRuntimeInSeconds` (default 900)
- Poll with `describe_compilation_job()` until status is `COMPLETED` or `FAILED`
- On success, log the compiled model S3 URI from `OutputConfig.S3OutputLocation`

**create_fleet.py**: Create edge device fleet using `sagemaker.create_device_fleet()`:
- Set `DeviceFleetName` to `{PROJECT_NAME}-{DEVICE_FLEET_NAME}-{ENV}`
- Set `OutputConfig` with `S3OutputPath` for edge inference data capture
- Set `RoleArn` for IoT role that allows S3 writes and SageMaker Edge Manager API access
- Optionally set `Description` and `Tags`
- The IoT role alias (IOT_ROLE_ALIAS) must be pre-created and mapped to an IAM role that grants `sagemaker:SendHeartbeat`, `sagemaker:GetDeviceRegistration`, `s3:PutObject`

**register_devices.py**: Register edge devices to the fleet using `sagemaker.register_devices()`:
- Accept DEVICE_NAMES list, construct `Device` objects with `DeviceName` and optional `Description`, `IotThingName`
- Call `register_devices()` with `DeviceFleetName` and `Devices` list
- Verify registration with `describe_device()` for each device

**manage_devices.py**: Fleet management utilities:
- `list_devices()`: Call `sagemaker.list_devices()` with `DeviceFleetName` to list all registered devices
- `describe_device()`: Call `sagemaker.describe_device()` to get device status, latest heartbeat, models loaded, registration time
- `deregister_devices()`: Call `sagemaker.deregister_devices()` to remove devices from fleet

**package_model.py**: Create edge packaging job using `sagemaker.create_edge_packaging_job()`:
- Set `CompilationJobName` referencing the completed Neo compilation job
- Set `ModelName` and `ModelVersion` for versioned edge deployment
- Set `OutputConfig` with `S3OutputLocation` for the packaged model artifact
- Set `ResourceKey` for model encryption on device (optional)
- Poll with `describe_edge_packaging_job()` until status is `COMPLETED`
- The packaged artifact includes the compiled model + Edge Manager agent runtime

**create_iot_job.py** (when OTA_ENABLED): Create IoT Job for OTA model update:
- Build job document JSON with: packaged model S3 URI, model name, model version, target device fleet, download and install instructions
- Call `iot.create_job()` with `jobId`, `targets` (thing group ARN from OTA_THING_GROUP), `document` (job document JSON), `targetSelection` ("SNAPSHOT" for one-time or "CONTINUOUS" for new devices)
- Set `jobExecutionsRolloutConfig` with `maximumPerMinute` for controlled rollout
- Set `abortConfig` with failure threshold to stop rollout on excessive failures
- Set `timeoutConfig` with `inProgressTimeoutInMinutes` for stalled devices

**ota_status.py**: Track OTA deployment status:
- Call `iot.describe_job()` to get overall job status and summary counts (QUEUED, IN_PROGRESS, SUCCEEDED, FAILED)
- Call `iot.list_job_executions_for_job()` to get per-device execution status
- Output a summary table: device name, status, last updated, failure reason (if any)

**metric_aggregator_lambda.py**: Lambda function triggered by S3 events when edge devices upload inference data:
- Parse S3 event to get bucket and key of uploaded inference result file
- Read the inference data (JSON or CSV) from S3
- Extract metrics: inference latency, prediction confidence, model version, device name, timestamp
- Publish custom CloudWatch metrics using `cloudwatch.put_metric_data()`:
  - `EdgeInferenceLatency` (per device, per model version)
  - `EdgeInferenceCount` (per device)
  - `EdgeModelVersion` (per device — track version distribution)
  - `EdgeDeviceHeartbeat` (last seen timestamp)
- Namespace: `{PROJECT_NAME}/EdgeML/{ENV}`

**deploy_lambda.py**: Deploy the metric aggregator Lambda:
- Create Lambda function with S3 read and CloudWatch write permissions
- Add S3 event notification on the edge data capture bucket with prefix filter for inference results
- Configure Lambda with appropriate timeout (60s) and memory (256MB)

**fleet_dashboard.py**: Create CloudWatch dashboard using `cloudwatch.put_dashboard()`:
- Widget 1: Active devices count (devices with heartbeat in last 15 minutes)
- Widget 2: Model version distribution across fleet (pie chart or bar)
- Widget 3: Average inference latency by device (line graph, p50/p95/p99)
- Widget 4: Inference throughput per device (invocations per minute)
- Widget 5: Device heartbeat status (table with last-seen timestamps)
- Widget 6: OTA deployment progress (if OTA_ENABLED)
- Dashboard name: `{PROJECT_NAME}-edge-fleet-{ENV}`

**delete_resources.py**: Clean up all resources:
- Deregister all devices from fleet
- Delete device fleet
- Cancel any active IoT Jobs
- Delete Lambda function and S3 event notification
- Delete CloudWatch dashboard
- Optionally delete compiled/packaged model artifacts from S3

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Code Scaffolding Hints

**Platform mapping helper:**
```python
PLATFORM_MAP = {
    "linux_arm64": {"Os": "LINUX", "Arch": "ARM64"},
    "linux_x86_64": {"Os": "LINUX", "Arch": "X86_64"},
    "android": {"Os": "ANDROID", "Arch": "ARM64"},
    "ios": {"Os": "LINUX", "Arch": "ARM64"},  # iOS uses CoreML converter post-Neo
}

FRAMEWORK_MAP = {
    "pytorch": "PYTORCH",
    "tensorflow": "TFLITE",  # or TENSORFLOW depending on input format
    "onnx": "ONNX",
}
```

**Neo compilation job:**
```python
import boto3
import time
import json

sagemaker = boto3.client("sagemaker", region_name=AWS_REGION)

compilation_job_name = f"{PROJECT_NAME}-neo-{TARGET_PLATFORM}-{ENV}-{int(time.time())}"

target_platform = PLATFORM_MAP[TARGET_PLATFORM]

response = sagemaker.create_compilation_job(
    CompilationJobName=compilation_job_name,
    RoleArn=sagemaker_role_arn,
    InputConfig={
        "S3Uri": MODEL_S3_URI,
        "DataInputConfig": json.dumps(MODEL_INPUT_SHAPE),
        "Framework": FRAMEWORK_MAP[FRAMEWORK],
    },
    OutputConfig={
        "S3OutputLocation": f"{S3_OUTPUT_PATH}compiled/",
        "TargetPlatform": target_platform,
    },
    StoppingCondition={
        "MaxRuntimeInSeconds": 900,
    },
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)

# Poll until complete
while True:
    status = sagemaker.describe_compilation_job(
        CompilationJobName=compilation_job_name
    )
    job_status = status["CompilationJobStatus"]
    if job_status == "COMPLETED":
        compiled_model_uri = status["ModelArtifacts"]["S3ModelArtifacts"]
        print(f"Compilation succeeded: {compiled_model_uri}")
        break
    elif job_status == "FAILED":
        raise RuntimeError(
            f"Compilation failed: {status.get('FailureReason', 'Unknown')}"
        )
    time.sleep(30)
```

**Create device fleet:**
```python
fleet_name = f"{PROJECT_NAME}-{DEVICE_FLEET_NAME}-{ENV}"

sagemaker.create_device_fleet(
    DeviceFleetName=fleet_name,
    OutputConfig={
        "S3OutputLocation": f"{S3_OUTPUT_PATH}data-capture/",
    },
    RoleArn=edge_device_role_arn,  # IAM role assumed via IoT role alias
    Description=f"Edge device fleet for {PROJECT_NAME} {ENV}",
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)
print(f"Device fleet created: {fleet_name}")
```

**Register devices:**
```python
devices = [
    {
        "DeviceName": device_name,
        "Description": f"Edge device {device_name}",
        "IotThingName": device_name,  # Maps to IoT thing for OTA
    }
    for device_name in DEVICE_NAMES
]

sagemaker.register_devices(
    DeviceFleetName=fleet_name,
    Devices=devices,
)

# Verify registration
for device_name in DEVICE_NAMES:
    device_info = sagemaker.describe_device(
        DeviceName=device_name,
        DeviceFleetName=fleet_name,
    )
    print(
        f"Device {device_name}: status={device_info.get('DeviceFleetName')}, "
        f"registered={device_info.get('RegistrationTime')}, "
        f"latest_heartbeat={device_info.get('LatestHeartbeat')}"
    )
```

**Describe device (fleet management):**
```python
def get_device_status(device_name, fleet_name):
    """Get detailed device status including loaded models."""
    device = sagemaker.describe_device(
        DeviceName=device_name,
        DeviceFleetName=fleet_name,
    )
    return {
        "device_name": device["DeviceName"],
        "fleet": device["DeviceFleetName"],
        "registration_time": device.get("RegistrationTime"),
        "latest_heartbeat": device.get("LatestHeartbeat"),
        "models": [
            {
                "name": m["ModelName"],
                "version": m["ModelVersion"],
                "latest_sample_time": m.get("LatestSampleTime"),
                "latest_inference": m.get("LatestInference"),
            }
            for m in device.get("Models", [])
        ],
    }


def list_fleet_devices(fleet_name):
    """List all devices in a fleet."""
    paginator = sagemaker.get_paginator("list_devices")
    devices = []
    for page in paginator.paginate(DeviceFleetName=fleet_name):
        devices.extend(page["DeviceSummaries"])
    return devices
```

**Edge packaging job:**
```python
packaging_job_name = f"{PROJECT_NAME}-edge-pkg-{ENV}-{int(time.time())}"
model_name = f"{PROJECT_NAME}-edge-model"
model_version = "1.0"

sagemaker.create_edge_packaging_job(
    EdgePackagingJobName=packaging_job_name,
    CompilationJobName=compilation_job_name,
    ModelName=model_name,
    ModelVersion=model_version,
    RoleArn=sagemaker_role_arn,
    OutputConfig={
        "S3OutputLocation": f"{S3_OUTPUT_PATH}packaged/",
    },
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)

# Poll until complete
while True:
    pkg_status = sagemaker.describe_edge_packaging_job(
        EdgePackagingJobName=packaging_job_name
    )
    status = pkg_status["EdgePackagingJobStatus"]
    if status == "COMPLETED":
        packaged_uri = pkg_status["ModelArtifact"]
        print(f"Packaging succeeded: {packaged_uri}")
        break
    elif status == "FAILED":
        raise RuntimeError(
            f"Packaging failed: {pkg_status.get('EdgePackagingJobStatusMessage', 'Unknown')}"
        )
    time.sleep(15)
```

**OTA model update via IoT Jobs:**
```python
import json

iot = boto3.client("iot", region_name=AWS_REGION)

job_id = f"{PROJECT_NAME}-ota-{model_version.replace('.', '-')}-{ENV}-{int(time.time())}"

job_document = json.dumps({
    "operation": "model_update",
    "model_name": model_name,
    "model_version": model_version,
    "model_s3_uri": packaged_uri,
    "fleet_name": fleet_name,
    "instructions": {
        "download": f"aws s3 cp {packaged_uri} /opt/ml/models/{model_name}/",
        "install": f"edge_manager_agent load --model-name {model_name} --model-path /opt/ml/models/{model_name}/",
        "verify": f"edge_manager_agent list-models",
    },
})

thing_group_arn = f"arn:aws:iot:{AWS_REGION}:{AWS_ACCOUNT_ID}:thinggroup/{OTA_THING_GROUP}"

response = iot.create_job(
    jobId=job_id,
    targets=[thing_group_arn],
    document=job_document,
    description=f"OTA model update: {model_name} v{model_version}",
    targetSelection="SNAPSHOT",  # One-time deployment; use "CONTINUOUS" for new devices
    jobExecutionsRolloutConfig={
        "maximumPerMinute": 10,  # Controlled rollout rate
    },
    abortConfig={
        "criteriaList": [
            {
                "failureType": "FAILED",
                "action": "CANCEL",
                "thresholdPercentage": 25.0,  # Abort if >25% fail
                "minNumberOfExecutedThings": 3,
            }
        ]
    },
    timeoutConfig={
        "inProgressTimeoutInMinutes": 30,
    },
    tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)
print(f"OTA job created: {job_id}")
```

**Track OTA deployment status:**
```python
def get_ota_status(job_id):
    """Get OTA job status and per-device breakdown."""
    job = iot.describe_job(jobId=job_id)
    summary = job["job"]["jobProcessDetails"]

    print(f"Job: {job_id}")
    print(f"  Status: {job['job']['status']}")
    print(f"  Queued: {summary.get('numberOfQueuedThings', 0)}")
    print(f"  In Progress: {summary.get('numberOfInProgressThings', 0)}")
    print(f"  Succeeded: {summary.get('numberOfSucceededThings', 0)}")
    print(f"  Failed: {summary.get('numberOfFailedThings', 0)}")

    # Per-device details
    executions = iot.list_job_executions_for_job(jobId=job_id)
    for execution in executions.get("executionSummaries", []):
        print(
            f"  Device {execution['thingArn'].split('/')[-1]}: "
            f"{execution['jobExecutionSummary']['status']}"
        )
```

**Lambda metric aggregator (S3 trigger):**
```python
import json
import boto3
from datetime import datetime

cloudwatch = boto3.client("cloudwatch")
s3 = boto3.client("s3")

NAMESPACE = f"{PROJECT_NAME}/EdgeML/{ENV}"


def lambda_handler(event, context):
    """Aggregate edge inference metrics from S3 data capture uploads."""
    for record in event["Records"]:
        bucket = record["s3"]["bucket"]["name"]
        key = record["s3"]["object"]["key"]

        # Read inference data from S3
        obj = s3.get_object(Bucket=bucket, Key=key)
        data = json.loads(obj["Body"].read().decode("utf-8"))

        device_name = data.get("device_name", "unknown")
        model_version = data.get("model_version", "unknown")
        latency_ms = data.get("inference_latency_ms", 0)
        confidence = data.get("prediction_confidence", 0)

        # Publish custom metrics
        cloudwatch.put_metric_data(
            Namespace=NAMESPACE,
            MetricData=[
                {
                    "MetricName": "EdgeInferenceLatency",
                    "Dimensions": [
                        {"Name": "DeviceName", "Value": device_name},
                        {"Name": "ModelVersion", "Value": str(model_version)},
                    ],
                    "Value": latency_ms,
                    "Unit": "Milliseconds",
                },
                {
                    "MetricName": "EdgeInferenceCount",
                    "Dimensions": [
                        {"Name": "DeviceName", "Value": device_name},
                    ],
                    "Value": 1,
                    "Unit": "Count",
                },
                {
                    "MetricName": "EdgePredictionConfidence",
                    "Dimensions": [
                        {"Name": "DeviceName", "Value": device_name},
                        {"Name": "ModelVersion", "Value": str(model_version)},
                    ],
                    "Value": confidence,
                    "Unit": "None",
                },
                {
                    "MetricName": "EdgeDeviceHeartbeat",
                    "Dimensions": [
                        {"Name": "DeviceName", "Value": device_name},
                    ],
                    "Value": 1,
                    "Unit": "Count",
                },
            ],
        )

    return {"statusCode": 200, "processed": len(event["Records"])}
```

**CloudWatch fleet monitoring dashboard:**
```python
import json

cloudwatch = boto3.client("cloudwatch", region_name=AWS_REGION)

dashboard_name = f"{PROJECT_NAME}-edge-fleet-{ENV}"
namespace = f"{PROJECT_NAME}/EdgeML/{ENV}"

dashboard_body = {
    "widgets": [
        {
            "type": "metric",
            "x": 0, "y": 0, "width": 12, "height": 6,
            "properties": {
                "title": "Edge Inference Latency (p50/p95/p99)",
                "metrics": [
                    [namespace, "EdgeInferenceLatency", {"stat": "p50", "label": "p50"}],
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
                "title": "Inference Throughput per Device",
                "metrics": [
                    [namespace, "EdgeInferenceCount", {"stat": "Sum"}],
                ],
                "period": 60,
                "view": "timeSeries",
            },
        },
        {
            "type": "metric",
            "x": 0, "y": 6, "width": 12, "height": 6,
            "properties": {
                "title": "Active Devices (Heartbeat in Last 15 min)",
                "metrics": [
                    [namespace, "EdgeDeviceHeartbeat", {"stat": "Sum"}],
                ],
                "period": 900,
                "view": "singleValue",
            },
        },
        {
            "type": "metric",
            "x": 12, "y": 6, "width": 12, "height": 6,
            "properties": {
                "title": "Prediction Confidence Distribution",
                "metrics": [
                    [namespace, "EdgePredictionConfidence", {"stat": "Average"}],
                    ["...", {"stat": "Minimum"}],
                    ["...", {"stat": "Maximum"}],
                ],
                "period": 300,
                "view": "timeSeries",
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

**DEPRECATION:** SageMaker Edge Manager APIs (`create_device_fleet`, `register_devices`, `create_edge_packaging_job`, `describe_device`, `deregister_devices`) are no longer available as of April 26, 2024. The fleet management and edge packaging code in this template is preserved for reference only. Use IoT Greengrass V2 (`edge/02`) for device fleet management and model deployment. SageMaker Neo compilation (`create_compilation_job`) remains fully supported.

**Compilation:** SageMaker Neo supports specific framework versions per target platform. Verify compatibility in the [Neo supported frameworks documentation](https://docs.aws.amazon.com/sagemaker/latest/dg/neo-supported-devices-edge-frameworks.html). Compilation jobs can take 5–15 minutes depending on model size. Use `ml.c5.4xlarge` or larger for compilation instances.

**Device Fleet:** Each device fleet requires an IoT role alias mapped to an IAM role. The IAM role must grant `sagemaker:SendHeartbeat`, `sagemaker:GetDeviceRegistration`, and `s3:PutObject` for data capture. Device names must be unique within a fleet and match IoT thing names for OTA integration.

**Edge Packaging:** Packaged models include the compiled model artifact and the Edge Manager agent runtime. The agent runs on the device and handles model loading, inference, data capture, and heartbeat reporting. Ensure the target device has sufficient storage for the packaged model.

**OTA Updates:** IoT Jobs require devices to be registered as IoT things in the same region. Use `SNAPSHOT` target selection for one-time deployments and `CONTINUOUS` for auto-deploying to newly registered devices. Set abort thresholds to prevent fleet-wide failures from bad model versions.

**Security:** Use IoT role aliases (not long-lived credentials) for device authentication. Encrypt model artifacts at rest in S3 using KMS keys from `devops/08`. Edge Manager agent communicates over TLS. Restrict IoT policies to minimum required actions.

**Cost:** Neo compilation jobs are billed per compilation instance hour. Edge Manager has no per-device charge — costs are S3 storage for data capture and CloudWatch for metrics. IoT Jobs are billed per remote action. Monitor data capture volume to control S3 costs in production fleets.

**Naming:** All resources follow `{PROJECT_NAME}-{component}-{ENV}` convention:
- Compilation job: `{PROJECT_NAME}-neo-{TARGET_PLATFORM}-{ENV}-{timestamp}`
- Device fleet: `{PROJECT_NAME}-{DEVICE_FLEET_NAME}-{ENV}`
- Packaging job: `{PROJECT_NAME}-edge-pkg-{ENV}-{timestamp}`
- IoT Job: `{PROJECT_NAME}-ota-{model_version}-{ENV}-{timestamp}`
- Dashboard: `{PROJECT_NAME}-edge-fleet-{ENV}`
- Lambda: `{PROJECT_NAME}-edge-metric-aggregator-{ENV}`

---

## Integration Points

- **Upstream**: `devops/04` → IAM roles for SageMaker execution role, IoT device role, and Lambda execution role
- **Upstream**: `mlops/01` → Trained model artifact in S3 (MODEL_S3_URI) from SageMaker training pipeline
- **Downstream**: `edge/02` → IoT Greengrass ML inference consumes Neo-compiled models from this template
- **Downstream**: `devops/03` → CloudWatch dashboards for fleet monitoring metrics
- **Downstream**: `devops/11` → Custom CloudWatch metrics published by the edge metric aggregator Lambda
