<!-- Template Version: 1.0 | boto3: 1.35+ | Greengrass V2 SDK: 1.x -->

# Template Edge 02 — IoT Greengrass V2 ML Inference

## Purpose
Generate a production-ready IoT Greengrass V2 edge ML inference pipeline: custom Greengrass component recipes with ML model artifacts, deployment configurations targeting thing groups with resource limits, model optimization scripts (TFLite conversion, ONNX INT8 quantization, PyTorch Mobile), Stream Manager buffered upload to S3 and Kinesis, offline inference fallback patterns, and component lifecycle health checks.

---

## Role Definition

You are an expert AWS IoT and edge ML engineer specializing in IoT Greengrass V2 with expertise in:
- Greengrass V2: custom component development with YAML recipes, artifact management, lifecycle scripts (install/run/shutdown), and inter-process communication (IPC)
- ML model optimization: TensorFlow Lite conversion with representative datasets, ONNX Runtime INT8 quantization, PyTorch Mobile tracing and scripting for resource-constrained devices
- Greengrass Stream Manager: buffered, prioritized data export to S3 and Kinesis with configurable persistence, bandwidth limits, and retry policies
- Greengrass deployments: thing group targeting, component version pinning, rollback configuration, resource limits (memory/CPU), and failure handling policies
- Edge inference patterns: batch and real-time inference loops, offline fallback with local result queuing, model hot-swap without service interruption
- IAM and IoT policies for Greengrass core devices, token exchange roles, and S3/Kinesis access

Generate complete, production-deployable code.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

COMPONENT_NAME:         [REQUIRED - name for the Greengrass ML inference component]
                        Example: image-classifier

ML_FRAMEWORK:           [REQUIRED - ML framework for inference runtime]
                        Options:
                        - pytorch (PyTorch / PyTorch Mobile)
                        - tflite (TensorFlow Lite)
                        - onnx (ONNX Runtime)

MODEL_S3_URI:           [REQUIRED - S3 URI of the model artifact (tar.gz or directory)]
                        Example: s3://my-bucket/models/classifier-v1.tar.gz

THING_GROUP_NAME:       [REQUIRED - IoT thing group for deployment targeting]
                        Example: factory-edge-devices

RESOURCE_LIMITS:        [OPTIONAL: {"memory_mb": 512, "cpu": 1.0} - resource constraints for the component]
                        memory_mb: maximum memory in MB
                        cpu: maximum vCPU allocation (e.g., 0.5, 1.0, 2.0)

STREAM_MANAGER_DESTINATION:
                        [OPTIONAL: s3 - destination for inference results]
                        Options:
                        - s3 (buffered upload to S3 bucket)
                        - kinesis (stream to Kinesis Data Stream)
                        - both (upload to S3 and stream to Kinesis)

OPTIMIZATION_TYPE:      [OPTIONAL: none - model optimization to apply before deployment]
                        Options:
                        - none (deploy model as-is)
                        - tflite (convert to TensorFlow Lite with quantization)
                        - onnx_int8 (ONNX Runtime INT8 quantization)
                        - torch_mobile (PyTorch Mobile optimization via tracing)

INFERENCE_MODE:         [OPTIONAL: realtime - inference execution pattern]
                        Options:
                        - realtime (continuous inference loop on incoming data)
                        - batch (process files from a local directory on schedule)

INPUT_SOURCE:           [OPTIONAL: local_dir - where inference input data comes from]
                        Options:
                        - local_dir (watch a local directory for new files)
                        - ipc (receive data via Greengrass IPC from other components)
                        - camera (capture from local camera device)

S3_RESULTS_BUCKET:      [OPTIONAL: "" - S3 bucket for Stream Manager uploads]
                        Example: my-project-edge-results

KINESIS_STREAM_NAME:    [OPTIONAL: "" - Kinesis stream for real-time result streaming]
                        Example: my-project-edge-stream

MODEL_VERSION:          [OPTIONAL: 1.0.0 - semantic version for the component]

