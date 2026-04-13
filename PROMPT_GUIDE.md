# Prompt Guide — How to Use These Templates Effectively

This guide explains how to get the best results from the F369 template library.

---

## 1. Anatomy of a Template

Every template in this library follows a consistent structure:

```
## 🎯 Purpose          → What Claude will generate
## 📋 Role Definition  → Sets Claude's expert persona
## 🧩 Context & Inputs → YOUR parameters go here (fill these in!)
## 📐 Task             → Step-by-step generation instructions
## 📦 Output Format    → Exact files Claude will produce
## ✅ Requirements     → Constraints, best practices enforced
## 💡 Scaffolding Hints → AWS APIs, patterns, SDK calls to use
## 🔗 Integration Points → How output connects to other templates
```

---

## 2. How to Fill In Parameters

Before sending a template, replace all `[PARAM_NAME]` placeholders:

```markdown
# Before (template as-is):
- PROJECT_NAME: [PROJECT_NAME]
- AWS_REGION: [AWS_REGION]
- ENV: [ENV]

# After (your values):
- PROJECT_NAME: myai-chatbot
- AWS_REGION: us-east-1
- ENV: dev
```

**Required parameters** are marked with `[REQUIRED]`.
**Optional parameters** have defaults shown as `[OPTIONAL: default_value]`.

---

## 3. Sending the Template to Claude

### Option A: Paste directly
Copy the entire template content (after filling parameters) and paste it as your message.

### Option B: Use a system prompt
For Claude API users, put the Role Definition in the system prompt and the rest as the user message:

```python
import anthropic

client = anthropic.Anthropic()
message = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=8192,
    system="You are an expert AWS MLOps engineer with deep expertise in SageMaker, Bedrock, and AWS CDK...",
    messages=[
        {"role": "user", "content": open("mlops/01_sagemaker_training_pipeline.md").read()}
    ]
)
```

### Option C: Claude Code (CLI)
```bash
cat mlops/01_sagemaker_training_pipeline.md | claude --print
```

---

## 4. Chaining Templates

Many templates depend on outputs from others. Use this chaining pattern:

### Example: Full LLM Fine-tuning to Production Deployment

```
Step 1: devops/04_iam_roles_policies_mlops.md
        → Outputs: IAM role ARNs

Step 2: devops/02_vpc_networking_ml.md
        → Outputs: VPC ID, subnet IDs, security group IDs

Step 3: iac/02_cdk_ml_llm_infrastructure.md
        (uses outputs from steps 1-2)
        → Outputs: CDK stack, S3 buckets, ECR registry URI

Step 4: mlops/00_sagemaker_ai_workspace.md        ◄── NEW
        (uses outputs from steps 1-3)
        → Outputs: SageMaker Domain, user profiles, Studio spaces,
                   Canvas, JumpStart, lifecycle configs, Git integration

Step 5: mlops/02_llm_finetuning_pipeline.md
        (uses outputs from steps 1-4, data scientists experiment in workspace)
        → Outputs: training job code, fine-tuned model S3 path

Step 6: mlops/10_model_registry_versioning.md
        (uses fine-tuned model from step 5)
        → Outputs: model package ARN, approved version

Step 7: mlops/03_llm_inference_deployment.md
        (uses model package from step 6)
        → Outputs: SageMaker endpoint or ECS service

Step 8: mlops/05_model_monitoring_drift.md
        (uses endpoint from step 7)
        → Outputs: monitoring schedule, CloudWatch alarms

Step 9: mlops/12_bedrock_guardrails_agents.md
        (uses endpoint from step 7 for Agents action groups)
        → Outputs: Guardrails, Agents, Prompt Management

Step 10: mlops/11_ml_governance_responsible_ai.md
         (uses model from step 6, endpoint from step 7)
         → Outputs: Model Cards, lineage, bias reports, audit trails

Step 11: mlops/13_continuous_training_data_versioning.md
         (uses monitoring from step 8, pipeline from step 5)
         → Outputs: drift-triggered retraining, data versioning

Step 12: cicd/03_codepipeline_multistage_ml.md
         (orchestrates steps 5-11 automatically)
         → Outputs: full CodePipeline definition
```

### Example: Enterprise ML Pipeline — Data Engineering → Security → MLOps → Observability → FinOps

This chain spans the new expansion areas for a governed, cost-optimized production ML workflow:

```
Step 1:  devops/04_iam_roles_policies_mlops.md
         → Outputs: IAM role ARNs for all downstream services

Step 2:  data/03_lake_formation_ml_governance.md
         (uses IAM roles from step 1)
         → Outputs: Lake Formation permissions, LF-Tags, governed tables

Step 3:  devops/08_kms_encryption_ml.md
         (uses IAM roles from step 1)
         → Outputs: KMS key ARNs for data and model encryption

Step 4:  data/01_glue_etl_ml_features.md
         (uses governed tables from step 2, KMS keys from step 3)
         → Outputs: Feature datasets in S3 (Parquet), Glue Data Catalog

Step 5:  mlops/01_sagemaker_training_pipeline.md
         (uses feature datasets from step 4)
         → Outputs: Trained model artifacts in S3

Step 6:  mlops/17_llm_evaluation_pipeline.md
         (uses model from step 5)
         → Outputs: Evaluation metrics (ROUGE, BLEU, BERTScore), approval gate

Step 7:  mlops/10_model_registry_versioning.md
         (uses approved model from step 6)
         → Outputs: Model package ARN, approved version

Step 8:  mlops/03_llm_inference_deployment.md
         (uses model package from step 7)
         → Outputs: SageMaker endpoint

Step 9:  devops/10_opentelemetry_ml_tracing.md
         (uses endpoint from step 8)
         → Outputs: Distributed traces via ADOT + X-Ray

Step 10: devops/11_custom_cloudwatch_model_quality.md
         (uses endpoint from step 8, traces from step 9)
         → Outputs: Custom model quality metrics, anomaly detectors

Step 11: finops/04_inference_cost_optimization.md
         (uses endpoint from step 8)
         → Outputs: Auto-scaling policies, scale-to-zero for dev

Step 12: finops/05_finops_dashboards_ml.md
         (uses cost data from step 11, metrics from step 10)
         → Outputs: Cost-per-inference dashboards, budget alerts
```

### Example: Strands Agent Workflow — SOP Authoring → Deployment → Multi-Agent → Observability → Guardrails

This chain builds a complete Strands agent system from SOP authoring through production deployment with observability and guardrails:

```
Step 1:  devops/04_iam_roles_policies_mlops.md
         → Outputs: IAM role ARNs for Lambda execution, Bedrock access,
                    AgentCore service role, DynamoDB session table access

Step 2:  mlops/23_agent_sop_authoring.md
         (uses IAM roles from step 1 for SOP loader permissions)
         → Outputs: Agent SOP markdown files with RFC 2119 keywords,
                    parameterized workflows, SOP loader/validator scripts

Step 3:  mlops/24_bedrock_prompt_management.md
         (uses SOPs from step 2 as managed prompt templates)
         → Outputs: Versioned prompts in Bedrock Prompt Management,
                    A/B testing variants, Flows integration config

Step 4:  mlops/20_strands_agent_lambda_deployment.md
         (uses IAM roles from step 1, SOPs from step 2, prompts from step 3)
         → Outputs: CDK stack with Lambda + API Gateway + DynamoDB,
                    Strands Agent with BedrockModel + MCP tools

Step 5:  mlops/21_strands_multi_agent_patterns.md
         (uses deployed agent from step 4 as base agent)
         → Outputs: Graph/Swarm/Workflow multi-agent orchestration,
                    shared state management, A2A protocol integration

Step 6:  devops/15_strands_agent_observability.md
         (uses deployed agents from steps 4-5)
         → Outputs: OTel tracing for agent loops and tool calls,
                    CloudWatch dashboards, latency alarms

Step 7:  devops/16_agent_guardrails_control.md
         (uses deployed agents from steps 4-5, observability from step 6)
         → Outputs: Agent Control YAML config, Bedrock Guardrails integration,
                    tool consent policies, prompt injection defense
```

---

## 5. Environment-Specific Overrides

Templates use `ENV` to parameterize resources. Typical differences:

| Resource | dev | stage | prod |
|----------|-----|-------|------|
| Instance type | ml.t3.medium | ml.m5.xlarge | ml.m5.4xlarge |
| Min instances | 0 (scale to zero) | 1 | 2 |
| Spot instances | Yes | Yes | No |
| Data size | 10% sample | 50% sample | Full dataset |
| Approval gate | None | Manual | Manual + Runbook |
| Retention | 7 days | 30 days | 1 year |

---

## 6. AWS Service Naming Conventions

