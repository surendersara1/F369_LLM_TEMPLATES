<!-- Template Version: 1.1 | boto3: 1.35+ | strands-agents: 1.x+ | CDK: 2.170+ | Model IDs: 2026-04-22 refresh -->

# Template 20 — Strands Agent Lambda Deployment

## Purpose
Generate a production-ready Strands Agent deployed to AWS Lambda using CDK: Agent initialization with BedrockModel provider and configurable system prompt, MCP tool server integration via MCPClient, conversation management with SlidingWindowConversationManager or SummarizingConversationManager, DynamoDB session persistence across invocations, API Gateway (REST or HTTP) integration, Lambda layer packaging with the official strands-agents layer ARN, and retry logic with exponential backoff for cold start and model invocation errors.

---

## Role Definition

You are an expert AWS AI engineer specializing in Strands Agents SDK deployment to AWS Lambda with expertise in:
- Strands Agents SDK: `Agent()` initialization, `BedrockModel()` provider configuration, `MCPClient()` tool server integration, custom `@tool` decorated functions
- Conversation management: `SlidingWindowConversationManager` for fixed-window history, `SummarizingConversationManager` for long conversations with automatic summarization
- AWS Lambda: Python runtime with Lambda layers for `strands-agents` package, cold start optimization, structured logging
- CDK infrastructure: `PythonFunction` construct for Lambda with layers, `RestApi` / `HttpApi` for API Gateway, `Table` for DynamoDB
- DynamoDB session persistence: load/save conversation state across Lambda invocations with TTL-based expiration
- API Gateway: REST API and HTTP API patterns with request/response mapping, CORS configuration, and authorization
- Error handling: retry logic with exponential backoff for Bedrock model invocation errors, structured error responses, MCP server connection fallback
- IAM policies for Bedrock model access (`bedrock:InvokeModel`, `bedrock:InvokeModelWithResponseStream`), DynamoDB read/write, and Lambda execution

Generate complete, production-deployable code.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED - must be a Bedrock supported region]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

AGENT_SYSTEM_PROMPT:    [REQUIRED - system prompt / instructions for the Strands agent]
                        Example: "You are a helpful assistant that answers questions
                        using available tools. Always cite your sources and be concise."

MODEL_ID:               [OPTIONAL: us.anthropic.claude-sonnet-4-7-20260109-v1:0]
                        Options (verify current availability):
                        - us.anthropic.claude-sonnet-4-7-20260109-v1:0 (Claude Sonnet 4.7)
                        - us.anthropic.claude-sonnet-4-7-20260109-v1:0 (Claude Sonnet 4.7)
                        - us.anthropic.claude-haiku-4-5-20251001-v1:0 (Claude Haiku 4.5)
                        - us.amazon.nova-pro-v1:0 (Amazon Nova Pro)
                        - us.amazon.nova-lite-v1:0 (Amazon Nova Lite)

MCP_SERVER_URIS:        [OPTIONAL - JSON list of MCP tool server configurations]
                        Example: [
                            {
                                "command": "uvx",
                                "args": ["awslabs.aws-documentation-mcp-server@latest"]
                            },
                            {
                                "command": "uvx",
                                "args": ["awslabs.s3-mcp-server@latest"]
                            }
                        ]

CONVERSATION_MANAGER_TYPE: [OPTIONAL: sliding_window]
                        Options:
                        - sliding_window — Fixed-size window of recent messages
                          (SlidingWindowConversationManager)
                        - summarizing — Summarizes older messages to stay within
                          token limits (SummarizingConversationManager)

WINDOW_SIZE:            [OPTIONAL: 40 - number of messages to retain in conversation window]

SESSION_TABLE_NAME:     [OPTIONAL: {PROJECT_NAME}-agent-sessions-{ENV}]
                        DynamoDB table name for session persistence

API_TYPE:               [OPTIONAL: rest]
                        Options:
                        - rest — API Gateway REST API (full feature set, request validation)
                        - http — API Gateway HTTP API (lower latency, lower cost)

CUSTOM_TOOLS:           [OPTIONAL - JSON list of custom tool definitions]
                        Example: [
                            {
                                "name": "lookup_order",
                                "description": "Look up order status by order ID",
                                "parameters": {"order_id": "string"}
                            }
                        ]