ROLLBACK_ENABLED:       [OPTIONAL: true - enable automatic rollback on deployment failure]
```

---

## Task

Generate complete IoT Greengrass V2 ML inference component and deployment pipeline:

```
{PROJECT_NAME}-greengrass-ml/
├── component/
│   ├── recipe.yaml                        # Greengrass V2 component recipe
│   ├── inference_engine.py                # Main inference loop (framework-specific)
│   ├── stream_export.py                   # Stream Manager client for result upload
│   ├── health_check.py                    # Component lifecycle health check
│   ├── offline_fallback.py                # Offline inference with local queuing
│   └── requirements.txt                   # Python dependencies for the component
├── optimization/
│   ├── optimize_tflite.py                 # TFLite conversion + quantization
│   ├── optimize_onnx.py                   # ONNX INT8 quantization
│   └── optimize_torch_mobile.py           # PyTorch Mobile tracing
├── deployment/
│   ├── create_component.py                # Register component version in Greengrass
│   ├── deploy.py                          # Create deployment targeting thing group
│   ├── rollback.py                        # Rollback to previous component version
│   └── deployment_status.py               # Check deployment status per device
├── monitoring/
│   └── dashboard.py                       # CloudWatch dashboard for edge fleet
├── cleanup/
│   └── delete_resources.py                # Remove deployments, components, artifacts
├── config.py                              # Central configuration
├── run_setup.py                           # CLI orchestrator
└── requirements.txt                       # Cloud-side dependencies
```

**config.py**: Central configuration dataclass with all parameters. Load from environment variables or CLI args. Validate required fields. Construct resource names using `{PROJECT_NAME}-{component}-{ENV}` convention. Map ML_FRAMEWORK to runtime dependencies and inference API. Map OPTIMIZATION_TYPE to optimization script selection.

**recipe.yaml**: Greengrass V2 component recipe in YAML format:
- `RecipeFormatVersion` set to `2020-01-25`
- `ComponentName` set to `{PROJECT_NAME}-{COMPONENT_NAME}-{ENV}`
- `ComponentVersion` set to MODEL_VERSION
- `ComponentDescription` describing the ML inference component
- `ComponentPublisher` set to PROJECT_NAME
- `ComponentConfiguration` with `DefaultConfiguration` containing: `ModelFramework` (ML_FRAMEWORK), `InferenceMode` (INFERENCE_MODE), `InputSource` (INPUT_SOURCE), `StreamDestination` (STREAM_MANAGER_DESTINATION), `HealthCheckIntervalSec` (30)
- `ComponentDependencies` on `aws.greengrass.StreamManager` (version `>=2.0.0`) and `aws.greengrass.TokenExchangeService`
- `Manifests` with `Platform` (os: linux), `Lifecycle` scripts:
  - `install`: `pip3 install -r {artifacts:decompressedPath}/{COMPONENT_NAME}/requirements.txt`
  - `run`: `python3 {artifacts:decompressedPath}/{COMPONENT_NAME}/inference_engine.py`
  - `shutdown`: `python3 {artifacts:decompressedPath}/{COMPONENT_NAME}/health_check.py --shutdown`
- `Artifacts` with two entries:
  - Component code archive: S3 URI to the zipped component code with `Unarchive: ZIP`
  - Model artifact: MODEL_S3_URI with `Unarchive: NONE` (or `ZIP`/`NONE` depending on format)
- `LinuxProcessParams` with `ResourceLimits` setting `Memory` (RESOURCE_LIMITS.memory_mb * 1024 in KB) and `Cpus` (RESOURCE_LIMITS.cpu)

**inference_engine.py**: Main inference loop, framework-specific:
- Load model based on ML_FRAMEWORK:
  - `pytorch`: Load with `torch.jit.load()` for TorchScript or `torch.load()` for standard models
  - `tflite`: Load with `tflite_runtime.interpreter.Interpreter(model_path=...)`
  - `onnx`: Load with `onnxruntime.InferenceSession(model_path)`
- Input handling based on INPUT_SOURCE:
  - `local_dir`: Watch directory using polling loop, process new files
  - `ipc`: Subscribe to Greengrass IPC topic using `awsiot.greengrasscoreipc` client
  - `camera`: Capture frames using OpenCV `cv2.VideoCapture(0)`
- Run inference loop:
  - Preprocess input (resize, normalize, convert to tensor)
  - Execute inference using framework-specific API
  - Post-process output (argmax, confidence threshold, label mapping)
  - Send results to `stream_export.py` for upload
  - If network unavailable, delegate to `offline_fallback.py`
- Handle graceful shutdown on SIGTERM/SIGINT
- Log inference metrics: latency, throughput, confidence scores

**stream_export.py**: Stream Manager client for buffered result upload:
- Initialize Stream Manager client using `stream_manager.StreamManagerClient()`
- Create or connect to a message stream with configurable settings:
  - `MessageStreamDefinition` with `name`, `maxSize` (256 MB default), `streamSegmentSize` (16 MB), `strategyOnFull` (OverwriteOldestData)
  - `ExportDefinition` with either:
    - `S3TaskExecutor` targeting S3_RESULTS_BUCKET with key prefix `{PROJECT_NAME}/{ENV}/results/`
    - `KinesisTaskExecutor` targeting KINESIS_STREAM_NAME with batch size and interval
    - Both if STREAM_MANAGER_DESTINATION is `both`
- `export_result(result_dict)` function: append JSON-serialized inference result to the stream
- Handle `StreamManagerException` for connection failures with retry logic
- Include `flush()` method for graceful shutdown to ensure all buffered data is exported

**health_check.py**: Component lifecycle health check:
- When called without args: run periodic health check
  - Verify model file exists and is loadable
  - Verify Stream Manager connection is active
  - Verify inference loop is responsive (check heartbeat file timestamp)
  - Write health status to a local file and publish to Greengrass IPC topic `{PROJECT_NAME}/health`
  - Publish CloudWatch custom metric `ComponentHealth` (1=healthy, 0=unhealthy) via Stream Manager or direct SDK call
- When called with `--shutdown`: graceful shutdown
  - Signal inference loop to stop
  - Flush Stream Manager buffers
  - Log final metrics summary

**offline_fallback.py**: Offline inference with local result queuing:
- Detect network connectivity by attempting Stream Manager connection or DNS resolution
- When offline:
  - Continue running inference locally (model is already on device)
  - Queue results to a local SQLite database or JSON file in `/tmp/{PROJECT_NAME}/offline_queue/`
  - Track queue size and age of oldest result
- When connectivity restored:
  - Drain offline queue through Stream Manager in chronological order
  - Delete successfully uploaded results from local queue
  - Log queue drain metrics (count, duration, oldest result age)
- Configurable max queue size (default 10,000 results) with oldest-first eviction

**requirements.txt** (component): Framework-specific dependencies:
- `pytorch`: `torch>=2.0`, `torchvision>=0.15`
- `tflite`: `tflite-runtime>=2.14`
- `onnx`: `onnxruntime>=1.16`
- Common: `numpy>=1.24`, `Pillow>=10.0`, `awsiotsdk>=1.0`, `stream_manager>=1.0`

**optimize_tflite.py**: TensorFlow Lite conversion and quantization:
- Load source TensorFlow SavedModel or Keras model
- Create `tf.lite.TFLiteConverter` from saved model
- Apply post-training quantization: `converter.optimizations = [tf.lite.Optimize.DEFAULT]`
- Optionally provide representative dataset for full integer quantization: `converter.representative_dataset = representative_data_gen`
- Set `converter.target_spec.supported_types = [tf.float16]` for float16 quantization or `[tf.int8]` for int8
- Convert and save `.tflite` file
- Log model size reduction (original vs optimized)
- Upload optimized model to S3 at `{MODEL_S3_URI}-optimized/`

**optimize_onnx.py**: ONNX Runtime INT8 quantization:
- Load ONNX model using `onnx.load()`
- Run `onnxruntime.quantization.quantize_dynamic()` for dynamic quantization or `quantize_static()` with calibration data
- Set `weight_type` to `QuantType.QInt8`
- Validate quantized model with `onnxruntime.InferenceSession` and sample input
- Save quantized model and log size reduction
- Upload optimized model to S3

**optimize_torch_mobile.py**: PyTorch Mobile optimization:
- Load PyTorch model and set to eval mode
- Create example input tensor matching model input shape
- Trace model using `torch.jit.trace(model, example_input)`
- Optimize for mobile using `torch.utils.mobile_optimizer.optimize_for_mobile(traced_model)`
- Save optimized model with `traced_model._save_for_lite_interpreter(output_path)`
- Validate by running inference on the optimized model
- Upload optimized model to S3

**create_component.py**: Register Greengrass component version:
- Read and validate `recipe.yaml`
- Upload component code archive to S3: `s3://{S3_RESULTS_BUCKET}/components/{COMPONENT_NAME}/{MODEL_VERSION}/`
- Upload model artifact to S3 if not already present
- Update recipe artifact URIs to point to uploaded S3 locations
- Call `greengrassv2.create_component_version()` with `inlineRecipe` (YAML bytes) to register the component
- Verify component creation with `greengrassv2.describe_component()` using the component ARN

