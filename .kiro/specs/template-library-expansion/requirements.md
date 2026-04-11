# Requirements Document

## Introduction

This document specifies requirements for expanding the F369 AWS MLOps & AI/LLM CI/CD Prompt Template Library from 27 templates across 4 directories to approximately 65 templates across 7 directories (4 existing + 3 new). The expansion covers 7 areas: AI/ML Gaps, Data Engineering, Security & Compliance, Cost Management (FinOps), Multi-Account / Enterprise, Observability, and Edge / Hybrid. Each new template follows the established format: markdown files with embedded code examples that users feed to LLMs to generate production-ready AWS infrastructure code. All templates must be validated against current AWS APIs (boto3 1.35+, Terraform AWS Provider 5.70+, CDK 2.170+).

## Glossary

- **Template**: A markdown file following the F369 format (Version comment, Purpose, Role Definition, Context & Inputs, Task, Output Format, Requirements & Constraints, Code Scaffolding Hints, Integration Points) that users paste into an LLM to generate deployable AWS infrastructure code.
- **Template_Library**: The collection of all F369 prompt template markdown files organized into category directories.
- **Library_Index**: The README.md and Library.md files that catalog all templates with descriptions and recommended build order.
- **PROMPT_GUIDE**: The PROMPT_GUIDE.md file that documents template anatomy, chaining patterns, and usage instructions.
- **Integration_Points_Section**: The section in each template that documents upstream dependencies and downstream consumers using the format `category/number` (e.g., `mlops/05`, `devops/03`).
- **Context_Inputs_Section**: The parameterized section at the top of each template where users fill in `[REQUIRED]` and `[OPTIONAL]` values before prompting.
- **Code_Scaffolding_Hints_Section**: The section providing concrete AWS SDK calls, API patterns, and code snippets that guide the LLM toward correct implementations.
- **Task_Section**: The section defining the project directory structure and file-by-file generation instructions.
- **Expansion_Area**: One of the 7 thematic groups of new templates (AI/ML Gaps, Data Engineering, Security & Compliance, Cost Management, Multi-Account / Enterprise, Observability, Edge / Hybrid).
- **mlops_Directory**: The existing directory containing ML operations templates (currently 00–13, expanding to 00–19).
- **devops_Directory**: The existing directory containing DevOps supporting templates (currently 01–04, expanding to 01–14).
- **data_Directory**: A new directory for data engineering layer templates (01–05).
- **finops_Directory**: A new directory for cost management templates (01–05).
- **enterprise_Directory**: A new directory for multi-account and enterprise governance templates (01–05).
- **edge_Directory**: A new directory for edge and hybrid deployment templates (01–03).
- **Bedrock_Agent**: An Amazon Bedrock feature that orchestrates multi-step tasks using foundation models, action groups (Lambda-backed tools), knowledge bases, and conversation memory.
- **Action_Group**: A set of API operations (defined via OpenAPI schema) that a Bedrock Agent can invoke through Lambda functions.
- **Bedrock_Flows**: A visual orchestration feature in Amazon Bedrock for chaining prompts, knowledge bases, agents, and custom Lambda nodes into directed acyclic graphs.
- **Bedrock_Marketplace**: The Amazon Bedrock model marketplace for deploying third-party and partner foundation models with provisioned throughput.
- **Cross_Region_Inference_Profile**: A Bedrock feature that routes inference requests across multiple AWS regions for higher availability and throughput.
- **Prompt_Caching**: A technique to cache repeated prompt prefixes or semantic query results to reduce latency and cost for LLM inference.
- **Glue_ETL**: AWS Glue Extract-Transform-Load jobs using PySpark for data processing and ML feature engineering.
- **Lake_Formation**: AWS Lake Formation service for governed data lake access with fine-grained permissions and cross-account data sharing.
- **EventBridge_ML_Orchestration**: Using Amazon EventBridge rules and event buses to trigger ML workflows based on events (training completion, model approval, drift detection).
- **Config_Rules**: AWS Config managed and custom rules that evaluate resource compliance against defined policies.
- **Macie**: Amazon Macie service for automated PII and sensitive data discovery in S3 buckets.
- **FinOps**: Financial operations practices for managing and optimizing cloud spending, specifically for ML workloads.
- **SCP**: Service Control Policy in AWS Organizations that sets permission guardrails across accounts.
- **Service_Catalog**: AWS Service Catalog for creating self-service portfolios of approved ML infrastructure products.
- **Control_Tower**: AWS Control Tower for setting up and governing multi-account AWS environments with guardrails.
- **OpenTelemetry**: The OpenTelemetry observability framework for distributed tracing across ML inference pipelines.
- **SageMaker_Clarify**: SageMaker feature for bias detection, model explainability, and fairness monitoring.
- **SageMaker_Edge_Manager**: SageMaker feature for compiling, deploying, and managing ML models on edge devices.
- **IoT_Greengrass**: AWS IoT Greengrass service for running ML inference on edge devices with component-based deployment.
- **Outposts**: AWS Outposts for running AWS infrastructure on-premises, enabling local ML training and inference.


## Requirements

---

### Requirement 1: Template Format Compliance

**User Story:** As a template library maintainer, I want all new templates to follow the established F369 format, so that users have a consistent experience across the entire library.

#### Acceptance Criteria

1. THE Template_Library SHALL include the following sections in every new template file in this exact order: Template Version comment, Title and Purpose, Role Definition, Context & Inputs, Task (with project structure and file-by-file instructions), Output Format, Requirements & Constraints, Code Scaffolding Hints, and Integration Points.
2. WHEN a new template is created, THE Template SHALL begin with an HTML comment in the format `<!-- Template Version: 1.0 | boto3: 1.35+ | [additional tool versions] -->`.
3. THE Context_Inputs_Section SHALL mark every parameter as either `[REQUIRED]` or `[OPTIONAL: default_value]` with clear descriptions.
4. THE Task_Section SHALL include an ASCII directory tree showing the complete project structure that the LLM will generate.
5. THE Task_Section SHALL include file-by-file descriptions with concrete AWS API calls, SDK method signatures, and configuration patterns for each generated file.
6. THE Integration_Points_Section SHALL reference upstream dependencies and downstream consumers using the `category/number` format (e.g., `mlops/05`, `devops/03`) with a one-line description of the relationship.
7. THE Code_Scaffolding_Hints_Section SHALL include working code snippets using boto3 1.35+, Terraform AWS Provider 5.70+, or CDK 2.170+ that are validated against current AWS API signatures.
8. WHEN a template supports multiple IaC tools, THE Context_Inputs_Section SHALL include an `IAC_TOOL` parameter with options `boto3 | terraform | cdk`.
9. THE Template SHALL use the naming convention `{PROJECT_NAME}-{component}-{ENV}` for all generated AWS resource names.
10. THE Template SHALL include `PROJECT_NAME`, `AWS_REGION`, `AWS_ACCOUNT_ID`, and `ENV` as the first four required parameters in the Context_Inputs_Section.

---

### Requirement 2: Library Index and Documentation Updates

**User Story:** As a library user, I want the README.md, Library.md, and PROMPT_GUIDE.md to reflect all new templates, so that I can discover and chain templates across the expanded library.

#### Acceptance Criteria