LAMBDA_TIMEOUT:         [OPTIONAL: 120 - Lambda timeout in seconds, max 900]
LAMBDA_MEMORY:          [OPTIONAL: 512 - Lambda memory in MB]
```

---

## Task

Generate complete Strands Agent Lambda deployment:

```
{PROJECT_NAME}-strands-lambda/
├── cdk/
│   ├── app.py                         # CDK app entry point
│   └── strands_agent_stack.py         # CDK stack: Lambda + API GW + DynamoDB
├── lambda_handler/
│   ├── handler.py                     # Lambda entry point with Agent init
│   ├── agent_config.py                # Agent configuration (model, tools, prompts)
│   ├── mcp_setup.py                   # MCPClient configuration for tool servers
│   └── session_manager.py            # DynamoDB session persistence
├── tools/
│   └── custom_tools.py               # Custom @tool decorated functions
├── tests/
│   └── test_handler.py
└── requirements.txt
```


**strands_agent_stack.py**: CDK stack defining all infrastructure:
- `PythonFunction` for the Lambda handler with the official `strands-agents` Lambda layer ARN, configured with LAMBDA_TIMEOUT and LAMBDA_MEMORY
- `Table` for DynamoDB session persistence with `session_id` as partition key, TTL attribute enabled, on-demand billing, encryption at rest. Table name: `{PROJECT_NAME}-agent-sessions-{ENV}`
- `RestApi` or `HttpApi` (based on API_TYPE) with POST `/invoke` endpoint mapped to the Lambda function, CORS enabled
- IAM role for Lambda with permissions: `bedrock:InvokeModel`, `bedrock:InvokeModelWithResponseStream` on the configured MODEL_ID, DynamoDB read/write on the session table
- Environment variables: `MODEL_ID`, `AGENT_SYSTEM_PROMPT`, `SESSION_TABLE_NAME`, `CONVERSATION_MANAGER_TYPE`, `WINDOW_SIZE`, `AWS_REGION`
- Tags: `Project={PROJECT_NAME}`, `Environment={ENV}`

**app.py**: CDK app entry point that instantiates `StrandsAgentStack` with parameters from environment variables or CDK context.

**handler.py**: Lambda entry point:
- Parse `prompt` and `session_id` from the API Gateway event body (JSON)
- If no `session_id` provided, generate a new UUID
- Load conversation history from DynamoDB via `session_manager.load_session()`
- Initialize `Agent()` with model from `agent_config.get_model()`, system prompt from `AGENT_SYSTEM_PROMPT`, tools from `custom_tools` and optionally `mcp_setup`, and conversation manager from `agent_config.get_conversation_manager()`
- Call `agent(prompt)` to get the response
- Save updated conversation state to DynamoDB via `session_manager.save_session()`
- Return structured JSON response with `response`, `session_id`, and `statusCode: 200`
- On error, return structured error response with `statusCode: 500`, `error` message, and `session_id`
- Include retry logic with exponential backoff for `ThrottlingException` and `ModelTimeoutException`

**agent_config.py**: Agent configuration module:
- `get_model()`: Initialize `BedrockModel(model_id=MODEL_ID, region_name=AWS_REGION, max_tokens=4096, temperature=0.7)`
- `get_conversation_manager()`: Return `SlidingWindowConversationManager(window_size=WINDOW_SIZE)` or `SummarizingConversationManager(window_size=WINDOW_SIZE)` based on `CONVERSATION_MANAGER_TYPE`
- `get_system_prompt()`: Return `AGENT_SYSTEM_PROMPT` from environment variable
- Load all configuration from environment variables with sensible defaults

**mcp_setup.py**: MCP tool server configuration:
- Parse `MCP_SERVER_URIS` from environment variable (JSON list)
- For each MCP server config, create `MCPClient(lambda: StdioServerParameters(command=config["command"], args=config["args"]))` 
- Return list of configured `MCPClient` instances for use as agent tools
- Handle connection failures gracefully — log warning and continue without MCP tools if a server is unreachable

**session_manager.py**: DynamoDB session persistence:
- `load_session(session_id)`: Retrieve conversation history from DynamoDB table, return empty list if session not found or expired
- `save_session(session_id, conversation_history)`: Store conversation history with TTL (default 1800 seconds)
- `delete_session(session_id)`: Remove session data
- Use `{PROJECT_NAME}-agent-sessions-{ENV}` table name from `SESSION_TABLE_NAME` environment variable
- TTL attribute for automatic session expiration

**custom_tools.py**: Custom tool definitions using the `@tool` decorator:
- Example tools demonstrating the `@tool` pattern with docstring descriptions, typed parameters, and return values
- Each tool function decorated with `@tool` from `strands` package
- Tools are imported and passed to `Agent(tools=[...])` in the handler
- Generate tool implementations based on CUSTOM_TOOLS parameter if provided

**test_handler.py**: Unit tests for the Lambda handler:
- Test successful invocation with prompt and session_id
- Test new session creation when no session_id provided
- Test error handling for model invocation failures
- Test session persistence round-trip (save then load)

**requirements.txt**: Python dependencies including `strands-agents`, `boto3`, `aws-cdk-lib` (for CDK stack).

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**Lambda Layer:** Use the official `strands-agents` Lambda layer ARN for the target region. The layer includes `strands-agents`, `boto3`, and core dependencies. Pin to a specific layer version in production to avoid unexpected updates.

**Cold Start:** Lambda cold starts with the Strands layer take ~2-5 seconds. Mitigate with provisioned concurrency for production workloads. Initialize the `BedrockModel` and `Agent` outside the handler function to reuse across warm invocations.

**Conversation Management:** Use `SlidingWindowConversationManager` for short, bounded conversations (default 40 messages). Use `SummarizingConversationManager` for longer conversations where full history exceeds model context limits. Always persist conversation state to DynamoDB between invocations.

**Session Persistence:** DynamoDB table uses on-demand billing with TTL for automatic session cleanup. Default TTL is 1800 seconds (30 min). Increase for long-running conversation workflows. Session data includes serialized conversation history.

**MCP Tools:** MCP server connections are established per invocation. If an MCP server is unreachable, the agent should degrade gracefully — operate without that tool and log a warning. Do not fail the entire invocation for a single MCP server failure.

**API Gateway:** REST API provides request validation, API keys, and usage plans. HTTP API provides lower latency and cost. Both support CORS and Lambda proxy integration. Use REST API for production APIs requiring throttling and API key management.

**Security:** Lambda execution role needs `bedrock:InvokeModel` and `bedrock:InvokeModelWithResponseStream` scoped to the specific MODEL_ID. DynamoDB permissions scoped to the session table. Use VPC endpoints for Bedrock API in private subnets. Enable CloudTrail for all Bedrock API calls.

**Error Responses:** All errors return structured JSON with `statusCode`, `error` message, and `session_id`. Never expose internal stack traces or model error details to API consumers. Log full error details to CloudWatch for debugging.

**Cost:** Billed per Lambda invocation + duration + Bedrock model tokens. Each agent invocation may trigger multiple model calls (agent loop iterations). Monitor with CloudWatch metrics: `Invocations`, `Duration`, `Errors`, and custom token usage metrics.

**Naming:** All resources follow `{PROJECT_NAME}-{component}-{ENV}` convention:
- Lambda function: `{PROJECT_NAME}-strands-agent-lambda-{ENV}`
- DynamoDB table: `{PROJECT_NAME}-agent-sessions-{ENV}`
- API Gateway: `{PROJECT_NAME}-agent-api-{ENV}`
- IAM role: `{PROJECT_NAME}-strands-agent-role-{ENV}`

---

## Code Scaffolding Hints

**BedrockModel initialization:**
```python
from strands.models.bedrock import BedrockModel

