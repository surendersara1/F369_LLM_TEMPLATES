<!-- Template Version: 1.0 | boto3: 1.35+ | strands-agents: 1.x+ | strands-agents-tools: 1.x+ -->

# Template 21 — Strands Multi-Agent Patterns

## Purpose
Generate a production-ready multi-agent system using the Strands Agents SDK with three orchestration patterns: Graph (DAG execution with conditional branching via the `graph` community tool), Swarm (dynamic agent handoff via the `swarm` community tool), and Workflow (sequential pipeline via the `workflow` community tool). Includes agent-as-tool composition with `use_agent`, cross-framework communication via the `a2a_client` tool, explicit reasoning with the `think` tool, shared state management through `agent.invocation_state`, and fallback routing with error propagation patterns that prevent cascading failures.

---

## Role Definition

You are an expert AI systems architect specializing in multi-agent orchestration with the Strands Agents SDK and expertise in:
- Strands Agents SDK: `Agent()` initialization, `BedrockModel()` provider configuration, custom `@tool` decorated functions, and `agent.invocation_state` for shared state across agents
- Multi-agent patterns: Graph (directed acyclic graph execution with conditional branching), Swarm (dynamic handoff between specialized agents), Workflow (sequential pipeline with intermediate result passing)
- Strands community tools: `graph` for DAG orchestration, `swarm` for dynamic routing, `workflow` for sequential pipelines, `use_agent` for agent-as-tool composition, `a2a_client` for Agent-to-Agent protocol communication, `think` for explicit reasoning steps, `handoff_to_user` for human-in-the-loop escalation
- Agent-to-Agent (A2A) protocol: cross-framework agent interoperability, A2A server endpoint configuration, and client-side agent discovery
- Shared state management: `agent.invocation_state` dictionary for passing context between agents, state initialization, and state propagation across pattern boundaries
- Error handling: fallback routing when agents fail or timeout, error propagation without cascading failures, graceful degradation in multi-agent DAGs and swarms
- AWS integration: Bedrock Runtime for model invocations, Lambda (optional) for deploying individual agents, Step Functions (optional) for durable multi-agent workflows
- IAM policies for Bedrock model access (`bedrock:InvokeModel`, `bedrock:InvokeModelWithResponseStream`) scoped to configured model IDs

Generate complete, production-deployable code.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED - must be a Bedrock supported region]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

PATTERN_TYPE:           [REQUIRED - graph | swarm | workflow]
                        Options:
                        - graph — Directed acyclic graph of agents with conditional
                          branching. Use when agents have dependency relationships
                          and tasks can be parallelized along independent branches.
                        - swarm — Dynamic handoff between specialized agents based
                          on context. Use when the routing decision depends on the
                          content of the conversation and agents self-select.
                        - workflow — Sequential pipeline of agents where each step
                          passes results to the next. Use when tasks follow a fixed
                          linear order with intermediate transformations.

AGENT_DEFINITIONS:      [REQUIRED - JSON list of agent definitions]
                        Example: [
                            {
                                "name": "researcher",
                                "system_prompt": "You research topics thoroughly...",
                                "tools": ["retrieve", "browser"],
                                "model_id": "us.anthropic.claude-sonnet-4-20250514-v1:0"
                            },
                            {
                                "name": "writer",
                                "system_prompt": "You write clear, structured content...",
                                "tools": ["editor"],
                                "model_id": "us.anthropic.claude-sonnet-4-20250514-v1:0"
                            },
                            {
                                "name": "reviewer",
                                "system_prompt": "You review content for accuracy...",
                                "tools": ["think"],
                                "model_id": "us.anthropic.claude-sonnet-4-20250514-v1:0"
                            }
                        ]

MODEL_ID:               [OPTIONAL: us.anthropic.claude-sonnet-4-20250514-v1:0]
                        Default model for agents without an explicit model_id.
                        Options (verify current availability):
                        - us.anthropic.claude-sonnet-4-20250514-v1:0 (Claude Sonnet 4)
                        - us.anthropic.claude-3-5-sonnet-20241022-v2:0 (Claude 3.5 Sonnet v2)
                        - us.anthropic.claude-3-5-haiku-20241022-v1:0 (Claude 3.5 Haiku)
                        - us.amazon.nova-pro-v1:0 (Amazon Nova Pro)
                        - us.amazon.nova-lite-v1:0 (Amazon Nova Lite)

