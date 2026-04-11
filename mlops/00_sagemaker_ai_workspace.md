<!-- Template Version: 1.0 | boto3: 1.35+ | sagemaker-python-sdk: 2.230+ | CDK: 2.170+ -->

# Template 00 — SageMaker AI Workspace (Domain + Studio + Users)

## Purpose
Generate a production-ready SageMaker AI workspace for data scientists: Domain creation, Studio setup, user profiles, JupyterLab Spaces, Code Editor, SageMaker Canvas, JumpStart, HyperPod, Data Wrangler, lifecycle configurations, custom images, Git integration, and the notebook-to-pipeline promotion bridge. **This is the foundational template — deploy first before any other mlops template.**

---

## Role Definition

You are an expert AWS ML platform engineer with deep expertise in:
- Amazon SageMaker Unified Studio (latest 2025 features)
- SageMaker Domain: VPC-only mode, EFS/S3 home directories, auth modes
- SageMaker User Profiles: per-user settings, execution roles, space management
- SageMaker Spaces: JupyterLab, Code Editor (VS Code), private and shared spaces
- SageMaker Canvas: low-code ML for business analysts
- SageMaker JumpStart: foundation model hub, fine-tuning, deployment
- SageMaker HyperPod: managed clusters for distributed LLM training
- SageMaker Data Wrangler: visual data preparation and feature engineering
- SageMaker Ground Truth / Ground Truth Plus: data labeling
- SageMaker Autopilot / AutoML: automated model building
- Lifecycle configurations: auto-shutdown idle notebooks, startup scripts
- Custom SageMaker images: bring-your-own container for notebooks
- Git repository integration: CodeCommit, GitHub, Bitbucket in Studio
- Notebook-to-Pipeline promotion: converting experiments to SageMaker Pipelines

Generate complete, production-deployable code with no placeholders.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED - e.g. ml-platform]
AWS_REGION:             [REQUIRED - e.g. us-east-1]
AWS_ACCOUNT_ID:         [REQUIRED - 12-digit]
ENV:                    [REQUIRED - dev | stage | prod]

# ─── Domain Configuration ───
AUTH_MODE:              [OPTIONAL: IAM | SSO]
                        IAM: for single-account, developer teams
                        SSO: for enterprise with AWS IAM Identity Center
SSO_GROUP_NAMES:        [OPTIONAL - comma-separated SSO groups for auto user provisioning]
VPC_ID:                 [OPTIONAL - existing VPC ID or create new]
SUBNET_IDS:             [OPTIONAL - comma-separated private subnet IDs]
SECURITY_GROUP_IDS:     [OPTIONAL - comma-separated SG IDs]

# ─── User Profiles ───
DATA_SCIENTISTS:        [REQUIRED - JSON list of users to provision]
    Example:
    [
        {"username": "ds-alice", "role": "senior", "gpu_access": true},
        {"username": "ds-bob", "role": "junior", "gpu_access": false},
        {"username": "analyst-carol", "role": "canvas_user", "gpu_access": false}
    ]
DEFAULT_INSTANCE_TYPE:  [OPTIONAL: ml.t3.medium for notebooks]
GPU_INSTANCE_TYPES:     [OPTIONAL: ml.g4dn.xlarge,ml.g5.2xlarge]

# ─── Storage ───
HOME_DIRECTORY_TYPE:    [OPTIONAL: EFS | S3]
                        EFS: persistent, POSIX-compatible (recommended for notebooks)
                        S3: cheaper, better for large datasets
EFS_SIZE_GB:            [OPTIONAL: 50 per user for dev, 200 for prod]
SHARED_DATA_S3_BUCKET:  [OPTIONAL: {PROJECT_NAME}-{AWS_ACCOUNT_ID}-{ENV}-shared-data]

# ─── Features to Enable ───
ENABLE_CANVAS:          [OPTIONAL: true - low-code ML for analysts]
ENABLE_JUMPSTART:       [OPTIONAL: true - foundation model hub]
ENABLE_HYPERPOD:        [OPTIONAL: false - only for large-scale LLM training]
ENABLE_DATA_WRANGLER:   [OPTIONAL: true - visual data prep]
ENABLE_GROUND_TRUTH:    [OPTIONAL: false - data labeling]
ENABLE_AUTOPILOT:       [OPTIONAL: true - AutoML]
ENABLE_CODE_EDITOR:     [OPTIONAL: true - VS Code in browser]