1. WHEN all new templates are added, THE Library_Index SHALL update README.md to include all new templates in the Template Map with file path, number, and one-line description.
2. WHEN all new templates are added, THE Library_Index SHALL update Library.md to include the new directory tree showing all 65+ templates across 7 directories.
3. WHEN new directories (data/, finops/, enterprise/, edge/) are created, THE Library_Index SHALL add new category sections in README.md with a table for each new directory.
4. THE Library_Index SHALL update the Recommended Build Order in README.md to incorporate data engineering, security, FinOps, enterprise, observability, and edge templates at appropriate positions.
5. THE Library_Index SHALL update the Architecture Overview ASCII diagram in README.md to show the new layers (Data Engineering, Security, FinOps, Enterprise, Observability, Edge).
6. THE PROMPT_GUIDE SHALL update the Template Interaction Map to include cross-references for all new templates.
7. THE PROMPT_GUIDE SHALL add at least one new chaining example that demonstrates a workflow spanning the new expansion areas (e.g., Data Engineering → MLOps → Observability → FinOps).

---

### Requirement 3: AI/ML Gaps — Bedrock Agents with Action Groups (mlops/14)

**User Story:** As an AI engineer, I want a template for full Bedrock Agent orchestration with Action Groups, so that I can build multi-step agentic workflows with Lambda-backed tools, OpenAPI schemas, conversation memory, and multi-agent collaboration.

#### Acceptance Criteria

1. THE Template SHALL generate a complete Bedrock Agent with configurable agent instructions, foundation model selection, idle session TTL, and guardrail attachment.
2. WHEN Action_Group definitions are provided, THE Template SHALL generate Lambda function handlers, OpenAPI schema files, and `bedrock_agent.create_agent_action_group()` calls for each action group.
3. THE Template SHALL generate OpenAPI 3.0 schema definitions for each Action_Group that define the API operations the agent can invoke.
4. THE Template SHALL include conversation memory configuration using Bedrock session attributes and DynamoDB-backed session persistence.
5. THE Template SHALL include a multi-agent collaboration pattern where a supervisor agent delegates to specialized sub-agents using `bedrock_agent.create_agent()` with `agentCollaboration` configuration.
6. THE Code_Scaffolding_Hints_Section SHALL include working `bedrock_agent_runtime.invoke_agent()` calls with session management, streaming response handling, and trace logging.
7. IF an Action_Group Lambda function returns an error, THEN THE Template SHALL include error handling patterns that return structured error responses to the agent for graceful degradation.
8. THE Integration_Points_Section SHALL reference `mlops/12` (Guardrails), `mlops/04` (RAG Knowledge Bases), and `devops/04` (IAM roles) as upstream dependencies.

---

### Requirement 4: AI/ML Gaps — Multimodal Processing Pipelines (mlops/15)

**User Story:** As an ML engineer, I want a template for image, video, and audio processing pipelines, so that I can build multimodal AI applications using Bedrock Claude vision, Rekognition, Transcribe, and Textract with SageMaker processing jobs.

#### Acceptance Criteria

1. THE Template SHALL generate processing pipelines for three modalities: image (Bedrock Claude vision + Rekognition), audio (Transcribe + Bedrock summarization), and document (Textract + Bedrock extraction).
2. WHEN image processing is selected, THE Template SHALL generate code using `bedrock_runtime.invoke_model()` with base64-encoded image payloads for Claude vision analysis and `rekognition.detect_labels()` / `rekognition.detect_text()` for structured extraction.
3. WHEN audio processing is selected, THE Template SHALL generate code using `transcribe.start_transcription_job()` for speech-to-text and a Bedrock post-processing step for summarization.
4. WHEN document processing is selected, THE Template SHALL generate code using `textract.start_document_analysis()` with FeatureTypes (TABLES, FORMS, QUERIES) and a Bedrock post-processing step for structured data extraction.
5. THE Template SHALL include SageMaker Processing job wrappers for batch multimodal processing with S3 input/output and configurable instance types.
6. THE Template SHALL include an S3 event-driven trigger pattern using EventBridge or S3 notifications to automatically process uploaded media files.
7. IF a media file exceeds the Bedrock model input size limit, THEN THE Template SHALL include chunking or downsampling strategies with size validation before API calls.

---

### Requirement 5: AI/ML Gaps — Bedrock Flows Visual Orchestration (mlops/16)

**User Story:** As an AI engineer, I want a template for Bedrock Flows, so that I can visually orchestrate chains of prompts, knowledge bases, agents, and custom Lambda nodes into directed acyclic graphs.

#### Acceptance Criteria

1. THE Template SHALL generate a complete Bedrock Flow definition using `bedrock_agent.create_flow()` with input nodes, prompt nodes, knowledge base nodes, agent nodes, Lambda nodes, condition nodes, and output nodes.
2. THE Template SHALL include at least three flow patterns: a simple prompt chain (input → prompt → output), a RAG-augmented flow (input → knowledge base → prompt → output), and a conditional branching flow (input → condition → branch A / branch B → output).
3. WHEN a Lambda node is included, THE Template SHALL generate the Lambda function handler code and the flow node configuration referencing the Lambda ARN.
4. THE Template SHALL include flow versioning using `bedrock_agent.create_flow_version()` and alias management using `bedrock_agent.create_flow_alias()` for environment-based deployment.
5. THE Code_Scaffolding_Hints_Section SHALL include working `bedrock_agent_runtime.invoke_flow()` calls with input payload formatting and response parsing.
6. THE Template SHALL include a flow testing utility that invokes the flow with sample inputs and validates output structure.

---

### Requirement 6: AI/ML Gaps — LLM Evaluation Pipeline (mlops/17)

**User Story:** As an ML engineer, I want a systematic LLM evaluation pipeline, so that I can measure model quality using automated metrics, human-in-the-loop evaluation, A/B testing, and Bedrock Model Evaluation jobs.

#### Acceptance Criteria

1. THE Template SHALL generate an automated evaluation pipeline that computes ROUGE, BLEU, and BERTScore metrics against a ground-truth dataset stored in S3.
2. THE Template SHALL generate a Bedrock Model Evaluation job using `bedrock.create_evaluation_job()` with configurable task types (Summarization, QuestionAndAnswer, Classification, TextGeneration) and metric names.
3. THE Template SHALL include a human-in-the-loop evaluation workflow using a Step Functions state machine that routes evaluation samples to human reviewers via SNS notifications and collects ratings through an API Gateway endpoint.
4. THE Template SHALL include an A/B testing pattern using SageMaker inference endpoints with production variants that split traffic between two model versions and collect per-variant metrics.
5. THE Template SHALL generate an evaluation report aggregator that compares automated metrics, human ratings, and A/B test results into a single dashboard-ready JSON output.
6. WHEN evaluation metrics fall below configurable thresholds, THE Template SHALL include an EventBridge rule that blocks model promotion in the SageMaker Model Registry.
7. THE Code_Scaffolding_Hints_Section SHALL include working code for computing ROUGE and BLEU using the `rouge-score` and `nltk` Python libraries, and BERTScore using the `bert-score` library.

---

### Requirement 7: AI/ML Gaps — Prompt Caching Patterns (mlops/18)

**User Story:** As an AI engineer, I want prompt caching patterns, so that I can reduce latency and cost for repeated LLM inference calls using Bedrock prompt caching, semantic caching with ElastiCache, and DynamoDB-based caching.

#### Acceptance Criteria

