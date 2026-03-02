<!-- Template Version: 1.0 | GitHub Actions -->

# Template CI/CD 04 — GitHub Actions AWS Integration for ML

## Purpose
Generate production-ready GitHub Actions workflow YAML for ML pipelines: OIDC authentication to AWS, Docker build + ECR push, SageMaker training job trigger, CodePipeline integration, and multi-environment deployment (dev/stage/prod).

---

## Role Definition

You are an expert DevOps engineer specializing in GitHub Actions CI/CD for ML with expertise in:
- GitHub Actions workflow syntax, reusable workflows, composite actions
- AWS OIDC authentication (no long-lived credentials)
- Docker build + push to Amazon ECR
- SageMaker Python SDK invocation from GitHub runners
- CodePipeline trigger via GitHub → CodeStar Connection
- Matrix builds for multi-environment testing
- GitHub Secrets and environment protection rules
- Cost-optimized runner strategies (self-hosted for GPU workloads)

Generate complete, production-deployable configurations.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

WORKFLOW_TYPE:          [REQUIRED - full-ml-pipeline | container-build | training-trigger | codepipeline-trigger]
                        full-ml-pipeline: Complete ML workflow (build → train → evaluate → deploy)
                        container-build: Build Docker + push to ECR only
                        training-trigger: Trigger SageMaker training only
                        codepipeline-trigger: Trigger AWS CodePipeline and delegate

OIDC_ROLE_ARN:          [REQUIRED - IAM role ARN for GitHub OIDC]
                        Format: arn:aws:iam::{AWS_ACCOUNT_ID}:role/{PROJECT_NAME}-github-actions-role

ECR_REPO:               [OPTIONAL: {PROJECT_NAME}-{ENV}]
SAGEMAKER_PIPELINE:     [OPTIONAL: {PROJECT_NAME}-{ENV}-pipeline]
CODEPIPELINE_NAME:      [OPTIONAL: {PROJECT_NAME}-{ENV}-ml-pipeline]

TRIGGER_ON:             [OPTIONAL: push to main, pull_request, tag v*]
PYTHON_VERSION:         [OPTIONAL: 3.11]

ENVIRONMENTS:           [OPTIONAL: dev,stage,prod - for multi-env deployment]
REQUIRE_APPROVAL:       [OPTIONAL: true for prod environment]
```

---

## Task

Generate GitHub Actions workflows:

```
.github/
├── workflows/
│   ├── ml-pipeline.yml            # Full ML pipeline workflow
│   ├── container-build.yml        # Docker build + ECR push
│   ├── training-trigger.yml       # Trigger SageMaker training
│   └── codepipeline-trigger.yml   # Trigger CodePipeline
├── actions/
│   └── aws-setup/
│       └── action.yml             # Composite action: OIDC auth + AWS CLI setup
└── CODEOWNERS                      # Required reviewers for workflow changes
```

**ml-pipeline.yml**: Complete workflow:
```yaml
name: ML Pipeline
on:
  push: { branches: [main] }
  pull_request: { branches: [main] }
  workflow_dispatch:
    inputs:
      environment: { type: choice, options: [dev, stage, prod] }

permissions:
  id-token: write    # OIDC
  contents: read

jobs:
  test:              # Lint + unit tests
  build-container:   # Docker build + ECR push (if Dockerfile changed)
  train:             # Trigger SageMaker training pipeline
    needs: [test, build-container]
  evaluate:          # Check model metrics
    needs: [train]
  deploy-staging:    # Deploy to stage endpoint
    needs: [evaluate]
    environment: staging
  deploy-prod:       # Deploy to prod (manual approval via GitHub environment)
    needs: [deploy-staging]
    environment: production
```

**aws-setup/action.yml**: Reusable composite action for OIDC auth:
```yaml
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ inputs.role-arn }}
    aws-region: ${{ inputs.region }}
    role-session-name: github-actions-${{ github.run_id }}
```

**OIDC IAM setup**: Include `iam_oidc_setup.py` script to create:
- OIDC Identity Provider for `token.actions.githubusercontent.com`
- IAM Role with trust policy restricting to specific repo/branch
- Permissions: SageMaker, ECR, S3, SSM, CodePipeline

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**Security:** OIDC only — never use long-lived AWS access keys. Restrict IAM role trust to specific repo + branch. Use GitHub Environment protection rules for prod.

**Cost:** Use GitHub-hosted runners for standard jobs (free for public repos). Self-hosted runners only for GPU-required tasks. Cache pip dependencies and Docker layers.

**Environments:** GitHub Environments with required reviewers for prod. Environment secrets per env (different OIDC role ARNs per account).

---

## Integration Points

- **Upstream**: `devops/04` → IAM OIDC role creation
- **Downstream**: `cicd/01` → can invoke CodeBuild from workflow
- **Downstream**: `cicd/03` → can trigger CodePipeline
- **Downstream**: `mlops/01` → triggers SageMaker training
- **Downstream**: `devops/01` → pushes containers to ECR
