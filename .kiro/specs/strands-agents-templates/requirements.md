# Requirements Document

## Introduction

This document specifies requirements for expanding the F369 AWS MLOps & AI/LLM CI/CD Prompt Template Library with 7 new templates covering the Strands Agents open-source SDK ecosystem and related AWS services. The expansion adds templates for Strands Agent Lambda deployment (mlops/20), multi-agent patterns (mlops/21), Bedrock AgentCore deployment (mlops/22), Agent SOP authoring (mlops/23), Bedrock Prompt Management (mlops/24), agent observability (devops/15), and agent guardrails/control (devops/16). These templates integrate with the existing library (currently 67 templates across 8 directories) and follow the established F369 format. All templates target the Strands Agents SDK (Python), boto3 1.35+, and CDK 2.170+.

## Glossary

- **Template**: A markdown file following the F369 format (Version comment, Purpose, Role Definition, Context & Inputs, Task, Output Format, Requirements & Constraints, Code Scaffolding Hints, Integration Points) that users paste into an LLM to generate deployable AWS infrastructure code.
- **Template_Library**: The collection of all F369 prompt template markdown files organized into category directories.
- **Library_Index**: The README.md and Library.md files that catalog all templates with descriptions and recommended build order.
- **PROMPT_GUIDE**: The PROMPT_GUIDE.md file that documents template anatomy, chaining patterns, and usage instructions.
- **Integration_Points_Section**: The section in each template that documents upstream dependencies and downstream consumers using the format `category/number` (e.g., `mlops/12`, `devops/10`).
- **Context_Inputs_Section**: The parameterized section at the top of each template where users fill in `[REQUIRED]` and `[OPTIONAL]` values before prompting.
- **Code_Scaffolding_Hints_Section**: The section providing concrete AWS SDK calls, API patterns, and code snippets that guide the LLM toward correct implementations.
- **Task_Section**: The section defining the project directory structure and file-by-file generation instructions.
- **Strands_SDK**: The Strands Agents open-source SDK (Python package `strands-agents`) that provides a model-driven approach to building AI agents with support for multiple model providers, tools, and deployment targets.
- **Strands_Agent**: An agent instance created via `strands.Agent()` with a model provider, system prompt, tools, and optional conversation manager.
- **Strands_Model_Provider**: A model backend for Strands agents, including `BedrockModel`, `AnthropicModel`, `OpenAIModel`, `OllamaModel`, and others.
- **MCP_Client**: A Model Context Protocol client (`strands.tools.mcp.MCPClient`) that connects Strands agents to MCP-compatible tool servers.
- **Strands_Community_Tools**: The `strands-agents-tools` package providing pre-built tools including `retrieve`, `memory`, `editor`, `shell`, `browser`, `use_aws`, `graph`, `swarm`, `workflow`, `a2a_client`, `think`, `batch`, and `handoff_to_user`.
- **Multi_Agent_Pattern**: An architectural pattern for coordinating multiple Strands agents, including Graph (DAG execution), Swarm (dynamic handoff), and Workflow (sequential pipeline).
- **A2A_Protocol**: The Agent-to-Agent protocol enabling interoperability between agents across different frameworks and deployments.
- **AgentCore_Runtime**: Amazon Bedrock AgentCore Runtime, a serverless runtime for deploying Strands agents with microVM session isolation, identity integration, and auto-scaling.
- **Agent_SOP**: A Standard Operating Procedure written in structured markdown with RFC 2119 keywords (MUST, SHOULD, MAY) that serves as a natural language workflow for AI agents.
- **Agent_Control**: Strands Agent Control, a runtime guardrails mechanism that defines what agents can and cannot do without code changes.
- **Conversation_Manager**: A Strands component that manages conversation history, including `SlidingWindowConversationManager` and `SummarizingConversationManager`.
- **Invocation_State**: Shared state dictionary passed between agents in multi-agent patterns via `agent.invocation_state` for cross-agent context sharing.
- **Bedrock_Prompt_Management**: The Amazon Bedrock Prompt Management API for creating, versioning, and managing prompt templates with variables and variants.
- **Lambda_Layer**: An AWS Lambda layer containing the `strands-agents` package and dependencies, used for deploying Strands agents to Lambda functions.

## Requirements

---

### Requirement 1: Template Format Compliance for Strands Templates

