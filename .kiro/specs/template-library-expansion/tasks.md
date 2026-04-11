# Tasks — Template Library Expansion

## AI/ML Gaps (mlops/14-19)

- [x] 1. Create `mlops/14_bedrock_agents_action_groups.md`
  - [x] 1.1 Write template version comment, title, purpose, and role definition for Bedrock Agents with Action Groups expertise
  - [x] 1.2 Write Context & Inputs section with parameters: ACTION_GROUPS (JSON list), AGENT_MODEL, AGENT_INSTRUCTION, KNOWLEDGE_BASE_ID, GUARDRAIL_ID, MULTI_AGENT_ENABLED, SESSION_TTL
  - [x] 1.3 Write Task section with directory tree and file-by-file instructions: agent creation, action group Lambda handlers, OpenAPI schema files, session persistence (DynamoDB), multi-agent supervisor pattern
  - [x] 1.4 Write Code Scaffolding Hints with working `bedrock_agent.create_agent()`, `create_agent_action_group()`, `bedrock_agent_runtime.invoke_agent()` calls including streaming response handling and trace logging
  - [x] 1.5 Write Integration Points referencing upstream `mlops/12` (Guardrails), `mlops/04` (RAG KB), `devops/04` (IAM) and downstream `mlops/16` (Flows), `devops/10` (tracing)
  - [x] 1.6 Write Requirements & Constraints and Output Format sections

- [x] 2. Create `mlops/15_multimodal_processing_pipelines.md`
  - [x] 2.1 Write template version comment, title, purpose, and role definition for multimodal AI processing expertise (vision, audio, document)
  - [x] 2.2 Write Context & Inputs section with parameters: MODALITIES (image/audio/document), PROCESSING_MODE (realtime/batch), S3_INPUT_PATH, S3_OUTPUT_PATH, INSTANCE_TYPE for SageMaker Processing
  - [x] 2.3 Write Task section with directory tree and file-by-file instructions: image pipeline (Bedrock Claude vision + Rekognition), audio pipeline (Transcribe + Bedrock summarization), document pipeline (Textract + Bedrock extraction), SageMaker Processing wrappers, S3 event triggers
  - [x] 2.4 Write Code Scaffolding Hints with working `bedrock_runtime.invoke_model()` (base64 image), `rekognition.detect_labels()`, `transcribe.start_transcription_job()`, `textract.start_document_analysis()` calls and chunking strategies for large files
  - [x] 2.5 Write Integration Points referencing upstream `devops/04` (IAM), `devops/02` (VPC) and downstream `data/01` (Glue for batch), `mlops/07` (Feature Store), `devops/10` (tracing)
  - [x] 2.6 Write Requirements & Constraints and Output Format sections

- [x] 3. Create `mlops/16_bedrock_flows_orchestration.md`
  - [x] 3.1 Write template version comment, title, purpose, and role definition for Bedrock Flows visual orchestration expertise
  - [x] 3.2 Write Context & Inputs section with parameters: FLOW_PATTERN (simple_chain/rag_augmented/conditional_branch), FLOW_NODES (JSON), LAMBDA_NODES, KB_ID
  - [x] 3.3 Write Task section with directory tree and file-by-file instructions: flow definition with input/prompt/KB/agent/Lambda/condition/output nodes, three flow patterns, Lambda node handlers, flow versioning and alias management, testing utility
  - [x] 3.4 Write Code Scaffolding Hints with working `bedrock_agent.create_flow()`, `create_flow_version()`, `create_flow_alias()`, `bedrock_agent_runtime.invoke_flow()` calls with node configuration examples
  - [x] 3.5 Write Integration Points referencing upstream `mlops/14` (Agents), `mlops/04` (RAG KB), `devops/04` (IAM) and downstream `devops/10` (tracing), `mlops/17` (evaluation)
  - [x] 3.6 Write Requirements & Constraints and Output Format sections

- [x] 4. Create `mlops/17_llm_evaluation_pipeline.md`
  - [x] 4.1 Write template version comment, title, purpose, and role definition for LLM evaluation and benchmarking expertise
  - [x] 4.2 Write Context & Inputs section with parameters: EVAL_TYPE (automated/human/ab_test/bedrock_eval), EVAL_DATASET_S3, METRICS (ROUGE/BLEU/BERTScore), MODEL_VARIANTS, THRESHOLD_CONFIG
  - [x] 4.3 Write Task section with directory tree and file-by-file instructions: automated metrics pipeline (ROUGE/BLEU/BERTScore), Bedrock Model Evaluation job, human-in-the-loop Step Functions workflow with SNS+API Gateway, A/B testing with SageMaker production variants, evaluation report aggregator, EventBridge gate for model promotion
  - [x] 4.4 Write Code Scaffolding Hints with working `bedrock.create_evaluation_job()`, ROUGE/BLEU/BERTScore Python library usage, Step Functions state machine definition, SageMaker production variant configuration
  - [x] 4.5 Write Integration Points referencing upstream `mlops/09` (Bedrock fine-tuning), `mlops/10` (Model Registry) and downstream `enterprise/05` (Central Registry), `mlops/13` (continuous training)
  - [x] 4.6 Write Requirements & Constraints and Output Format sections

- [x] 5. Create `mlops/18_prompt_caching_patterns.md`
  - [x] 5.1 Write template version comment, title, purpose, and role definition for LLM inference optimization and caching expertise
  - [x] 5.2 Write Context & Inputs section with parameters: CACHE_STRATEGY (bedrock_native/semantic/exact_match), ELASTICACHE_CLUSTER, DYNAMODB_TABLE, SIMILARITY_THRESHOLD, TTL_SECONDS, MODEL_ID
  - [x] 5.3 Write Task section with directory tree and file-by-file instructions: Bedrock Converse API caching with cachePoint markers, ElastiCache Redis semantic cache with embedding vectors, DynamoDB exact-match cache with TTL, cost reduction analysis, cache invalidation strategies (TTL/manual/model-version)
  - [x] 5.4 Write Code Scaffolding Hints with working `bedrock_runtime.converse()` with `cachePoint` content block, Redis vector similarity search, DynamoDB get/put with TTL, CloudWatch cost metrics
  - [x] 5.5 Write Integration Points referencing upstream `mlops/04` (RAG), `mlops/03` (inference endpoints), `devops/04` (IAM) and downstream `devops/11` (custom metrics), `finops/01` (cost tracking)
  - [x] 5.6 Write Requirements & Constraints and Output Format sections

