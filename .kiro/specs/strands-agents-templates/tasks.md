# Implementation Plan: Strands Agents Templates

## Overview

Create 7 new F369 prompt templates for the Strands Agents SDK ecosystem, expanding the library from 67 to 74 templates. Each template follows the established F369 format with version comment, title/purpose, role definition, context & inputs, task section, output format, requirements & constraints, code scaffolding hints, and integration points. After all templates are created, update Library.md, README.md, and PROMPT_GUIDE.md to reflect the expanded library.

## Tasks

## Strands Agent Core Templates (mlops/20-24)

- [x] 1. Create `mlops/20_strands_agent_lambda_deployment.md`
  - [x] 1.1 Write template version comment (`<!-- Template Version: 1.0 | boto3: 1.35+ | strands-agents: 1.x+ | CDK: 2.170+ -->`), title, purpose, and role definition for Strands Agent Lambda deployment expertise with BedrockModel, MCP tools, and conversation management
  - [x] 1.2 Write Context & Inputs section with required parameters (PROJECT_NAME, AWS_REGION, AWS_ACCOUNT_ID, ENV) and template-specific parameters: AGENT_SYSTEM_PROMPT, MODEL_ID, MCP_SERVER_URIS, CONVERSATION_MANAGER_TYPE (sliding_window/summarizing), WINDOW_SIZE, SESSION_TABLE_NAME, API_TYPE (rest/http)
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 3.1, 3.2, 3.3, 3.5, 3.6, 3.7_
  - [x] 1.3 Write Task section with ASCII directory tree and file-by-file instructions: CDK stack (PythonFunction + API Gateway + DynamoDB), Lambda handler with Agent() initialization, agent_config.py with BedrockModel() setup, mcp_setup.py with MCPClient() configuration, session_manager.py for DynamoDB persistence, custom_tools.py with @tool decorator examples
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5, 3.6, 3.7, 1.5, 1.9_
  - [x] 1.4 Write Code Scaffolding Hints with working `Agent()`, `BedrockModel()`, `MCPClient()`, `SlidingWindowConversationManager`, Lambda handler pattern, DynamoDB session load/save, and retry logic with exponential backoff
    - _Requirements: 1.8, 3.8, 3.9_
  - [x] 1.5 Write Integration Points referencing upstream `devops/04` (IAM), `iac/02` (CDK patterns), `mlops/12` (Bedrock Guardrails) and downstream `mlops/21` (multi-agent), `devops/15` (agent observability), `devops/16` (agent guardrails)
    - _Requirements: 3.10, 10.1, 10.2_
  - [x] 1.6 Write Requirements & Constraints (naming convention, Lambda layer ARN, cold start handling, error responses) and Output Format sections
    - _Requirements: 1.6, 1.9, 3.8_

- [x] 2. Create `mlops/21_strands_multi_agent_patterns.md`
  - [x] 2.1 Write template version comment, title, purpose, and role definition for multi-agent orchestration expertise covering Graph, Swarm, and Workflow patterns with Strands community tools
  - [x] 2.2 Write Context & Inputs section with required parameters (PROJECT_NAME, AWS_REGION, AWS_ACCOUNT_ID, ENV) and template-specific parameters: PATTERN_TYPE (graph/swarm/workflow), AGENT_DEFINITIONS (JSON), A2A_ENABLED, THINK_TOOL_ENABLED, FALLBACK_STRATEGY
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 4.1_
  - [x] 2.3 Write Task section with ASCII directory tree and file-by-file instructions: agent_definitions.py, shared_state.py (invocation_state), graph_pattern.py (DAG with graph tool), swarm_pattern.py (dynamic handoff with swarm tool), workflow_pattern.py (sequential pipeline with workflow tool), a2a_server.py, orchestrator/main.py, error_handling.py with fallback routing
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5, 4.6, 4.7, 4.8, 4.9, 1.5, 1.9_
  - [x] 2.4 Write Code Scaffolding Hints with working `graph`, `swarm`, `workflow`, `use_agent`, `a2a_client`, `think` community tool patterns, `agent.invocation_state` shared state, and fallback routing examples
    - _Requirements: 1.8, 4.2, 4.3, 4.4, 4.5, 4.6, 4.7, 4.8, 4.9_
  - [x] 2.5 Write Integration Points referencing upstream `mlops/20` (Lambda deployment), `mlops/14` (Bedrock Agents comparison) and downstream `devops/15` (agent observability)
    - _Requirements: 4.10, 10.1_
  - [x] 2.6 Write Requirements & Constraints and Output Format sections
    - _Requirements: 1.6, 1.9_