A2A_ENABLED:            [OPTIONAL: false]
                        Enable Agent-to-Agent protocol for cross-framework
                        communication. When true, generates a2a_server.py and
                        a2a_client configuration for interoperability with
                        external agent frameworks.

THINK_TOOL_ENABLED:     [OPTIONAL: true]
                        Enable the think community tool for explicit reasoning
                        steps. When true, agents use the think tool to reason
                        step by step before taking actions or making handoff
                        decisions.

FALLBACK_STRATEGY:      [OPTIONAL: skip_and_continue]
                        Options:
                        - skip_and_continue — Skip the failed agent and continue
                          with remaining agents in the pattern. Log the failure.
                        - retry_then_fallback — Retry the failed agent up to
                          MAX_RETRIES times, then route to a fallback agent.
                        - fail_fast — Halt the entire pattern on first agent
                          failure and return a structured error response.

MAX_RETRIES:            [OPTIONAL: 2 - max retry attempts per agent before fallback]

SHARED_STATE_KEYS:      [OPTIONAL - JSON list of initial shared state keys]
                        Example: ["research_results", "draft_content", "review_notes"]
                        These keys are initialized in agent.invocation_state and
                        passed between agents in the pattern.

GRAPH_EDGES:            [OPTIONAL - JSON adjacency list for graph pattern]
                        Example: {
                            "researcher": ["writer"],
                            "writer": ["reviewer"],
                            "reviewer": []
                        }
                        Required when PATTERN_TYPE is graph. Defines the DAG
                        edges between agent nodes.

HANDOFF_CONDITIONS:     [OPTIONAL - JSON map of swarm handoff rules]
                        Example: {
                            "researcher": "Route here for fact-finding tasks",
                            "writer": "Route here for content creation tasks",
                            "reviewer": "Route here for quality review tasks"
                        }
                        Used when PATTERN_TYPE is swarm. Defines when to
                        hand off to each agent.
