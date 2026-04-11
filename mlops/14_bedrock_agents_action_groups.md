<!-- Template Version: 1.0 | boto3: 1.35+ -->

# Template 14 — Bedrock Agents with Action Groups

## Purpose
Generate a production-ready Amazon Bedrock Agent with Action Groups: agent creation with configurable instructions and guardrails, Lambda-backed action groups with OpenAPI schemas, DynamoDB session persistence, streaming response handling with trace logging, and multi-agent supervisor collaboration pattern.

---

## Role Definition

You are an expert AWS AI engineer specializing in Amazon Bedrock Agents with expertise in:
- Bedrock Agents: agent creation, action groups, knowledge base attachment, session management
- Action Groups: Lambda function handlers, OpenAPI 3.0 schema definitions, function schema definitions
- Multi-agent collaboration: supervisor agents, collaborator routing, conversation history sharing
- Bedrock Agent Runtime: invoke_agent with streaming responses, trace event parsing, session state
- DynamoDB for conversation session persistence and context management
- IAM policies for Bedrock Agent service roles, Lambda execution roles, and cross-service access
- Guardrail integration for safe agent responses
- Error handling patterns for graceful degradation in agentic workflows

Generate complete, production-deployable code.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED - must be a Bedrock Agents supported region]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

AGENT_MODEL:            [OPTIONAL: anthropic.claude-3-5-sonnet-20241022-v2:0]
                        Options (verify current availability):
                        - anthropic.claude-3-5-sonnet-20241022-v2:0 (Claude 3.5 Sonnet v2)
                        - anthropic.claude-3-5-haiku-20241022-v1:0 (Claude 3.5 Haiku)
                        - anthropic.claude-3-haiku-20240307-v1:0 (Claude 3 Haiku)
                        - amazon.nova-pro-v1:0 (Amazon Nova Pro)
                        - amazon.nova-lite-v1:0 (Amazon Nova Lite)

AGENT_INSTRUCTION:      [REQUIRED - system prompt / instructions for the agent]
                        Example: "You are a customer support agent. Help users check order
                        status, create support tickets, and answer product questions using
                        the available tools. Always be polite and concise."

ACTION_GROUPS:          [REQUIRED - JSON list of action groups to create]
                        Example: [
                            {
                                "name": "order_management",
                                "description": "Look up order status and manage returns",
                                "operations": [
                                    {"name": "getOrderStatus", "description": "Get order status by order ID",
                                     "parameters": {"orderId": {"type": "string", "required": true}}},
                                    {"name": "initiateReturn", "description": "Start a return for an order",
                                     "parameters": {"orderId": {"type": "string", "required": true},
                                                    "reason": {"type": "string", "required": true}}}
                                ]
                            },
                            {
                                "name": "ticket_management",
                                "description": "Create and update support tickets",
                                "operations": [
                                    {"name": "createTicket", "description": "Create a support ticket",
                                     "parameters": {"subject": {"type": "string", "required": true},
                                                    "description": {"type": "string", "required": true},
                                                    "priority": {"type": "string", "required": false}}}
                                ]
                            }
                        ]

KNOWLEDGE_BASE_ID:      [OPTIONAL - Bedrock Knowledge Base ID from mlops/04 RAG pipeline]
GUARDRAIL_ID:           [OPTIONAL - Bedrock Guardrail ID from mlops/12]
GUARDRAIL_VERSION:      [OPTIONAL: DRAFT]

SESSION_TTL:            [OPTIONAL: 1800 - idle session TTL in seconds]
SESSION_PERSISTENCE:    [OPTIONAL: true - enable DynamoDB session persistence]

MULTI_AGENT_ENABLED:    [OPTIONAL: false - enable multi-agent supervisor pattern]
COLLABORATOR_AGENTS:    [OPTIONAL - JSON list of sub-agent configs for multi-agent mode]
                        Example: [
                            {"name": "order_specialist", "agent_id": "AGENT_ID_1",
                             "alias_id": "ALIAS_ID_1",
                             "instruction": "Handles all order-related queries"},
                            {"name": "billing_specialist", "agent_id": "AGENT_ID_2",
                             "alias_id": "ALIAS_ID_2",
                             "instruction": "Handles billing and payment queries"}
                        ]

