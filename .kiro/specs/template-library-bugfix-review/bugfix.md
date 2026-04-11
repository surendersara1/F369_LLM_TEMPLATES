# Bugfix Requirements Document

## Introduction

The F369 AWS MLOps & AI/LLM CI/CD Prompt Template Library contains 27 markdown template files across 4 directories (cicd/, devops/, iac/, mlops/) with embedded code examples in Terraform, CDK Python, YAML, Dockerfiles, IAM policies, Python scripts, and shell commands. Systematic review has identified bugs spanning incorrect AWS API references, deprecated SDK versions, wrong API parameter names, incorrect model identifiers, security misconfigurations, Terraform/CDK syntax errors, and inconsistent scaffolding hints. These bugs would cause generated code to fail at deploy time or produce insecure infrastructure.

## Bug Analysis

### Current Behavior (Defect)

**Category A: Deprecated/Incorrect SDK Versions & API References**

1.1 WHEN a user generates code from `mlops/02_llm_finetuning_pipeline.md` THEN the system produces a SageMaker HuggingFace Estimator with `transformers_version="4.36"` and `pytorch_version="2.1"` and `py_version="py310"`, which are outdated and inconsistent with the template header claiming `transformers: 4.45+` and `peft: 0.12+` compatibility, causing version mismatch errors when the generated training script imports features from newer library versions

1.2 WHEN a user generates code from `mlops/01_sagemaker_training_pipeline.md` THEN the system references `sagemaker.log_metric()` in the train.py scaffolding hints, which is not a valid SageMaker Python SDK function — metrics are logged by printing to stdout with regex-matched patterns, not via a direct SDK call

1.3 WHEN a user generates code from `mlops/06_experiment_tracking.md` for managed-mlflow THEN the system references `sagemaker.create_mlflow_tracking_server()` as a SageMaker Python SDK method, but the actual API is `boto3 sagemaker client.create_mlflow_tracking_server()` — the scaffolding hint conflates the high-level SDK with the boto3 client

1.4 WHEN a user generates code from `mlops/06_experiment_tracking.md` for managed-mlflow THEN the system uses `describe_mlflow_tracking_server()` and reads `["TrackingServerUrl"]` from the response, but the actual boto3 response key is `TrackingServerUrl` nested under the response — the code does not show proper response parsing and the field name may differ from the actual API response schema

**Category B: Incorrect Bedrock Model Identifiers & API Parameters**

1.5 WHEN a user generates code from `iac/03_terraform_bedrock_opensearch_rag.md` THEN the system references `GENERATION_MODEL_ARN` as `arn:aws:bedrock:{region}::foundation-model/anthropic.claude-3-5-sonnet-20241022-v2:0` which uses an incorrect model ID format — the correct Bedrock model ID for Claude 3.5 Sonnet v2 is `anthropic.claude-3-5-sonnet-20241022-v2:0` but the ARN format with double-colon (`::`) before `foundation-model` is correct only for foundation models, and the version suffix format may not match the actual Bedrock model registry

1.6 WHEN a user generates code from `mlops/09_bedrock_finetuning.md` THEN the system provides an `invoke_model` scaffolding hint using `{"prompt": "...", "maxTokens": 512, "temperature": 0.7}` as the request body for a custom fine-tuned model, but the request body schema varies by model family (Titan uses `inputText`/`textGenerationConfig`, Llama uses a different schema) — the generic body will fail for most model families

1.7 WHEN a user generates code from `mlops/09_bedrock_finetuning.md` THEN the system lists `cohere.command-light-text-v14` as a fine-tunable Bedrock model, but Bedrock fine-tuning support for Cohere models uses `cohere.command-text-v14` or `cohere.command-light-text-v14` — the model ID may be incorrect as Bedrock fine-tuning availability changes and `command-light` may not support customization

1.8 WHEN a user generates code from `mlops/09_bedrock_finetuning.md` THEN the system specifies Llama/Meta training data format as `{"prompt": "<s>[INST] ... [/INST]", "completion": "..."}` using Llama 2 chat template syntax, but the `BASE_MODEL_ID` options list Llama 3.1 models which use a completely different chat template format (`<|begin_of_text|><|start_header_id|>...`), causing fine-tuning data format mismatch