```

---

## Task

Generate complete Strands multi-agent pattern implementation:

```
{PROJECT_NAME}-multi-agent/
├── agents/
│   ├── agent_definitions.py           # All agent definitions with roles and tools
│   ├── shared_state.py                # invocation_state management and propagation
│   └── tools_registry.py             # Tool registration and import per agent
├── patterns/
│   ├── graph_pattern.py               # DAG execution with graph community tool
│   ├── swarm_pattern.py               # Dynamic handoff with swarm community tool
│   └── workflow_pattern.py            # Sequential pipeline with workflow community tool
├── a2a/
│   ├── a2a_server.py                  # A2A protocol server endpoint
│   └── a2a_client_config.py           # A2A client configuration for external agents
├── orchestrator/
│   ├── main.py                        # Entry point for multi-agent execution
│   └── error_handling.py             # Fallback routing and error propagation
├── tests/
│   └── test_patterns.py
└── requirements.txt
```


**agent_definitions.py**: Agent factory module that creates all agents from AGENT_DEFINITIONS:
- `create_agent(name, system_prompt, tools, model_id)`: Factory function that initializes a `strands.Agent()` with `BedrockModel(model_id=model_id, region_name=AWS_REGION, max_tokens=4096)`, the provided system prompt, and resolved tool list
- `create_all_agents(agent_definitions)`: Iterate over AGENT_DEFINITIONS JSON, call `create_agent()` for each entry, return a dictionary mapping agent name to `Agent` instance
- If THINK_TOOL_ENABLED is true, append the `think` tool from `strands_tools` to every agent's tool list
- Each agent's `BedrockModel` uses the agent-specific `model_id` if provided, otherwise falls back to the global MODEL_ID
- Import tools by name from `tools_registry.resolve_tools()` which maps tool name strings to actual tool objects from `strands_tools`

**shared_state.py**: Shared state management using `agent.invocation_state`:
- `initialize_state(agent, shared_state_keys)`: Set up `agent.invocation_state` with empty values for each key in SHARED_STATE_KEYS
- `propagate_state(source_agent, target_agent)`: Copy all `invocation_state` entries from the source agent to the target agent after the source completes execution
- `get_state(agent, key)`: Retrieve a value from `agent.invocation_state` with a default of `None`
- `set_state(agent, key, value)`: Set a value in `agent.invocation_state`
- `merge_states(agents)`: Merge `invocation_state` from multiple agents into a single dictionary, with last-write-wins for conflicting keys
- State is the primary mechanism for passing context between agents in all three patterns

**tools_registry.py**: Tool resolution and registration:
- `resolve_tools(tool_names)`: Map tool name strings (e.g., `"retrieve"`, `"browser"`, `"editor"`) to imported tool objects from `strands_tools`
- Import community tools: `from strands_tools import retrieve, memory, editor, shell, browser, use_aws, graph, swarm, workflow, use_agent, a2a_client, think, handoff_to_user`
- Return a list of resolved tool objects for each agent definition
- Log a warning for unrecognized tool names and skip them

**graph_pattern.py**: DAG execution using the `graph` community tool:
- `build_graph(agents, graph_edges)`: Construct a directed acyclic graph from AGENT_DEFINITIONS and GRAPH_EDGES. Each node is an agent, each edge defines execution dependency
- `run_graph(coordinator, prompt, agents, graph_edges, shared_state)`: Create a coordinator `Agent` with the `graph` tool, pass the agent definitions and edge configuration, and invoke the coordinator with the user prompt. The `graph` tool handles DAG traversal, parallel execution of independent branches, and conditional branching based on agent outputs
- The coordinator agent's system prompt instructs it to execute the graph by invoking agents in topological order, respecting edge dependencies
- After each agent node completes, propagate `invocation_state` to downstream agents via `shared_state.propagate_state()`
- Support conditional edges where the coordinator decides whether to follow an edge based on the upstream agent's output
- On agent failure, apply FALLBACK_STRATEGY: skip the failed node and continue with independent branches, or halt the graph

**swarm_pattern.py**: Dynamic handoff using the `swarm` community tool:
- `build_swarm(agents, handoff_conditions)`: Configure a swarm from AGENT_DEFINITIONS and HANDOFF_CONDITIONS. Each agent has a description of when it should receive a handoff
- `run_swarm(swarm_agent, prompt, agents, handoff_conditions, shared_state)`: Create a swarm coordinator `Agent` with the `swarm` tool, pass agent definitions and handoff conditions, and invoke with the user prompt. The `swarm` tool dynamically routes to the most appropriate agent based on conversation context
- The swarm coordinator's system prompt describes all available agents and their specializations, enabling context-aware routing
- Agents in the swarm can hand off to each other — the `swarm` tool manages the handoff chain
- After each handoff, propagate `invocation_state` between the handing-off agent and the receiving agent
- On agent failure, the swarm coordinator routes to the next best agent based on HANDOFF_CONDITIONS

**workflow_pattern.py**: Sequential pipeline using the `workflow` community tool:
- `build_workflow(agents, step_order)`: Configure a sequential pipeline from AGENT_DEFINITIONS. The step order is determined by the order of agents in AGENT_DEFINITIONS (first agent runs first)
- `run_workflow(pipeline_agent, prompt, agents, shared_state)`: Create a pipeline coordinator `Agent` with the `workflow` tool, pass the ordered agent list, and invoke with the user prompt. The `workflow` tool executes agents sequentially, passing each agent's output as input to the next
- Each step receives the previous step's output via `invocation_state` propagation
- The pipeline coordinator's system prompt instructs it to execute steps in order and pass intermediate results forward
- On step failure, apply FALLBACK_STRATEGY: skip the failed step and pass the last successful output forward, retry the step, or halt the pipeline

**a2a_server.py** (generated only when A2A_ENABLED is true): A2A protocol server:
- Configure an A2A-compatible server endpoint that exposes local Strands agents to external agent frameworks
- Define an agent card with agent name, description, capabilities, and supported input/output formats
- Handle incoming A2A requests: parse the A2A message, route to the appropriate local agent, execute, and return the A2A response
- Use `a2a_client` from `strands_tools` for outbound communication with external A2A-compatible agents
- Server runs on a configurable port with health check endpoint

**a2a_client_config.py** (generated only when A2A_ENABLED is true): A2A client configuration:
- `get_a2a_client_tool(remote_agent_url)`: Configure the `a2a_client` community tool with the URL of a remote A2A-compatible agent
- `discover_agents(a2a_directory_url)`: Query an A2A agent directory to discover available remote agents and their capabilities
- Return configured `a2a_client` tool instances that can be added to any local agent's tool list

**orchestrator/main.py**: Entry point for multi-agent execution:
- Parse PATTERN_TYPE to select the appropriate pattern module (graph, swarm, or workflow)
- Load AGENT_DEFINITIONS and create all agents via `agent_definitions.create_all_agents()`
- Initialize shared state via `shared_state.initialize_state()` with SHARED_STATE_KEYS
- If A2A_ENABLED, configure A2A client tools and add them to relevant agents
- Call the selected pattern's `run_*()` function with the user prompt, agents, and configuration
- Wrap execution in `error_handling.safe_execute()` for top-level error handling
- Return the final result with execution metadata: pattern type, agents invoked, total duration, and final shared state
- Support both CLI invocation (`python -m orchestrator.main --prompt "..."`) and programmatic import

**orchestrator/error_handling.py**: Fallback routing and error propagation:
- `safe_execute(func, *args, fallback_result=None, **kwargs)`: Wrap any agent invocation with try/except, apply FALLBACK_STRATEGY on failure
- `retry_with_backoff(func, max_retries, base_delay=1.0)`: Retry a failed agent invocation with exponential backoff up to MAX_RETRIES attempts
- `fallback_route(failed_agent_name, agents, prompt)`: When an agent fails and FALLBACK_STRATEGY is `retry_then_fallback`, select an alternative agent from the available pool and route the prompt to it
- `propagate_error(error, agent_name, pattern_type)`: Create a structured error record with agent name, error type, timestamp, and pattern context — do not raise to prevent cascading failures
- `AgentExecutionError`: Custom exception class with agent name, original error, and retry count
- Log all errors with structured JSON including `agent_name`, `pattern_type`, `error_type`, `retry_count`, and `fallback_agent` (if applicable)

**test_patterns.py**: Unit tests:
- Test graph pattern execution with a simple 3-node DAG
- Test swarm pattern routing to the correct agent based on prompt content
- Test workflow pattern sequential execution with state propagation
- Test fallback routing when an agent raises an exception
- Test shared state propagation between agents

**requirements.txt**: Python dependencies including `strands-agents`, `strands-agents-tools`, `boto3`.

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**Pattern Selection:** Choose Graph when agents have dependency relationships and independent branches can run in parallel. Choose Swarm when routing depends on conversation content and agents self-select dynamically. Choose Workflow when tasks follow a fixed linear order with intermediate transformations. Generate only the selected pattern's files, but always include `agents/`, `orchestrator/`, and `tests/` directories.

**Community Tools:** All multi-agent patterns use community tools from the `strands-agents-tools` package: `graph`, `swarm`, `workflow`, `use_agent`, `a2a_client`, `think`, `handoff_to_user`. Import tools with `from strands_tools import <tool_name>`. Pin `strands-agents-tools` to a specific version in production to avoid unexpected behavior changes.

**Shared State:** Use `agent.invocation_state` as the primary mechanism for passing context between agents. Initialize state keys before pattern execution. Propagate state explicitly after each agent completes. Do not rely on global variables or module-level state for cross-agent communication.

**Fallback Strategy:** Every multi-agent pattern must implement a fallback strategy for agent failures. `skip_and_continue` is the safest default — it prevents cascading failures by skipping failed agents. `retry_then_fallback` adds resilience with retries before routing to an alternative agent. `fail_fast` is appropriate only when partial results are unacceptable.

**Error Propagation:** Never let a single agent failure crash the entire multi-agent pattern. Log all errors with structured JSON including agent name, error type, and pattern context. Return partial results when possible. The orchestrator should always return a response, even if some agents failed.

**A2A Protocol:** A2A integration is optional and adds complexity. Enable only when agents need to communicate with external frameworks (LangGraph, CrewAI, AutoGen). The A2A server endpoint must include authentication and rate limiting in production. A2A communication adds network latency — set appropriate timeouts.

**Think Tool:** The `think` tool enables explicit reasoning without consuming tool call budget. Enable it for agents that make complex routing decisions (swarm coordinators) or need to plan multi-step actions (graph coordinators). The think tool's output is visible in traces for debugging.

**Model Selection:** Each agent in the pattern can use a different model. Use higher-capability models (Claude Sonnet) for coordinators and complex reasoning agents. Use faster, cheaper models (Claude Haiku, Nova Lite) for simple task agents. All models must be available in the configured AWS_REGION.

**Security:** Each agent's `BedrockModel` requires `bedrock:InvokeModel` and `bedrock:InvokeModelWithResponseStream` IAM permissions scoped to the specific model ID. When deploying to Lambda, each agent Lambda function should have its own least-privilege IAM role. A2A server endpoints must use authentication (API keys or IAM) in production.

**Cost:** Multi-agent patterns multiply model invocation costs — each agent in the pattern makes independent Bedrock API calls. The coordinator agent adds overhead with its own model calls. Monitor total token usage across all agents with CloudWatch custom metrics. Use the `think` tool judiciously as it adds to token consumption.

**Naming:** All resources follow `{PROJECT_NAME}-{component}-{ENV}` convention:
- Multi-agent orchestrator: `{PROJECT_NAME}-multi-agent-orchestrator-{ENV}`
- Individual agents: `{PROJECT_NAME}-agent-{agent_name}-{ENV}`
- A2A server: `{PROJECT_NAME}-a2a-server-{ENV}`
- Shared state table (if persisted): `{PROJECT_NAME}-agent-state-{ENV}`

---

## Code Scaffolding Hints

**Agent definition factory:**
```python
from strands import Agent
from strands.models.bedrock import BedrockModel