**User Story:** As a template library maintainer, I want all 7 new Strands-related templates to follow the established F369 format, so that users have a consistent experience across the entire library.

#### Acceptance Criteria

1. THE Template_Library SHALL include the following sections in every new Strands template file in this exact order: Template Version comment, Title and Purpose, Role Definition, Context & Inputs, Task (with project structure and file-by-file instructions), Output Format, Requirements & Constraints, Code Scaffolding Hints, and Integration Points.
2. WHEN a new Strands template is created, THE Template SHALL begin with an HTML comment in the format `<!-- Template Version: 1.0 | boto3: 1.35+ | strands-agents: 1.x+ | [additional tool versions] -->`.
3. THE Context_Inputs_Section SHALL include `PROJECT_NAME`, `AWS_REGION`, `AWS_ACCOUNT_ID`, and `ENV` as the first four required parameters.
4. THE Context_Inputs_Section SHALL mark every parameter as either `[REQUIRED]` or `[OPTIONAL: default_value]` with clear descriptions.
5. THE Task_Section SHALL include an ASCII directory tree showing the complete project structure that the LLM will generate.
6. THE Task_Section SHALL include file-by-file descriptions with concrete Strands SDK calls, AWS API calls, and configuration patterns for each generated file.
7. THE Integration_Points_Section SHALL reference upstream dependencies and downstream consumers using the `category/number` format (e.g., `mlops/12`, `devops/04`) with a one-line description of the relationship.
8. THE Code_Scaffolding_Hints_Section SHALL include working code snippets using `strands-agents` SDK, boto3 1.35+, or CDK 2.170+ that are validated against current API signatures.
9. THE Template SHALL use the naming convention `{PROJECT_NAME}-{component}-{ENV}` for all generated AWS resource names.

---

### Requirement 2: Library Index and Documentation Updates

**User Story:** As a library user, I want the README.md, Library.md, and PROMPT_GUIDE.md to reflect all 7 new Strands templates, so that I can discover and chain templates across the expanded library.

#### Acceptance Criteria

1. WHEN all 7 new templates are added, THE Library_Index SHALL update README.md to include all new templates in the Template Map with file path, number, and one-line description.
2. WHEN all 7 new templates are added, THE Library_Index SHALL update Library.md to include the new entries in the directory tree showing all 74 templates across 8 directories.
3. THE Library_Index SHALL update the Recommended Build Order in README.md to incorporate Strands agent deployment, multi-agent patterns, AgentCore, SOPs, prompt management, observability, and guardrails at appropriate positions after the existing Agentic AI section.
4. THE Library_Index SHALL update the Architecture Overview ASCII diagram in README.md to show the Strands Agents ecosystem as a sub-layer within the MLOps Core and Observability layers.
5. THE PROMPT_GUIDE SHALL update the Template Interaction Map to include cross-references for all 7 new templates.
6. THE PROMPT_GUIDE SHALL add a new chaining example that demonstrates a Strands agent workflow: Agent SOP authoring → Strands Lambda deployment → multi-agent orchestration → observability → guardrails.

---

### Requirement 3: Strands Agent Lambda Deployment (mlops/20)

**User Story:** As an AI engineer, I want a template for deploying Strands agents to AWS Lambda using CDK, so that I can run serverless agents with MCP tools, Bedrock model provider, and conversation management.

#### Acceptance Criteria

1. THE Template SHALL generate a complete Strands Agent Lambda deployment with CDK stack, Lambda handler, Strands Agent configuration, and BedrockModel provider setup.
2. THE Template SHALL generate a Lambda handler that initializes a `strands.Agent()` with configurable system prompt, `BedrockModel()` provider, and tool list.
3. WHEN MCP tool server URIs are provided, THE Template SHALL generate `MCPClient()` configuration that connects the agent to specified MCP servers.
4. THE Template SHALL generate a CDK stack using `PythonFunction` or equivalent construct that includes the official Strands Lambda layer ARN for the `strands-agents` package.
5. THE Template SHALL include conversation management configuration using `SlidingWindowConversationManager` or `SummarizingConversationManager` with configurable window size.
6. THE Template SHALL generate DynamoDB table configuration for session persistence across Lambda invocations.
7. THE Template SHALL generate API Gateway (REST or HTTP API) integration for invoking the agent Lambda function with request/response mapping.
8. IF the Lambda function encounters a cold start timeout or model invocation error, THEN THE Template SHALL include retry logic with exponential backoff and structured error responses.
9. THE Code_Scaffolding_Hints_Section SHALL include working `Agent()`, `BedrockModel()`, `MCPClient()`, and Lambda handler code patterns from the Strands SDK.
10. THE Integration_Points_Section SHALL reference `devops/04` (IAM roles), `iac/02` (CDK patterns), `mlops/12` (Bedrock Guardrails), and `devops/15` (agent observability) as related templates.