- [x] 6. Create `mlops/19_bedrock_marketplace_models.md`
  - [x] 6.1 Write template version comment, title, purpose, and role definition for Bedrock Marketplace and provisioned throughput expertise
  - [x] 6.2 Write Context & Inputs section with parameters: PROVIDER_FILTER, MODEL_FILTER, MODEL_UNITS, COMMITMENT_TERM, INFERENCE_PROFILE_REGIONS, BENCHMARK_PROMPTS_S3
  - [x] 6.3 Write Task section with directory tree and file-by-file instructions: marketplace model discovery, provisioned throughput deployment, cross-region inference profile configuration, model comparison benchmarking utility, CloudWatch alarms for utilization, cleanup scripts
  - [x] 6.4 Write Code Scaffolding Hints with working `bedrock.list_foundation_models()`, `create_provisioned_model_throughput()`, `create_inference_profile()` calls and benchmark comparison logic
  - [x] 6.5 Write Integration Points referencing upstream `devops/04` (IAM), `mlops/12` (Guardrails) and downstream `mlops/17` (evaluation), `devops/12` (Bedrock logging), `finops/01` (cost tracking)
  - [x] 6.6 Write Requirements & Constraints and Output Format sections

## Data Engineering (data/01-05)

- [x] 7. Create `data/01_glue_etl_ml_features.md`
  - [x] 7.1 Write template version comment, title, purpose, and role definition for Glue ETL and ML feature engineering expertise
  - [x] 7.2 Write Context & Inputs section with parameters: GLUE_JOB_NAME, S3_SOURCE_PATH, S3_OUTPUT_PATH, TRANSFORM_TYPES (numerical/categorical/timestamp/text), DATA_QUALITY_RULES, CRAWLER_SCHEDULE, OUTPUT_FORMAT (parquet/feature_store)
  - [x] 7.3 Write Task section with directory tree and file-by-file instructions: PySpark ETL scripts with ML transforms, Glue Data Quality rulesets, Crawler configurations, job bookmark for incremental processing, Parquet and Feature Store output formats, SNS notification on quality failure
  - [x] 7.4 Write Code Scaffolding Hints with working PySpark `GlueContext`, `DynamicFrame`, `ResolveChoice` examples, `glue.create_crawler()`, `create_data_quality_ruleset()` calls
  - [x] 7.5 Write Integration Points referencing upstream `devops/04` (IAM), `data/03` (Lake Formation) and downstream `mlops/07` (Feature Store), `mlops/01` (training), `data/05` (EventBridge)
  - [x] 7.6 Write Requirements & Constraints and Output Format sections

- [-] 8. Create `data/02_kinesis_realtime_features.md`
  - [ ] 8.1 Write template version comment, title, purpose, and role definition for real-time streaming and feature computation expertise
  - [ ] 8.2 Write Context & Inputs section with parameters: STREAM_NAME, SHARD_COUNT, FEATURE_SCHEMA (JSON), WINDOW_SIZES, FIREHOSE_DESTINATION (s3/opensearch), DLQ_QUEUE_NAME
  - [ ] 8.3 Write Task section with directory tree and file-by-file instructions: Kinesis Data Stream + Lambda consumer for feature computation, Firehose delivery to S3 Parquet + OpenSearch, Kinesis Analytics Flink app for windowed aggregations, feature schema definition, DLQ routing, Feature Store online store integration
  - [ ] 8.4 Write Code Scaffolding Hints with working `kinesis.put_record()`, Lambda event source mapping, Firehose configuration, Flink SQL examples, `sagemaker_featurestore_runtime.put_record()` calls
  - [ ] 8.5 Write Integration Points referencing upstream `devops/04` (IAM), `devops/02` (VPC) and downstream `mlops/07` (Feature Store), `data/01` (Glue for batch backfill), `devops/10` (tracing)
  - [ ] 8.6 Write Requirements & Constraints and Output Format sections

- [-] 9. Create `data/03_lake_formation_ml_governance.md`
  - [ ] 9.1 Write template version comment, title, purpose, and role definition for data lake governance and Lake Formation expertise
  - [ ] 9.2 Write Context & Inputs section with parameters: DATABASE_NAME, TABLE_DEFINITIONS (JSON), LF_TAGS (JSON), CROSS_ACCOUNT_IDS, SAGEMAKER_ROLE_ARN, AUDIT_ENABLED
  - [ ] 9.3 Write Task section with directory tree and file-by-file instructions: Lake Formation resource registration, LF-TBAC tag-based access control, column-level and row-level permissions, cross-account data sharing, Glue Data Catalog table definitions with schema evolution, CloudTrail audit logging
  - [ ] 9.4 Write Code Scaffolding Hints with working `lakeformation.register_resource()`, `grant_permissions()`, `add_lf_tags_to_resource()`, `create_lf_tag()` calls
  - [ ] 9.5 Write Integration Points referencing upstream `devops/04` (IAM), `data/01` (Glue ETL) and downstream `mlops/01` (training data access), `enterprise/01` (SCPs), `devops/05` (Config compliance)
  - [ ] 9.6 Write Requirements & Constraints and Output Format sections

- [-] 10. Create `data/04_s3_data_lifecycle_ml.md`
  - [ ] 10.1 Write template version comment, title, purpose, and role definition for S3 storage optimization and data lifecycle expertise
  - [ ] 10.2 Write Context & Inputs section with parameters: BUCKET_NAMES (JSON), LIFECYCLE_RULES (JSON with age thresholds), INTELLIGENT_TIERING_CONFIG, BATCH_OPERATION_TYPE, ACCESS_POINT_CONFIGS
  - [ ] 10.3 Write Task section with directory tree and file-by-file instructions: S3 Intelligent-Tiering configuration, lifecycle rules (Standard→IT→Glacier IR→Deep Archive), S3 Batch Operations for bulk management, S3 Access Points per team/project, S3 Inventory configuration, restore-and-wait pattern with EventBridge
  - [ ] 10.4 Write Code Scaffolding Hints with working `s3control.create_job()`, `create_access_point()`, lifecycle rule JSON, Intelligent-Tiering configuration, restore request examples
  - [ ] 10.5 Write Integration Points referencing upstream `devops/04` (IAM), `devops/08` (KMS) and downstream `data/01` (Glue reads from S3), `mlops/01` (training data), `finops/01` (cost tracking)
  - [ ] 10.6 Write Requirements & Constraints and Output Format sections

