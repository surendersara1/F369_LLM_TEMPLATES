<!-- Template Version: 1.0 | boto3: 1.35+ -->

# Template 16 — Bedrock Flows Visual Orchestration

## Purpose
Generate a production-ready Amazon Bedrock Flows orchestration pipeline: flow definition with input, prompt, knowledge base, agent, Lambda, condition, and output nodes wired into directed acyclic graphs, three reusable flow patterns (simple prompt chain, RAG-augmented, conditional branching), Lambda node handler code, flow versioning with aliases for environment-based deployment, and a testing utility for validating flow execution end-to-end.

---

## Role Definition

You are an expert AWS AI engineer specializing in Amazon Bedrock Flows with expertise in:
- Bedrock Flows: flow creation, node configuration, connection wiring, expression-based input/output mapping
- Flow node types: Input, Output, Prompt (inline and managed), KnowledgeBase, Agent, Lambda, Condition, Iterator, Collector
- Flow lifecycle: create → prepare → version → alias → invoke, with immutable version snapshots
- Bedrock Agent Runtime: invoke_flow with streaming response parsing, flowCompletionEvent handling
- Lambda node integration: event schema for flow Lambda invocations, structured response format
- Condition node logic: relational operators (==, !=, >, <, >=, <=) and logical operators (and, or, not) for branching
- Flow expression syntax: JSONPath-style expressions ($.data, $.data.fieldName) for extracting node inputs
- IAM policies for Bedrock Flows service roles with permissions for Bedrock models, Knowledge Bases, Agents, Lambda, and S3
- Flow patterns: simple chains, RAG-augmented pipelines, conditional branching, iterative processing

Generate complete, production-deployable code.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED - must be a Bedrock Flows supported region]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

FLOW_PATTERN:           [REQUIRED - simple_chain | rag_augmented | conditional_branch]
                        simple_chain: Input → Prompt → Output (basic prompt chaining)
                        rag_augmented: Input → KnowledgeBase → Prompt → Output (RAG pipeline)
                        conditional_branch: Input → Condition → Branch A / Branch B → Output

PROMPT_MODEL:           [OPTIONAL: anthropic.claude-3-5-sonnet-20241022-v2:0]
                        Options (verify current availability):
                        - anthropic.claude-3-5-sonnet-20241022-v2:0 (Claude 3.5 Sonnet v2)
                        - anthropic.claude-3-5-haiku-20241022-v1:0 (Claude 3.5 Haiku)
                        - amazon.nova-pro-v1:0 (Amazon Nova Pro)
                        - amazon.nova-lite-v1:0 (Amazon Nova Lite)

PROMPT_TEMPLATE:        [OPTIONAL: "Answer the following question: {{query}}"]
                        Inline prompt text with {{variable}} placeholders for prompt nodes

FLOW_NODES:             [OPTIONAL - JSON list of additional node definitions]
                        Example: [
                            {
                                "type": "Prompt",
                                "name": "Summarizer",
                                "modelId": "amazon.nova-lite-v1:0",
                                "template": "Summarize: {{text}}",
                                "inputs": [{"name": "text", "type": "String"}]
                            },
                            {
                                "type": "KnowledgeBase",
                                "name": "KBLookup",
                                "knowledgeBaseId": "KB_ID_HERE"
                            }
                        ]

LAMBDA_NODES:           [OPTIONAL - JSON list of Lambda node configurations]
                        Example: [
                            {
                                "name": "DataEnricher",
                                "functionArn": "arn:aws:lambda:REGION:ACCOUNT:function:enricher",
                                "description": "Enriches input data with external API calls"
                            }
                        ]

KB_ID:                  [OPTIONAL - Bedrock Knowledge Base ID for RAG-augmented pattern]
                        Required when FLOW_PATTERN = rag_augmented

AGENT_ID:               [OPTIONAL - Bedrock Agent ID for agent nodes]
AGENT_ALIAS_ID:         [OPTIONAL - Bedrock Agent Alias ID for agent nodes]

CONDITION_EXPRESSION:   [OPTIONAL: "input_type == \"technical\""]
                        Condition expression for conditional_branch pattern
                        Uses relational (==, !=, >, <) and logical (and, or, not) operators

GUARDRAIL_ID:           [OPTIONAL - Bedrock Guardrail ID to attach to prompt nodes]
GUARDRAIL_VERSION:      [OPTIONAL: DRAFT]

