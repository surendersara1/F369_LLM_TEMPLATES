<!-- Template Version: 1.1 | boto3: 1.35+ | opentelemetry-api: 1.27+ | opentelemetry-sdk: 1.27+ | aws-opentelemetry-distro: 0.7+ | Model IDs: 2026-04-22 refresh -->

# Template DevOps 10 — OpenTelemetry Distributed Tracing for ML Inference

## Purpose
Generate production-ready distributed tracing infrastructure for ML inference workloads: AWS Distro for OpenTelemetry (ADOT) Collector deployment as a Lambda layer or ECS sidecar, instrumented Lambda functions with custom spans covering request parsing, model invocation (Bedrock and SageMaker), response formatting, and guardrail evaluation, X-Ray trace groups with ML-specific filter expressions, custom span attributes for ML metadata (model ID, token counts, latency breakdowns), X-Ray Insights notifications for latency threshold breaches, and a trace analysis utility that computes p50/p95/p99 latency percentiles across inference pipeline stages.

---

## Role Definition

You are an expert AWS observability engineer and distributed tracing specialist with expertise in:
- OpenTelemetry: OTel API, SDK, context propagation, span processors, exporters, and semantic conventions
- AWS Distro for OpenTelemetry (ADOT): Lambda layer deployment, ECS sidecar collector, ADOT Collector configuration (receivers, processors, exporters)
- AWS X-Ray: trace groups, filter expressions, sampling rules, trace summaries, analytics, Insights notifications
- ML inference tracing: instrumenting `bedrock_runtime.invoke_model()`, `bedrock_runtime.converse()`, `sagemaker_runtime.invoke_endpoint()` with custom spans capturing model ID, latency, token usage, and error rates
- Lambda instrumentation: ADOT Lambda layer, auto-instrumentation for boto3, manual span creation for business logic stages
- ECS sidecar pattern: ADOT Collector as a sidecar container in ECS task definitions, shared localhost networking for trace export
- Custom span attributes: defining ML-specific attributes (model.id, model.provider, tokens.input, tokens.output, latency_ms, guardrail.action) following OpenTelemetry semantic conventions
- Trace analysis: querying X-Ray trace summaries for percentile latency computation, error rate analysis, and bottleneck identification across inference pipeline stages
- CloudWatch integration: publishing trace-derived metrics (latency percentiles, error rates) to CloudWatch for dashboarding and alarming

Generate complete, production-deployable distributed tracing infrastructure code.

---

## Context & Inputs

```
PROJECT_NAME:               [REQUIRED]
AWS_REGION:                 [REQUIRED]
AWS_ACCOUNT_ID:             [REQUIRED]
ENV:                        [REQUIRED - dev | stage | prod]

DEPLOYMENT_MODE:            [REQUIRED - lambda_layer | ecs_sidecar]
                            Determines how the ADOT Collector is deployed:
                            - lambda_layer: ADOT Lambda layer for Lambda-based inference functions.
                              Uses the AWS-managed ADOT Lambda layer ARN. Auto-instruments boto3
                              calls and supports manual span creation.
                            - ecs_sidecar: ADOT Collector as a sidecar container in ECS task
                              definitions. Listens on localhost:4317 (gRPC) and localhost:4318
                              (HTTP) for OTLP trace export from the application container.

COLLECTOR_CONFIG:           [OPTIONAL: default ADOT config exporting to X-Ray]
                            Custom ADOT Collector configuration YAML. If not provided,
                            generates a default config with:
                            - Receiver: OTLP (gRPC on 4317, HTTP on 4318)
                            - Processor: batch (timeout 5s, send_batch_size 256)
                            - Exporter: awsxray (region from AWS_REGION)
                            Only applicable for ecs_sidecar mode. Lambda layer uses
                            built-in configuration.

TRACE_GROUP_NAME:           [OPTIONAL: {PROJECT_NAME}-ml-inference-{ENV}]
                            X-Ray trace group name for filtering ML inference traces.
                            The trace group uses a filter expression to isolate traces
                            from ML inference workloads.

CUSTOM_SPAN_ATTRIBUTES:     [OPTIONAL: default ML attributes]
                            JSON object defining additional custom span attributes to
                            capture on inference spans. Merged with the default ML
                            attributes (model.id, model.provider, tokens.input,
                            tokens.output, latency_ms).
                            Example:
                            {
                              "customer.id": "string",
                              "request.priority": "string",
                              "cache.hit": "bool"
                            }

LATENCY_THRESHOLD_MS:       [OPTIONAL: 5000]
                            Latency threshold in milliseconds for X-Ray Insights
                            notifications. When p95 latency exceeds this threshold,
                            X-Ray Insights triggers a CloudWatch alarm and SNS notification.

EXPORT_DESTINATION:         [OPTIONAL: xray]
                            Trace export destination. Currently supports:
                            - xray: AWS X-Ray (default, recommended for AWS-native tracing)
                            Future options may include CloudWatch Logs or third-party backends.

INFERENCE_FUNCTION_NAME:    [OPTIONAL: {PROJECT_NAME}-inference-{ENV}]
                            Name of the Lambda function or ECS service being instrumented.
                            Used for trace group filter expressions and alarm naming.

SNS_TOPIC_ARN:              [OPTIONAL: none]
                            SNS topic ARN for X-Ray Insights latency threshold notifications.
                            If not provided, creates a new SNS topic.
                            Typically read from SSM: /mlops/{PROJECT_NAME}/{ENV}/sns-alerts-arn
```

---

## Task

Generate complete OpenTelemetry distributed tracing infrastructure for ML inference:

```
{PROJECT_NAME}-otel-ml-tracing/
├── collector/
│   ├── adot_lambda_layer.py              # ADOT Lambda layer configuration and setup
│   ├── adot_ecs_sidecar.py              # ADOT ECS sidecar task definition fragment
│   └── collector_config.yaml             # ADOT Collector configuration (ECS mode)
├── instrumentation/
│   ├── tracer_setup.py                   # OpenTelemetry tracer provider initialization
│   ├── bedrock_tracing.py               # Instrumented Bedrock invoke_model/converse wrappers
│   ├── sagemaker_tracing.py             # Instrumented SageMaker invoke_endpoint wrapper
│   ├── inference_handler.py             # Full inference Lambda handler with span stages
│   └── span_attributes.py              # ML-specific span attribute definitions
├── xray/
│   ├── create_trace_group.py            # X-Ray trace group with ML filter expression
│   ├── sampling_rules.py               # X-Ray sampling rules for ML inference
│   └── insights_notification.py         # X-Ray Insights + CloudWatch alarm for latency
├── analysis/
│   ├── trace_analysis.py               # Trace analysis utility (p50/p95/p99 latency)
│   └── bottleneck_report.py            # Identify slowest spans across inference stages
├── infrastructure/
│   ├── ssm_outputs.py                   # Write trace group ARN and config to SSM
│   └── config.py                        # Central configuration
├── run_setup.py                          # CLI orchestrator
└── requirements.txt
```

**config.py**: Central configuration dataclass with all parameters. Load from environment variables or CLI args. Validate required fields. Construct resource names using `{PROJECT_NAME}-{component}-{ENV}` convention. Parse CUSTOM_SPAN_ATTRIBUTES from JSON string. Set defaults for TRACE_GROUP_NAME, LATENCY_THRESHOLD_MS, EXPORT_DESTINATION, and INFERENCE_FUNCTION_NAME. Validate DEPLOYMENT_MODE is one of `lambda_layer` or `ecs_sidecar`.

**tracer_setup.py**: OpenTelemetry tracer provider initialization:
- Import `opentelemetry.api.trace`, `opentelemetry.sdk.trace`, `opentelemetry.sdk.resources`
- Create a `Resource` with `service.name={PROJECT_NAME}-inference-{ENV}`, `service.version`, `deployment.environment={ENV}`, `cloud.provider=aws`, `cloud.region={AWS_REGION}`
- For Lambda mode: configure `TracerProvider` with `BatchSpanProcessor` and OTLP exporter pointing to the ADOT Lambda layer extension endpoint (`http://localhost:4317`)
- For ECS mode: configure `TracerProvider` with `BatchSpanProcessor` and OTLP exporter pointing to the sidecar collector (`http://localhost:4317`)
- Export a `get_tracer(name)` function that returns a named tracer from the configured provider
- Include `force_flush()` wrapper for Lambda cold start and shutdown handling

**bedrock_tracing.py**: Instrumented Bedrock wrappers:
- `traced_invoke_model(model_id, body, **kwargs)`: create a span named `bedrock.invoke_model` with attributes `model.id`, `model.provider` (extracted from model_id), `gen_ai.system=aws.bedrock`. Call `bedrock_runtime.invoke_model()`. On response, add attributes `tokens.input`, `tokens.output` (parsed from response usage metadata), `latency_ms` (measured). On error, record exception on span and set span status to ERROR.
- `traced_converse(model_id, messages, **kwargs)`: create a span named `bedrock.converse` with attributes `model.id`, `model.provider`, `gen_ai.system=aws.bedrock`. Call `bedrock_runtime.converse()`. Parse `usage.inputTokens` and `usage.outputTokens` from response. Add `tokens.input`, `tokens.output`, `latency_ms`, `stop_reason` attributes.
- Both functions use `tracer.start_as_current_span()` context manager for automatic span lifecycle management.

**sagemaker_tracing.py**: Instrumented SageMaker wrapper:
- `traced_invoke_endpoint(endpoint_name, body, content_type, **kwargs)`: create a span named `sagemaker.invoke_endpoint` with attributes `sagemaker.endpoint.name`, `model.id` (from endpoint name or custom attribute), `request.content_type`. Call `sagemaker_runtime.invoke_endpoint()`. Add `response.content_type`, `latency_ms`, `response.size_bytes` attributes. On error, record exception and set span status.

**inference_handler.py**: Full inference Lambda handler with span stages:
- Create a root span `inference.request` for the entire request lifecycle
- Child span `inference.parse_request`: parse and validate the incoming event/request body. Add `request.source`, `request.size_bytes` attributes.
- Child span `inference.model_invocation`: call `traced_invoke_model()` or `traced_converse()` from `bedrock_tracing.py` (or `traced_invoke_endpoint()` from `sagemaker_tracing.py`). This is the primary latency contributor.
- Child span `inference.guardrail_evaluation` (optional): if guardrails are enabled, trace the guardrail check. Add `guardrail.id`, `guardrail.action` (ALLOW/BLOCK), `guardrail.latency_ms` attributes.
- Child span `inference.format_response`: format the model output into the API response. Add `response.size_bytes`, `response.format` attributes.
- Add all CUSTOM_SPAN_ATTRIBUTES to the root span.
- Call `tracer_provider.force_flush()` before returning (critical for Lambda to ensure spans are exported).