---

### Requirement 4: Strands Multi-Agent Patterns (mlops/21)

**User Story:** As an AI engineer, I want a template for implementing Graph, Swarm, and Workflow multi-agent patterns with the Strands SDK, so that I can build coordinated multi-agent systems with shared state and dynamic handoff.

#### Acceptance Criteria

1. THE Template SHALL generate implementations for three multi-agent patterns: Graph (DAG execution), Swarm (dynamic handoff), and Workflow (sequential pipeline).
2. WHEN the Graph pattern is selected, THE Template SHALL generate a directed acyclic graph of agents using the `graph` community tool with node definitions, edge connections, and conditional branching.
3. WHEN the Swarm pattern is selected, THE Template SHALL generate a swarm configuration using the `swarm` community tool with agent definitions, handoff conditions, and dynamic routing logic.
4. WHEN the Workflow pattern is selected, THE Template SHALL generate a sequential pipeline using the `workflow` community tool with ordered agent steps and intermediate result passing.
5. THE Template SHALL generate agents-as-tools patterns using the `use_agent` community tool where one agent invokes another agent as a callable tool.
6. THE Template SHALL generate shared state management using `agent.invocation_state` for passing context between agents in all three patterns.
7. WHEN A2A protocol integration is enabled, THE Template SHALL generate `a2a_client` tool configuration for cross-framework agent communication.
8. THE Template SHALL generate a `think` tool integration for agents that require explicit reasoning steps before taking actions.
9. IF an agent in a multi-agent pattern fails or times out, THEN THE Template SHALL include fallback routing and error propagation patterns that prevent cascading failures.
10. THE Integration_Points_Section SHALL reference `mlops/20` (Strands Lambda deployment), `mlops/14` (Bedrock Agents for comparison), and `devops/15` (agent observability) as related templates.

---

### Requirement 5: Strands AgentCore Deployment (mlops/22)

**User Story:** As an AI engineer, I want a template for deploying Strands agents to Amazon Bedrock AgentCore Runtime, so that I can run agents with microVM session isolation, identity integration, and auto-scaling.

#### Acceptance Criteria

1. THE Template SHALL generate a complete AgentCore Runtime deployment configuration for a Strands agent, including runtime endpoint setup and agent packaging.
2. THE Template SHALL generate session persistence configuration that leverages AgentCore microVM isolation for per-session state management.
3. WHEN an identity provider is specified, THE Template SHALL generate identity integration configuration supporting Cognito, Entra ID, and Okta as identity providers.
4. THE Template SHALL generate auto-scaling configuration for the AgentCore Runtime deployment with configurable minimum and maximum instance counts.
5. THE Template SHALL generate both Python and TypeScript SDK examples for invoking the deployed AgentCore agent endpoint.
6. THE Template SHALL include a comparison section in the Role Definition that explains when to choose AgentCore Runtime over Lambda deployment (mlops/20) based on session duration, state requirements, and isolation needs.
7. IF the AgentCore Runtime deployment fails health checks, THEN THE Template SHALL include health check configuration and rollback patterns.
8. THE Code_Scaffolding_Hints_Section SHALL include working AgentCore Runtime deployment API calls and session management patterns.
9. THE Integration_Points_Section SHALL reference `mlops/20` (Lambda alternative), `devops/04` (IAM roles), `devops/15` (agent observability), and `devops/16` (agent guardrails) as related templates.

---

### Requirement 6: Agent SOP Authoring (mlops/23)

**User Story:** As an AI engineer or technical writer, I want a template for creating Agent SOPs using the standardized markdown format, so that I can write natural language workflows that balance flexibility and control for AI agents.

#### Acceptance Criteria

