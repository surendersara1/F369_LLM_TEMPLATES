<!-- Template Version: 1.0 | Docker: 24+ | boto3: 1.35+ -->

# Template DevOps 01 — ECR & ML Docker Containers

## Purpose
Generate production-ready multi-stage Dockerfiles for ML training and inference containers, Amazon ECR repository setup with lifecycle policies, image scanning configuration, and container build automation.

---

## Role Definition

You are an expert DevOps engineer specializing in containerized ML workloads with expertise in:
- Multi-stage Docker builds for minimal image size
- NVIDIA CUDA base images for GPU workloads
- SageMaker compatible containers (train and serve protocols)
- Amazon ECR: repository management, lifecycle policies, image scanning
- Docker layer caching optimization for ML dependencies
- Security: non-root users, vulnerability scanning, minimal base images

Generate complete, production-deployable configurations.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

CONTAINER_TYPE:         [REQUIRED - training | inference | both]
ML_FRAMEWORK:           [OPTIONAL: pytorch | tensorflow | huggingface]
CUDA_VERSION:           [OPTIONAL: 12.1 for GPU, none for CPU-only]
PYTHON_VERSION:         [OPTIONAL: 3.11]

BASE_IMAGE:             [OPTIONAL - custom base or use AWS DLC]
                        Options:
                        - nvidia/cuda:12.1.0-runtime-ubuntu22.04 (GPU custom)
                        - python:3.11-slim (CPU custom)
                        - AWS DLC: auto-selected based on ML_FRAMEWORK

SERVING_FRAMEWORK:      [OPTIONAL: vllm | tgi | fastapi | flask]
ECR_REPO_NAME:          [OPTIONAL: {PROJECT_NAME}-{ENV}]
LIFECYCLE_MAX_IMAGES:   [OPTIONAL: 10 for dev, 30 for prod]
ENABLE_SCANNING:        [OPTIONAL: true]
```

---

## Task

Generate Docker and ECR configuration:

```
containers/
├── training/
│   ├── Dockerfile                 # Multi-stage training container
│   ├── requirements.txt
│   └── .dockerignore
├── inference/
│   ├── Dockerfile                 # Multi-stage inference container
│   ├── requirements.txt
│   └── .dockerignore
├── ecr/
│   ├── create_repositories.py     # Create ECR repos with policies
│   ├── lifecycle_policy.json      # ECR lifecycle policy
│   └── build_push.sh              # Build + tag + push to ECR
└── scripts/
    └── scan_results.py            # Check ECR scan results, fail on CRITICAL
```

**training/Dockerfile**: Multi-stage build:
- Stage 1 (builder): install all dependencies, compile extensions
- Stage 2 (runtime): copy only needed artifacts, non-root user, SageMaker entry point compatibility (`/opt/ml/` paths)

**inference/Dockerfile**: Multi-stage with serving framework:
- vLLM: `FROM vllm/vllm-openai:latest`
- TGI: `FROM ghcr.io/huggingface/text-generation-inference:latest`
- FastAPI: custom with `uvicorn`, health check endpoint
- SageMaker compatible: expose `/ping` and `/invocations`

**create_repositories.py**: boto3 `ecr.create_repository()` with image scanning enabled, KMS encryption, lifecycle policy attachment.

**lifecycle_policy.json**: Keep last N tagged images, expire untagged after 1 day, keep `latest` and `prod-*` tags permanently.

**build_push.sh**: Complete build automation:
```bash
aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_URI
docker build -t $REPO:$GIT_SHA -t $REPO:latest .
docker push $REPO:$GIT_SHA && docker push $REPO:latest
```

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**Image Size:** Training images < 5GB, inference images < 3GB. Use multi-stage builds. Don't include training data in images. Use `.dockerignore` aggressively.

**Security:** Non-root user in final stage. No secrets in Dockerfile (use runtime env vars or Secrets Manager). Enable ECR image scanning, fail CI on CRITICAL vulnerabilities.

**SageMaker Compatibility:** Training container must accept `/opt/ml/` paths. Inference must expose `/ping` (GET) and `/invocations` (POST). Exit code 0 on success.

---

## Integration Points

- **Downstream**: `mlops/01` → training container for SageMaker jobs
- **Downstream**: `mlops/03` → inference container for endpoints
- **Downstream**: `iac/04` → ECS task definition references ECR images
- **Downstream**: `cicd/01` → CodeBuild builds and pushes containers
- **Downstream**: `cicd/04` → GitHub Actions builds and pushes containers