**span_attributes.py**: ML-specific span attribute definitions:
- Define constants for standard ML span attribute keys: `MODEL_ID = "model.id"`, `MODEL_PROVIDER = "model.provider"`, `TOKENS_INPUT = "tokens.input"`, `TOKENS_OUTPUT = "tokens.output"`, `LATENCY_MS = "latency_ms"`, `GUARDRAIL_ID = "guardrail.id"`, `GUARDRAIL_ACTION = "guardrail.action"`, `GEN_AI_SYSTEM = "gen_ai.system"`, `STOP_REASON = "stop_reason"`
- `build_ml_attributes(model_id, latency_ms, tokens_in, tokens_out, **extra)`: helper that constructs a dict of span attributes from ML inference results, merging with CUSTOM_SPAN_ATTRIBUTES.
- `parse_model_provider(model_id)`: extract provider from Bedrock model ID (e.g., `anthropic` from `us.anthropic.claude-sonnet-4-7-20260109-v1:0`).

**adot_lambda_layer.py**: ADOT Lambda layer configuration:
- Provide the ADOT Lambda layer ARN for the target region: `arn:aws:lambda:{AWS_REGION}:901920570463:layer:aws-otel-python-amd64-ver-1-25-0:1` (update version as needed)
- Generate Lambda function configuration updates using `lambda_client.update_function_configuration()`:
  - Add ADOT layer ARN to `Layers`
  - Set environment variable `AWS_LAMBDA_EXEC_WRAPPER=/opt/otel-instrument` for auto-instrumentation
  - Set `OTEL_SERVICE_NAME={PROJECT_NAME}-inference-{ENV}`
  - Set `OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317`
  - Set `OTEL_TRACES_EXPORTER=otlp`
  - Set `OTEL_PROPAGATORS=xray`
  - Set `OTEL_PYTHON_DISABLED_INSTRUMENTATIONS=urllib3` (reduce noise from health checks)
- Include IAM policy additions for `xray:PutTraceSegments`, `xray:PutTelemetryRecords`

**adot_ecs_sidecar.py**: ADOT ECS sidecar task definition fragment:
- Generate a container definition for the ADOT Collector sidecar:
  - Image: `public.ecr.aws/aws-observability/aws-otel-collector:latest`
  - Port mappings: 4317 (gRPC OTLP), 4318 (HTTP OTLP), 13133 (health check)
  - Environment: `AOT_CONFIG_CONTENT` set to the collector config YAML (from `collector_config.yaml`)
  - Essential: `false` (sidecar should not stop the main container)
  - Log configuration: awslogs driver to CloudWatch log group
- Generate the application container environment variables:
  - `OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317`
  - `OTEL_SERVICE_NAME={PROJECT_NAME}-inference-{ENV}`
  - `OTEL_TRACES_EXPORTER=otlp`
  - `OTEL_PROPAGATORS=xray`
- Include ECS task role IAM policy additions for `xray:PutTraceSegments`, `xray:PutTelemetryRecords`

**collector_config.yaml**: ADOT Collector configuration for ECS sidecar mode:
- Receivers: `otlp` with gRPC (0.0.0.0:4317) and HTTP (0.0.0.0:4318) endpoints
- Processors: `batch` with timeout 5s and send_batch_size 256; `resource` processor to add `cloud.provider`, `cloud.region`, `service.name` attributes
- Exporters: `awsxray` with region set to AWS_REGION
- Service pipeline: traces → [otlp receiver] → [batch processor, resource processor] → [awsxray exporter]
- Extensions: `health_check` on port 13133

**create_trace_group.py**: X-Ray trace group with ML filter expression:
- Call `xray.create_group()` with:
  - `GroupName`: TRACE_GROUP_NAME (default `{PROJECT_NAME}-ml-inference-{ENV}`)
  - `FilterExpression`: `service("{PROJECT_NAME}-inference-{ENV}") AND annotation.model_id BEGINSWITH "anthropic" OR annotation.model_id BEGINSWITH "amazon" OR annotation.model_id BEGINSWITH "meta" OR http.url CONTAINS "invoke"` — filter to ML inference traces
  - `InsightsConfiguration`: `InsightsEnabled=True`, `NotificationsEnabled=True`
- Tag the group with `Project={PROJECT_NAME}`, `Environment={ENV}`
- Return the group ARN for SSM storage

**sampling_rules.py**: X-Ray sampling rules for ML inference:
- Create a sampling rule using `xray.create_sampling_rule()`:
  - `RuleName`: `{PROJECT_NAME}-ml-inference-{ENV}`
  - `Priority`: 100
  - `FixedRate`: 1.0 for dev (trace everything), 0.1 for prod (10% sampling)
  - `ReservoirSize`: 10 (guaranteed traces per second)
  - `ServiceName`: `{PROJECT_NAME}-inference-{ENV}`
  - `ServiceType`: `AWS::Lambda::Function` or `AWS::ECS::Container` based on DEPLOYMENT_MODE
  - `Host`: `*`
  - `HTTPMethod`: `*`
  - `URLPath`: `*`
  - `ResourceARN`: `*`
- In prod, create a second rule for error traces with `FixedRate=1.0` to capture all errors

**insights_notification.py**: X-Ray Insights + CloudWatch alarm for latency:
- X-Ray Insights automatically detects anomalies when enabled on the trace group
- Create a CloudWatch alarm for the X-Ray Insights notification:
  - Use EventBridge rule to capture X-Ray Insights events: `source=aws.xray`, `detail-type=X-Ray Insight`
  - Target: SNS topic (SNS_TOPIC_ARN or newly created topic)
  - Filter pattern: match insights where `State=ACTIVE` and the trace group matches TRACE_GROUP_NAME
