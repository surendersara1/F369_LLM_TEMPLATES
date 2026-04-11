# Template Library Bugfix Review — Design

## Overview

The F369 AWS MLOps & AI/LLM CI/CD Prompt Template Library contains 27 markdown template files with embedded code examples (Terraform, CDK Python, YAML, Dockerfiles, IAM policies, Python scripts). Systematic review identified 30 bugs across 9 categories in 16 files. These bugs include incorrect AWS API references, deprecated SDK versions, wrong API parameter names, incorrect model identifiers, security misconfigurations, Terraform/CDK syntax errors, and inconsistent scaffolding hints. The fix approach is file-by-file: edit each affected markdown template to correct the embedded code examples, then validate each correction against AWS documentation.

## Glossary

- **Bug_Condition (C)**: A code example or scaffolding hint in a markdown template that contains an incorrect API reference, deprecated version, wrong parameter, security misconfiguration, or syntax error that would cause generated code to fail or produce insecure infrastructure
- **Property (P)**: Each corrected code example SHALL match current AWS API specifications, use supported SDK versions, follow least-privilege security principles, and use correct syntax for the target language/tool
- **Preservation**: Template structure, parameter substitution patterns, file output specifications, environment conventions, and all correct code examples must remain unchanged
- **Template file**: A markdown `.md` file containing embedded code blocks that serve as scaffolding hints for LLM code generation
- **Scaffolding hint**: An embedded code example within a template that guides the LLM to produce specific infrastructure code

## Bug Details

### Bug Condition

The bugs manifest when a user feeds a template to an LLM for code generation and the embedded code examples contain incorrect API references, deprecated versions, wrong parameters, or security misconfigurations. The generated code will either fail at deploy time, produce insecure infrastructure, or behave inconsistently across templates.

**Formal Specification:**
```
FUNCTION isBugCondition(input)
  INPUT: input of type TemplateCodeExample
  OUTPUT: boolean

  RETURN (input.apiReference != currentAWSDocumentation(input.service, input.apiCall))
         OR (input.sdkVersion < minimumSupportedVersion(input.sdk))
         OR (input.parameterName NOT IN validParameters(input.apiCall))
         OR (input.modelId NOT IN currentBedrockModelRegistry())
         OR (input.iamPermissions VIOLATES leastPrivilegePrinciple())
         OR (input.syntaxValid == false FOR input.targetLanguage)
         OR (input.portConfig INCONSISTENT_WITH input.relatedTemplates)
END FUNCTION
```

### Examples

- Bug 1.1: `mlops/02` uses `transformers_version="4.36"` but header claims `4.45+` — version mismatch causes import errors
- Bug 1.9: `mlops/12` sets `outputStrength` on `PROMPT_ATTACK` filter — Bedrock API rejects this configuration
- Bug 1.11: `iac/04` uses `gpu_count=1` on `add_container()` — CDK synthesis fails, no such parameter
- Bug 1.16: `devops/04` uses `sagemaker:*` on `Resource: *` — overly permissive, violates least-privilege
- Bug 1.25: `mlops/05` uses `capture_options=["REQUEST", "RESPONSE"]` — correct values are `["Input", "Output"]`
- Bug 1.26: `iac/04` uses port 8080 for vLLM but vLLM defaults to 8000 — health check failures

## Expected Behavior

### Preservation Requirements

**Unchanged Behaviors:**
- All template file structures (headers, parameter sections, task sections, output file lists) must remain intact
- All correct code examples that are not identified as bugs must remain unchanged
- Environment-specific configuration patterns (dev/stage/prod) must be preserved
- Parameter substitution patterns (`[PARAMETER_NAME]`) must be preserved
- Cross-template references that are correct must be preserved
- All markdown formatting, section headers, and documentation text not related to bugs must be preserved

**Scope:**
All template content that does NOT contain one of the 30 identified bugs should be completely unaffected by fixes. This includes:
- Template headers and parameter definitions
- Correct code examples and scaffolding hints
- Documentation text and explanations
- File structure specifications in Task sections
- Environment convention tables

## Hypothesized Root Cause

Based on the bug analysis, the root causes fall into these categories:

1. **Stale SDK/API References**: Templates were written against older SDK versions and not updated as AWS released new DLC images, API versions, and model IDs. Examples: bugs 1.1, 1.23 (outdated DLC versions), bugs 1.3, 1.4 (wrong SDK layer for MLflow API)

