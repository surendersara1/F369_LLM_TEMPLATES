<!-- Template Version: 1.1 | boto3: 1.35+ | strands-agents: 1.x+ | Model IDs: 2026-04-22 refresh -->

# Template 24 — Bedrock Prompt Management

## Purpose
Generate a complete Bedrock Prompt Management infrastructure: prompt creation with variable placeholders using `bedrock_agent.create_prompt()`, immutable versioning with `create_prompt_version()`, prompt retrieval with `get_prompt()` and local fallback caching, A/B testing with multiple prompt variants and traffic splitting logic, a prompt lifecycle workflow (draft → review → approved → deployed) with version immutability after deployment, integration code for loading managed prompts into Strands Agent system prompts and Bedrock model invocations, Bedrock Flows prompt node configuration referencing managed prompts by ARN, and promote/rollback scripts for cross-environment prompt deployment.

---

## Role Definition

You are an expert AWS AI engineer specializing in Amazon Bedrock Prompt Management with expertise in:
- Bedrock Prompt Management APIs: `bedrock_agent.create_prompt()` for creating prompt templates with variable placeholders, `create_prompt_version()` for creating immutable versioned snapshots, `get_prompt()` for retrieving prompt definitions and specific versions
- Prompt template design: variable placeholders using Bedrock prompt template syntax (`{{variable_name}}`), input variable definitions, inference configuration (temperature, maxTokens, topP), and model-specific template formatting
- Prompt variants: defining multiple variants within a single prompt resource for A/B testing, each with independent template text, model ID, and inference configuration — enabling controlled experimentation across prompt strategies
- Prompt lifecycle management: draft → review → approved → deployed workflow with version immutability after deployment, environment promotion (dev → stage → prod), and rollback to previous versions
- A/B testing: traffic splitting logic that routes requests to different prompt variants based on configurable weights, with metrics collection for comparing variant performance
- Strands Agents SDK integration: retrieving versioned prompts from Bedrock Prompt Management and loading them as `system_prompt` in `Agent()` for dynamic prompt updates without code redeployment
- Bedrock Flows integration: configuring prompt nodes within Bedrock Flow definitions that reference managed prompts by ARN, enabling centralized prompt management across flow-based orchestrations
- Fallback caching: local cache for prompt versions that serves the most recent successfully retrieved version when the Prompt Management API is unavailable
- IAM policies for Bedrock Prompt Management: `bedrock:CreatePrompt`, `bedrock:CreatePromptVersion`, `bedrock:GetPrompt`, `bedrock:ListPrompts`, `bedrock:DeletePrompt` scoped to project resources

Generate complete, production-deployable code.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED - must be a Bedrock supported region]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

PROMPT_NAME:            [REQUIRED - identifier for the managed prompt]
                        Example: "qa-prompt", "summarization-prompt", "agent-system-prompt"
                        Used in resource naming: {PROJECT_NAME}-{PROMPT_NAME}-{ENV}

PROMPT_TEMPLATE_TEXT:   [REQUIRED - the prompt template text with variable placeholders]
                        Example: "Answer the question based on the provided context.

                        Context: {{context}}

                        Question: {{question}}

                        Provide a concise, accurate answer."

PROMPT_VARIABLES:       [REQUIRED - JSON list of input variable definitions]
                        Example: [
                            {"name": "context", "description": "Retrieved context passages"},
                            {"name": "question", "description": "User question to answer"}
                        ]

MODEL_ID:               [OPTIONAL: us.anthropic.claude-sonnet-4-7-20260109-v1:0]
                        The default model ID for the prompt's primary variant.
                        Options (verify current availability):
                        - us.anthropic.claude-sonnet-4-7-20260109-v1:0 (Claude Sonnet 4.7)
                        - us.anthropic.claude-sonnet-4-7-20260109-v1:0 (Claude Sonnet 4.7)
                        - us.anthropic.claude-haiku-4-5-20251001-v1:0 (Claude Haiku 4.5)
                        - us.amazon.nova-pro-v1:0 (Amazon Nova Pro)
                        - us.amazon.nova-lite-v1:0 (Amazon Nova Lite)

