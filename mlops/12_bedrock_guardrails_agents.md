<!-- Template Version: 1.1 | boto3: 1.35+ | Model IDs: 2026-04-22 refresh -->

# Template 12 — Bedrock Guardrails, Agents & Prompt Management

## Purpose
Generate production-ready Amazon Bedrock safety and orchestration infrastructure: Guardrails (content filtering, PII redaction, hallucination detection, denied topics), Bedrock Agents for multi-step task automation, Prompt Management for versioned prompt templates, and Model Evaluation for automated quality assessment.

---

## Role Definition

You are an expert AWS AI engineer specializing in generative AI safety and orchestration with expertise in:
- Amazon Bedrock Guardrails: content filters, denied topics, sensitive info filters, word filters, contextual grounding, automated reasoning
- Amazon Bedrock Agents: action groups, knowledge bases, agent instructions, session management
- Amazon Bedrock Prompt Management: prompt templates, versioning, variables, prompt flows
- Amazon Bedrock Model Evaluation: automatic evaluation jobs, human evaluation, custom metrics
- Responsible AI: content safety, PII protection, hallucination prevention, toxicity detection
- Integration with RAG pipelines, Lambda functions, and API Gateway

Generate complete, production-deployable code.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

COMPONENTS:             [REQUIRED - guardrails | agents | prompts | evaluation | all]

# ─── Guardrails Config ───
GUARDRAIL_NAME:         [OPTIONAL: {PROJECT_NAME}-{ENV}-guardrail]
CONTENT_FILTER_STRENGTH:[OPTIONAL: MEDIUM for dev, HIGH for prod]
                        Filters: HATE, INSULTS, SEXUAL, VIOLENCE, MISCONDUCT, PROMPT_ATTACK
DENIED_TOPICS:          [OPTIONAL - JSON list of topics to block]
    Example: [
        {"name": "competitor_advice", "definition": "Advice about using competitor products",
         "examples": ["How do I use Google Cloud ML?", "Compare with Azure ML"]},
        {"name": "financial_advice", "definition": "Specific investment or financial recommendations"}
    ]
PII_ENTITIES_TO_REDACT: [OPTIONAL: EMAIL,PHONE,SSN,CREDIT_CARD,NAME,ADDRESS]
PII_ACTION:             [OPTIONAL: ANONYMIZE | BLOCK]
BLOCKED_WORDS:          [OPTIONAL - comma-separated words to block]
GROUNDING_THRESHOLD:    [OPTIONAL: 0.7 - contextual grounding check for RAG hallucination]
RELEVANCE_THRESHOLD:    [OPTIONAL: 0.7 - relevance check for RAG responses]

# ─── Agents Config ───
AGENT_NAME:             [OPTIONAL: {PROJECT_NAME}-{ENV}-agent]
AGENT_MODEL:            [OPTIONAL: us.anthropic.claude-sonnet-4-7-20260109-v1:0]
AGENT_INSTRUCTION:      [OPTIONAL - system prompt for the agent]
KNOWLEDGE_BASE_ID:      [OPTIONAL - from mlops/04 RAG pipeline]
ACTION_GROUPS:          [OPTIONAL - JSON list of agent action groups]
    Example: [
        {"name": "lookup_order", "description": "Look up order status",
         "lambda_arn": "arn:aws:lambda:..."},
        {"name": "create_ticket", "description": "Create support ticket",
         "lambda_arn": "arn:aws:lambda:..."}
    ]

# ─── Prompt Management ───
PROMPT_TEMPLATES:       [OPTIONAL - JSON list of prompt templates to create]
    Example: [
        {"name": "qa_prompt", "model": "us.anthropic.claude-sonnet-4-7-20260109-v1:0",
         "template": "Answer based on context:\n{{context}}\n\nQuestion: {{question}}"},
        {"name": "summarize_prompt", "model": "us.anthropic.claude-haiku-4-5-20251001-v1:0",
         "template": "Summarize the following:\n{{document}}"}
    ]