model = BedrockModel(
    model_id="us.anthropic.claude-sonnet-4-7-20260109-v1:0",
    region_name="us-east-1",
    max_tokens=4096,
    temperature=0.7,
)
```

**Agent initialization with conversation manager:**
```python
from strands import Agent
from strands.agent.conversation_manager import SlidingWindowConversationManager

agent = Agent(
    model=model,
    system_prompt="You are a helpful assistant...",
    tools=[tool_1, tool_2],
    conversation_manager=SlidingWindowConversationManager(window_size=40),
)

response = agent("What can you help me with?")
print(response)
```

**MCPClient for tool servers:**
```python
from strands.tools.mcp import MCPClient
from mcp import StdioServerParameters

mcp_client = MCPClient(lambda: StdioServerParameters(
    command="uvx",
    args=["awslabs.aws-documentation-mcp-server@latest"],
))

with mcp_client:
    agent = Agent(model=model, tools=[mcp_client])
    response = agent("What is Amazon Bedrock?")
```

**Custom tool with @tool decorator:**
```python
from strands import tool

@tool
def lookup_order(order_id: str) -> dict:
    """Look up order status by order ID.

    Args:
        order_id: The unique order identifier.

    Returns:
        Order details including status, items, and shipping info.
    """
    # Business logic here
    return {"order_id": order_id, "status": "shipped", "eta": "2025-01-15"}