1. THE Template SHALL generate a Bedrock prompt caching implementation using the Bedrock Converse API with cache point markers in the message content for system prompt and context caching.
2. THE Template SHALL generate a semantic caching layer using ElastiCache (Redis) that stores embedding vectors of queries and returns cached responses for semantically similar queries above a configurable similarity threshold.
3. THE Template SHALL generate a DynamoDB-based exact-match cache with TTL-based expiration for deterministic prompt-response pairs.
4. THE Template SHALL include cost reduction analysis code that compares cached vs uncached invocation costs and logs savings metrics to CloudWatch.
5. THE Template SHALL include cache invalidation strategies: TTL-based expiration, manual invalidation via API, and model-version-based invalidation when the underlying model changes.
6. WHEN a cache miss occurs, THE Template SHALL invoke the Bedrock model, store the response in the cache, and return the response to the caller in a single code path.
7. THE Code_Scaffolding_Hints_Section SHALL include working `bedrock_runtime.converse()` calls demonstrating the `cachePoint` content block for Bedrock-native prompt caching.

---

### Requirement 8: AI/ML Gaps — Bedrock Marketplace Models (mlops/19)

**User Story:** As an ML engineer, I want a template for deploying third-party models from Bedrock Marketplace, so that I can use partner models with provisioned throughput and cross-region inference profiles.

#### Acceptance Criteria

1. THE Template SHALL generate code to list available Bedrock Marketplace models using `bedrock.list_foundation_models()` filtered by provider and customization support.
2. THE Template SHALL generate provisioned throughput deployment for marketplace models using `bedrock.create_provisioned_model_throughput()` with configurable model units and commitment terms.
3. THE Template SHALL include cross-region inference profile configuration using `bedrock.create_inference_profile()` for routing requests across multiple regions for availability and throughput.
4. THE Template SHALL include a model comparison utility that benchmarks latency, throughput, and cost across multiple marketplace models using identical evaluation prompts.
5. WHEN a provisioned throughput deployment is created, THE Template SHALL include CloudWatch alarms for monitoring utilization and auto-scaling recommendations.
6. THE Template SHALL include cleanup scripts that delete provisioned throughput resources to prevent ongoing charges in non-production environments.


---

### Requirement 9: Data Engineering — Glue ETL for ML Feature Engineering (data/01)

**User Story:** As a data engineer, I want a template for Glue ETL jobs that produce ML features, so that I can build PySpark-based feature engineering pipelines with data quality checks, crawlers, and incremental processing.

#### Acceptance Criteria

1. THE Template SHALL generate Glue ETL job scripts using PySpark with configurable transforms for ML feature engineering: numerical encoding, categorical one-hot encoding, timestamp feature extraction, and text tokenization.
2. THE Template SHALL generate Glue Data Quality ruleset definitions using `glue.create_data_quality_ruleset()` with rules for completeness, uniqueness, freshness, and custom SQL-based checks.
3. THE Template SHALL generate Glue Crawler configurations using `glue.create_crawler()` for automatic schema discovery on S3 data sources with scheduling.
4. THE Template SHALL include job bookmark configuration for incremental processing that tracks previously processed data and only processes new records.
5. THE Template SHALL generate output in both Parquet format (for SageMaker training) and Feature Store ingestion format (compatible with `mlops/07`).
6. THE Code_Scaffolding_Hints_Section SHALL include working PySpark transform examples using `GlueContext`, `DynamicFrame`, and `ResolveChoice` for common ML feature patterns.
7. IF a Glue Data Quality check fails, THEN THE Template SHALL include an SNS notification and a Step Functions callback that halts downstream ML pipeline execution.

---

### Requirement 10: Data Engineering — Kinesis Real-Time Feature Computation (data/02)

**User Story:** As a data engineer, I want a template for real-time feature computation using Kinesis, so that I can process streaming data into ML features using Kinesis Data Streams, Lambda, Firehose, and Kinesis Analytics.

#### Acceptance Criteria

1. THE Template SHALL generate a Kinesis Data Stream with configurable shard count and a Lambda consumer function that computes real-time ML features from incoming records.
2. THE Template SHALL generate a Kinesis Data Firehose delivery stream configured to write processed features to S3 in Parquet format and optionally to OpenSearch for real-time feature serving.
3. THE Template SHALL generate a Kinesis Data Analytics (Apache Flink) application for streaming aggregations: sliding window averages, counts, and percentile computations over configurable time windows.
4. THE Template SHALL include a feature schema definition that maps raw stream fields to computed ML features with data type validation.
5. WHEN the Lambda consumer encounters a malformed record, THE Template SHALL include dead-letter queue (SQS) routing and CloudWatch error metric emission.
6. THE Template SHALL include integration with SageMaker Feature Store online store for real-time feature serving using `sagemaker_featurestore_runtime.put_record()`.
7. THE Code_Scaffolding_Hints_Section SHALL include working `kinesis.put_record()` producer examples and Lambda event source mapping configuration.

---

### Requirement 11: Data Engineering — Lake Formation ML Governance (data/03)

**User Story:** As a data governance lead, I want a template for Lake Formation governed data access, so that ML teams can access training data with fine-grained permissions, data catalog integration, and cross-account sharing.

#### Acceptance Criteria

1. THE Template SHALL generate Lake Formation database and table registrations using `lakeformation.register_resource()` and `lakeformation.grant_permissions()` for S3-backed data lakes.
2. THE Template SHALL generate fine-grained column-level and row-level permissions using Lake Formation tag-based access control (LF-TBAC) with configurable LF-Tags for data classification (e.g., `sensitivity=high`, `team=ml-research`).
3. THE Template SHALL generate cross-account data sharing configurations using `lakeformation.grant_permissions()` with `CatalogId` targeting external AWS account IDs.
4. THE Template SHALL include Glue Data Catalog table definitions with schema evolution support for ML datasets that change over time.
5. WHEN a SageMaker execution role requests data access, THE Template SHALL include Lake Formation permission grants scoped to the specific tables and columns required for training.
6. THE Template SHALL include an audit logging configuration using CloudTrail and Lake Formation audit logs for tracking data access by ML workloads.

---

### Requirement 12: Data Engineering — S3 Data Lifecycle for ML (data/04)

**User Story:** As an ML platform engineer, I want a template for S3 data lifecycle management, so that I can optimize storage costs for ML datasets using intelligent tiering, lifecycle policies, batch operations, and access points.

#### Acceptance Criteria

1. THE Template SHALL generate S3 bucket configurations with Intelligent-Tiering for ML datasets, including archive access tier configuration for infrequently accessed training data.
2. THE Template SHALL generate S3 Lifecycle rules that transition ML artifacts through storage classes: Standard → Intelligent-Tiering → Glacier Instant Retrieval → Glacier Deep Archive based on configurable age thresholds.
3. THE Template SHALL generate S3 Batch Operations job definitions using `s3control.create_job()` for bulk dataset management: copying datasets across buckets, tagging, and restoring from Glacier.
4. THE Template SHALL generate S3 Access Points with access point policies that isolate data access per ML team or project using `s3control.create_access_point()`.
5. THE Template SHALL include S3 Inventory configuration for tracking dataset sizes, storage classes, and encryption status across ML data buckets.
6. WHEN an ML training job requires data in Glacier, THE Template SHALL include a restore-and-wait pattern using S3 restore requests with EventBridge notifications for restore completion.

---

### Requirement 13: Data Engineering — EventBridge ML Orchestration (data/05)

**User Story:** As an ML platform engineer, I want a template for EventBridge-driven ML orchestration, so that I can build event-driven architectures that trigger ML workflows based on training completion, model approval, and drift detection events.

#### Acceptance Criteria