# ─── Lifecycle & Governance ───
AUTO_SHUTDOWN_IDLE_MINS:[OPTIONAL: 60 for dev, 120 for prod]
MAX_RUNTIME_HOURS:      [OPTIONAL: 8 for dev, 24 for prod]
COST_BUDGET_PER_USER:   [OPTIONAL: 500 - monthly USD limit per user]

# ─── Git Integration ───
GIT_REPOS:              [OPTIONAL - JSON list of repos to connect]
    Example:
    [
        {"provider": "github", "repo_url": "https://github.com/org/ml-experiments"},
        {"provider": "codecommit", "repo_name": "ml-shared-utils"}
    ]

# ─── Custom Images ───
CUSTOM_IMAGES:          [OPTIONAL - JSON list of custom notebook images]
    Example:
    [
        {"name": "pytorch-custom", "ecr_uri": "{AWS_ACCOUNT_ID}.dkr.ecr.{AWS_REGION}.amazonaws.com/custom-pytorch:latest"},
        {"name": "llm-research", "ecr_uri": "{AWS_ACCOUNT_ID}.dkr.ecr.{AWS_REGION}.amazonaws.com/llm-toolkit:latest"}
    ]
```

---

## Task

Generate a complete SageMaker AI workspace:

### 1. Project Structure
```
{PROJECT_NAME}-workspace/
├── infrastructure/
│   ├── domain_setup.py            # Create SageMaker Domain
│   ├── user_profiles.py           # Provision user profiles
│   ├── spaces.py                  # Create JupyterLab + Code Editor spaces
│   ├── efs_setup.py               # EFS file system for home directories
│   └── security_groups.py         # Security groups for Domain
├── features/
│   ├── canvas_setup.py            # Enable SageMaker Canvas
│   ├── jumpstart_config.py        # JumpStart model access policies
│   ├── hyperpod_cluster.py        # HyperPod cluster for distributed training
│   ├── data_wrangler_setup.py     # Data Wrangler access + shared flows
│   ├── ground_truth_setup.py      # Labeling workforce + job templates
│   └── autopilot_config.py        # AutoML job templates
├── lifecycle/
│   ├── auto_shutdown.sh           # Lifecycle config: auto-stop idle notebooks
│   ├── on_start.sh                # Lifecycle config: install packages on startup
│   ├── install_extensions.sh      # JupyterLab extensions (Git, variable inspector)
│   └── lifecycle_manager.py       # Register lifecycle configs with Domain
├── custom_images/
│   ├── build_custom_image.py      # Build + push custom SageMaker image to ECR
│   ├── register_image.py          # Register with SageMaker as AppImageConfig
│   └── Dockerfile.research        # Sample custom image for research notebooks
├── git_integration/
│   ├── connect_repos.py           # Connect Git repos to SageMaker Studio
│   └── codecommit_mirror.py       # Mirror GitHub repos to CodeCommit (optional)
├── governance/
│   ├── iam_roles.py               # Per-user execution roles (scoped permissions)
│   ├── budget_alerts.py           # AWS Budgets per-user cost alerts
│   ├── service_catalog.py         # Service Catalog products for self-service
│   └── tagging_policy.py          # Mandatory tags on all SageMaker resources
├── notebook_to_pipeline/
│   ├── promote_notebook.py        # Convert notebook experiment → SageMaker Pipeline
│   ├── experiment_registry.py     # Track which experiments are pipeline-ready
│   └── pipeline_template_generator.py  # Auto-generate pipeline skeleton from notebook
├── cdk/
│   ├── workspace_stack.py         # CDK stack for entire workspace
│   ├── user_stack.py              # CDK stack for user provisioning
│   └── features_stack.py          # CDK stack for Canvas, JumpStart, etc.
├── config/
│   ├── workspace_config.py        # All config in one dataclass
│   └── user_policies.json         # IAM policy templates per role (senior/junior/analyst)
├── run_setup.py                   # CLI: python run_setup.py --env dev --action create
└── README.md                      # Workspace usage guide for data scientists
```

### 2. infrastructure/domain_setup.py

Create SageMaker Domain with full configuration:

```python
# Key API: sagemaker.create_domain()
domain_config = {
    "DomainName": f"{PROJECT_NAME}-{ENV}",
    "AuthMode": AUTH_MODE,  # "IAM" or "SSO"
    "VpcId": VPC_ID,
    "SubnetIds": SUBNET_IDS,
    "DefaultUserSettings": {
        "ExecutionRole": default_execution_role_arn,
        "SecurityGroups": SECURITY_GROUP_IDS,
        "SharingSettings": {
            "NotebookOutputOption": "Allowed",
            "S3OutputPath": f"s3://{SHARED_DATA_S3_BUCKET}/notebook-outputs/"
        },
        # NOTE: JupyterServerAppSettings and KernelGatewayAppSettings are for
        # the legacy Studio Classic experience (DefaultLandingUri: "app:JupyterServer:").
        # For the new Studio experience (DefaultLandingUri: "studio::"), use
        # JupyterLabAppSettings and CodeEditorAppSettings instead.
        "JupyterLabAppSettings": {
            "DefaultResourceSpec": {
                "InstanceType": DEFAULT_INSTANCE_TYPE,
                "SageMakerImageArn": default_image_arn
            },
            "CustomImages": custom_image_configs,
            "LifecycleConfigArns": [jupyterlab_lifecycle_arn],
            "CodeRepositories": git_repo_configs
        },
        "CodeEditorAppSettings": {
            "DefaultResourceSpec": {
                "InstanceType": DEFAULT_INSTANCE_TYPE
            },
            "LifecycleConfigArns": [code_editor_lifecycle_arn]
        },
        "CanvasAppSettings": {
            "TimeSeriesForecastingSettings": {"Status": "ENABLED"},
            "ModelRegisterSettings": {"Status": "ENABLED"},
            "DirectDeploySettings": {"Status": "ENABLED"}
        } if ENABLE_CANVAS else {},
        "DefaultLandingUri": "studio::",
        "StudioWebPortal": "ENABLED"
    },
    "DomainSettings": {
        "SecurityGroupIds": SECURITY_GROUP_IDS,
        "DockerSettings": {
            "EnableDockerAccess": "ENABLED",  # For custom containers in notebooks
            "VpcOnlyTrustedAccounts": [AWS_ACCOUNT_ID]
        }
    },
    "DefaultSpaceSettings": {
        "ExecutionRole": default_execution_role_arn,
        "SecurityGroups": SECURITY_GROUP_IDS,
        "JupyterLabAppSettings": {
            "DefaultResourceSpec": {"InstanceType": DEFAULT_INSTANCE_TYPE}
        }
    },
    "AppNetworkAccessType": "VpcOnly",
    "Tags": standard_tags
}
```

### 3. infrastructure/user_profiles.py

Provision per-user profiles with role-based settings:

- **Senior Data Scientists**: GPU access, large instances, full SageMaker API access, can create endpoints
- **Junior Data Scientists**: CPU-only by default, request GPU via Service Catalog, read-only to prod data
- **Canvas Users (Analysts)**: Canvas access only, no notebook, no terminal, pre-built models only

```python
for user in DATA_SCIENTISTS:
    role_policy = load_policy(f"user_policies/{user['role']}.json")
    execution_role = create_user_execution_role(user, role_policy)

    sagemaker.create_user_profile(
        DomainId=domain_id,
        UserProfileName=user["username"],
        UserSettings={
            "ExecutionRole": execution_role,
            "JupyterLabAppSettings": {
                "DefaultResourceSpec": {
                    "InstanceType": "ml.g5.2xlarge" if user["gpu_access"] else "ml.t3.medium"
                }
            },
            "CanvasAppSettings": canvas_settings if user["role"] == "canvas_user" else {},
        },
        Tags=[{"Key": "UserRole", "Value": user["role"]}, ...]
    )