- Create a CloudWatch alarm on the custom metric `{PROJECT_NAME}/MLInference/P95Latency` (published by `trace_analysis.py`):
  - Threshold: LATENCY_THRESHOLD_MS
  - Period: 300 seconds
  - Evaluation periods: 2
  - Alarm action: SNS_TOPIC_ARN

**trace_analysis.py**: Trace analysis utility for latency percentiles:
- `get_trace_summaries(start_time, end_time)`: call `xray.get_trace_summaries()` with `FilterExpression` matching the trace group, paginate through all results
- `compute_latency_percentiles(trace_summaries)`: compute p50, p95, p99 latency from `ResponseTime` field across all trace summaries. Return dict with `p50_ms`, `p95_ms`, `p99_ms`, `count`, `error_rate`.
- `get_span_breakdown(trace_ids)`: call `xray.batch_get_traces()` for a sample of trace IDs, parse segments and subsegments to compute per-stage latency breakdown (parse_request, model_invocation, guardrail_evaluation, format_response). Return sorted list of stages by average latency.
- `publish_latency_metrics(percentiles)`: publish p50, p95, p99 latency to CloudWatch custom metrics under namespace `{PROJECT_NAME}/MLInference` with dimensions `Environment={ENV}`, `Service=inference`.

**bottleneck_report.py**: Identify slowest spans across inference stages:
- Call `trace_analysis.get_span_breakdown()` for recent traces
- Rank stages by average latency contribution
- Identify traces where `model_invocation` exceeds LATENCY_THRESHOLD_MS
- Generate a JSON report with: top 5 slowest traces, per-stage latency averages, error rate per stage, recommendations (e.g., "model_invocation accounts for 85% of total latency — consider prompt caching via mlops/18")

**ssm_outputs.py**: Write trace configuration to SSM Parameter Store:
- Path: `/mlops/{PROJECT_NAME}/{ENV}/xray-trace-group-arn` — trace group ARN
- Path: `/mlops/{PROJECT_NAME}/{ENV}/otel-service-name` — OTel service name
- Path: `/mlops/{PROJECT_NAME}/{ENV}/otel-collector-endpoint` — collector OTLP endpoint
- Other templates read from these SSM paths for trace correlation

**run_setup.py**: CLI orchestrator that runs setup steps in order:
1. Load and validate configuration
2. Create X-Ray trace group with ML filter expression
3. Create X-Ray sampling rules
4. Configure ADOT Collector (Lambda layer or ECS sidecar based on DEPLOYMENT_MODE)
5. Set up X-Ray Insights notification and CloudWatch alarm
6. Write outputs to SSM
7. Run initial trace analysis (if traces exist)
8. Print summary with trace group ARN, sampling rule, collector config, and alarm ARN

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**OpenTelemetry Standards:** Use OpenTelemetry API and SDK (version 1.27+) for all instrumentation. Follow OpenTelemetry semantic conventions for span naming: `{system}.{operation}` (e.g., `bedrock.invoke_model`, `sagemaker.invoke_endpoint`). Use `gen_ai.*` semantic convention attributes where applicable (e.g., `gen_ai.system`, `gen_ai.request.model`). Do not use X-Ray SDK directly — use ADOT (which includes the X-Ray exporter) for forward compatibility with OpenTelemetry ecosystem.

**ADOT Lambda Layer:** For Lambda deployment, use the AWS-managed ADOT Lambda layer. Set `AWS_LAMBDA_EXEC_WRAPPER=/opt/otel-instrument` to enable auto-instrumentation of boto3 calls. Auto-instrumentation captures basic AWS SDK spans automatically; add manual spans for business logic stages (request parsing, guardrail evaluation, response formatting). Call `tracer_provider.force_flush()` before the Lambda handler returns to ensure all spans are exported before the execution environment freezes. The ADOT layer adds ~200ms cold start overhead — acceptable for inference workloads but document this tradeoff.

**ADOT ECS Sidecar:** For ECS deployment, run the ADOT Collector as a sidecar container in the same task definition. The application container exports traces to `localhost:4317` (gRPC OTLP). The sidecar batches and exports to X-Ray. Set the sidecar as `essential: false` so the main container continues if the collector crashes. Use the `batch` processor with `timeout: 5s` and `send_batch_size: 256` to balance latency and throughput. Health check the collector on port 13133.

**Custom Span Attributes:** Every ML inference span must include these attributes: `model.id` (Bedrock model ID or SageMaker endpoint name), `model.provider` (extracted from model ID), `tokens.input` (input token count), `tokens.output` (output token count), `latency_ms` (measured invocation latency). Optional attributes: `guardrail.id`, `guardrail.action`, `cache.hit`, `customer.id`. Use `span.set_attribute()` for individual attributes. Attribute values must be strings, integers, floats, or booleans — not nested objects.

**X-Ray Trace Groups:** Create a dedicated trace group for ML inference traces using a filter expression that matches the OTel service name and ML-specific annotations. Trace groups enable focused analysis, separate sampling rules, and Insights anomaly detection scoped to ML workloads. Enable Insights on the trace group for automatic anomaly detection. Insights detects latency spikes, error rate increases, and throughput changes.

**X-Ray Sampling:** In dev, sample 100% of traces (`FixedRate=1.0`) for full visibility. In prod, sample 10% of normal traces (`FixedRate=0.1`) but 100% of error traces (`FixedRate=1.0` with fault filter) to capture all failures. Use `ReservoirSize=10` to guarantee at least 10 traces per second regardless of sampling rate. Create separate sampling rules for ML inference vs. other workloads.