1. THE Template SHALL generate EventBridge rules for SageMaker events: `SageMaker Training Job State Change`, `SageMaker Endpoint Deployment State Change`, `SageMaker Model Package State Change`, and `SageMaker Pipeline Execution State Change`.
2. THE Template SHALL generate a custom event bus for ML-specific events with a schema registry defining event schemas for custom events (drift detected, evaluation complete, data quality check passed).
3. THE Template SHALL generate EventBridge rules that trigger Step Functions state machines for multi-step ML workflows (e.g., training complete → evaluate → approve → deploy).
4. THE Template SHALL include event pattern matching examples for filtering on specific training job statuses, model package approval statuses, and pipeline execution outcomes.
5. WHEN a drift detection event is published, THE Template SHALL include an EventBridge rule that triggers a retraining Step Functions workflow referencing `mlops/13`.
6. THE Template SHALL include cross-account event forwarding configuration for publishing ML events from workload accounts to a central observability account.
7. THE Code_Scaffolding_Hints_Section SHALL include working `events.put_rule()`, `events.put_targets()`, and `events.put_events()` calls with properly structured event detail payloads.


---

### Requirement 14: Security & Compliance — Config Rules for ML Compliance (devops/05)

**User Story:** As a security engineer, I want a template for AWS Config rules that enforce ML resource compliance, so that I can automatically detect non-compliant SageMaker resources, enforce encryption, restrict instance types, and implement ML-specific governance policies.

#### Acceptance Criteria

1. THE Template SHALL generate AWS Config managed rules for ML resources: `sagemaker-endpoint-configuration-kms-key-configured`, `sagemaker-notebook-instance-kms-key-configured`, and `sagemaker-notebook-no-direct-internet-access`.
2. THE Template SHALL generate custom Config rules using Lambda evaluators that check: SageMaker endpoints use only approved instance types from a configurable allowlist, SageMaker training jobs run in VPC-only mode, and Bedrock model access is restricted to approved model IDs.
3. THE Template SHALL generate a Config conformance pack that bundles all ML compliance rules into a single deployable unit using `config.put_conformance_pack()`.
4. THE Template SHALL include automated remediation using Systems Manager Automation documents that correct non-compliant resources (e.g., enable encryption on unencrypted notebook instances).
5. WHEN a Config rule evaluates a resource as NON_COMPLIANT, THE Template SHALL include an EventBridge rule that sends notifications via SNS and logs the finding to Security Hub.
6. THE Template SHALL include a compliance dashboard query using Config advanced queries (SQL) that reports compliance percentages across all ML resources.

---

### Requirement 15: Security & Compliance — GuardDuty and Security Hub for ML (devops/06)

**User Story:** As a security engineer, I want a template for GuardDuty and Security Hub integration for ML workloads, so that I can detect threats, aggregate security findings, automate remediation, and maintain compliance dashboards.

#### Acceptance Criteria

1. THE Template SHALL generate GuardDuty detector configuration with S3 protection enabled for ML data buckets and Lambda protection enabled for ML processing functions.
2. THE Template SHALL generate Security Hub custom actions and automation rules using `securityhub.create_automation_rule()` that auto-resolve low-severity findings and escalate high-severity findings for ML resources.
3. THE Template SHALL generate a Lambda-based automated remediation workflow triggered by Security Hub findings: isolate compromised SageMaker notebook instances, revoke IAM credentials, and quarantine suspicious S3 objects.
4. THE Template SHALL include Security Hub custom insights using `securityhub.create_insight()` that filter findings by ML resource types (SageMaker, Bedrock, Glue) for focused security monitoring.
5. WHEN GuardDuty detects anomalous API calls to SageMaker or Bedrock services, THE Template SHALL include an EventBridge rule that triggers the remediation Lambda within 5 minutes.
6. THE Template SHALL include a compliance standards mapping that maps Security Hub findings to relevant compliance frameworks (SOC 2, HIPAA, GDPR) for ML workloads.

---

### Requirement 16: Security & Compliance — Macie PII Detection in Training Data (devops/07)

**User Story:** As a data privacy engineer, I want a template for Macie PII detection in training data, so that I can automatically discover and classify sensitive data in S3 buckets used for ML training, and trigger remediation workflows.

#### Acceptance Criteria

1. THE Template SHALL generate Macie classification jobs using `macie2.create_classification_job()` targeting S3 buckets that contain ML training data, with configurable scheduling (one-time or recurring).
2. THE Template SHALL generate custom data identifiers using `macie2.create_custom_data_identifier()` for domain-specific sensitive data patterns (e.g., medical record numbers, internal employee IDs) beyond Macie's built-in detectors.
3. THE Template SHALL include an automated remediation workflow using EventBridge + Step Functions that: quarantines S3 objects containing PII, notifies the data owner via SNS, and triggers a data sanitization Lambda function.
4. THE Template SHALL generate a data sanitization Lambda function that redacts or anonymizes PII fields in JSONL and CSV training data files using configurable redaction patterns.
5. WHEN Macie detects PII in a training data bucket, THE Template SHALL include a mechanism to block SageMaker training jobs from accessing the affected data until remediation is complete (using S3 bucket policy updates or Lake Formation permission revocation).
6. THE Template SHALL include a Macie findings aggregation query using `macie2.get_finding_statistics()` that reports PII detection rates per bucket and data classification distribution.

---

### Requirement 17: Security & Compliance — KMS Encryption for ML (devops/08)

**User Story:** As a security engineer, I want a template for KMS key management across ML workloads, so that I can encrypt model artifacts, training data, endpoint volumes, and manage key rotation and cross-account key sharing.

#### Acceptance Criteria

1. THE Template SHALL generate customer-managed KMS keys with separate keys for: ML training data encryption, model artifact encryption, SageMaker endpoint volume encryption, and Bedrock custom model encryption.
2. THE Template SHALL generate KMS key policies that grant usage permissions to SageMaker execution roles, Bedrock service roles, Glue job roles, and CodeBuild roles using the `kms:ViaService` condition key to restrict usage to specific AWS services.
3. THE Template SHALL include automatic key rotation configuration using `kms.enable_key_rotation()` with a configurable rotation period.
4. THE Template SHALL generate cross-account key sharing policies using KMS key policy grants that allow specified external AWS accounts to use keys for model deployment.
5. THE Template SHALL include KMS key alias management using the naming convention `alias/{PROJECT_NAME}/{ENV}/{purpose}` (e.g., `alias/myai/prod/training-data`).
6. WHEN a KMS key is scheduled for deletion, THE Template SHALL include a CloudWatch alarm that detects `kms:ScheduleKeyDeletion` API calls and sends an immediate SNS notification.
7. THE Code_Scaffolding_Hints_Section SHALL include working examples of passing KMS key ARNs to `sagemaker.create_training_job()`, `sagemaker.create_endpoint_config()`, and `bedrock.create_model_customization_job()` for encryption configuration.

---

### Requirement 18: Security & Compliance — VPC Endpoint Policies for ML (devops/09)

**User Story:** As a network security engineer, I want a template for VPC endpoint policies scoped to ML services, so that I can restrict network access to SageMaker, Bedrock, and S3 endpoints with fine-grained policies.

#### Acceptance Criteria

