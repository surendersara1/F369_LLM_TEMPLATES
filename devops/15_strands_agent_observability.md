<!-- Template Version: 1.0 | boto3: 1.35+ | strands-agents: 1.x+ | opentelemetry-api: 1.x+ -->

# Template DevOps 15 — Strands Agent Observability

## Purpose
Generate production-ready observability infrastructure for Strands Agents: OpenTelemetry instrumentation with custom spans for agent loop execution and individual tool calls, token usage tracking per invocation, CloudWatch custom metrics for agent-level KPIs (invocations per minute, average tool calls per invocation, error rate, p50/p90/p99 latency), CloudWatch dashboard with widgets for agent health, tool usage distribution, token consumption trends, and error analysis, CloudWatch alarms for latency threshold breaches and error rate spikes with SNS notifications, X-Ray trace integration via AWS Distro for OpenTelemetry (ADOT), and AgentCore-native observability configuration for agents deployed to Bedrock AgentCore Runtime.

---

## Role Definition

You are an expert AWS observability engineer specializing in Strands Agents SDK instrumentation with expertise in:
- Strands Agents SDK: agent loop lifecycle, tool call hooks, `Agent()` trace metadata, built-in tracing integration points
- OpenTelemetry: OTel API and SDK for Python, `TracerProvider`, `BatchSpanProcessor`, custom span creation with `tracer.start_as_current_span()`, span attributes, status codes, and exception recording
- AWS Distro for OpenTelemetry (ADOT): Lambda layer deployment, ECS sidecar collector, ADOT Collector configuration (receivers, processors, exporters) for X-Ray backend
- AWS X-Ray: trace groups, filter expressions, sampling rules, trace analysis, Insights anomaly detection for agent workloads
- CloudWatch custom metrics: `cloudwatch.put_metric_data()` for agent KPIs with dimensions (AgentName, Environment), metric math for derived metrics
- CloudWatch dashboards: JSON dashboard body definitions with metric widgets, text widgets, log query widgets for agent health visualization
- CloudWatch alarms: threshold alarms on custom agent metrics, composite alarms, SNS notification actions
- Token tracking: input/output token counting per agent invocation and per tool call, aggregation for cost estimation
- Agent-specific tracing patterns: wrapping the Strands agent loop with parent spans, creating child spans per tool call with tool name, input size, output size, duration, and success/failure attributes
- AgentCore observability: native observability configuration for agents deployed to Bedrock AgentCore Runtime, integration with AgentCore dashboards
- Extends `devops/10` (OpenTelemetry ML Tracing) with Strands agent-specific instrumentation — reuses the OTel tracer provider setup, ADOT collector patterns, and X-Ray trace group conventions from that template

Generate complete, production-deployable observability infrastructure code.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED - must be a region supporting X-Ray and CloudWatch]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

AGENT_NAMES:            [REQUIRED - JSON list of Strands agent names to instrument]
                        Each agent name is used as a CloudWatch metric dimension
                        and span attribute for filtering traces and metrics.
                        Example: ["customer-support-agent", "data-analyst-agent",
                                  "code-review-agent"]

DEPLOYMENT_TYPE:        [REQUIRED - lambda | agentcore]
                        Determines the ADOT collector deployment mode and
                        observability configuration:
                        - lambda — ADOT Lambda layer for agents deployed via
                          mlops/20 (Strands Lambda Deployment). Uses the
                          AWS-managed ADOT Lambda layer ARN with auto-instrumentation.
                        - agentcore — AgentCore-native observability for agents
                          deployed via mlops/22 (Strands AgentCore Deployment).
                          Uses AgentCore built-in metrics and tracing integration.

LATENCY_THRESHOLD_MS:   [OPTIONAL: 5000]
                        Latency threshold in milliseconds for agent invocation
                        alarms. When p99 latency exceeds this threshold, a
                        CloudWatch alarm triggers SNS notification.

DASHBOARD_NAME:         [OPTIONAL: {PROJECT_NAME}-strands-agents-{ENV}]
                        CloudWatch dashboard name for agent health visualization.

ADOT_COLLECTOR_CONFIG:  [OPTIONAL: default ADOT config exporting to X-Ray]
                        Custom ADOT Collector configuration YAML. If not provided,
                        generates a default config with:
                        - Receiver: OTLP (gRPC on 4317, HTTP on 4318)
                        - Processor: batch (timeout 5s, send_batch_size 256)
                        - Exporter: awsxray (region from AWS_REGION)
                        Only applicable for lambda deployment type. AgentCore
                        uses built-in collector configuration.

AGENTCORE_OBSERVABILITY_ENABLED: [OPTIONAL: true]
                        Enable AgentCore-native observability features including
                        built-in session metrics, runtime health, and integration
                        with the AgentCore observability dashboard. Only applicable
                        when DEPLOYMENT_TYPE is agentcore.

SNS_TOPIC_ARN:          [OPTIONAL: none]
                        SNS topic ARN for alarm notifications. If not provided,
                        creates a new SNS topic for agent observability alerts.
                        Typically read from SSM: /mlops/{PROJECT_NAME}/{ENV}/sns-alerts-arn