All generated resources follow this naming pattern:
```
{PROJECT_NAME}-{component}-{ENV}
```

Examples:
- `myai-training-job-dev`
- `myai-inference-endpoint-prod`
- `myai-feature-group-stage`
- `myai-pipeline-bucket-prod`

This ensures no naming collisions across environments.

---

## 7. Getting Multi-File Output

When a template asks Claude to generate multiple files, use this prompt addition:

```
Please output each file with a clear header like:

### FILE: path/to/filename.ext
```[language]
[code content]
```

Do this for ALL files in the output.
```

---

## 8. Iterating on Generated Code

After getting the initial output, use these follow-up prompts:

| Need | Follow-up Prompt |
|------|-----------------|
| Add cost optimization | "Add spot instance support and auto-scaling to scale-to-zero for dev" |
| Harden security | "Apply least-privilege IAM and ensure no public endpoints" |
| Add error handling | "Add retry logic and dead letter queues for all async operations" |
| Multi-region | "Modify to support active-active deployment in us-east-1 and eu-west-1" |
| Reduce complexity | "Simplify to the minimal viable implementation for a proof of concept" |

---

## 9. Validating Generated Code

After Claude generates infrastructure code, validate it:

```bash
# For Terraform:
terraform init && terraform validate && terraform plan

# For CDK:
cdk synth && cdk diff

# For CloudFormation directly:
aws cloudformation validate-template --template-body file://template.yaml

# For Python SageMaker SDK code:
python -m py_compile pipeline.py && pylint pipeline.py

# For buildspec.yml:
aws codebuild start-build --project-name test-project
```

---

## 10. Template Versioning

Each template is tagged with:
```
<!-- Template Version: 1.0 | AWS SDK: boto3 1.35+ | CDK: 2.170+ | Terraform: 1.9+ -->
```

If AWS services update their APIs, regenerate using the latest template version.

---

## 11. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Missing IAM permissions | Always run `devops/04` first and reference its output ARNs |
| S3 bucket name conflicts | Use `${PROJECT_NAME}-${AWS_ACCOUNT_ID}-${ENV}-artifacts` pattern |
| Region-specific service availability | Check if Bedrock/SageMaker feature is available in your `AWS_REGION` |
| Container image cold starts | Enable SageMaker multi-model endpoints or keep min instances ≥ 1 for prod |
| Step Functions timeout | Set `HeartbeatSeconds` for long-running SageMaker training jobs |
| ECR image lifecycle | Always add lifecycle policy to avoid unbounded storage costs |
| Glue job bookmark misconfiguration | Enable job bookmarks for incremental processing; reset bookmarks when reprocessing is needed (`data/01`) |
| Kinesis shard limits throttling producers | Monitor `WriteProvisionedThroughputExceeded`; use enhanced fan-out or increase shard count (`data/02`) |
| Lake Formation permissions not propagating | Grant permissions to both IAM role AND Lake Formation; allow time for permission propagation (`data/03`) |
| Config rule evaluation lag | Custom Config rules evaluate asynchronously — allow up to 15 min before checking compliance status (`devops/05`) |
| Macie classification job scan costs | Scope Macie jobs to specific S3 prefixes; avoid scanning entire buckets to control costs (`devops/07`) |
| KMS key policy too restrictive | Always include the root account principal in key policy to prevent lockout; use `kms:ViaService` conditions (`devops/08`) |
| Savings Plans commitment lock-in | Start with Compute Savings Plans (flexible) before committing to SageMaker-specific plans; use 1-year no-upfront first (`finops/03`) |
| Spot instance interruption during training | Enable SageMaker managed spot with checkpointing; set `MaxWaitTimeInSeconds` > `MaxRuntimeInSeconds` (`finops/02`) |
| SCP deny effects blocking break-glass access | Always include a break-glass IAM role condition in SCPs; test SCPs in a sandbox OU first (`enterprise/01`) |
| Cross-account IAM trust misconfiguration | Verify trust policies on both sides; use `sts:ExternalId` for third-party accounts (`enterprise/02`) |
| OpenTelemetry high-cardinality spans | Set sampling rates appropriately (1-5% for prod); avoid per-request custom attributes that explode cardinality (`devops/10`) |
| Bedrock invocation logging costs | Log to S3 (not CloudWatch Logs) for high-volume workloads; use lifecycle rules on the logging bucket (`devops/12`) |
| Edge device connectivity gaps | Design for offline inference with local model cache; sync results when connectivity resumes (`edge/01`, `edge/02`) |
| Model compilation compatibility on edge | Verify target device hardware against SageMaker Neo supported frameworks and chip architectures before compiling (`edge/01`) |
| Strands Lambda layer cold start | Use provisioned concurrency for prod; keep Lambda layer under 250 MB unzipped; pre-initialize Agent outside handler (`mlops/20`) |
| MCP server connection timeout | Set MCP client timeout to 30s+; implement connection retry with backoff; validate MCP server URI before agent init (`mlops/20`, `mlops/21`) |
| Agent Control fail-closed behavior | Agent Control denies by default when config is missing — always deploy `agent_control.yaml` before agent code; test deny rules in dev first (`devops/16`) |
| AgentCore session state loss | Configure session TTL and persistence; use health checks to detect endpoint drift; implement session recovery on `FAILED` status (`mlops/22`) |
| Multi-agent cascading failures | Set per-agent timeouts and fallback routing; isolate agent failures with circuit breakers; avoid unbounded recursive agent calls (`mlops/21`) |