- [ ] 11. Create `data/05_eventbridge_ml_orchestration.md`
  - [ ] 11.1 Write template version comment, title, purpose, and role definition for event-driven ML orchestration expertise
  - [ ] 11.2 Write Context & Inputs section with parameters: EVENT_BUS_NAME, SAGEMAKER_EVENTS (JSON list), CUSTOM_EVENT_SCHEMAS (JSON), STEP_FUNCTIONS_ARNS, CROSS_ACCOUNT_TARGETS
  - [ ] 11.3 Write Task section with directory tree and file-by-file instructions: EventBridge rules for SageMaker events (training/endpoint/model package/pipeline state changes), custom event bus with schema registry, Step Functions triggers for multi-step ML workflows, event pattern matching examples, drift→retrain workflow, cross-account event forwarding
  - [ ] 11.4 Write Code Scaffolding Hints with working `events.put_rule()`, `put_targets()`, `put_events()`, `schemas.create_registry()` calls with properly structured event detail payloads
  - [ ] 11.5 Write Integration Points referencing upstream `devops/04` (IAM), `mlops/05` (drift detection), `mlops/13` (continuous training) and downstream `mlops/10` (model registry events), `enterprise/05` (central registry events), `devops/05` (Config rule triggers)
  - [ ] 11.6 Write Requirements & Constraints and Output Format sections

## Security & Compliance (devops/05-09)

- [ ] 12. Create `devops/05_config_rules_ml_compliance.md`
  - [ ] 12.1 Write template version comment, title, purpose, and role definition for AWS Config and ML compliance expertise
  - [ ] 12.2 Write Context & Inputs section with parameters: MANAGED_RULES (list), CUSTOM_RULES (JSON with Lambda evaluators), APPROVED_INSTANCE_TYPES, CONFORMANCE_PACK_NAME, REMEDIATION_ENABLED
  - [ ] 12.3 Write Task section with directory tree and file-by-file instructions: managed Config rules for SageMaker (KMS, VPC, internet access), custom Lambda evaluators (instance type allowlist, VPC-only training, Bedrock model ID restrictions), conformance pack, SSM Automation remediation documents, EventBridge→SNS→Security Hub notification, compliance dashboard SQL queries
  - [ ] 12.4 Write Code Scaffolding Hints with working `config.put_config_rule()`, `put_conformance_pack()`, Lambda evaluator function pattern, SSM Automation document YAML, Config advanced query SQL
  - [ ] 12.5 Write Integration Points referencing upstream `devops/04` (IAM), `devops/08` (KMS) and downstream `devops/06` (Security Hub findings), `enterprise/01` (SCPs), `enterprise/04` (Control Tower guardrails)
  - [ ] 12.6 Write Requirements & Constraints and Output Format sections

- [ ] 13. Create `devops/06_guardduty_securityhub_ml.md`
  - [ ] 13.1 Write template version comment, title, purpose, and role definition for threat detection and security operations expertise for ML workloads
  - [ ] 13.2 Write Context & Inputs section with parameters: GUARDDUTY_FEATURES (s3_protection/lambda_protection), AUTOMATION_RULES (JSON), CUSTOM_INSIGHTS (JSON), REMEDIATION_ACTIONS, COMPLIANCE_FRAMEWORKS
  - [ ] 13.3 Write Task section with directory tree and file-by-file instructions: GuardDuty detector with S3+Lambda protection, Security Hub automation rules (auto-resolve low/escalate high), Lambda remediation workflow (isolate notebooks, revoke credentials, quarantine S3), custom insights for ML resources, EventBridge rule for anomalous SageMaker/Bedrock API calls, compliance framework mapping
  - [ ] 13.4 Write Code Scaffolding Hints with working `guardduty.create_detector()`, `securityhub.create_automation_rule()`, `create_insight()`, Lambda remediation function patterns
  - [ ] 13.5 Write Integration Points referencing upstream `devops/04` (IAM), `devops/05` (Config findings) and downstream `enterprise/04` (Control Tower), `devops/03` (CloudWatch dashboards)
  - [ ] 13.6 Write Requirements & Constraints and Output Format sections

- [ ] 14. Create `devops/07_macie_pii_training_data.md`
  - [ ] 14.1 Write template version comment, title, purpose, and role definition for data privacy and PII detection expertise
  - [ ] 14.2 Write Context & Inputs section with parameters: TARGET_BUCKETS (list), SCAN_SCHEDULE (one-time/recurring), CUSTOM_DATA_IDENTIFIERS (JSON), REMEDIATION_ACTION (quarantine/sanitize/notify), BLOCK_TRAINING_ON_PII
  - [ ] 14.3 Write Task section with directory tree and file-by-file instructions: Macie classification jobs, custom data identifiers for domain-specific patterns, EventBridge+Step Functions remediation workflow (quarantine→notify→sanitize), data sanitization Lambda for JSONL/CSV redaction, training job blocking mechanism (S3 bucket policy or Lake Formation revocation), findings aggregation queries
  - [ ] 14.4 Write Code Scaffolding Hints with working `macie2.create_classification_job()`, `create_custom_data_identifier()`, `get_finding_statistics()`, S3 bucket policy update patterns, Lambda sanitization function
  - [ ] 14.5 Write Integration Points referencing upstream `devops/04` (IAM), `data/03` (Lake Formation) and downstream `data/01` (Glue sanitized data), `devops/06` (Security Hub findings), `enterprise/01` (SCPs)
  - [ ] 14.6 Write Requirements & Constraints and Output Format sections