METRIC_NAMESPACE:       [OPTIONAL: {PROJECT_NAME}/StrandsAgents]
                        CloudWatch custom metric namespace for agent KPIs.

ERROR_RATE_THRESHOLD:   [OPTIONAL: 5]
                        Error rate threshold as a percentage. When agent error
                        rate exceeds this value, a CloudWatch alarm triggers.
```

---

## Task

Generate complete Strands Agent observability infrastructure:

```
{PROJECT_NAME}-agent-observability/
├── instrumentation/
│   ├── tracer_setup.py                # OTel tracer provider for Strands agents
│   ├── agent_tracing.py               # Agent loop and tool call span wrappers
│   ├── token_tracking.py              # Token usage tracking per invocation
│   └── span_attributes.py            # Agent-specific span attribute constants
├── metrics/
│   └── agent_metrics.py               # CloudWatch custom metrics publisher
├── dashboards/
│   ├── create_dashboard.py            # CloudWatch dashboard creation script
│   └── dashboard_widgets.py           # Widget definitions for agent KPIs
├── alarms/
│   ├── latency_alarm.py               # Alarm when agent latency exceeds threshold
│   └── error_rate_alarm.py            # Alarm on agent error rate spike
├── agentcore/
│   └── agentcore_observability.py     # AgentCore-native observability config
├── config.py                          # Central configuration
├── run_setup.py                       # CLI orchestrator
└── requirements.txt
```


**config.py**: Central configuration dataclass with all parameters. Load from environment variables or CLI args. Validate required fields. Construct resource names using `{PROJECT_NAME}-{component}-{ENV}` convention. Parse AGENT_NAMES from JSON string. Set defaults for LATENCY_THRESHOLD_MS (5000), DASHBOARD_NAME, METRIC_NAMESPACE, ERROR_RATE_THRESHOLD (5). Validate DEPLOYMENT_TYPE is one of `lambda` or `agentcore`.

**tracer_setup.py**: OpenTelemetry tracer provider initialization for Strands agents:
- Import `opentelemetry.api.trace`, `opentelemetry.sdk.trace`, `opentelemetry.sdk.resources`
- Create a `Resource` with `service.name={PROJECT_NAME}-strands-agent-{ENV}`, `service.version`, `deployment.environment={ENV}`, `cloud.provider=aws`, `cloud.region={AWS_REGION}`
- For Lambda mode: configure `TracerProvider` with `BatchSpanProcessor` and OTLP exporter pointing to the ADOT Lambda layer extension endpoint (`http://localhost:4317`)
- For AgentCore mode: configure `TracerProvider` with `BatchSpanProcessor` and OTLP exporter pointing to the AgentCore collector endpoint
- Export a `get_tracer(name)` function that returns a named tracer from the configured provider, defaulting to `"strands-agent"` tracer name
- Include `force_flush()` wrapper for Lambda cold start and shutdown handling
- Reuses the OTel provider pattern from `devops/10` (OpenTelemetry ML Tracing) with agent-specific resource attributes

**agent_tracing.py**: Agent loop and tool call span wrappers:
- `traced_agent_invocation(agent, prompt, session_id, agent_name)`: create a parent span named `strands.agent.invoke` with attributes `agent.name`, `agent.session_id`, `agent.prompt_length`. Call `agent(prompt)`. On response, add attributes `agent.latency_ms`, `agent.response_length`, `agent.loop_iterations`, `agent.tool_calls_count`. On error, record exception on span and set span status to ERROR. Publish metrics via `agent_metrics.publish_invocation_metrics()`.
- `traced_tool_call(tool_name, tool_input)`: create a child span named `strands.tool.{tool_name}` with attributes `tool.name`, `tool.input_size`. Measure execution time. On success, add `tool.latency_ms`, `tool.success=True`, `tool.output_size`. On error, set `tool.success=False`, record exception, set span status to ERROR.
- `create_agent_span_wrapper(agent_name)`: return a decorator/wrapper that automatically instruments any Strands Agent invocation with the tracing pattern above, suitable for wrapping `agent()` calls in Lambda handlers or orchestrators.
- Both functions use `tracer.start_as_current_span()` context manager for automatic span lifecycle management.

**token_tracking.py**: Token usage tracking per invocation:
- `TokenTracker` class that accumulates input and output token counts across agent loop iterations within a single invocation
- `record_tokens(input_tokens, output_tokens, model_id)`: add token counts to the running total, record per-iteration breakdown
- `get_summary()`: return dict with `total_input_tokens`, `total_output_tokens`, `total_tokens`, `iterations`, `avg_tokens_per_iteration`, `model_id`
- `add_to_span(span)`: set token tracking attributes on the current OTel span: `agent.tokens.input`, `agent.tokens.output`, `agent.tokens.total`, `agent.iterations`
- Integration point: called from `agent_tracing.py` after each agent invocation to capture token usage from the Strands agent response metadata

