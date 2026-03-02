<!-- Template Version: 1.0 | Bitbucket Pipelines -->

# Template CI/CD 05 — Bitbucket Pipelines AWS Integration for ML

## Purpose
Generate production-ready Bitbucket Pipelines configuration for ML workflows: OIDC/access key authentication to AWS, Docker build + ECR push, SageMaker training triggers, CodePipeline integration via CodeStar Connections, and multi-environment deployment with Jira integration.

---

## Role Definition

You are an expert DevOps engineer specializing in Bitbucket Pipelines CI/CD for ML with expertise in:
- Bitbucket Pipelines: `bitbucket-pipelines.yml`, step types, pipes, caches, services
- AWS OIDC authentication with Bitbucket (identity provider setup)
- Docker build + push to Amazon ECR using Bitbucket pipes
- CodeStar Connection for AWS CodePipeline integration
- Bitbucket Deployment environments and deployment permissions
- Jira integration for deployment tracking
- Atlassian ecosystem integration (Confluence, Jira, Bitbucket)

Generate complete, production-deployable configurations.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

WORKFLOW_TYPE:          [REQUIRED - full-ml-pipeline | container-build | codepipeline-trigger]
AUTH_METHOD:            [OPTIONAL: oidc | access-keys]
                        oidc: Bitbucket OIDC provider (recommended)
                        access-keys: Repository variables AWS_ACCESS_KEY_ID/SECRET (legacy)

OIDC_ROLE_ARN:          [REQUIRED if oidc - IAM role ARN for Bitbucket OIDC]
ECR_REPO:               [OPTIONAL: {PROJECT_NAME}-{ENV}]
CODEPIPELINE_NAME:      [OPTIONAL: {PROJECT_NAME}-{ENV}-ml-pipeline]
SAGEMAKER_PIPELINE:     [OPTIONAL: {PROJECT_NAME}-{ENV}-pipeline]

TRIGGER_BRANCHES:       [OPTIONAL: main for prod, develop for stage, feature/* for dev]
JIRA_PROJECT_KEY:       [OPTIONAL - for deployment tracking]
```

---

## Task

Generate Bitbucket Pipelines configuration:

```
project_root/
├── bitbucket-pipelines.yml        # Main pipeline configuration
├── scripts/
│   ├── aws-auth.sh                # OIDC authentication helper
│   ├── trigger-training.sh        # Trigger SageMaker training
│   ├── build-push-ecr.sh          # Build Docker + push to ECR
│   └── trigger-codepipeline.sh    # Start CodePipeline execution
└── deployment/
    └── iam_bitbucket_oidc.py      # Create IAM OIDC provider for Bitbucket
```

**bitbucket-pipelines.yml**:
```yaml
image: python:3.11

definitions:
  caches:
    pip: ~/.cache/pip
  steps:
    - step: &test
        name: Test & Lint
        caches: [pip]
        script:
          - pip install -r requirements.txt
          - pytest tests/ -v
          - ruff check .
    - step: &build-container
        name: Build & Push Docker to ECR
        services: [docker]
        caches: [docker]
        script:
          - source scripts/aws-auth.sh
          - bash scripts/build-push-ecr.sh
    - step: &trigger-training
        name: Trigger SageMaker Training
        script:
          - source scripts/aws-auth.sh
          - bash scripts/trigger-training.sh
    - step: &deploy
        name: Deploy Model
        deployment: $ENV
        trigger: manual  # for stage/prod
        script:
          - source scripts/aws-auth.sh
          - bash scripts/trigger-codepipeline.sh

pipelines:
  branches:
    develop:
      - step: *test
      - step: *build-container
      - step: *trigger-training
      - step:
          <<: *deploy
          deployment: staging
    main:
      - step: *test
      - step: *build-container
      - step: *trigger-training
      - step:
          <<: *deploy
          deployment: production
          trigger: manual
  pull-requests:
    '**':
      - step: *test
```

**aws-auth.sh**: OIDC token exchange with AWS STS `AssumeRoleWithWebIdentity` using Bitbucket's `BITBUCKET_STEP_OIDC_TOKEN`.

**iam_bitbucket_oidc.py**: Create IAM OIDC Provider for `api.bitbucket.org/2.0/workspaces/{workspace}/pipelines-config/identity/oidc`, IAM role with trust policy for Bitbucket workspace UUID.

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**Security:** OIDC preferred over access keys. Restrict IAM trust to specific Bitbucket workspace and repository UUIDs. Use Deployment variables for env-specific secrets.

**Cost:** Bitbucket Pipelines has limited free build minutes (50 min/month free, then paid). Use caching aggressively. Delegate heavy compute to AWS (CodeBuild/SageMaker).

**Atlassian Integration:** Tag deployments with Jira issue keys for tracking. Use `atlassian/jira-release-pipe` for release notes.

---

## Integration Points

- **Upstream**: `devops/04` → IAM OIDC role for Bitbucket
- **Downstream**: `cicd/03` → triggers CodePipeline
- **Downstream**: `cicd/01` → can invoke CodeBuild
- **Downstream**: `mlops/01` → triggers SageMaker training
- **Downstream**: `devops/01` → pushes containers to ECR