**deploy.py**: Create Greengrass deployment targeting thing group:
- Build deployment configuration:
  - `targetArn`: thing group ARN from THING_GROUP_NAME
  - `deploymentName`: `{PROJECT_NAME}-{COMPONENT_NAME}-{ENV}-{timestamp}`
  - `components`: dict with component name → version and configuration merge
  - `deploymentPolicies`:
    - `failureHandlingPolicy`: `ROLLBACK` if ROLLBACK_ENABLED else `DO_NOTHING`
    - `componentUpdatePolicy`: `NOTIFY_COMPONENTS` with 60-second timeout
  - `iotJobConfiguration`:
    - `jobExecutionsRolloutConfig` with `maximumPerMinute` (10 for prod, 50 for dev)
    - `abortConfig` with failure threshold (25% for prod, 50% for dev)
- Call `greengrassv2.create_deployment()` with the configuration
- Log deployment ID and target

**rollback.py**: Rollback to previous component version:
- Call `greengrassv2.list_components()` to find previous versions of the component
- Identify the last known good version (previous to current)
- Create a new deployment with the previous component version
- Monitor rollback deployment status

**deployment_status.py**: Check deployment status:
- Call `greengrassv2.get_deployment()` to get overall deployment status
- Call `greengrassv2.list_effective_deployments()` for each core device in the thing group
- Report per-device status: COMPLETED, IN_PROGRESS, FAILED, CANCELED
- For failed devices, call `greengrassv2.get_component()` to check component error logs

**dashboard.py**: CloudWatch dashboard for Greengrass ML fleet:
- Widget 1: Inference latency (p50/p95/p99) from custom metrics
- Widget 2: Inference throughput per device (count per minute)
- Widget 3: Component health status across fleet
- Widget 4: Stream Manager export success/failure rates
- Widget 5: Offline queue depth (when devices lose connectivity)
- Widget 6: Model version distribution across fleet
- Dashboard name: `{PROJECT_NAME}-greengrass-ml-{ENV}`

**delete_resources.py**: Clean up all resources:
- Cancel active deployments using `greengrassv2.cancel_deployment()`
- Delete component versions using `greengrassv2.delete_component()`
- Delete component artifacts from S3
- Delete CloudWatch dashboard
- Optionally remove thing group (with confirmation prompt)

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Code Scaffolding Hints