**span_attributes.py**: Agent-specific span attribute constants:
- Define constants for Strands agent span attribute keys: `AGENT_NAME = "agent.name"`, `AGENT_SESSION_ID = "agent.session_id"`, `AGENT_PROMPT_LENGTH = "agent.prompt_length"`, `AGENT_RESPONSE_LENGTH = "agent.response_length"`, `AGENT_LATENCY_MS = "agent.latency_ms"`, `AGENT_LOOP_ITERATIONS = "agent.loop_iterations"`, `AGENT_TOOL_CALLS_COUNT = "agent.tool_calls_count"`, `AGENT_TOKENS_INPUT = "agent.tokens.input"`, `AGENT_TOKENS_OUTPUT = "agent.tokens.output"`, `AGENT_TOKENS_TOTAL = "agent.tokens.total"`
- Define tool span attribute keys: `TOOL_NAME = "tool.name"`, `TOOL_INPUT_SIZE = "tool.input_size"`, `TOOL_OUTPUT_SIZE = "tool.output_size"`, `TOOL_LATENCY_MS = "tool.latency_ms"`, `TOOL_SUCCESS = "tool.success"`
- `build_agent_attributes(agent_name, session_id, latency_ms, **extra)`: helper that constructs a dict of span attributes from agent invocation results

**agent_metrics.py**: CloudWatch custom metrics publisher for agent KPIs:
- `publish_invocation_metrics(agent_name, latency_ms, tool_calls_count, error, tokens_total)`: publish metrics to CloudWatch namespace `{METRIC_NAMESPACE}` with dimensions `AgentName` and `Environment`:
  - `InvocationsPerMinute` (Count) — incremented per invocation
  - `AvgToolCallsPerInvocation` (Count) — tool calls in this invocation
  - `ErrorRate` (Count) — 1 if error, 0 if success (used with SampleCount statistic)
  - `P99Latency` (Milliseconds) — raw latency value (CloudWatch computes percentiles)
  - `TokensConsumed` (Count) — total tokens for this invocation
- `publish_tool_metrics(agent_name, tool_name, latency_ms, success)`: publish per-tool metrics with additional `ToolName` dimension:
  - `ToolCallCount` (Count) — incremented per tool call
  - `ToolLatency` (Milliseconds) — tool execution latency
  - `ToolErrorCount` (Count) — 1 if tool failed, 0 if success
- Use `cloudwatch.put_metric_data()` with batching (max 1000 metric data points per call)
- Handle `cloudwatch.put_metric_data()` failures gracefully — log warning and continue without blocking agent execution

**dashboard_widgets.py**: Widget definitions for agent KPIs:
- `build_dashboard_body(agent_names, metric_namespace, env)`: construct the CloudWatch dashboard JSON body with the following widgets:
  - **Agent Invocations**: metric widget showing `InvocationsPerMinute` per agent, Sum statistic, 60-second period
  - **Agent Latency (p50/p90/p99)**: metric widget showing `P99Latency` with p50, p90, p99 statistics, 300-second period, line chart
  - **Tool Usage Distribution**: metric widget showing `ToolCallCount` per tool name, Sum statistic, bar chart view
  - **Token Consumption Trends**: metric widget showing `TokensConsumed` per agent, Sum statistic, 300-second period, line chart
  - **Error Analysis**: metric widget showing `ErrorRate` per agent with SampleCount and Sum statistics, 300-second period
  - **Tool Latency Heatmap**: metric widget showing `ToolLatency` per tool name, p50/p90/p99 statistics
- Each widget includes proper title, dimensions, period, and stat configuration

**create_dashboard.py**: CloudWatch dashboard creation script:
- Call `cloudwatch.put_dashboard()` with `DashboardName=DASHBOARD_NAME` and `DashboardBody` from `dashboard_widgets.build_dashboard_body()`
- Tag the dashboard with `Project={PROJECT_NAME}`, `Environment={ENV}`
- Print the dashboard URL for quick access: `https://{AWS_REGION}.console.aws.amazon.com/cloudwatch/home?region={AWS_REGION}#dashboards:name={DASHBOARD_NAME}`

**latency_alarm.py**: CloudWatch alarm for agent latency threshold:
- Create a CloudWatch alarm using `cloudwatch.put_metric_alarm()`:
  - `AlarmName`: `{PROJECT_NAME}-agent-p99-latency-{ENV}`
  - `Namespace`: METRIC_NAMESPACE
  - `MetricName`: `P99Latency`
  - `Statistic`: `p99` (extended statistic)
  - `Threshold`: LATENCY_THRESHOLD_MS
  - `ComparisonOperator`: `GreaterThanThreshold`
  - `EvaluationPeriods`: 3
  - `Period`: 300 seconds
  - `AlarmActions`: [SNS_TOPIC_ARN]
  - `Dimensions`: `[{"Name": "Environment", "Value": ENV}]`
  - `TreatMissingData`: `notBreaching`
