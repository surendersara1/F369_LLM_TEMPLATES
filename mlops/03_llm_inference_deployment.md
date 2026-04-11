<!-- Template Version: 1.1 | boto3: 1.35+ | sagemaker-python-sdk: 2.230+ -->

# Template 03 — LLM Inference Deployment

## Purpose
Generate production-ready LLM inference infrastructure on AWS: SageMaker Real-time Endpoint (managed, auto-scaling) OR SageMaker Inference Components (multi-model packing on shared endpoints) OR ECS with vLLM/TGI (self-managed, maximum control). Includes blue/green deployment, auto-scaling, health monitoring, and cost optimization.

---

## Role Definition

You are an expert AWS MLOps engineer specializing in LLM serving infrastructure with expertise in:
- SageMaker Real-time Endpoints with custom containers
- **SageMaker Inference Components**: multi-model packing on shared endpoints, per-model auto-scaling, resource isolation (NEW 2025)
- Amazon ECS with GPU support (EC2 launch type) for vLLM/TGI
- vLLM and HuggingFace Text Generation Inference (TGI) serving frameworks
- Blue/green and canary deployment strategies
- Application Load Balancer, CloudFront, API Gateway for LLM APIs
- Auto-scaling policies for inference workloads (target tracking, step scaling)
- AWS SageMaker Inference Recommender for instance type selection
- AWS Inferentia2 / Trainium for cost-optimized inference

Generate complete, production-deployable code with actual implementations.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED - e.g. legal-assistant-api]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

DEPLOYMENT_TYPE:        [REQUIRED - sagemaker | inference-components | ecs-vllm | ecs-tgi]
                        sagemaker: SageMaker Real-time Endpoint (fully managed, single model)
                        inference-components: SageMaker Inference Components (multi-model, shared endpoint)
                        ecs-vllm: ECS EC2 (GPU) with vLLM (highest throughput)
                        ecs-tgi: ECS EC2 (GPU) with HF TGI (Hugging Face native)