VARIANTS:               [OPTIONAL - JSON list of variant definitions for A/B testing]
                        When provided, creates multiple prompt variants with independent
                        template text, model ID, and inference configuration.
                        Example: [
                            {
                                "name": "concise",
                                "template_text": "Answer briefly: {{question}}",
                                "model_id": "us.anthropic.claude-haiku-4-5-20251001-v1:0",
                                "temperature": 0.3,
                                "max_tokens": 512,
                                "weight": 50
                            },
                            {
                                "name": "detailed",
                                "template_text": "Provide a detailed answer with examples: {{question}}",
                                "model_id": "us.anthropic.claude-sonnet-4-7-20260109-v1:0",
                                "temperature": 0.7,
                                "max_tokens": 2048,
                                "weight": 50
                            }
                        ]

FLOWS_INTEGRATION_ENABLED: [OPTIONAL: false]
                        When true, generates Bedrock Flow prompt node configuration
                        that references the managed prompt by ARN. Enables centralized
                        prompt management within flow-based orchestrations.

LIFECYCLE_WORKFLOW_ENABLED: [OPTIONAL: true]
                        When true, generates the full prompt lifecycle workflow:
                        draft → review → approved → deployed, with version immutability
                        after deployment and environment promotion scripts.

FALLBACK_CACHE_TTL:     [OPTIONAL: 300 - cache TTL in seconds for prompt retrieval fallback]
                        Local cache duration for the most recent successfully retrieved
                        prompt version. Used when the Prompt Management API is unavailable.