def create_agent(name: str, system_prompt: str, tools: list,
                 model_id: str, region: str) -> Agent:
    """Create a Strands agent from a definition."""
    model = BedrockModel(
        model_id=model_id,
        region_name=region,
        max_tokens=4096,
        temperature=0.7,
    )
    agent = Agent(
        model=model,
        system_prompt=system_prompt,
        tools=tools,
    )
    return agent


# Example: create agents from definitions
researcher = create_agent(
    name="researcher",
    system_prompt="You research topics thoroughly using available tools...",
    tools=[retrieve, browser, think],
    model_id="us.anthropic.claude-sonnet-4-20250514-v1:0",
    region="us-east-1",
)

writer = create_agent(
    name="writer",
    system_prompt="You write clear, structured content based on research...",
    tools=[editor, think],
    model_id="us.anthropic.claude-sonnet-4-20250514-v1:0",
    region="us-east-1",
)

reviewer = create_agent(
    name="reviewer",
    system_prompt="You review content for accuracy and completeness...",
    tools=[think],
    model_id="us.anthropic.claude-sonnet-4-20250514-v1:0",
    region="us-east-1",
)
```

**Graph pattern — DAG execution with the `graph` community tool:**
```python
from strands import Agent
from strands.models.bedrock import BedrockModel
from strands_tools import graph, think