MEMORY_ENABLED:         [OPTIONAL: false - enable cross-session memory]
MEMORY_STORAGE_DAYS:    [OPTIONAL: 30 - days to retain session summaries]
```

---

## Task

Generate complete Bedrock Agent with Action Groups:

```
{PROJECT_NAME}-bedrock-agent/
├── agent/
│   ├── create_agent.py                # Create Bedrock Agent with guardrail + KB
│   ├── prepare_and_deploy.py          # Prepare agent, create alias for deployment
│   └── config.py                      # All configuration in one place
├── action_groups/
│   ├── create_action_groups.py        # Create action groups with OpenAPI schemas
│   ├── schemas/
│   │   └── {action_group_name}_openapi.json   # OpenAPI 3.0 schema per action group
│   └── lambda_handlers/
│       └── {action_group_name}/
│           └── handler.py             # Lambda handler per action group
├── session/
│   ├── session_store.py               # DynamoDB session persistence layer
│   └── create_session_table.py        # Create DynamoDB table for sessions
├── multi_agent/
│   ├── create_supervisor.py           # Supervisor agent with collaborator association
│   └── invoke_supervisor.py           # Invoke supervisor for multi-agent routing
├── invoke/
│   ├── invoke_agent.py                # Invoke agent with streaming + trace logging
│   └── api_handler.py                 # Lambda handler for API Gateway integration
├── cleanup/
│   └── delete_resources.py            # Delete agent, action groups, aliases
├── run_setup.py                       # CLI orchestrator
└── requirements.txt
```


**config.py**: Central configuration dataclass with all parameters. Load from environment variables or CLI args. Validate required fields. Construct resource names using `{PROJECT_NAME}-{component}-{ENV}` convention.

**create_agent.py**: Create Bedrock Agent using `bedrock_agent.create_agent()`:
- Set `foundationModel`, `instruction`, `idleSessionTTLInSeconds`
- Attach guardrail via `guardrailConfiguration` if GUARDRAIL_ID provided
- Enable memory via `memoryConfiguration` with `SESSION_SUMMARY` type if MEMORY_ENABLED
- Set `agentCollaboration` to `SUPERVISOR` or `SUPERVISOR_ROUTER` if MULTI_AGENT_ENABLED
- Tag with Project and Environment
- After creation, attach knowledge base via `bedrock_agent.associate_agent_knowledge_base()` if KNOWLEDGE_BASE_ID provided

**prepare_and_deploy.py**: Prepare agent with `bedrock_agent.prepare_agent()`, poll until status is `PREPARED`, then create alias with `bedrock_agent.create_agent_alias()` for versioned deployment. Use DRAFT version for dev, numbered alias for stage/prod.

**create_action_groups.py**: For each action group in ACTION_GROUPS:
1. Generate OpenAPI 3.0 JSON schema from the operations definition
2. Deploy Lambda handler function (or reference existing ARN)
3. Call `bedrock_agent.create_agent_action_group()` with `apiSchema` (inline payload) and `actionGroupExecutor` (Lambda ARN)
4. Also create `AMAZON.UserInput` action group to allow agent to ask clarifying questions

**{action_group_name}_openapi.json**: OpenAPI 3.0 schema defining paths, operations, parameters, and response schemas for each action group. Each operation maps to a Lambda handler function.

**handler.py** (per action group): Lambda function that receives the agent's action group invocation event:
- Parse `apiPath`, `httpMethod`, `parameters`, and `requestBody` from the event
- Route to the appropriate business logic function based on the operation
- Return structured response with `responseBody` containing `TEXT` content type
- On error, return structured error response with `actionGroup`, `apiPath`, and error details so the agent can gracefully degrade

**session_store.py**: DynamoDB-backed session persistence:
- `save_session(session_id, attributes)`: Store session attributes with TTL
- `get_session(session_id)`: Retrieve session attributes
- `update_session(session_id, key, value)`: Update individual attribute
- `delete_session(session_id)`: Clean up session
- TTL set to SESSION_TTL for automatic expiration

**create_session_table.py**: Create DynamoDB table `{PROJECT_NAME}-{ENV}-agent-sessions` with `session_id` as partition key, TTL attribute enabled, on-demand billing, encryption at rest.

**create_supervisor.py** (when MULTI_AGENT_ENABLED):
- Create supervisor agent with `agentCollaboration="SUPERVISOR"` (or `"SUPERVISOR_ROUTER"` for lower latency)
- For each collaborator in COLLABORATOR_AGENTS, call `bedrock_agent.associate_agent_collaborator()` with `collaboratorName`, `agentDescriptor` (alias ARN), `collaborationInstruction`, and `relayConversationHistory`
- Prepare and deploy supervisor agent

**invoke_agent.py**: Invoke agent with streaming response handling:
- Call `bedrock_agent_runtime.invoke_agent()` with `agentId`, `agentAliasId`, `sessionId`, `inputText`, `enableTrace=True`
- Configure `streamingConfigurations` with `streamFinalResponse=True`
- Iterate over `completion` events, collecting `chunk` bytes and logging `trace` events
- Pass `sessionState` with `sessionAttributes` and `promptSessionAttributes` for context persistence
- Log orchestration traces (PreProcessingTrace, OrchestrationTrace, PostProcessingTrace) to CloudWatch

**invoke_supervisor.py**: Same as invoke_agent.py but targets the supervisor agent. The supervisor routes to collaborator agents automatically based on the user query.

**api_handler.py**: Lambda handler for API Gateway integration. Accept POST with `{"prompt": "...", "session_id": "..."}`, invoke agent, return streamed response. Load/save session attributes from DynamoDB.

**delete_resources.py**: Clean up all resources: delete agent aliases, disassociate collaborators, delete action groups, delete agent. Delete DynamoDB session table if SESSION_PERSISTENCE enabled.

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**Agent:** Use DRAFT version during development. Create numbered alias for stage/prod deployment. Always attach a guardrail to every agent in production. Agent instructions should be clear and specific — vague instructions lead to poor tool selection.

**Action Groups:** Each action group must have either an OpenAPI schema (`apiSchema`) or a function schema (`functionSchema`), plus a Lambda executor. OpenAPI schemas must be valid OpenAPI 3.0. Lambda handlers must return the response in the exact format Bedrock expects (`responseBody` with `TEXT` content type).

**Session Management:** Default idle session TTL is 1800 seconds (30 min). For long conversations, increase TTL. Use DynamoDB for session persistence across Lambda cold starts. Enable `SESSION_SUMMARY` memory type for cross-session context retention.

**Multi-Agent:** Maximum 10 collaborator agents per supervisor. Each collaborator must be deployed (have an alias) before association. Use `SUPERVISOR_ROUTER` mode for lower latency when collaborators can independently answer queries. Use `SUPERVISOR` mode when the supervisor needs to synthesize responses from multiple collaborators.

**Security:** Agent service role needs `bedrock:InvokeModel`, `bedrock:InvokeModelWithResponseStream` for the configured model. Lambda execution roles need only the permissions required for their specific business logic. Use VPC endpoints for Bedrock API in private subnets. Enable CloudTrail for all Bedrock Agent API calls.

**Cost:** Agents are billed per invocation plus underlying model cost. Each action group invocation triggers a Lambda execution. Multi-agent patterns multiply costs — each collaborator invocation is a separate agent invocation. Monitor with CloudWatch metrics: `InvocationCount`, `InvocationLatency`, `InvocationErrors`.

**Naming:** All resources follow `{PROJECT_NAME}-{component}-{ENV}` convention:
- Agent: `{PROJECT_NAME}-agent-{ENV}`
- Action group Lambda: `{PROJECT_NAME}-ag-{action_group_name}-{ENV}`
- DynamoDB table: `{PROJECT_NAME}-agent-sessions-{ENV}`
- Alias: `{ENV}-latest`

---

## Code Scaffolding Hints

**Create agent:**
```python
bedrock_agent = boto3.client("bedrock-agent", region_name=AWS_REGION)