MAX_TOKENS:             [OPTIONAL: 2048 - max tokens for prompt node inference]
TEMPERATURE:            [OPTIONAL: 0.7 - temperature for prompt node inference]
```

---

## Task

Generate complete Bedrock Flows orchestration:

```
{PROJECT_NAME}-bedrock-flows/
├── config.py                              # Central configuration
├── flows/
│   ├── create_flow.py                     # Create flow with nodes and connections
│   ├── flow_patterns/
│   │   ├── simple_chain.py                # Input → Prompt → Output pattern
│   │   ├── rag_augmented.py               # Input → KB → Prompt → Output pattern
│   │   └── conditional_branch.py          # Input → Condition → Branch A/B → Output pattern
│   ├── node_builders.py                   # Helper functions to build each node type
│   └── connection_builders.py             # Helper functions to wire node connections
├── lambda_nodes/
│   └── handler.py                         # Lambda handler for flow Lambda nodes
├── versioning/
│   ├── prepare_and_version.py             # Prepare flow, create version snapshot
│   └── alias_manager.py                   # Create and update flow aliases
├── invoke/
│   ├── invoke_flow.py                     # Invoke flow with streaming response parsing
│   └── api_handler.py                     # Lambda handler for API Gateway integration
├── testing/
│   └── test_flow.py                       # Flow testing utility with sample inputs
├── cleanup/
│   └── delete_resources.py                # Delete aliases, versions, and flows
├── run_setup.py                           # CLI orchestrator
└── requirements.txt
```


**config.py**: Central configuration dataclass with all parameters. Load from environment variables or CLI args. Validate required fields. Construct resource names using `{PROJECT_NAME}-{component}-{ENV}` convention. Validate FLOW_PATTERN is one of the three supported patterns. If FLOW_PATTERN is `rag_augmented`, validate KB_ID is provided.

**create_flow.py**: Create a Bedrock Flow using `bedrock_agent.create_flow()`:
- Import the appropriate pattern module based on FLOW_PATTERN
- Build nodes list and connections list from the selected pattern
- If additional FLOW_NODES or LAMBDA_NODES are provided, merge them into the definition
- Call `bedrock_agent.create_flow()` with `name`, `description`, `executionRoleArn`, and `definition` containing `nodes` and `connections`
- Tag with Project and Environment
- Return flow ID for subsequent operations

**simple_chain.py**: Build a simple prompt chain flow:
- FlowInput node (type: Input) with output named `document` of type `Object`
- Prompt node (type: Prompt) with inline prompt configuration using PROMPT_MODEL and PROMPT_TEMPLATE, input mapped from FlowInput via expression `$.data.query`
- FlowOutput node (type: Output) with input mapped from Prompt node's `modelCompletion` output
- Data connections wiring FlowInput → Prompt → FlowOutput

**rag_augmented.py**: Build a RAG-augmented flow:
- FlowInput node with output type `Object`
- KnowledgeBase node (type: KnowledgeBase) configured with KB_ID, input `retrievalQuery` mapped from FlowInput via `$.data.query`
- Prompt node with inline prompt that includes `{{context}}` and `{{query}}` variables, context mapped from KB node's `outputText` output, query mapped from FlowInput
- FlowOutput node mapped from Prompt node's `modelCompletion`
- Data connections wiring FlowInput → KB → Prompt → FlowOutput, plus FlowInput → Prompt (for query passthrough)

**conditional_branch.py**: Build a conditional branching flow:
- FlowInput node with output type `Object`
- Condition node (type: Condition) with input mapped from FlowInput, conditions defined using CONDITION_EXPRESSION (e.g., `input_type == "technical"`) plus a `default` condition
- Two Prompt nodes: Branch A (technical prompt) and Branch B (general prompt), each with their own model and template configuration
- Two FlowOutput nodes, one per branch
- Data connections from FlowInput → Condition, conditional connections from Condition → Branch A and Condition → Branch B (default), data connections from each branch prompt → its output node

**node_builders.py**: Helper functions to construct each node type:
- `build_input_node(output_type)`: Build FlowInput node
- `build_output_node(name, input_expression)`: Build FlowOutput node
- `build_prompt_node(name, model_id, template, variables, guardrail_config)`: Build inline Prompt node with inferenceConfiguration
- `build_kb_node(name, kb_id)`: Build KnowledgeBase node
- `build_agent_node(name, agent_id, agent_alias_id)`: Build Agent node
- `build_lambda_node(name, function_arn)`: Build Lambda node
- `build_condition_node(name, inputs, conditions)`: Build Condition node with expressions

**connection_builders.py**: Helper functions to wire connections:
- `build_data_connection(source_node, source_output, target_node, target_input)`: Build a Data type connection
- `build_conditional_connection(source_node, target_node, condition_name)`: Build a Conditional type connection

**handler.py** (Lambda node handler): Lambda function invoked by Bedrock Flows Lambda nodes:
- Parse the flow Lambda event: extract `node`, `flow`, `messageVersion`, and input payload
- Execute business logic (data enrichment, external API calls, transformations)
- Return response in the format Bedrock Flows expects: `{"nodeOutputs": [{"nodeOutputName": "output_name", "content": {"document": result}}]}`
- On error, return structured error with details for flow error handling

**prepare_and_version.py**: Prepare and version a flow:
- Call `bedrock_agent.prepare_flow(flowIdentifier=flow_id)` to validate and prepare the working draft
- Poll `bedrock_agent.get_flow(flowIdentifier=flow_id)` until status is `PREPARED`
- Call `bedrock_agent.create_flow_version(flowIdentifier=flow_id)` to create an immutable version snapshot
- Return version number for alias creation

**alias_manager.py**: Manage flow aliases for deployment:
- `create_alias(flow_id, version, alias_name)`: Call `bedrock_agent.create_flow_alias()` with `routingConfiguration` pointing to the version
- `update_alias(flow_id, alias_id, new_version)`: Call `bedrock_agent.update_flow_alias()` to point alias to a new version (for blue-green deployment)
- `list_aliases(flow_id)`: List all aliases for a flow
- `delete_alias(flow_id, alias_id)`: Delete an alias

**invoke_flow.py**: Invoke a flow with streaming response handling:
- Call `bedrock_agent_runtime.invoke_flow()` with `flowIdentifier`, `flowAliasIdentifier`, and `inputs` array
- Iterate over `responseStream` events, collecting `flowOutputEvent` content and checking `flowCompletionEvent` for `SUCCESS` status
- Return parsed output document and completion status
- Handle errors and timeouts gracefully

**api_handler.py**: Lambda handler for API Gateway integration:
- Accept POST with `{"query": "...", "parameters": {...}}`
- Invoke flow via `invoke_flow()`
- Return flow output as JSON response

**test_flow.py**: Flow testing utility:
- Define sample inputs for each FLOW_PATTERN
- Invoke the flow with test inputs
- Validate output structure and content
- Print results with pass/fail status
- Support dry-run mode that validates flow definition without invoking

**delete_resources.py**: Clean up all resources:
- Delete flow aliases via `bedrock_agent.delete_flow_alias()`
- Delete flow versions via `bedrock_agent.delete_flow_version()`
- Delete the flow via `bedrock_agent.delete_flow()`

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**Flow Structure:** Every flow must have exactly one FlowInput node and at least one FlowOutput node. Every node output must be connected to a downstream node input. Node names must be unique within a flow. Use JSONPath-style expressions (`$.data`, `$.data.fieldName`) to extract inputs from upstream node outputs.

**Node Types:** Prompt nodes require either an inline prompt definition or a Prompt Management reference. KnowledgeBase nodes require a valid KB_ID. Agent nodes require both agent ID and alias ID. Lambda nodes require a deployed Lambda function ARN. Condition nodes require at least one named condition plus a `default` condition.

**Versioning:** Always prepare a flow before creating a version — `prepare_flow()` validates the definition. Versions are immutable snapshots. Use aliases to point to versions for deployment. Update aliases to roll forward or roll back without changing application code.

**Lambda Nodes:** Lambda functions invoked by flows receive a specific event format with `node`, `flow`, and input data. Response must include `nodeOutputs` array with `nodeOutputName` and `content.document`. Lambda timeout should accommodate the expected processing time (default 30 seconds, max 900 seconds).

**Security:** Flow service role needs `bedrock:InvokeModel` for configured models, `bedrock:Retrieve` for Knowledge Bases, `bedrock:InvokeAgent` for Agent nodes, and `lambda:InvokeFunction` for Lambda nodes. Use VPC endpoints for Bedrock API in private subnets. Enable CloudTrail for all Bedrock Flows API calls.

**Cost:** Flows are billed per node step execution plus underlying resource costs (model invocations, KB retrievals, Lambda executions). Condition nodes and Input/Output nodes have no additional cost. Monitor with CloudWatch metrics. Minimize unnecessary nodes to reduce per-invocation cost.

**Naming:** All resources follow `{PROJECT_NAME}-{component}-{ENV}` convention:
- Flow: `{PROJECT_NAME}-flow-{FLOW_PATTERN}-{ENV}`
- Flow alias: `{ENV}-latest`
- Lambda node function: `{PROJECT_NAME}-flow-lambda-{node_name}-{ENV}`

---

## Code Scaffolding Hints


**Build flow nodes — input, prompt, and output:**
```python
import boto3
import json

