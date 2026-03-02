<!-- Template Version: 1.0 | AWS CDK: 2.170+ | Python -->

# Template IaC 04 — CDK Python: ECS LLM Inference with vLLM/TGI

## Purpose
Generate a production-ready CDK Python stack for LLM inference on ECS: Fargate or EC2 (GPU) cluster, Application Load Balancer, vLLM or TGI task definition, ECR repository, auto-scaling, CloudWatch monitoring, and API Gateway integration.

---

## Role Definition

You are an expert AWS infrastructure engineer specializing in containerized LLM serving with CDK expertise in:
- ECS with GPU support (EC2 launch type with g4dn/g5 instances)
- ECS Fargate for CPU-based inference or small models
- Application Load Balancer with health checks and HTTPS
- vLLM and HuggingFace TGI container configurations
- ECS Service Auto Scaling (target tracking on CPU/custom metrics)
- GPU instance management: capacity providers, placement strategies
- CDK constructs for ECS patterns (ApplicationLoadBalancedEc2Service, etc.)

Generate complete, production-deployable CDK application.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

SERVING_FRAMEWORK:      [REQUIRED - vllm | tgi]
LAUNCH_TYPE:            [REQUIRED - ec2-gpu | fargate]
                        ec2-gpu: GPU instances (g4dn, g5) for large models
                        fargate: CPU only, for small models or embedding serving

GPU_INSTANCE_TYPE:      [OPTIONAL: g5.2xlarge - for ec2-gpu launch type]
MODEL_S3_PATH:          [OPTIONAL - s3://bucket/path/to/model/ for pre-downloaded models]
MODEL_ID:               [OPTIONAL - HuggingFace model ID if downloading at runtime]

CONTAINER_CPU:          [OPTIONAL: 16384 (16 vCPU) for GPU instance]
CONTAINER_MEMORY:       [OPTIONAL: 65536 (64GB) for g5.2xlarge]
CONTAINER_GPU:          [OPTIONAL: 1]
CONTAINER_PORT:         [OPTIONAL: 8080]

MIN_TASKS:              [OPTIONAL: 0 for dev, 1 for stage, 2 for prod]
MAX_TASKS:              [OPTIONAL: 1 for dev, 3 for stage, 10 for prod]

ENABLE_HTTPS:           [OPTIONAL: true]
ACM_CERT_ARN:           [OPTIONAL - ACM certificate ARN for HTTPS]
DOMAIN_NAME:            [OPTIONAL - e.g. llm-api.example.com]

API_KEY_ENABLED:        [OPTIONAL: true]
WAF_ENABLED:            [OPTIONAL: false for dev, true for prod]
```

---

## Task

Generate CDK application for ECS LLM inference:

```
cdk_ecs_llm/
├── app.py
├── cdk.json
├── requirements.txt
├── stacks/
│   ├── vpc_stack.py               # VPC (or import existing)
│   ├── ecr_stack.py               # ECR repository + lifecycle policy
│   ├── ecs_cluster_stack.py       # ECS Cluster + capacity provider
│   ├── llm_service_stack.py       # ECS Service + task def + ALB
│   ├── autoscaling_stack.py       # Service auto-scaling
│   └── monitoring_stack.py        # CloudWatch dashboard + alarms
├── constructs/
│   ├── vllm_task_definition.py    # Custom construct: vLLM task def
│   └── tgi_task_definition.py     # Custom construct: TGI task def
├── config/
│   └── environments.py            # Per-env settings
├── docker/
│   ├── Dockerfile.vllm            # vLLM container with model download
│   └── Dockerfile.tgi             # TGI container with model download
└── tests/
    └── test_stacks.py
```

**stacks/ecs_cluster_stack.py**: ECS Cluster with EC2 capacity provider (GPU instances), auto-scaling group with GPU AMI (`/aws/service/ecs/optimized-ami/amazon-linux-2/gpu/recommended`), placement strategy `binpack` on GPU.

**stacks/llm_service_stack.py**: ECS Service with:
- Task definition: vLLM or TGI container with GPU resource reservation
- ALB: health check on `/health`, target group with slow_start (120s for model loading)
- Environment variables: MODEL_ID, MAX_MODEL_LEN, GPU_MEMORY_UTILIZATION, QUANTIZATION
- Secrets from Secrets Manager (HF_TOKEN)
- CloudWatch log group

**constructs/vllm_task_definition.py**: Custom L3 construct:
```python
container = task_def.add_container("vllm",
    image=ecs.ContainerImage.from_ecr_repository(ecr_repo, tag),
    gpu_count=1,
    memory_limit_mib=65536,
    environment={
        "MODEL_ID": model_id,
        "MAX_MODEL_LEN": "4096",
        "GPU_MEMORY_UTILIZATION": "0.90",
        "QUANTIZATION": "awq"
    },
    logging=ecs.LogDrivers.aws_logs(stream_prefix="vllm"),
    health_check=ecs.HealthCheck(command=["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"])
)
```

**docker/Dockerfile.vllm**:
```dockerfile
FROM vllm/vllm-openai:latest
COPY scripts/download_model.py /app/
RUN python /app/download_model.py  # Pre-download model for faster cold start
EXPOSE 8080
CMD ["python", "-m", "vllm.entrypoints.openai.api_server", ...]
```

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**GPU:** ECS EC2 with GPU requires: ECS-optimized GPU AMI, `resourceRequirements` in task def with `GPU=1`, capacity provider with managed scaling.

**Cold Start:** Model loading takes 2-5 min. Use ALB `slow_start` duration = 300s. Pre-download model into Docker image for faster startup. Health check grace period = 300s.

**Cost:** g5.2xlarge on-demand: ~$1.21/hr. Spot: ~$0.36/hr. Use spot for dev. Scale to zero for dev (min_tasks=0, scale up on first request via ALB).

**Security:** No public IP on ECS tasks. ALB in public subnet, tasks in private. WAF on ALB for rate limiting in prod. API key validation via Lambda@Edge or ALB listener rules.

---

## Integration Points

- **Upstream**: `mlops/02` or `mlops/09` → fine-tuned model to serve
- **Upstream**: `devops/01` → ECR for container images
- **Upstream**: `devops/02` → VPC networking
- **Alternative to**: `mlops/03` SageMaker endpoint approach
- **Downstream**: `devops/03` → CloudWatch monitoring
- **Downstream**: `cicd/02` → CodeDeploy for blue/green updates