- [x] 3. Checkpoint — Verify mlops/20 and mlops/21 templates
  - Ensure both templates follow F369 format with all required sections in correct order. Verify version comments, required parameters, ASCII directory trees, and integration point cross-references. Ask the user if questions arise.

- [x] 4. Create `mlops/22_strands_agentcore_deployment.md`
  - [x] 4.1 Write template version comment, title, purpose, and role definition for AgentCore Runtime deployment expertise with comparison to Lambda deployment (mlops/20)
  - [x] 4.2 Write Context & Inputs section with required parameters (PROJECT_NAME, AWS_REGION, AWS_ACCOUNT_ID, ENV) and template-specific parameters: IDENTITY_PROVIDER (cognito/entra_id/okta), IDENTITY_CONFIG (JSON), MIN_INSTANCES, MAX_INSTANCES, NETWORK_MODE (public/vpc), MODEL_ID, AGENT_SYSTEM_PROMPT
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 5.1, 5.3, 5.4_
  - [x] 4.3 Write Task section with ASCII directory tree and file-by-file instructions: agent_app.py (Strands Agent entry point), deploy_agentcore.py (Runtime endpoint creation), identity_setup.py (Cognito/Entra ID/Okta), scaling_config.py, health_check.py with rollback, invoke_agent.py (Python SDK), invoke_agent.ts (TypeScript SDK), CDK stack for supporting resources
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5, 5.6, 5.7, 1.5, 1.9_
  - [x] 4.4 Write Code Scaffolding Hints with working `bedrock-agentcore` client calls: `create_runtime_endpoint()`, `invoke_agent()`, `get_runtime_endpoint()` health check, identity config, and auto-scaling configuration
    - _Requirements: 1.8, 5.8_
  - [x] 4.5 Write Integration Points referencing upstream `mlops/20` (Lambda alternative), `devops/04` (IAM), `iac/02` (CDK) and downstream `devops/15` (agent observability), `devops/16` (agent guardrails)
    - _Requirements: 5.9, 10.2_
  - [x] 4.6 Write Requirements & Constraints (Lambda vs AgentCore decision matrix, naming convention, health check patterns) and Output Format sections
    - _Requirements: 1.6, 1.9, 5.6, 5.7_

- [x] 5. Create `mlops/23_agent_sop_authoring.md`
  - [x] 5.1 Write template version comment, title, purpose, and role definition for Agent SOP authoring expertise with RFC 2119 keywords and structured markdown workflows
  - [x] 5.2 Write Context & Inputs section with required parameters (PROJECT_NAME, AWS_REGION, AWS_ACCOUNT_ID, ENV) and template-specific parameters: SOP_NAME, SOP_PURPOSE, SOP_INPUTS (JSON with parameter types), SOP_CHAINING_ENABLED, PROGRESS_TRACKING_ENABLED, RESUMABILITY_ENABLED
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 6.1, 6.3, 6.4, 6.5, 6.6_
  - [x] 5.3 Write Task section with ASCII directory tree and file-by-file instructions: SOP templates (data_retrieval.md, report_generation.md, approval_workflow.md), sop_schema.md (format specification), sop_chaining.md (SOP-to-SOP references), sop_loader.py, sop_validator.py, progress_tracker.py, example runner scripts
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5, 6.6, 6.7, 1.5, 1.9_
  - [x] 5.4 Write Code Scaffolding Hints with working examples of loading SOPs as Strands Agent system prompts, `{{parameter}}` substitution, RFC 2119 keyword usage, progress tracking markers, and SOP chaining patterns
    - _Requirements: 1.8, 6.8_
  - [x] 5.5 Write Integration Points referencing upstream (none — SOPs are authoring artifacts) and downstream `mlops/20` (Lambda deployment for SOP execution), `mlops/21` (multi-agent for SOP orchestration), `mlops/24` (prompt management for SOP versioning)
    - _Requirements: 6.9, 10.3_
  - [x] 5.6 Write Requirements & Constraints and Output Format sections
    - _Requirements: 1.6, 1.9_