**Greengrass V2 component recipe (YAML):**
```yaml
---
RecipeFormatVersion: "2020-01-25"
ComponentName: "{PROJECT_NAME}-{COMPONENT_NAME}-{ENV}"
ComponentVersion: "1.0.0"
ComponentDescription: "ML inference component using {ML_FRAMEWORK} for edge devices"
ComponentPublisher: "{PROJECT_NAME}"
ComponentConfiguration:
  DefaultConfiguration:
    ModelFramework: "pytorch"        # pytorch | tflite | onnx
    InferenceMode: "realtime"        # realtime | batch
    InputSource: "local_dir"         # local_dir | ipc | camera
    InputDir: "/tmp/ml_input"
    StreamDestination: "s3"          # s3 | kinesis | both
    HealthCheckIntervalSec: 30
    ConfidenceThreshold: 0.5
    BatchSize: 1
ComponentDependencies:
  aws.greengrass.StreamManager:
    VersionRequirement: ">=2.0.0"
    DependencyType: HARD
  aws.greengrass.TokenExchangeService:
    VersionRequirement: ">=2.0.0"
    DependencyType: HARD
Manifests:
  - Platform:
      os: linux
    Lifecycle:
      install:
        script: |-
          pip3 install -r {artifacts:decompressedPath}/component/requirements.txt
        timeout: 300
      run:
        script: |-
          python3 {artifacts:decompressedPath}/component/inference_engine.py \
            --model-path {artifacts:path}/model \
            --config '{configuration:/}'
        timeout: -1   # Run indefinitely
      shutdown:
        script: |-
          python3 {artifacts:decompressedPath}/component/health_check.py --shutdown
        timeout: 30
    Artifacts:
      - URI: "s3://{BUCKET}/components/{COMPONENT_NAME}/{VERSION}/component.zip"
        Unarchive: ZIP
      - URI: "s3://{BUCKET}/models/{COMPONENT_NAME}/{VERSION}/model.tar.gz"
        Unarchive: NONE
    LinuxProcessParams:
      ResourceLimits:
        Memory: 524288    # 512 MB in KB
        Cpus: 1.0
```

**Register component version:**
```python
import boto3
import yaml

greengrassv2 = boto3.client("greengrassv2", region_name=AWS_REGION)

# Read and encode recipe
with open("component/recipe.yaml", "r") as f:
    recipe_content = f.read()

response = greengrassv2.create_component_version(
    inlineRecipe=recipe_content.encode("utf-8"),
    tags={
        "Project": PROJECT_NAME,
        "Environment": ENV,
    },
)

component_arn = response["arn"]
component_name = response["componentName"]
component_version = response["componentVersion"]
print(f"Component registered: {component_name} v{component_version}")
print(f"ARN: {component_arn}")

# Verify
desc = greengrassv2.describe_component(arn=component_arn)
print(f"Status: {desc['status']['componentState']}")
```

**Create deployment targeting thing group:**
```python
import time

greengrassv2 = boto3.client("greengrassv2", region_name=AWS_REGION)

thing_group_arn = (
    f"arn:aws:iot:{AWS_REGION}:{AWS_ACCOUNT_ID}:thinggroup/{THING_GROUP_NAME}"
)

component_name = f"{PROJECT_NAME}-{COMPONENT_NAME}-{ENV}"
deployment_name = f"{PROJECT_NAME}-{COMPONENT_NAME}-{ENV}-{int(time.time())}"

response = greengrassv2.create_deployment(
    targetArn=thing_group_arn,
    deploymentName=deployment_name,
    components={
        component_name: {
            "componentVersion": MODEL_VERSION,
            "configurationUpdate": {
                "merge": '{"ModelFramework":"' + ML_FRAMEWORK + '",'
                         '"InferenceMode":"' + INFERENCE_MODE + '",'
                         '"StreamDestination":"' + STREAM_MANAGER_DESTINATION + '"}'
            },
            "runWith": {
                "posixUser": "ggc_user:ggc_group",
            },
        },
        # Stream Manager dependency (managed component)
        "aws.greengrass.StreamManager": {
            "componentVersion": "2.1.0",
            "configurationUpdate": {
                "merge": '{"STREAM_MANAGER_STORE_ROOT_DIR":"/tmp/stream_manager"}'
            },
        },
    },
    deploymentPolicies={
        "failureHandlingPolicy": "ROLLBACK",  # ROLLBACK | DO_NOTHING
        "componentUpdatePolicy": {
            "timeoutInSeconds": 60,
            "action": "NOTIFY_COMPONENTS",  # NOTIFY_COMPONENTS | SKIP_NOTIFY_COMPONENTS
        },
    },
    iotJobConfiguration={
        "jobExecutionsRolloutConfig": {
            "maximumPerMinute": 10,
        },
        "abortConfig": {
            "criteriaList": [
                {
                    "failureType": "FAILED",
                    "action": "CANCEL",
                    "thresholdPercentage": 25.0,
                    "minNumberOfExecutedThings": 2,
                }
            ]
        },
        "timeoutConfig": {
            "inProgressTimeoutInMinutes": 15,
        },
    },
    tags={
        "Project": PROJECT_NAME,
        "Environment": ENV,
    },
)

deployment_id = response["deploymentId"]
print(f"Deployment created: {deployment_id}")
print(f"Target: {THING_GROUP_NAME}")
```

**Check deployment status:**
```python
def check_deployment_status(deployment_id):
    """Check overall deployment and per-device status."""
    deployment = greengrassv2.get_deployment(deploymentId=deployment_id)
    print(f"Deployment: {deployment_id}")
    print(f"  Status: {deployment['deploymentStatus']}")
    print(f"  Target: {deployment['targetArn']}")

    # List core devices in the thing group and check each
    iot = boto3.client("iot", region_name=AWS_REGION)
    things = iot.list_things_in_thing_group(thingGroupName=THING_GROUP_NAME)

    for thing in things.get("things", []):
        try:
            effective = greengrassv2.list_effective_deployments(
                coreDeviceThingName=thing
            )
            for dep in effective.get("effectiveDeployments", []):
                if dep["deploymentId"] == deployment_id:
                    print(
                        f"  Device {thing}: {dep['coreDeviceExecutionStatus']} "
                        f"(reason: {dep.get('reason', 'N/A')})"
                    )
        except Exception as e:
            print(f"  Device {thing}: Error checking status - {e}")
```

