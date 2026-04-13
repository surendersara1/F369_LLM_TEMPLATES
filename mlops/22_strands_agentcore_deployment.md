<!-- Template Version: 1.0 | boto3: 1.35+ | strands-agents: 1.x+ | CDK: 2.170+ | bedrock-agentcore: latest -->

# Template 22 — Strands AgentCore Deployment

## Purpose
Generate a production-ready Strands Agent deployed to Amazon Bedrock AgentCore Runtime: Agent packaging and runtime endpoint creation with microVM session isolation, identity provider integration (Cognito, Entra ID, Okta), auto-scaling with configurable min/max instances, health check and rollback patterns, both Python and TypeScript SDK invocation examples, and a CDK stack for supporting resources. This template is the stateful, long-running counterpart to the Lambda deployment in mlops/20.

---

## Role Definition

You are an expert AWS AI engineer specializing in Amazon Bedrock AgentCore Runtime deployment with expertise in:
- Strands Agents SDK: `Agent()` initialization, `BedrockModel()` provider configuration, custom `@tool` decorated functions, session persistence via AgentCore microVM isolation
- Amazon Bedrock AgentCore Runtime: runtime endpoint creation, agent packaging, microVM session management, environment variable configuration, network modes (public/VPC)
- Identity integration: Amazon Cognito user pools, Microsoft Entra ID (Azure AD), and Okta as identity providers for agent endpoint authentication
- Auto-scaling: configurable minimum and maximum instance counts for AgentCore Runtime deployments with warm pool management
- Health checks and rollback: endpoint status monitoring, automated rollback on health check failure, SNS notification for deployment issues
- CDK infrastructure: supporting resources (S3 artifact bucket, IAM roles, SNS topics) for AgentCore deployments
- Python and TypeScript SDK: `bedrock-agentcore` and `bedrock-agentcore-runtime` client usage for deployment and invocation
- IAM policies for AgentCore service roles (`bedrock-agentcore:*`), Bedrock model access (`bedrock:InvokeModel`), and supporting resource permissions

**When to choose AgentCore over Lambda (mlops/20):**
- **Session duration**: AgentCore supports minutes-to-hours sessions with microVM isolation; Lambda is limited to 15 minutes max
- **State management**: AgentCore provides per-session microVM state without external persistence; Lambda requires DynamoDB for session state
- **Identity integration**: AgentCore natively supports Cognito, Entra ID, and Okta; Lambda relies on API Gateway authorizers
- **Scaling model**: AgentCore uses configurable min/max instances with warm pools; Lambda uses automatic concurrency scaling
- **Cost model**: AgentCore bills per-session plus compute time; Lambda bills per-invocation plus duration
- **Best for**: Long conversations, complex multi-step workflows, stateful agent sessions, enterprise identity requirements

Use Lambda (mlops/20) for event-driven, short-duration, stateless agent invocations behind API Gateway.

Generate complete, production-deployable code.


---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED - must be a Bedrock AgentCore supported region]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

AGENT_SYSTEM_PROMPT:    [REQUIRED - system prompt / instructions for the Strands agent]
                        Example: "You are a helpful assistant that manages long-running
                        workflows. Maintain conversation context across interactions
                        and use available tools to complete multi-step tasks."

MODEL_ID:               [OPTIONAL: us.anthropic.claude-sonnet-4-20250514-v1:0]
                        Options (verify current availability):
                        - us.anthropic.claude-sonnet-4-20250514-v1:0 (Claude Sonnet 4)
                        - us.anthropic.claude-3-5-sonnet-20241022-v2:0 (Claude 3.5 Sonnet v2)
                        - us.anthropic.claude-3-5-haiku-20241022-v1:0 (Claude 3.5 Haiku)
                        - us.amazon.nova-pro-v1:0 (Amazon Nova Pro)
                        - us.amazon.nova-lite-v1:0 (Amazon Nova Lite)