**Latency Budgets:** Define latency budgets per inference stage: request parsing (<10ms), model invocation (varies by model, typically 500-5000ms), guardrail evaluation (<100ms), response formatting (<10ms). The trace analysis utility compares actual latencies against these budgets and flags violations. Publish p50/p95/p99 latency metrics to CloudWatch for alarming.

**Context Propagation:** Use the X-Ray propagator (`OTEL_PROPAGATORS=xray`) for trace context propagation across AWS services. This ensures traces from Lambda → Bedrock → DynamoDB are correlated. For cross-service calls (e.g., API Gateway → Lambda → Bedrock), the X-Ray trace header (`X-Amzn-Trace-Id`) propagates automatically when using ADOT.

**Cost:** X-Ray charges $5.00 per million traces recorded and $0.50 per million traces retrieved. In prod with 10% sampling, a service handling 1M requests/day generates ~100K traces/day (~$15/month). ADOT Collector is free. CloudWatch custom metrics cost $0.30/metric/month. Budget for X-Ray costs in high-throughput inference scenarios.

**Naming:** All resources follow `{PROJECT_NAME}-{component}-{ENV}` convention:
- Trace group: `{PROJECT_NAME}-ml-inference-{ENV}`
- Sampling rule: `{PROJECT_NAME}-ml-inference-{ENV}`
- OTel service name: `{PROJECT_NAME}-inference-{ENV}`
- CloudWatch alarm: `{PROJECT_NAME}-ml-p95-latency-{ENV}`
- SSM parameter: `/mlops/{PROJECT_NAME}/{ENV}/xray-trace-group-arn`
- SNS topic: `{PROJECT_NAME}-ml-latency-alerts-{ENV}`

---

## Code Scaffolding Hints

**OpenTelemetry tracer provider initialization:**
```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource

resource = Resource.create({
    "service.name": f"{PROJECT_NAME}-inference-{ENV}",
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

tracer = trace.get_tracer("ml-inference", "1.0.0")


def force_flush():
    """Flush all pending spans — call before Lambda returns."""
    provider.force_flush(timeout_millis=5000)
```

**Instrumented Bedrock invoke_model with custom spans:**
```python
import boto3
import json
import time
from opentelemetry import trace
from opentelemetry.trace import StatusCode

bedrock_runtime = boto3.client("bedrock-runtime", region_name=AWS_REGION)
tracer = trace.get_tracer("ml-inference")


def parse_model_provider(model_id: str) -> str:
    """Extract provider from Bedrock model ID (e.g., 'anthropic' from 'us.anthropic.claude-sonnet-4-7...')."""
    return model_id.split(".")[0] if "." in model_id else "unknown"


def traced_invoke_model(model_id: str, body: dict, **kwargs):
    """Invoke Bedrock model with OpenTelemetry tracing."""
    with tracer.start_as_current_span("bedrock.invoke_model") as span:
        span.set_attribute("model.id", model_id)
        span.set_attribute("model.provider", parse_model_provider(model_id))
        span.set_attribute("gen_ai.system", "aws.bedrock")
        span.set_attribute("gen_ai.request.model", model_id)

        start = time.perf_counter()
        try:
            response = bedrock_runtime.invoke_model(
                modelId=model_id,
                body=json.dumps(body),
                contentType="application/json",
                accept="application/json",
                **kwargs,
            )
            latency_ms = (time.perf_counter() - start) * 1000
            response_body = json.loads(response["body"].read())

            # Extract token usage (Claude format)
            usage = response_body.get("usage", {})
            span.set_attribute("tokens.input", usage.get("input_tokens", 0))
            span.set_attribute("tokens.output", usage.get("output_tokens", 0))
            span.set_attribute("latency_ms", latency_ms)
            span.set_attribute("stop_reason", response_body.get("stop_reason", "unknown"))

            return response_body

        except Exception as e:
            latency_ms = (time.perf_counter() - start) * 1000
            span.set_attribute("latency_ms", latency_ms)
            span.set_status(StatusCode.ERROR, str(e))
            span.record_exception(e)
            raise
```

**Instrumented Bedrock Converse API with custom spans:**
```python
def traced_converse(model_id: str, messages: list, system: list = None, **kwargs):
    """Invoke Bedrock Converse API with OpenTelemetry tracing."""
    with tracer.start_as_current_span("bedrock.converse") as span:
        span.set_attribute("model.id", model_id)
        span.set_attribute("model.provider", parse_model_provider(model_id))
        span.set_attribute("gen_ai.system", "aws.bedrock")
        span.set_attribute("gen_ai.request.model", model_id)

        start = time.perf_counter()
        try:
            converse_params = {"modelId": model_id, "messages": messages, **kwargs}
            if system:
                converse_params["system"] = system

            response = bedrock_runtime.converse(**converse_params)
            latency_ms = (time.perf_counter() - start) * 1000

            usage = response.get("usage", {})
            span.set_attribute("tokens.input", usage.get("inputTokens", 0))
            span.set_attribute("tokens.output", usage.get("outputTokens", 0))
            span.set_attribute("latency_ms", latency_ms)
            span.set_attribute("stop_reason", response.get("stopReason", "unknown"))

            return response

        except Exception as e:
            latency_ms = (time.perf_counter() - start) * 1000
            span.set_attribute("latency_ms", latency_ms)
            span.set_status(StatusCode.ERROR, str(e))
            span.record_exception(e)
            raise
```

