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

---

## 12. Template Interaction Map

```
                    ┌──────────────┐
                    │ devops/04    │  IAM Roles & Policies
                    │ (Always First)│
                    └──────┬───────┘
                           │ roleArns
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        devops/02     devops/01    devops/03
        VPC/Subnets   ECR Registry  CloudWatch
              │            │
              └────────────┘
                     │ infra
              ┌──────▼──────┐
              │  iac/02     │  CDK Stack (or iac/01 Terraform)
              │  iac/03/04  │
              └──────┬──────┘
                     │ resources
              ┌──────▼──────────────────────────────────────┐
              │  mlops/00 — SageMaker AI Workspace          │
              │  Domain + Studio + Users + Canvas + JumpStart│
              │  Data Scientists experiment here             │
              └──────┬──────────────────────────────────────┘
                     │ notebook_to_pipeline bridge
         ┌───────────┼──────────────────┐
         ▼           ▼                  ▼
    mlops/07    mlops/01,02,09      mlops/04
    Feature     Training Pipeline   RAG Pipeline
    Store            │
                     ▼
                mlops/06        mlops/10
                Experiments     Model Registry
                     │               │
                     └───────┬───────┘
                             ▼
                   mlops/03         mlops/05
                   Inference +      Monitoring
                   Inf. Components
                        │
                   mlops/12         mlops/11
                   Guardrails +     Governance +
                   Agents           Bias + Audit
                        │
                    ┌────▼────────┐
                    │  mlops/08   │  Full E2E Pipeline
                    │ (Orchestrate│
                    │  all above) │
                    └────┬────────┘
                         │
                    mlops/13         Continuous Training
                    Drift→Retrain    + Data Versioning
                         │
           ┌─────────────┼──────────────┐
           ▼             ▼              ▼
      cicd/03        cicd/04        cicd/05
      CodePipeline   GitHub         Bitbucket
           │         Actions        Pipelines
      ┌────┴────┐
      ▼         ▼
   cicd/01   cicd/02
   CodeBuild  CodeDeploy
```