- [ ] 15. Create `devops/08_kms_encryption_ml.md`
  - [ ] 15.1 Write template version comment, title, purpose, and role definition for KMS encryption and key management expertise for ML workloads
  - [ ] 15.2 Write Context & Inputs section with parameters: KEY_PURPOSES (training_data/model_artifacts/endpoint_volumes/bedrock_models), KEY_ROTATION_PERIOD, CROSS_ACCOUNT_IDS, KEY_ALIAS_PREFIX
  - [ ] 15.3 Write Task section with directory tree and file-by-file instructions: separate KMS keys per purpose, key policies with `kms:ViaService` conditions, automatic key rotation, cross-account key sharing grants, key alias management (`alias/{PROJECT_NAME}/{ENV}/{purpose}`), CloudWatch alarm for `ScheduleKeyDeletion`, examples of passing KMS ARNs to SageMaker and Bedrock API calls
  - [ ] 15.4 Write Code Scaffolding Hints with working `kms.create_key()`, `enable_key_rotation()`, `create_alias()`, `create_grant()` calls and key policy JSON with `kms:ViaService` conditions, plus SageMaker/Bedrock encryption parameter examples
  - [ ] 15.5 Write Integration Points referencing upstream `devops/04` (IAM) and downstream `devops/05` (Config encryption rules), `enterprise/02` (cross-account deployment), `data/03` (Lake Formation), `data/04` (S3 lifecycle)
  - [ ] 15.6 Write Requirements & Constraints and Output Format sections

- [ ] 16. Create `devops/09_vpc_endpoint_policies_ml.md`
  - [ ] 16.1 Write template version comment, title, purpose, and role definition for VPC endpoint security and network policy expertise
  - [ ] 16.2 Write Context & Inputs section with parameters: ML_ENDPOINTS (list of service names), ENDPOINT_POLICIES (JSON), ALLOWED_BUCKETS (list), ALLOWED_SECURITY_GROUPS, SUBNET_IDS
  - [ ] 16.3 Write Task section with directory tree and file-by-file instructions: VPC interface endpoints for SageMaker API/runtime, Bedrock/Bedrock-runtime/Bedrock-agent-runtime, S3 gateway endpoint with bucket-scoped policies, endpoint policies with resource tag conditions, security group configurations, parameterized endpoint creation pattern, policy testing utility
  - [ ] 16.4 Write Code Scaffolding Hints with working `ec2.create_vpc_endpoint()` calls for each ML service, endpoint policy JSON documents with `aws:ResourceAccount` and `sagemaker:ResourceTag` conditions, security group rules
  - [ ] 16.5 Write Integration Points referencing upstream `devops/02` (VPC), `devops/04` (IAM) and downstream `devops/05` (Config compliance), `enterprise/04` (Control Tower baselines)
  - [ ] 16.6 Write Requirements & Constraints and Output Format sections

## Cost Management / FinOps (finops/01-05)

- [ ] 17. Create `finops/01_cost_allocation_ml.md`
  - [ ] 17.1 Write template version comment, title, purpose, and role definition for FinOps and ML cost management expertise
  - [ ] 17.2 Write Context & Inputs section with parameters: MANDATORY_TAGS (JSON), BUDGET_THRESHOLDS (JSON with service-level monthly limits), ANOMALY_THRESHOLD, COST_CENTER_MAPPING, NOTIFICATION_EMAIL
  - [ ] 17.3 Write Task section with directory tree and file-by-file instructions: tag policy enforcement via Organizations, AWS Budgets per ML service with 50/80/100% notifications, Cost Explorer queries by service/instance type/tag, Cost Anomaly Detection monitors, per-experiment cost tracking utility, budget breach IAM restriction action
  - [ ] 17.4 Write Code Scaffolding Hints with working `budgets.create_budget()`, `ce.get_cost_and_usage()`, `ce.create_anomaly_monitor()`, `ce.create_anomaly_subscription()`, `organizations.create_policy()` (tag policy) calls
  - [ ] 17.5 Write Integration Points referencing upstream `devops/04` (IAM) and downstream `finops/05` (dashboards), `finops/03` (Savings Plans), `devops/12` (Bedrock logging costs)
  - [ ] 17.6 Write Requirements & Constraints and Output Format sections

- [ ] 18. Create `finops/02_spot_instance_strategies_ml.md`
  - [ ] 18.1 Write template version comment, title, purpose, and role definition for SageMaker spot training and cost optimization expertise
  - [ ] 18.2 Write Context & Inputs section with parameters: TRAINING_INSTANCE_TYPES (list for diversification), CHECKPOINT_S3_PATH, MAX_WAIT_TIME, MAX_RUNTIME, FALLBACK_TO_ON_DEMAND, SPOT_SAVINGS_COMPARISON
  - [ ] 18.3 Write Task section with directory tree and file-by-file instructions: SageMaker managed spot training configuration, S3 checkpointing for interruption recovery, spot savings calculator using Pricing API, fallback-to-on-demand pattern with CloudWatch interruption detection, instance type diversification recommendations, CloudWatch alarm + SNS for spot interruptions
  - [ ] 18.4 Write Code Scaffolding Hints with working `sagemaker.create_training_job()` with `EnableManagedSpotTraining`, checkpoint config, `pricing.get_products()` for spot vs on-demand comparison, CloudWatch metric filter for interruptions
  - [ ] 18.5 Write Integration Points referencing upstream `devops/04` (IAM), `mlops/01` (training pipeline) and downstream `finops/01` (cost tracking), `finops/05` (dashboards)
  - [ ] 18.6 Write Requirements & Constraints and Output Format sections

- [ ] 19. Create `finops/03_savings_plans_ml.md`
  - [ ] 19.1 Write template version comment, title, purpose, and role definition for Savings Plans analysis and ML infrastructure cost planning expertise
  - [ ] 19.2 Write Context & Inputs section with parameters: ANALYSIS_PERIOD (30/60/90 days), PLAN_TYPES (compute/ml), UTILIZATION_THRESHOLD, WORKLOAD_PROFILE (JSON with instance types and hours)
  - [ ] 19.3 Write Task section with directory tree and file-by-file instructions: Savings Plans utilization and coverage reports, purchase recommendations filtered to SageMaker/compute, right-sizing analysis for endpoints and training instances, cost projection model (on-demand vs 1yr/3yr plans), CloudWatch dashboards for utilization/coverage/savings, utilization threshold alarm
  - [ ] 19.4 Write Code Scaffolding Hints with working `savingsplans.describe_savings_plans()`, `ce.get_savings_plans_utilization()`, `ce.get_savings_plans_purchase_recommendation()`, `ce.get_rightsizing_recommendation()` calls
  - [ ] 19.5 Write Integration Points referencing upstream `finops/01` (cost data) and downstream `finops/05` (dashboards), `enterprise/01` (SCPs for governance)
  - [ ] 19.6 Write Requirements & Constraints and Output Format sections