bedrock_agent = boto3.client("bedrock-agent", region_name=AWS_REGION)

# Input node — validates InvokeFlow content is a JSON object
input_node = {
    "type": "Input",
    "name": "FlowInput",
    "outputs": [
        {
            "name": "document",
            "type": "Object",
        }
    ],
}

# Prompt node — inline prompt with variable substitution
prompt_node = {
    "type": "Prompt",
    "name": "MainPrompt",
    "configuration": {
        "prompt": {
            "sourceConfiguration": {
                "inline": {
                    "modelId": PROMPT_MODEL,
                    "templateType": "TEXT",
                    "inferenceConfiguration": {
                        "text": {
                            "temperature": TEMPERATURE,
                            "maxTokens": MAX_TOKENS,
                        }
                    },
                    "templateConfiguration": {
                        "text": {
                            "text": PROMPT_TEMPLATE,
                        }
                    },
                }
            }
        }
    },
    "inputs": [
        {
            "name": "query",
            "type": "String",
            "expression": "$.data.query",
        }
    ],
    "outputs": [
        {
            "name": "modelCompletion",
            "type": "String",
        }
    ],
}

# Output node — returns model completion as flow output
output_node = {
    "type": "Output",
    "name": "FlowOutput",
    "inputs": [
        {
            "name": "document",
            "type": "String",
            "expression": "$.data",
        }
    ],
}
```

**Build KnowledgeBase node for RAG-augmented pattern:**
```python
kb_node = {
    "type": "KnowledgeBase",
    "name": "KBRetrieval",
    "configuration": {
        "knowledgeBase": {
            "knowledgeBaseId": KB_ID,
            "modelId": PROMPT_MODEL,  # model used for response generation
        }
    },
    "inputs": [
        {
            "name": "retrievalQuery",
            "type": "String",
            "expression": "$.data.query",
        }
    ],
    "outputs": [
        {
            "name": "outputText",
            "type": "String",
        }
    ],
}
```

**Build Condition node for conditional branching:**
```python
condition_node = {
    "type": "Condition",
    "name": "RouteByType",
    "inputs": [
        {
            "name": "input_type",
            "type": "String",
            "expression": "$.data.input_type",
        }
    ],
    "configuration": {
        "condition": {
            "conditions": [
                {
                    "name": "is_technical",
                    "expression": "input_type == \"technical\"",
                },
                {
                    "name": "default",
                    "expression": "default",
                },
            ]
        }
    },
}
```

**Build Agent node:**
```python
agent_node = {
    "type": "Agent",
    "name": "AgentProcessor",
    "configuration": {
        "agent": {
            "agentId": AGENT_ID,
            "agentAliasId": AGENT_ALIAS_ID,
        }
    },
    "inputs": [
        {
            "name": "agentInputText",
            "type": "String",
            "expression": "$.data",
        }
    ],
    "outputs": [
        {
            "name": "agentResponse",
            "type": "String",
        }
    ],
}
```

**Build Lambda node:**
```python
lambda_node = {
    "type": "LambdaFunction",
    "name": "DataEnricher",
    "configuration": {
        "lambdaFunction": {
            "lambdaArn": f"arn:aws:lambda:{AWS_REGION}:{AWS_ACCOUNT_ID}:function:{PROJECT_NAME}-flow-lambda-enricher-{ENV}",
        }
    },
    "inputs": [
        {
            "name": "codeHookInput",
            "type": "Object",
            "expression": "$.data",
        }
    ],
    "outputs": [
        {
            "name": "codeHookOutput",
            "type": "Object",
        }
    ],
}
```

**Wire connections between nodes:**
```python
def build_data_connection(source_node, source_output, target_node, target_input):
    """Build a data connection between two nodes."""
    return {
        "name": f"{source_node}_{target_node}_{target_input}",
        "source": source_node,
        "target": target_node,
        "type": "Data",
        "configuration": {
            "data": {
                "sourceOutput": source_output,
                "targetInput": target_input,
            }
        },
    }