INFERENCE_TEMPERATURE:  [OPTIONAL: 0.7 - default temperature for the primary variant]
INFERENCE_MAX_TOKENS:   [OPTIONAL: 2048 - default max tokens for the primary variant]
```

---

## Task

Generate complete Bedrock Prompt Management infrastructure:

```
{PROJECT_NAME}-prompt-management/
├── prompts/
│   ├── create_prompt.py               # Create prompt templates with variants
│   ├── version_manager.py            # Version lifecycle: draft → review → approved → deployed
│   ├── prompt_retriever.py           # Retrieve and cache prompt versions
│   └── ab_testing.py                 # A/B test prompt variants with traffic splitting
├── integration/
│   ├── strands_integration.py        # Load managed prompts into Strands agents
│   ├── flows_integration.py          # Reference prompts in Bedrock Flow nodes
│   └── fallback_cache.py            # Local cache fallback for prompt retrieval
├── lifecycle/
│   ├── promote_prompt.py             # Promote prompt across environments
│   └── rollback_prompt.py           # Rollback to previous version
├── config.py                         # Shared configuration and constants
├── run_setup.py                      # Entry point to create and version prompts
└── requirements.txt
```


**create_prompt.py**: Prompt creation module using the Bedrock Prompt Management API:
- `create_prompt(name: str, template_text: str, variables: list[dict], model_id: str, inference_config: dict, tags: dict) -> dict`: Call `bedrock_agent.create_prompt()` with a single default variant containing the template text, input variables, model ID, and inference configuration (temperature, maxTokens). Return the prompt ID and ARN.
- `create_prompt_with_variants(name: str, variants: list[dict], default_variant: str, tags: dict) -> dict`: Call `bedrock_agent.create_prompt()` with multiple variants for A/B testing. Each variant has independent template text, model ID, and inference configuration. Return the prompt ID, ARN, and variant names.
- `delete_prompt(prompt_id: str) -> None`: Call `bedrock_agent.delete_prompt()` to remove a prompt resource. Only allowed for draft prompts that have no deployed versions.
- `list_prompts(project_name: str, env: str) -> list[dict]`: Call `bedrock_agent.list_prompts()` filtered by project tags. Return list of prompt summaries with ID, name, version count, and last updated timestamp.
- All functions use `{PROJECT_NAME}-{PROMPT_NAME}-{ENV}` naming convention for prompt resources.
- All functions include structured logging, error handling for `ValidationException` and `ConflictException`, and tag propagation with `Project={PROJECT_NAME}`, `Environment={ENV}`.

**version_manager.py**: Prompt version lifecycle management:
- `create_version(prompt_id: str, description: str) -> dict`: Call `bedrock_agent.create_prompt_version()` to create an immutable version snapshot. Return the version number and ARN.
- `get_version(prompt_id: str, version: str) -> dict`: Call `bedrock_agent.get_prompt()` with the specific version to retrieve the full prompt definition including template text, variables, and inference configuration.
- `list_versions(prompt_id: str) -> list[dict]`: List all versions of a prompt with version number, description, creation timestamp, and lifecycle status.
- `PromptLifecycle` class implementing the draft → review → approved → deployed workflow:
  - `__init__(self, prompt_id: str, metadata_table: str)`: Initialize with prompt ID and DynamoDB table name for lifecycle metadata storage.
  - `set_status(self, version: str, status: str)`: Update lifecycle status for a version. Valid transitions: draft → review → approved → deployed. Reject invalid transitions. Store status in DynamoDB with timestamp and actor.
  - `get_status(self, version: str) -> str`: Retrieve current lifecycle status for a version.
  - `get_deployed_version(self) -> str | None`: Return the currently deployed version number, or None if no version is deployed.
  - Version immutability: once a version status is set to "deployed", it MUST NOT be modified or deleted. New changes require creating a new version.
- DynamoDB table for lifecycle metadata: partition key `prompt_id`, sort key `version`, attributes: `status`, `updated_at`, `updated_by`, `description`.
- Include structured logging for all lifecycle transitions.

**prompt_retriever.py**: Prompt retrieval with caching:
- `retrieve_prompt(prompt_id: str, version: str = None) -> dict`: Call `bedrock_agent.get_prompt()` to retrieve a prompt definition. If version is None, retrieve the latest version. Return the full prompt definition including template text, variables, variants, and inference configuration.
- `retrieve_template_text(prompt_id: str, version: str = None, variant_name: str = None) -> str`: Retrieve just the template text for a specific variant. If variant_name is None, use the default variant. Return the raw template text with `{{variable}}` placeholders.
- `fill_template(template_text: str, variables: dict) -> str`: Replace `{{variable}}` placeholders in the template text with provided values. Raise `ValueError` for any variable referenced in the template but missing from the variables dict.
- Integration with `fallback_cache.py`: on successful retrieval, cache the prompt definition. On retrieval failure (`ServiceException`, `ResourceNotFoundException`), fall back to the cached version with a warning log.
- Include structured logging for retrieval operations, cache hits, and cache misses.

**ab_testing.py**: A/B testing with traffic splitting:
- `ABTestRouter` class:
  - `__init__(self, prompt_id: str, variant_weights: dict[str, int])`: Initialize with prompt ID and variant weight mapping (e.g., `{"concise": 50, "detailed": 50}`). Weights are relative — they do not need to sum to 100.
  - `select_variant(self, session_id: str = None) -> str`: Select a variant based on configured weights using weighted random selection. If session_id is provided, use consistent hashing to ensure the same session always gets the same variant (sticky routing).
  - `record_result(self, variant_name: str, metrics: dict) -> None`: Record A/B test result metrics (latency, user satisfaction, task completion) for a variant invocation. Store in CloudWatch custom metrics under `{PROJECT_NAME}/PromptABTest` namespace.
  - `get_variant_stats(self) -> dict`: Retrieve aggregated metrics per variant from CloudWatch for comparison.
- `run_ab_test(prompt_id: str, variables: dict, variant_weights: dict, session_id: str = None) -> dict`: End-to-end A/B test execution — select variant, retrieve prompt template for that variant, fill variables, and return the filled prompt text along with the selected variant name.
- Include CloudWatch metric publishing for variant selection counts and response quality metrics.

**strands_integration.py**: Load managed prompts into Strands agents:
- `load_prompt_as_system_prompt(prompt_id: str, version: str, variables: dict = None) -> str`: Retrieve a managed prompt version, optionally fill template variables, and return the text ready for use as a Strands Agent `system_prompt`.
- `create_agent_with_managed_prompt(prompt_id: str, version: str, model: BedrockModel, tools: list, variables: dict = None) -> Agent`: Create a fully configured `strands.Agent()` with the managed prompt as system prompt, the provided model, and tools. Convenience function for the common pattern of prompt retrieval → agent creation.
- `refresh_agent_prompt(agent: Agent, prompt_id: str, version: str, variables: dict = None) -> Agent`: Update an existing agent's system prompt with a new managed prompt version without recreating the agent. Enables hot-swapping prompts in long-running agents.
- Include fallback to cached prompt version if retrieval fails.
- Include structured logging for prompt loading and agent creation.

**flows_integration.py**: Bedrock Flow prompt node configuration (generated when FLOWS_INTEGRATION_ENABLED is true):
- `create_prompt_flow_node(prompt_id: str, version: str, node_name: str) -> dict`: Generate a Bedrock Flow prompt node definition that references a managed prompt by ARN. Return the node configuration dict suitable for inclusion in a Bedrock Flow definition.
- `create_flow_with_prompt_node(flow_name: str, prompt_nodes: list[dict], connections: list[dict]) -> dict`: Call `bedrock_agent.create_flow()` with prompt nodes referencing managed prompts. Return the flow ID and ARN.
- `update_flow_prompt_version(flow_id: str, node_name: str, new_version: str) -> dict`: Update a prompt node in an existing flow to reference a new prompt version. Call `bedrock_agent.update_flow()` with the updated node configuration.
- `invoke_flow_with_prompt(flow_id: str, flow_alias_id: str, inputs: list[dict]) -> dict`: Call `bedrock_agent_runtime.invoke_flow()` to execute a flow containing prompt nodes. Return the flow output.
- Include the prompt node ARN format: `arn:aws:bedrock:{AWS_REGION}:{AWS_ACCOUNT_ID}:prompt/{prompt_id}:{version}`

**fallback_cache.py**: Local cache fallback for prompt retrieval failures:
- `PromptCache` class:
  - `__init__(self, cache_dir: str = "/tmp/prompt_cache", ttl_seconds: int = 300)`: Initialize with cache directory and TTL.
  - `get(self, prompt_id: str, version: str = None) -> dict | None`: Retrieve cached prompt definition. Return None if not cached or expired.
  - `put(self, prompt_id: str, version: str, prompt_data: dict) -> None`: Cache a prompt definition with TTL.
  - `get_latest_cached(self, prompt_id: str) -> dict | None`: Return the most recently cached version for a prompt ID, regardless of specific version. Used as last-resort fallback.
  - `clear(self, prompt_id: str = None) -> None`: Clear cache for a specific prompt or all prompts.
- Cache storage: JSON files in the cache directory, named `{prompt_id}_{version}.json` with a metadata file tracking TTL expiration.
- Thread-safe file operations with file locking for concurrent Lambda invocations.
- Include structured logging for cache hits, misses, and expirations.

**promote_prompt.py**: Cross-environment prompt promotion:
- `promote_prompt(source_prompt_id: str, source_version: str, target_env: str, target_region: str = None) -> dict`: Retrieve a prompt version from the source environment, create the same prompt in the target environment (if it doesn't exist), and create a new version with the same template text and configuration. Return the target prompt ID and version.
- `copy_prompt_config(source_prompt: dict) -> dict`: Extract the prompt configuration (template text, variables, variants, inference config) from a source prompt definition for recreation in the target environment.
- Support cross-region promotion when `target_region` differs from `AWS_REGION`.
- Include validation: only prompts with "approved" lifecycle status can be promoted.
- Include structured logging and dry-run mode for previewing promotion without executing.

**rollback_prompt.py**: Rollback to a previous prompt version:
- `rollback_to_version(prompt_id: str, target_version: str) -> dict`: Update the lifecycle metadata to mark the target version as "deployed" and the current deployed version as "rolled_back". Does not delete any versions — maintains full audit trail.
- `get_rollback_candidates(prompt_id: str) -> list[dict]`: List all versions that can be rolled back to (versions with "approved" or "rolled_back" status). Return version number, description, creation timestamp, and previous deployment history.
- Include validation: target version must exist and must have been previously approved.
- Include structured logging and confirmation prompt for production rollbacks.

**config.py**: Shared configuration and constants:
- Load all configuration from environment variables: `PROJECT_NAME`, `AWS_REGION`, `AWS_ACCOUNT_ID`, `ENV`, `PROMPT_NAME`, `MODEL_ID`, `FALLBACK_CACHE_TTL`
- Define resource naming functions: `get_prompt_name()` → `{PROJECT_NAME}-{PROMPT_NAME}-{ENV}`, `get_metadata_table_name()` → `{PROJECT_NAME}-prompt-lifecycle-{ENV}`
- Define default tags: `{"Project": PROJECT_NAME, "Environment": ENV}`
- Initialize shared boto3 clients: `bedrock_agent`, `bedrock_agent_runtime`, `dynamodb`, `cloudwatch`

**run_setup.py**: Entry point script for initial prompt setup:
- Parse command-line arguments or environment variables for prompt configuration
- Call `create_prompt()` or `create_prompt_with_variants()` to create the prompt resource
- Call `create_version()` to create the initial version
- If LIFECYCLE_WORKFLOW_ENABLED, set initial lifecycle status to "draft"
- If FLOWS_INTEGRATION_ENABLED, create the prompt flow node configuration
- Print summary of created resources with IDs and ARNs

**requirements.txt**: Python dependencies:
- `strands-agents>=1.0.0`
- `boto3>=1.35.0`

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**Prompt Naming:** All prompt resources follow `{PROJECT_NAME}-{PROMPT_NAME}-{ENV}` naming convention. Example: `myai-qa-prompt-prod`. Tags `Project={PROJECT_NAME}` and `Environment={ENV}` are applied to all resources for cost allocation and filtering.

**Version Immutability:** Prompt versions created via `create_prompt_version()` are immutable — the template text, variables, and inference configuration cannot be modified after creation. To change a prompt, update the draft prompt resource and create a new version. This ensures deployed prompts are stable and auditable.

**Lifecycle Workflow:** When LIFECYCLE_WORKFLOW_ENABLED is true, every prompt version follows the lifecycle: draft → review → approved → deployed. Only one version can be in "deployed" status at a time per prompt. Promotion to "deployed" automatically sets the previous deployed version to "superseded". Lifecycle metadata is stored in a DynamoDB table with partition key `prompt_id` and sort key `version`.

**A/B Testing:** Variant weights are relative integers — they do not need to sum to 100. Traffic splitting uses weighted random selection with optional sticky routing via session ID consistent hashing. A/B test metrics are published to CloudWatch under the `{PROJECT_NAME}/PromptABTest` namespace for variant comparison.

**Fallback Cache:** The local cache stores prompt definitions as JSON files in `/tmp/prompt_cache` (configurable). Default TTL is 300 seconds. On retrieval failure, the cache serves the most recent successfully retrieved version. Cache is thread-safe with file locking for concurrent Lambda invocations. Always log a warning when serving from cache fallback.

**Flows Integration:** When FLOWS_INTEGRATION_ENABLED is true, prompt nodes reference managed prompts using the ARN format: `arn:aws:bedrock:{AWS_REGION}:{AWS_ACCOUNT_ID}:prompt/{prompt_id}:{version}`. Updating a flow's prompt version requires calling `update_flow()` with the new prompt ARN — flows do not automatically pick up new prompt versions.

**Security:** IAM policies for Prompt Management require `bedrock:CreatePrompt`, `bedrock:CreatePromptVersion`, `bedrock:GetPrompt`, `bedrock:ListPrompts`, and `bedrock:DeletePrompt` scoped to the project's prompt resources. Use resource-level permissions with ARN patterns: `arn:aws:bedrock:{AWS_REGION}:{AWS_ACCOUNT_ID}:prompt/*`. For Flows integration, add `bedrock:InvokeFlow` permission. Never expose prompt IDs or ARNs in client-facing API responses.

**Error Handling:** Handle `ValidationException` for invalid prompt template syntax (missing variable definitions, unsupported model ID). Handle `ConflictException` when creating a prompt with a name that already exists. Handle `ResourceNotFoundException` when retrieving a deleted or non-existent prompt version. All errors return structured responses with error type, message, and suggested remediation.

**Cost:** Bedrock Prompt Management API calls are billed per request. Prompt storage is included at no additional cost. Minimize API calls by caching prompt definitions locally and retrieving only when cache expires. A/B testing increases API calls proportionally to traffic — use caching to mitigate. Bedrock Flows invocations are billed separately per flow execution.

**Naming Convention:** All generated AWS resources follow `{PROJECT_NAME}-{component}-{ENV}`:
- Prompt resource: `{PROJECT_NAME}-{PROMPT_NAME}-{ENV}`
- Lifecycle metadata table: `{PROJECT_NAME}-prompt-lifecycle-{ENV}`
- CloudWatch namespace: `{PROJECT_NAME}/PromptABTest`
- Cache directory: `/tmp/prompt_cache/{PROJECT_NAME}`

---

## Code Scaffolding Hints

**Create a prompt template with variables:**
```python
import boto3

bedrock_agent = boto3.client("bedrock-agent", region_name="us-east-1")

response = bedrock_agent.create_prompt(
    name=f"{PROJECT_NAME}-qa-prompt-{ENV}",
    description="Question answering prompt with context injection",
    defaultVariant="v1",
    variants=[
        {
            "name": "v1",
            "modelId": "us.anthropic.claude-sonnet-4-7-20260109-v1:0",
            "templateType": "TEXT",
            "templateConfiguration": {
                "text": {
                    "text": "Answer based on context:\n{{context}}\n\nQuestion: {{question}}",
                    "inputVariables": [
                        {"name": "context"},
                        {"name": "question"},
                    ],
                }
            },
            "inferenceConfiguration": {
                "text": {
                    "temperature": 0.7,
                    "maxTokens": 2048,
                }
            },
        }
    ],
    tags={"Project": PROJECT_NAME, "Environment": ENV},
)

prompt_id = response["id"]
prompt_arn = response["arn"]
```

**Create an immutable prompt version:**
```python
version_response = bedrock_agent.create_prompt_version(
    promptIdentifier=prompt_id,
    description="Production-ready v1 — approved after review",
)

prompt_version = version_response["version"]
version_arn = version_response["arn"]
# Version is now immutable — template text and config cannot be changed
```

**Retrieve a prompt version:**
```python
prompt = bedrock_agent.get_prompt(
    promptIdentifier=prompt_id,
    promptVersion=prompt_version,
)

# Extract template text from the default variant
default_variant_name = prompt["defaultVariant"]
for variant in prompt["variants"]:
    if variant["name"] == default_variant_name:
        template_text = variant["templateConfiguration"]["text"]["text"]
        input_variables = variant["templateConfiguration"]["text"]["inputVariables"]
        break
```

**Create prompt with multiple variants for A/B testing:**
```python
response = bedrock_agent.create_prompt(
    name=f"{PROJECT_NAME}-ab-prompt-{ENV}",
    description="A/B test: concise vs detailed answers",
    defaultVariant="concise",
    variants=[
        {
            "name": "concise",
            "modelId": "us.anthropic.claude-haiku-4-5-20251001-v1:0",
            "templateType": "TEXT",
            "templateConfiguration": {
                "text": {
                    "text": "Answer briefly and directly: {{question}}",
                    "inputVariables": [{"name": "question"}],
                }
            },
            "inferenceConfiguration": {
                "text": {"temperature": 0.3, "maxTokens": 512}
            },
        },
        {
            "name": "detailed",
            "modelId": "us.anthropic.claude-sonnet-4-7-20260109-v1:0",
            "templateType": "TEXT",
            "templateConfiguration": {
                "text": {
                    "text": "Provide a detailed answer with examples:\n\n{{question}}",
                    "inputVariables": [{"name": "question"}],
                }
            },
            "inferenceConfiguration": {
                "text": {"temperature": 0.7, "maxTokens": 2048}
            },
        },
    ],
    tags={"Project": PROJECT_NAME, "Environment": ENV},
)
```

**A/B testing with traffic splitting:**
```python
import hashlib
import random


class ABTestRouter:
    """Route requests to prompt variants based on configured weights."""

    def __init__(self, variant_weights: dict[str, int]):
        self.variant_weights = variant_weights
        self.total_weight = sum(variant_weights.values())

    def select_variant(self, session_id: str = None) -> str:
        """Select a variant using weighted random or sticky session routing."""
        if session_id:
            # Consistent hashing for sticky sessions
            hash_val = int(hashlib.sha256(session_id.encode()).hexdigest(), 16)
            point = hash_val % self.total_weight
        else:
            point = random.randint(0, self.total_weight - 1)

        cumulative = 0
        for variant_name, weight in self.variant_weights.items():
            cumulative += weight
            if point < cumulative:
                return variant_name

        return list(self.variant_weights.keys())[-1]


# Usage
router = ABTestRouter(variant_weights={"concise": 50, "detailed": 50})
selected = router.select_variant(session_id="user-123")
```

**Load managed prompt into Strands Agent:**
```python
from strands import Agent
from strands.models.bedrock import BedrockModel


def load_prompt_as_system_prompt(prompt_id, version, variables=None):
    """Retrieve managed prompt and prepare as agent system prompt."""
    prompt = bedrock_agent.get_prompt(
        promptIdentifier=prompt_id,
        promptVersion=version,
    )

    # Extract template text from default variant
    default_variant = prompt["defaultVariant"]
    template_text = None
    for variant in prompt["variants"]:
        if variant["name"] == default_variant:
            template_text = variant["templateConfiguration"]["text"]["text"]
            break

    # Fill template variables if provided
    if variables and template_text:
        for key, value in variables.items():
            template_text = template_text.replace("{{" + key + "}}", str(value))

    return template_text


# Create agent with managed prompt
system_prompt = load_prompt_as_system_prompt(
    prompt_id=prompt_id,
    version="1",
    variables={"context": "Retrieved context here..."},
)

model = BedrockModel(
    model_id="us.anthropic.claude-sonnet-4-7-20260109-v1:0",
    region_name="us-east-1",
    max_tokens=4096,
)

agent = Agent(
    model=model,
    system_prompt=system_prompt,
    tools=[retrieval_tool, answer_tool],
)

response = agent("What is Amazon Bedrock?")
```

**Bedrock Flow prompt node referencing a managed prompt:**
```python
# Prompt node configuration for a Bedrock Flow
prompt_flow_node = {
    "type": "Prompt",
    "name": "ManagedPromptNode",
    "configuration": {
        "prompt": {
            "sourceConfiguration": {
                "resource": {
                    "promptArn": (
                        f"arn:aws:bedrock:{AWS_REGION}:{AWS_ACCOUNT_ID}"
                        f":prompt/{prompt_id}:{prompt_version}"
                    ),
                }
            }
        }
    },
    "inputs": [
        {
            "name": "context",
            "type": "String",
            "expression": "$.data.context",
        },
        {
            "name": "question",
            "type": "String",
            "expression": "$.data.question",
        },
    ],
    "outputs": [
        {
            "name": "modelCompletion",
            "type": "String",
        },
    ],
}

# Invoke a flow containing the prompt node
bedrock_agent_runtime = boto3.client("bedrock-agent-runtime", region_name=AWS_REGION)

flow_response = bedrock_agent_runtime.invoke_flow(
    flowIdentifier=flow_id,
    flowAliasIdentifier=flow_alias_id,
    inputs=[
        {
            "content": {"document": {"context": context_text, "question": user_question}},
            "nodeName": "FlowInputNode",
            "nodeOutputName": "document",
        }
    ],
)

# Process streaming response
for event in flow_response["responseStream"]:
    if "flowOutputEvent" in event:
        output = event["flowOutputEvent"]["content"]["document"]
```

**Fallback cache pattern:**
```python
import json
import os
import time
import logging
import fcntl

logger = logging.getLogger(__name__)


class PromptCache:
    """Local file-based cache for prompt definitions with TTL."""

    def __init__(self, cache_dir="/tmp/prompt_cache", ttl_seconds=300):
        self.cache_dir = cache_dir
        self.ttl_seconds = ttl_seconds
        os.makedirs(cache_dir, exist_ok=True)

    def _cache_path(self, prompt_id, version):
        return os.path.join(self.cache_dir, f"{prompt_id}_{version}.json")

    def get(self, prompt_id, version):
        """Retrieve cached prompt definition if not expired."""
        path = self._cache_path(prompt_id, version)
        if not os.path.exists(path):
            logger.debug("Cache miss: %s v%s", prompt_id, version)
            return None

        with open(path, "r") as f:
            fcntl.flock(f, fcntl.LOCK_SH)
            data = json.load(f)
            fcntl.flock(f, fcntl.LOCK_UN)

        if time.time() > data.get("expires_at", 0):
            logger.debug("Cache expired: %s v%s", prompt_id, version)
            return None

        logger.info("Cache hit: %s v%s", prompt_id, version)
        return data["prompt_data"]

    def put(self, prompt_id, version, prompt_data):
        """Cache a prompt definition with TTL."""
        path = self._cache_path(prompt_id, version)
        cache_entry = {
            "prompt_data": prompt_data,
            "cached_at": time.time(),
            "expires_at": time.time() + self.ttl_seconds,
        }

        with open(path, "w") as f:
            fcntl.flock(f, fcntl.LOCK_EX)
            json.dump(cache_entry, f)
            fcntl.flock(f, fcntl.LOCK_UN)

        logger.info("Cached: %s v%s (TTL: %ds)", prompt_id, version, self.ttl_seconds)

    def get_latest_cached(self, prompt_id):
        """Return the most recently cached version as last-resort fallback."""
        latest = None
        latest_time = 0

        for filename in os.listdir(self.cache_dir):
            if filename.startswith(f"{prompt_id}_") and filename.endswith(".json"):
                path = os.path.join(self.cache_dir, filename)
                with open(path, "r") as f:
                    data = json.load(f)
                cached_at = data.get("cached_at", 0)
                if cached_at > latest_time:
                    latest_time = cached_at
                    latest = data["prompt_data"]

        if latest:
            logger.warning("Fallback: serving latest cached version for %s", prompt_id)
        return latest


# Usage with retrieval fallback
cache = PromptCache(ttl_seconds=300)


def retrieve_prompt_with_fallback(prompt_id, version):
    """Retrieve prompt with cache fallback on API failure."""
    # Try cache first
    cached = cache.get(prompt_id, version)
    if cached:
        return cached

    # Try API
    try:
        prompt = bedrock_agent.get_prompt(
            promptIdentifier=prompt_id,
            promptVersion=version,
        )
        cache.put(prompt_id, version, prompt)
        return prompt

    except Exception as e:
        logger.warning("Prompt retrieval failed: %s. Trying fallback cache.", str(e))
        fallback = cache.get_latest_cached(prompt_id)
        if fallback:
            return fallback
        raise  # Re-raise if no fallback available
```

**Prompt lifecycle workflow:**
```python
import boto3
import time

dynamodb = boto3.resource("dynamodb", region_name=AWS_REGION)
lifecycle_table = dynamodb.Table(f"{PROJECT_NAME}-prompt-lifecycle-{ENV}")

VALID_TRANSITIONS = {
    "draft": ["review"],
    "review": ["approved", "draft"],
    "approved": ["deployed", "review"],
    "deployed": ["superseded"],
    "superseded": [],
    "rolled_back": ["deployed"],
}


class PromptLifecycle:
    """Manage prompt version lifecycle: draft → review → approved → deployed."""

    def __init__(self, prompt_id):
        self.prompt_id = prompt_id

    def set_status(self, version, new_status, actor="system"):
        """Update lifecycle status with transition validation."""
        current = self.get_status(version)
        if current and new_status not in VALID_TRANSITIONS.get(current, []):
            raise ValueError(
                f"Invalid transition: {current} → {new_status}. "
                f"Allowed: {VALID_TRANSITIONS.get(current, [])}"
            )

        # If deploying, supersede the current deployed version
        if new_status == "deployed":
            current_deployed = self.get_deployed_version()
            if current_deployed and current_deployed != version:
                self.set_status(current_deployed, "superseded", actor)

        lifecycle_table.put_item(
            Item={
                "prompt_id": self.prompt_id,
                "version": version,
                "status": new_status,
                "updated_at": int(time.time()),
                "updated_by": actor,
            }
        )

    def get_status(self, version):
        """Get current lifecycle status for a version."""
        response = lifecycle_table.get_item(
            Key={"prompt_id": self.prompt_id, "version": version}
        )
        item = response.get("Item")
        return item["status"] if item else None

    def get_deployed_version(self):
        """Return the currently deployed version number."""
        response = lifecycle_table.query(
            KeyConditionExpression="prompt_id = :pid",
            FilterExpression="status = :status",
            ExpressionAttributeValues={
                ":pid": self.prompt_id,
                ":status": "deployed",
            },
        )
        items = response.get("Items", [])
        return items[0]["version"] if items else None
```

---

## Integration Points

- **Upstream**: `mlops/12` → Bedrock Guardrails for prompt safety — apply content filtering and PII redaction guardrails to prompt template outputs before they reach end users
- **Upstream**: `devops/04` → IAM roles for Bedrock Prompt Management API access (`bedrock:CreatePrompt`, `bedrock:GetPrompt`, `bedrock:CreatePromptVersion`) and DynamoDB lifecycle metadata table permissions
- **Downstream**: `mlops/16` → Bedrock Flows references managed prompts by ARN in prompt flow nodes, enabling centralized prompt management across flow-based orchestrations
- **Downstream**: `mlops/17` → LLM Evaluation pipeline evaluates prompt quality across versions and variants, using A/B test metrics to compare prompt effectiveness
- **Downstream**: `mlops/20` → Strands Agent Lambda deployment retrieves managed prompt versions as agent system prompts, enabling dynamic prompt updates without code redeployment
- **Downstream**: `mlops/23` → Agent SOPs can be stored as managed prompts for versioning, lifecycle management, and centralized retrieval via the Prompt Management API
- **Downstream**: `mlops/22` → AgentCore deployment retrieves managed prompts for long-running agent sessions with hot-swappable prompt versions