1.9 WHEN a user generates code from `mlops/12_bedrock_guardrails_agents.md` THEN the system includes `"PROMPT_ATTACK"` in the content filter config with `"outputStrength": "NONE"`, but the Bedrock Guardrails API requires `PROMPT_ATTACK` filter to only be applied to inputs — setting `outputStrength` at all for `PROMPT_ATTACK` type may cause an API validation error

**Category C: Terraform/CDK Syntax & Resource Errors**

1.10 WHEN a user generates code from `iac/03_terraform_bedrock_opensearch_rag.md` THEN the system references `aws_bedrockagent_knowledge_base` and `aws_bedrockagent_data_source` Terraform resource types, but the correct AWS provider resource names are `aws_bedrockagent_knowledge_base` (correct) and `aws_bedrockagent_data_source` (correct) — however the template does not specify the minimum AWS provider version required for these resources (5.60+), and the template header says `AWS Provider: 5.70+` but does not enforce this in the providers.tf scaffolding

1.11 WHEN a user generates code from `iac/04_cdk_ecs_llm_inference.md` THEN the system references `ecs.ContainerImage.from_ecr_repository(ecr_repo, tag)` and sets `gpu_count=1` on `add_container()`, but the CDK `ContainerDefinition` does not have a `gpu_count` parameter — GPU resources must be specified via `container.add_resource_requirements()` or through `LinuxParameters` with device requests, causing a CDK synthesis error

1.12 WHEN a user generates code from `iac/02_cdk_ml_llm_infrastructure.md` THEN the system instructs to use `sagemaker.CfnDomain` for SageMaker Domain creation but references `Fn.import_value()` for cross-stack references, while the correct CDK Python import is `cdk.Fn.import_value()` or simply using stack output attributes directly — the inconsistent reference style may confuse generated code

**Category D: Incorrect YAML/Config Syntax in CI/CD Templates**

1.13 WHEN a user generates code from `cicd/05_bitbucket_pipelines_aws.md` THEN the system shows a `bitbucket-pipelines.yml` with `caches: [docker]` on the build-container step, but Bitbucket Pipelines does not support a built-in `docker` cache type — Docker layer caching requires the `docker` service and `docker-layer-caching: true` option, causing a pipeline validation error

1.14 WHEN a user generates code from `cicd/05_bitbucket_pipelines_aws.md` THEN the system defines `caches: pip: ~/.cache/pip` under `definitions`, but the correct Bitbucket Pipelines custom cache syntax requires the path to be nested under the cache name as a string value, and the tilde (`~`) path may not resolve correctly in the Bitbucket Pipelines container environment

1.15 WHEN a user generates code from `cicd/04_github_actions_aws_integration.md` THEN the system shows `on: push: { branches: [main] }` using flow-style YAML for the trigger, which while valid YAML may cause confusion and is inconsistent with GitHub Actions documentation conventions that use block-style — more critically, the template does not show the `workflow_dispatch` input properly nested under the `on` key in the full example

**Category E: Security Misconfigurations**

1.16 WHEN a user generates code from `devops/04_iam_roles_policies_mlops.md` THEN the system specifies the SageMaker execution role with `sagemaker:*` scoped by resource tag or name prefix, but the scaffolding description says "scoped by resource tag or name prefix" without providing the actual IAM condition keys — the generated policy may default to `sagemaker:*` on `Resource: *` which is overly permissive

1.17 WHEN a user generates code from `devops/04_iam_roles_policies_mlops.md` THEN the system specifies the CodeBuild role with `ecr:*` on project repos, but `ecr:*` includes destructive actions like `ecr:DeleteRepository` and `ecr:DeleteLifecyclePolicy` that CodeBuild should never need, violating least-privilege principle

1.18 WHEN a user generates code from `mlops/02_llm_finetuning_pipeline.md` THEN the system shows `environment={"HUGGING_FACE_HUB_TOKEN": hf_token}` passed directly to the SageMaker Estimator, contradicting the template's own security requirement that says "HuggingFace token from AWS Secrets Manager (not env vars in plain text)" — the scaffolding hint directly violates the stated security constraint

1.19 WHEN a user generates code from `devops/04_iam_roles_policies_mlops.md` THEN the system specifies the Bedrock access role with trust policy for `bedrock.amazonaws.com` and permission `bedrock:*` for customization, but `bedrock:*` includes `bedrock:DeleteCustomModel`, `bedrock:DeleteProvisionedModelThroughput`, and `bedrock:DeleteGuardrail` which should not be granted to the service role