- Create per-agent latency alarms for each agent in AGENT_NAMES with additional `AgentName` dimension
- If SNS_TOPIC_ARN is not provided, create a new SNS topic `{PROJECT_NAME}-agent-latency-alerts-{ENV}`

**error_rate_alarm.py**: CloudWatch alarm for agent error rate:
- Create a CloudWatch alarm using metric math:
  - `m1`: `ErrorRate` metric with Sum statistic (total errors)
  - `m2`: `InvocationsPerMinute` metric with SampleCount statistic (total invocations)
  - `e1`: metric math expression `(m1 / m2) * 100` — error rate as percentage
  - `AlarmName`: `{PROJECT_NAME}-agent-error-rate-{ENV}`
  - `Threshold`: ERROR_RATE_THRESHOLD (default 5%)
  - `ComparisonOperator`: `GreaterThanThreshold`
  - `EvaluationPeriods`: 3
  - `Period`: 300 seconds
  - `AlarmActions`: [SNS_TOPIC_ARN]
  - `TreatMissingData`: `notBreaching`
- Create per-agent error rate alarms for each agent in AGENT_NAMES

**agentcore_observability.py**: AgentCore-native observability configuration:
- Only generated when DEPLOYMENT_TYPE is `agentcore`
- Configure AgentCore built-in observability features:
  - Enable session-level metrics (session duration, messages per session, tool calls per session)
  - Enable runtime health metrics (active sessions, memory usage, scaling events)
  - Configure trace export to X-Ray from AgentCore runtime
  - Set up integration with the AgentCore observability dashboard
- `configure_agentcore_observability(endpoint_id)`: call AgentCore APIs to enable observability on the deployed endpoint
- `get_agentcore_metrics(endpoint_id)`: retrieve AgentCore-native metrics for inclusion in the CloudWatch dashboard
- Include health check monitoring that publishes AgentCore endpoint status to CloudWatch

**run_setup.py**: CLI orchestrator that runs setup steps in order:
1. Load and validate configuration
2. Initialize OTel tracer provider (tracer_setup.py)
3. Create CloudWatch dashboard (create_dashboard.py)
4. Create latency alarm (latency_alarm.py)
5. Create error rate alarm (error_rate_alarm.py)
6. If DEPLOYMENT_TYPE is agentcore, configure AgentCore observability (agentcore_observability.py)
7. Print summary with dashboard URL, alarm ARNs, and tracer configuration

**requirements.txt**: Python dependencies including `strands-agents`, `boto3`, `opentelemetry-api`, `opentelemetry-sdk`, `opentelemetry-exporter-otlp-proto-grpc`.

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**OpenTelemetry Standards:** Use OpenTelemetry API and SDK (version 1.x+) for all agent instrumentation. Follow OpenTelemetry semantic conventions for span naming: `strands.agent.invoke` for agent invocations, `strands.tool.{tool_name}` for individual tool calls. Use `gen_ai.*` semantic convention attributes where applicable. Do not use X-Ray SDK directly — use ADOT (which includes the X-Ray exporter) for forward compatibility with the OpenTelemetry ecosystem. Reuse the OTel tracer provider patterns from `devops/10` (OpenTelemetry ML Tracing).

**ADOT Lambda Layer:** For Lambda-deployed agents (DEPLOYMENT_TYPE=lambda), use the AWS-managed ADOT Lambda layer. Set `AWS_LAMBDA_EXEC_WRAPPER=/opt/otel-instrument` to enable auto-instrumentation of boto3 calls. Add manual spans for agent loop stages and tool calls. Call `tracer_provider.force_flush()` before the Lambda handler returns to ensure all spans are exported before the execution environment freezes. The ADOT layer adds ~200ms cold start overhead — acceptable for agent workloads but document this tradeoff.

**Non-Blocking Metrics:** All CloudWatch metric publishing (`put_metric_data`) must be non-blocking. If metric publishing fails, log a warning and continue agent execution. Never let observability failures impact agent response latency or availability. Use try/except around all `cloudwatch.put_metric_data()` calls.

**Custom Metrics Dimensions:** Use `AgentName` and `Environment` as primary dimensions for all agent-level metrics. Add `ToolName` as an additional dimension for tool-level metrics. Keep dimension cardinality manageable — avoid using session IDs or prompt content as dimensions. CloudWatch charges $0.30/metric/month per unique dimension combination.

**Dashboard Design:** Dashboard widgets should provide at-a-glance agent health visibility. Include invocations (throughput), latency percentiles (performance), tool usage (behavior), token consumption (cost), and error analysis (reliability). Use 60-second periods for invocation counts and 300-second periods for latency and error metrics. Position high-priority widgets (invocations, latency) at the top of the dashboard.

**Alarm Configuration:** Use `TreatMissingData=notBreaching` for all alarms to avoid false positives during low-traffic periods. Set evaluation periods to 3 (15 minutes at 300-second period) to smooth out transient spikes. Use metric math for error rate alarms to compute percentage from raw error counts and invocation counts. Create per-agent alarms for production environments; use aggregate alarms for dev/stage.

