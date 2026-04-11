<!-- Template Version: 1.0 | boto3: 1.35+ -->

# Template 19 — Bedrock Marketplace Models

## Purpose
Generate production-ready infrastructure for deploying third-party models from Amazon Bedrock Marketplace: marketplace model discovery and subscription, SageMaker-backed endpoint deployment via `CreateMarketplaceModelEndpoint`, provisioned throughput for foundation models with configurable model units and commitment terms, cross-region inference profile configuration for availability and throughput, model comparison benchmarking utility for latency/throughput/cost evaluation, CloudWatch alarms for utilization monitoring, and cleanup scripts to prevent ongoing charges.

---

## Role Definition

You are an expert AWS AI engineer specializing in Amazon Bedrock Marketplace and model deployment with expertise in:
- Bedrock Marketplace: model discovery via `list_foundation_models()`, subscription workflows, endpoint deployment via `create_marketplace_model_endpoint()`, endpoint lifecycle management
- Provisioned Throughput: `create_provisioned_model_throughput()` for dedicated capacity, model unit allocation, commitment terms (no-commitment, 1-month, 6-month), throughput monitoring
- Cross-Region Inference Profiles: `create_inference_profile()` for application-level routing, system-defined inference profiles for multi-region distribution, cost and metric tracking per profile
- Model Benchmarking: latency measurement, throughput testing, cost-per-token analysis, side-by-side model comparison across marketplace and foundation models
- CloudWatch monitoring: utilization alarms for provisioned throughput, invocation metrics, error rate tracking
- Cost management: provisioned throughput billing awareness, marketplace endpoint instance costs, cleanup automation for non-production environments
- IAM policies for Bedrock model access, SageMaker endpoint roles, and cross-region inference permissions

Generate complete, production-deployable code.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED - must be a Bedrock-supported region]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

PROVIDER_FILTER:        [OPTIONAL - filter marketplace models by provider name]
                        Examples: "Meta", "Mistral", "Cohere", "AI21 Labs",
                        "Stability AI", "Anthropic"
                        Leave empty to list all available providers.

MODEL_FILTER:           [OPTIONAL - filter models by name or capability]
                        Examples: "llama", "mistral", "command", "jamba"
                        Used with list_foundation_models() byOutputModality
                        and byInferenceType filters.

DEPLOYMENT_MODE:        [REQUIRED - marketplace | provisioned | both]
                        marketplace: Deploy third-party models from Bedrock
                                     Marketplace via SageMaker-backed endpoints
                        provisioned: Create provisioned throughput for
                                     foundation models with dedicated capacity
                        both: Generate code for both deployment modes

MARKETPLACE_MODEL_ARN:  [OPTIONAL - SageMaker Hub content ARN for marketplace model]
                        Required when DEPLOYMENT_MODE is marketplace or both.
                        Format: arn:aws:sagemaker:{region}:aws:hub-content/
                                SageMakerPublicHub/Model/{model-name}/{version}

MARKETPLACE_INSTANCE_TYPE: [OPTIONAL: ml.g5.2xlarge]
                        SageMaker instance type for marketplace endpoint.
                        Options: ml.g5.xlarge, ml.g5.2xlarge, ml.g5.4xlarge,
                        ml.g5.12xlarge, ml.g5.48xlarge, ml.p4d.24xlarge,
                        ml.inf2.xlarge, ml.inf2.8xlarge

MARKETPLACE_INSTANCE_COUNT: [OPTIONAL: 1]
                        Number of instances for marketplace endpoint.

MODEL_ID:               [OPTIONAL - Bedrock foundation model ID for provisioned throughput]
                        Required when DEPLOYMENT_MODE is provisioned or both.
                        Examples:
                        - anthropic.claude-3-5-sonnet-20241022-v2:0
                        - amazon.nova-pro-v1:0
                        - meta.llama3-1-70b-instruct-v1:0
                        - Custom model ARN from fine-tuning job

MODEL_UNITS:            [OPTIONAL: 1 - number of provisioned model units]
                        Each model unit provides a fixed amount of throughput.
                        Request MU quota increase via AWS Support before purchase.

COMMITMENT_TERM:        [OPTIONAL: no-commitment]
                        Options:
                        - no-commitment (pay-as-you-go, can delete anytime)
                        - one-month (1-month commitment, lower rate)
                        - six-months (6-month commitment, lowest rate)

INFERENCE_PROFILE_REGIONS: [OPTIONAL - JSON list of regions for cross-region inference]
                        Example: ["us-east-1", "us-west-2", "eu-west-1"]
                        Creates an application inference profile that routes
                        requests across specified regions for higher throughput.
                        Uses system-defined inference profile ARNs.