```

**Lambda handler pattern:**
```python
import json
import os
import uuid
from strands import Agent
from strands.models.bedrock import BedrockModel
from strands.agent.conversation_manager import SlidingWindowConversationManager

# Initialize outside handler for warm start reuse
model = BedrockModel(
    model_id=os.environ.get("MODEL_ID", "us.anthropic.claude-sonnet-4-7-20260109-v1:0"),
    region_name=os.environ.get("AWS_REGION", "us-east-1"),
    max_tokens=4096,
)

SYSTEM_PROMPT = os.environ.get("AGENT_SYSTEM_PROMPT", "You are a helpful assistant.")
WINDOW_SIZE = int(os.environ.get("WINDOW_SIZE", "40"))


def lambda_handler(event, context):
    body = json.loads(event.get("body", "{}"))
    prompt = body.get("prompt", "")
    session_id = body.get("session_id", str(uuid.uuid4()))

    try:
        # Load session state from DynamoDB
        session = load_session(session_id)

        agent = Agent(
            model=model,
            system_prompt=SYSTEM_PROMPT,
            tools=tools,
            conversation_manager=SlidingWindowConversationManager(window_size=WINDOW_SIZE),
        )

        response = agent(prompt)

        # Save session state
        save_session(session_id, agent.conversation_manager)

        return {
            "statusCode": 200,
            "headers": {"Content-Type": "application/json"},
            "body": json.dumps({
                "response": str(response),
                "session_id": session_id,
            }),
        }

    except Exception as e:
        return {
            "statusCode": 500,
            "headers": {"Content-Type": "application/json"},
            "body": json.dumps({
                "error": "An error occurred processing your request.",
                "session_id": session_id,
            }),
        }
```

**DynamoDB session persistence:**
```python
import boto3
import time
import os

dynamodb = boto3.resource("dynamodb", region_name=os.environ.get("AWS_REGION"))
table = dynamodb.Table(os.environ.get("SESSION_TABLE_NAME"))


def load_session(session_id):
    """Load conversation history from DynamoDB."""
    response = table.get_item(Key={"session_id": session_id})
    item = response.get("Item")
    if item and item.get("ttl", 0) > time.time():
        return item.get("conversation_history", [])
    return []


def save_session(session_id, conversation_manager, ttl_seconds=1800):
    """Save conversation history to DynamoDB with TTL."""
    table.put_item(
        Item={
            "session_id": session_id,
            "conversation_history": conversation_manager.conversation_history,
            "ttl": int(time.time()) + ttl_seconds,
            "updated_at": int(time.time()),
        }
    )


def delete_session(session_id):
    """Delete session data from DynamoDB."""
    table.delete_item(Key={"session_id": session_id})
```

**Retry logic with exponential backoff:**
```python
import time
import logging

logger = logging.getLogger(__name__)


def invoke_with_retry(agent, prompt, max_retries=3, base_delay=1.0):
    """Invoke agent with exponential backoff retry for transient errors."""
    for attempt in range(max_retries + 1):
        try:
            return agent(prompt)
        except Exception as e:
            error_name = type(e).__name__
            if error_name in ("ThrottlingException", "ModelTimeoutException",
                              "ServiceUnavailableException") and attempt < max_retries:
                delay = base_delay * (2 ** attempt)
                logger.warning(
                    "Retry %d/%d after %s: %s (waiting %.1fs)",
                    attempt + 1, max_retries, error_name, str(e), delay,
                )
                time.sleep(delay)
            else:
                raise
```

**CDK stack (PythonFunction + API Gateway + DynamoDB):**
```python
from aws_cdk import (
    Stack, Duration, RemovalPolicy, CfnOutput,
    aws_lambda as _lambda,
    aws_apigateway as apigw,
    aws_dynamodb as dynamodb,
    aws_iam as iam,
)
from constructs import Construct