```

### 4. infrastructure/spaces.py

Create JupyterLab and Code Editor spaces:

- **Private spaces**: one per user (user's own workspace)
- **Shared spaces**: team collaboration spaces (read-write for team, read-only for others)

```python
# Private space per user
sagemaker.create_space(
    DomainId=domain_id,
    SpaceName=f"{username}-workspace",
    SpaceSettings={
        "JupyterLabAppSettings": {...},
        "AppType": "JupyterLab",
        "SpaceStorageSettings": {
            "EbsStorageSettings": {"EbsVolumeSizeInGb": 50}
        }
    },
    OwnershipSettings={"OwnerUserProfileName": username},
    SpaceSharingSettings={"SharingType": "Private"}
)

# Shared team space
sagemaker.create_space(
    DomainId=domain_id,
    SpaceName=f"{PROJECT_NAME}-team-space",
    SpaceSettings={...},
    SpaceSharingSettings={"SharingType": "Shared"}
)
```

### 5. lifecycle/auto_shutdown.sh

Lifecycle configuration to auto-stop idle notebook instances:

```bash
#!/bin/bash
# Auto-shutdown idle JupyterLab after AUTO_SHUTDOWN_IDLE_MINS minutes
IDLE_TIME=${AUTO_SHUTDOWN_IDLE_MINS:-60}