**Instrumented SageMaker invoke_endpoint with custom spans:**
```python
sagemaker_runtime = boto3.client("sagemaker-runtime", region_name=AWS_REGION)


def traced_invoke_endpoint(endpoint_name: str, body: bytes, content_type: str = "application/json", **kwargs):
    """Invoke SageMaker endpoint with OpenTelemetry tracing."""
    with tracer.start_as_current_span("sagemaker.invoke_endpoint") as span:
        span.set_attribute("sagemaker.endpoint.name", endpoint_name)
        span.set_attribute("model.id", endpoint_name)
        span.set_attribute("request.content_type", content_type)
        span.set_attribute("request.size_bytes", len(body))

        start = time.perf_counter()
        try:
            response = sagemaker_runtime.invoke_endpoint(
                EndpointName=endpoint_name,
                Body=body,
                ContentType=content_type,
                **kwargs,
            )
            latency_ms = (time.perf_counter() - start) * 1000
            response_body = response["Body"].read()

            span.set_attribute("latency_ms", latency_ms)
            span.set_attribute("response.content_type", response.get("ContentType", ""))
            span.set_attribute("response.size_bytes", len(response_body))

            return response_body

        except Exception as e:
            latency_ms = (time.perf_counter() - start) * 1000
            span.set_attribute("latency_ms", latency_ms)
            span.set_status(StatusCode.ERROR, str(e))
            span.record_exception(e)
            raise
```

**Full inference Lambda handler with span stages:**
```python
import json
from tracer_setup import tracer, force_flush
from bedrock_tracing import traced_converse
from span_attributes import build_ml_attributes

def lambda_handler(event, context):
    """Inference Lambda handler with full OpenTelemetry span instrumentation."""
    with tracer.start_as_current_span("inference.request") as root_span:
        root_span.set_attribute("faas.invocation_id", context.aws_request_id)

        # Stage 1: Parse request
        with tracer.start_as_current_span("inference.parse_request") as parse_span:
            body = json.loads(event.get("body", "{}"))
            model_id = body.get("model_id", "us.anthropic.claude-sonnet-4-7-20260109-v1:0")
            messages = body.get("messages", [])
            parse_span.set_attribute("request.source", event.get("requestContext", {}).get("identity", {}).get("sourceIp", "unknown"))
            parse_span.set_attribute("request.size_bytes", len(event.get("body", "")))

        # Stage 2: Model invocation
        with tracer.start_as_current_span("inference.model_invocation"):
            response = traced_converse(
                model_id=model_id,
                messages=messages,
                system=[{"text": "You are a helpful assistant."}],
            )

        # Stage 3: Guardrail evaluation (optional)
        guardrail_action = "NONE"
        # with tracer.start_as_current_span("inference.guardrail_evaluation") as guard_span:
        #     guardrail_result = evaluate_guardrail(response)
        #     guardrail_action = guardrail_result.get("action", "ALLOW")
        #     guard_span.set_attribute("guardrail.id", GUARDRAIL_ID)
        #     guard_span.set_attribute("guardrail.action", guardrail_action)

        # Stage 4: Format response
        with tracer.start_as_current_span("inference.format_response") as fmt_span:
            output_text = response["output"]["message"]["content"][0]["text"]
            result = {"statusCode": 200, "body": json.dumps({"response": output_text})}
            fmt_span.set_attribute("response.size_bytes", len(result["body"]))

        # Add ML attributes to root span
        root_span.set_attribute("model.id", model_id)
        root_span.set_attribute("guardrail.action", guardrail_action)

    force_flush()
    return result
```

**ADOT Collector configuration (ECS sidecar):**
```yaml
# collector_config.yaml — ADOT Collector configuration for ECS sidecar mode
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 5s
    send_batch_size: 256
  resource:
    attributes:
      - key: cloud.provider
        value: aws
        action: upsert
      - key: cloud.region
        value: ${AWS_REGION}
        action: upsert

exporters:
  awsxray:
    region: ${AWS_REGION}

extensions:
  health_check:
    endpoint: 0.0.0.0:13133

service:
  extensions: [health_check]
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch, resource]
      exporters: [awsxray]
```

**ADOT Lambda layer configuration:**
```python
import boto3

lambda_client = boto3.client("lambda", region_name=AWS_REGION)

# AWS-managed ADOT Python Lambda layer ARN (update version as needed)
ADOT_LAYER_ARN = f"arn:aws:lambda:{AWS_REGION}:901920570463:layer:aws-otel-python-amd64-ver-1-25-0:1"

def configure_lambda_adot(function_name: str):
    """Add ADOT Lambda layer and configure OpenTelemetry environment variables."""
    # Get current configuration
    current = lambda_client.get_function_configuration(FunctionName=function_name)
    current_layers = [layer["Arn"] for layer in current.get("Layers", [])]

    # Add ADOT layer if not present
    if ADOT_LAYER_ARN not in current_layers:
        current_layers.append(ADOT_LAYER_ARN)

    # Merge environment variables
    current_env = current.get("Environment", {}).get("Variables", {})
    current_env.update({
        "AWS_LAMBDA_EXEC_WRAPPER": "/opt/otel-instrument",
        "OTEL_SERVICE_NAME": f"{PROJECT_NAME}-inference-{ENV}",
        "OTEL_EXPORTER_OTLP_ENDPOINT": "http://localhost:4317",
        "OTEL_TRACES_EXPORTER": "otlp",
        "OTEL_PROPAGATORS": "xray",
        "OTEL_PYTHON_DISABLED_INSTRUMENTATIONS": "urllib3",
    })

    lambda_client.update_function_configuration(
        FunctionName=function_name,
        Layers=current_layers,
        Environment={"Variables": current_env},
    )
    print(f"ADOT layer configured for {function_name}")
```