create_params = {
    "agentName": f"{PROJECT_NAME}-agent-{ENV}",
    "foundationModel": AGENT_MODEL,
    "instruction": AGENT_INSTRUCTION,
    "agentResourceRoleArn": agent_role_arn,
    "idleSessionTTLInSeconds": SESSION_TTL,
    "description": f"{PROJECT_NAME} agent for {ENV}",
    "tags": {"Project": PROJECT_NAME, "Environment": ENV},
}

# Attach guardrail if configured
if GUARDRAIL_ID:
    create_params["guardrailConfiguration"] = {
        "guardrailIdentifier": GUARDRAIL_ID,
        "guardrailVersion": GUARDRAIL_VERSION,
    }

# Enable cross-session memory
if MEMORY_ENABLED:
    create_params["memoryConfiguration"] = {
        "enabledMemoryTypes": ["SESSION_SUMMARY"],
        "storageDays": MEMORY_STORAGE_DAYS,
    }

# Enable multi-agent collaboration
if MULTI_AGENT_ENABLED:
    create_params["agentCollaboration"] = "SUPERVISOR"  # or "SUPERVISOR_ROUTER"

response = bedrock_agent.create_agent(**create_params)
agent_id = response["agent"]["agentId"]
```

**Attach knowledge base:**
```python
if KNOWLEDGE_BASE_ID:
    bedrock_agent.associate_agent_knowledge_base(
        agentId=agent_id,
        agentVersion="DRAFT",
        knowledgeBaseId=KNOWLEDGE_BASE_ID,
        description="RAG knowledge base for document Q&A",
    )