IDENTITY_PROVIDER:      [OPTIONAL: none]
                        Options:
                        - none — No identity integration (public endpoint)
                        - cognito — Amazon Cognito user pool authentication
                        - entra_id — Microsoft Entra ID (Azure AD) authentication
                        - okta — Okta authentication

IDENTITY_CONFIG:        [OPTIONAL - JSON identity provider configuration]
                        Cognito example:
                        {
                            "userPoolId": "us-east-1_XXXXXXXXX",
                            "clientId": "xxxxxxxxxxxxxxxxxxxxxxxxxx"
                        }
                        Entra ID example:
                        {
                            "tenantId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
                            "clientId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
                            "authority": "https://login.microsoftonline.com/{tenantId}"
                        }
                        Okta example:
                        {
                            "domain": "your-org.okta.com",
                            "clientId": "xxxxxxxxxxxxxxxxxxxxxxxxxx",
                            "authServerId": "default"
                        }

MIN_INSTANCES:          [OPTIONAL: 0 - minimum AgentCore instances (0 for dev, 1+ for prod)]
MAX_INSTANCES:          [OPTIONAL: 2 - maximum AgentCore instances for auto-scaling]

NETWORK_MODE:           [OPTIONAL: PUBLIC]
                        Options:
                        - PUBLIC — Publicly accessible endpoint
                        - VPC — Private endpoint within a VPC

VPC_CONFIG:             [OPTIONAL - required when NETWORK_MODE is VPC]
                        {
                            "subnetIds": ["subnet-xxx", "subnet-yyy"],
                            "securityGroupIds": ["sg-xxx"]
                        }

CUSTOM_TOOLS:           [OPTIONAL - JSON list of custom tool definitions]
                        Example: [
                            {
                                "name": "query_database",
                                "description": "Query internal database for records",
                                "parameters": {"query": "string", "limit": "int"}
                            }
                        ]

ARTIFACT_BUCKET:        [OPTIONAL: {PROJECT_NAME}-agentcore-artifacts-{ENV}]
                        S3 bucket for agent package artifacts