class StrandsAgentStack(Stack):
    def __init__(self, scope: Construct, id: str, **kwargs):
        super().__init__(scope, id, **kwargs)

        project_name = self.node.try_get_context("project_name")
        env_name = self.node.try_get_context("env")

        # DynamoDB session table
        session_table = dynamodb.Table(
            self, "SessionTable",
            table_name=f"{project_name}-agent-sessions-{env_name}",
            partition_key=dynamodb.Attribute(
                name="session_id", type=dynamodb.AttributeType.STRING
            ),
            billing_mode=dynamodb.BillingMode.PAY_PER_REQUEST,
            removal_policy=RemovalPolicy.DESTROY,
            time_to_live_attribute="ttl",
            encryption=dynamodb.TableEncryption.AWS_MANAGED,
        )

        # Strands agent Lambda layer
        strands_layer = _lambda.LayerVersion.from_layer_version_arn(
            self, "StrandsLayer",
            layer_version_arn="arn:aws:lambda:{region}:XXX:layer:strands-agents:1",
        )

        # Lambda function
        agent_fn = _lambda.Function(
            self, "AgentFunction",
            function_name=f"{project_name}-strands-agent-lambda-{env_name}",
            runtime=_lambda.Runtime.PYTHON_3_13,
            handler="handler.lambda_handler",
            code=_lambda.Code.from_asset("lambda_handler"),
            layers=[strands_layer],
            timeout=Duration.seconds(120),
            memory_size=512,
            environment={
                "MODEL_ID": "us.anthropic.claude-sonnet-4-7-20260109-v1:0",
                "AGENT_SYSTEM_PROMPT": "You are a helpful assistant.",
                "SESSION_TABLE_NAME": session_table.table_name,
                "CONVERSATION_MANAGER_TYPE": "sliding_window",
                "WINDOW_SIZE": "40",
            },
        )

        # Grant DynamoDB access
        session_table.grant_read_write_data(agent_fn)

        # Grant Bedrock model access
        agent_fn.add_to_role_policy(iam.PolicyStatement(
            actions=["bedrock:InvokeModel", "bedrock:InvokeModelWithResponseStream"],
            resources=[f"arn:aws:bedrock:*::foundation-model/*"],
        ))

        # API Gateway
        api = apigw.RestApi(
            self, "AgentApi",
            rest_api_name=f"{project_name}-agent-api-{env_name}",
            default_cors_preflight_options=apigw.CorsOptions(
                allow_origins=apigw.Cors.ALL_ORIGINS,
                allow_methods=["POST", "OPTIONS"],
            ),
        )

        invoke_resource = api.root.add_resource("invoke")
        invoke_resource.add_method(
            "POST",
            apigw.LambdaIntegration(agent_fn),
        )

        CfnOutput(self, "ApiUrl", value=api.url)
        CfnOutput(self, "SessionTableName", value=session_table.table_name)
```

---

## Integration Points

- **Upstream**: `devops/04` → IAM roles for Lambda execution role and Bedrock model access policies
- **Upstream**: `iac/02` → CDK patterns and stack conventions for Lambda + API Gateway + DynamoDB infrastructure
- **Upstream**: `mlops/12` → Bedrock Guardrail ID for content filtering on agent model invocations
- **Upstream**: `mlops/23` → Agent SOPs loaded as system prompts for structured agent workflows
- **Upstream**: `mlops/24` → Managed prompt versions retrieved from Bedrock Prompt Management for agent system prompts
- **Downstream**: `mlops/21` → Multi-agent patterns that deploy individual agents as Lambda functions using this template
- **Downstream**: `devops/15` → Agent observability with OpenTelemetry tracing and CloudWatch metrics for this Lambda agent
- **Downstream**: `devops/16` → Agent guardrails and control policies applied to this Lambda agent at runtime
- **Alternative to**: `mlops/22` → AgentCore deployment for long-running, stateful agents with microVM session isolation (use this template for event-driven, short-duration agents)
- **Comparison**: `mlops/14` → Bedrock Agents with Action Groups uses the managed Bedrock Agent service; this template uses the open-source Strands SDK for full control over agent logic