cat > /home/sagemaker-user/.auto-shutdown.py << 'SCRIPT'
import subprocess, json, time, os
IDLE_MINUTES = int(os.environ.get("IDLE_TIME", 60))
# Check kernel activity via Jupyter API
# If all kernels idle > IDLE_MINUTES, shutdown the app
SCRIPT

# Register as cron job running every 5 minutes
(crontab -l 2>/dev/null; echo "*/5 * * * * python /home/sagemaker-user/.auto-shutdown.py") | crontab -
```

### 6. lifecycle/on_start.sh

Startup script for every notebook instance:

```bash
#!/bin/bash
# Install team-standard packages
pip install --quiet sagemaker boto3 pandas scikit-learn mlflow torch transformers peft
# Configure Git
git config --global user.name "$SM_USER_PROFILE_NAME"
# Set up AWS CLI defaults
aws configure set region $AWS_REGION
# Clone team repos
git clone https://github.com/org/ml-shared-utils /home/sagemaker-user/shared-utils 2>/dev/null || true
```

### 7. features/jumpstart_config.py

Configure JumpStart access:
- Allowed foundation models (whitelist for cost control)
- Pre-approved models for one-click deployment in dev
- Block expensive models (e.g., 70B+ params) for junior users
- JumpStart model deployment to dev endpoint with auto-shutdown

### 8. features/hyperpod_cluster.py

SageMaker HyperPod setup (if ENABLE_HYPERPOD):
- Cluster with p4d/p5 instances for distributed LLM training
- Slurm integration for job scheduling
- Auto-resume on instance failures
- Shared EFS for training data across nodes
- Cost controls: max cluster lifetime, scheduled shutdown

### 9. features/canvas_setup.py

SageMaker Canvas for business analysts:
- Enable time series forecasting, NLP, CV models
- Connect to shared data in S3 / Redshift / Athena
- Allow model registration to SageMaker Model Registry
- Allow direct deployment to dev endpoint (not prod)

### 10. notebook_to_pipeline/promote_notebook.py

**Critical bridge: experimentation → automated pipeline**

Convert a data scientist's notebook experiment into a SageMaker Pipeline:

```python
def promote_to_pipeline(notebook_path: str, experiment_run_id: str, pipeline_name: str):
    """
    Takes a validated notebook experiment and generates a SageMaker Pipeline.

    Steps:
    1. Parse notebook cells → identify preprocessing, training, evaluation steps
    2. Extract hyperparameters from experiment run
    3. Extract data paths from experiment run
    4. Generate pipeline_definition.py using mlops/08 template structure
    5. Generate scripts/ (preprocess.py, train.py, evaluate.py) from notebook cells
    6. Register pipeline and create first execution
    7. Tag pipeline with source experiment_run_id for lineage
    """
    # Read experiment metadata from SageMaker Experiments
    experiment = get_experiment_run(experiment_run_id)

    # Parse notebook into pipeline steps
    steps = parse_notebook_to_steps(notebook_path)

    # Generate pipeline code from template
    pipeline_code = generate_pipeline(
        steps=steps,
        hyperparameters=experiment.parameters,
        metrics=experiment.metrics,
        data_paths=experiment.input_data_config,
        pipeline_name=pipeline_name
    )

    # Write generated pipeline to repo
    write_pipeline_to_repo(pipeline_code, pipeline_name)

    # Create PR for review
    create_pull_request(
        branch=f"promote/{pipeline_name}",
        title=f"Promote experiment {experiment_run_id} to pipeline",
        description=f"Metrics: {experiment.metrics}\nSource notebook: {notebook_path}"
    )