```


---

## Task

Generate complete Strands Agent AgentCore Runtime deployment:

```
{PROJECT_NAME}-agentcore/
├── agent/
│   ├── agent_app.py                   # Strands Agent application entry point
│   ├── agent_config.py                # Agent configuration (model, tools, prompts)
│   ├── tools/
│   │   └── custom_tools.py           # Custom @tool decorated functions
│   └── requirements.txt              # Agent dependencies (strands-agents, etc.)
├── deployment/
│   ├── deploy_agentcore.py            # AgentCore Runtime endpoint creation
│   ├── identity_setup.py             # Identity provider integration (Cognito/Entra ID/Okta)
│   ├── scaling_config.py             # Auto-scaling configuration
│   └── health_check.py              # Health check and rollback patterns
├── invoke/
│   ├── invoke_agent.py               # Python SDK invocation example
│   └── invoke_agent.ts              # TypeScript SDK invocation example
├── cdk/
│   ├── app.py                        # CDK app entry point
│   └── agentcore_stack.py            # CDK stack for supporting resources
├── tests/
│   └── test_deployment.py
└── requirements.txt
```

**agent_app.py**: Strands Agent application entry point for AgentCore Runtime:
- Initialize `BedrockModel(model_id=MODEL_ID, region_name=AWS_REGION, max_tokens=4096, temperature=0.7)` from environment variables
- Initialize `Agent()` with model, system prompt from `AGENT_SYSTEM_PROMPT`, and tools from `custom_tools`
- Expose the agent as the AgentCore Runtime entry point — AgentCore manages session isolation via microVM, so no external session persistence is needed
- Handle incoming requests by calling `agent(prompt)` and returning the response
- Include structured logging with session ID, request ID, and agent metrics
- On error, return structured error response with error type and safe message (no internal details exposed)

**agent_config.py**: Agent configuration module:
- `get_model()`: Initialize `BedrockModel(model_id=MODEL_ID, region_name=AWS_REGION, max_tokens=4096, temperature=0.7)`
- `get_system_prompt()`: Return `AGENT_SYSTEM_PROMPT` from environment variable
- `get_tools()`: Return list of custom tools from `custom_tools` module
- Load all configuration from environment variables with sensible defaults

**custom_tools.py**: Custom tool definitions using the `@tool` decorator:
- Example tools demonstrating the `@tool` pattern with docstring descriptions, typed parameters, and return values
- Each tool function decorated with `@tool` from `strands` package
- Generate tool implementations based on CUSTOM_TOOLS parameter if provided

**agent/requirements.txt**: Agent-specific dependencies:
- `strands-agents>=1.0.0`
- `boto3>=1.35.0`
- Any additional dependencies for custom tools

**deploy_agentcore.py**: AgentCore Runtime deployment script:
- Package the `agent/` directory into a zip archive and upload to S3 artifact bucket (`ARTIFACT_BUCKET`)
- Call `agentcore_client.create_runtime_endpoint()` with:
  - `name`: `{PROJECT_NAME}-agentcore-{ENV}`
  - `agentRuntimeArtifact`: S3 reference to the uploaded agent package
  - `roleArn`: AgentCore service role ARN from CDK stack output or SSM parameter
  - `networkConfiguration`: `{"networkMode": NETWORK_MODE}` with optional VPC config
  - `environmentVariables`: `MODEL_ID`, `AGENT_SYSTEM_PROMPT`, `AWS_REGION`
- If IDENTITY_PROVIDER is not `none`, attach identity configuration via `identity_setup.py`
- Apply auto-scaling configuration via `scaling_config.py`
- Wait for endpoint status to become `ACTIVE` with polling and timeout
- Output the endpoint ID and invocation URL on success
- On failure, log the error details and exit with non-zero status

**identity_setup.py**: Identity provider integration:
- Parse IDENTITY_PROVIDER and IDENTITY_CONFIG parameters
- For `cognito`: Configure Cognito user pool authentication with `userPoolId` and `clientId`
- For `entra_id`: Configure Microsoft Entra ID authentication with `tenantId`, `clientId`, and `authority` URL
- For `okta`: Configure Okta authentication with `domain`, `clientId`, and `authServerId`
- Return the identity configuration dict for use in `create_runtime_endpoint()` or as a post-deployment update
- Validate identity config fields before applying — raise descriptive errors for missing required fields

**scaling_config.py**: Auto-scaling configuration:
- Configure minimum instances (`MIN_INSTANCES`) and maximum instances (`MAX_INSTANCES`)
- For `dev` environment: default `MIN_INSTANCES=0`, `MAX_INSTANCES=2`
- For `prod` environment: default `MIN_INSTANCES=1`, `MAX_INSTANCES=10`
- Return scaling configuration dict for use in endpoint creation or update
- Include warm pool configuration for production to reduce cold start latency

**health_check.py**: Health check and rollback patterns:
- `check_health(endpoint_id)`: Call `agentcore_client.get_runtime_endpoint(endpointId=endpoint_id)` and check `status` field
- Expected statuses: `ACTIVE` (healthy), `CREATING` (deploying), `UPDATING` (updating), `FAILED` (unhealthy)
- `wait_for_active(endpoint_id, timeout_seconds=600)`: Poll endpoint status until `ACTIVE` or timeout
- `rollback(endpoint_id, previous_artifact_key)`: On health check failure, redeploy with the previous agent package artifact from S3
- `notify_failure(endpoint_id, error_details)`: Publish failure notification to SNS topic `{PROJECT_NAME}-agentcore-alerts-{ENV}`
- Include retry logic for transient API errors during health checks

**invoke_agent.py**: Python SDK invocation example:
- Initialize `bedrock-agentcore-runtime` client
- Call `agentcore_runtime.invoke_agent(endpointId=endpoint_id, sessionId=session_id, inputText=prompt)`
- Handle streaming and non-streaming response modes
- Demonstrate session continuity — multiple invocations with the same `sessionId` maintain conversation context
- Include error handling for endpoint unavailable, throttling, and session timeout errors

**invoke_agent.ts**: TypeScript SDK invocation example:
- Initialize `BedrockAgentCoreRuntimeClient` from `@aws-sdk/client-bedrock-agentcore-runtime`
- Call `InvokeAgentCommand` with `endpointId`, `sessionId`, and `inputText`
- Handle streaming response with async iteration
- Demonstrate session continuity with multiple sequential invocations
- Include error handling with typed catch blocks for `ThrottlingException` and `ResourceNotFoundException`

**agentcore_stack.py**: CDK stack for supporting resources:
- S3 bucket for agent package artifacts: `{PROJECT_NAME}-agentcore-artifacts-{ENV}` with versioning enabled, encryption at rest, lifecycle rules for old versions
- IAM role for AgentCore service: permissions for `bedrock:InvokeModel`, `bedrock:InvokeModelWithResponseStream` on the configured MODEL_ID, S3 read on artifact bucket, CloudWatch Logs write
- SNS topic for deployment alerts: `{PROJECT_NAME}-agentcore-alerts-{ENV}` with email subscription (configurable)
- SSM parameters for endpoint configuration: store endpoint ID, role ARN, and artifact bucket name for cross-stack reference
- Tags: `Project={PROJECT_NAME}`, `Environment={ENV}`

**app.py**: CDK app entry point that instantiates `AgentCoreStack` with parameters from environment variables or CDK context.

**test_deployment.py**: Unit tests for the deployment scripts:
- Test agent package creation and S3 upload
- Test endpoint creation with valid parameters
- Test identity configuration validation for each provider type
- Test health check polling logic
- Test rollback trigger on health check failure

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**Lambda vs AgentCore Decision Matrix:**

| Factor | Lambda (mlops/20) | AgentCore (mlops/22) |
|--------|-------------------|----------------------|
| Session duration | < 15 min | Minutes to hours |
| State management | DynamoDB + conversation manager | MicroVM session isolation |
| Cold start | ~2-5s with layer | Warm pool available |
| Scaling | Automatic (Lambda concurrency) | Configurable min/max instances |
| Identity integration | IAM + API Gateway authorizer | Cognito, Entra ID, Okta native |
| Cost model | Per-invocation + duration | Per-session + compute time |
| Best for | Event-driven, short tasks, APIs | Long conversations, complex workflows |

Choose Lambda when you need event-driven, stateless agent invocations with minimal infrastructure. Choose AgentCore when you need long-running sessions, enterprise identity, or per-session state isolation.

**Agent Packaging:** Package the `agent/` directory as a zip archive containing `agent_app.py`, `agent_config.py`, `tools/`, and `requirements.txt`. Upload to the S3 artifact bucket with versioning enabled. Each deployment creates a new S3 object version, enabling rollback to any previous package.

**Identity Integration:** When IDENTITY_PROVIDER is set, the endpoint requires authenticated requests. Cognito uses JWT tokens from the user pool. Entra ID uses OAuth 2.0 tokens from the Azure AD tenant. Okta uses OAuth 2.0 tokens from the Okta authorization server. Validate identity config fields before deployment — missing fields cause endpoint creation failure.

**Auto-Scaling:** Set `MIN_INSTANCES=0` for dev environments to minimize cost (cold start on first request). Set `MIN_INSTANCES=1+` for production to maintain warm instances. `MAX_INSTANCES` caps the scaling ceiling. AgentCore warm pools reduce cold start latency for production workloads.

**Health Checks:** After endpoint creation or update, poll `get_runtime_endpoint()` until status is `ACTIVE`. If status transitions to `FAILED`, trigger rollback to the previous artifact version. Set a deployment timeout (default 600 seconds) to prevent indefinite waiting. Publish SNS notifications on failure for operational awareness.

**Rollback Pattern:** Maintain the previous S3 object key for the agent package. On health check failure, call `update_runtime_endpoint()` with the previous artifact. If rollback also fails, send critical SNS alert and require manual intervention.

**Network Mode:** Use `PUBLIC` for development and testing. Use `VPC` for production workloads that need to access private resources (databases, internal APIs). VPC mode requires subnet IDs and security group IDs in the VPC_CONFIG parameter.

**Security:** AgentCore service role needs `bedrock:InvokeModel` and `bedrock:InvokeModelWithResponseStream` scoped to the specific MODEL_ID. S3 read permissions scoped to the artifact bucket. CloudWatch Logs write for agent logging. Use least-privilege IAM policies. Enable CloudTrail for all AgentCore API calls.

**Error Responses:** All errors return structured JSON with `status`, `error` message, and `session_id`. Never expose internal stack traces, model error details, or identity configuration to API consumers. Log full error details to CloudWatch for debugging.

**Cost:** Billed per AgentCore session plus compute time. Each session may run for minutes to hours. Monitor with CloudWatch metrics: endpoint invocations, session duration, and error rate. Use `MIN_INSTANCES=0` in non-production to avoid idle compute costs.

**Naming:** All resources follow `{PROJECT_NAME}-{component}-{ENV}` convention:
- AgentCore endpoint: `{PROJECT_NAME}-agentcore-{ENV}`
- S3 artifact bucket: `{PROJECT_NAME}-agentcore-artifacts-{ENV}`
- IAM role: `{PROJECT_NAME}-agentcore-role-{ENV}`
- SNS topic: `{PROJECT_NAME}-agentcore-alerts-{ENV}`
- SSM parameters: `/{PROJECT_NAME}/{ENV}/agentcore/*`

---

## Code Scaffolding Hints

**Strands Agent entry point for AgentCore:**
```python
import os
import logging
from strands import Agent
from strands.models.bedrock import BedrockModel

logger = logging.getLogger(__name__)

# Initialize model and agent at module level for reuse across requests
model = BedrockModel(
    model_id=os.environ.get("MODEL_ID", "us.anthropic.claude-sonnet-4-20250514-v1:0"),
    region_name=os.environ.get("AWS_REGION", "us-east-1"),
    max_tokens=4096,
    temperature=0.7,
)

SYSTEM_PROMPT = os.environ.get("AGENT_SYSTEM_PROMPT", "You are a helpful assistant.")

agent = Agent(
    model=model,
    system_prompt=SYSTEM_PROMPT,
    tools=tools,
)


def handle_request(prompt: str, session_id: str) -> dict:
    """Handle an incoming agent request within the AgentCore microVM session."""
    try:
        # AgentCore manages session isolation — each session runs in its own microVM
        # No external session persistence needed; state is maintained in-process
        response = agent(prompt)

        return {
            "response": str(response),
            "session_id": session_id,
            "status": "success",
        }

    except Exception as e:
        logger.error("Agent invocation failed: %s", str(e), exc_info=True)
        return {
            "error": "An error occurred processing your request.",
            "session_id": session_id,
            "status": "error",
        }
```

**AgentCore Runtime endpoint creation:**
```python
import boto3
import os

AWS_REGION = os.environ.get("AWS_REGION", "us-east-1")
PROJECT_NAME = os.environ.get("PROJECT_NAME")
ENV = os.environ.get("ENV", "dev")

agentcore_client = boto3.client("bedrock-agentcore", region_name=AWS_REGION)

# Create runtime endpoint for Strands agent
response = agentcore_client.create_runtime_endpoint(
    name=f"{PROJECT_NAME}-agentcore-{ENV}",
    agentRuntimeArtifact={
        "s3": {
            "s3BucketName": f"{PROJECT_NAME}-agentcore-artifacts-{ENV}",
            "s3ObjectKey": "agent-package.zip",
        }
    },
    roleArn=f"arn:aws:iam::{AWS_ACCOUNT_ID}:role/{PROJECT_NAME}-agentcore-role-{ENV}",
    networkConfiguration={
        "networkMode": "PUBLIC",  # or "VPC" for private endpoints
    },
    environmentVariables={
        "MODEL_ID": "us.anthropic.claude-sonnet-4-20250514-v1:0",
        "AGENT_SYSTEM_PROMPT": "You are a helpful assistant...",
        "AWS_REGION": AWS_REGION,
    },
)

endpoint_id = response["endpointId"]
print(f"Endpoint created: {endpoint_id}")
```

**Identity provider configuration:**
```python
# Cognito identity configuration
cognito_config = {
    "cognitoConfig": {
        "userPoolId": "us-east-1_XXXXXXXXX",
        "clientId": "xxxxxxxxxxxxxxxxxxxxxxxxxx",
    }
}

# Microsoft Entra ID configuration
entra_id_config = {
    "entraIdConfig": {
        "tenantId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
        "clientId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
        "authority": "https://login.microsoftonline.com/{tenantId}",
    }
}

# Okta configuration
okta_config = {
    "oktaConfig": {
        "domain": "your-org.okta.com",
        "clientId": "xxxxxxxxxxxxxxxxxxxxxxxxxx",
        "authServerId": "default",
    }
}


def get_identity_config(provider: str, config: dict) -> dict:
    """Build identity configuration for the specified provider."""
    if provider == "cognito":
        return {
            "cognitoConfig": {
                "userPoolId": config["userPoolId"],
                "clientId": config["clientId"],
            }
        }
    elif provider == "entra_id":
        return {
            "entraIdConfig": {
                "tenantId": config["tenantId"],
                "clientId": config["clientId"],
                "authority": config.get(
                    "authority",
                    f"https://login.microsoftonline.com/{config['tenantId']}",
                ),
            }
        }
    elif provider == "okta":
        return {
            "oktaConfig": {
                "domain": config["domain"],
                "clientId": config["clientId"],
                "authServerId": config.get("authServerId", "default"),
            }
        }
    else:
        return {}
```

**Auto-scaling configuration:**
```python
def get_scaling_config(env: str, min_instances: int = None, max_instances: int = None) -> dict:
    """Build auto-scaling configuration based on environment."""
    defaults = {
        "dev": {"minInstances": 0, "maxInstances": 2},
        "stage": {"minInstances": 0, "maxInstances": 5},
        "prod": {"minInstances": 1, "maxInstances": 10},
    }

    env_defaults = defaults.get(env, defaults["dev"])

    return {
        "minInstances": min_instances if min_instances is not None else env_defaults["minInstances"],
        "maxInstances": max_instances if max_instances is not None else env_defaults["maxInstances"],
    }
```


**Health check and rollback:**
```python
import boto3
import time
import logging

logger = logging.getLogger(__name__)

agentcore_client = boto3.client("bedrock-agentcore", region_name=AWS_REGION)
sns_client = boto3.client("sns", region_name=AWS_REGION)


def check_health(endpoint_id: str) -> str:
    """Check AgentCore endpoint health status."""
    response = agentcore_client.get_runtime_endpoint(endpointId=endpoint_id)
    status = response["status"]  # ACTIVE, CREATING, UPDATING, FAILED
    logger.info("Endpoint %s status: %s", endpoint_id, status)
    return status


def wait_for_active(endpoint_id: str, timeout_seconds: int = 600, poll_interval: int = 15) -> bool:
    """Poll endpoint status until ACTIVE or timeout."""
    start = time.time()
    while time.time() - start < timeout_seconds:
        status = check_health(endpoint_id)
        if status == "ACTIVE":
            logger.info("Endpoint %s is ACTIVE", endpoint_id)
            return True
        if status == "FAILED":
            logger.error("Endpoint %s has FAILED", endpoint_id)
            return False
        logger.info("Waiting for endpoint %s (status: %s)...", endpoint_id, status)
        time.sleep(poll_interval)

    logger.error("Timeout waiting for endpoint %s to become ACTIVE", endpoint_id)
    return False


def rollback(endpoint_id: str, previous_artifact_key: str):
    """Rollback to previous agent package on health check failure."""
    logger.warning("Rolling back endpoint %s to artifact %s", endpoint_id, previous_artifact_key)
    agentcore_client.update_runtime_endpoint(
        endpointId=endpoint_id,
        agentRuntimeArtifact={
            "s3": {
                "s3BucketName": f"{PROJECT_NAME}-agentcore-artifacts-{ENV}",
                "s3ObjectKey": previous_artifact_key,
            }
        },
    )
    success = wait_for_active(endpoint_id)
    if not success:
        notify_failure(endpoint_id, "Rollback also failed")


def notify_failure(endpoint_id: str, error_details: str):
    """Send SNS notification on deployment failure."""
    topic_arn = f"arn:aws:sns:{AWS_REGION}:{AWS_ACCOUNT_ID}:{PROJECT_NAME}-agentcore-alerts-{ENV}"
    sns_client.publish(
        TopicArn=topic_arn,
        Subject=f"AgentCore Deployment Failed: {endpoint_id}",
        Message=f"Endpoint {endpoint_id} health check failed.\n\nDetails: {error_details}",
    )
```

**Python SDK invocation:**
```python
import boto3

agentcore_runtime = boto3.client("bedrock-agentcore-runtime", region_name=AWS_REGION)


def invoke_agent(endpoint_id: str, session_id: str, prompt: str) -> str:
    """Invoke a deployed AgentCore agent."""
    response = agentcore_runtime.invoke_agent(
        endpointId=endpoint_id,
        sessionId=session_id,
        inputText=prompt,
    )

    # Handle response
    result = response.get("output", {}).get("text", "")
    return result


# Session continuity example — same session_id maintains conversation context
session_id = "user-session-001"
response_1 = invoke_agent(endpoint_id, session_id, "What is Amazon Bedrock?")
print(f"Response 1: {response_1}")

response_2 = invoke_agent(endpoint_id, session_id, "Tell me more about its pricing.")
print(f"Response 2: {response_2}")  # Agent remembers the previous context
```

**TypeScript SDK invocation:**
```typescript
import {
  BedrockAgentCoreRuntimeClient,
  InvokeAgentCommand,
} from "@aws-sdk/client-bedrock-agentcore-runtime";

const client = new BedrockAgentCoreRuntimeClient({ region: "us-east-1" });

async function invokeAgent(
  endpointId: string,
  sessionId: string,
  inputText: string
): Promise<string> {
  const command = new InvokeAgentCommand({
    endpointId,
    sessionId,
    inputText,
  });

  try {
    const response = await client.send(command);
    return response.output?.text ?? "";
  } catch (error) {
    if (error instanceof Error && error.name === "ThrottlingException") {
      console.error("Rate limited — retry with backoff");
      throw error;
    }
    if (error instanceof Error && error.name === "ResourceNotFoundException") {
      console.error("Endpoint not found — verify endpoint ID");
      throw error;
    }
    throw error;
  }
}

// Session continuity example
async function main() {
  const endpointId = "your-endpoint-id";
  const sessionId = "user-session-001";

  const response1 = await invokeAgent(endpointId, sessionId, "What is Amazon Bedrock?");
  console.log("Response 1:", response1);

  const response2 = await invokeAgent(endpointId, sessionId, "Tell me more about pricing.");
  console.log("Response 2:", response2); // Agent remembers previous context
}

main().catch(console.error);
```

**CDK stack for supporting resources:**
```python
from aws_cdk import (
    Stack, RemovalPolicy, CfnOutput,
    aws_s3 as s3,
    aws_iam as iam,
    aws_sns as sns,
    aws_sns_subscriptions as subs,
    aws_ssm as ssm,
)
from constructs import Construct


class AgentCoreStack(Stack):
    def __init__(self, scope: Construct, id: str, **kwargs):
        super().__init__(scope, id, **kwargs)

        project_name = self.node.try_get_context("project_name")
        env_name = self.node.try_get_context("env")

        # S3 bucket for agent package artifacts
        artifact_bucket = s3.Bucket(
            self, "ArtifactBucket",
            bucket_name=f"{project_name}-agentcore-artifacts-{env_name}",
            versioned=True,
            encryption=s3.BucketEncryption.S3_MANAGED,
            removal_policy=RemovalPolicy.RETAIN,
            lifecycle_rules=[
                s3.LifecycleRule(
                    noncurrent_version_expiration=Duration.days(90),
                )
            ],
        )

        # IAM role for AgentCore service
        agentcore_role = iam.Role(
            self, "AgentCoreRole",
            role_name=f"{project_name}-agentcore-role-{env_name}",
            assumed_by=iam.ServicePrincipal("bedrock-agentcore.amazonaws.com"),
        )

        agentcore_role.add_to_policy(iam.PolicyStatement(
            actions=["bedrock:InvokeModel", "bedrock:InvokeModelWithResponseStream"],
            resources=[f"arn:aws:bedrock:*::foundation-model/*"],
        ))

        agentcore_role.add_to_policy(iam.PolicyStatement(
            actions=["s3:GetObject"],
            resources=[artifact_bucket.arn_for_objects("*")],
        ))

        agentcore_role.add_to_policy(iam.PolicyStatement(
            actions=["logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents"],
            resources=["*"],
        ))

        # SNS topic for deployment alerts
        alerts_topic = sns.Topic(
            self, "AlertsTopic",
            topic_name=f"{project_name}-agentcore-alerts-{env_name}",
        )

        # SSM parameters for cross-stack reference
        ssm.StringParameter(
            self, "RoleArnParam",
            parameter_name=f"/{project_name}/{env_name}/agentcore/role-arn",
            string_value=agentcore_role.role_arn,
        )

        ssm.StringParameter(
            self, "ArtifactBucketParam",
            parameter_name=f"/{project_name}/{env_name}/agentcore/artifact-bucket",
            string_value=artifact_bucket.bucket_name,
        )

        CfnOutput(self, "ArtifactBucketName", value=artifact_bucket.bucket_name)
        CfnOutput(self, "AgentCoreRoleArn", value=agentcore_role.role_arn)
        CfnOutput(self, "AlertsTopicArn", value=alerts_topic.topic_arn)
```


---

## Integration Points

- **Upstream**: `devops/04` → IAM roles for AgentCore service role and Bedrock model access policies
- **Upstream**: `iac/02` → CDK patterns and stack conventions for S3 artifact bucket, IAM roles, and SNS topics
- **Upstream**: `mlops/23` → Agent SOPs loaded as system prompts for structured agent workflows running on AgentCore
- **Upstream**: `mlops/24` → Managed prompt versions retrieved from Bedrock Prompt Management for agent system prompts
- **Downstream**: `devops/15` → Agent observability with OpenTelemetry tracing and CloudWatch metrics for this AgentCore agent
- **Downstream**: `devops/16` → Agent guardrails and control policies applied to this AgentCore agent at runtime
- **Alternative to**: `mlops/20` → Lambda deployment for event-driven, short-duration agents (use this template for long-running, stateful agents with session isolation and identity integration)
- **Comparison**: `mlops/14` → Bedrock Agents with Action Groups uses the managed Bedrock Agent service; this template uses the open-source Strands SDK on AgentCore Runtime for full control over agent logic with microVM isolation