**Stream Manager client for result export:**
```python
import json
import logging
from stream_manager import (
    StreamManagerClient,
    MessageStreamDefinition,
    StrategyOnFull,
    ExportDefinition,
    S3ExportTaskExecutorConfig,
    S3ExportTaskDefinition,
    KinesisConfig,
    StatusConfig,
    StatusLevel,
    StatusMessage,
)

logger = logging.getLogger(__name__)

STREAM_NAME = f"{PROJECT_NAME}-inference-results"
STATUS_STREAM = f"{PROJECT_NAME}-stream-status"


def create_stream_manager_client():
    """Initialize Stream Manager client and create export stream."""
    client = StreamManagerClient()

    # Build export definitions based on destination
    export_definition = ExportDefinition()

    if STREAM_MANAGER_DESTINATION in ("s3", "both"):
        export_definition = ExportDefinition(
            s3_task_executor=[
                S3ExportTaskExecutorConfig(
                    identifier=f"{PROJECT_NAME}-s3-export",
                    size_threshold_for_multipart_upload_bytes=5 * 1024 * 1024,
                    status_config=StatusConfig(
                        status_level=StatusLevel.INFO,
                        status_stream_name=STATUS_STREAM,
                    ),
                )
            ]
        )

    if STREAM_MANAGER_DESTINATION in ("kinesis", "both"):
        kinesis_configs = [
            KinesisConfig(
                identifier=f"{PROJECT_NAME}-kinesis-export",
                kinesis_stream_name=KINESIS_STREAM_NAME,
                batch_size=10,
                batch_interval_millis=5000,
            )
        ]
        if export_definition.s3_task_executor:
            export_definition.kinesis = kinesis_configs
        else:
            export_definition = ExportDefinition(kinesis=kinesis_configs)

    try:
        client.create_message_stream(
            MessageStreamDefinition(
                name=STREAM_NAME,
                max_size=268435456,  # 256 MB
                stream_segment_size=16777216,  # 16 MB
                strategy_on_full=StrategyOnFull.OverwriteOldestData,
                export_definition=export_definition,
            )
        )
        logger.info(f"Created stream: {STREAM_NAME}")
    except Exception:
        # Stream may already exist from previous deployment
        logger.info(f"Stream already exists: {STREAM_NAME}")

    return client


def export_result(client, result_dict):
    """Export inference result through Stream Manager."""
    if STREAM_MANAGER_DESTINATION in ("s3", "both"):
        # For S3 export, wrap result in S3ExportTaskDefinition
        s3_task = S3ExportTaskDefinition(
            input_url=None,  # Will use inline data
            bucket=S3_RESULTS_BUCKET,
            key=f"{PROJECT_NAME}/{ENV}/results/{result_dict['timestamp']}.json",
        )
        client.append_message(
            STREAM_NAME,
            json.dumps(result_dict).encode("utf-8"),
        )
    else:
        # For Kinesis, append raw JSON
        client.append_message(
            STREAM_NAME,
            json.dumps(result_dict).encode("utf-8"),
        )
    logger.debug(f"Exported result: {result_dict.get('inference_id', 'unknown')}")


def flush_and_close(client):
    """Flush remaining data and close Stream Manager client."""
    try:
        # Read status stream for any export errors
        try:
            messages = client.read_messages(
                STATUS_STREAM, min_message_count=1, read_timeout_millis=1000
            )
            for msg in messages:
                status = StatusMessage.from_json_bytes(msg.payload)
                if status.status_level == StatusLevel.ERROR:
                    logger.error(f"Export error: {status.message}")
        except Exception:
            pass
        client.close()
        logger.info("Stream Manager client closed")
    except Exception as e:
        logger.error(f"Error closing Stream Manager: {e}")
```