MODEL_ARTIFACT_S3_PATH: [REQUIRED - s3://bucket/path/to/model/]
                        Can be: merged full model, LoRA adapter, or Bedrock model ARN

INFERENCE_INSTANCE_TYPE:[OPTIONAL: ml.g5.2xlarge for sagemaker]
                        SageMaker: ml.g4dn.xlarge, ml.g5.2xlarge, ml.inf2.xlarge
                        ECS EC2: g4dn.xlarge, g5.2xlarge, p3.2xlarge

MIN_INSTANCES:          [OPTIONAL: 0 for dev (scale-to-zero), 1 for stage, 2 for prod]
MAX_INSTANCES:          [OPTIONAL: 1 for dev, 3 for stage, 10 for prod]

MAX_CONCURRENT_REQUESTS:[OPTIONAL: 8]
MAX_INPUT_TOKENS:       [OPTIONAL: 4096]
MAX_OUTPUT_TOKENS:      [OPTIONAL: 1024]
QUANTIZATION:           [OPTIONAL: none | awq | gptq | bitsandbytes]
                        AWQ recommended for production (best speed/quality tradeoff)

DEPLOYMENT_STRATEGY:    [OPTIONAL: blue-green | canary | rolling]
CANARY_TRAFFIC_PERCENT: [OPTIONAL: 10]

API_KEY_REQUIRED:       [OPTIONAL: true]
CORS_ORIGINS:           [OPTIONAL: * for dev, specific domains for prod]

ENDPOINT_NAME:          [OPTIONAL: {PROJECT_NAME}-{ENV}-endpoint]

# Inference Components-specific (only if DEPLOYMENT_TYPE = inference-components)
MODELS:                 [OPTIONAL - comma-separated model names to pack on shared endpoint]
                        e.g. summarizer-v1,classifier-v2,embedder-v1
COMPUTE_PER_MODEL:      [OPTIONAL - GPU fractions per model, e.g. 0.5,0.25,0.25]
MIN_COPIES_PER_MODEL:   [OPTIONAL - min model copies, e.g. 1,1,0]
MAX_COPIES_PER_MODEL:   [OPTIONAL - max model copies, e.g. 4,2,2]
```

---

## Task

### OPTION A: DEPLOYMENT_TYPE = sagemaker

```
{PROJECT_NAME}-inference/
├── endpoint/
│   ├── model_config.py            # SageMaker Model creation (container, model data)
│   ├── endpoint_config.py         # EndpointConfig with production variants
│   ├── deploy_endpoint.py         # Create/update endpoint with blue/green
│   ├── autoscaling.py             # Target tracking auto-scaling policies
│   └── health_check.py            # Endpoint health monitoring + CloudWatch
├── container/
│   ├── Dockerfile                 # Custom inference container (if needed)
│   ├── serve.py                   # Model server entry point
│   └── requirements.txt
├── client/
│   ├── inference_client.py        # Python client (OpenAI-compatible wrapper)
│   └── load_test.py               # Locust load testing script
├── config/
│   └── inference_config.py
└── deploy.py                      # CLI: deploy model to endpoint
```

**model_config.py**: Create SageMaker Model from S3 artifact + DLC container image (HuggingFace LLM DLC, PyTorch DLC). Set environment variables: `HF_MODEL_ID`, `SM_NUM_GPUS`, `MAX_INPUT_LENGTH`, `MAX_TOTAL_TOKENS`.

**endpoint_config.py**: ProductionVariant with instance type, initial instance count, volume size, model data download timeout. For blue/green: two variants with traffic splitting.

**deploy_endpoint.py**: Create or update endpoint. Blue/green deployment: create new variant → shift traffic (10% → 50% → 100%) → remove old variant. Rollback on CloudWatch alarm.

**autoscaling.py**: Application Auto Scaling target tracking on `SageMakerVariantInvocationsPerInstance`. Scale-to-zero for dev (using scheduled scaling). Step scaling for burst traffic.

### OPTION B: DEPLOYMENT_TYPE = inference-components

```
{PROJECT_NAME}-inference-components/
├── endpoint/
│   ├── shared_endpoint.py          # Create shared endpoint (compute pool)
│   ├── inference_component.py      # Create/update inference components per model
│   ├── component_autoscaling.py    # Per-component auto-scaling (independent)
│   ├── traffic_router.py           # Route requests to specific components
│   └── component_lifecycle.py      # Add/remove/update models without downtime
├── monitoring/
│   ├── component_metrics.py        # Per-component CloudWatch metrics
│   └── cost_allocation.py          # Per-model cost tracking via tags
├── client/
│   ├── multi_model_client.py       # Client that targets specific inference components
│   └── load_test.py                # Per-component load testing
├── config/
│   └── components_config.py
└── deploy.py                       # CLI: manage inference components
```

**shared_endpoint.py**: Create a single SageMaker endpoint with managed instance scaling. The endpoint acts as a shared compute pool. Multiple models share GPU resources.

**inference_component.py**: Create InferenceComponent per model, specifying: model artifact, container, compute resource requirements (NumberOfCpuCoresRequired, MinMemoryRequiredInMb, NumberOfAcceleratorDevicesRequired), copy count. Models are packed onto shared endpoint instances.

**component_autoscaling.py**: Independent auto-scaling per inference component. Scale each model's copy count based on its own invocation metrics. Scale-to-zero for low-traffic models while keeping the endpoint warm.

**component_lifecycle.py**: Add new models, update existing models, remove models — all without endpoint downtime. Rolling update for model version changes.

### OPTION C: DEPLOYMENT_TYPE = ecs-vllm

```
{PROJECT_NAME}-ecs-inference/
├── infrastructure/
│   ├── ecs_cluster.py              # ECS cluster with GPU EC2 instances
│   ├── task_definition.py          # Task def with vLLM/TGI container
│   ├── service.py                  # ECS service with ALB
│   ├── alb.py                      # Application Load Balancer + HTTPS
│   └── autoscaling.py              # ECS service auto-scaling
├── container/
│   ├── Dockerfile.vllm             # vLLM serving container
│   ├── Dockerfile.tgi              # TGI serving container
│   └── entrypoint.sh               # Container startup script
├── client/
│   ├── inference_client.py         # OpenAI-compatible client
│   └── load_test.py                # Locust load testing script
├── monitoring/
│   ├── cloudwatch_dashboard.py     # GPU utilization, latency, throughput
│   └── health_check.py             # ALB health check + alarms
├── config/
│   └── ecs_config.py
└── deploy.py                       # CLI: deploy ECS service
```

**ecs_cluster.py**: ECS cluster with EC2 capacity provider (GPU instances). Auto Scaling Group for GPU instance pool. Spot instances for dev, On-Demand for prod.

**task_definition.py**: Task definition with vLLM container, GPU resource reservation, environment variables (`--model`, `--tensor-parallel-size`, `--max-model-len`, `--quantization`). Health check command.

**service.py**: ECS service with desired count, ALB target group, circuit breaker, deployment configuration (min/max healthy percent).

**Dockerfile.vllm**: Based on `vllm/vllm-openai:latest`. Copy model weights or mount from EFS. Expose OpenAI-compatible API on port 8000.

---

## Output Format

Output ALL files for chosen DEPLOYMENT_TYPE with headers: `### FILE: [path]`

---

## Requirements & Constraints

**SageMaker Endpoint:** Use HuggingFace LLM DLC for most models. Set `ModelDataDownloadTimeoutInSeconds=1200` for large models. Enable server-side encryption. Use VPC mode for prod.

**Inference Components (NEW):** Shared endpoint reduces cost by 40-60% for multi-model scenarios. Each component scales independently (0 to N copies). Component update is zero-downtime. Max 15 inference components per endpoint. Use `NumberOfAcceleratorDevicesRequired` for GPU fraction allocation.

**ECS (vLLM/TGI):** vLLM provides highest throughput via continuous batching + PagedAttention. TGI is best for HuggingFace-native models. Always use `--tensor-parallel-size` matching GPU count. Enable CUDA MPS for multi-model sharing.

**All options:** Enable CloudWatch metrics (latency p50/p95/p99, throughput, GPU utilization, error rate). Set up health checks with 30s intervals. Implement request timeouts (60s default, 120s for long generation). Add API Gateway or ALB for rate limiting.

**Cost optimization:** Use Inferentia2 (`ml.inf2.*`) for 40% lower inference cost where supported. Use spot instances in dev. Scale-to-zero in dev environments. Use Savings Plans for prod steady-state.

---

## Code Scaffolding Hints

**SageMaker Endpoint:**
```python
from sagemaker.huggingface import HuggingFaceModel
# NOTE: Verify latest available DLC versions via SageMaker DLC release notes
# or `aws ecr describe-images --repository-name huggingface-pytorch-inference`
model = HuggingFaceModel(
    model_data=MODEL_ARTIFACT_S3_PATH,
    role=role_arn,
    transformers_version="4.51",
    pytorch_version="2.6",
    py_version="py312",
    env={"HF_MODEL_ID": "/opt/ml/model", "SM_NUM_GPUS": "1",
         "MAX_INPUT_LENGTH": str(MAX_INPUT_TOKENS), "MAX_TOTAL_TOKENS": "8192"}
)
predictor = model.deploy(
    initial_instance_count=MIN_INSTANCES,
    instance_type=INFERENCE_INSTANCE_TYPE,
    endpoint_name=ENDPOINT_NAME
)
```

**Inference Components:**
```python
# Step 1: Create endpoint config (shared compute pool — no model references)
# ManagedInstanceScaling and RoutingConfig are set at the ProductionVariant level
# within CreateEndpointConfig. The endpoint acts as a compute pool for inference components.
sm_client.create_endpoint_config(
    EndpointConfigName=f"{PROJECT_NAME}-{ENV}-shared-config",
    ExecutionRoleArn=role_arn,
    ProductionVariants=[{
        "VariantName": "shared-compute",
        "InstanceType": INFERENCE_INSTANCE_TYPE,
        "InitialInstanceCount": MIN_INSTANCES,
        "ManagedInstanceScaling": {
            "Status": "ENABLED",
            "MinInstanceCount": MIN_INSTANCES,
            "MaxInstanceCount": MAX_INSTANCES
        },
        "RoutingConfig": {"RoutingStrategy": "LEAST_OUTSTANDING_REQUESTS"}
    }]
)

# Step 2: Create endpoint from config
sm_client.create_endpoint(
    EndpointName=f"{PROJECT_NAME}-{ENV}-endpoint",
    EndpointConfigName=f"{PROJECT_NAME}-{ENV}-shared-config"
)

# Step 3: Create inference component per model (separate from endpoint config)
# Each component specifies its own model, container, and resource requirements.
sm_client.create_inference_component(
    InferenceComponentName=f"{PROJECT_NAME}-{model_name}-{ENV}",
    EndpointName=endpoint_name,
    VariantName="shared-compute",
    Specification={
        "Container": {
            "Image": container_image,
            "ArtifactUrl": model_s3_path
        },
        "ComputeResourceRequirements": {
            "NumberOfCpuCoresRequired": 2,
            "MinMemoryRequiredInMb": 4096,
            "NumberOfAcceleratorDevicesRequired": 1  # GPU fraction
        }
    },
    RuntimeConfig={"CopyCount": 1}
)

# Step 4: Auto-scale individual component
aas_client.register_scalable_target(
    ServiceNamespace="sagemaker",
    ResourceId=f"inference-component/{component_name}",
    ScalableDimension="sagemaker:inference-component:DesiredCopyCount",
    MinCapacity=0, MaxCapacity=4
)
```

**ECS + vLLM:**
```python
# Task definition
task_def = {
    "containerDefinitions": [{
        "name": "vllm-server",
        "image": f"{AWS_ACCOUNT_ID}.dkr.ecr.{AWS_REGION}.amazonaws.com/{PROJECT_NAME}-vllm:latest",
        "command": ["--model", "/models/llm", "--tensor-parallel-size", "1",
                    "--max-model-len", str(MAX_INPUT_TOKENS + MAX_OUTPUT_TOKENS),
                    "--quantization", QUANTIZATION],
        "portMappings": [{"containerPort": 8000, "protocol": "tcp"}],
        "resourceRequirements": [{"type": "GPU", "value": "1"}],
        "healthCheck": {"command": ["CMD-SHELL", "curl -f http://localhost:8000/health || exit 1"]}
    }]
}
```

---

## Integration Points

- **Upstream**: `mlops/10` → deploy approved model from registry
- **Upstream**: `mlops/02` → deploy fine-tuned model
- **Upstream**: `devops/01` → container images from ECR
- **Downstream**: `mlops/05` → monitor deployed endpoint
- **Downstream**: `mlops/04` → RAG pipeline calls inference endpoint
- **Downstream**: `cicd/02` → CodeDeploy for blue/green endpoint updates
- **Downstream**: `mlops/11` → governance tracks deployed models