def build_conditional_connection(source_node, target_node, condition_name):
    """Build a conditional connection from a condition node to a target."""
    return {
        "name": f"{source_node}_{target_node}_{condition_name}",
        "source": source_node,
        "target": target_node,
        "type": "Conditional",
        "configuration": {
            "conditional": {
                "condition": condition_name,
            }
        },
    }

# Simple chain connections: FlowInput → Prompt → FlowOutput
simple_chain_connections = [
    build_data_connection("FlowInput", "document", "MainPrompt", "query"),
    build_data_connection("MainPrompt", "modelCompletion", "FlowOutput", "document"),
]

# RAG-augmented connections: FlowInput → KB → Prompt → FlowOutput
rag_connections = [
    build_data_connection("FlowInput", "document", "KBRetrieval", "retrievalQuery"),
    build_data_connection("KBRetrieval", "outputText", "MainPrompt", "context"),
    build_data_connection("FlowInput", "document", "MainPrompt", "query"),
    build_data_connection("MainPrompt", "modelCompletion", "FlowOutput", "document"),
]

# Conditional branch connections: FlowInput → Condition → Branch A/B → Output A/B
conditional_connections = [
    build_data_connection("FlowInput", "document", "RouteByType", "input_type"),
    build_conditional_connection("RouteByType", "TechnicalPrompt", "is_technical"),
    build_conditional_connection("RouteByType", "GeneralPrompt", "default"),
    build_data_connection("FlowInput", "document", "TechnicalPrompt", "query"),
    build_data_connection("FlowInput", "document", "GeneralPrompt", "query"),
    build_data_connection("TechnicalPrompt", "modelCompletion", "TechnicalOutput", "document"),
    build_data_connection("GeneralPrompt", "modelCompletion", "GeneralOutput", "document"),
]
```

**Create the flow:**
```python
response = bedrock_agent.create_flow(
    name=f"{PROJECT_NAME}-flow-{FLOW_PATTERN}-{ENV}",
    description=f"Bedrock Flow ({FLOW_PATTERN}) for {PROJECT_NAME} in {ENV}",
    executionRoleArn=flow_service_role_arn,
    definition={
        "nodes": [input_node, prompt_node, output_node],  # adjust per pattern
        "connections": simple_chain_connections,            # adjust per pattern
    },
    tags={"Project": PROJECT_NAME, "Environment": ENV},
)