BENCHMARK_PROMPTS_S3:   [OPTIONAL - s3://bucket/path/to/benchmark_prompts.jsonl]
                        JSONL file with benchmark prompts for model comparison.
                        Format: {"prompt": "...", "max_tokens": 256}
                        If not provided, uses built-in benchmark prompts.

BENCHMARK_MODELS:       [OPTIONAL - JSON list of model IDs to compare]
                        Example: ["anthropic.claude-3-5-sonnet-20241022-v2:0",
                                  "amazon.nova-pro-v1:0",
                                  "meta.llama3-1-70b-instruct-v1:0"]

GUARDRAIL_ID:           [OPTIONAL - Bedrock Guardrail ID from mlops/12]
GUARDRAIL_VERSION:      [OPTIONAL: DRAFT]
```

---

## Task

Generate complete Bedrock Marketplace and provisioned throughput deployment:

```
{PROJECT_NAME}-bedrock-marketplace/
├── config.py                              # Central configuration
├── discovery/
│   ├── list_models.py                     # List and filter marketplace models
│   └── model_details.py                   # Get model details and pricing info
├── marketplace/
│   ├── deploy_marketplace_endpoint.py     # Deploy marketplace model to SageMaker endpoint
│   ├── manage_endpoint.py                 # Update, deregister, reregister endpoint
│   └── invoke_marketplace_model.py        # Invoke marketplace model via Bedrock APIs
├── provisioned/
│   ├── create_provisioned_throughput.py   # Create provisioned throughput for foundation model
│   ├── manage_throughput.py               # Update, monitor, delete provisioned throughput
│   └── invoke_provisioned_model.py        # Invoke model via provisioned throughput ARN
├── inference_profile/
│   ├── create_inference_profile.py        # Create cross-region application inference profile
│   ├── list_profiles.py                   # List system-defined and application profiles
│   └── invoke_via_profile.py             # Invoke model through inference profile
├── benchmark/
│   ├── run_benchmark.py                   # Run latency/throughput/cost benchmark
│   ├── compare_models.py                  # Side-by-side model comparison report
│   └── benchmark_prompts.py               # Built-in benchmark prompt sets
├── monitoring/
│   ├── cloudwatch_alarms.py               # CloudWatch alarms for utilization
│   └── usage_dashboard.py                 # CloudWatch dashboard for model usage
├── cleanup/
│   └── delete_resources.py                # Delete provisioned throughput, endpoints, profiles
├── run_setup.py                           # CLI orchestrator
└── requirements.txt
```


**config.py**: Central configuration dataclass with all parameters. Load from environment variables or CLI args. Validate required fields based on DEPLOYMENT_MODE. Construct resource names using `{PROJECT_NAME}-{component}-{ENV}` convention. Validate MODEL_ID exists and supports provisioned throughput when DEPLOYMENT_MODE is provisioned. Parse INFERENCE_PROFILE_REGIONS as list.

**list_models.py**: Discover available Bedrock models using `bedrock.list_foundation_models()`:
- Filter by `byProvider` (PROVIDER_FILTER) to narrow to specific vendors
- Filter by `byOutputModality` (TEXT, IMAGE, EMBEDDING) for capability filtering
- Filter by `byInferenceType` (ON_DEMAND, PROVISIONED) to find provisionable models
- Display model ID, provider, name, input/output modalities, customization support, and inference types
- Highlight models available in Bedrock Marketplace vs standard foundation models

**model_details.py**: Get detailed model information using `bedrock.get_foundation_model()`:
- Retrieve model lifecycle status, supported customizations, and supported inference types
- Check if model supports provisioned throughput (`PROVISIONED` in `inferenceTypesSupported`)
- Display pricing tier information and regional availability
- Output structured JSON for downstream comparison

**deploy_marketplace_endpoint.py**: Deploy a Bedrock Marketplace model using `bedrock.create_marketplace_model_endpoint()`:
- Specify `modelSourceIdentifier` (SageMaker Hub content ARN), `endpointName`, and `endpointConfig` with SageMaker instance type, count, and execution role
- Set `acceptEula=True` for models requiring license agreement acceptance
- Poll endpoint status using `bedrock.get_marketplace_model_endpoint()` until `endpointStatus` is `InService` and `status` is `REGISTERED`
- Tag with Project and Environment

**manage_endpoint.py**: Manage marketplace endpoint lifecycle:
- `update_marketplace_model_endpoint()`: Update instance type or count
- `deregister_marketplace_model_endpoint()`: Deregister endpoint from Bedrock (keeps SageMaker endpoint)
- `register_marketplace_model_endpoint()`: Re-register a SageMaker endpoint with Bedrock
- `list_marketplace_model_endpoints()`: List all marketplace endpoints with status

**invoke_marketplace_model.py**: Invoke marketplace model through Bedrock APIs:
- Use `bedrock_runtime.invoke_model()` with the SageMaker endpoint ARN as `modelId`
- Alternatively use `bedrock_runtime.converse()` for models supporting the Converse API
- Apply guardrail if GUARDRAIL_ID is configured
- Parse response based on model-specific output format
- Log invocation latency and token usage

**create_provisioned_throughput.py**: Create provisioned throughput using `bedrock.create_provisioned_model_throughput()`:
- Set `modelId` (foundation model or custom model ARN), `provisionedModelName`, `modelUnits`, and `commitmentDuration`
- Poll status using `bedrock.get_provisioned_model_throughput()` until `status` is `InService`
- Return `provisionedModelArn` for use as `modelId` in inference calls
- Tag with Project and Environment

**manage_throughput.py**: Manage provisioned throughput lifecycle:
- `update_provisioned_model_throughput()`: Change model units or swap to a different custom model (same base model family)
- `get_provisioned_model_throughput()`: Check status, utilization, and commitment details
- `list_provisioned_model_throughputs()`: List all provisioned throughputs with status
- `delete_provisioned_model_throughput()`: Delete provisioned throughput (only for no-commitment or after commitment expires)

**invoke_provisioned_model.py**: Invoke model via provisioned throughput ARN:
- Use `bedrock_runtime.converse()` with `provisionedModelArn` as `modelId`
- Track invocation count and latency for utilization monitoring
- Apply guardrail if configured
- Compare response quality with on-demand invocation

**create_inference_profile.py**: Create cross-region application inference profile using `bedrock.create_inference_profile()`:
- For single-region tracking: set `modelSource.copyFrom` to the foundation model ARN in the target region
- For cross-region routing: set `modelSource.copyFrom` to the system-defined inference profile ARN (e.g., `us.anthropic.claude-3-5-sonnet-20241022-v2:0`)
- Return `inferenceProfileArn` for use as `modelId` in inference calls
- Tag with Project and Environment

**list_profiles.py**: List inference profiles using `bedrock.list_inference_profiles()`:
- Filter by `typeEquals` (`SYSTEM_DEFINED` or `APPLICATION`) to separate system profiles from custom ones
- Display profile name, ARN, status, model source, and associated regions
- Show which system-defined profiles are available for cross-region routing

**invoke_via_profile.py**: Invoke model through inference profile:
- Use `bedrock_runtime.converse()` with `inferenceProfileArn` as `modelId`
- Bedrock automatically routes to the optimal region based on availability
- Log which region served the request (from response metadata)
- Track per-profile invocation metrics

**run_benchmark.py**: Run latency/throughput/cost benchmark:
- Load benchmark prompts from S3 (BENCHMARK_PROMPTS_S3) or use built-in prompts
- For each model in BENCHMARK_MODELS, run N invocations measuring: first-token latency, total latency, tokens per second, input/output token counts
- Compute p50, p95, p99 latency percentiles
- Estimate cost per 1K tokens based on model pricing
- Output results as JSON and optional CSV

**compare_models.py**: Side-by-side model comparison report:
- Aggregate benchmark results across all tested models
- Rank models by latency, throughput, cost, and quality (if ground truth provided)
- Generate comparison table with columns: model_id, p50_latency_ms, p95_latency_ms, tokens_per_sec, cost_per_1k_tokens, quality_score
- Recommend optimal model based on configurable priority (latency vs cost vs quality)

**benchmark_prompts.py**: Built-in benchmark prompt sets:
- Short prompts (< 100 tokens): simple Q&A, classification
- Medium prompts (100-500 tokens): summarization, extraction
- Long prompts (500-2000 tokens): analysis, code generation
- Each prompt includes expected output length for consistent comparison

**cloudwatch_alarms.py**: CloudWatch alarms for model utilization:
- Provisioned throughput: alarm on `ProvisionedModelUtilization` exceeding 80% (scale-up recommendation) and below 20% (cost waste alert)
- Marketplace endpoint: alarm on SageMaker `InvocationsPerInstance`, `ModelLatency`, `Invocation5XXErrors`
- Inference profile: alarm on `InvocationCount` and `InvocationLatency` per profile
- All alarms send SNS notifications

**usage_dashboard.py**: CloudWatch dashboard for model usage:
- Widgets for: invocation count by model, latency percentiles, error rates, provisioned throughput utilization gauge, cost-per-invocation trend
- Separate sections for marketplace endpoints, provisioned throughput, and inference profiles
- Auto-refresh at 1-minute intervals

**delete_resources.py**: Clean up all resources to prevent ongoing charges:
- Delete provisioned throughput (check commitment status first — warn if under commitment)
- Delete marketplace endpoints via `delete_marketplace_model_endpoint()`
- Delete application inference profiles via `delete_inference_profile()`
- Delete CloudWatch alarms and dashboards
- Confirmation prompt before deletion in prod environment

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**Marketplace Models:** Bedrock Marketplace models are deployed to SageMaker-managed endpoints. You must subscribe to a model before deploying. Endpoints incur SageMaker instance costs (per-hour billing). Use `CreateMarketplaceModelEndpoint` for deployment — do not create SageMaker endpoints directly. Marketplace models can be accessed via Bedrock `InvokeModel` and `Converse` APIs using the endpoint ARN as `modelId`. Marketplace models are compatible with Bedrock Agents, Knowledge Bases, and Guardrails.

**Provisioned Throughput:** Request model unit (MU) quota increase via AWS Support before purchasing. No-commitment provisioned throughput can be deleted anytime. Committed throughput (1-month or 6-month) cannot be deleted before the commitment expires. Provisioned throughput is billed per hour per model unit regardless of utilization. Monitor utilization — under-utilized provisioned throughput is wasted cost. The `provisionedModelArn` returned by `CreateProvisionedModelThroughput` is used as `modelId` for inference.

**Cross-Region Inference Profiles:** Application inference profiles track metrics and costs per profile. System-defined inference profiles route requests across pre-configured regions (e.g., `us.anthropic.claude-3-5-sonnet-20241022-v2:0` routes across US regions). Create application inference profiles from system-defined profiles for cross-region routing with per-application cost tracking. Inference profiles do not add latency — routing is handled at the Bedrock service level. IAM policies must allow `bedrock:InvokeModel` on the inference profile ARN.

**Security:** Marketplace endpoint execution roles need SageMaker permissions. Provisioned throughput requires `bedrock:CreateProvisionedModelThroughput` and `bedrock:InvokeModel` on the provisioned model ARN. Inference profiles require `bedrock:CreateInferenceProfile` and `bedrock:InvokeModel` on the profile ARN. Use VPC endpoints for Bedrock API calls in private subnets. Enable CloudTrail for all Bedrock API calls.

**Cost:** Marketplace endpoints: SageMaker instance costs (per-hour). Provisioned throughput: per-hour per model unit (varies by model and commitment term). On-demand inference: per-token pricing. Cross-region inference profiles: standard per-token pricing for the model, no additional routing cost. Always delete non-production provisioned throughput and marketplace endpoints when not in use.

**Naming:** All resources follow `{PROJECT_NAME}-{component}-{ENV}` convention:
- Marketplace endpoint: `{PROJECT_NAME}-mkt-{model_short_name}-{ENV}`
- Provisioned throughput: `{PROJECT_NAME}-pt-{ENV}`
- Inference profile: `{PROJECT_NAME}-profile-{ENV}`
- CloudWatch alarms: `{PROJECT_NAME}-{component}-alarm-{ENV}`
- Dashboard: `{PROJECT_NAME}-marketplace-dashboard-{ENV}`

---

## Code Scaffolding Hints


**List and filter marketplace models:**
```python
import boto3
import json

bedrock = boto3.client("bedrock", region_name=AWS_REGION)

def list_marketplace_models(provider_filter: str = None, inference_type: str = None) -> list:
    """List available Bedrock foundation models with optional filters."""
    params = {}
    if provider_filter:
        params["byProvider"] = provider_filter
    if inference_type:
        params["byInferenceType"] = inference_type  # ON_DEMAND | PROVISIONED

    response = bedrock.list_foundation_models(**params)
    models = response.get("modelSummaries", [])

    results = []
    for model in models:
        results.append({
            "modelId": model["modelId"],
            "modelName": model["modelName"],
            "providerName": model["providerName"],
            "inputModalities": model["inputModalities"],
            "outputModalities": model["outputModalities"],
            "inferenceTypesSupported": model["inferenceTypesSupported"],
            "customizationsSupported": model.get("customizationsSupported", []),
            "modelLifecycle": model.get("modelLifecycle", {}).get("status", "ACTIVE"),
        })

    return results


def get_model_details(model_id: str) -> dict:
    """Get detailed information about a specific model."""
    response = bedrock.get_foundation_model(modelIdentifier=model_id)
    model = response["modelDetails"]
    return {
        "modelId": model["modelId"],
        "modelName": model["modelName"],
        "providerName": model["providerName"],
        "inferenceTypesSupported": model["inferenceTypesSupported"],
        "supportsProvisioned": "PROVISIONED" in model["inferenceTypesSupported"],
        "responseStreamingSupported": model.get("responseStreamingSupported", False),
        "customizationsSupported": model.get("customizationsSupported", []),
        "modelLifecycle": model.get("modelLifecycle", {}).get("status", "ACTIVE"),
    }
```

**Deploy marketplace model endpoint:**
```python
import time

def deploy_marketplace_endpoint(
    model_source_arn: str,
    endpoint_name: str,
    instance_type: str = "ml.g5.2xlarge",
    instance_count: int = 1,
    execution_role_arn: str = None,
    accept_eula: bool = True,
) -> dict:
    """Deploy a Bedrock Marketplace model to a SageMaker-backed endpoint."""
    endpoint_config = {
        "sageMaker": {
            "initialInstanceCount": instance_count,
            "instanceType": instance_type,
        }
    }
    if execution_role_arn:
        endpoint_config["sageMaker"]["executionRole"] = execution_role_arn

    create_params = {
        "modelSourceIdentifier": model_source_arn,
        "endpointName": endpoint_name,
        "endpointConfig": endpoint_config,
        "tags": [
            {"key": "Project", "value": PROJECT_NAME},
            {"key": "Environment", "value": ENV},
        ],
    }
    if accept_eula:
        create_params["acceptEula"] = True

    response = bedrock.create_marketplace_model_endpoint(**create_params)
    endpoint_arn = response["marketplaceModelEndpoint"]["endpointArn"]

    # Poll until endpoint is InService and REGISTERED
    while True:
        status_response = bedrock.get_marketplace_model_endpoint(
            endpointArn=endpoint_arn
        )
        endpoint = status_response["marketplaceModelEndpoint"]
        ep_status = endpoint["endpointStatus"]
        reg_status = endpoint["status"]

        if ep_status == "InService" and reg_status == "REGISTERED":
            break
        elif ep_status == "Failed":
            raise RuntimeError(
                f"Marketplace endpoint failed: {endpoint.get('endpointStatusMessage', 'Unknown error')}"
            )
        time.sleep(30)

    return {
        "endpoint_arn": endpoint_arn,
        "endpoint_status": ep_status,
        "registration_status": reg_status,
    }
```

**Invoke marketplace model via Bedrock Converse API:**
```python
bedrock_runtime = boto3.client("bedrock-runtime", region_name=AWS_REGION)

def invoke_marketplace_model(endpoint_arn: str, prompt: str, max_tokens: int = 1024) -> dict:
    """Invoke a Bedrock Marketplace model using the Converse API."""
    converse_params = {
        "modelId": endpoint_arn,  # Use endpoint ARN as modelId
        "messages": [
            {
                "role": "user",
                "content": [{"text": prompt}],
            }
        ],
        "inferenceConfig": {
            "maxTokens": max_tokens,
            "temperature": 0.7,
            "topP": 0.9,
        },
    }

    # Attach guardrail if configured
    if GUARDRAIL_ID:
        converse_params["guardrailConfig"] = {
            "guardrailIdentifier": GUARDRAIL_ID,
            "guardrailVersion": GUARDRAIL_VERSION,
        }

    response = bedrock_runtime.converse(**converse_params)

    usage = response.get("usage", {})
    output_message = response["output"]["message"]
    response_text = "".join(
        block.get("text", "") for block in output_message["content"]
    )

    return {
        "response": response_text,
        "input_tokens": usage.get("inputTokens", 0),
        "output_tokens": usage.get("outputTokens", 0),
        "stop_reason": response.get("stopReason", ""),
    }
```

**Create provisioned throughput:**
```python
def create_provisioned_throughput(
    model_id: str,
    model_units: int = 1,
    commitment_duration: str = "NoCommitment",
) -> dict:
    """Create provisioned throughput for a Bedrock foundation or custom model."""
    # commitment_duration: NoCommitment | OneMonth | SixMonths
    provisioned_name = f"{PROJECT_NAME}-pt-{ENV}"

    create_params = {
        "provisionedModelName": provisioned_name,
        "modelId": model_id,
        "modelUnits": model_units,
        "tags": [
            {"key": "Project", "value": PROJECT_NAME},
            {"key": "Environment", "value": ENV},
        ],
    }

    # Only include commitmentDuration for committed purchases
    if commitment_duration != "NoCommitment":
        create_params["commitmentDuration"] = commitment_duration

    response = bedrock.create_provisioned_model_throughput(**create_params)
    provisioned_arn = response["provisionedModelArn"]

    # Poll until InService
    while True:
        status_response = bedrock.get_provisioned_model_throughput(
            provisionedModelId=provisioned_arn
        )
        status = status_response["status"]
        if status == "InService":
            break
        elif status == "Failed":
            raise RuntimeError(
                f"Provisioned throughput failed: {status_response.get('failureMessage', 'Unknown')}"
            )
        time.sleep(30)

    return {
        "provisioned_model_arn": provisioned_arn,
        "status": status,
        "model_units": model_units,
        "commitment": commitment_duration,
    }
```

**Invoke model via provisioned throughput:**
```python
def invoke_provisioned_model(provisioned_model_arn: str, prompt: str, max_tokens: int = 1024) -> dict:
    """Invoke model using provisioned throughput ARN."""
    response = bedrock_runtime.converse(
        modelId=provisioned_model_arn,  # Use provisioned throughput ARN
        messages=[
            {
                "role": "user",
                "content": [{"text": prompt}],
            }
        ],
        inferenceConfig={
            "maxTokens": max_tokens,
            "temperature": 0.7,
            "topP": 0.9,
        },
    )

    usage = response.get("usage", {})
    output_message = response["output"]["message"]
    response_text = "".join(
        block.get("text", "") for block in output_message["content"]
    )

    return {
        "response": response_text,
        "input_tokens": usage.get("inputTokens", 0),
        "output_tokens": usage.get("outputTokens", 0),
        "stop_reason": response.get("stopReason", ""),
        "model_arn": provisioned_model_arn,
    }
```


**Create cross-region application inference profile:**
```python
def create_inference_profile(
    profile_name: str,
    model_source_arn: str,
    description: str = "",
) -> dict:
    """Create an application inference profile for cost tracking or cross-region routing.

    For single-region tracking:
        model_source_arn = "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-5-sonnet-20241022-v2:0"

    For cross-region routing (use system-defined inference profile ARN):
        model_source_arn = "arn:aws:bedrock:us-east-1:{account_id}:inference-profile/us.anthropic.claude-3-5-sonnet-20241022-v2:0"
    """
    create_params = {
        "inferenceProfileName": profile_name,
        "modelSource": {
            "copyFrom": model_source_arn,
        },
        "tags": [
            {"key": "Project", "value": PROJECT_NAME},
            {"key": "Environment", "value": ENV},
        ],
    }
    if description:
        create_params["description"] = description

    response = bedrock.create_inference_profile(**create_params)

    return {
        "inference_profile_arn": response["inferenceProfileArn"],
        "status": response["status"],
    }


def list_inference_profiles(type_filter: str = None) -> list:
    """List inference profiles. type_filter: SYSTEM_DEFINED | APPLICATION"""
    params = {}
    if type_filter:
        params["typeEquals"] = type_filter

    response = bedrock.list_inference_profiles(**params)
    profiles = response.get("inferenceProfileSummaries", [])

    results = []
    for profile in profiles:
        results.append({
            "profileName": profile["inferenceProfileName"],
            "profileArn": profile["inferenceProfileArn"],
            "status": profile["status"],
            "type": profile["type"],
            "models": [m["modelArn"] for m in profile.get("models", [])],
        })

    return results
```

**Invoke model through inference profile:**
```python
def invoke_via_inference_profile(inference_profile_arn: str, prompt: str, max_tokens: int = 1024) -> dict:
    """Invoke model through an inference profile for cross-region routing."""
    response = bedrock_runtime.converse(
        modelId=inference_profile_arn,  # Use inference profile ARN as modelId
        messages=[
            {
                "role": "user",
                "content": [{"text": prompt}],
            }
        ],
        inferenceConfig={
            "maxTokens": max_tokens,
            "temperature": 0.7,
            "topP": 0.9,
        },
    )

    usage = response.get("usage", {})
    output_message = response["output"]["message"]
    response_text = "".join(
        block.get("text", "") for block in output_message["content"]
    )

    return {
        "response": response_text,
        "input_tokens": usage.get("inputTokens", 0),
        "output_tokens": usage.get("outputTokens", 0),
        "stop_reason": response.get("stopReason", ""),
        "profile_arn": inference_profile_arn,
    }
```

**Model comparison benchmark:**
```python
import time
import json
import statistics

def run_benchmark(model_ids: list, prompts: list, num_runs: int = 5) -> dict:
    """Benchmark multiple models with identical prompts for comparison."""
    results = {}

    for model_id in model_ids:
        model_results = {"latencies_ms": [], "tokens_per_sec": [], "total_input_tokens": 0, "total_output_tokens": 0, "errors": 0}

        for prompt_data in prompts:
            prompt_text = prompt_data["prompt"]
            max_tokens = prompt_data.get("max_tokens", 256)

            for _ in range(num_runs):
                try:
                    start = time.time()
                    response = bedrock_runtime.converse(
                        modelId=model_id,
                        messages=[{"role": "user", "content": [{"text": prompt_text}]}],
                        inferenceConfig={"maxTokens": max_tokens, "temperature": 0.7},
                    )
                    elapsed_ms = (time.time() - start) * 1000

                    usage = response.get("usage", {})
                    output_tokens = usage.get("outputTokens", 0)
                    input_tokens = usage.get("inputTokens", 0)

                    model_results["latencies_ms"].append(elapsed_ms)
                    model_results["total_input_tokens"] += input_tokens
                    model_results["total_output_tokens"] += output_tokens

                    if output_tokens > 0 and elapsed_ms > 0:
                        tps = (output_tokens / elapsed_ms) * 1000
                        model_results["tokens_per_sec"].append(tps)

                except Exception as e:
                    model_results["errors"] += 1

        # Compute statistics
        latencies = model_results["latencies_ms"]
        tps_values = model_results["tokens_per_sec"]

        results[model_id] = {
            "total_invocations": len(latencies),
            "errors": model_results["errors"],
            "latency_p50_ms": round(statistics.median(latencies), 1) if latencies else 0,
            "latency_p95_ms": round(sorted(latencies)[int(len(latencies) * 0.95)] if latencies else 0, 1),
            "latency_p99_ms": round(sorted(latencies)[int(len(latencies) * 0.99)] if latencies else 0, 1),
            "avg_tokens_per_sec": round(statistics.mean(tps_values), 1) if tps_values else 0,
            "total_input_tokens": model_results["total_input_tokens"],
            "total_output_tokens": model_results["total_output_tokens"],
        }

    return results


def generate_comparison_report(benchmark_results: dict) -> str:
    """Generate a formatted comparison report from benchmark results."""
    header = f"{'Model ID':<55} {'p50 (ms)':<10} {'p95 (ms)':<10} {'TPS':<8} {'Errors':<7}"
    separator = "-" * len(header)
    lines = [header, separator]

    for model_id, stats in sorted(benchmark_results.items(), key=lambda x: x[1]["latency_p50_ms"]):
        lines.append(
            f"{model_id:<55} {stats['latency_p50_ms']:<10} {stats['latency_p95_ms']:<10} "
            f"{stats['avg_tokens_per_sec']:<8} {stats['errors']:<7}"
        )

    return "\n".join(lines)
```

**CloudWatch alarms for provisioned throughput utilization:**
```python
cloudwatch = boto3.client("cloudwatch", region_name=AWS_REGION)

def create_utilization_alarms(provisioned_model_name: str, sns_topic_arn: str):
    """Create CloudWatch alarms for provisioned throughput utilization."""

    # High utilization alarm — recommend scaling up
    cloudwatch.put_metric_alarm(
        AlarmName=f"{PROJECT_NAME}-pt-high-utilization-{ENV}",
        AlarmDescription="Provisioned throughput utilization exceeds 80% — consider adding model units",
        Namespace="AWS/Bedrock",
        MetricName="ProvisionedModelUtilization",
        Dimensions=[
            {"Name": "ProvisionedModelName", "Value": provisioned_model_name},
        ],
        Statistic="Average",
        Period=300,
        EvaluationPeriods=3,
        Threshold=80.0,
        ComparisonOperator="GreaterThanThreshold",
        AlarmActions=[sns_topic_arn],
        Tags=[
            {"Key": "Project", "Value": PROJECT_NAME},
            {"Key": "Environment", "Value": ENV},
        ],
    )

    # Low utilization alarm — cost waste alert
    cloudwatch.put_metric_alarm(
        AlarmName=f"{PROJECT_NAME}-pt-low-utilization-{ENV}",
        AlarmDescription="Provisioned throughput utilization below 20% — consider reducing model units or deleting",
        Namespace="AWS/Bedrock",
        MetricName="ProvisionedModelUtilization",
        Dimensions=[
            {"Name": "ProvisionedModelName", "Value": provisioned_model_name},
        ],
        Statistic="Average",
        Period=300,
        EvaluationPeriods=6,
        Threshold=20.0,
        ComparisonOperator="LessThanThreshold",
        AlarmActions=[sns_topic_arn],
        Tags=[
            {"Key": "Project", "Value": PROJECT_NAME},
            {"Key": "Environment", "Value": ENV},
        ],
    )


def create_marketplace_endpoint_alarms(endpoint_name: str, sns_topic_arn: str):
    """Create CloudWatch alarms for marketplace SageMaker endpoint."""

    # Error rate alarm
    cloudwatch.put_metric_alarm(
        AlarmName=f"{PROJECT_NAME}-mkt-errors-{ENV}",
        AlarmDescription="Marketplace endpoint 5XX error rate exceeds threshold",
        Namespace="AWS/SageMaker",
        MetricName="Invocation5XXErrors",
        Dimensions=[
            {"Name": "EndpointName", "Value": endpoint_name},
            {"Name": "VariantName", "Value": "AllTraffic"},
        ],
        Statistic="Sum",
        Period=300,
        EvaluationPeriods=2,
        Threshold=5.0,
        ComparisonOperator="GreaterThanThreshold",
        AlarmActions=[sns_topic_arn],
        Tags=[
            {"Key": "Project", "Value": PROJECT_NAME},
            {"Key": "Environment", "Value": ENV},
        ],
    )

    # High latency alarm
    cloudwatch.put_metric_alarm(
        AlarmName=f"{PROJECT_NAME}-mkt-latency-{ENV}",
        AlarmDescription="Marketplace endpoint model latency exceeds threshold",
        Namespace="AWS/SageMaker",
        MetricName="ModelLatency",
        Dimensions=[
            {"Name": "EndpointName", "Value": endpoint_name},
            {"Name": "VariantName", "Value": "AllTraffic"},
        ],
        Statistic="p95",
        Period=300,
        EvaluationPeriods=3,
        Threshold=10000000.0,  # 10 seconds in microseconds
        ComparisonOperator="GreaterThanThreshold",
        AlarmActions=[sns_topic_arn],
        Tags=[
            {"Key": "Project", "Value": PROJECT_NAME},
            {"Key": "Environment", "Value": ENV},
        ],
    )
```

**Cleanup — delete all resources:**
```python
import logging

logger = logging.getLogger(__name__)

def delete_provisioned_throughput(provisioned_model_id: str):
    """Delete provisioned throughput. Warns if under commitment."""
    try:
        status = bedrock.get_provisioned_model_throughput(
            provisionedModelId=provisioned_model_id
        )
        commitment = status.get("commitmentDuration", "NoCommitment")
        if commitment != "NoCommitment":
            logger.warning(
                "Provisioned throughput %s has active commitment (%s). "
                "Deletion may not be allowed until commitment expires.",
                provisioned_model_id, commitment,
            )

        bedrock.delete_provisioned_model_throughput(
            provisionedModelId=provisioned_model_id
        )
        logger.info("Deleted provisioned throughput: %s", provisioned_model_id)
    except bedrock.exceptions.ResourceNotFoundException:
        logger.info("Provisioned throughput already deleted: %s", provisioned_model_id)


def delete_marketplace_endpoint(endpoint_arn: str):
    """Delete a Bedrock Marketplace endpoint."""
    try:
        bedrock.delete_marketplace_model_endpoint(endpointArn=endpoint_arn)
        logger.info("Deleted marketplace endpoint: %s", endpoint_arn)
    except bedrock.exceptions.ResourceNotFoundException:
        logger.info("Marketplace endpoint already deleted: %s", endpoint_arn)


def delete_inference_profile(inference_profile_id: str):
    """Delete an application inference profile."""
    try:
        bedrock.delete_inference_profile(
            inferenceProfileIdentifier=inference_profile_id
        )
        logger.info("Deleted inference profile: %s", inference_profile_id)
    except bedrock.exceptions.ResourceNotFoundException:
        logger.info("Inference profile already deleted: %s", inference_profile_id)


def cleanup_all(provisioned_ids: list = None, endpoint_arns: list = None, profile_ids: list = None):
    """Delete all marketplace resources. Use in non-production environments."""
    if ENV == "prod":
        confirm = input("WARNING: Deleting production resources. Type 'yes' to confirm: ")
        if confirm != "yes":
            logger.info("Cleanup cancelled.")
            return

    for pid in (provisioned_ids or []):
        delete_provisioned_throughput(pid)

    for arn in (endpoint_arns or []):
        delete_marketplace_endpoint(arn)

    for profile_id in (profile_ids or []):
        delete_inference_profile(profile_id)

    logger.info("Cleanup complete.")
```

---

## Integration Points

- **Upstream**: `devops/04` → IAM roles for Bedrock model access, SageMaker endpoint execution roles, and cross-region inference permissions
- **Upstream**: `mlops/12` → Bedrock Guardrail ID for applying guardrails to marketplace and provisioned model invocations
- **Downstream**: `mlops/17` → LLM evaluation pipeline uses benchmark results to compare marketplace models against foundation models
- **Downstream**: `devops/12` → Bedrock invocation logging captures all marketplace and provisioned throughput invocations for analysis
- **Downstream**: `finops/01` → Cost allocation tags on provisioned throughput and marketplace endpoints feed into FinOps cost tracking