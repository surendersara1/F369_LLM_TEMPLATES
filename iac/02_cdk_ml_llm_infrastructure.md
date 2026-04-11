<!-- Template Version: 1.0 | AWS CDK: 2.170+ | Python -->

# Template IaC 02 — CDK Python Stack for ML/LLM Infrastructure

## Purpose
Generate a production-ready AWS CDK Python application for complete ML/LLM infrastructure: SageMaker Domain, training infrastructure, inference endpoints, Model Monitor, Feature Store, S3, KMS, IAM — with environment-aware configuration for dev/stage/prod. **This is the recommended IaC approach for AWS-native teams.**

---

## Role Definition

You are an expert AWS infrastructure engineer and CDK specialist with expertise in:
- AWS CDK v2 (Python): Constructs, Stacks, Stages, Pipelines
- CDK best practices: L2/L3 constructs, aspect-based tagging, context values
- SageMaker CDK constructs: Domain, Endpoints, Feature Store
- CDK Pipelines for self-mutating CI/CD
- Multi-stack architecture: shared infra stack + ML stack + monitoring stack
- Environment-specific configuration via CDK context and cdk.json

Generate complete, production-deployable CDK application.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

COMPONENTS:             [OPTIONAL: all | networking | sagemaker | endpoints | monitoring]
VPC_ID:                 [OPTIONAL - import existing VPC or create new]
ENABLE_STUDIO:          [OPTIONAL: true]
ENABLE_FEATURE_STORE:   [OPTIONAL: true]
ENABLE_MODEL_MONITOR:   [OPTIONAL: true]

CDK_PIPELINE_ENABLED:   [OPTIONAL: false - enable self-mutating CDK Pipeline]
CDK_PIPELINE_REPO:      [OPTIONAL: GitHub repo for CDK Pipeline source]
```

---

## Task

Generate CDK Python application:

```
cdk_ml_infra/
├── app.py                         # CDK App entry point
├── cdk.json                       # CDK configuration + context
├── requirements.txt               # CDK + constructs dependencies
├── stacks/
│   ├── networking_stack.py        # VPC, subnets, VPC endpoints, SGs
│   ├── iam_stack.py               # All IAM roles + policies
│   ├── storage_stack.py           # S3 buckets + KMS keys
│   ├── sagemaker_domain_stack.py  # SageMaker Domain + Studio
│   ├── training_stack.py          # ECR, training job configs
│   ├── inference_stack.py         # Endpoint configs, auto-scaling
│   ├── feature_store_stack.py     # Feature Store groups
│   └── monitoring_stack.py        # Model Monitor, CloudWatch
├── constructs/
│   ├── sagemaker_endpoint.py      # L3 construct: endpoint + auto-scaling + monitoring
│   └── ml_bucket.py               # L3 construct: encrypted S3 with lifecycle
├── config/
│   └── environments.py            # Env-specific config dataclass
├── tests/
│   ├── test_stacks.py             # CDK assertion tests
│   └── conftest.py
└── scripts/
    └── deploy.sh                  # cdk deploy with env selection
```

**app.py**: Instantiate all stacks with env-specific config:
```python
app = cdk.App()
env_config = get_config(app.node.try_get_context("env") or "dev")
networking = NetworkingStack(app, f"{PROJECT}-networking-{env}", config=env_config)
iam = IamStack(app, f"{PROJECT}-iam-{env}", config=env_config)
storage = StorageStack(app, f"{PROJECT}-storage-{env}", config=env_config, kms_key=iam.kms_key)
sagemaker = SagemakerDomainStack(app, f"{PROJECT}-sagemaker-{env}", vpc=networking.vpc, ...)
```

**stacks/networking_stack.py**: VPC with isolated subnets for SageMaker, interface VPC endpoints (sagemaker.api, sagemaker.runtime, ecr, s3, logs, sts, kms), security groups.

**stacks/sagemaker_domain_stack.py**: `sagemaker.CfnDomain` with VPC-only mode, default user settings, JupyterServer and KernelGateway app configs.

**constructs/sagemaker_endpoint.py**: Custom L3 construct wrapping: CfnEndpointConfig, CfnEndpoint, ApplicationAutoScaling, CloudWatch alarms. Reusable across stacks.

**config/environments.py**: Dataclass with all env-specific values (instance types, scaling, retention).

**tests/test_stacks.py**: CDK assertion tests: `template.has_resource_properties("AWS::SageMaker::Domain", ...)`.

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**CDK Version:** 2.170+. Use L2 constructs where available, CfnResource for SageMaker resources without L2 support.

**Multi-Stack:** Separate stacks for networking, IAM, storage, SageMaker. Cross-stack references via direct stack attribute references (e.g., `vpc=networking.vpc`) for same-environment stacks, or `cdk.CfnOutput` + `cdk.Fn.import_value()` for cross-stage references (using `import aws_cdk as cdk`).

**Security:** KMS encryption for all storage. VPC-only mode. cdk-nag for compliance checks.

**Testing:** CDK assertion tests for every stack. Snapshot tests for drift detection.

---

## Integration Points

- **Alternative to**: `iac/01` (Terraform version of same infrastructure)
- **Downstream**: All `mlops/` templates use resources from these stacks
- **Downstream**: `cicd/03` → CDK Pipeline can self-manage deployment