```

### 11. governance/iam_roles.py

Per-role execution role policies:

**senior_data_scientist.json**:
- Full SageMaker API (training, endpoints, pipelines)
- GPU instance access
- S3 read/write on project buckets
- ECR push (custom containers)
- Bedrock InvokeModel
- Can register models and create pipelines

**junior_data_scientist.json**:
- SageMaker notebooks and processing only
- CPU instances only (GPU via approval)
- S3 read-only on prod data, read/write on dev
- No endpoint creation
- No pipeline creation (must go through promotion flow)

**canvas_user.json**:
- SageMaker Canvas only
- S3 read-only
- No API access, no notebooks, no terminal

### 12. governance/budget_alerts.py

Per-user cost controls:
- AWS Budgets per user tag (UserProfile name)
- Alert at 80% of COST_BUDGET_PER_USER
- Hard stop at 100%: Lambda revokes user's GPU instance permissions
- Monthly reset
- Dashboard showing cost by user

### 13. cdk/workspace_stack.py

Complete CDK stack wrapping all components:
```python
class SageMakerWorkspaceStack(Stack):
    def __init__(self, scope, id, config, **kwargs):
        super().__init__(scope, id, **kwargs)
        # EFS for home directories
        self.efs = efs.FileSystem(self, "HomeEFS", vpc=config.vpc, ...)
        # SageMaker Domain
        self.domain = sagemaker.CfnDomain(self, "Domain", ...)
        # User profiles
        for user in config.users:
            sagemaker.CfnUserProfile(self, f"User-{user.name}", ...)
        # Lifecycle configs
        sagemaker.CfnStudioLifecycleConfig(self, "AutoShutdown", ...)
        # Custom images
        for img in config.custom_images:
            sagemaker.CfnAppImageConfig(self, f"Image-{img.name}", ...)