- [ ] 20. Create `finops/04_inference_cost_optimization.md`
  - [ ] 20.1 Write template version comment, title, purpose, and role definition for ML inference cost optimization and auto-scaling expertise
  - [ ] 20.2 Write Context & Inputs section with parameters: ENDPOINT_NAME, SCALING_METRIC (InvocationsPerInstance/ModelLatency), MIN_CAPACITY, MAX_CAPACITY, SCALE_TO_ZERO_SCHEDULE, INSTANCE_FAMILIES (inf2/g5/c7g), BATCH_TRANSFORM_ENABLED, MULTI_MODEL_ENABLED
  - [ ] 20.3 Write Task section with directory tree and file-by-file instructions: auto-scaling policies (target tracking + step scaling), scale-to-zero pattern with scheduled actions + CloudWatch alarm scale-up, instance type benchmarking across Inferentia2/GPU/Graviton, batch transform job configuration, multi-model endpoint setup, utilization-based downsizing Lambda
  - [ ] 20.4 Write Code Scaffolding Hints with working `application_autoscaling.register_scalable_target()`, `put_scaling_policy()`, `sagemaker.create_transform_job()`, `create_model()` with `MultiModelConfig`, scheduled scaling actions
  - [ ] 20.5 Write Integration Points referencing upstream `devops/04` (IAM), `mlops/03` (inference endpoints) and downstream `devops/13` (cost-per-inference dashboards), `finops/01` (cost tracking), `finops/05` (dashboards)
  - [ ] 20.6 Write Requirements & Constraints and Output Format sections

- [ ] 21. Create `finops/05_finops_dashboards_ml.md`
  - [ ] 21.1 Write template version comment, title, purpose, and role definition for FinOps visualization and ML cost reporting expertise
  - [ ] 21.2 Write Context & Inputs section with parameters: DASHBOARD_NAME, SERVICES_TO_TRACK (list), QUICKSIGHT_ENABLED, CHARGEBACK_ENABLED, BUDGET_INTEGRATION, REPORT_FREQUENCY
  - [ ] 21.3 Write Task section with directory tree and file-by-file instructions: CloudWatch dashboard with ML spend widgets (by service, trend, per-endpoint, budget gauges), QuickSight dataset + dashboard for Cost Explorer drill-down, cost-per-inference metric math, chargeback report generator (CSV by business unit), budget status widget integration, automatic widget creation for new endpoints/jobs
  - [ ] 21.4 Write Code Scaffolding Hints with working `cloudwatch.put_dashboard()` with metric math widgets, `quicksight.create_data_set()`, `create_dashboard()`, `create_analysis()` calls, CSV report generation code
  - [ ] 21.5 Write Integration Points referencing upstream `finops/01` (cost data), `finops/04` (inference costs), `devops/12` (Bedrock logging), `devops/13` (cost-per-inference) and downstream `enterprise/01` (governance reporting)
  - [ ] 21.6 Write Requirements & Constraints and Output Format sections

## Multi-Account / Enterprise (enterprise/01-05)

- [ ] 22. Create `enterprise/01_organizations_scps_ml.md`
  - [ ] 22.1 Write template version comment, title, purpose, and role definition for AWS Organizations and ML governance policy expertise
  - [ ] 22.2 Write Context & Inputs section with parameters: APPROVED_INSTANCE_TYPES (list), APPROVED_REGIONS (list), APPROVED_BEDROCK_MODELS (list), TARGET_OUS (list), BREAK_GLASS_ROLE_ARNS, ENCRYPTION_REQUIRED
  - [ ] 22.3 Write Task section with directory tree and file-by-file instructions: SCPs denying unapproved SageMaker instance types (`sagemaker:InstanceTypes` condition), SCPs enforcing KMS encryption (`sagemaker:VolumeKmsKey`/`OutputKmsKey`), region restriction SCPs, Bedrock model ID restriction SCPs, SCP attachment to OUs, break-glass exception process with condition keys
  - [ ] 22.4 Write Code Scaffolding Hints with working `organizations.create_policy()`, `attach_policy()` calls and complete SCP JSON policy documents with `Deny` statements and condition blocks
  - [ ] 22.5 Write Integration Points referencing upstream `devops/04` (IAM) and downstream `enterprise/04` (Control Tower), `devops/05` (Config rules), `devops/08` (KMS)
  - [ ] 22.6 Write Requirements & Constraints and Output Format sections

- [ ] 23. Create `enterprise/02_cross_account_model_deployment.md`
  - [ ] 23.1 Write template version comment, title, purpose, and role definition for cross-account ML deployment pipeline expertise
  - [ ] 23.2 Write Context & Inputs section with parameters: DEV_ACCOUNT_ID, STAGING_ACCOUNT_ID, PROD_ACCOUNT_ID, PIPELINE_ACCOUNT_ID, SHARED_ARTIFACT_BUCKET, KMS_KEY_ARN, MODEL_PACKAGE_GROUP
  - [ ] 23.3 Write Task section with directory tree and file-by-file instructions: CodePipeline spanning 3 accounts with cross-account IAM roles, shared S3 artifact bucket with KMS, SageMaker Model Registry cross-account policies, model promotion workflow (register→approve→deploy per stage), EventBridge trigger on model approval, rollback procedures
  - [ ] 23.4 Write Code Scaffolding Hints with working cross-account IAM trust policies, `sagemaker.put_model_package_group_policy()`, CodePipeline cross-account action configuration, EventBridge cross-account rule patterns
  - [ ] 23.5 Write Integration Points referencing upstream `devops/04` (IAM), `devops/08` (KMS), `mlops/10` (Model Registry) and downstream `enterprise/05` (Central Registry), `cicd/03` (CodePipeline)
  - [ ] 23.6 Write Requirements & Constraints and Output Format sections