```

**Create action group with OpenAPI schema:**
```python
import json

openapi_schema = {
    "openapi": "3.0.0",
    "info": {"title": f"{action_group['name']} API", "version": "1.0.0"},
    "paths": {},
}

for op in action_group["operations"]:
    path = f"/{op['name']}"
    parameters = []
    for param_name, param_config in op.get("parameters", {}).items():
        parameters.append({
            "name": param_name,
            "in": "query",
            "description": param_config.get("description", param_name),
            "required": param_config.get("required", False),
            "schema": {"type": param_config.get("type", "string")},
        })
    openapi_schema["paths"][path] = {
        "get": {
            "operationId": op["name"],
            "summary": op["description"],
            "parameters": parameters,
            "responses": {
                "200": {
                    "description": "Successful response",
                    "content": {"application/json": {"schema": {"type": "object"}}},
                }
            },
        }
    }

response = bedrock_agent.create_agent_action_group(
    agentId=agent_id,
    agentVersion="DRAFT",
    actionGroupName=action_group["name"],
    description=action_group["description"],
    actionGroupExecutor={"lambda_": lambda_arn},
    apiSchema={"payload": json.dumps(openapi_schema)},
)
```

**Create user input action group (allow agent to ask clarifying questions):**
```python
bedrock_agent.create_agent_action_group(
    agentId=agent_id,
    agentVersion="DRAFT",
    actionGroupName="UserInputAction",
    parentActionGroupSignature="AMAZON.UserInput",
    actionGroupState="ENABLED",
)
```

**Prepare and deploy agent:**
```python
import time

bedrock_agent.prepare_agent(agentId=agent_id)

# Poll until agent is prepared
while True:
    agent_status = bedrock_agent.get_agent(agentId=agent_id)
    status = agent_status["agent"]["agentStatus"]
    if status == "PREPARED":
        break
    elif status == "FAILED":
        raise RuntimeError(f"Agent preparation failed: {agent_status['agent'].get('failureReasons')}")
    time.sleep(5)

# Create alias for deployment
alias_response = bedrock_agent.create_agent_alias(
    agentId=agent_id,
    agentAliasName=f"{ENV}-latest",
    description=f"Alias for {ENV} environment",
)
alias_id = alias_response["agentAlias"]["agentAliasId"]
```

**Invoke agent with streaming and trace logging:**
```python
import logging

bedrock_agent_runtime = boto3.client("bedrock-agent-runtime", region_name=AWS_REGION)
logger = logging.getLogger(__name__)

def invoke_agent(agent_id, alias_id, session_id, input_text, session_attributes=None):
    """Invoke Bedrock Agent with streaming response and trace logging."""
    invoke_params = {
        "agentId": agent_id,
        "agentAliasId": alias_id,
        "sessionId": session_id,
        "inputText": input_text,
        "enableTrace": True,
        "streamingConfigurations": {
            "streamFinalResponse": True,
            "applyGuardrailInterval": 20,
        },
    }

    # Pass session state for context persistence
    if session_attributes:
        invoke_params["sessionState"] = {
            "sessionAttributes": session_attributes,
        }

    response = bedrock_agent_runtime.invoke_agent(**invoke_params)

    completion = ""
    traces = []

    for event in response.get("completion"):
        # Collect streamed response chunks
        if "chunk" in event:
            chunk_text = event["chunk"]["bytes"].decode("utf-8")
            completion += chunk_text

        # Log trace events for observability
        if "trace" in event:
            trace_event = event["trace"]["trace"]
            traces.append(trace_event)
            for trace_type, trace_data in trace_event.items():
                logger.info("Trace [%s]: %s", trace_type, json.dumps(trace_data, default=str))

    return {"completion": completion, "traces": traces, "session_id": session_id}