model = BedrockModel(
    model_id="us.anthropic.claude-sonnet-4-20250514-v1:0",
    region_name="us-east-1",
    max_tokens=4096,
)

# The coordinator agent uses the graph tool to orchestrate a DAG of agents.
# Define sub-agents as nodes; the graph tool manages execution order,
# parallel branches, and conditional edges.
coordinator = Agent(
    model=model,
    system_prompt="""You coordinate a team of agents in a directed graph.
    
Available agents:
- researcher: Researches topics using retrieve and browser tools
- writer: Writes content based on research findings
- reviewer: Reviews content for accuracy and quality

Graph structure:
- researcher → writer (writer depends on researcher output)
- writer → reviewer (reviewer depends on writer output)

Execute agents in topological order. Pass results between agents
using shared state. If an agent fails, skip it and continue with
independent branches.""",
    tools=[graph, think],
)

# Invoke the coordinator — the graph tool handles DAG traversal
response = coordinator("Research AWS Lambda best practices, write a summary, and review it")
print(response)
```

**Swarm pattern — dynamic handoff with the `swarm` community tool:**
```python
from strands import Agent
from strands.models.bedrock import BedrockModel
from strands_tools import swarm, think

model = BedrockModel(
    model_id="us.anthropic.claude-sonnet-4-20250514-v1:0",
    region_name="us-east-1",
    max_tokens=4096,
)

# The swarm coordinator dynamically routes to specialized agents
# based on conversation context. Agents can hand off to each other.
swarm_coordinator = Agent(
    model=model,
    system_prompt="""You route requests to the most appropriate specialist agent.

Available specialists:
- researcher: For fact-finding, data gathering, and information retrieval tasks
- writer: For content creation, drafting, and text generation tasks
- reviewer: For quality review, fact-checking, and editing tasks

Analyze the user's request and route to the best specialist.
Specialists can hand off to each other when they need a different expertise.
Think step by step about which specialist is most appropriate.""",
    tools=[swarm, think],
)