**Category F: Incorrect OpenSearch/Vector Search Configuration**

1.20 WHEN a user generates code from `mlops/04_rag_pipeline.md` THEN the system provides an AOSS index mapping using `"engine": "nmslib"` for the HNSW vector search method, but Amazon OpenSearch Serverless (AOSS) only supports the `faiss` and `nmslib` engines for k-NN — however, for AOSS collections of type VECTORSEARCH, the recommended and default engine is `faiss`, and using `nmslib` with `cosinesimil` space type may not be supported in AOSS (only in managed OpenSearch domains)

1.21 WHEN a user generates code from `mlops/04_rag_pipeline.md` THEN the system specifies `"knn.algo_param.ef_search": 512` in index settings, but for AOSS the k-NN settings are managed at the collection level and cannot be set via index settings — this setting is only valid for self-managed OpenSearch domains, causing an index creation error on AOSS

**Category G: Inconsistent Cross-Template References & Missing Parameters**

1.22 WHEN a user generates code from `mlops/03_llm_inference_deployment.md` for inference-components THEN the system shows `create_endpoint_config()` with `ManagedInstanceScaling` and `RoutingConfig` inside `ProductionVariants`, but the `ManagedInstanceScaling` parameter is part of the `CreateEndpointConfig` API only when using inference components — the scaffolding mixes endpoint config creation with inference component creation patterns, and `RoutingConfig` with `RoutingStrategy` is set at the endpoint config level, not the variant level

1.23 WHEN a user generates code from `mlops/03_llm_inference_deployment.md` THEN the system shows `HuggingFaceModel` with `transformers_version="4.37"` and `pytorch_version="2.1"` and `py_version="py310"`, which are outdated versions that may not include the latest HuggingFace LLM DLC features and bug fixes — the template header claims `sagemaker-python-sdk: 2.230+` but the DLC versions referenced are from early 2024

1.24 WHEN a user generates code from `mlops/00_sagemaker_ai_workspace.md` THEN the system references `SageMaker Unified Studio (latest 2025 features)` in the role definition but the actual API calls use `create_domain()` with `DefaultUserSettings` containing `JupyterServerAppSettings` — JupyterServer app type is the legacy Studio Classic experience, while the new Studio experience uses `JupyterLabAppSettings` and `CodeEditorAppSettings`, creating confusion about which Studio experience is being configured

1.25 WHEN a user generates code from `mlops/05_model_monitoring_drift.md` THEN the system shows `DataCaptureConfig` with `capture_options=["REQUEST", "RESPONSE"]` but the correct SageMaker SDK enum values are `["Input", "Output"]` — using `"REQUEST"` and `"RESPONSE"` will cause a validation error when creating the data capture configuration

**Category H: Docker/Container Errors**

1.26 WHEN a user generates code from `iac/04_cdk_ecs_llm_inference.md` THEN the system shows `Dockerfile.vllm` with `FROM vllm/vllm-openai:latest` followed by `EXPOSE 8080` and `CMD ["python", "-m", "vllm.entrypoints.openai.api_server", ...]`, but the vLLM OpenAI-compatible server defaults to port 8000, not 8080 — the port mismatch between the Dockerfile EXPOSE, the container health check (`curl -f http://localhost:8080/health`), and the actual vLLM default port will cause health check failures

1.27 WHEN a user generates code from `mlops/03_llm_inference_deployment.md` for ecs-vllm THEN the system shows the ECS task definition with `"portMappings": [{"containerPort": 8000}]` which correctly uses port 8000, but `iac/04_cdk_ecs_llm_inference.md` uses port 8080 for the same vLLM container — the two templates that should produce compatible configurations have conflicting port numbers

**Category I: Python SDK / boto3 API Errors**

1.28 WHEN a user generates code from `mlops/12_bedrock_guardrails_agents.md` THEN the system uses `bedrock_runtime.apply_guardrail()` with `content=[{"text": {"text": query}}]`, but the correct `ApplyGuardrail` API parameter structure is `content=[{"text": {"text": query, "qualifiers": ["query"]}}]` — the missing `qualifiers` field may cause the guardrail to not properly classify the content as input vs grounding source