- [ ] 24. Create `enterprise/03_service_catalog_ml.md`
  - [ ] 24.1 Write template version comment, title, purpose, and role definition for Service Catalog and self-service ML platform expertise
  - [ ] 24.2 Write Context & Inputs section with parameters: PORTFOLIO_NAME, PRODUCTS (JSON list: sagemaker_domain/training_env/inference_endpoint), ALLOWED_ROLES, MANDATORY_TAGS, INVENTORY_TABLE_NAME
  - [ ] 24.3 Write Task section with directory tree and file-by-file instructions: Service Catalog portfolio, three CloudFormation-backed products (SageMaker Domain, training environment, inference endpoint), launch constraints with approved IAM roles, tag update constraints for mandatory tags, post-provisioning Lambda for DynamoDB inventory registration, product versioning
  - [ ] 24.4 Write Code Scaffolding Hints with working `servicecatalog.create_portfolio()`, `create_product()`, `create_constraint()`, `create_provisioning_artifact()`, `associate_principal_with_portfolio()` calls and CloudFormation product templates
  - [ ] 24.5 Write Integration Points referencing upstream `devops/04` (IAM), `enterprise/01` (SCPs) and downstream `enterprise/04` (Control Tower), `finops/01` (cost tracking via tags)
  - [ ] 24.6 Write Requirements & Constraints and Output Format sections

- [ ] 25. Create `enterprise/04_control_tower_ml.md`
  - [ ] 25.1 Write template version comment, title, purpose, and role definition for Control Tower landing zone and ML account governance expertise
  - [ ] 25.2 Write Context & Inputs section with parameters: ML_OU_NAME, ACCOUNT_BASELINE_CONFIG (JSON), DETECTIVE_GUARDRAILS (list), PREVENTIVE_GUARDRAILS (list), ACCOUNT_FACTORY_TEMPLATE
  - [ ] 25.3 Write Task section with directory tree and file-by-file instructions: CfCT customizations for ML account provisioning (VPC, IAM, S3), detective guardrails via Config rules (encryption, VPC-only, instance types), preventive guardrails via SCPs, account baseline deployment (SageMaker VPC endpoint, Bedrock VPC endpoint, CloudTrail, Config recorder), lifecycle event hook for Step Functions ML infrastructure deployment, Config aggregator compliance dashboard
  - [ ] 25.4 Write Code Scaffolding Hints with working CfCT manifest YAML, Config rule definitions, SCP policy documents, Step Functions state machine for account baseline, Config aggregator queries
  - [ ] 25.5 Write Integration Points referencing upstream `enterprise/01` (SCPs), `devops/05` (Config rules), `devops/09` (VPC endpoints) and downstream `enterprise/03` (Service Catalog), `enterprise/02` (cross-account deployment)
  - [ ] 25.6 Write Requirements & Constraints and Output Format sections

- [ ] 26. Create `enterprise/05_centralized_model_registry.md`
  - [ ] 26.1 Write template version comment, title, purpose, and role definition for centralized model registry and cross-account model sharing expertise
  - [ ] 26.2 Write Context & Inputs section with parameters: REGISTRY_ACCOUNT_ID, PRODUCER_ACCOUNTS (list), CONSUMER_ACCOUNTS (list), MODEL_PACKAGE_GROUPS (JSON), APPROVAL_WORKFLOW_ENABLED, LINEAGE_TRACKING
  - [ ] 26.3 Write Task section with directory tree and file-by-file instructions: centralized SageMaker Model Registry in shared services account, cross-account resource policies (read for consumers, write for producers), EventBridge approval notification workflow, model lineage metadata (training data S3, job ARN, eval metrics, Git commit), cross-account event publishing on approval, model catalog query utility
  - [ ] 26.4 Write Code Scaffolding Hints with working `sagemaker.put_model_package_group_policy()`, `create_model_package()` with `CustomerMetadataProperties`, `update_model_package()`, `list_model_packages()` with filters, EventBridge cross-account event patterns
  - [ ] 26.5 Write Integration Points referencing upstream `mlops/10` (Model Registry), `mlops/17` (evaluation metrics), `devops/04` (IAM) and downstream `enterprise/02` (cross-account deployment), `mlops/03` (inference from approved models)
  - [ ] 26.6 Write Requirements & Constraints and Output Format sections

## Observability (devops/10-14)

- [ ] 27. Create `devops/10_opentelemetry_ml_tracing.md`
  - [ ] 27.1 Write template version comment, title, purpose, and role definition for OpenTelemetry distributed tracing and ML observability expertise
  - [ ] 27.2 Write Context & Inputs section with parameters: DEPLOYMENT_MODE (lambda_layer/ecs_sidecar), COLLECTOR_CONFIG, TRACE_GROUP_NAME, CUSTOM_SPAN_ATTRIBUTES (JSON), LATENCY_THRESHOLD_MS, EXPORT_DESTINATION (xray)
  - [ ] 27.3 Write Task section with directory tree and file-by-file instructions: ADOT Collector configuration (Lambda layer or ECS sidecar), instrumented Lambda function with custom spans (request parsing, model invocation, response formatting, guardrail evaluation), X-Ray trace group with ML filter expressions, custom span attributes for ML metadata (model.id, latency_ms, tokens), X-Ray Insights notification for latency threshold, trace analysis utility (p50/p95/p99)
  - [ ] 27.4 Write Code Scaffolding Hints with working OpenTelemetry instrumentation code for `bedrock_runtime.invoke_model()` and `sagemaker_runtime.invoke_endpoint()`, ADOT collector YAML config, `xray.create_group()`, `xray.get_trace_summaries()` calls
  - [ ] 27.5 Write Integration Points referencing upstream `devops/04` (IAM), `mlops/03` (inference endpoints), `mlops/14` (Bedrock Agents) and downstream `devops/13` (cost-per-inference), `devops/11` (custom metrics)
  - [ ] 27.6 Write Requirements & Constraints and Output Format sections

- [ ] 28. Create `devops/11_custom_cloudwatch_model_quality.md`
  - [ ] 28.1 Write template version comment, title, purpose, and role definition for CloudWatch custom metrics and ML model quality monitoring expertise
  - [ ] 28.2 Write Context & Inputs section with parameters: ENDPOINT_NAMES (list), METRIC_NAMESPACE, METRICS_TO_PUBLISH (tokens_per_second/latency_percentiles/error_rate/embedding_similarity), ANOMALY_DETECTION_ENABLED, DASHBOARD_NAME
  - [ ] 28.3 Write Task section with directory tree and file-by-file instructions: custom metric publishing code (tokens/sec, latency p50/p95/p99, error rate, embedding cosine similarity), metric math expressions (cost per 1K tokens, error rate %, throughput efficiency), anomaly detection models, metric publishing Lambda from CloudWatch Logs, anomaly breach alarm + SNS, dashboard with all custom metrics by endpoint and model version
  - [ ] 28.4 Write Code Scaffolding Hints with working `cloudwatch.put_metric_data()` with statistic sets, `put_anomaly_detector()`, metric math expression syntax, Lambda log parsing patterns
  - [ ] 28.5 Write Integration Points referencing upstream `devops/04` (IAM), `mlops/03` (inference endpoints), `devops/03` (CloudWatch base) and downstream `devops/13` (cost-per-inference), `finops/05` (dashboards), `devops/14` (Clarify bias)
  - [ ] 28.6 Write Requirements & Constraints and Output Format sections