**Inference engine (PyTorch example):**
```python
import torch
import numpy as np
import time
import json
import signal
import logging
import os
from pathlib import Path

logger = logging.getLogger(__name__)

# Graceful shutdown flag
shutdown_requested = False


def signal_handler(signum, frame):
    global shutdown_requested
    shutdown_requested = True
    logger.info("Shutdown signal received")


signal.signal(signal.SIGTERM, signal_handler)
signal.signal(signal.SIGINT, signal_handler)


def load_model_pytorch(model_path):
    """Load PyTorch model for inference."""
    device = torch.device("cpu")
    model = torch.jit.load(model_path, map_location=device)
    model.eval()
    logger.info(f"PyTorch model loaded from {model_path}")
    return model


def load_model_tflite(model_path):
    """Load TFLite model for inference."""
    import tflite_runtime.interpreter as tflite

    interpreter = tflite.Interpreter(model_path=model_path)
    interpreter.allocate_tensors()
    logger.info(f"TFLite model loaded from {model_path}")
    return interpreter


def load_model_onnx(model_path):
    """Load ONNX model for inference."""
    import onnxruntime as ort

    session = ort.InferenceSession(
        model_path,
        providers=["CPUExecutionProvider"],
    )
    logger.info(f"ONNX model loaded from {model_path}")
    return session


def run_inference_pytorch(model, input_data):
    """Run inference with PyTorch model."""
    with torch.no_grad():
        tensor = torch.from_numpy(input_data).float()
        output = model(tensor)
        probabilities = torch.softmax(output, dim=1)
        confidence, predicted = torch.max(probabilities, 1)
        return {
            "class_id": predicted.item(),
            "confidence": confidence.item(),
            "raw_output": output.numpy().tolist(),
        }


def run_inference_tflite(interpreter, input_data):
    """Run inference with TFLite model."""
    input_details = interpreter.get_input_details()
    output_details = interpreter.get_output_details()
    interpreter.set_tensor(input_details[0]["index"], input_data.astype(np.float32))
    interpreter.invoke()
    output = interpreter.get_tensor(output_details[0]["index"])
    class_id = int(np.argmax(output))
    confidence = float(np.max(output))
    return {
        "class_id": class_id,
        "confidence": confidence,
        "raw_output": output.tolist(),
    }


def run_inference_onnx(session, input_data):
    """Run inference with ONNX model."""
    input_name = session.get_inputs()[0].name
    output = session.run(None, {input_name: input_data.astype(np.float32)})
    class_id = int(np.argmax(output[0]))
    confidence = float(np.max(output[0]))
    return {
        "class_id": class_id,
        "confidence": confidence,
        "raw_output": output[0].tolist(),
    }


def inference_loop(model, framework, input_dir, stream_client, config):
    """Main inference loop watching input directory."""
    processed_files = set()
    heartbeat_path = f"/tmp/{PROJECT_NAME}_heartbeat"

    FRAMEWORK_RUNNERS = {
        "pytorch": run_inference_pytorch,
        "tflite": run_inference_tflite,
        "onnx": run_inference_onnx,
    }
    run_fn = FRAMEWORK_RUNNERS[framework]

    while not shutdown_requested:
        # Update heartbeat for health check
        Path(heartbeat_path).write_text(str(time.time()))

        # Scan input directory for new files
        input_path = Path(input_dir)
        if input_path.exists():
            for file_path in sorted(input_path.glob("*")):
                if file_path.name in processed_files:
                    continue
                if not file_path.is_file():
                    continue

                try:
                    start_time = time.time()

                    # Load and preprocess input
                    input_data = preprocess_input(file_path)

                    # Run inference
                    result = run_fn(model, input_data)
                    latency_ms = (time.time() - start_time) * 1000

                    # Build result record
                    result_record = {
                        "inference_id": f"{int(time.time() * 1000)}",
                        "timestamp": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime()),
                        "device_name": os.environ.get("AWS_IOT_THING_NAME", "unknown"),
                        "model_version": MODEL_VERSION,
                        "input_file": file_path.name,
                        "class_id": result["class_id"],
                        "confidence": result["confidence"],
                        "latency_ms": round(latency_ms, 2),
                    }

                    # Export via Stream Manager
                    from stream_export import export_result
                    export_result(stream_client, result_record)

                    processed_files.add(file_path.name)
                    logger.info(
                        f"Inference: {file_path.name} → class={result['class_id']} "
                        f"conf={result['confidence']:.3f} latency={latency_ms:.1f}ms"
                    )

                except Exception as e:
                    logger.error(f"Inference error on {file_path.name}: {e}")

        time.sleep(0.1)  # Polling interval

    logger.info("Inference loop stopped")
```

**Offline fallback pattern:**
```python
import json
import sqlite3
import time
import logging
from pathlib import Path

logger = logging.getLogger(__name__)

QUEUE_DIR = f"/tmp/{PROJECT_NAME}/offline_queue"
QUEUE_DB = f"{QUEUE_DIR}/queue.db"
MAX_QUEUE_SIZE = 10000


def init_offline_queue():
    """Initialize SQLite-based offline queue."""
    Path(QUEUE_DIR).mkdir(parents=True, exist_ok=True)
    conn = sqlite3.connect(QUEUE_DB)
    conn.execute("""
        CREATE TABLE IF NOT EXISTS results (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            timestamp TEXT NOT NULL,
            payload TEXT NOT NULL,
            created_at REAL NOT NULL
        )
    """)
    conn.commit()
    return conn


def queue_result(conn, result_dict):
    """Queue inference result locally when offline."""
    # Enforce max queue size (evict oldest)
    count = conn.execute("SELECT COUNT(*) FROM results").fetchone()[0]
    if count >= MAX_QUEUE_SIZE:
        conn.execute(
            "DELETE FROM results WHERE id IN "
            "(SELECT id FROM results ORDER BY created_at ASC LIMIT ?)",
            (count - MAX_QUEUE_SIZE + 1,),
        )

    conn.execute(
        "INSERT INTO results (timestamp, payload, created_at) VALUES (?, ?, ?)",
        (result_dict["timestamp"], json.dumps(result_dict), time.time()),
    )
    conn.commit()
    logger.info(f"Queued offline result (queue size: {count + 1})")


def drain_queue(conn, stream_client):
    """Drain offline queue through Stream Manager when connectivity restored."""
    from stream_export import export_result

    cursor = conn.execute(
        "SELECT id, payload FROM results ORDER BY created_at ASC"
    )
    drained = 0
    for row in cursor:
        row_id, payload = row
        try:
            result_dict = json.loads(payload)
            export_result(stream_client, result_dict)
            conn.execute("DELETE FROM results WHERE id = ?", (row_id,))
            drained += 1
        except Exception as e:
            logger.error(f"Failed to drain result {row_id}: {e}")
            break  # Stop draining on first failure (likely still offline)

    conn.commit()
    if drained > 0:
        logger.info(f"Drained {drained} results from offline queue")
    return drained


def is_online():
    """Check if Stream Manager / network is available."""
    try:
        from stream_manager import StreamManagerClient
        client = StreamManagerClient()
        client.close()
        return True
    except Exception:
        return False
```