1.29 WHEN a user generates code from `mlops/07_feature_store.md` THEN the system states "max 500 records per `put_record` batch" in the requirements, but `put_record` is a single-record API — the batch API is `batch_get_record` for reads, and for batch writes you use `feature_group.ingest()` with the SageMaker SDK which handles batching internally, not `put_record`

1.30 WHEN a user generates code from `mlops/04_rag_pipeline.md` THEN the system shows Bedrock embedding invocation using `bedrock_runtime.invoke_model()` with `modelId="amazon.titan-embed-text-v2:0"` and body `{"inputText": chunk_text, "dimensions": 1536, "normalize": True}`, but the Titan Embed V2 API uses `"normalize"` as a boolean which should be lowercase `true` in JSON (not Python `True`), and the response parsing `json.loads(response["body"].read())["embedding"]` is correct but the template says "batch 25 docs/call" while `invoke_model` only processes one document per call — batch embedding requires `invoke_model` to be called per-document or using the Titan batch API

### Expected Behavior (Correct)

**Category A: Deprecated/Incorrect SDK Versions & API References**

2.1 WHEN a user generates code from `mlops/02_llm_finetuning_pipeline.md` THEN the system SHALL produce a SageMaker HuggingFace Estimator with `transformers_version="4.45"` (or latest compatible), `pytorch_version="2.5"`, and `py_version="py311"` matching the template header's stated compatibility, and SHALL note that users should verify the latest available DLC versions via `aws ecr describe-images`

2.2 WHEN a user generates code from `mlops/01_sagemaker_training_pipeline.md` THEN the system SHALL reference metric logging via stdout print statements matching the `metric_definitions` regex patterns (e.g., `print(f"train_loss: {loss}")`) rather than a non-existent `sagemaker.log_metric()` function

2.3 WHEN a user generates code from `mlops/06_experiment_tracking.md` for managed-mlflow THEN the system SHALL correctly reference `boto3.client("sagemaker").create_mlflow_tracking_server()` as the boto3 client method, not as a SageMaker Python SDK method

2.4 WHEN a user generates code from `mlops/06_experiment_tracking.md` for managed-mlflow THEN the system SHALL show the correct response parsing for `describe_mlflow_tracking_server()` matching the actual boto3 API response schema

**Category B: Incorrect Bedrock Model Identifiers & API Parameters**

2.5 WHEN a user generates code from `iac/03_terraform_bedrock_opensearch_rag.md` THEN the system SHALL use verified Bedrock model ARN formats and SHALL include a comment noting that model IDs should be verified against `aws bedrock list-foundation-models` for the target region

2.6 WHEN a user generates code from `mlops/09_bedrock_finetuning.md` THEN the system SHALL provide model-family-specific `invoke_model` request body examples (Titan: `{"inputText": "...", "textGenerationConfig": {...}}`, Llama: `{"prompt": "...", "max_gen_len": 512}`) rather than a generic body that fails for all families

2.7 WHEN a user generates code from `mlops/09_bedrock_finetuning.md` THEN the system SHALL only list Bedrock model IDs that are verified to support fine-tuning and SHALL include a note to check `aws bedrock list-foundation-models --by-customization-type FINE_TUNING` for current availability

2.8 WHEN a user generates code from `mlops/09_bedrock_finetuning.md` THEN the system SHALL provide Llama 3.1 chat template format for Llama 3.1 models and Llama 2 format only for Llama 2 models, matching the actual model family's expected training data format

2.9 WHEN a user generates code from `mlops/12_bedrock_guardrails_agents.md` THEN the system SHALL configure `PROMPT_ATTACK` filter with `inputStrength` only and SHALL NOT set `outputStrength` for the `PROMPT_ATTACK` filter type, matching the Bedrock Guardrails API requirements

**Category C: Terraform/CDK Syntax & Resource Errors**

2.10 WHEN a user generates code from `iac/03_terraform_bedrock_opensearch_rag.md` THEN the system SHALL include a `required_providers` block in `providers.tf` enforcing `aws >= 5.70.0` and SHALL document which provider version introduced each Bedrock resource type

2.11 WHEN a user generates code from `iac/04_cdk_ecs_llm_inference.md` THEN the system SHALL use `container.add_resource_requirements(gpu_count=1)` or the correct CDK API for GPU resource reservation in ECS task definitions, not a non-existent `gpu_count` parameter on `add_container()`