1. THE Template SHALL generate VPC interface endpoints for ML services: `com.amazonaws.{region}.sagemaker.api`, `com.amazonaws.{region}.sagemaker.runtime`, `com.amazonaws.{region}.bedrock`, `com.amazonaws.{region}.bedrock-runtime`, and `com.amazonaws.{region}.bedrock-agent-runtime`.
2. THE Template SHALL generate VPC endpoint policies that restrict access to specific S3 buckets (ML data and artifact buckets only) on the S3 gateway endpoint using `aws:ResourceAccount` and bucket ARN conditions.
3. THE Template SHALL generate VPC endpoint policies for SageMaker API and runtime endpoints that restrict access to specific SageMaker resources using `sagemaker:ResourceTag/Project` condition keys.
4. THE Template SHALL include security group configurations for each VPC endpoint that restrict inbound traffic to specific CIDR ranges or security groups used by ML workloads.
5. WHEN a new ML service endpoint is required, THE Template SHALL include a parameterized endpoint creation pattern that accepts service name, policy document, and subnet placement as inputs.
6. THE Template SHALL include a VPC endpoint policy testing utility that validates policies by attempting allowed and denied API calls and reporting results.


---

### Requirement 19: Cost Management — Cost Allocation for ML Workloads (finops/01)

**User Story:** As a FinOps practitioner, I want a template for ML cost allocation and tracking, so that I can tag resources, set budgets, detect cost anomalies, and track per-experiment costs across ML workloads.

#### Acceptance Criteria

1. THE Template SHALL generate a cost allocation tagging strategy with mandatory tags (`Project`, `Environment`, `Team`, `CostCenter`, `ExperimentId`) and enforcement using AWS Organizations tag policies.
2. THE Template SHALL generate AWS Budgets using `budgets.create_budget()` with configurable monthly thresholds for ML services (SageMaker, Bedrock, Glue, S3) and notification actions at 50%, 80%, and 100% of budget.
3. THE Template SHALL generate Cost Explorer queries using `ce.get_cost_and_usage()` that break down ML spending by service, instance type, tag (experiment, team), and usage type.
4. THE Template SHALL generate Cost Anomaly Detection monitors using `ce.create_anomaly_monitor()` for ML service categories with configurable anomaly thresholds and SNS alert subscriptions.
5. THE Template SHALL include a per-experiment cost tracking utility that queries Cost Explorer by `ExperimentId` tag and generates a cost report per ML experiment.
6. WHEN a budget threshold is exceeded, THE Template SHALL include an AWS Budgets action that sends SNS notifications and optionally applies an IAM policy that restricts new resource creation.

---

### Requirement 20: Cost Management — Spot Instance Strategies for ML Training (finops/02)

**User Story:** As an ML engineer, I want a template for Spot instance strategies, so that I can reduce SageMaker training costs using managed spot training, checkpointing, and fallback patterns.

#### Acceptance Criteria

1. THE Template SHALL generate SageMaker managed spot training configuration using `sagemaker.create_training_job()` with `EnableManagedSpotTraining=True`, `MaxWaitTimeInSeconds`, and `MaxRuntimeInSeconds` parameters.
2. THE Template SHALL generate checkpointing configuration with S3 checkpoint paths that enable training job resumption after spot interruptions.
3. THE Template SHALL include a spot savings calculator that compares on-demand vs spot pricing for configurable instance types and training durations using `pricing.get_products()`.
4. THE Template SHALL include a fallback-to-on-demand pattern that detects repeated spot interruptions (via CloudWatch metrics) and automatically retries the training job with on-demand instances.
5. THE Template SHALL include spot instance diversification recommendations that suggest multiple instance types (e.g., `ml.p3.2xlarge`, `ml.p3.8xlarge`, `ml.g5.2xlarge`) to increase spot capacity availability.
6. WHEN a spot interruption occurs during training, THE Template SHALL include a CloudWatch alarm and SNS notification with the interruption details and checkpoint status.

---

### Requirement 21: Cost Management — Savings Plans for ML (finops/03)

**User Story:** As a FinOps practitioner, I want a template for Savings Plans analysis and planning for ML workloads, so that I can optimize long-term ML infrastructure costs with compute and ML savings plans.

#### Acceptance Criteria

1. THE Template SHALL generate Savings Plans utilization and coverage reports using `savingsplans.describe_savings_plans()` and `ce.get_savings_plans_utilization()` for current ML workload analysis.
2. THE Template SHALL generate Savings Plans purchase recommendations using `ce.get_savings_plans_purchase_recommendation()` filtered to SageMaker and compute usage types.
3. THE Template SHALL include a right-sizing analysis utility using `ce.get_rightsizing_recommendation()` that identifies over-provisioned SageMaker endpoints and training instances.
4. THE Template SHALL generate a cost projection model that compares on-demand, 1-year no-upfront, 1-year partial-upfront, and 3-year savings plan costs for a configurable ML workload profile.
5. THE Template SHALL include CloudWatch dashboards showing Savings Plans utilization percentage, coverage percentage, and net savings over time.
6. WHEN Savings Plans utilization drops below a configurable threshold (default 80%), THE Template SHALL include a CloudWatch alarm that notifies the FinOps team via SNS.

---

### Requirement 22: Cost Management — Inference Cost Optimization (finops/04)

**User Story:** As an ML platform engineer, I want a template for inference cost optimization, so that I can reduce serving costs using auto-scaling, scale-to-zero, specialized hardware, batch transform, and multi-model endpoints.

#### Acceptance Criteria

1. THE Template SHALL generate SageMaker endpoint auto-scaling policies using `application_autoscaling.register_scalable_target()` and `application_autoscaling.put_scaling_policy()` with target tracking on `InvocationsPerInstance` and step scaling on `ModelLatency`.
2. THE Template SHALL generate a scale-to-zero pattern for dev/stage endpoints using a scheduled scaling action that sets `MinCapacity=0` during off-hours and a CloudWatch alarm that scales up on incoming invocations.
3. THE Template SHALL include instance type comparison code that benchmarks inference latency and cost across `ml.inf2` (Inferentia2), `ml.g5` (GPU), and `ml.c7g` (Graviton) instance families for a given model.
4. THE Template SHALL generate a batch transform job configuration using `sagemaker.create_transform_job()` as a cost-effective alternative to real-time endpoints for high-latency-tolerant workloads.
5. THE Template SHALL generate a multi-model endpoint configuration using `sagemaker.create_model()` with `MultiModelConfig` that hosts multiple models on a single endpoint to reduce per-model infrastructure cost.
6. WHEN endpoint utilization drops below a configurable threshold for a sustained period, THE Template SHALL include a CloudWatch alarm that triggers a Lambda function to reduce instance count or switch to a smaller instance type.

---

### Requirement 23: Cost Management — FinOps Dashboards (finops/05)

**User Story:** As a FinOps practitioner, I want a template for ML cost dashboards, so that I can visualize ML spend, cost-per-inference metrics, budget alerts, and chargeback reports using CloudWatch and QuickSight.

#### Acceptance Criteria

1. THE Template SHALL generate CloudWatch dashboards with widgets for: total ML spend by service (SageMaker, Bedrock, Glue, S3), spend trend over 30/60/90 days, cost per endpoint, and budget utilization gauges.
2. THE Template SHALL generate QuickSight dataset and dashboard definitions using `quicksight.create_data_set()` and `quicksight.create_dashboard()` that visualize Cost Explorer data with drill-down by team, project, and experiment.
3. THE Template SHALL include a cost-per-inference metric computation using CloudWatch metric math that divides hourly endpoint cost by `Invocations` count to produce a `CostPerInvocation` custom metric.
4. THE Template SHALL generate chargeback report code that allocates ML costs to business units based on resource tags and outputs CSV reports suitable for finance team consumption.
5. THE Template SHALL include budget alert integration that embeds budget status widgets in the CloudWatch dashboard with color-coded thresholds (green/yellow/red).
6. WHEN a new ML endpoint or training job is created, THE Template SHALL include automatic dashboard widget creation using CloudWatch API calls that add the new resource to the appropriate dashboard.