```

---

## Output Format

Output ALL files with headers:
```
### FILE: {PROJECT_NAME}-workspace/infrastructure/domain_setup.py
```

---

## Requirements & Constraints

**Domain:**
- VPC-only mode (AppNetworkAccessType: VpcOnly) — no public internet from notebooks
- EFS for home directories (persistent across sessions)
- Docker access enabled for custom container builds in notebooks
- Studio Web Portal enabled for unified experience

**Users:**
- Every user gets their own execution role (no shared roles)
- GPU access is opt-in, controlled by IAM policy per user role
- Canvas users are isolated from SageMaker API
- SSO mode for enterprise (auto-provision users from SSO groups)

**Cost Control:**
- Lifecycle config: auto-shutdown idle notebooks after AUTO_SHUTDOWN_IDLE_MINS
- Max runtime: kill notebooks after MAX_RUNTIME_HOURS (prevents overnight runs)
- Per-user budget alerts via AWS Budgets
- Instance type restrictions per user role
- JumpStart model whitelist to prevent accidental expensive deployments

**Security:**
- All traffic within VPC (VPC-only mode)
- KMS encryption for EFS, EBS volumes, S3
- No direct internet access — use NAT gateway or VPC endpoints
- Audit logging: CloudTrail for all SageMaker API calls
- Mandatory tags: Project, Environment, User, CostCenter

**Governance:**
- Service Catalog for self-service infrastructure requests (GPU instance, endpoint)
- Tag enforcement via SCP or IAM condition keys
- Model must pass through registry before production deployment
- Notebook promotion requires PR review (never direct push to pipeline)

---

## Code Scaffolding Hints

**Create Domain:**
```python
sm_client = boto3.client("sagemaker")
response = sm_client.create_domain(
    DomainName=f"{PROJECT_NAME}-{ENV}",
    AuthMode="IAM",
    VpcId=vpc_id,
    SubnetIds=subnet_ids,
    DefaultUserSettings={...},
    DomainSettings={"DockerSettings": {"EnableDockerAccess": "ENABLED"}},
    AppNetworkAccessType="VpcOnly",
    Tags=[{"Key": "Project", "Value": PROJECT_NAME}]
)
domain_id = response["DomainArn"].split("/")[-1]
```

**Create User Profile:**
```python
sm_client.create_user_profile(
    DomainId=domain_id,
    UserProfileName=username,
    UserSettings={"ExecutionRole": role_arn, ...}
)
```

**Create Space:**
```python
sm_client.create_space(
    DomainId=domain_id,
    SpaceName=f"{username}-lab",
    SpaceSettings={"JupyterLabAppSettings": {...}, "AppType": "JupyterLab"},
    OwnershipSettings={"OwnerUserProfileName": username},
    SpaceSharingSettings={"SharingType": "Private"}
)
```

**Register Lifecycle Config:**
```python
sm_client.create_studio_lifecycle_config(
    StudioLifecycleConfigName=f"{PROJECT_NAME}-auto-shutdown",
    StudioLifecycleConfigContent=base64.b64encode(script.encode()).decode(),
    StudioLifecycleConfigAppType="JupyterLab"
)
```

**Register Custom Image:**
```python
sm_client.create_image(ImageName="pytorch-custom", RoleArn=role_arn)
sm_client.create_image_version(ImageName="pytorch-custom", BaseImage=ecr_uri)
sm_client.create_app_image_config(
    AppImageConfigName="pytorch-custom-config",
    JupyterLabAppImageConfig={"ContainerConfig": {"ContainerArguments": [], "ContainerEntrypoint": []}}
)
```

---

## Integration Points

- **This is template 00 — deploy FIRST before anything else**
- **Downstream**: `devops/04` → IAM roles created here feed into all other templates
- **Downstream**: `devops/02` → VPC networking used by this domain
- **Downstream**: `mlops/06` → experiments tracked here promote to pipelines
- **Downstream**: `mlops/08` → notebook_to_pipeline bridge generates SageMaker Pipelines
- **Downstream**: `mlops/01` → training pipeline triggered from promoted experiments
- **Downstream**: `cicd/03` → CodePipeline picks up promoted model for stage/prod

---

## The Complete 3-Layer Workflow

```
┌─────────────────────────────────────────────────────────────────────┐
│  LAYER 1: Data Scientist Workspace  (THIS TEMPLATE)                │
│                                                                     │
│  SageMaker Domain + Studio                                          │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐              │
│  │JupyterLab│ │Code Editor│ │  Canvas  │ │ JumpStart│              │
│  │ Spaces   │ │ (VS Code) │ │ (Analysts)│ │(FM Hub) │              │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘              │
│       │             │            │             │                     │
│       ▼             ▼            ▼             ▼                     │
│  ┌─────────────────────────────────────────────────┐               │
│  │  Experiment Tracking (mlops/06)                  │               │
│  │  Feature Store (mlops/07)                        │               │
│  │  Data Wrangler + Ground Truth                    │               │
│  └───────────────────────┬─────────────────────────┘               │
│                          │                                          │
│                          ▼                                          │
│  ┌───────────────────────────────────────┐                         │
│  │  notebook_to_pipeline/promote.py      │  ◄── PR Review Gate     │
│  │  "Promote experiment to pipeline"     │                         │
│  └───────────────────────┬───────────────┘                         │
└──────────────────────────┼──────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│  LAYER 2: Dev Training Pipeline  (mlops/01, 08, cicd/01)           │
│                                                                     │
│  GitHub/Bitbucket push ──► CodeBuild ──► SageMaker Pipeline        │
│                                                                     │
│  Preprocess → Train → Evaluate → Condition                         │
│                                      │                              │
│                            ┌─────────┴──────────┐                  │
│                            │ Metric > Threshold? │                  │
│                            ├─── YES ─────────────┤                  │
│                            │                     │                  │
│                            ▼                     ▼                  │
│                    Register Model          Fail + Notify            │
│                    (mlops/10)                                       │
│                            │                                        │
│                            ▼                                        │
│                    Auto-deploy to DEV endpoint                      │
│                    (mlops/03, DEPLOY_AFTER_REGISTER=true)           │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│  LAYER 3: Production Pipeline  (cicd/03, cicd/02, mlops/03, 05)    │
│                                                                     │
│  CodePipeline: Source → Build → Train → Evaluate → APPROVE         │
│                                                        │            │
│                                                        ▼            │
│                                              Deploy to STAGE        │
│                                              (canary 10%)           │
│                                                        │            │
│                                                   Monitor 24h       │
│                                                   (mlops/05)        │
│                                                        │            │
│                                                        ▼            │
│                                              MANUAL APPROVE         │
│                                                        │            │
│                                                        ▼            │
│                                              Deploy to PROD         │
│                                              (blue/green)           │
│                                                        │            │
│                                                        ▼            │
│                                              Model Monitoring       │
│                                              Auto-rollback          │
│                                              Drift → Retrain        │
└─────────────────────────────────────────────────────────────────────┘
```