1. THE Template SHALL generate Agent SOP markdown files using the standardized format with title, purpose, scope, inputs, outputs, and numbered procedural steps.
2. THE Template SHALL use RFC 2119 keywords (MUST, SHOULD, MAY, MUST NOT, SHOULD NOT) in procedural steps to define obligation levels for agent behavior.
3. THE Template SHALL generate parameterized input sections using `{{parameter_name}}` syntax with type annotations and default values.
4. WHEN SOP chaining is enabled, THE Template SHALL generate SOP-to-SOP references that allow one SOP to invoke another SOP as a sub-procedure with parameter passing.
5. THE Template SHALL generate progress tracking markers that enable agents to report completion status for each procedural step.
6. THE Template SHALL include resumability patterns that allow agents to resume an SOP from a specific step after interruption.
7. THE Template SHALL generate example SOPs for common agent tasks: data retrieval, report generation, and multi-step approval workflows.
8. THE Code_Scaffolding_Hints_Section SHALL include working examples of loading SOPs as Strands Agent system prompts and as standalone workflow documents.
9. THE Integration_Points_Section SHALL reference `mlops/20` (Strands Lambda deployment for SOP execution), `mlops/21` (multi-agent patterns for SOP orchestration), and `mlops/24` (prompt management for SOP versioning) as related templates.

---

### Requirement 7: Bedrock Prompt Management (mlops/24)

**User Story:** As an AI engineer, I want a template for using the Bedrock Prompt Management API, so that I can create versioned prompts with variants, integrate with Bedrock Flows, and A/B test prompt versions.

#### Acceptance Criteria

1. THE Template SHALL generate complete Bedrock Prompt Management infrastructure including prompt creation, versioning, and retrieval using `bedrock_agent.create_prompt()`, `create_prompt_version()`, and `get_prompt()` API calls.
2. THE Template SHALL generate prompt template definitions with variable placeholders using the Bedrock prompt template syntax.
3. WHEN multiple prompt variants are defined, THE Template SHALL generate variant configurations for A/B testing with traffic splitting logic.
4. THE Template SHALL generate integration code that retrieves versioned prompts from Bedrock Prompt Management and passes them to Strands Agent system prompts or Bedrock model invocations.
5. THE Template SHALL generate a prompt lifecycle workflow: draft → review → approved → deployed, with version immutability after deployment.
6. WHEN Bedrock Flows integration is enabled, THE Template SHALL generate prompt node configurations that reference managed prompts by ARN within a Bedrock Flow definition.
7. IF a prompt version retrieval fails, THEN THE Template SHALL include fallback logic that uses the most recent successfully cached prompt version.
8. THE Code_Scaffolding_Hints_Section SHALL include working `bedrock_agent.create_prompt()`, `create_prompt_version()`, `get_prompt()`, and `bedrock_agent_runtime.invoke_flow()` code patterns.
9. THE Integration_Points_Section SHALL reference `mlops/16` (Bedrock Flows), `mlops/17` (LLM Evaluation for prompt quality), `mlops/12` (Bedrock Guardrails for prompt safety), and `mlops/23` (Agent SOPs as managed prompts) as related templates.

---

### Requirement 8: Strands Agent Observability (devops/15)

**User Story:** As a DevOps engineer, I want a template for OpenTelemetry tracing and monitoring of Strands agents, so that I can track agent loop execution, tool calls, token usage, and latency across agent deployments.

#### Acceptance Criteria

1. THE Template SHALL generate OpenTelemetry instrumentation configuration for Strands agents using the built-in `strands.tools.mcp.mcp_instrumentation` module and custom span creation.
2. THE Template SHALL generate custom spans for each tool call within the agent loop, capturing tool name, input parameters, output summary, duration, and success/failure status.
3. THE Template SHALL generate agent loop metrics including total loop iterations, tokens consumed (input and output), model latency per iteration, and total agent execution time.
4. THE Template SHALL generate X-Ray trace integration using the AWS Distro for OpenTelemetry (ADOT) collector with Strands-specific trace segments.
5. THE Template SHALL generate CloudWatch custom metrics for agent-level KPIs: invocations per minute, average tool calls per invocation, error rate, and p50/p90/p99 latency.
6. WHEN AgentCore deployment is used, THE Template SHALL generate AgentCore-native observability configuration that integrates with the AgentCore observability dashboard.
7. THE Template SHALL generate CloudWatch dashboard definitions with widgets for agent health, tool usage distribution, token consumption trends, and error analysis.
8. IF an agent tool call exceeds a configurable latency threshold, THEN THE Template SHALL generate CloudWatch alarm configuration that triggers SNS notifications.
9. THE Code_Scaffolding_Hints_Section SHALL include working OpenTelemetry tracer setup, custom span creation, and CloudWatch metric publishing code patterns for Strands agents.
10. THE Integration_Points_Section SHALL reference `devops/10` (base OpenTelemetry patterns), `devops/03` (CloudWatch monitoring), `devops/12` (Bedrock invocation logging), `mlops/20` (Lambda deployment), and `mlops/22` (AgentCore deployment) as related templates.