---

### Requirement 24: Multi-Account / Enterprise — Organizations SCPs for ML (enterprise/01)

**User Story:** As a cloud architect, I want a template for AWS Organizations SCPs that enforce ML governance, so that I can deny unapproved instance types, enforce encryption, restrict regions, and apply ML-specific guardrails across all accounts.

#### Acceptance Criteria

1. THE Template SHALL generate SCPs using `organizations.create_policy()` that deny SageMaker training and endpoint creation with instance types not in a configurable allowlist using the `sagemaker:InstanceTypes` condition key.
2. THE Template SHALL generate SCPs that enforce KMS encryption on all SageMaker resources by denying `sagemaker:Create*` actions when `sagemaker:VolumeKmsKey` or `sagemaker:OutputKmsKey` condition keys are not present.
3. THE Template SHALL generate SCPs that restrict ML workloads to approved AWS regions using the `aws:RequestedRegion` condition key.
4. THE Template SHALL generate SCPs that deny Bedrock model access for unapproved model IDs using `bedrock:InvokeModel` with `bedrock:ModelId` condition key restrictions.
5. THE Template SHALL include SCP attachment configuration using `organizations.attach_policy()` targeting specific organizational units (OUs) for ML workload accounts.
6. IF an SCP blocks a legitimate ML operation, THEN THE Template SHALL include a documented exception process using SCP condition keys that allow specific IAM roles (break-glass roles) to bypass restrictions.
7. THE Code_Scaffolding_Hints_Section SHALL include complete SCP JSON policy documents with properly structured `Deny` statements and condition blocks.

---

### Requirement 25: Multi-Account / Enterprise — Cross-Account Model Deployment (enterprise/02)

**User Story:** As an MLOps engineer, I want a template for cross-account model deployment pipelines, so that I can promote models from dev to staging to production accounts using CodePipeline and SageMaker Model Registry.

#### Acceptance Criteria

1. THE Template SHALL generate a CodePipeline that spans three AWS accounts (dev, staging, prod) using cross-account IAM roles and a shared artifact S3 bucket with KMS encryption.
2. THE Template SHALL generate cross-account IAM roles in each target account using `iam.create_role()` with trust policies that allow the pipeline account to assume the deployment role.
3. THE Template SHALL generate SageMaker Model Registry cross-account access policies using `sagemaker.put_model_package_group_policy()` that allow staging and prod accounts to read model packages from the dev account registry.
4. THE Template SHALL include a model promotion workflow: register in dev → approve for staging → deploy to staging → approve for prod → deploy to prod, with manual approval gates at each promotion step.
5. WHEN a model package is approved in the SageMaker Model Registry, THE Template SHALL include an EventBridge rule that triggers the cross-account deployment pipeline automatically.
6. THE Template SHALL include rollback procedures that revert to the previous model version in the target account using SageMaker endpoint update with the previous model package ARN.

---

### Requirement 26: Multi-Account / Enterprise — Service Catalog for ML Environments (enterprise/03)

**User Story:** As an ML platform engineer, I want a template for Service Catalog ML products, so that data science teams can self-service provision approved ML environments including SageMaker domains, training environments, and inference endpoints.

#### Acceptance Criteria

1. THE Template SHALL generate a Service Catalog portfolio using `servicecatalog.create_portfolio()` containing ML infrastructure products with configurable access for specified IAM roles and groups.
2. THE Template SHALL generate at least three Service Catalog products: a SageMaker Domain product (using CloudFormation), a training environment product (S3 bucket + ECR repo + IAM role), and an inference endpoint product (endpoint config + auto-scaling).
3. THE Template SHALL include launch constraints using `servicecatalog.create_constraint()` that enforce the use of approved IAM roles for provisioning, preventing privilege escalation.
4. THE Template SHALL include tag update constraints that automatically apply mandatory tags (Project, Environment, CostCenter, Team) to all provisioned products.
5. WHEN a data scientist provisions an ML environment through Service Catalog, THE Template SHALL include a post-provisioning Lambda function that registers the environment in a central inventory DynamoDB table.
6. THE Template SHALL include product versioning using `servicecatalog.create_provisioning_artifact()` that allows platform teams to update ML environment templates without disrupting existing provisioned environments.

---

### Requirement 27: Multi-Account / Enterprise — Control Tower for ML Teams (enterprise/04)

**User Story:** As a cloud architect, I want a template for Control Tower landing zone customizations for ML teams, so that I can provision ML workload accounts with guardrails, detective controls, and ML-specific configurations.

#### Acceptance Criteria

1. THE Template SHALL generate Control Tower Account Factory customizations using Customizations for Control Tower (CfCT) that provision ML workload accounts with pre-configured VPCs, IAM roles, and S3 buckets.
2. THE Template SHALL generate detective guardrails using AWS Config rules that monitor ML resource compliance (encryption, VPC-only mode, approved instance types) in all ML workload accounts.
3. THE Template SHALL generate preventive guardrails using SCPs that enforce ML governance policies (region restrictions, instance type allowlists, encryption requirements) at the OU level.
4. THE Template SHALL include account baseline configurations that deploy: a SageMaker VPC endpoint, a Bedrock VPC endpoint, CloudTrail logging, and Config recorder in each new ML account.
5. WHEN a new ML workload account is created via Account Factory, THE Template SHALL include a lifecycle event hook that triggers a Step Functions workflow to deploy ML-specific infrastructure.
6. THE Template SHALL include a guardrail compliance dashboard using Config aggregator that shows compliance status across all ML workload accounts.

---

### Requirement 28: Multi-Account / Enterprise — Centralized Model Registry (enterprise/05)

**User Story:** As an MLOps engineer, I want a template for a centralized model registry with cross-account access, so that all ML teams can share models through a single SageMaker Model Registry with approval workflows and lineage tracking.

#### Acceptance Criteria

1. THE Template SHALL generate a centralized SageMaker Model Registry in a shared services account with model package groups organized by team and use case.
2. THE Template SHALL generate cross-account resource policies using `sagemaker.put_model_package_group_policy()` that grant read access to consumer accounts and write access to producer accounts.
3. THE Template SHALL generate a model approval workflow using EventBridge rules that notify approvers via SNS when a new model package is registered, and update the approval status via `sagemaker.update_model_package()`.
4. THE Template SHALL include model lineage tracking that records the training data S3 path, training job ARN, evaluation metrics, and Git commit hash as model package metadata using `sagemaker.create_model_package()` with `CustomerMetadataProperties`.
5. WHEN a model package is approved in the centralized registry, THE Template SHALL include an EventBridge rule that publishes a cross-account event to consumer accounts for automated deployment.
6. THE Template SHALL include a model catalog query utility using `sagemaker.list_model_packages()` with filters by approval status, creation date, and custom metadata that enables teams to discover available models.


---

### Requirement 29: Observability — OpenTelemetry for ML Inference Tracing (devops/10)

**User Story:** As a platform engineer, I want a template for OpenTelemetry distributed tracing across ML inference pipelines, so that I can trace requests from API Gateway through Lambda to SageMaker or Bedrock endpoints with custom spans for model latency.

#### Acceptance Criteria

