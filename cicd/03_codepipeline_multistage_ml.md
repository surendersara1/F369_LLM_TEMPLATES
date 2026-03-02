<!-- Template Version: 1.0 | boto3: 1.35+ | AWS CodePipeline V2 -->

# Template CI/CD 03 — Multi-Stage CodePipeline for ML

## Purpose
Generate a production-ready AWS CodePipeline with dev/stage/prod stages for ML model lifecycle: Source → Build → Train → Evaluate → Approve → Deploy-Stage → Approve → Deploy-Prod, with manual approval gates and automated quality checks.

---

## Role Definition

You are an expert AWS DevOps engineer specializing in ML CI/CD with expertise in:
- AWS CodePipeline V2 with pipeline type YAML
- Multi-stage pipelines: Source, Build, Test, Approval, Deploy
- CodeStar Connections for GitHub/Bitbucket source integration
- CodeBuild for ML training execution
- Manual approval actions with SNS notifications
- EventBridge integration for pipeline event monitoring
- Cross-account deployments for separate dev/stage/prod accounts
- Pipeline as Code using CloudFormation/CDK/Terraform

Generate complete, production-deployable code.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

SOURCE_PROVIDER:        [REQUIRED - GitHub | Bitbucket | CodeCommit]
REPO_NAME:              [REQUIRED - e.g. org/ml-project]
BRANCH_NAME:            [OPTIONAL: main for prod, develop for stage, feature/* for dev]
CODESTAR_CONNECTION_ARN:[REQUIRED for GitHub/Bitbucket]

PIPELINE_NAME:          [OPTIONAL: {PROJECT_NAME}-{ENV}-ml-pipeline]
ARTIFACT_BUCKET:        [OPTIONAL: {PROJECT_NAME}-{AWS_ACCOUNT_ID}-{ENV}-pipeline-artifacts]

STAGES:                 [OPTIONAL: source,build,train,evaluate,approve,deploy]
INCLUDE_MANUAL_APPROVAL:[OPTIONAL: false for dev, true for stage/prod]
APPROVAL_SNS_TOPIC:     [OPTIONAL - SNS topic ARN for approval notifications]
APPROVAL_TIMEOUT_HOURS: [OPTIONAL: 72]

DEPLOY_TARGET:          [OPTIONAL: sagemaker-endpoint | ecs-service]
CROSS_ACCOUNT_PROD_ID:  [OPTIONAL - separate AWS account ID for prod deployment]
```

---

## Task

Generate multi-stage CodePipeline:

```
codepipeline/
├── pipeline/
│   ├── pipeline_definition.py     # boto3: create/update full pipeline
│   ├── stages/
│   │   ├── source_stage.py        # CodeStar Connection source action
│   │   ├── build_stage.py         # CodeBuild: lint, test, container build
│   │   ├── train_stage.py         # CodeBuild: trigger SageMaker training
│   │   ├── evaluate_stage.py      # CodeBuild: run model evaluation
│   │   ├── approve_stage.py       # Manual approval with SNS
│   │   └── deploy_stage.py        # CodeDeploy or Lambda for endpoint update
│   └── pipeline_role.py           # IAM role for CodePipeline
├── eventbridge/
│   ├── pipeline_events.py         # EventBridge rules for pipeline state changes
│   └── notifications.py           # SNS/Slack notifications
├── cdk/
│   └── pipeline_stack.py          # CDK alternative for pipeline creation
├── config/
│   └── pipeline_config.py
└── run_pipeline_setup.py          # CLI to create/update pipeline
```

**Pipeline stages:**

```
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│  Source   │──▶│  Build   │──▶│  Train   │──▶│ Evaluate │
│ (GitHub)  │   │(CodeBuild)│   │(SageMaker)│   │ (Metrics) │
└──────────┘   └──────────┘   └──────────┘   └──────────┘
                                                     │
                                              ┌──────▼───────┐
                                              │   Approve     │ (stage/prod only)
                                              │ (Manual Gate) │
                                              └──────┬───────┘
                                              ┌──────▼───────┐
                                              │   Deploy      │
                                              │ (CodeDeploy)  │
                                              └──────────────┘
```

**source_stage.py**: CodeStar Connection action pulling from GitHub/Bitbucket on branch push. Output artifact: SourceArtifact.

**build_stage.py**: CodeBuild project: install deps, run unit tests, lint, build Docker image if needed. Output artifact: BuildArtifact.

**train_stage.py**: CodeBuild project: invoke `python run_pipeline.py --env $ENV --wait`. Wait for SageMaker training job completion. Output: model artifact S3 path written to SSM.

**evaluate_stage.py**: CodeBuild project: run evaluation, check metric against threshold. Write pass/fail to output artifact.

**approve_stage.py**: Manual approval action with SNS notification containing: model metrics, training job link, experiment comparison URL.

**deploy_stage.py**: Lambda or CodeDeploy action: deploy approved model to SageMaker endpoint or ECS service. Blue/green traffic shifting.

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**Pipeline:** Use V2 pipeline type for latest features. Artifact store in S3 with KMS encryption. Pipeline execution history: retain 90 days.

**Approval:** SNS notification must include: model name, version, accuracy, link to experiment. Timeout: 72 hours. Auto-reject if timeout.

**Cross-account:** Use cross-account IAM roles for prod deployment. Pipeline in central account, deploy to prod account via assumed role.

**Triggers:** Prod pipeline triggered by tag (v*). Stage by merge to main. Dev by push to feature branch.

---

## Integration Points

- **Upstream**: `cicd/04` → GitHub Actions can trigger this pipeline
- **Upstream**: `cicd/05` → Bitbucket Pipelines can trigger this pipeline
- **Components**: `cicd/01` → CodeBuild projects used in build/train/eval stages
- **Components**: `cicd/02` → CodeDeploy used in deploy stage
- **Downstream**: `mlops/08` → SageMaker Pipeline is invoked by train stage
- **Downstream**: `mlops/10` → model registration happens in evaluate stage