```

**Action group Lambda handler with error handling:**
```python
def lambda_handler(event, context):
    """Lambda handler for Bedrock Agent action group invocation."""
    agent = event.get("agent", {})
    action_group = event.get("actionGroup", "")
    api_path = event.get("apiPath", "")
    http_method = event.get("httpMethod", "")
    parameters = {p["name"]: p["value"] for p in event.get("parameters", [])}
    request_body = event.get("requestBody", {})

    try:
        # Route to business logic based on api_path
        if api_path == "/getOrderStatus":
            result = get_order_status(parameters.get("orderId"))
        elif api_path == "/initiateReturn":
            result = initiate_return(parameters.get("orderId"), parameters.get("reason"))
        elif api_path == "/createTicket":
            result = create_ticket(
                parameters.get("subject"),
                parameters.get("description"),
                parameters.get("priority", "medium"),
            )
        else:
            result = {"error": f"Unknown operation: {api_path}"}

        response_body = {"TEXT": {"body": json.dumps(result)}}

    except Exception as e:
        # Return structured error so the agent can gracefully degrade
        response_body = {
            "TEXT": {
                "body": json.dumps({
                    "error": str(e),
                    "actionGroup": action_group,
                    "apiPath": api_path,
                    "message": "The operation failed. Please try again or ask the user for more information.",
                })
            }
        }

    return {
        "messageVersion": "1.0",
        "response": {
            "actionGroup": action_group,
            "apiPath": api_path,
            "httpMethod": http_method,
            "httpStatusCode": 200,
            "responseBody": response_body,
        },
    }
```

**DynamoDB session persistence:**
```python
import boto3
import time

dynamodb = boto3.resource("dynamodb", region_name=AWS_REGION)
table = dynamodb.Table(f"{PROJECT_NAME}-agent-sessions-{ENV}")

def save_session(session_id, attributes, ttl_seconds=1800):
    table.put_item(
        Item={
            "session_id": session_id,
            "attributes": attributes,
            "ttl": int(time.time()) + ttl_seconds,
            "updated_at": int(time.time()),
        }
    )

def get_session(session_id):
    response = table.get_item(Key={"session_id": session_id})
    item = response.get("Item")
    if item and item.get("ttl", 0) > time.time():
        return item.get("attributes", {})
    return {}
```

**Multi-agent supervisor — associate collaborators:**
```python
# After creating supervisor agent with agentCollaboration="SUPERVISOR"
for collaborator in COLLABORATOR_AGENTS:
    bedrock_agent.associate_agent_collaborator(
        agentId=supervisor_agent_id,
        agentVersion="DRAFT",
        agentDescriptor={
            "aliasArn": f"arn:aws:bedrock:{AWS_REGION}:{AWS_ACCOUNT_ID}:agent-alias/{collaborator['agent_id']}/{collaborator['alias_id']}"
        },
        collaboratorName=collaborator["name"],
        collaborationInstruction=collaborator["instruction"],
        relayConversationHistory="TO_COLLABORATOR",
    )

# Prepare supervisor after associating all collaborators
bedrock_agent.prepare_agent(agentId=supervisor_agent_id)
```

---

## Integration Points

- **Upstream**: `mlops/12` → Bedrock Guardrail ID for safe agent responses
- **Upstream**: `mlops/04` → RAG Knowledge Base ID for document Q&A capability
- **Upstream**: `devops/04` → IAM roles for Bedrock Agent service role and Lambda execution roles
- **Downstream**: `mlops/16` → Bedrock Flows can orchestrate this agent as a node
- **Downstream**: `devops/10` → OpenTelemetry tracing for agent invocation spans
- **Alternative to**: `mlops/12` (agents section) → this template provides deeper Action Group and multi-agent patterns