1. THE Template SHALL generate an OpenTelemetry Collector configuration (ADOT — AWS Distro for OpenTelemetry) deployed as a Lambda layer or ECS sidecar that exports traces to AWS X-Ray.
2. THE Template SHALL generate instrumented Lambda function code using the `opentelemetry-api` and `opentelemetry-sdk` Python packages with custom spans for: request parsing, model invocation, response formatting, and guardrail evaluation.
3. THE Template SHALL include X-Ray trace group configuration using `xray.create_group()` with filter expressions that isolate ML inference traces from other application traces.
4. THE Template SHALL generate custom span attributes for ML-specific metadata: `model.id`, `model.latency_ms`, `model.input_tokens`, `model.output_tokens`, `model.endpoint_name`, and `guardrail.action`.
5. WHEN a trace shows model latency exceeding a configurable threshold, THE Template SHALL include an X-Ray Insights notification that alerts the operations team via SNS.
6. THE Template SHALL include a trace analysis utility using `xray.get_trace_summaries()` and `xray.batch_get_traces()` that computes p50, p95, and p99 latency distributions for ML inference spans.
7. THE Code_Scaffolding_Hints_Section SHALL include working OpenTelemetry instrumentation code for both `bedrock_runtime.invoke_model()` and `sagemaker_runtime.invoke_endpoint()` calls.

---

### Requirement 30: Observability — Custom CloudWatch Metrics for Model Quality (devops/11)

**User Story:** As an ML engineer, I want a template for custom CloudWatch metrics that track model quality, so that I can monitor token throughput, latency percentiles, error rates, and embedding quality scores with metric math and anomaly detection.

#### Acceptance Criteria

1. THE Template SHALL generate custom CloudWatch metric publishing code using `cloudwatch.put_metric_data()` for: tokens per second (input and output), model latency (p50, p95, p99 via statistic sets), invocation error rate, and embedding cosine similarity scores.
2. THE Template SHALL generate CloudWatch metric math expressions that compute derived metrics: cost per 1K tokens (using price constants and token count metrics), error rate percentage (errors / total invocations * 100), and throughput efficiency (tokens / latency).
3. THE Template SHALL generate CloudWatch Anomaly Detection models using `cloudwatch.put_anomaly_detector()` for model latency and token throughput metrics that automatically detect deviations from normal patterns.
4. THE Template SHALL include a metric publishing Lambda function that processes SageMaker endpoint CloudWatch Logs and extracts custom metrics from inference response payloads.
5. WHEN a custom metric breaches an anomaly detection band, THE Template SHALL include a CloudWatch alarm that triggers an SNS notification with the metric name, current value, and expected range.
6. THE Template SHALL include a CloudWatch dashboard definition with widgets for all custom model quality metrics, organized by endpoint and model version.

---

### Requirement 31: Observability — Bedrock Invocation Logging (devops/12)

**User Story:** As an AI operations engineer, I want a template for Bedrock model invocation logging, so that I can capture all Bedrock API calls to CloudWatch Logs and S3, analyze usage with Athena, and build cost-per-invocation dashboards.

#### Acceptance Criteria

1. THE Template SHALL generate Bedrock model invocation logging configuration using `bedrock.put_model_invocation_logging_configuration()` with CloudWatch Logs destination and S3 archival destination.
2. THE Template SHALL generate an Athena table definition and named queries for analyzing Bedrock invocation logs stored in S3: queries for total invocations by model, average latency by model, input/output token distribution, and error rate by model.
3. THE Template SHALL generate a cost-per-invocation tracking utility that joins Bedrock invocation logs with model pricing data to compute per-invocation and per-token costs.
4. THE Template SHALL include a CloudWatch Logs Insights query library with pre-built queries for: top models by invocation count, slowest invocations, highest token usage, guardrail intervention rate, and error analysis.
5. WHEN Bedrock invocation logs show error rates exceeding a configurable threshold for a specific model, THE Template SHALL include a CloudWatch alarm that notifies the operations team and optionally triggers a model failover Lambda.
6. THE Template SHALL include S3 lifecycle rules for invocation log archival: retain in Standard for 30 days, transition to Glacier Instant Retrieval after 90 days, and delete after 1 year (configurable per environment).

---

### Requirement 32: Observability — Cost-Per-Inference Dashboards (devops/13)

**User Story:** As a FinOps practitioner, I want a template for cost-per-inference dashboards, so that I can track per-model costs, per-customer cost allocation, and margin analysis using CloudWatch and QuickSight.

#### Acceptance Criteria

1. THE Template SHALL generate a CloudWatch dashboard with per-model cost-per-inference widgets using metric math that divides hourly infrastructure cost by invocation count for each SageMaker endpoint and Bedrock model.
2. THE Template SHALL generate a per-customer cost allocation system using a DynamoDB table that maps API keys or user IDs to invocation counts and computed costs, updated by a Lambda function processing API Gateway access logs.
3. THE Template SHALL generate QuickSight visualizations using `quicksight.create_analysis()` that show: cost trends by model, cost comparison across models for the same task, per-customer cost breakdown, and margin analysis (revenue per API call minus infrastructure cost).
4. THE Template SHALL include a cost alerting mechanism using CloudWatch alarms that trigger when cost-per-inference exceeds a configurable threshold, indicating potential model inefficiency or traffic anomalies.
5. WHEN a new model endpoint is deployed, THE Template SHALL include automatic registration in the cost tracking system with the model's pricing parameters (instance type, Bedrock model pricing tier).
6. THE Template SHALL include a monthly cost report generator that outputs a CSV with columns: model_id, total_invocations, total_cost, avg_cost_per_invocation, p99_latency, and customer_breakdown.

---

### Requirement 33: Observability — SageMaker Clarify Real-Time Bias Monitoring (devops/14)

**User Story:** As a responsible AI engineer, I want a template for SageMaker Clarify real-time bias monitoring, so that I can detect bias drift in production, provide online explainability, and trigger automated alerts and remediation workflows.

#### Acceptance Criteria

1. THE Template SHALL generate a SageMaker Clarify real-time explainability endpoint configuration using `sagemaker.create_endpoint_config()` with `ClarifyExplainerConfig` that provides feature attribution scores (SHAP values) for each inference request.
2. THE Template SHALL generate a bias drift monitoring schedule using `sagemaker.create_monitoring_schedule()` with `MonitoringType='ModelBiasMonitor'` that evaluates bias metrics (DPL — Difference in Positive Proportions in Labels, DI — Disparate Impact, KL — KL Divergence) on a configurable schedule.
3. THE Template SHALL include a bias baseline creation step using `sagemaker.create_model_bias_job_definition()` with a training dataset that establishes expected bias metric values.
4. WHEN bias metrics exceed configurable thresholds, THE Template SHALL include a CloudWatch alarm that triggers an SNS notification to the responsible AI team and publishes an event to EventBridge for automated remediation.
5. THE Template SHALL include a bias report visualization utility that parses Clarify JSON output and generates human-readable bias reports with metric explanations and recommended actions.
6. THE Template SHALL include integration with `mlops/11` (ML Governance) to record bias monitoring results as part of the model audit trail using SageMaker Model Cards.


---

### Requirement 34: Edge / Hybrid — SageMaker Edge Deployment (edge/01)

**User Story:** As an edge ML engineer, I want a template for SageMaker Edge Manager deployment, so that I can compile models with Neo, manage edge device fleets, package models for edge deployment, perform OTA updates, and monitor edge inference.

#### Acceptance Criteria