---

## 12. Template Interaction Map

74 templates across 8 directories organized into 10 layers:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  LAYER 1 — FOUNDATION                                                      │
│  devops/01 ECR  ·  devops/02 VPC  ·  devops/03 CloudWatch  ·  devops/04 IAM│
│                          │                                                  │
│  devops/04 (IAM) is ALWAYS FIRST — all other templates consume role ARNs   │
└──────────┬──────────────┬──────────────┬──────────────┬─────────────────────┘
           │              │              │              │
           ▼              ▼              ▼              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  LAYER 2 — SECURITY & COMPLIANCE                                           │
│  devops/05 Config Rules  ◄── devops/08 KMS key ARNs                        │
│  devops/06 GuardDuty + Security Hub                                        │
│  devops/07 Macie PII Detection                                             │
│  devops/08 KMS Encryption  ──► data/01, data/03, enterprise/02             │
│  devops/09 VPC Endpoint Policies  ◄── devops/02 VPC                        │
└──────────┬──────────────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  LAYER 3 — DATA ENGINEERING                                                │
│  data/01 Glue ETL  ──► mlops/07 Feature Store, mlops/01 Training, data/02  │
│  data/02 Kinesis Real-Time Features  ──► mlops/07 Feature Store            │
│  data/03 Lake Formation Governance  ◄── devops/08 KMS                      │
│  data/04 S3 Data Lifecycle                                                 │
│  data/05 EventBridge ML  ──► mlops/13 Continuous Training, devops/05,      │
│                               enterprise/05 Central Registry               │
└──────────┬──────────────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  LAYER 4 — MLOps CORE                                                      │
│                                                                             │
│  Existing (mlops/00-13):                                                   │
│  00 Workspace · 01 Training · 02 Fine-tuning · 03 Inference · 04 RAG      │
│  05 Monitoring · 06 Experiments · 07 Feature Store · 08 E2E Pipeline       │
│  09 Bedrock Fine-tuning · 10 Model Registry · 11 Governance               │
│  12 Guardrails & Agents · 13 Continuous Training                           │
│                                                                             │
│  AI/ML Gaps (mlops/14-19):                                                 │
│  14 Bedrock Agents  ──► mlops/16 Flows, devops/10 OpenTelemetry            │
│  15 Multimodal Pipelines                                                   │
│  16 Bedrock Flows  ◄── mlops/14 Agents                                     │
│  17 LLM Evaluation  ──► mlops/10 Registry, enterprise/05 Central Registry  │
│  18 Prompt Caching                                                         │
│  19 Marketplace Models                                                     │
│                                                                             │
│  Strands Agents (mlops/20-24):                                             │
│  20 Strands Lambda Deploy  ◄── mlops/23 SOPs, mlops/24 Prompt Mgmt        │
│     ──► mlops/21 Multi-Agent, devops/15 Observability, devops/16 Guardrails│
│  21 Multi-Agent Patterns  ◄── mlops/20 Lambda, devops/15 Observability     │
│  22 AgentCore Deploy  ──► devops/15 Observability, devops/16 Guardrails    │
│  23 Agent SOP Authoring  ──► mlops/20 Lambda, mlops/21 Multi-Agent,        │
│     mlops/24 Prompt Mgmt                                                   │
│  24 Bedrock Prompt Mgmt  ──► mlops/20 Lambda, mlops/16 Flows, mlops/17    │
│     Evaluation, mlops/23 SOPs                                              │
└──────────┬──────────────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  LAYER 5 — OBSERVABILITY                                                   │
│  devops/10 OpenTelemetry  ──► devops/13 Cost-per-Inference, devops/11      │
│  devops/11 Custom CloudWatch Metrics  ◄── devops/10                        │
│  devops/12 Bedrock Invocation Logging  ──► devops/13, finops/01            │
│  devops/13 Cost-per-Inference Dashboards  ◄── devops/10, devops/12         │
│  devops/14 Clarify Real-Time Bias Monitoring                               │
│  devops/15 Strands Agent Observability  ◄── devops/10, devops/03           │
│     ──► mlops/20 Lambda, mlops/22 AgentCore                               │
│  devops/16 Agent Guardrails & Control  ◄── mlops/12 Bedrock Guardrails     │
│     ──► devops/15 Observability, mlops/20 Lambda, mlops/22 AgentCore       │
└──────────┬──────────────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  LAYER 6 — FINOPS / COST MANAGEMENT                                       │
│  finops/01 Cost Allocation  ──► finops/05 Dashboards, devops/13            │
│  finops/02 Spot Instance Strategies  ──► finops/01                         │
│  finops/03 Savings Plans                                                   │
│  finops/04 Inference Cost Optimization  ──► devops/13, finops/05           │
│  finops/05 FinOps Dashboards  ◄── finops/01, finops/04                     │
└──────────┬──────────────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  LAYER 7 — ENTERPRISE / MULTI-ACCOUNT                                      │
│  enterprise/01 Organizations SCPs  ──► enterprise/04 Control Tower         │
│  enterprise/02 Cross-Account Deploy  ◄── devops/08 KMS, enterprise/05      │
│  enterprise/03 Service Catalog  ◄── enterprise/04                          │
│  enterprise/04 Control Tower  ──► enterprise/03                            │
│  enterprise/05 Centralized Registry  ──► enterprise/02, mlops/03           │
└──────────┬──────────────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  LAYER 8 — EDGE / HYBRID                                                   │
│  edge/01 SageMaker Edge Deployment  ──► edge/02 IoT Greengrass             │
│  edge/02 IoT Greengrass ML Inference                                       │
│  edge/03 Outposts ML Patterns                                              │
└──────────┬──────────────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  LAYER 9 — IaC                                                             │
│  iac/01 Terraform SageMaker  ·  iac/02 CDK ML/LLM  ·  iac/03 Terraform    │
│  Bedrock+RAG  ·  iac/04 CDK ECS LLM Inference                             │
└──────────┬──────────────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  LAYER 10 — CI/CD                                                          │
│  cicd/01 CodeBuild  ·  cicd/02 CodeDeploy  ·  cicd/03 CodePipeline        │
│  cicd/04 GitHub Actions  ·  cicd/05 Bitbucket Pipelines                    │
└─────────────────────────────────────────────────────────────────────────────┘