- [x] 6. Create `mlops/24_bedrock_prompt_management.md`
  - [x] 6.1 Write template version comment, title, purpose, and role definition for Bedrock Prompt Management API expertise with versioning, variants, and Flows integration
  - [x] 6.2 Write Context & Inputs section with required parameters (PROJECT_NAME, AWS_REGION, AWS_ACCOUNT_ID, ENV) and template-specific parameters: PROMPT_NAME, PROMPT_TEMPLATE_TEXT, PROMPT_VARIABLES (JSON), VARIANTS (JSON for A/B testing), FLOWS_INTEGRATION_ENABLED, LIFECYCLE_WORKFLOW_ENABLED
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 7.1, 7.2, 7.3_
  - [x] 6.3 Write Task section with ASCII directory tree and file-by-file instructions: create_prompt.py (prompt creation with variants), version_manager.py (draft→review→approved→deployed lifecycle), prompt_retriever.py (retrieve and cache versions), ab_testing.py (traffic splitting), strands_integration.py (load into Strands agents), flows_integration.py (Bedrock Flow prompt nodes), fallback_cache.py, promote/rollback scripts
    - _Requirements: 7.1, 7.2, 7.3, 7.4, 7.5, 7.6, 7.7, 1.5, 1.9_
  - [x] 6.4 Write Code Scaffolding Hints with working `bedrock_agent.create_prompt()`, `create_prompt_version()`, `get_prompt()`, variant configuration, `bedrock_agent_runtime.invoke_flow()` prompt node reference, and fallback cache pattern
    - _Requirements: 1.8, 7.8_
  - [x] 6.5 Write Integration Points referencing upstream `mlops/12` (Bedrock Guardrails for prompt safety) and downstream `mlops/16` (Bedrock Flows), `mlops/17` (LLM Evaluation for prompt quality), `mlops/23` (Agent SOPs as managed prompts)
    - _Requirements: 7.9, 10.3_
  - [x] 6.6 Write Requirements & Constraints and Output Format sections
    - _Requirements: 1.6, 1.9_

- [x] 7. Checkpoint — Verify mlops/22, mlops/23, and mlops/24 templates
  - Ensure all three templates follow F369 format with all required sections in correct order. Verify version comments, required parameters, ASCII directory trees, code scaffolding hints with correct SDK calls, and integration point cross-references. Ask the user if questions arise.

## Strands Agent Operations Templates (devops/15-16)

- [x] 8. Create `devops/15_strands_agent_observability.md`
  - [x] 8.1 Write template version comment (`<!-- Template Version: 1.0 | boto3: 1.35+ | strands-agents: 1.x+ | opentelemetry-api: 1.x+ -->`), title, purpose, and role definition for Strands agent observability expertise extending devops/10 OpenTelemetry patterns
  - [x] 8.2 Write Context & Inputs section with required parameters (PROJECT_NAME, AWS_REGION, AWS_ACCOUNT_ID, ENV) and template-specific parameters: AGENT_NAMES (list), DEPLOYMENT_TYPE (lambda/agentcore), LATENCY_THRESHOLD_MS, DASHBOARD_NAME, ADOT_COLLECTOR_CONFIG, AGENTCORE_OBSERVABILITY_ENABLED
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 8.1, 8.4, 8.6_
  - [x] 8.3 Write Task section with ASCII directory tree and file-by-file instructions: tracer_setup.py (OTel provider), agent_tracing.py (agent loop + tool call spans), token_tracking.py, agent_metrics.py (CloudWatch custom metrics publisher), dashboard_widgets.py (invocations, latency p50/p90/p99, tool usage, token trends, error analysis), latency_alarm.py, error_rate_alarm.py, agentcore_observability.py
    - _Requirements: 8.1, 8.2, 8.3, 8.4, 8.5, 8.6, 8.7, 8.8, 1.5, 1.9_
  - [x] 8.4 Write Code Scaffolding Hints with working OpenTelemetry tracer setup, custom span creation for agent invocations and tool calls, `cloudwatch.put_metric_data()` for agent KPIs, CloudWatch dashboard JSON definition, and alarm configuration
    - _Requirements: 1.8, 8.9_
  - [x] 8.5 Write Integration Points referencing upstream `devops/10` (base OpenTelemetry), `devops/03` (CloudWatch), `devops/12` (Bedrock logging) and downstream `mlops/20` (Lambda deployment), `mlops/22` (AgentCore deployment)
    - _Requirements: 8.10, 10.2, 10.4_
  - [x] 8.6 Write Requirements & Constraints and Output Format sections
    - _Requirements: 1.6, 1.9_