1. THE Template SHALL generate a SageMaker Neo compilation job using `sagemaker.create_compilation_job()` with configurable target platforms (Linux ARM64, Linux x86_64, Android, iOS) and framework (PyTorch, TensorFlow, ONNX).
2. THE Template SHALL generate an edge device fleet using `sagemaker.create_device_fleet()` with S3 output configuration for edge inference data capture and IoT role alias for device authentication.
3. THE Template SHALL generate edge packaging jobs using `sagemaker.create_edge_packaging_job()` that package compiled models with the Edge Manager agent for deployment to devices.
4. THE Template SHALL include an OTA model update workflow using IoT Jobs that pushes new model versions to registered edge devices and tracks deployment status per device.
5. WHEN an edge device reports inference results to S3, THE Template SHALL include a Lambda function triggered by S3 events that aggregates edge inference metrics and publishes them to CloudWatch.
6. THE Template SHALL include a device fleet monitoring dashboard using CloudWatch metrics for: active devices, model version distribution across fleet, inference latency per device, and heartbeat status.
7. THE Code_Scaffolding_Hints_Section SHALL include working `sagemaker.register_devices()` and `sagemaker.describe_device()` calls for fleet management.

---

### Requirement 35: Edge / Hybrid — IoT Greengrass ML Inference (edge/02)

**User Story:** As an IoT engineer, I want a template for IoT Greengrass ML inference, so that I can deploy ML models as Greengrass components, run local inference on edge devices, optimize models for constrained hardware, and upload results via stream manager.

#### Acceptance Criteria

1. THE Template SHALL generate a Greengrass V2 component recipe (YAML) and artifact for an ML inference component that loads a model from S3 and runs inference using a configurable ML framework (PyTorch, TensorFlow Lite, ONNX Runtime).
2. THE Template SHALL generate a Greengrass deployment configuration using `greengrassv2.create_deployment()` that targets a thing group and includes the ML inference component with configurable resource limits (memory, CPU).
3. THE Template SHALL include model optimization scripts for edge deployment: TensorFlow Lite conversion, ONNX quantization (INT8), and PyTorch mobile optimization using `torch.jit.trace()`.
4. THE Template SHALL generate a Greengrass Stream Manager configuration that buffers inference results locally and uploads them to S3 or Kinesis Data Streams when connectivity is available.
5. WHEN the edge device loses network connectivity, THE Template SHALL include a local inference fallback pattern that continues processing using the cached model and queues results for later upload.
6. THE Template SHALL include a component lifecycle management pattern with health check scripts that monitor inference component status and restart on failure.
7. THE Code_Scaffolding_Hints_Section SHALL include a working Greengrass component recipe YAML with artifact URIs, lifecycle scripts, and configuration parameters.

---

### Requirement 36: Edge / Hybrid — Outposts ML Patterns (edge/03)

**User Story:** As a hybrid cloud architect, I want a template for ML workloads on AWS Outposts, so that I can run SageMaker training and inference on-premises for data residency compliance while maintaining integration with cloud-based ML services.

#### Acceptance Criteria

1. THE Template SHALL generate SageMaker endpoint deployment configurations that target Outposts subnets using the `SubnetId` parameter pointing to Outposts-hosted subnets for local inference.
2. THE Template SHALL generate a hybrid training pattern where data preprocessing runs on Outposts (using ECS on Outposts or EC2 on Outposts) and model training runs in the AWS Region, with S3 on Outposts for local data staging.
3. THE Template SHALL include S3 on Outposts bucket configuration using `s3outposts.create_bucket()` for local training data storage with data residency compliance.
4. THE Template SHALL include a data synchronization pattern using S3 DataSync or S3 replication that selectively syncs model artifacts from the Region to Outposts and training data from Outposts to the Region.
5. WHEN the network connection between Outposts and the AWS Region is degraded, THE Template SHALL include a local inference continuity pattern that serves predictions from the locally deployed model without Region connectivity.
6. THE Template SHALL include monitoring for Outposts ML workloads using CloudWatch agent on Outposts instances that publishes local inference metrics to the Region CloudWatch.


---

### Requirement 37: Cross-Cutting Integration Points Consistency

**User Story:** As a template library maintainer, I want all new templates to have accurate and bidirectional integration points, so that users can chain templates correctly and understand dependencies across the expanded library.

#### Acceptance Criteria

1. WHEN a new template references an existing template as an upstream dependency, THE Integration_Points_Section of the existing template SHALL be updated to include the new template as a downstream consumer.
2. WHEN a new template references another new template, THE Integration_Points_Section SHALL include bidirectional references (upstream in the consumer, downstream in the producer).
3. THE Integration_Points_Section SHALL categorize each reference as one of: `Upstream` (this template consumes output from the referenced template), `Downstream` (this template produces output consumed by the referenced template), `Components` (this template uses resources defined in the referenced template), or `Alternative to` (this template provides an alternative approach to the referenced template).
4. THE Template_Library SHALL maintain referential integrity: every template reference in an Integration_Points_Section SHALL point to a template file that exists in the library.
5. THE Integration_Points_Section SHALL include a one-line description of the data or resource exchanged between templates (e.g., "IAM role ARNs from SSM parameters", "model package ARN for deployment").

---

### Requirement 38: New Directory Structure Creation

**User Story:** As a template library maintainer, I want the three new directories (data/, finops/, enterprise/, edge/) to be created with consistent structure, so that the expanded library maintains organizational coherence.

#### Acceptance Criteria

1. WHEN the expansion is implemented, THE Template_Library SHALL create four new directories: `data/` (5 templates), `finops/` (5 templates), `enterprise/` (5 templates), and `edge/` (3 templates).
2. THE Template_Library SHALL use sequential zero-padded numbering within each new directory starting from `01` (e.g., `data/01_glue_etl_ml_features.md`, `data/02_kinesis_realtime_features.md`).
3. THE Template_Library SHALL use the file naming convention `{number}_{descriptive_snake_case_name}.md` consistent with existing templates.
4. WHEN new templates are added to existing directories (mlops/, devops/), THE Template_Library SHALL continue the existing numbering sequence without gaps (mlops/14 through mlops/19, devops/05 through devops/14).
5. THE Template_Library SHALL maintain the existing directory naming convention of lowercase single-word or abbreviated directory names.

---

### Requirement 39: AWS API Validation and Version Compatibility

**User Story:** As a template library user, I want all code examples in new templates to use current AWS API signatures, so that generated code works without modification against current AWS services.

#### Acceptance Criteria

1. THE Code_Scaffolding_Hints_Section SHALL use boto3 1.35+ API signatures that match the current AWS service API documentation for all AWS service calls.
2. WHEN a template includes Terraform examples, THE Template SHALL use AWS Provider 5.70+ resource and data source syntax.
3. WHEN a template includes CDK examples, THE Template SHALL use CDK 2.170+ construct library syntax with Python bindings.
4. THE Template SHALL specify service-specific version requirements in the Template Version comment when a feature requires a minimum SDK version (e.g., `<!-- Template Version: 1.0 | boto3: 1.35+ | Bedrock Flows requires boto3 1.35.50+ -->`).
5. IF an AWS service feature is available only in specific regions, THEN THE Template SHALL document the region availability constraint in the Requirements & Constraints section.
6. THE Template SHALL avoid using deprecated AWS API calls and SHALL use the current recommended API patterns (e.g., `bedrock_runtime.converse()` instead of `bedrock_runtime.invoke_model()` for multi-turn conversations).