**ADOT ECS sidecar task definition fragment:**
```python
adot_sidecar_container = {
    "name": "adot-collector",
    "image": "public.ecr.aws/aws-observability/aws-otel-collector:latest",
    "essential": False,
    "portMappings": [
        {"containerPort": 4317, "protocol": "tcp"},   # gRPC OTLP
        {"containerPort": 4318, "protocol": "tcp"},   # HTTP OTLP
        {"containerPort": 13133, "protocol": "tcp"},  # Health check
    ],
    "environment": [
        {"name": "AOT_CONFIG_CONTENT", "value": COLLECTOR_CONFIG_YAML},
    ],
    "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
            "awslogs-group": f"/ecs/{PROJECT_NAME}-adot-{ENV}",
            "awslogs-region": AWS_REGION,
            "awslogs-stream-prefix": "adot",
        },
    },
    "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:13133/ || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
    },
}

# Application container environment variables for OTel export
app_container_otel_env = [
    {"name": "OTEL_EXPORTER_OTLP_ENDPOINT", "value": "http://localhost:4317"},
    {"name": "OTEL_SERVICE_NAME", "value": f"{PROJECT_NAME}-inference-{ENV}"},
    {"name": "OTEL_TRACES_EXPORTER", "value": "otlp"},
    {"name": "OTEL_PROPAGATORS", "value": "xray"},
]
```

**X-Ray trace group with ML filter expression:**
```python
import boto3

xray = boto3.client("xray", region_name=AWS_REGION)

def create_ml_trace_group(group_name: str):
    """Create X-Ray trace group for ML inference traces with Insights enabled."""
    filter_expr = (
        f'service("{PROJECT_NAME}-inference-{ENV}") '
        f'AND (annotation.model_id BEGINSWITH "anthropic" '
        f'OR annotation.model_id BEGINSWITH "amazon" '
        f'OR annotation.model_id BEGINSWITH "meta" '
        f'OR annotation.model_id BEGINSWITH "cohere")'
    )

    response = xray.create_group(
        GroupName=group_name,
        FilterExpression=filter_expr,
        InsightsConfiguration={
            "InsightsEnabled": True,
            "NotificationsEnabled": True,
        },
        Tags=[
            {"Key": "Project", "Value": PROJECT_NAME},
            {"Key": "Environment", "Value": ENV},
        ],
    )
    group_arn = response["Group"]["GroupARN"]
    print(f"X-Ray trace group created: {group_name} ({group_arn})")
    return group_arn
```

**X-Ray sampling rules:**
```python
def create_ml_sampling_rule(env: str):
    """Create X-Ray sampling rule for ML inference traces."""
    fixed_rate = 1.0 if env == "dev" else 0.1

    response = xray.create_sampling_rule(
        SamplingRule={
            "RuleName": f"{PROJECT_NAME}-ml-inference-{env}",
            "Priority": 100,
            "FixedRate": fixed_rate,
            "ReservoirSize": 10,
            "ServiceName": f"{PROJECT_NAME}-inference-{env}",
            "ServiceType": "*",
            "Host": "*",
            "HTTPMethod": "*",
            "URLPath": "*",
            "ResourceARN": "*",
            "Version": 1,
        }
    )
    print(f"Sampling rule created: {PROJECT_NAME}-ml-inference-{env} (rate={fixed_rate})")
    return response["SamplingRuleRecord"]

    # In prod, also create an error sampling rule (100% of errors)
    if env == "prod":
        xray.create_sampling_rule(
            SamplingRule={
                "RuleName": f"{PROJECT_NAME}-ml-errors-{env}",
                "Priority": 50,
                "FixedRate": 1.0,
                "ReservoirSize": 5,
                "ServiceName": f"{PROJECT_NAME}-inference-{env}",
                "ServiceType": "*",
                "Host": "*",
                "HTTPMethod": "*",
                "URLPath": "*",
                "ResourceARN": "*",
                "Version": 1,
                "Attributes": {"error": "true"},
            }
        )
        print(f"Error sampling rule created: {PROJECT_NAME}-ml-errors-{env} (rate=1.0)")
```