KEY CROSS-LAYER REFERENCES:
  devops/04 (IAM)          ──► ALL templates (upstream for every layer)
  devops/08 (KMS)          ──► devops/05, enterprise/02, data/01, data/03
  data/01 (Glue)           ──► mlops/07, mlops/01, data/02
  data/05 (EventBridge)    ──► mlops/13, devops/05, enterprise/05
  mlops/14 (Agents)        ──► mlops/16, devops/10
  mlops/17 (Evaluation)    ──► mlops/10, enterprise/05
  enterprise/05 (Registry) ──► enterprise/02, mlops/03
  devops/10 (OpenTelemetry)──► devops/13, devops/11, devops/15
  devops/12 (Bedrock Logs) ──► devops/13, finops/01
  finops/01 (Cost Alloc)   ──► finops/05, devops/13
  mlops/20 (Strands Lambda)──► mlops/21, devops/15, devops/16
  mlops/22 (AgentCore)     ──► devops/15, devops/16
  mlops/23 (Agent SOPs)    ──► mlops/20, mlops/21, mlops/24
  mlops/24 (Prompt Mgmt)   ──► mlops/20, mlops/16, mlops/17
  devops/15 (Agent Obs)    ──► mlops/20, mlops/22
  devops/16 (Agent Guard)  ──► devops/15, mlops/20, mlops/22
```