- [x] 9. Create `devops/16_agent_guardrails_control.md`
  - [x] 9.1 Write template version comment, title, purpose, and role definition for Strands Agent Control and Bedrock Guardrails integration expertise
  - [x] 9.2 Write Context & Inputs section with required parameters (PROJECT_NAME, AWS_REGION, AWS_ACCOUNT_ID, ENV) and template-specific parameters: GUARDRAIL_ID, GUARDRAIL_VERSION, ALLOWED_TOOLS (list), DENIED_TOOLS (list), SENSITIVE_TOOLS (list requiring consent), MAX_LOOP_ITERATIONS, MAX_TOKENS_PER_SESSION, RATE_LIMIT_PER_MINUTE
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 9.1, 9.2, 9.3, 9.7_
  - [x] 9.3 Write Task section with ASCII directory tree and file-by-file instructions: agent_control.yaml (runtime guardrails config), control_loader.py, control_enforcer.py, bedrock_guardrails.py (apply_guardrail integration), tool_consent.py (consent middleware), consent_policies.yaml, input_sanitizer.py (prompt injection defense), output_validator.py, system_prompt_protection.py, rate_limiter.py, token_budget.py, violation_logger.py, audit_trail.py
    - _Requirements: 9.1, 9.2, 9.3, 9.4, 9.5, 9.6, 9.7, 9.8, 1.5, 1.9_
  - [x] 9.4 Write Code Scaffolding Hints with working Agent Control YAML config, `bedrock_runtime.apply_guardrail()` integration, ToolConsentMiddleware class, prompt injection sanitization patterns, TokenBudgetController, and structured violation logging
    - _Requirements: 1.8, 9.9_
  - [x] 9.5 Write Integration Points referencing upstream `mlops/12` (Bedrock Guardrails), `devops/04` (IAM) and downstream `devops/15` (violation tracking via observability), `mlops/20` (Lambda deployment), `mlops/22` (AgentCore deployment)
    - _Requirements: 9.10, 10.2, 10.5_
  - [x] 9.6 Write Requirements & Constraints and Output Format sections
    - _Requirements: 1.6, 1.9_

- [x] 10. Checkpoint — Verify devops/15 and devops/16 templates
  - Ensure both templates follow F369 format with all required sections. Verify cross-references between all 7 new templates form a coherent integration graph. Verify all 7 templates reference `devops/04` (IAM) as upstream. Ask the user if questions arise.

## Library Index and Documentation Updates

- [x] 11. Update `Library.md` with expanded directory tree and build order
  - [x] 11.1 Update the directory tree to show all 74 templates across 8 directories, adding mlops/20-24 and devops/15-16 with one-line descriptions
  - [x] 11.2 Update the template count from 67 to 74 (mlops from 20 to 25, devops from 14 to 16)
  - [x] 11.3 Add a new Phase (Strands Agents) to the Recommended Build Order: `devops/04` → `mlops/23` → `mlops/24` → `mlops/20` or `mlops/22` → `mlops/21` → `devops/15` → `devops/16`
    - _Requirements: 2.2, 2.3_

- [x] 12. Update `README.md` with all new templates
  - [x] 12.1 Add new rows to the MLOps table for mlops/20-24 (Strands Lambda, Multi-Agent, AgentCore, Agent SOPs, Prompt Management) with file path and one-line description
  - [x] 12.2 Add new rows to the DevOps table for devops/15-16 (Agent Observability, Agent Guardrails/Control) with file path and one-line description
  - [x] 12.3 Update the Architecture Overview ASCII diagram to show the Strands Agents ecosystem as a sub-layer within MLOps Core and Observability layers
  - [x] 12.4 Update the Recommended Build Order to add a Strands Agents step after the existing Agentic AI section (step 10), incorporating the deployment chain: `mlops/23` → `mlops/24` → `mlops/20`/`mlops/22` → `mlops/21` → `devops/15` → `devops/16`
  - [x] 12.5 Update the library description from "67 LLM prompt templates" to "74 LLM prompt templates"
    - _Requirements: 2.1, 2.3, 2.4_

- [x] 13. Update `PROMPT_GUIDE.md` with new interaction map and chaining example
  - [x] 13.1 Update the Template Interaction Map ASCII diagram to include all 7 new Strands templates with cross-references in the MLOps Core and Observability layers
  - [x] 13.2 Add a new chaining example: Strands Agent Workflow — `devops/04` (IAM) → `mlops/23` (Agent SOP) → `mlops/24` (Prompt Management) → `mlops/20` (Lambda Deploy) → `mlops/21` (Multi-Agent) → `devops/15` (Observability) → `devops/16` (Guardrails)
  - [x] 13.3 Update the Common Pitfalls table with entries for Strands templates (Lambda layer cold start, MCP server connection, Agent Control fail-closed, AgentCore session state, multi-agent cascading failures)
    - _Requirements: 2.5, 2.6, 10.6_

- [x] 14. Final checkpoint — Verify all 7 templates and documentation updates
  - Ensure all 7 new templates exist with correct F369 format. Verify Library.md shows 74 templates, README.md template map includes all new entries, PROMPT_GUIDE.md has updated interaction map and Strands chaining example. Verify cross-template integration graph is complete and bidirectional. Ask the user if questions arise.

## Notes

- Each template is a markdown prompt file, not executable code — no unit tests or property tests apply
- Each task references specific requirements for traceability
- Checkpoints ensure incremental validation of F369 format compliance
- The Strands deployment chain order (SOP → Prompt Mgmt → Lambda/AgentCore → Multi-Agent → Observability → Guardrails) should be maintained across all documentation updates