- [ ] 29. Create `devops/12_bedrock_invocation_logging.md`
  - [ ] 29.1 Write template version comment, title, purpose, and role definition for Bedrock observability and invocation log analysis expertise
  - [ ] 29.2 Write Context & Inputs section with parameters: LOG_DESTINATIONS (cloudwatch_logs/s3), S3_LOG_BUCKET, ATHENA_DATABASE, LOG_RETENTION_DAYS, ALERT_ERROR_RATE_THRESHOLD, FAILOVER_ENABLED
  - [ ] 29.3 Write Task section with directory tree and file-by-file instructions: Bedrock invocation logging configuration (CloudWatch Logs + S3), Athena table DDL and named queries (invocations by model, avg latency, token distribution, error rate), cost-per-invocation tracking joining logs with pricing, CloudWatch Logs Insights query library, error rate alarm with optional model failover Lambda, S3 lifecycle rules for log archival
  - [ ] 29.4 Write Code Scaffolding Hints with working `bedrock.put_model_invocation_logging_configuration()`, Athena CREATE TABLE DDL for Bedrock log schema, CloudWatch Logs Insights query syntax, S3 lifecycle rule JSON
  - [ ] 29.5 Write Integration Points referencing upstream `devops/04` (IAM), `mlops/12` (Guardrails logging) and downstream `devops/13` (cost-per-inference), `finops/01` (cost data), `finops/05` (dashboards)
  - [ ] 29.6 Write Requirements & Constraints and Output Format sections

- [ ] 30. Create `devops/13_cost_per_inference_dashboards.md`
  - [ ] 30.1 Write template version comment, title, purpose, and role definition for inference cost analytics and per-customer cost allocation expertise
  - [ ] 30.2 Write Context & Inputs section with parameters: ENDPOINTS_TO_TRACK (list), BEDROCK_MODELS_TO_TRACK (list), CUSTOMER_TRACKING_ENABLED, QUICKSIGHT_ENABLED, COST_ALERT_THRESHOLD, REPORT_FREQUENCY
  - [ ] 30.3 Write Task section with directory tree and file-by-file instructions: CloudWatch dashboard with per-model cost-per-inference metric math, DynamoDB per-customer cost allocation table + Lambda processor, QuickSight analysis (cost trends, model comparison, per-customer breakdown, margin analysis), cost alerting alarms, automatic registration for new endpoints, monthly CSV cost report generator
  - [ ] 30.4 Write Code Scaffolding Hints with working CloudWatch metric math (cost/invocations), DynamoDB table schema and Lambda update function, `quicksight.create_analysis()`, CSV report generation code
  - [ ] 30.5 Write Integration Points referencing upstream `devops/11` (custom metrics), `devops/12` (Bedrock logging), `finops/04` (inference cost optimization) and downstream `finops/05` (FinOps dashboards), `finops/01` (cost allocation)
  - [ ] 30.6 Write Requirements & Constraints and Output Format sections

- [ ] 31. Create `devops/14_clarify_realtime_bias_monitoring.md`
  - [ ] 31.1 Write template version comment, title, purpose, and role definition for SageMaker Clarify, responsible AI, and bias detection expertise
  - [ ] 31.2 Write Context & Inputs section with parameters: ENDPOINT_NAME, BIAS_METRICS (DPL/DI/KL), MONITORING_SCHEDULE, BASELINE_DATASET_S3, BIAS_THRESHOLDS (JSON), EXPLAINABILITY_ENABLED
  - [ ] 31.3 Write Task section with directory tree and file-by-file instructions: Clarify real-time explainability endpoint config with `ClarifyExplainerConfig` (SHAP values), bias drift monitoring schedule with `ModelBiasMonitor`, bias baseline creation with training dataset, CloudWatch alarm + SNS + EventBridge for threshold breaches, bias report visualization utility, integration with mlops/11 for Model Cards audit trail
  - [ ] 31.4 Write Code Scaffolding Hints with working `sagemaker.create_endpoint_config()` with `ClarifyExplainerConfig`, `create_model_bias_job_definition()`, `create_monitoring_schedule()` with `MonitoringType='ModelBiasMonitor'`, Clarify JSON output parsing
  - [ ] 31.5 Write Integration Points referencing upstream `devops/04` (IAM), `mlops/03` (inference endpoints), `mlops/05` (model monitoring) and downstream `mlops/11` (governance audit trail), `devops/03` (CloudWatch dashboards)
  - [ ] 31.6 Write Requirements & Constraints and Output Format sections

## Edge / Hybrid (edge/01-03)

- [ ] 32. Create `edge/01_sagemaker_edge_deployment.md`
  - [ ] 32.1 Write template version comment, title, purpose, and role definition for SageMaker Edge Manager and edge ML deployment expertise
  - [ ] 32.2 Write Context & Inputs section with parameters: TARGET_PLATFORM (linux_arm64/linux_x86_64/android/ios), FRAMEWORK (pytorch/tensorflow/onnx), DEVICE_FLEET_NAME, S3_OUTPUT_PATH, IOT_ROLE_ALIAS, OTA_ENABLED
  - [ ] 32.3 Write Task section with directory tree and file-by-file instructions: SageMaker Neo compilation job for target platform, device fleet creation with S3 data capture and IoT role alias, edge packaging job, OTA model update workflow via IoT Jobs, Lambda for S3 edge inference metric aggregation, CloudWatch fleet monitoring dashboard (active devices, model version distribution, latency, heartbeat)
  - [ ] 32.4 Write Code Scaffolding Hints with working `sagemaker.create_compilation_job()`, `create_device_fleet()`, `create_edge_packaging_job()`, `register_devices()`, `describe_device()` calls and IoT Jobs configuration
  - [ ] 32.5 Write Integration Points referencing upstream `devops/04` (IAM), `mlops/01` (trained model) and downstream `edge/02` (IoT Greengrass), `devops/03` (CloudWatch dashboards), `devops/11` (custom metrics)
  - [ ] 32.6 Write Requirements & Constraints and Output Format sections