# The swarm tool dynamically selects the right agent
response = swarm_coordinator("I need a well-researched blog post about serverless architectures")
print(response)
```

**Workflow pattern — sequential pipeline with the `workflow` community tool:**
```python
from strands import Agent
from strands.models.bedrock import BedrockModel
from strands_tools import workflow, think

model = BedrockModel(
    model_id="us.anthropic.claude-sonnet-4-20250514-v1:0",
    region_name="us-east-1",
    max_tokens=4096,
)

# The workflow coordinator executes agents in a fixed sequential order.
# Each step's output becomes the next step's input.
pipeline = Agent(
    model=model,
    system_prompt="""You execute a sequential pipeline of agents in this order:

Step 1 — researcher: Gather information on the topic
Step 2 — writer: Write content based on the research
Step 3 — reviewer: Review and refine the written content

Execute each step in order. Pass the output of each step as input
to the next step. If a step fails, log the error and pass the last
successful output to the next step.""",
    tools=[workflow, think],
)

# The workflow tool executes steps sequentially
response = pipeline("Create a technical guide on multi-agent AI systems")
print(response)
```

**Agents-as-tools pattern with `use_agent`:**
```python
from strands import Agent
from strands.models.bedrock import BedrockModel
from strands_tools import use_agent, think

model = BedrockModel(
    model_id="us.anthropic.claude-sonnet-4-20250514-v1:0",
    region_name="us-east-1",
    max_tokens=4096,
)

# The orchestrator invokes sub-agents as callable tools.
# Each sub-agent is treated as a tool the orchestrator can call.
orchestrator = Agent(
    model=model,
    system_prompt="""You orchestrate sub-agents to complete complex tasks.

You can invoke these sub-agents as tools:
- researcher: Call for information gathering
- writer: Call for content creation
- reviewer: Call for quality review

Decide which sub-agents to call and in what order based on the task.
You may call the same sub-agent multiple times if needed.""",
    tools=[use_agent, think],
)

response = orchestrator("Produce a reviewed technical document on AWS Bedrock")
print(response)
```

**A2A protocol — cross-framework agent communication:**
```python
from strands import Agent
from strands.models.bedrock import BedrockModel
from strands_tools import a2a_client, think

model = BedrockModel(
    model_id="us.anthropic.claude-sonnet-4-20250514-v1:0",
    region_name="us-east-1",
    max_tokens=4096,
)

# Agent with A2A client tool for communicating with external agents
# running on different frameworks (e.g., LangGraph, CrewAI, AutoGen)
agent_with_a2a = Agent(
    model=model,
    system_prompt="""You can communicate with external agents via the A2A protocol.

Use the a2a_client tool to send requests to remote agents when you need
capabilities not available locally. Provide the remote agent's URL and
the task description.""",
    tools=[a2a_client, think],
)

response = agent_with_a2a("Ask the external data analysis agent to summarize Q4 sales data")
print(response)
```

**Think tool — explicit reasoning before actions:**
```python
from strands import Agent
from strands.models.bedrock import BedrockModel
from strands_tools import think, retrieve, editor

model = BedrockModel(
    model_id="us.anthropic.claude-sonnet-4-20250514-v1:0",
    region_name="us-east-1",
    max_tokens=4096,
)

# The think tool enables explicit reasoning steps.
# The agent thinks through its approach before taking actions.
reasoning_agent = Agent(
    model=model,
    system_prompt="""You are a careful, methodical agent.

Before taking any action, use the think tool to:
1. Analyze the request and identify what needs to be done
2. Plan your approach step by step
3. Consider potential issues or edge cases
4. Then execute your plan using available tools

Always think before acting.""",
    tools=[think, retrieve, editor],
)

response = reasoning_agent("Analyze the performance implications of using DynamoDB vs Aurora for session storage")
print(response)
```

**Shared state via `agent.invocation_state`:**
```python
from strands import Agent
from strands.models.bedrock import BedrockModel

model = BedrockModel(
    model_id="us.anthropic.claude-sonnet-4-20250514-v1:0",
    region_name="us-east-1",
    max_tokens=4096,
)


def initialize_state(agent: Agent, keys: list):
    """Initialize shared state keys on an agent."""
    for key in keys:
        agent.invocation_state[key] = None


def propagate_state(source: Agent, target: Agent):
    """Copy invocation_state from source agent to target agent."""
    for key, value in source.invocation_state.items():
        target.invocation_state[key] = value