2.12 WHEN a user generates code from `iac/02_cdk_ml_llm_infrastructure.md` THEN the system SHALL use consistent CDK cross-stack reference patterns, either `cdk.CfnOutput` + `cdk.Fn.import_value()` or direct stack attribute references, with correct Python import paths

**Category D: Incorrect YAML/Config Syntax in CI/CD Templates**

2.13 WHEN a user generates code from `cicd/05_bitbucket_pipelines_aws.md` THEN the system SHALL use `services: [docker]` for Docker support and SHALL NOT reference a `docker` cache type — Docker layer caching SHALL be configured via `options: docker: true` or the `atlassian/docker-layer-caching` pipe

2.14 WHEN a user generates code from `cicd/05_bitbucket_pipelines_aws.md` THEN the system SHALL define custom pip cache with the correct Bitbucket Pipelines syntax using absolute paths (e.g., `/root/.cache/pip` or `$HOME/.cache/pip`) rather than tilde-based paths

2.15 WHEN a user generates code from `cicd/04_github_actions_aws_integration.md` THEN the system SHALL use block-style YAML consistent with GitHub Actions documentation conventions and SHALL show the complete `on` trigger configuration including `workflow_dispatch` inputs properly nested

**Category E: Security Misconfigurations**

2.16 WHEN a user generates code from `devops/04_iam_roles_policies_mlops.md` THEN the system SHALL provide explicit IAM condition keys (`aws:ResourceTag/Project`, `sagemaker:ResourceTag/Project`) in the SageMaker execution role policy to scope `sagemaker:*` to project-tagged resources only, with example condition blocks

2.17 WHEN a user generates code from `devops/04_iam_roles_policies_mlops.md` THEN the system SHALL scope the CodeBuild ECR permissions to only `ecr:GetAuthorizationToken`, `ecr:BatchCheckLayerAvailability`, `ecr:GetDownloadUrlForLayer`, `ecr:BatchGetImage`, `ecr:PutImage`, `ecr:InitiateLayerUpload`, `ecr:UploadLayerPart`, `ecr:CompleteLayerUpload` — the minimum set needed for Docker build and push

2.18 WHEN a user generates code from `mlops/02_llm_finetuning_pipeline.md` THEN the system SHALL show the HuggingFace token being retrieved from Secrets Manager at runtime inside the training script, not passed as a plain-text environment variable to the Estimator, consistent with the template's own security requirements

2.19 WHEN a user generates code from `devops/04_iam_roles_policies_mlops.md` THEN the system SHALL scope the Bedrock access role to specific actions needed (`bedrock:InvokeModel`, `bedrock:CreateModelCustomizationJob`, `bedrock:GetModelCustomizationJob`, `bedrock:CreateProvisionedModelThroughput`, `bedrock:GetProvisionedModelThroughput`) without granting `bedrock:Delete*` actions

**Category F: Incorrect OpenSearch/Vector Search Configuration**

2.20 WHEN a user generates code from `mlops/04_rag_pipeline.md` THEN the system SHALL use `"engine": "faiss"` for AOSS VECTORSEARCH collections and SHALL note that `nmslib` is only supported on self-managed OpenSearch domains

2.21 WHEN a user generates code from `mlops/04_rag_pipeline.md` THEN the system SHALL NOT include `knn.algo_param.ef_search` in AOSS index settings and SHALL note that k-NN algorithm parameters are managed at the AOSS collection level, not the index level

**Category G: Inconsistent Cross-Template References & Missing Parameters**

2.22 WHEN a user generates code from `mlops/03_llm_inference_deployment.md` for inference-components THEN the system SHALL correctly separate the endpoint config creation (with `ProductionVariants` including `ManagedInstanceScaling`) from the inference component creation, and SHALL place `RoutingConfig` at the correct API level

2.23 WHEN a user generates code from `mlops/03_llm_inference_deployment.md` THEN the system SHALL use DLC versions consistent with the stated SDK compatibility (`sagemaker-python-sdk: 2.230+`) and SHALL recommend users verify available DLC versions via the SageMaker DLC release notes

2.24 WHEN a user generates code from `mlops/00_sagemaker_ai_workspace.md` THEN the system SHALL clearly distinguish between Studio Classic (`JupyterServerAppSettings`) and the new Studio experience (`JupyterLabAppSettings`, `CodeEditorAppSettings`) and SHALL default to the new Studio experience with `JupyterLabAppSettings` as the primary configuration