# ─── Evaluation Config ───
EVAL_MODEL:             [OPTIONAL - model to evaluate]
EVAL_DATASET_S3:        [OPTIONAL - s3://bucket/path/to/eval_dataset.jsonl]
EVAL_METRICS:           [OPTIONAL: correctness,helpfulness,safety,coherence,faithfulness]
```

---

## Task

Generate complete Bedrock safety and orchestration:

```
{PROJECT_NAME}-bedrock-platform/
├── guardrails/
│   ├── create_guardrail.py            # Create Bedrock Guardrail with all policies
│   ├── update_guardrail.py            # Update policies (versioned)
│   ├── test_guardrail.py              # Test guardrail with sample inputs
│   └── guardrail_config.py            # Configuration dataclass
├── agents/
│   ├── create_agent.py                # Create Bedrock Agent
│   ├── action_groups/
│   │   ├── create_action_group.py     # Lambda-backed action groups
│   │   └── lambda_functions/
│   │       ├── order_lookup/handler.py
│   │       └── ticket_create/handler.py
│   ├── knowledge_base_attach.py       # Attach KB to agent
│   ├── invoke_agent.py                # Test agent invocation
│   └── agent_alias.py                 # Create alias for versioned deployment
├── prompts/
│   ├── create_prompt.py               # Create versioned prompt templates
│   ├── create_prompt_flow.py          # Visual prompt flow (multi-step)
│   ├── prompt_versions.py             # Version management and rollback
│   └── ab_test_prompts.py             # A/B test between prompt versions
├── evaluation/
│   ├── create_eval_job.py             # Bedrock Model Evaluation job
│   ├── custom_evaluator.py            # Custom evaluation metrics
│   └── eval_report.py                 # Parse and visualize evaluation results
├── integration/
│   ├── api_gateway_setup.py           # API Gateway → Lambda → Agent/Guardrail
│   ├── lambda_invoke.py               # Lambda wrapper for guardrailed inference
│   └── rag_with_guardrails.py         # RAG pipeline with guardrails applied
├── config/
│   └── bedrock_config.py
└── run_setup.py
```

### guardrails/create_guardrail.py

Complete guardrail with all policy types:

```python
response = bedrock.create_guardrail(
    name=GUARDRAIL_NAME,
    description=f"Safety guardrail for {PROJECT_NAME} {ENV}",
    topicPolicyConfig={
        "topicsConfig": [
            {"name": topic["name"], "definition": topic["definition"],
             "examples": topic["examples"], "type": "DENY"}
            for topic in DENIED_TOPICS
        ]
    },
    contentPolicyConfig={
        "filtersConfig": [
            {"type": "SEXUAL", "inputStrength": CONTENT_FILTER_STRENGTH, "outputStrength": CONTENT_FILTER_STRENGTH},
            {"type": "VIOLENCE", "inputStrength": CONTENT_FILTER_STRENGTH, "outputStrength": CONTENT_FILTER_STRENGTH},
            {"type": "HATE", "inputStrength": CONTENT_FILTER_STRENGTH, "outputStrength": CONTENT_FILTER_STRENGTH},
            {"type": "INSULTS", "inputStrength": CONTENT_FILTER_STRENGTH, "outputStrength": CONTENT_FILTER_STRENGTH},
            {"type": "MISCONDUCT", "inputStrength": CONTENT_FILTER_STRENGTH, "outputStrength": CONTENT_FILTER_STRENGTH},
            {"type": "PROMPT_ATTACK", "inputStrength": CONTENT_FILTER_STRENGTH}  # PROMPT_ATTACK is input-only
        ]
    },
    sensitiveInformationPolicyConfig={
        "piiEntitiesConfig": [
            {"type": entity, "action": PII_ACTION}
            for entity in PII_ENTITIES_TO_REDACT
        ]
    },
    wordPolicyConfig={
        "wordsConfig": [{"text": word} for word in BLOCKED_WORDS],
        "managedWordListsConfig": [{"type": "PROFANITY"}]
    },
    contextualGroundingPolicyConfig={
        "filtersConfig": [
            {"type": "GROUNDING", "threshold": GROUNDING_THRESHOLD},
            {"type": "RELEVANCE", "threshold": RELEVANCE_THRESHOLD}
        ]
    },
    blockedInputMessaging="Your request was blocked for safety reasons.",
    blockedOutputsMessaging="The response was blocked for safety reasons.",
    tags=[{"key": "Project", "value": PROJECT_NAME}, {"key": "Environment", "value": ENV}]
)
guardrail_id = response["guardrailId"]

# Create a version (immutable snapshot)
bedrock.create_guardrail_version(guardrailIdentifier=guardrail_id, description=f"v1 - initial")
```

### agents/create_agent.py

Create Bedrock Agent with guardrail:

```python
response = bedrock_agent.create_agent(
    agentName=AGENT_NAME,
    foundationModel=AGENT_MODEL,
    instruction=AGENT_INSTRUCTION,
    guardrailConfiguration={"guardrailIdentifier": guardrail_id, "guardrailVersion": "1"},
    idleSessionTTLInSeconds=1800,
    description=f"{PROJECT_NAME} agent for {ENV}"
)
agent_id = response["agent"]["agentId"]

# Attach knowledge base
bedrock_agent.associate_agent_knowledge_base(
    agentId=agent_id, agentVersion="DRAFT",
    knowledgeBaseId=KNOWLEDGE_BASE_ID,
    description="RAG knowledge base for document Q&A"
)

# Prepare agent (compile)
bedrock_agent.prepare_agent(agentId=agent_id)

# Create alias for deployment
bedrock_agent.create_agent_alias(agentId=agent_id, agentAliasName=f"{ENV}-latest")
```

### integration/rag_with_guardrails.py

RAG pipeline with guardrails applied to both input and output:

```python
def rag_query_with_guardrails(query: str, guardrail_id: str, kb_id: str):
    # Apply guardrail to INPUT
    input_check = bedrock_runtime.apply_guardrail(
        guardrailIdentifier=guardrail_id, guardrailVersion="1",
        source="INPUT", content=[{"text": {"text": query, "qualifiers": ["query"]}}]
    )
    if input_check["action"] == "GUARDRAIL_INTERVENED":
        return {"blocked": True, "reason": "Input blocked by guardrail"}

    # RAG retrieve + generate
    response = bedrock_agent_runtime.retrieve_and_generate(
        input={"text": query},
        retrieveAndGenerateConfiguration={
            "type": "KNOWLEDGE_BASE",
            "knowledgeBaseConfiguration": {
                "knowledgeBaseId": kb_id,
                "modelArn": model_arn,
                "generationConfiguration": {
                    "guardrailConfiguration": {
                        "guardrailId": guardrail_id, "guardrailVersion": "1"
                    }
                }
            }
        }
    )
    return response
```

### evaluation/create_eval_job.py

Bedrock Model Evaluation:

```python
response = bedrock.create_evaluation_job(
    jobName=f"{PROJECT_NAME}-eval-{timestamp}",
    roleArn=eval_role_arn,
    evaluationConfig={
        "automated": {
            "datasetMetricConfigs": [{
                "taskType": "Summarization",  # or QuestionAndAnswer, Classification, etc.
                "dataset": {"name": "eval-dataset", "datasetLocation": {"s3Uri": EVAL_DATASET_S3}},
                "metricNames": EVAL_METRICS
            }]
        }
    },
    inferenceConfig={
        "models": [{
            "bedrockModel": {
                "modelIdentifier": EVAL_MODEL,
                "inferenceParams": json.dumps({"temperature": 0.0, "maxTokens": 1024})
            }
        }]
    },
    outputDataConfig={"s3Uri": f"s3://{artifact_bucket}/evaluations/{timestamp}/"}
)
```

---

## Output Format

Output ALL files for chosen COMPONENTS with headers: `### FILE: [path]`

---

## Requirements & Constraints

**Guardrails:** Always version guardrails (immutable snapshots). Test with adversarial inputs before prod. Apply to BOTH input and output. Contextual grounding check mandatory for RAG to prevent hallucination.

**Agents:** Use DRAFT version for dev, create numbered alias for stage/prod. Attach guardrail to every agent. Session TTL: 30 min default. Log all agent traces to CloudWatch.

**PII:** ANONYMIZE replaces PII with placeholder `[NAME]`. BLOCK rejects the entire request. Use ANONYMIZE for dev (debugging), BLOCK for prod.

**Evaluation:** Run evaluation job on every model change. Compare against baseline metrics. Block deployment if safety score drops below threshold.

**Cost:** Guardrails: $0.75 per 1K text units assessed. Budget: estimate based on request volume. Agents: billed per invocation + underlying model cost.

---

## Integration Points

- **Upstream**: `mlops/04` → RAG pipeline uses guardrails for safe responses
- **Upstream**: `mlops/09` → fine-tuned Bedrock model uses guardrails
- **Downstream**: `mlops/11` → guardrail violations feed into governance audit
- **Downstream**: `devops/03` → CloudWatch dashboard for guardrail metrics
- **Downstream**: `cicd/03` → evaluation job runs in pipeline before deployment
- **Downstream**: `mlops/14` → Bedrock Agents attach guardrails for safe agentic workflows
- **Downstream**: `devops/12` → Bedrock invocation logging captures guardrail assessment traces