**AgentCore Observability:** When DEPLOYMENT_TYPE is agentcore, leverage AgentCore-native observability features (session metrics, runtime health) in addition to custom CloudWatch metrics. AgentCore provides built-in tracing and metrics that complement the custom instrumentation from this template. Configure trace export to X-Ray from the AgentCore runtime for end-to-end trace correlation.

**Token Tracking:** Track input and output tokens separately per agent invocation. Aggregate token counts across agent loop iterations (a single invocation may trigger multiple model calls). Use token metrics for cost estimation and budget alerting. Token counts are extracted from Strands agent response metadata when available.

**Trace Correlation:** Ensure trace context propagation between agent invocations and downstream AWS service calls (Bedrock, DynamoDB, S3). Use the X-Ray propagator (`OTEL_PROPAGATORS=xray`) for automatic trace header propagation. For multi-agent patterns (mlops/21), create parent/child span relationships to visualize the full orchestration flow.

**Cost:** X-Ray charges $5.00 per million traces recorded. CloudWatch custom metrics cost $0.30/metric/month per unique dimension combination. CloudWatch dashboards cost $3.00/dashboard/month. In prod with multiple agents and tools, budget for 10-20 unique metric dimension combinations per agent. Use sampling in production (10% normal traces, 100% error traces) to control X-Ray costs.

**Naming:** All resources follow `{PROJECT_NAME}-{component}-{ENV}` convention:
- CloudWatch dashboard: `{PROJECT_NAME}-strands-agents-{ENV}`
- Latency alarm: `{PROJECT_NAME}-{agent_name}-p99-latency-{ENV}`
- Error rate alarm: `{PROJECT_NAME}-{agent_name}-error-rate-{ENV}`
- Metric namespace: `{PROJECT_NAME}/StrandsAgents`
- OTel service name: `{PROJECT_NAME}-strands-agent-{ENV}`
- SNS topic: `{PROJECT_NAME}-agent-latency-alerts-{ENV}`

---

## Code Scaffolding Hints

**OpenTelemetry tracer provider for Strands agents:**
```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource

resource = Resource.create({
    "service.name": f"{PROJECT_NAME}-strands-agent-{ENV}",
    "service.version": "1.0.0",
    "deployment.environment": ENV,
    "cloud.provider": "aws",
    "cloud.region": AWS_REGION,
    "cloud.account.id": AWS_ACCOUNT_ID,
})

provider = TracerProvider(resource=resource)
otlp_exporter = OTLPSpanExporter(endpoint="http://localhost:4317", insecure=True)
provider.add_span_processor(BatchSpanProcessor(otlp_exporter))
trace.set_tracer_provider(provider)

tracer = trace.get_tracer("strands-agent", "1.0.0")


def get_tracer(name: str = "strands-agent") -> trace.Tracer:
    """Return a named tracer from the configured provider."""
    return trace.get_tracer(name, "1.0.0")


def force_flush():
    """Flush all pending spans — call before Lambda returns."""
    provider.force_flush(timeout_millis=5000)
```

**Custom span creation for agent invocations:**
```python
import time
from opentelemetry import trace
from opentelemetry.trace import StatusCode
from strands import Agent

tracer = trace.get_tracer("strands-agent", "1.0.0")


def traced_agent_invocation(agent: Agent, prompt: str, session_id: str, agent_name: str):
    """Invoke a Strands agent with full OpenTelemetry tracing."""
    with tracer.start_as_current_span("strands.agent.invoke") as span:
        span.set_attribute("agent.name", agent_name)
        span.set_attribute("agent.session_id", session_id)
        span.set_attribute("agent.prompt_length", len(prompt))

        start = time.perf_counter()
        try:
            response = agent(prompt)
            latency_ms = (time.perf_counter() - start) * 1000

            span.set_attribute("agent.latency_ms", latency_ms)
            span.set_attribute("agent.response_length", len(str(response)))

            # Extract metrics from agent state if available
            if hasattr(agent, "trace"):
                span.set_attribute(
                    "agent.loop_iterations", agent.trace.get("loop_count", 0)
                )
                span.set_attribute(
                    "agent.tool_calls_count", agent.trace.get("tool_calls", 0)
                )

            return response

        except Exception as e:
            latency_ms = (time.perf_counter() - start) * 1000
            span.set_attribute("agent.latency_ms", latency_ms)
            span.set_status(StatusCode.ERROR, str(e))
            span.record_exception(e)
            raise
```

**Custom span creation for tool calls:**
```python
def traced_tool_call(tool_name: str, tool_input: dict, execute_fn):
    """Wrap a tool call with OpenTelemetry span."""
    with tracer.start_as_current_span(f"strands.tool.{tool_name}") as span:
        span.set_attribute("tool.name", tool_name)
        span.set_attribute("tool.input_size", len(str(tool_input)))

        start = time.perf_counter()
        try:
            result = execute_fn(tool_input)
            latency_ms = (time.perf_counter() - start) * 1000

            span.set_attribute("tool.latency_ms", latency_ms)
            span.set_attribute("tool.success", True)
            span.set_attribute("tool.output_size", len(str(result)))
            return result

        except Exception as e:
            latency_ms = (time.perf_counter() - start) * 1000
            span.set_attribute("tool.latency_ms", latency_ms)
            span.set_attribute("tool.success", False)
            span.set_status(StatusCode.ERROR, str(e))
            span.record_exception(e)
            raise
```