- [ ] 33. Create `edge/02_iot_greengrass_ml_inference.md`
  - [ ] 33.1 Write template version comment, title, purpose, and role definition for IoT Greengrass V2 and edge ML inference expertise
  - [ ] 33.2 Write Context & Inputs section with parameters: COMPONENT_NAME, ML_FRAMEWORK (pytorch/tflite/onnx), MODEL_S3_URI, THING_GROUP_NAME, RESOURCE_LIMITS (memory/cpu), STREAM_MANAGER_DESTINATION (s3/kinesis), OPTIMIZATION_TYPE (tflite/onnx_int8/torch_mobile)
  - [ ] 33.3 Write Task section with directory tree and file-by-file instructions: Greengrass V2 component recipe (YAML) with ML inference artifact, deployment configuration targeting thing group with resource limits, model optimization scripts (TFLite conversion, ONNX INT8 quantization, PyTorch mobile), Stream Manager configuration for buffered upload to S3/Kinesis, offline inference fallback pattern, component lifecycle health check scripts
  - [ ] 33.4 Write Code Scaffolding Hints with working Greengrass component recipe YAML (artifact URIs, lifecycle scripts, configuration), `greengrassv2.create_deployment()`, Stream Manager SDK usage, model optimization code snippets
  - [ ] 33.5 Write Integration Points referencing upstream `edge/01` (compiled models), `devops/04` (IAM) and downstream `data/02` (Kinesis for streaming results), `devops/03` (CloudWatch dashboards)
  - [ ] 33.6 Write Requirements & Constraints and Output Format sections

- [ ] 34. Create `edge/03_outposts_ml_patterns.md`
  - [ ] 34.1 Write template version comment, title, purpose, and role definition for AWS Outposts and hybrid ML architecture expertise
  - [ ] 34.2 Write Context & Inputs section with parameters: OUTPOST_SUBNET_IDS (list), S3_OUTPOSTS_BUCKET, DATA_SYNC_DIRECTION (outpost_to_region/region_to_outpost/bidirectional), LOCAL_INFERENCE_ENABLED, DATA_RESIDENCY_REGION
  - [ ] 34.3 Write Task section with directory tree and file-by-file instructions: SageMaker endpoint deployment targeting Outposts subnets, hybrid training pattern (preprocessing on Outposts ECS/EC2, training in Region), S3 on Outposts bucket configuration, DataSync/S3 replication for model artifact and training data sync, local inference continuity pattern for degraded connectivity, CloudWatch agent on Outposts for local metrics
  - [ ] 34.4 Write Code Scaffolding Hints with working `s3outposts.create_bucket()`, SageMaker endpoint config with Outposts `SubnetId`, DataSync task configuration, CloudWatch agent config for Outposts
  - [ ] 34.5 Write Integration Points referencing upstream `devops/04` (IAM), `devops/02` (VPC/Outposts networking), `mlops/03` (inference deployment) and downstream `devops/03` (CloudWatch monitoring), `finops/01` (cost tracking)
  - [ ] 34.6 Write Requirements & Constraints and Output Format sections

## Index and Documentation Updates

- [ ] 35. Update `README.md` with all new templates
  - [ ] 35.1 Add new category tables for data/, finops/, enterprise/, edge/ directories with template number, file path, and one-line description for each new template
  - [ ] 35.2 Add new rows to existing MLOps and DevOps tables for mlops/14-19 and devops/05-14
  - [ ] 35.3 Update the Architecture Overview ASCII diagram to show Data Engineering, Security, FinOps, Enterprise, Observability, and Edge layers
  - [ ] 35.4 Update the Recommended Build Order to incorporate all expansion areas at appropriate positions

- [ ] 36. Update `Library.md` with expanded directory tree and build order
  - [ ] 36.1 Update the directory tree to show all 67 templates across 8 directories with one-line descriptions
  - [ ] 36.2 Update the template count from 29 to 67
  - [ ] 36.3 Update the Recommended Build Order to include data engineering, security, FinOps, enterprise, observability, and edge templates

- [ ] 37. Update `PROMPT_GUIDE.md` with new interaction map and chaining examples
  - [ ] 37.1 Update the Template Interaction Map ASCII diagram to include all new templates and their cross-references
  - [ ] 37.2 Add a new chaining example demonstrating a workflow spanning expansion areas (e.g., Data Engineering → MLOps → Observability → FinOps)
  - [ ] 37.3 Update the Common Pitfalls table with new entries for expansion area templates

- [ ] 38. Update existing template Integration Points for bidirectional references
  - [ ] 38.1 Update `devops/04` (IAM) Integration Points to add downstream references to all new templates that consume IAM roles
  - [ ] 38.2 Update `devops/02` (VPC) Integration Points to add downstream references to `devops/09` (VPC endpoints), `edge/03` (Outposts), `data/02` (Kinesis)
  - [ ] 38.3 Update `devops/03` (CloudWatch) Integration Points to add downstream references to `devops/10-14` (observability templates), `edge/01-02` (edge monitoring)
  - [ ] 38.4 Update `mlops/03` (Inference) Integration Points to add downstream references to `finops/04` (inference cost), `devops/10` (tracing), `devops/14` (Clarify bias)
  - [ ] 38.5 Update `mlops/04` (RAG) Integration Points to add downstream references to `mlops/14` (Agents), `mlops/16` (Flows)
  - [ ] 38.6 Update `mlops/05` (Monitoring) Integration Points to add downstream references to `data/05` (EventBridge drift events), `devops/14` (Clarify bias)
  - [ ] 38.7 Update `mlops/10` (Model Registry) Integration Points to add downstream references to `enterprise/02` (cross-account), `enterprise/05` (central registry)
  - [ ] 38.8 Update `mlops/12` (Guardrails) Integration Points to add downstream references to `mlops/14` (Agents), `devops/12` (Bedrock logging)
  - [ ] 38.9 Update `mlops/13` (Continuous Training) Integration Points to add downstream references to `data/05` (EventBridge triggers)