flow_id = response["id"]
```

**Prepare flow, create version, and create alias:**
```python
import time

# Prepare the flow (validates definition)
bedrock_agent.prepare_flow(flowIdentifier=flow_id)

# Poll until prepared
while True:
    flow_status = bedrock_agent.get_flow(flowIdentifier=flow_id)
    status = flow_status["status"]
    if status == "Prepared":
        break
    elif status == "Failed":
        raise RuntimeError(f"Flow preparation failed: {flow_status.get('validations', [])}")
    time.sleep(5)

# Create immutable version snapshot
version_response = bedrock_agent.create_flow_version(flowIdentifier=flow_id)
flow_version = version_response["version"]

# Create alias pointing to the version
alias_response = bedrock_agent.create_flow_alias(
    flowIdentifier=flow_id,
    name=f"{ENV}-latest",
    description=f"Alias for {ENV} environment pointing to version {flow_version}",
    routingConfiguration=[
        {
            "flowVersion": flow_version,
        }
    ],
)
flow_alias_id = alias_response["id"]
```

**Update alias to point to a new version (blue-green deployment):**
```python
bedrock_agent.update_flow_alias(
    flowIdentifier=flow_id,
    aliasIdentifier=flow_alias_id,
    name=f"{ENV}-latest",
    routingConfiguration=[
        {
            "flowVersion": new_version,
        }
    ],
)
```

**Invoke flow with streaming response parsing:**
```python
bedrock_agent_runtime = boto3.client("bedrock-agent-runtime", region_name=AWS_REGION)

def invoke_flow(flow_id, flow_alias_id, input_data):
    """Invoke a Bedrock Flow and parse the streaming response."""
    response = bedrock_agent_runtime.invoke_flow(
        flowIdentifier=flow_id,
        flowAliasIdentifier=flow_alias_id,
        inputs=[
            {
                "content": {
                    "document": input_data,
                },
                "nodeName": "FlowInput",
                "nodeOutputName": "document",
            }
        ],
    )

    result = {}
    for event in response.get("responseStream"):
        result.update(event)

    if result.get("flowCompletionEvent", {}).get("completionReason") == "SUCCESS":
        output = result.get("flowOutputEvent", {}).get("content", {}).get("document")
        return {"status": "SUCCESS", "output": output}
    else:
        return {
            "status": "FAILED",
            "reason": result.get("flowCompletionEvent", {}).get("completionReason", "UNKNOWN"),
        }