**Token tracking per invocation:**
```python
class TokenTracker:
    """Accumulate token usage across agent loop iterations."""

    def __init__(self):
        self.total_input_tokens = 0
        self.total_output_tokens = 0
        self.iterations = 0

    def record_tokens(self, input_tokens: int, output_tokens: int, model_id: str = ""):
        self.total_input_tokens += input_tokens
        self.total_output_tokens += output_tokens
        self.iterations += 1

    def get_summary(self) -> dict:
        total = self.total_input_tokens + self.total_output_tokens
        return {
            "total_input_tokens": self.total_input_tokens,
            "total_output_tokens": self.total_output_tokens,
            "total_tokens": total,
            "iterations": self.iterations,
            "avg_tokens_per_iteration": total / max(self.iterations, 1),
        }

    def add_to_span(self, span):
        summary = self.get_summary()
        span.set_attribute("agent.tokens.input", summary["total_input_tokens"])
        span.set_attribute("agent.tokens.output", summary["total_output_tokens"])
        span.set_attribute("agent.tokens.total", summary["total_tokens"])
        span.set_attribute("agent.iterations", summary["iterations"])
```

**CloudWatch custom metrics publisher for agent KPIs:**
```python
import boto3

cloudwatch = boto3.client("cloudwatch", region_name=AWS_REGION)

METRIC_NAMESPACE = f"{PROJECT_NAME}/StrandsAgents"


def publish_invocation_metrics(
    agent_name: str,
    latency_ms: float,
    tool_calls_count: int,
    error: bool,
    tokens_total: int,
):
    """Publish agent invocation metrics to CloudWatch."""
    try:
        cloudwatch.put_metric_data(
            Namespace=METRIC_NAMESPACE,
            MetricData=[
                {
                    "MetricName": "InvocationsPerMinute",
                    "Value": 1,
                    "Unit": "Count",
                    "Dimensions": [
                        {"Name": "AgentName", "Value": agent_name},
                        {"Name": "Environment", "Value": ENV},
                    ],
                },
                {
                    "MetricName": "AvgToolCallsPerInvocation",
                    "Value": tool_calls_count,
                    "Unit": "Count",
                    "Dimensions": [
                        {"Name": "AgentName", "Value": agent_name},
                        {"Name": "Environment", "Value": ENV},
                    ],
                },
                {
                    "MetricName": "ErrorRate",
                    "Value": 1 if error else 0,
                    "Unit": "Count",
                    "Dimensions": [
                        {"Name": "AgentName", "Value": agent_name},
                        {"Name": "Environment", "Value": ENV},
                    ],
                },
                {
                    "MetricName": "P99Latency",
                    "Value": latency_ms,
                    "Unit": "Milliseconds",
                    "Dimensions": [
                        {"Name": "AgentName", "Value": agent_name},
                        {"Name": "Environment", "Value": ENV},
                    ],
                },
                {
                    "MetricName": "TokensConsumed",
                    "Value": tokens_total,
                    "Unit": "Count",
                    "Dimensions": [
                        {"Name": "AgentName", "Value": agent_name},
                        {"Name": "Environment", "Value": ENV},
                    ],
                },
            ],
        )
    except Exception as e:
        # Non-blocking — log warning and continue agent execution
        import logging

        logging.getLogger(__name__).warning("Failed to publish metrics: %s", e)


def publish_tool_metrics(
    agent_name: str, tool_name: str, latency_ms: float, success: bool
):
    """Publish per-tool metrics to CloudWatch."""
    try:
        cloudwatch.put_metric_data(
            Namespace=METRIC_NAMESPACE,
            MetricData=[
                {
                    "MetricName": "ToolCallCount",
                    "Value": 1,
                    "Unit": "Count",
                    "Dimensions": [
                        {"Name": "AgentName", "Value": agent_name},
                        {"Name": "ToolName", "Value": tool_name},
                        {"Name": "Environment", "Value": ENV},
                    ],
                },
                {
                    "MetricName": "ToolLatency",
                    "Value": latency_ms,
                    "Unit": "Milliseconds",
                    "Dimensions": [
                        {"Name": "AgentName", "Value": agent_name},
                        {"Name": "ToolName", "Value": tool_name},
                        {"Name": "Environment", "Value": ENV},
                    ],
                },
                {
                    "MetricName": "ToolErrorCount",
                    "Value": 0 if success else 1,
                    "Unit": "Count",
                    "Dimensions": [
                        {"Name": "AgentName", "Value": agent_name},
                        {"Name": "ToolName", "Value": tool_name},
                        {"Name": "Environment", "Value": ENV},
                    ],
                },
            ],
        )
    except Exception as e:
        import logging

        logging.getLogger(__name__).warning("Failed to publish tool metrics: %s", e)
```