---

### Requirement 9: Agent Guardrails and Control (devops/16)

**User Story:** As a security engineer, I want a template for implementing Strands Agent Control runtime guardrails and Bedrock Guardrails integration, so that I can define what agents can and cannot do at runtime without code changes.

#### Acceptance Criteria

1. THE Template SHALL generate Strands Agent Control configuration that defines runtime guardrails for agent behavior including allowed tools, denied actions, and execution boundaries.
2. THE Template SHALL generate Bedrock Guardrails integration code that attaches content filtering, PII redaction, and denied topic policies to Strands agent model invocations.
3. THE Template SHALL generate tool consent policies that require explicit approval before agents execute sensitive tools (e.g., write operations, external API calls, shell commands).
4. THE Template SHALL generate prompt injection defense patterns including input sanitization, system prompt protection, and output validation for Strands agents.
5. WHEN a guardrail violation is detected, THE Template SHALL generate structured violation logging that captures the violation type, triggering input, agent context, and remediation action taken.
6. THE Template SHALL generate a guardrails configuration file (YAML or JSON) that can be modified without redeploying agent code, enabling runtime policy updates.
7. THE Template SHALL generate rate limiting and token budget controls that prevent agents from exceeding configurable invocation or token consumption limits per session.
8. IF an agent attempts to execute a denied tool or action, THEN THE Template SHALL generate a graceful denial response that informs the user and logs the attempt without exposing internal policy details.
9. THE Code_Scaffolding_Hints_Section SHALL include working Agent Control configuration, Bedrock Guardrails attachment code, tool consent middleware, and prompt injection defense patterns.
10. THE Integration_Points_Section SHALL reference `mlops/12` (Bedrock Guardrails upstream), `devops/04` (IAM roles), `devops/15` (agent observability for violation tracking), `mlops/20` (Lambda deployment), and `mlops/22` (AgentCore deployment) as related templates.

---

### Requirement 10: Cross-Template Integration Graph for Strands Ecosystem

**User Story:** As a library user, I want the 7 new Strands templates to form a coherent integration graph with each other and with existing templates, so that I can chain them together for end-to-end agent workflows.

#### Acceptance Criteria

1. THE Template_Library SHALL ensure that `mlops/20` (Strands Lambda) references `mlops/21` (multi-agent) as a downstream consumer for deploying multi-agent patterns to Lambda.
2. THE Template_Library SHALL ensure that `mlops/22` (AgentCore) and `mlops/20` (Lambda) both reference `devops/15` (observability) and `devops/16` (guardrails) as downstream integration points.
3. THE Template_Library SHALL ensure that `mlops/23` (Agent SOPs) references `mlops/24` (Prompt Management) for versioning SOPs as managed prompts.
4. THE Template_Library SHALL ensure that `devops/15` (observability) extends the patterns from `devops/10` (OpenTelemetry) with agent-specific instrumentation.
5. THE Template_Library SHALL ensure that `devops/16` (guardrails/control) integrates with `mlops/12` (Bedrock Guardrails) as an upstream dependency for content filtering policies.
6. WHEN the PROMPT_GUIDE is updated, THE PROMPT_GUIDE SHALL include a Strands Agent deployment chain: `devops/04` (IAM) → `mlops/23` (SOP) → `mlops/24` (Prompt Mgmt) → `mlops/20` (Lambda Deploy) → `mlops/21` (Multi-Agent) → `devops/15` (Observability) → `devops/16` (Guardrails).
7. THE Template_Library SHALL ensure that all 7 new templates reference `devops/04` (IAM) as an upstream dependency for Lambda execution roles, AgentCore service roles, and Bedrock model access policies.
