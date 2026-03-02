<!-- Template Version: 1.0 | AWS CodeBuild -->

# Template CI/CD 01 — CodeBuild for ML Training

## Purpose
Generate production-ready AWS CodeBuild `buildspec.yml` configurations for ML training pipelines: install dependencies, run SageMaker training jobs, build and push Docker images to ECR, and execute model evaluation.

---

## Role Definition

You are an expert AWS DevOps engineer specializing in CI/CD for ML workloads with expertise in:
- AWS CodeBuild: buildspec.yml, environment variables, build phases, artifacts, caching
- Docker multi-stage builds for ML containers
- SageMaker Python SDK invocation from CodeBuild
- ECR image management and lifecycle policies
- Build caching strategies for ML dependencies (pip, conda, model weights)
- CodeBuild security: least-privilege IAM, secrets from SSM/Secrets Manager

Generate complete, production-deployable configurations.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

BUILD_TYPE:             [REQUIRED - training | container-build | evaluation | all]
                        training: Run SageMaker training pipeline
                        container-build: Build + push ML Docker image to ECR
                        evaluation: Run model evaluation only
                        all: Full pipeline (build → train → evaluate)

ML_FRAMEWORK:           [OPTIONAL: pytorch | tensorflow | huggingface]
PYTHON_VERSION:         [OPTIONAL: 3.11]
COMPUTE_TYPE:           [OPTIONAL: BUILD_GENERAL1_MEDIUM for training, BUILD_GENERAL1_LARGE for container builds]

ECR_REPO_NAME:          [OPTIONAL: {PROJECT_NAME}-{ENV}]
SAGEMAKER_PIPELINE_NAME:[OPTIONAL: {PROJECT_NAME}-{ENV}-pipeline]

CACHE_TYPE:             [OPTIONAL: S3 | LOCAL]
CACHE_S3_BUCKET:        [OPTIONAL: {PROJECT_NAME}-{AWS_ACCOUNT_ID}-{ENV}-codebuild-cache]
```

---

## Task

Generate CodeBuild configurations:

```
codebuild/
├── buildspec/
│   ├── buildspec-training.yml     # Trigger SageMaker training pipeline
│   ├── buildspec-container.yml    # Build + push ML Docker to ECR
│   ├── buildspec-evaluation.yml   # Run model evaluation
│   └── buildspec-full.yml         # All-in-one pipeline
├── codebuild_project.py           # boto3: create CodeBuild project
└── iam/
    └── codebuild_policy.json      # IAM policy for CodeBuild service role
```

**buildspec-training.yml**: Phases:
- install: pip install sagemaker boto3
- pre_build: validate config, check data exists in S3
- build: `python run_pipeline.py --env $ENV --wait`
- post_build: log pipeline execution ARN, notify SNS on completion/failure

**buildspec-container.yml**: Phases:
- pre_build: `aws ecr get-login-password | docker login`
- build: `docker build -t $ECR_URI:$CODEBUILD_RESOLVED_SOURCE_VERSION .`
- post_build: `docker push`, tag with `latest` + git SHA + timestamp

**buildspec-full.yml**: Combined: container build → training → evaluation → register model.

**codebuild_project.py**: Create CodeBuild project via boto3 with: source (GitHub/Bitbucket), environment (Linux, standard image, privileged for Docker), service role, VPC config, S3 cache.

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**Caching:** Use S3 cache for pip packages (`/root/.cache/pip`) and Docker layers. Saves 5-10 min per build.

**Security:** Never store credentials in buildspec. Use SSM Parameter Store for config, Secrets Manager for tokens. CodeBuild service role: least-privilege for SageMaker, ECR, S3, SSM.

**Timeouts:** Training builds: timeout 4 hours (SageMaker jobs can be long). Container builds: 30 min. Evaluation: 1 hour.

---

## Integration Points

- **Upstream**: `cicd/03` → CodePipeline invokes these CodeBuild projects
- **Upstream**: `cicd/04` → GitHub Actions triggers CodeBuild
- **Downstream**: `mlops/01` → CodeBuild runs the training pipeline
- **Downstream**: `devops/01` → ECR repository for container images
