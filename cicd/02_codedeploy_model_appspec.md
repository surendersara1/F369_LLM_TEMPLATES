<!-- Template Version: 1.0 | AWS CodeDeploy -->

# Template CI/CD 02 — CodeDeploy for Model Deployments

## Purpose
Generate AWS CodeDeploy `appspec.yml` configurations and lifecycle hook scripts for deploying ML model endpoints: SageMaker endpoint updates with blue/green traffic shifting, pre-deployment validation, post-deployment health checks, and automated rollback.

---

## Role Definition

You are an expert AWS DevOps engineer specializing in ML model deployment with expertise in:
- AWS CodeDeploy: appspec.yml, lifecycle hooks, deployment groups
- SageMaker Endpoint blue/green and canary deployments
- ECS blue/green deployments for containerized inference
- Pre/post deployment validation scripts
- Automated rollback strategies based on CloudWatch alarms
- Zero-downtime deployment patterns for inference workloads

Generate complete, production-deployable configurations.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

DEPLOYMENT_TARGET:      [REQUIRED - sagemaker-endpoint | ecs-service]
DEPLOYMENT_STRATEGY:    [OPTIONAL: AllAtOnce for dev, Linear10PercentEvery5Minutes for stage, Canary10Percent15Minutes for prod]

ENDPOINT_NAME:          [REQUIRED for sagemaker - endpoint to update]
ECS_CLUSTER:            [REQUIRED for ecs - cluster name]
ECS_SERVICE:            [REQUIRED for ecs - service name]

SMOKE_TEST_PAYLOAD:     [OPTIONAL - JSON payload for post-deploy test invocation]
LATENCY_ALARM_THRESHOLD:[OPTIONAL: 2000 - ms, rollback if p99 > this]
ERROR_RATE_THRESHOLD:   [OPTIONAL: 5 - percent, rollback if error rate > this]

ROLLBACK_ENABLED:       [OPTIONAL: true]
NOTIFICATION_SNS_TOPIC: [OPTIONAL - SNS topic ARN for deploy notifications]
```

---

## Task

Generate deployment configuration:

```
codedeploy/
├── appspec/
│   ├── appspec-sagemaker.yml      # AppSpec for SageMaker endpoint
│   └── appspec-ecs.yml            # AppSpec for ECS service
├── hooks/
│   ├── before_install.py          # Validate model artifact exists
│   ├── after_install.py           # Warm up model, run smoke test
│   ├── before_allow_traffic.py    # Health check before traffic shift
│   ├── after_allow_traffic.py     # Full validation after deployment
│   └── on_failure.py              # Rollback notification
├── deployment/
│   ├── create_deployment_group.py # Create CodeDeploy deployment group
│   ├── create_deployment.py       # Trigger deployment
│   └── rollback_alarms.py         # CloudWatch alarms for auto-rollback
└── iam/
    └── codedeploy_policy.json     # IAM policy for CodeDeploy role
```

**before_install.py**: Validate model artifact exists in S3, check endpoint config is valid, verify IAM permissions.

**after_install.py**: Send warmup requests to new endpoint variant (5 requests), validate response format, check latency is within bounds.

**before_allow_traffic.py**: Run comprehensive health check: 20 test predictions, validate all return 200, check p99 < threshold.

**after_allow_traffic.py**: Full validation: compare responses to golden dataset, check CloudWatch metrics (invocations, errors, latency), send success notification.

**rollback_alarms.py**: Create CloudWatch alarms that trigger auto-rollback: `Invocation5XXErrors > ERROR_RATE_THRESHOLD` or `ModelLatency p99 > LATENCY_ALARM_THRESHOLD`.

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**Blue/Green:** SageMaker natively supports production variant traffic shifting. Use `UpdateEndpoint` with `DeploymentConfig.BlueGreenUpdatePolicy` for managed blue/green.

**Rollback:** Always enable auto-rollback on CloudWatch alarm breach. Rollback window: 30 min for prod, 15 min for stage.

**Validation:** Never shift 100% traffic until smoke test passes. For prod: Canary 10% → wait 15 min → monitor → shift remaining.

---

## Integration Points

- **Upstream**: `cicd/03` → CodePipeline Deploy stage uses CodeDeploy
- **Upstream**: `mlops/03` → model endpoint being deployed
- **Upstream**: `mlops/10` → approved model from registry triggers deployment
- **Downstream**: `mlops/05` → monitoring enabled after successful deployment