**Trace analysis utility — p50/p95/p99 latency:**
```python
import boto3
import statistics
from datetime import datetime, timedelta

xray = boto3.client("xray", region_name=AWS_REGION)
cloudwatch = boto3.client("cloudwatch", region_name=AWS_REGION)


def get_trace_summaries(start_time: datetime, end_time: datetime) -> list:
    """Retrieve X-Ray trace summaries for ML inference traces."""
    summaries = []
    paginator = xray.get_paginator("get_trace_summaries")
    for page in paginator.paginate(
        StartTime=start_time,
        EndTime=end_time,
        FilterExpression=f'service("{PROJECT_NAME}-inference-{ENV}")',
        Sampling=False,
    ):
        summaries.extend(page.get("TraceSummaries", []))
    return summaries


def compute_latency_percentiles(summaries: list) -> dict:
    """Compute p50, p95, p99 latency from trace summaries."""
    if not summaries:
        return {"p50_ms": 0, "p95_ms": 0, "p99_ms": 0, "count": 0, "error_rate": 0.0}

    latencies = [s["ResponseTime"] * 1000 for s in summaries]  # Convert to ms
    errors = sum(1 for s in summaries if s.get("HasError", False) or s.get("HasFault", False))

    latencies.sort()
    n = len(latencies)
    return {
        "p50_ms": round(latencies[int(n * 0.50)], 2),
        "p95_ms": round(latencies[int(n * 0.95)], 2),
        "p99_ms": round(latencies[int(n * 0.99)], 2),
        "count": n,
        "error_rate": round(errors / n, 4),
    }


def publish_latency_metrics(percentiles: dict):
    """Publish latency percentiles to CloudWatch custom metrics."""
    cloudwatch.put_metric_data(
        Namespace=f"{PROJECT_NAME}/MLInference",
        MetricData=[
            {
                "MetricName": "P50Latency",
                "Value": percentiles["p50_ms"],
                "Unit": "Milliseconds",
                "Dimensions": [
                    {"Name": "Environment", "Value": ENV},
                    {"Name": "Service", "Value": "inference"},
                ],
            },
            {
                "MetricName": "P95Latency",
                "Value": percentiles["p95_ms"],
                "Unit": "Milliseconds",
                "Dimensions": [
                    {"Name": "Environment", "Value": ENV},
                    {"Name": "Service", "Value": "inference"},
                ],
            },
            {
                "MetricName": "P99Latency",
                "Value": percentiles["p99_ms"],
                "Unit": "Milliseconds",
                "Dimensions": [
                    {"Name": "Environment", "Value": ENV},
                    {"Name": "Service", "Value": "inference"},
                ],
            },
            {
                "MetricName": "ErrorRate",
                "Value": percentiles["error_rate"],
                "Unit": "None",
                "Dimensions": [
                    {"Name": "Environment", "Value": ENV},
                    {"Name": "Service", "Value": "inference"},
                ],
            },
        ],
    )
    print(f"Published latency metrics: p50={percentiles['p50_ms']}ms, p95={percentiles['p95_ms']}ms, p99={percentiles['p99_ms']}ms")
```

**X-Ray Insights EventBridge notification:**
```python
import boto3
import json

events = boto3.client("events", region_name=AWS_REGION)
sns = boto3.client("sns", region_name=AWS_REGION)
cloudwatch = boto3.client("cloudwatch", region_name=AWS_REGION)


def setup_insights_notification(trace_group_name: str, sns_topic_arn: str):
    """Create EventBridge rule for X-Ray Insights notifications."""
    rule_name = f"{PROJECT_NAME}-xray-insights-{ENV}"

    events.put_rule(
        Name=rule_name,
        EventPattern=json.dumps({
            "source": ["aws.xray"],
            "detail-type": ["X-Ray Insight"],
            "detail": {
                "State": ["ACTIVE"],
                "GroupName": [trace_group_name],
            },
        }),
        State="ENABLED",
        Description=f"X-Ray Insights notification for {trace_group_name}",
        Tags=[
            {"Key": "Project", "Value": PROJECT_NAME},
            {"Key": "Environment", "Value": ENV},
        ],
    )

    events.put_targets(
        Rule=rule_name,
        Targets=[
            {
                "Id": "sns-notification",
                "Arn": sns_topic_arn,
            }
        ],
    )
    print(f"Insights notification rule created: {rule_name} → {sns_topic_arn}")


def create_latency_alarm(sns_topic_arn: str, threshold_ms: float):
    """Create CloudWatch alarm for p95 latency threshold."""
    alarm_name = f"{PROJECT_NAME}-ml-p95-latency-{ENV}"

    cloudwatch.put_metric_alarm(
        AlarmName=alarm_name,
        AlarmDescription=f"P95 inference latency exceeds {threshold_ms}ms",
        Namespace=f"{PROJECT_NAME}/MLInference",
        MetricName="P95Latency",
        Dimensions=[
            {"Name": "Environment", "Value": ENV},
            {"Name": "Service", "Value": "inference"},
        ],
        Statistic="Maximum",
        Period=300,
        EvaluationPeriods=2,
        Threshold=threshold_ms,
        ComparisonOperator="GreaterThanThreshold",
        AlarmActions=[sns_topic_arn],
        OKActions=[sns_topic_arn],
        Tags=[
            {"Key": "Project", "Value": PROJECT_NAME},
            {"Key": "Environment", "Value": ENV},
        ],
    )
    print(f"Latency alarm created: {alarm_name} (threshold={threshold_ms}ms)")
```

---

## Integration Points

- **Upstream**: `devops/04` → IAM roles for Lambda and ECS tasks that need `xray:PutTraceSegments` and `xray:PutTelemetryRecords` permissions; the inference function execution role must include these permissions for trace export
- **Upstream**: `mlops/03` → SageMaker inference endpoints that are instrumented with `traced_invoke_endpoint()`; this template wraps SageMaker runtime calls with OpenTelemetry spans
- **Upstream**: `mlops/14` → Bedrock Agents whose Lambda action group handlers are instrumented with ADOT; agent invocation traces flow through the spans defined in this template
- **Upstream**: `devops/02` → VPC networking configuration for ECS sidecar mode; the ADOT Collector sidecar needs outbound access to X-Ray endpoints (or via VPC endpoint from `devops/09`)
- **Downstream**: `devops/13` → Cost-per-inference dashboards consume the latency metrics and trace data published by this template to correlate cost with performance
- **Downstream**: `devops/11` → Custom CloudWatch model quality metrics consume the `tokens.input`, `tokens.output`, and `latency_ms` span attributes published to CloudWatch by the trace analysis utility
- **Downstream**: `devops/03` → CloudWatch dashboards reference the custom metrics namespace `{PROJECT_NAME}/MLInference` for latency percentile widgets
- **Downstream**: `mlops/18` → Prompt caching patterns use the `cache.hit` span attribute defined here to measure cache effectiveness in traces