**TFLite optimization script:**
```python
import tensorflow as tf
import numpy as np
import os


def convert_to_tflite(saved_model_dir, output_path, quantization="dynamic"):
    """Convert TensorFlow SavedModel to TFLite with quantization."""
    converter = tf.lite.TFLiteConverter.from_saved_model(saved_model_dir)

    if quantization == "dynamic":
        converter.optimizations = [tf.lite.Optimize.DEFAULT]
    elif quantization == "float16":
        converter.optimizations = [tf.lite.Optimize.DEFAULT]
        converter.target_spec.supported_types = [tf.float16]
    elif quantization == "int8":
        converter.optimizations = [tf.lite.Optimize.DEFAULT]
        converter.target_spec.supported_types = [tf.int8]

        # Representative dataset for full integer quantization
        def representative_data_gen():
            for _ in range(100):
                yield [np.random.randn(1, 224, 224, 3).astype(np.float32)]

        converter.representative_dataset = representative_data_gen

    tflite_model = converter.convert()

    with open(output_path, "wb") as f:
        f.write(tflite_model)

    original_size = sum(
        os.path.getsize(os.path.join(dp, f))
        for dp, _, fn in os.walk(saved_model_dir)
        for f in fn
    )
    optimized_size = os.path.getsize(output_path)
    reduction = (1 - optimized_size / original_size) * 100
    print(f"Original: {original_size / 1024 / 1024:.1f} MB")
    print(f"Optimized: {optimized_size / 1024 / 1024:.1f} MB")
    print(f"Reduction: {reduction:.1f}%")
    return output_path
```

**ONNX INT8 quantization script:**
```python
import onnx
from onnxruntime.quantization import quantize_dynamic, QuantType
import os


def quantize_onnx_model(input_model_path, output_model_path):
    """Apply dynamic INT8 quantization to ONNX model."""
    quantize_dynamic(
        model_input=input_model_path,
        model_output=output_model_path,
        weight_type=QuantType.QInt8,
    )

    original_size = os.path.getsize(input_model_path)
    quantized_size = os.path.getsize(output_model_path)
    reduction = (1 - quantized_size / original_size) * 100
    print(f"Original: {original_size / 1024 / 1024:.1f} MB")
    print(f"Quantized: {quantized_size / 1024 / 1024:.1f} MB")
    print(f"Reduction: {reduction:.1f}%")

    # Validate quantized model
    import onnxruntime as ort
    import numpy as np

    session = ort.InferenceSession(
        output_model_path, providers=["CPUExecutionProvider"]
    )
    input_name = session.get_inputs()[0].name
    input_shape = session.get_inputs()[0].shape
    # Replace dynamic dims with 1
    shape = [1 if isinstance(d, str) else d for d in input_shape]
    dummy_input = np.random.randn(*shape).astype(np.float32)
    output = session.run(None, {input_name: dummy_input})
    print(f"Validation passed. Output shape: {[o.shape for o in output]}")
    return output_model_path
```

**PyTorch Mobile optimization script:**
```python
import torch
from torch.utils.mobile_optimizer import optimize_for_mobile
import os


def optimize_for_edge(model_path, output_path, input_shape=(1, 3, 224, 224)):
    """Optimize PyTorch model for mobile/edge deployment."""
    model = torch.load(model_path, map_location="cpu")
    model.eval()

    # Trace the model
    example_input = torch.randn(*input_shape)
    traced_model = torch.jit.trace(model, example_input)

    # Optimize for mobile
    optimized_model = optimize_for_mobile(traced_model)

    # Save for lite interpreter
    optimized_model._save_for_lite_interpreter(output_path)

    original_size = os.path.getsize(model_path)
    optimized_size = os.path.getsize(output_path)
    reduction = (1 - optimized_size / original_size) * 100
    print(f"Original: {original_size / 1024 / 1024:.1f} MB")
    print(f"Optimized: {optimized_size / 1024 / 1024:.1f} MB")
    print(f"Reduction: {reduction:.1f}%")

    # Validate
    loaded = torch.jit.load(output_path)
    test_output = loaded(example_input)
    print(f"Validation passed. Output shape: {test_output.shape}")
    return output_path
```