2. **Incorrect API Parameter Names**: Template authors used parameter names from memory or documentation for a different API version. Examples: bugs 1.25 (`REQUEST`/`RESPONSE` vs `Input`/`Output`), 1.11 (`gpu_count` not a CDK parameter), 1.28 (missing `qualifiers` field)

3. **Model ID / Format Mismatches**: Bedrock model IDs and training data formats were not verified against the actual model registry. Examples: bugs 1.6, 1.7, 1.8 (wrong invoke body, wrong model IDs, wrong chat template)

4. **Security Shortcuts**: IAM policies used wildcard actions for convenience rather than enumerating least-privilege actions. Examples: bugs 1.16, 1.17, 1.19 (`sagemaker:*`, `ecr:*`, `bedrock:*`)

5. **Cross-Template Inconsistency**: Templates developed independently without cross-referencing shared configurations. Examples: bugs 1.26, 1.27 (port 8080 vs 8000 for vLLM across templates)

6. **Platform-Specific Syntax Errors**: CI/CD platform YAML syntax was not validated against platform documentation. Examples: bugs 1.13, 1.14 (Bitbucket Pipelines cache syntax)

7. **API Scope Confusion**: Conflating single-record APIs with batch APIs, or mixing endpoint-level vs component-level parameters. Examples: bugs 1.22, 1.29, 1.30

## Correctness Properties

Property 1: Bug Condition — Corrected Code Examples Match AWS Documentation

_For any_ template code example that was identified as buggy (isBugCondition returns true), the fixed template SHALL contain a corrected code example that matches current AWS API documentation, uses supported SDK versions, follows correct parameter names, and uses valid syntax for the target language/tool.

**Validates: Requirements 2.1–2.30**

Property 2: Preservation — Unchanged Template Content

_For any_ template content that is NOT identified as buggy (isBugCondition returns false), the fixed template SHALL be identical to the original template, preserving all correct code examples, documentation, structure, and formatting.

**Validates: Requirements 3.1–3.10**

## Fix Implementation

### Changes Required

Fixes are organized by file. Each file is edited independently and validated against AWS documentation.

**File**: `mlops/02_llm_finetuning_pipeline.md`
- Bug 1.1: Update HuggingFace Estimator versions to `transformers_version="4.45"`, `pytorch_version="2.5"`, `py_version="py311"`
- Bug 1.18: Replace plain-text `environment={"HUGGING_FACE_HUB_TOKEN": hf_token}` with Secrets Manager retrieval inside training script

**File**: `mlops/01_sagemaker_training_pipeline.md`
- Bug 1.2: Replace `sagemaker.log_metric()` with stdout print pattern matching `metric_definitions` regex

**File**: `mlops/06_experiment_tracking.md`
- Bug 1.3: Change `sagemaker.create_mlflow_tracking_server()` to `boto3.client("sagemaker").create_mlflow_tracking_server()`
- Bug 1.4: Fix `describe_mlflow_tracking_server()` response parsing to match actual boto3 response schema

**File**: `iac/03_terraform_bedrock_opensearch_rag.md`
- Bug 1.5: Verify and correct Bedrock model ARN format, add comment about `list-foundation-models`
- Bug 1.10: Add `required_providers` block enforcing `aws >= 5.70.0` in providers.tf scaffolding

**File**: `mlops/09_bedrock_finetuning.md`
- Bug 1.6: Provide model-family-specific `invoke_model` request body examples
- Bug 1.7: Verify fine-tunable model IDs, add note about `list-foundation-models --by-customization-type`
- Bug 1.8: Update Llama training data format to Llama 3.1 chat template

**File**: `mlops/12_bedrock_guardrails_agents.md`
- Bug 1.9: Remove `outputStrength` from `PROMPT_ATTACK` filter config
- Bug 1.28: Add `qualifiers` field to `apply_guardrail()` content structure

**File**: `iac/04_cdk_ecs_llm_inference.md`
- Bug 1.11: Replace `gpu_count=1` with correct CDK GPU resource API
- Bug 1.26: Change port 8080 to 8000 across Dockerfile, health check, and port mapping
- Bug 1.27: Ensure port 8000 consistency with `mlops/03`