**CloudWatch dashboard JSON definition:**
```python
import json


def build_dashboard_body(agent_names: list, metric_namespace: str, env: str) -> str:
    """Build CloudWatch dashboard JSON body for Strands agent KPIs."""
    widgets = [
        {
            "type": "metric",
            "x": 0,
            "y": 0,
            "width": 12,
            "height": 6,
            "properties": {
                "title": "Agent Invocations",
                "metrics": [
                    [
                        metric_namespace,
                        "InvocationsPerMinute",
                        "AgentName",
                        name,
                        "Environment",
                        env,
                    ]
                    for name in agent_names
                ],
                "period": 60,
                "stat": "Sum",
                "view": "timeSeries",
            },
        },
        {
            "type": "metric",
            "x": 12,
            "y": 0,
            "width": 12,
            "height": 6,
            "properties": {
                "title": "Agent Latency (p50/p90/p99)",
                "metrics": [
                    item
                    for name in agent_names
                    for item in [
                        [
                            metric_namespace,
                            "P99Latency",
                            "AgentName",
                            name,
                            "Environment",
                            env,
                            {"stat": "p99", "label": f"{name} p99"},
                        ],
                        [
                            metric_namespace,
                            "P99Latency",
                            "AgentName",
                            name,
                            "Environment",
                            env,
                            {"stat": "p90", "label": f"{name} p90"},
                        ],
                        [
                            metric_namespace,
                            "P99Latency",
                            "AgentName",
                            name,
                            "Environment",
                            env,
                            {"stat": "p50", "label": f"{name} p50"},
                        ],
                    ]
                ],
                "period": 300,
                "view": "timeSeries",
            },
        },
        {
            "type": "metric",
            "x": 0,
            "y": 6,
            "width": 12,
            "height": 6,
            "properties": {
                "title": "Tool Usage Distribution",
                "metrics": [
                    [
                        metric_namespace,
                        "ToolCallCount",
                        "ToolName",
                        tool,
                        "Environment",
                        env,
                    ]
                    for tool in [
                        "retrieve",
                        "shell",
                        "editor",
                        "use_aws",
                        "think",
                        "browser",
                    ]
                ],
                "period": 300,
                "stat": "Sum",
                "view": "bar",
            },
        },
        {
            "type": "metric",
            "x": 12,
            "y": 6,
            "width": 12,
            "height": 6,
            "properties": {
                "title": "Token Consumption Trends",
                "metrics": [
                    [
                        metric_namespace,
                        "TokensConsumed",
                        "AgentName",
                        name,
                        "Environment",
                        env,
                    ]
                    for name in agent_names
                ],
                "period": 300,
                "stat": "Sum",
                "view": "timeSeries",
            },
        },
        {
            "type": "metric",
            "x": 0,
            "y": 12,
            "width": 12,
            "height": 6,
            "properties": {
                "title": "Error Analysis",
                "metrics": [
                    [
                        metric_namespace,
                        "ErrorRate",
                        "AgentName",
                        name,
                        "Environment",
                        env,
                        {"stat": "Sum", "label": f"{name} errors"},
                    ]
                    for name in agent_names
                ],
                "period": 300,
                "view": "timeSeries",
            },
        },
        {
            "type": "metric",
            "x": 12,
            "y": 12,
            "width": 12,
            "height": 6,
            "properties": {
                "title": "Tool Latency Heatmap",
                "metrics": [
                    item
                    for tool in ["retrieve", "shell", "editor", "use_aws"]
                    for item in [
                        [
                            metric_namespace,
                            "ToolLatency",
                            "ToolName",
                            tool,
                            "Environment",
                            env,
                            {"stat": "p99", "label": f"{tool} p99"},
                        ],
                        [
                            metric_namespace,
                            "ToolLatency",
                            "ToolName",
                            tool,
                            "Environment",
                            env,
                            {"stat": "p50", "label": f"{tool} p50"},
                        ],
                    ]
                ],
                "period": 300,
                "view": "timeSeries",
            },
        },
    ]

    return json.dumps({"widgets": widgets})
```

**CloudWatch latency alarm configuration:**
```python
import boto3

cloudwatch = boto3.client("cloudwatch", region_name=AWS_REGION)


def create_latency_alarm(
    agent_name: str,
    threshold_ms: float,
    sns_topic_arn: str,
    project_name: str,
    env: str,
    metric_namespace: str,
):
    """Create CloudWatch alarm for agent p99 latency threshold."""
    cloudwatch.put_metric_alarm(
        AlarmName=f"{project_name}-{agent_name}-p99-latency-{env}",
        AlarmDescription=(
            f"Strands agent {agent_name} p99 latency exceeds {threshold_ms}ms"
        ),
        Namespace=metric_namespace,
        MetricName="P99Latency",
        Dimensions=[
            {"Name": "AgentName", "Value": agent_name},
            {"Name": "Environment", "Value": env},
        ],
        ExtendedStatistic="p99",
        Period=300,
        EvaluationPeriods=3,
        Threshold=threshold_ms,
        ComparisonOperator="GreaterThanThreshold",
        AlarmActions=[sns_topic_arn],
        TreatMissingData="notBreaching",
        Tags=[
            {"Key": "Project", "Value": project_name},
            {"Key": "Environment", "Value": env},
        ],
    )
```