# Example invocations per pattern:

# Simple chain
result = invoke_flow(flow_id, flow_alias_id, {"query": "Explain serverless computing"})

# RAG-augmented
result = invoke_flow(flow_id, flow_alias_id, {"query": "What is our refund policy?"})

# Conditional branch
result = invoke_flow(flow_id, flow_alias_id, {
    "query": "How do I configure VPC endpoints?",
    "input_type": "technical",
})
```

**Lambda node handler for Bedrock Flows:**
```python
import json

def lambda_handler(event, context):
    """Lambda handler invoked by a Bedrock Flows Lambda node.

    Event structure from Bedrock Flows:
    {
        "node": {"name": "DataEnricher", "type": "LambdaFunction"},
        "flow": {"flowArn": "...", "flowAliasArn": "..."},
        "messageVersion": "1.0",
        "codeHookInput": { ... input data ... }
    }
    """
    node_name = event.get("node", {}).get("name", "")
    input_data = event.get("codeHookInput", {})

    try:
        # Execute business logic based on node name or input
        enriched = enrich_data(input_data)

        return {
            "messageVersion": "1.0",
            "codeHookOutput": enriched,
        }

    except Exception as e:
        return {
            "messageVersion": "1.0",
            "codeHookOutput": {
                "error": str(e),
                "node": node_name,
                "message": "Lambda node processing failed",
            },
        }


def enrich_data(data):
    """Example business logic — enrich input data."""
    # Add metadata, call external APIs, transform data, etc.
    return {
        **data,
        "enriched": True,
        "timestamp": __import__("datetime").datetime.utcnow().isoformat(),
    }
```

**Flow testing utility:**
```python
import json

def test_flow(flow_id, flow_alias_id, test_cases):
    """Test a flow with sample inputs and validate outputs."""
    results = []

    for i, test in enumerate(test_cases):
        print(f"Running test {i + 1}/{len(test_cases)}: {test['name']}")

        try:
            result = invoke_flow(flow_id, flow_alias_id, test["input"])

            passed = result["status"] == "SUCCESS"
            if passed and test.get("validate"):
                passed = test["validate"](result["output"])

            results.append({
                "name": test["name"],
                "status": "PASS" if passed else "FAIL",
                "output": result.get("output", ""),
            })
            print(f"  {'PASS' if passed else 'FAIL'}")

        except Exception as e:
            results.append({"name": test["name"], "status": "ERROR", "error": str(e)})
            print(f"  ERROR: {e}")

    passed = sum(1 for r in results if r["status"] == "PASS")
    print(f"\nResults: {passed}/{len(results)} passed")
    return results


# Example test cases
SIMPLE_CHAIN_TESTS = [
    {
        "name": "Basic query",
        "input": {"query": "What is Amazon Bedrock?"},
        "validate": lambda output: isinstance(output, str) and len(output) > 0,
    },
    {
        "name": "Empty query handling",
        "input": {"query": ""},
        "validate": lambda output: output is not None,
    },
]

RAG_TESTS = [
    {
        "name": "Knowledge base query",
        "input": {"query": "What are the pricing details?"},
        "validate": lambda output: isinstance(output, str) and len(output) > 0,
    },
]

CONDITIONAL_TESTS = [
    {
        "name": "Technical branch",
        "input": {"query": "Configure IAM policies", "input_type": "technical"},
        "validate": lambda output: isinstance(output, str),
    },
    {
        "name": "General branch (default)",
        "input": {"query": "Tell me about your company", "input_type": "general"},
        "validate": lambda output: isinstance(output, str),
    },
]
```

---

## Integration Points

- **Upstream**: `mlops/14` → Bedrock Agent ID and alias ID for Agent nodes in flows
- **Upstream**: `mlops/04` → RAG Knowledge Base ID for KnowledgeBase nodes in flows
- **Upstream**: `devops/04` → IAM service role for Bedrock Flows with permissions for models, KBs, Agents, Lambda, S3
- **Downstream**: `devops/10` → OpenTelemetry tracing for flow invocation spans and node-level latency
- **Downstream**: `mlops/17` → LLM evaluation pipeline to assess flow output quality across patterns