2.25 WHEN a user generates code from `mlops/05_model_monitoring_drift.md` THEN the system SHALL use the correct SageMaker SDK `DataCaptureConfig` capture options `["Input", "Output"]` instead of `["REQUEST", "RESPONSE"]`

**Category H: Docker/Container Errors**

2.26 WHEN a user generates code from `iac/04_cdk_ecs_llm_inference.md` THEN the system SHALL use port 8000 (the vLLM default) consistently across the Dockerfile EXPOSE directive, the container health check URL, the ECS port mapping, and the ALB target group configuration

2.27 WHEN a user generates code from `iac/04_cdk_ecs_llm_inference.md` and `mlops/03_llm_inference_deployment.md` THEN the system SHALL use the same port number (8000) for vLLM containers across both templates to ensure cross-template compatibility

**Category I: Python SDK / boto3 API Errors**

2.28 WHEN a user generates code from `mlops/12_bedrock_guardrails_agents.md` THEN the system SHALL use the correct `ApplyGuardrail` API content structure including appropriate qualifiers for input classification

2.29 WHEN a user generates code from `mlops/07_feature_store.md` THEN the system SHALL correctly describe `put_record` as a single-record API and SHALL reference `feature_group.ingest()` for batch writes, with accurate documentation of the batch ingestion API

2.30 WHEN a user generates code from `mlops/04_rag_pipeline.md` THEN the system SHALL correctly show that `invoke_model` processes one document per call and SHALL provide a batch embedding helper function that calls `invoke_model` in a loop with proper rate limiting, and SHALL use JSON-valid `true`/`false` (not Python `True`/`False`) in the request body example

### Unchanged Behavior (Regression Prevention)

3.1 WHEN a user generates code from any template with correct parameter substitution THEN the system SHALL CONTINUE TO produce the complete file structure as specified in each template's Task section

3.2 WHEN a user generates code from templates with environment-specific configuration (dev/stage/prod) THEN the system SHALL CONTINUE TO apply environment-appropriate defaults (spot instances for dev, manual approval for prod, etc.)

3.3 WHEN a user generates code from `mlops/01_sagemaker_training_pipeline.md` with valid ML_FRAMEWORK and metric definitions THEN the system SHALL CONTINUE TO produce a working SageMaker Pipeline with ProcessingStep, TrainingStep, ConditionStep, and ModelStep in correct dependency order

3.4 WHEN a user generates code from `devops/02_vpc_networking_ml.md` THEN the system SHALL CONTINUE TO create VPC with public, private, and isolated subnets, VPC endpoints for AWS services, and security groups with correct ingress/egress rules

3.5 WHEN a user generates code from `cicd/03_codepipeline_multistage_ml.md` THEN the system SHALL CONTINUE TO produce a multi-stage CodePipeline V2 with Source, Build, Train, Evaluate, Approve, and Deploy stages with correct stage dependencies

3.6 WHEN a user generates code from `mlops/04_rag_pipeline.md` with `RAG_APPROACH=bedrock-kb` THEN the system SHALL CONTINUE TO produce a working Bedrock Knowledge Base setup with AOSS collection, data source, and retrieve_and_generate query function

3.7 WHEN a user generates code from `mlops/08_sagemaker_pipelines_e2e.md` THEN the system SHALL CONTINUE TO produce a complete Pipeline DAG with preprocessing, training, evaluation, condition, registration, and optional deployment steps

3.8 WHEN a user generates code from `mlops/13_continuous_training_data_versioning.md` THEN the system SHALL CONTINUE TO produce a Step Functions state machine with budget check, frequency limiter, data validation, training, champion-challenger comparison, and promotion decision steps

3.9 WHEN a user generates code from `mlops/12_bedrock_guardrails_agents.md` THEN the system SHALL CONTINUE TO produce guardrails with content filters, denied topics, PII entity filters, word filters, and contextual grounding policies

3.10 WHEN a user generates code from `devops/04_iam_roles_policies_mlops.md` for GitHub OIDC THEN the system SHALL CONTINUE TO produce an OIDC identity provider for `token.actions.githubusercontent.com` with trust policy restricting to specific repo and branch via `sub` claim condition