**File**: `iac/02_cdk_ml_llm_infrastructure.md`
- Bug 1.12: Fix cross-stack reference pattern to use consistent `cdk.Fn.import_value()` syntax

**File**: `cicd/05_bitbucket_pipelines_aws.md`
- Bug 1.13: Replace `caches: [docker]` with `services: [docker]` and proper Docker layer caching config
- Bug 1.14: Fix custom pip cache path from `~/.cache/pip` to absolute path

**File**: `cicd/04_github_actions_aws_integration.md`
- Bug 1.15: Convert flow-style YAML to block-style, fix `workflow_dispatch` nesting

**File**: `devops/04_iam_roles_policies_mlops.md`
- Bug 1.16: Add IAM condition keys for SageMaker resource scoping
- Bug 1.17: Replace `ecr:*` with minimum required ECR actions for CodeBuild
- Bug 1.19: Replace `bedrock:*` with specific required Bedrock actions

**File**: `mlops/04_rag_pipeline.md`
- Bug 1.20: Change `"engine": "nmslib"` to `"engine": "faiss"` for AOSS
- Bug 1.21: Remove `knn.algo_param.ef_search` from AOSS index settings
- Bug 1.30: Fix JSON boolean (`true` not `True`), clarify single-doc-per-call, add batch helper

**File**: `mlops/03_llm_inference_deployment.md`
- Bug 1.22: Separate endpoint config from inference component creation patterns
- Bug 1.23: Update DLC versions to match stated SDK compatibility
- Bug 1.27: Ensure vLLM port 8000 consistency with `iac/04`

**File**: `mlops/05_model_monitoring_drift.md`
- Bug 1.25: Change `capture_options=["REQUEST", "RESPONSE"]` to `["Input", "Output"]`

**File**: `mlops/00_sagemaker_ai_workspace.md`
- Bug 1.24: Update to use `JupyterLabAppSettings` for new Studio experience, clarify Studio Classic vs new Studio

**File**: `mlops/07_feature_store.md`
- Bug 1.29: Correct `put_record` documentation to single-record API, reference `feature_group.ingest()` for batch

## Testing Strategy

### Validation Approach

Since these are markdown template files (not executable code), testing follows a documentation-validation approach: each fix is validated against current AWS documentation using MCP servers, then the template is reviewed to ensure no unintended changes to surrounding content.

### Exploratory Bug Condition Checking

**Goal**: Confirm each bug exists in the current template file before applying fixes.

**Test Plan**: Read each affected file, locate the buggy code example, and confirm it matches the bug description from the requirements document.

**Test Cases**:
1. Read each file and confirm the buggy code snippet exists as described
2. Cross-reference with AWS documentation to confirm the code is indeed incorrect
3. For cross-template bugs (1.27), confirm inconsistency exists between both files

**Expected Counterexamples**:
- Code examples that don't match current AWS API documentation
- Version numbers that are outdated or inconsistent with template headers
- IAM policies with overly broad permissions

### Fix Checking

**Goal**: Verify that each corrected code example matches current AWS documentation.

**Pseudocode:**
```
FOR ALL bugfix WHERE isBugCondition(originalCode) DO
  fixedCode := applyFix(originalCode, bugfix)
  ASSERT matchesAWSDocumentation(fixedCode)
  ASSERT validSyntax(fixedCode, targetLanguage)
END FOR
```

### Preservation Checking

**Goal**: Verify that no unintended changes were made to template content outside the bug fixes.

**Pseudocode:**
```
FOR ALL templateSection WHERE NOT containsBug(templateSection) DO
  ASSERT originalTemplate(templateSection) = fixedTemplate(templateSection)
END FOR
```

**Testing Approach**: Manual diff review of each file after fixes to ensure only the targeted code examples were changed.

**Test Cases**:
1. Template headers and parameter sections unchanged
2. Task sections and output file lists unchanged
3. Correct code examples not modified
4. Markdown formatting preserved

### Unit Tests

- Verify each corrected API reference matches AWS documentation
- Verify each corrected version number is currently supported
- Verify each corrected IAM policy follows least-privilege

### Property-Based Tests

- For each file, verify that only the identified buggy sections were modified
- For cross-template fixes, verify consistency across all affected templates

### Integration Tests

- Verify cross-template references remain consistent after all fixes
- Verify the template library index (README.md, Library.md) still accurately describes templates