**CloudWatch error rate alarm with metric math:**
```python
def create_error_rate_alarm(
    agent_name: str,
    threshold_pct: float,
    sns_topic_arn: str,
    project_name: str,
    env: str,
    metric_namespace: str,
):
    """Create CloudWatch alarm for agent error rate using metric math."""
    cloudwatch.put_metric_alarm(
        AlarmName=f"{project_name}-{agent_name}-error-rate-{env}",
        AlarmDescription=(
            f"Strands agent {agent_name} error rate exceeds {threshold_pct}%"
        ),
        Metrics=[
            {
                "Id": "errors",
                "MetricStat": {
                    "Metric": {
                        "Namespace": metric_namespace,
                        "MetricName": "ErrorRate",
                        "Dimensions": [
                            {"Name": "AgentName", "Value": agent_name},
                            {"Name": "Environment", "Value": env},
                        ],
                    },
                    "Period": 300,
                    "Stat": "Sum",
                },
                "ReturnData": False,
            },
            {
                "Id": "invocations",
                "MetricStat": {
                    "Metric": {
                        "Namespace": metric_namespace,
                        "MetricName": "InvocationsPerMinute",
                        "Dimensions": [
                            {"Name": "AgentName", "Value": agent_name},
                            {"Name": "Environment", "Value": env},
                        ],
                    },
                    "Period": 300,
                    "Stat": "SampleCount",
                },
                "ReturnData": False,
            },
            {
                "Id": "error_rate",
                "Expression": "(errors / invocations) * 100",
                "Label": "Error Rate %",
                "ReturnData": True,
            },
        ],
        EvaluationPeriods=3,
        Threshold=threshold_pct,
        ComparisonOperator="GreaterThanThreshold",
        AlarmActions=[sns_topic_arn],
        TreatMissingData="notBreaching",
        Tags=[
            {"Key": "Project", "Value": project_name},
            {"Key": "Environment", "Value": env},
        ],
    )
```

**AgentCore observability configuration:**
```python
import boto3

agentcore_client = boto3.client("bedrock-agentcore", region_name=AWS_REGION)


def configure_agentcore_observability(endpoint_id: str, project_name: str, env: str):
    """Enable AgentCore-native observability on a deployed endpoint."""
    # Enable observability features on the AgentCore runtime endpoint
    agentcore_client.update_runtime_endpoint(
        endpointId=endpoint_id,
        observabilityConfiguration={
            "tracingEnabled": True,
            "metricsEnabled": True,
            "logLevel": "INFO" if env == "prod" else "DEBUG",
        },
    )

    # Retrieve AgentCore-native metrics for dashboard integration
    return get_agentcore_metrics(endpoint_id)


def get_agentcore_metrics(endpoint_id: str) -> dict:
    """Retrieve AgentCore-native metrics for dashboard integration."""
    response = agentcore_client.get_runtime_endpoint(endpointId=endpoint_id)
    return {
        "status": response.get("status"),
        "active_sessions": response.get("activeSessions", 0),
        "endpoint_id": endpoint_id,
    }
```

---

## Integration Points

- **Upstream**: `devops/10` → Base OpenTelemetry ML Tracing patterns — this template extends devops/10 with Strands agent-specific instrumentation, reusing the OTel tracer provider setup, ADOT collector configuration, and X-Ray trace group conventions
- **Upstream**: `devops/03` → CloudWatch Monitoring & Alerting — agent metrics, dashboards, and alarms integrate with the CloudWatch infrastructure patterns from devops/03
- **Upstream**: `devops/04` → IAM roles for X-Ray trace publishing (`xray:PutTraceSegments`, `xray:PutTelemetryRecords`) and CloudWatch metric/dashboard/alarm permissions
- **Upstream**: `devops/12` → Bedrock Invocation Logging — correlate Strands agent traces with Bedrock model invocation logs for end-to-end visibility from agent prompt to model response
- **Downstream**: `mlops/20` → Strands Agent Lambda Deployment — instrument Lambda-deployed agents with the tracing and metrics patterns from this template
- **Downstream**: `mlops/22` → Strands AgentCore Deployment — instrument AgentCore-deployed agents with AgentCore-native observability and custom CloudWatch metrics
- **Downstream**: `mlops/21` → Strands Multi-Agent Patterns — trace multi-agent orchestration with parent/child span relationships across agent invocations in Graph, Swarm, and Workflow patterns
- **Downstream**: `devops/16` → Agent Guardrails and Control — violation tracking and guardrail event metrics feed into the observability dashboard and alarms from this template