def merge_states(agents: list[Agent]) -> dict:
    """Merge invocation_state from multiple agents (last-write-wins)."""
    merged = {}
    for agent in agents:
        merged.update(agent.invocation_state)
    return merged


# Example: state propagation in a workflow
researcher = Agent(model=model, system_prompt="Research the topic...", tools=tools)
writer = Agent(model=model, system_prompt="Write based on research...", tools=tools)

# Initialize shared state
initialize_state(researcher, ["research_results", "sources"])
initialize_state(writer, ["research_results", "sources", "draft"])

# Run researcher — it populates invocation_state
researcher("Research AWS multi-agent patterns")

# Propagate state from researcher to writer
propagate_state(researcher, writer)

# Writer now has access to research_results via its invocation_state
writer("Write a summary based on the research results in your context")
```

**Fallback routing and error propagation:**
```python
import time
import logging
import json

logger = logging.getLogger(__name__)


class AgentExecutionError(Exception):
    """Custom exception for agent execution failures."""

    def __init__(self, agent_name: str, original_error: Exception, retry_count: int = 0):
        self.agent_name = agent_name
        self.original_error = original_error
        self.retry_count = retry_count
        super().__init__(f"Agent '{agent_name}' failed after {retry_count} retries: {original_error}")


def retry_with_backoff(func, max_retries: int = 2, base_delay: float = 1.0):
    """Retry a function with exponential backoff."""
    for attempt in range(max_retries + 1):
        try:
            return func()
        except Exception as e:
            if attempt < max_retries:
                delay = base_delay * (2 ** attempt)
                logger.warning("Retry %d/%d: %s (waiting %.1fs)", attempt + 1, max_retries, e, delay)
                time.sleep(delay)
            else:
                raise


def safe_execute(agent, prompt: str, agent_name: str, fallback_strategy: str = "skip_and_continue",
                 max_retries: int = 2, fallback_agents: dict = None):
    """Execute an agent with fallback strategy on failure."""
    try:
        if fallback_strategy == "retry_then_fallback":
            return retry_with_backoff(lambda: agent(prompt), max_retries=max_retries)
        else:
            return agent(prompt)

    except Exception as e:
        error_record = {
            "event": "agent_execution_error",
            "agent_name": agent_name,
            "error_type": type(e).__name__,
            "error_message": str(e),
            "fallback_strategy": fallback_strategy,
        }
        logger.error(json.dumps(error_record))

        if fallback_strategy == "fail_fast":
            raise AgentExecutionError(agent_name, e)

        if fallback_strategy == "retry_then_fallback" and fallback_agents:
            # Route to an alternative agent
            for name, fallback_agent in fallback_agents.items():
                if name != agent_name:
                    logger.info("Falling back from '%s' to '%s'", agent_name, name)
                    try:
                        return fallback_agent(prompt)
                    except Exception:
                        continue

        # skip_and_continue: return None and let the pattern continue
        logger.warning("Skipping failed agent '%s', continuing with pattern", agent_name)
        return None
```

---

## Integration Points

- **Upstream**: `devops/04` → IAM roles and policies for Bedrock model access (`bedrock:InvokeModel`) used by all agents in the multi-agent pattern
- **Upstream**: `mlops/20` → Strands Agent Lambda deployment for deploying individual agents as Lambda functions that participate in multi-agent patterns
- **Upstream**: `mlops/22` → Strands AgentCore deployment for deploying long-running agents with session isolation that participate in multi-agent patterns
- **Upstream**: `mlops/23` → Agent SOPs loaded as system prompts for agents within the multi-agent pattern, defining structured workflows per agent role
- **Upstream**: `mlops/24` → Managed prompt versions retrieved from Bedrock Prompt Management for agent system prompts within the pattern
- **Downstream**: `devops/15` → Agent observability with OpenTelemetry tracing for multi-agent execution spans, per-agent tool call tracking, and pattern-level latency metrics
- **Downstream**: `devops/16` → Agent guardrails and control policies applied to individual agents within the multi-agent pattern at runtime
- **Comparison**: `mlops/14` → Bedrock Agents with Action Groups uses the managed Bedrock Agent service with built-in orchestration; this template uses the open-source Strands SDK community tools (`graph`, `swarm`, `workflow`) for full control over multi-agent coordination logic
- **Comparison**: `mlops/16` → Bedrock Flows provides managed orchestration with visual flow builder; this template provides code-first orchestration with Strands community tools for dynamic, agent-driven coordination