**Component health check:**
```python
import sys
import time
import json
import logging
from pathlib import Path

logger = logging.getLogger(__name__)

HEARTBEAT_PATH = f"/tmp/{PROJECT_NAME}_heartbeat"
HEALTH_STATUS_PATH = f"/tmp/{PROJECT_NAME}_health.json"
HEARTBEAT_TIMEOUT_SEC = 60


def run_health_check():
    """Check component health and report status."""
    checks = {}

    # Check 1: Model file exists
    model_path = Path(MODEL_PATH)
    checks["model_loaded"] = model_path.exists()

    # Check 2: Inference loop is responsive (heartbeat file updated recently)
    heartbeat = Path(HEARTBEAT_PATH)
    if heartbeat.exists():
        last_beat = float(heartbeat.read_text())
        age = time.time() - last_beat
        checks["inference_responsive"] = age < HEARTBEAT_TIMEOUT_SEC
        checks["heartbeat_age_sec"] = round(age, 1)
    else:
        checks["inference_responsive"] = False

    # Check 3: Stream Manager connection
    try:
        from stream_manager import StreamManagerClient
        client = StreamManagerClient()
        client.close()
        checks["stream_manager"] = True
    except Exception:
        checks["stream_manager"] = False

    # Overall health
    checks["healthy"] = all([
        checks["model_loaded"],
        checks["inference_responsive"],
        checks["stream_manager"],
    ])

    # Write status file
    status = {
        "timestamp": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime()),
        "component": f"{PROJECT_NAME}-{COMPONENT_NAME}-{ENV}",
        "checks": checks,
    }
    Path(HEALTH_STATUS_PATH).write_text(json.dumps(status, indent=2))
    logger.info(f"Health check: {'HEALTHY' if checks['healthy'] else 'UNHEALTHY'}")
    return checks


def graceful_shutdown():
    """Graceful shutdown: flush buffers and log final metrics."""
    logger.info("Initiating graceful shutdown...")
    try:
        from stream_export import flush_and_close
        from stream_export import create_stream_manager_client
        client = create_stream_manager_client()
        flush_and_close(client)
    except Exception as e:
        logger.error(f"Error during shutdown flush: {e}")
    logger.info("Shutdown complete")


if __name__ == "__main__":
    if "--shutdown" in sys.argv:
        graceful_shutdown()
    else:
        run_health_check()
```

---

## Requirements & Constraints

**Greengrass Core:** Target devices must run Greengrass V2 nucleus 2.9+ with Java 11+ runtime. The Greengrass core device must be registered as an IoT thing with a valid certificate and IoT policy granting `greengrass:*` and `iot:*` actions. The token exchange role must grant S3 read access for component artifacts and model downloads.

**Stream Manager:** Requires the `aws.greengrass.StreamManager` managed component (v2.0+). Stream Manager persists data to local disk — ensure sufficient storage for the configured `maxSize` (default 256 MB). When exporting to S3, the token exchange role must grant `s3:PutObject` on the target bucket. When exporting to Kinesis, the role must grant `kinesis:PutRecord` and `kinesis:PutRecords` on the target stream.

**Model Optimization:** TFLite conversion requires TensorFlow 2.14+ installed on the build machine (not on the edge device). ONNX quantization requires `onnxruntime>=1.16`. PyTorch Mobile optimization requires `torch>=2.0`. Optimization scripts run in the cloud or on a build server — only the optimized model is deployed to edge devices.

**Resource Limits:** Set `LinuxProcessParams.ResourceLimits` in the recipe to prevent ML inference from consuming all device resources. Typical values: 512 MB memory and 1.0 CPU for image classification, 1024 MB and 2.0 CPU for object detection. Monitor actual usage and adjust. Exceeding memory limits causes the component to be terminated by the nucleus.

**Offline Operation:** Edge devices may lose network connectivity. The offline fallback pattern queues results locally in SQLite (max 10,000 records by default). When connectivity is restored, results are drained in chronological order. Design inference logic to be fully functional without network access — the model and all dependencies must be on-device.

**Deployment Rollback:** Set `failureHandlingPolicy` to `ROLLBACK` in production to automatically revert to the previous component version if deployment fails on a device. Use `componentUpdatePolicy` with `NOTIFY_COMPONENTS` to allow the running component to gracefully shut down before update.

**Security:** Use the Greengrass token exchange service for temporary AWS credentials — never embed long-lived credentials in component code. Encrypt model artifacts at rest in S3 using KMS keys from `devops/08`. Component code runs as `ggc_user:ggc_group` with restricted filesystem access. Restrict IoT policies to minimum required actions.

**Cost:** Greengrass V2 has no per-device charge for custom components. Costs come from: S3 storage for component artifacts and model files, S3 PUT requests from Stream Manager exports, Kinesis shard hours and PUT record charges, and CloudWatch custom metric charges. Monitor Stream Manager export volume to control costs in large fleets.

**Naming:** All resources follow `{PROJECT_NAME}-{component}-{ENV}` convention:
- Component: `{PROJECT_NAME}-{COMPONENT_NAME}-{ENV}`
- Deployment: `{PROJECT_NAME}-{COMPONENT_NAME}-{ENV}-{timestamp}`
- Stream: `{PROJECT_NAME}-inference-results`
- Dashboard: `{PROJECT_NAME}-greengrass-ml-{ENV}`
- S3 prefix: `{PROJECT_NAME}/{ENV}/results/`

---

## Integration Points

- **Upstream**: `edge/01` → Neo-compiled and edge-packaged model artifacts (MODEL_S3_URI) from SageMaker Edge Manager pipeline
- **Upstream**: `devops/04` → IAM roles for Greengrass token exchange role, S3 access, and Kinesis write permissions
- **Downstream**: `data/02` → Kinesis Data Stream receives real-time inference results via Stream Manager for downstream feature computation
- **Downstream**: `devops/03` → CloudWatch dashboards consume custom metrics (inference latency, throughput, device health) published by the component
- **Downstream**: `devops/11` → Custom CloudWatch model quality metrics from edge inference results
- **Alternative to**: `edge/01` → For devices that run Greengrass V2 instead of the standalone SageMaker Edge Manager agent
