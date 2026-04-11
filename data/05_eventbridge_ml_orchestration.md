<!-- Template Version: 1.0 | boto3: 1.35+ | EventBridge: 2015-10-07 | EventBridge Schemas: 2019-12-02 -->

# Template Data 05 — EventBridge ML Orchestration

## Purpose
Generate a production-ready Amazon EventBridge event-driven orchestration layer for ML workflows: EventBridge rules for SageMaker events (training job state change, endpoint deployment state change, model package state change, pipeline execution state change), a custom event bus with schema registry for ML-specific events (drift detected, evaluation complete, data quality passed), Step Functions triggers for multi-step ML workflows, event pattern matching examples for filtering on specific statuses and outcomes, a drift-detection-to-retraining workflow, and cross-account event forwarding for publishing ML events from workload accounts to a central observability account.

---

## Role Definition

You are an expert AWS event-driven architecture engineer and ML platform specialist with expertise in:
- Amazon EventBridge: rules, event buses, event patterns, targets, input transformers, dead-letter queues
- EventBridge Schema Registry: `schemas.create_registry()`, `schemas.create_schema()`, schema discovery, OpenAPI/JSONSchema formats
- EventBridge cross-account event forwarding: resource-based policies on event buses, cross-account targets, PutEvents permissions
- SageMaker event integration: training job state change, endpoint deployment state change, model package state change, pipeline execution state change event patterns
- AWS Step Functions: state machine triggers from EventBridge, input/output processing, error handling, callback patterns
- Event pattern matching: prefix matching, numeric matching, exists filters, content-based filtering on event detail fields
- ML workflow orchestration: training → evaluation → approval → deployment pipelines, drift → retrain loops, data quality gating
- IAM policies for EventBridge rules, cross-account event bus access, and Step Functions execution roles
- CloudWatch monitoring for EventBridge: `FailedInvocations`, `TriggeredRules`, `Invocations` metrics

Generate complete, production-deployable code.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

EVENT_BUS_NAME:         [OPTIONAL: default]
                         Name suffix for the custom event bus.
                         If "default", rules are created on the default event bus.
                         Otherwise, a custom bus is created: {PROJECT_NAME}-ml-events-{ENV}
                         Use a custom bus to isolate ML events from other application events.

SAGEMAKER_EVENTS:       [REQUIRED - JSON list of SageMaker event types to monitor]
                         Example:
                         [
                           {
                             "event_type": "SageMaker Training Job State Change",
                             "filter_status": ["Completed", "Failed"],
                             "target": "step_functions"
                           },
                           {
                             "event_type": "SageMaker Endpoint Deployment State Change",
                             "filter_status": ["UPDATE_ENDPOINT_COMPLETED", "UPDATE_ENDPOINT_FAILED"],
                             "target": "sns"
                           },
                           {
                             "event_type": "SageMaker Model Package State Change",
                             "filter_status": ["Completed"],
                             "filter_approval": ["Approved"],
                             "target": "step_functions"
                           },
                           {
                             "event_type": "SageMaker Model Building Pipeline Execution Status Change",
                             "filter_status": ["Succeeded", "Failed"],
                             "target": "sns"
                           }
                         ]

CUSTOM_EVENT_SCHEMAS:   [OPTIONAL: default ML event schemas]
                         JSON object defining custom ML event schemas for the schema registry.
                         Example:
                         {
                           "DriftDetected": {
                             "source": "ml.monitoring",
                             "detail_type": "ML Drift Detected",
                             "schema": {
                               "endpoint_name": "string",
                               "drift_type": "string",
                               "drift_score": "number",
                               "baseline_metrics": "object",
                               "current_metrics": "object",
                               "timestamp": "string"
                             }
                           },
                           "EvaluationComplete": {
                             "source": "ml.evaluation",
                             "detail_type": "ML Evaluation Complete",
                             "schema": {
                               "model_package_arn": "string",
                               "metrics": "object",
                               "passed_threshold": "boolean",
                               "evaluation_job_arn": "string"
                             }
                           },
                           "DataQualityPassed": {
                             "source": "ml.data_quality",
                             "detail_type": "ML Data Quality Passed",
                             "schema": {
                               "dataset_path": "string",
                               "ruleset_name": "string",
                               "overall_score": "number",
                               "rule_results": "object"
                             }
                           }
                         }

STEP_FUNCTIONS_ARNS:    [OPTIONAL - JSON mapping of workflow names to Step Functions ARNs]
                         Step Functions state machines to trigger from EventBridge rules.
                         Example:
                         {
                           "training_complete_workflow": "arn:aws:states:us-east-1:111122223333:stateMachine:ml-deploy-pipeline",
                           "drift_retrain_workflow": "arn:aws:states:us-east-1:111122223333:stateMachine:ml-retrain-pipeline",
                           "model_approved_workflow": "arn:aws:states:us-east-1:111122223333:stateMachine:ml-promotion-pipeline"
                         }
                         If not provided, placeholder ARNs are used in generated code.

CROSS_ACCOUNT_TARGETS:  [OPTIONAL: none]
                         JSON list of cross-account event forwarding configurations.
                         Example:
                         [
                           {
                             "target_account_id": "999988887777",
                             "target_event_bus_arn": "arn:aws:events:us-east-1:999988887777:event-bus/central-ml-events",
                             "event_types": ["all"]
                           }
                         ]
                         Used to forward ML events to a central observability or governance account.

SNS_TOPIC_ARN:          [OPTIONAL - SNS topic for event notifications]
                         If not provided, a new topic is created: {PROJECT_NAME}-ml-events-{ENV}

DLQ_ARN:                [OPTIONAL - SQS dead-letter queue ARN for failed event deliveries]
                         If not provided, a new DLQ is created: {PROJECT_NAME}-ml-events-dlq-{ENV}
```

---

## Task

Generate complete EventBridge ML orchestration infrastructure:

```
{PROJECT_NAME}-eventbridge-ml/
├── rules/
│   ├── sagemaker_training_rule.py         # EventBridge rule for training job state changes
│   ├── sagemaker_endpoint_rule.py         # EventBridge rule for endpoint deployment state changes
│   ├── sagemaker_model_package_rule.py    # EventBridge rule for model package state changes
│   ├── sagemaker_pipeline_rule.py         # EventBridge rule for pipeline execution state changes
│   └── custom_event_rules.py             # EventBridge rules for custom ML events
├── event_bus/
│   ├── create_custom_bus.py               # Create custom event bus for ML events
│   ├── bus_policy.py                      # Resource-based policy for cross-account access
│   └── schema_registry.py                # Schema registry with ML event schemas
├── workflows/
│   ├── training_complete_trigger.py       # Step Functions trigger on training completion
│   ├── drift_retrain_workflow.py          # Drift detection → retraining workflow trigger
│   └── model_approval_trigger.py          # Model approval → deployment workflow trigger
├── cross_account/
│   ├── event_forwarding.py                # Cross-account event forwarding rules
│   └── target_bus_policy.py              # Target account event bus policy generator
├── publishers/
│   ├── publish_custom_event.py            # Publish custom ML events to the event bus
│   └── event_patterns.py                 # Event pattern matching examples and utilities
├── infrastructure/
│   ├── config.py                          # Central configuration
│   ├── create_sns_topic.py               # SNS topic for event notifications
│   └── create_dlq.py                     # SQS dead-letter queue for failed deliveries
├── run_setup.py                           # CLI orchestrator
└── requirements.txt
```

**config.py**: Central configuration dataclass with all parameters. Load from environment variables or CLI args. Validate required fields. Construct resource names using `{PROJECT_NAME}-{component}-{ENV}` convention. Parse SAGEMAKER_EVENTS, CUSTOM_EVENT_SCHEMAS, STEP_FUNCTIONS_ARNS, CROSS_ACCOUNT_TARGETS from JSON strings. Determine whether to use default or custom event bus based on EVENT_BUS_NAME.

**sagemaker_training_rule.py**: Create EventBridge rule for SageMaker Training Job State Change events:
- Call `events.put_rule()` with `Name` set to `{PROJECT_NAME}-training-state-{ENV}`
- Set `EventBusName` to the custom bus name or `default`
- Set `EventPattern` matching `source: ["aws.sagemaker"]`, `detail-type: ["SageMaker Training Job State Change"]`
- Filter on `detail.TrainingJobStatus` using the `filter_status` list from SAGEMAKER_EVENTS (e.g., `["Completed", "Failed"]`)
- Set `State` to `ENABLED`
- Add target using `events.put_targets()`: Step Functions state machine for `Completed` status (trigger deployment pipeline), SNS topic for `Failed` status (alert on failure)
- Use `InputTransformer` to extract `TrainingJobName`, `TrainingJobArn`, `TrainingJobStatus` from the event detail and pass as structured input to the target
- Configure `DeadLetterConfig` with DLQ_ARN for failed deliveries
- Tag with Project and Environment

**sagemaker_endpoint_rule.py**: Create EventBridge rule for SageMaker Endpoint Deployment State Change events:
- Call `events.put_rule()` with `Name` set to `{PROJECT_NAME}-endpoint-state-{ENV}`
- Set `EventPattern` matching `source: ["aws.sagemaker"]`, `detail-type: ["SageMaker Endpoint Deployment State Change"]`
- Filter on `detail.EndpointStatus` for deployment completion or failure
- Add SNS target for notifications on endpoint deployment status changes
- Use `InputTransformer` to extract `EndpointName`, `EndpointStatus`, `EndpointConfigName`

**sagemaker_model_package_rule.py**: Create EventBridge rule for SageMaker Model Package State Change events:
- Call `events.put_rule()` with `Name` set to `{PROJECT_NAME}-model-pkg-state-{ENV}`
- Set `EventPattern` matching `source: ["aws.sagemaker"]`, `detail-type: ["SageMaker Model Package State Change"]`
- Filter on `detail.ModelPackageStatus` for `Completed` and `detail.ModelApprovalStatus` for `Approved`
- Add Step Functions target to trigger model promotion/deployment pipeline when a model is approved
- Use `InputTransformer` to extract `ModelPackageArn`, `ModelPackageGroupName`, `ModelApprovalStatus`

**sagemaker_pipeline_rule.py**: Create EventBridge rule for SageMaker Pipeline Execution State Change events:
- Call `events.put_rule()` with `Name` set to `{PROJECT_NAME}-pipeline-state-{ENV}`
- Set `EventPattern` matching `source: ["aws.sagemaker"]`, `detail-type: ["SageMaker Model Building Pipeline Execution Status Change"]`
- Filter on `detail.currentPipelineExecutionStatus` for `Succeeded` or `Failed`
- Add SNS target for pipeline completion notifications
- Use `InputTransformer` to extract `pipelineArn`, `pipelineExecutionArn`, `currentPipelineExecutionStatus`

**custom_event_rules.py**: Create EventBridge rules for custom ML events defined in CUSTOM_EVENT_SCHEMAS:
- For each custom event schema, create a rule matching the custom `source` and `detail-type`
- `DriftDetected` rule: trigger the drift retrain Step Functions workflow
- `EvaluationComplete` rule: trigger SNS notification with evaluation results; if `passed_threshold` is true, trigger model promotion workflow
- `DataQualityPassed` rule: trigger downstream training pipeline via Step Functions
- Each rule uses the custom event bus (if configured)

**create_custom_bus.py**: Create a custom event bus for ML events:
- Call `events.create_event_bus()` with `Name` set to `{PROJECT_NAME}-ml-events-{ENV}`
- Tag with Project and Environment
- If CROSS_ACCOUNT_TARGETS is provided, attach a resource-based policy allowing cross-account PutEvents

**bus_policy.py**: Generate resource-based policy for the custom event bus:
- Allow the local account to put events
- If CROSS_ACCOUNT_TARGETS is provided, allow target accounts to receive events via `events:PutEvents` permission
- Use `events.put_permission()` to attach the policy to the custom event bus
- Scope permissions using `aws:PrincipalOrgID` condition if all accounts are in the same AWS Organization

**schema_registry.py**: Create EventBridge Schema Registry with ML event schemas:
- Call `schemas.create_registry()` with `RegistryName` set to `{PROJECT_NAME}-ml-schemas-{ENV}`
- For each schema in CUSTOM_EVENT_SCHEMAS, call `schemas.create_schema()` with:
  - `RegistryName`: the created registry
  - `SchemaName`: the event name (e.g., `DriftDetected`)
  - `Type`: `OpenApi3` (JSONSchema Draft 4 format)
  - `Content`: JSON schema defining the event structure with required fields and types
- Tag registry and schemas with Project and Environment

**training_complete_trigger.py**: Configure Step Functions trigger on training completion:
- Create EventBridge rule matching `SageMaker Training Job State Change` with `TrainingJobStatus: Completed`
- Add Step Functions state machine as target using `events.put_targets()` with:
  - `Arn`: the state machine ARN from STEP_FUNCTIONS_ARNS
  - `RoleArn`: IAM role allowing EventBridge to start Step Functions execution
  - `Input`: transformed event payload with training job details
- The triggered workflow evaluates the trained model, registers it in Model Registry, and optionally deploys

**drift_retrain_workflow.py**: Configure drift detection → retraining workflow:
- Create EventBridge rule on the custom event bus matching `source: ml.monitoring`, `detail-type: ML Drift Detected`
- Filter on `detail.drift_score` using numeric matching (e.g., `> 0.1` threshold)
- Add Step Functions target that triggers the retraining pipeline from `mlops/13`
- Use `InputTransformer` to pass `endpoint_name`, `drift_type`, `drift_score` to the retraining workflow
- Include a second target: SNS notification to alert the ML team about detected drift

**model_approval_trigger.py**: Configure model approval → deployment workflow:
- Create EventBridge rule matching `SageMaker Model Package State Change` with `ModelApprovalStatus: Approved`
- Add Step Functions target that triggers the deployment pipeline
- Use `InputTransformer` to pass `ModelPackageArn`, `ModelPackageGroupName` to the deployment workflow
- The triggered workflow deploys the approved model to staging, runs integration tests, and promotes to production

**event_forwarding.py**: Configure cross-account event forwarding:
- For each target in CROSS_ACCOUNT_TARGETS, create an EventBridge rule that matches ML events
- Add the target account's event bus as a target using `events.put_targets()` with:
  - `Arn`: the target event bus ARN
  - `RoleArn`: IAM role allowing EventBridge to put events to the target bus
- If `event_types` is `["all"]`, match all events on the custom ML event bus
- If specific event types are listed, create separate rules per event type

**target_bus_policy.py**: Generate the resource-based policy for the target account's event bus:
- Output the JSON policy document that the target account must apply to their event bus
- Allow `events:PutEvents` from the source account's EventBridge service principal
- Scope with `aws:SourceAccount` condition

**publish_custom_event.py**: Utility for publishing custom ML events:
- Provide `publish_event()` function that calls `events.put_events()` with:
  - `Source`: from CUSTOM_EVENT_SCHEMAS (e.g., `ml.monitoring`)
  - `DetailType`: from CUSTOM_EVENT_SCHEMAS (e.g., `ML Drift Detected`)
  - `Detail`: JSON string with event payload matching the schema
  - `EventBusName`: the custom event bus name
- Include helper functions for each custom event type: `publish_drift_event()`, `publish_evaluation_event()`, `publish_data_quality_event()`
- Validate event payload against the schema before publishing
- Handle `PutEvents` failures by checking `FailedEntryCount` in the response

**event_patterns.py**: Event pattern matching examples and utilities:
- Provide reusable event pattern builders for common ML filtering scenarios:
  - Filter training jobs by name prefix: `detail.TrainingJobName` with prefix matching
  - Filter model packages by approval status: `detail.ModelApprovalStatus` exact match
  - Filter pipeline executions by pipeline ARN: `detail.pipelineArn` exact match
  - Filter custom drift events by score threshold: `detail.drift_score` numeric matching
  - Filter by resource tags: `detail.Tags.Project` exact match
- Include a `validate_event_pattern()` function using `events.test_event_pattern()` to verify patterns match expected events

**create_sns_topic.py**: Create SNS topic for ML event notifications:
- Call `sns.create_topic()` with `Name` set to `{PROJECT_NAME}-ml-events-{ENV}`
- Add email subscription if notification email is provided
- Set topic policy allowing EventBridge to publish
- Tag with Project and Environment

**create_dlq.py**: Create SQS dead-letter queue for failed event deliveries:
- Call `sqs.create_queue()` with `QueueName` set to `{PROJECT_NAME}-ml-events-dlq-{ENV}`
- Set `MessageRetentionPeriod` to 14 days (maximum)
- Set queue policy allowing EventBridge to send messages
- Create CloudWatch alarm on `ApproximateNumberOfMessagesVisible` > 0 to alert on failed deliveries

**run_setup.py**: CLI orchestrator that runs setup steps in order:
1. Create custom event bus (if EVENT_BUS_NAME is not `default`)
2. Create schema registry and register ML event schemas
3. Create SNS topic and DLQ
4. Create SageMaker event rules (training, endpoint, model package, pipeline)
5. Create custom event rules (drift, evaluation, data quality)
6. Configure Step Functions triggers
7. Configure cross-account event forwarding (if CROSS_ACCOUNT_TARGETS provided)
8. Print summary with event bus name, rule names, schema registry name, and target configurations

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**Event Bus:** Use a custom event bus for ML-specific events to isolate them from application events on the default bus. Custom buses support resource-based policies for cross-account access and can be independently monitored. The default bus receives all AWS service events (SageMaker, Glue, etc.) automatically — create rules on the default bus for AWS service events and on the custom bus for custom ML events. Custom event bus names follow `{PROJECT_NAME}-ml-events-{ENV}`.

**SageMaker Event Patterns:** SageMaker sends events to the default EventBridge bus with `source: aws.sagemaker`. Key event types for ML orchestration: `SageMaker Training Job State Change` (detail field: `TrainingJobStatus` — `InProgress`, `Completed`, `Failed`, `Stopping`, `Stopped`), `SageMaker Endpoint Deployment State Change` (detail field: `EndpointStatus`), `SageMaker Model Package State Change` (detail fields: `ModelPackageStatus`, `ModelApprovalStatus` — `Approved`, `Rejected`, `PendingManualApproval`), `SageMaker Model Building Pipeline Execution Status Change` (detail field: `currentPipelineExecutionStatus` — `Executing`, `Succeeded`, `Failed`, `Stopping`, `Stopped`). Always filter on specific statuses to avoid triggering on intermediate state changes.

**Event Pattern Matching:** EventBridge supports prefix matching (`"prefix": "ml-"`), numeric matching (`"numeric": [">", 0.1]`), exists filters (`"exists": true`), and anything-but matching (`"anything-but": ["Failed"]`). Use these to build precise event patterns that minimize false triggers. Test patterns using `events.test_event_pattern()` before deploying rules.

**Step Functions Integration:** Use EventBridge rules to trigger Step Functions state machines for multi-step ML workflows. The EventBridge target requires an IAM role with `states:StartExecution` permission on the target state machine. Use `InputTransformer` to reshape the event payload into the format expected by the state machine. Set `RetryPolicy` with `MaximumRetryAttempts` and `MaximumEventAgeInSeconds` on the target for transient failures.

**Schema Registry:** Register custom ML event schemas in an EventBridge Schema Registry to enable schema discovery, code bindings, and event validation. Use `schemas.create_registry()` for the registry and `schemas.create_schema()` with `Type=OpenApi3` for each schema. Schema content follows JSONSchema Draft 4 format embedded in an OpenAPI 3.0 wrapper. Schema discovery can be enabled on the custom event bus to automatically detect and register new event schemas.

**Cross-Account Forwarding:** Forward ML events from workload accounts to a central observability or governance account using EventBridge cross-account targets. The source account creates a rule with the target account's event bus ARN as the target. The target account must have a resource-based policy on their event bus allowing `events:PutEvents` from the source account. Use `aws:PrincipalOrgID` condition to scope access to accounts within the same AWS Organization. The forwarding IAM role needs `events:PutEvents` permission on the target bus.

**Dead-Letter Queues:** Configure SQS dead-letter queues on all EventBridge targets to capture events that fail delivery. Set `MaximumRetryAttempts` to 3 and `MaximumEventAgeInSeconds` to 86400 (24 hours) on each target's retry policy. Monitor the DLQ with a CloudWatch alarm on `ApproximateNumberOfMessagesVisible` to detect delivery failures. DLQ messages contain the original event, the error code, and the error message for debugging.

**Security:** EventBridge rules require IAM roles to invoke targets (Step Functions, Lambda, SNS). Use least-privilege policies: `states:StartExecution` for Step Functions targets, `sns:Publish` for SNS targets, `events:PutEvents` for cross-account bus targets. Custom event bus policies should use `aws:SourceAccount` and `aws:PrincipalOrgID` conditions to restrict access. Encrypt SNS topics and SQS DLQs with KMS keys from `devops/08`.

**Monitoring:** Monitor EventBridge rules using CloudWatch metrics: `TriggeredRules` (rule matched an event), `Invocations` (target was invoked), `FailedInvocations` (target invocation failed), `ThrottledRules` (rule was throttled). Create CloudWatch alarms on `FailedInvocations > 0` for all critical ML workflow rules. Use CloudWatch Logs for EventBridge to log all matched events for debugging (enable via `events.put_rule()` with a CloudWatch Logs target).

**Naming:** All resources follow `{PROJECT_NAME}-{component}-{ENV}` convention:
- Custom event bus: `{PROJECT_NAME}-ml-events-{ENV}`
- Rules: `{PROJECT_NAME}-{event_type}-{ENV}` (e.g., `{PROJECT_NAME}-training-state-{ENV}`)
- Schema registry: `{PROJECT_NAME}-ml-schemas-{ENV}`
- SNS topic: `{PROJECT_NAME}-ml-events-{ENV}`
- DLQ: `{PROJECT_NAME}-ml-events-dlq-{ENV}`
- IAM roles: `{PROJECT_NAME}-eb-{target_type}-role-{ENV}`

---

## Code Scaffolding Hints

**Create EventBridge rule for SageMaker Training Job State Change:**
```python
import boto3
import json

events = boto3.client("events", region_name=AWS_REGION)

def create_training_job_rule(rule_name, event_bus_name, filter_statuses,
                              target_arn, target_role_arn, dlq_arn=None):
    """Create EventBridge rule for SageMaker training job state changes."""
    event_pattern = {
        "source": ["aws.sagemaker"],
        "detail-type": ["SageMaker Training Job State Change"],
        "detail": {
            "TrainingJobStatus": filter_statuses,
        },
    }

    # Create the rule
    events.put_rule(
        Name=rule_name,
        EventBusName=event_bus_name,
        EventPattern=json.dumps(event_pattern),
        State="ENABLED",
        Description=f"Trigger on SageMaker training job {'/'.join(filter_statuses)}",
        Tags=[
            {"Key": "Project", "Value": PROJECT_NAME},
            {"Key": "Environment", "Value": ENV},
        ],
    )

    # Add target with input transformer
    target = {
        "Id": f"{rule_name}-target",
        "Arn": target_arn,
        "RoleArn": target_role_arn,
        "InputTransformer": {
            "InputPathsMap": {
                "jobName": "$.detail.TrainingJobName",
                "jobArn": "$.detail.TrainingJobArn",
                "status": "$.detail.TrainingJobStatus",
                "endTime": "$.detail.TrainingEndTime",
            },
            "InputTemplate": '{"trainingJobName": <jobName>, "trainingJobArn": <jobArn>, '
                             '"status": <status>, "trainingEndTime": <endTime>}',
        },
        "RetryPolicy": {
            "MaximumRetryAttempts": 3,
            "MaximumEventAgeInSeconds": 86400,
        },
    }

    if dlq_arn:
        target["DeadLetterConfig"] = {"Arn": dlq_arn}

    events.put_targets(
        Rule=rule_name,
        EventBusName=event_bus_name,
        Targets=[target],
    )
    print(f"Created rule {rule_name} targeting {target_arn}")

# Example: Trigger Step Functions on training completion
create_training_job_rule(
    rule_name=f"{PROJECT_NAME}-training-state-{ENV}",
    event_bus_name="default",  # SageMaker events go to default bus
    filter_statuses=["Completed", "Failed"],
    target_arn=STEP_FUNCTIONS_ARNS["training_complete_workflow"],
    target_role_arn=f"arn:aws:iam::{AWS_ACCOUNT_ID}:role/{PROJECT_NAME}-eb-sfn-role-{ENV}",
    dlq_arn=DLQ_ARN,
)
```

**Create EventBridge rule for SageMaker Model Package State Change (approval-based):**
```python
def create_model_package_rule(rule_name, event_bus_name, filter_approval_statuses,
                               target_arn, target_role_arn, dlq_arn=None):
    """Create EventBridge rule for model package approval state changes."""
    event_pattern = {
        "source": ["aws.sagemaker"],
        "detail-type": ["SageMaker Model Package State Change"],
        "detail": {
            "ModelPackageStatus": ["Completed"],
            "ModelApprovalStatus": filter_approval_statuses,
        },
    }

    events.put_rule(
        Name=rule_name,
        EventBusName=event_bus_name,
        EventPattern=json.dumps(event_pattern),
        State="ENABLED",
        Description=f"Trigger on model package {'/'.join(filter_approval_statuses)}",
        Tags=[
            {"Key": "Project", "Value": PROJECT_NAME},
            {"Key": "Environment", "Value": ENV},
        ],
    )

    target = {
        "Id": f"{rule_name}-target",
        "Arn": target_arn,
        "RoleArn": target_role_arn,
        "InputTransformer": {
            "InputPathsMap": {
                "pkgArn": "$.detail.ModelPackageArn",
                "groupName": "$.detail.ModelPackageGroupName",
                "approval": "$.detail.ModelApprovalStatus",
                "version": "$.detail.ModelPackageVersion",
            },
            "InputTemplate": '{"modelPackageArn": <pkgArn>, "groupName": <groupName>, '
                             '"approvalStatus": <approval>, "version": <version>}',
        },
        "RetryPolicy": {
            "MaximumRetryAttempts": 3,
            "MaximumEventAgeInSeconds": 86400,
        },
    }

    if dlq_arn:
        target["DeadLetterConfig"] = {"Arn": dlq_arn}

    events.put_targets(
        Rule=rule_name,
        EventBusName=event_bus_name,
        Targets=[target],
    )
    print(f"Created rule {rule_name} for model approval events")

# Example: Trigger deployment pipeline on model approval
create_model_package_rule(
    rule_name=f"{PROJECT_NAME}-model-pkg-state-{ENV}",
    event_bus_name="default",
    filter_approval_statuses=["Approved"],
    target_arn=STEP_FUNCTIONS_ARNS["model_approved_workflow"],
    target_role_arn=f"arn:aws:iam::{AWS_ACCOUNT_ID}:role/{PROJECT_NAME}-eb-sfn-role-{ENV}",
    dlq_arn=DLQ_ARN,
)
```

**Create custom event bus and resource-based policy:**
```python
def create_custom_event_bus(bus_name, cross_account_ids=None):
    """Create a custom event bus for ML events with optional cross-account access."""
    response = events.create_event_bus(
        Name=bus_name,
        Tags=[
            {"Key": "Project", "Value": PROJECT_NAME},
            {"Key": "Environment", "Value": ENV},
        ],
    )
    print(f"Created custom event bus: {bus_name} (ARN: {response['EventBusArn']})")

    # Attach cross-account policy if needed
    if cross_account_ids:
        policy = {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Sid": "AllowCrossAccountPutEvents",
                    "Effect": "Allow",
                    "Principal": {"AWS": [
                        f"arn:aws:iam::{acct_id}:root"
                        for acct_id in cross_account_ids
                    ]},
                    "Action": "events:PutEvents",
                    "Resource": response["EventBusArn"],
                }
            ],
        }
        events.put_permission(
            EventBusName=bus_name,
            Policy=json.dumps(policy),
        )
        print(f"Attached cross-account policy for accounts: {cross_account_ids}")

    return response["EventBusArn"]

# Create custom ML event bus
bus_arn = create_custom_event_bus(
    bus_name=f"{PROJECT_NAME}-ml-events-{ENV}",
    cross_account_ids=[t["target_account_id"] for t in CROSS_ACCOUNT_TARGETS]
    if CROSS_ACCOUNT_TARGETS else None,
)
```

**Create schema registry with ML event schemas:**
```python
schemas = boto3.client("schemas", region_name=AWS_REGION)

def create_ml_schema_registry(registry_name, event_schemas):
    """Create EventBridge Schema Registry and register ML event schemas."""
    # Create the registry
    schemas.create_registry(
        RegistryName=registry_name,
        Description=f"ML event schemas for {PROJECT_NAME} ({ENV})",
        Tags={"Project": PROJECT_NAME, "Environment": ENV},
    )
    print(f"Created schema registry: {registry_name}")

    # Register each schema
    for schema_name, schema_def in event_schemas.items():
        # Build OpenAPI 3.0 schema with JSONSchema content
        openapi_schema = {
            "openapi": "3.0.0",
            "info": {
                "title": schema_name,
                "version": "1.0.0",
            },
            "paths": {},
            "components": {
                "schemas": {
                    schema_name: {
                        "type": "object",
                        "properties": {
                            "source": {"type": "string", "enum": [schema_def["source"]]},
                            "detail-type": {"type": "string", "enum": [schema_def["detail_type"]]},
                            "detail": {
                                "type": "object",
                                "properties": {
                                    field: {"type": field_type}
                                    for field, field_type in schema_def["schema"].items()
                                },
                                "required": list(schema_def["schema"].keys()),
                            },
                        },
                    }
                }
            },
        }

        schemas.create_schema(
            RegistryName=registry_name,
            SchemaName=schema_name,
            Type="OpenApi3",
            Content=json.dumps(openapi_schema),
            Description=f"Schema for {schema_def['detail_type']} events",
            Tags={"Project": PROJECT_NAME, "Environment": ENV},
        )
        print(f"Registered schema: {schema_name}")

# Create registry and register schemas
create_ml_schema_registry(
    registry_name=f"{PROJECT_NAME}-ml-schemas-{ENV}",
    event_schemas=CUSTOM_EVENT_SCHEMAS,
)
```

**Publish custom ML events to the event bus:**
```python
from datetime import datetime

def publish_event(event_bus_name, source, detail_type, detail):
    """Publish a custom ML event to the EventBridge event bus."""
    response = events.put_events(
        Entries=[
            {
                "Source": source,
                "DetailType": detail_type,
                "Detail": json.dumps(detail),
                "EventBusName": event_bus_name,
                "Time": datetime.utcnow(),
            }
        ]
    )

    if response["FailedEntryCount"] > 0:
        for entry in response["Entries"]:
            if "ErrorCode" in entry:
                print(f"Failed to publish event: {entry['ErrorCode']} - {entry['ErrorMessage']}")
        raise RuntimeError(f"Failed to publish {response['FailedEntryCount']} event(s)")

    print(f"Published {detail_type} event to {event_bus_name}")
    return response["Entries"][0]["EventId"]


def publish_drift_event(event_bus_name, endpoint_name, drift_type,
                         drift_score, baseline_metrics, current_metrics):
    """Publish a drift detection event."""
    return publish_event(
        event_bus_name=event_bus_name,
        source="ml.monitoring",
        detail_type="ML Drift Detected",
        detail={
            "endpoint_name": endpoint_name,
            "drift_type": drift_type,
            "drift_score": drift_score,
            "baseline_metrics": baseline_metrics,
            "current_metrics": current_metrics,
            "timestamp": datetime.utcnow().isoformat(),
        },
    )


def publish_evaluation_event(event_bus_name, model_package_arn, metrics,
                              passed_threshold, evaluation_job_arn):
    """Publish an evaluation complete event."""
    return publish_event(
        event_bus_name=event_bus_name,
        source="ml.evaluation",
        detail_type="ML Evaluation Complete",
        detail={
            "model_package_arn": model_package_arn,
            "metrics": metrics,
            "passed_threshold": passed_threshold,
            "evaluation_job_arn": evaluation_job_arn,
        },
    )


def publish_data_quality_event(event_bus_name, dataset_path, ruleset_name,
                                overall_score, rule_results):
    """Publish a data quality passed event."""
    return publish_event(
        event_bus_name=event_bus_name,
        source="ml.data_quality",
        detail_type="ML Data Quality Passed",
        detail={
            "dataset_path": dataset_path,
            "ruleset_name": ruleset_name,
            "overall_score": overall_score,
            "rule_results": rule_results,
        },
    )


# Example: Publish drift detection event
publish_drift_event(
    event_bus_name=f"{PROJECT_NAME}-ml-events-{ENV}",
    endpoint_name=f"{PROJECT_NAME}-inference-{ENV}",
    drift_type="data_drift",
    drift_score=0.25,
    baseline_metrics={"accuracy": 0.95, "f1": 0.93},
    current_metrics={"accuracy": 0.88, "f1": 0.85},
)
```

**Create drift detection → retraining workflow rule:**
```python
def create_drift_retrain_rule(rule_name, event_bus_name, drift_threshold,
                               sfn_arn, sfn_role_arn, sns_topic_arn, dlq_arn=None):
    """Create EventBridge rule that triggers retraining on drift detection."""
    event_pattern = {
        "source": ["ml.monitoring"],
        "detail-type": ["ML Drift Detected"],
        "detail": {
            "drift_score": [{"numeric": [">", drift_threshold]}],
        },
    }

    events.put_rule(
        Name=rule_name,
        EventBusName=event_bus_name,
        EventPattern=json.dumps(event_pattern),
        State="ENABLED",
        Description=f"Trigger retraining when drift score > {drift_threshold}",
        Tags=[
            {"Key": "Project", "Value": PROJECT_NAME},
            {"Key": "Environment", "Value": ENV},
        ],
    )

    targets = [
        # Target 1: Step Functions retraining workflow
        {
            "Id": f"{rule_name}-sfn",
            "Arn": sfn_arn,
            "RoleArn": sfn_role_arn,
            "InputTransformer": {
                "InputPathsMap": {
                    "endpoint": "$.detail.endpoint_name",
                    "driftType": "$.detail.drift_type",
                    "score": "$.detail.drift_score",
                },
                "InputTemplate": '{"endpointName": <endpoint>, "driftType": <driftType>, '
                                 '"driftScore": <score>, "action": "retrain"}',
            },
            "RetryPolicy": {
                "MaximumRetryAttempts": 2,
                "MaximumEventAgeInSeconds": 3600,
            },
        },
        # Target 2: SNS notification to ML team
        {
            "Id": f"{rule_name}-sns",
            "Arn": sns_topic_arn,
            "InputTransformer": {
                "InputPathsMap": {
                    "endpoint": "$.detail.endpoint_name",
                    "driftType": "$.detail.drift_type",
                    "score": "$.detail.drift_score",
                    "timestamp": "$.detail.timestamp",
                },
                "InputTemplate": '"[{ENV}] Drift detected on <endpoint>: '
                                 'type=<driftType>, score=<score> at <timestamp>. '
                                 'Retraining workflow triggered."',
            },
        },
    ]

    if dlq_arn:
        for t in targets:
            t["DeadLetterConfig"] = {"Arn": dlq_arn}

    events.put_targets(
        Rule=rule_name,
        EventBusName=event_bus_name,
        Targets=targets,
    )
    print(f"Created drift→retrain rule: {rule_name}")

# Example: Trigger retraining when drift score exceeds 0.1
create_drift_retrain_rule(
    rule_name=f"{PROJECT_NAME}-drift-retrain-{ENV}",
    event_bus_name=f"{PROJECT_NAME}-ml-events-{ENV}",
    drift_threshold=0.1,
    sfn_arn=STEP_FUNCTIONS_ARNS["drift_retrain_workflow"],
    sfn_role_arn=f"arn:aws:iam::{AWS_ACCOUNT_ID}:role/{PROJECT_NAME}-eb-sfn-role-{ENV}",
    sns_topic_arn=SNS_TOPIC_ARN,
    dlq_arn=DLQ_ARN,
)
```

**Configure cross-account event forwarding:**
```python
def create_cross_account_forwarding(rule_name, source_bus_name, target_bus_arn,
                                     forwarding_role_arn, event_types=None):
    """Forward ML events from workload account to central observability account."""
    # Match all events on the custom bus or specific event types
    if event_types and event_types != ["all"]:
        event_pattern = {
            "detail-type": event_types,
        }
    else:
        # Match all events on this bus (catch-all)
        event_pattern = {
            "source": [{"prefix": ""}],
        }

    events.put_rule(
        Name=rule_name,
        EventBusName=source_bus_name,
        EventPattern=json.dumps(event_pattern),
        State="ENABLED",
        Description=f"Forward ML events to {target_bus_arn}",
        Tags=[
            {"Key": "Project", "Value": PROJECT_NAME},
            {"Key": "Environment", "Value": ENV},
        ],
    )

    events.put_targets(
        Rule=rule_name,
        EventBusName=source_bus_name,
        Targets=[
            {
                "Id": f"{rule_name}-xacct",
                "Arn": target_bus_arn,
                "RoleArn": forwarding_role_arn,
                "RetryPolicy": {
                    "MaximumRetryAttempts": 3,
                    "MaximumEventAgeInSeconds": 86400,
                },
            }
        ],
    )
    print(f"Created cross-account forwarding rule: {rule_name} → {target_bus_arn}")

# Example: Forward all ML events to central observability account
for target_cfg in CROSS_ACCOUNT_TARGETS:
    create_cross_account_forwarding(
        rule_name=f"{PROJECT_NAME}-forward-{target_cfg['target_account_id'][:4]}-{ENV}",
        source_bus_name=f"{PROJECT_NAME}-ml-events-{ENV}",
        target_bus_arn=target_cfg["target_event_bus_arn"],
        forwarding_role_arn=f"arn:aws:iam::{AWS_ACCOUNT_ID}:role/{PROJECT_NAME}-eb-forward-role-{ENV}",
        event_types=target_cfg.get("event_types"),
    )
```

**Event pattern matching examples:**
```python
# Pattern 1: Filter training jobs by name prefix
training_prefix_pattern = {
    "source": ["aws.sagemaker"],
    "detail-type": ["SageMaker Training Job State Change"],
    "detail": {
        "TrainingJobName": [{"prefix": f"{PROJECT_NAME}-"}],
        "TrainingJobStatus": ["Completed"],
    },
}

# Pattern 2: Filter model packages by approval status
model_approval_pattern = {
    "source": ["aws.sagemaker"],
    "detail-type": ["SageMaker Model Package State Change"],
    "detail": {
        "ModelApprovalStatus": ["Approved"],
        "ModelPackageGroupName": [{"prefix": f"{PROJECT_NAME}-"}],
    },
}

# Pattern 3: Filter pipeline executions by specific pipeline ARN
pipeline_pattern = {
    "source": ["aws.sagemaker"],
    "detail-type": ["SageMaker Model Building Pipeline Execution Status Change"],
    "detail": {
        "pipelineArn": [f"arn:aws:sagemaker:{AWS_REGION}:{AWS_ACCOUNT_ID}:pipeline/{PROJECT_NAME}-pipeline"],
        "currentPipelineExecutionStatus": ["Succeeded", "Failed"],
    },
}

# Pattern 4: Filter custom drift events by score threshold (numeric matching)
drift_threshold_pattern = {
    "source": ["ml.monitoring"],
    "detail-type": ["ML Drift Detected"],
    "detail": {
        "drift_score": [{"numeric": [">", 0.1]}],
        "drift_type": ["data_drift", "concept_drift"],
    },
}

# Pattern 5: Anything-but filter (all statuses except InProgress)
non_inprogress_pattern = {
    "source": ["aws.sagemaker"],
    "detail-type": ["SageMaker Training Job State Change"],
    "detail": {
        "TrainingJobStatus": [{"anything-but": ["InProgress", "Stopping"]}],
    },
}


def validate_event_pattern(pattern, event):
    """Validate an event pattern matches an expected event using EventBridge API."""
    response = events.test_event_pattern(
        EventPattern=json.dumps(pattern),
        Event=json.dumps(event),
    )
    return response["Result"]


# Example: Validate the training prefix pattern
test_event = {
    "source": "aws.sagemaker",
    "detail-type": "SageMaker Training Job State Change",
    "detail": {
        "TrainingJobName": f"{PROJECT_NAME}-xgboost-20240101",
        "TrainingJobStatus": "Completed",
    },
}
assert validate_event_pattern(training_prefix_pattern, test_event), "Pattern should match"
```

**Create SQS dead-letter queue for failed event deliveries:**
```python
sqs = boto3.client("sqs", region_name=AWS_REGION)
cloudwatch = boto3.client("cloudwatch", region_name=AWS_REGION)

def create_event_dlq(queue_name):
    """Create SQS DLQ for failed EventBridge event deliveries."""
    response = sqs.create_queue(
        QueueName=queue_name,
        Attributes={
            "MessageRetentionPeriod": "1209600",  # 14 days (maximum)
            "VisibilityTimeout": "300",
        },
        tags={"Project": PROJECT_NAME, "Environment": ENV},
    )
    queue_url = response["QueueUrl"]

    # Get queue ARN for policy
    attrs = sqs.get_queue_attributes(
        QueueUrl=queue_url, AttributeNames=["QueueArn"]
    )
    queue_arn = attrs["Attributes"]["QueueArn"]

    # Allow EventBridge to send messages to the DLQ
    policy = {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "AllowEventBridgeSend",
                "Effect": "Allow",
                "Principal": {"Service": "events.amazonaws.com"},
                "Action": "sqs:SendMessage",
                "Resource": queue_arn,
                "Condition": {
                    "ArnLike": {
                        "aws:SourceArn": f"arn:aws:events:{AWS_REGION}:{AWS_ACCOUNT_ID}:rule/*"
                    }
                },
            }
        ],
    }
    sqs.set_queue_attributes(
        QueueUrl=queue_url,
        Attributes={"Policy": json.dumps(policy)},
    )

    # CloudWatch alarm for failed deliveries
    cloudwatch.put_metric_alarm(
        AlarmName=f"{PROJECT_NAME}-eb-dlq-messages-{ENV}",
        MetricName="ApproximateNumberOfMessagesVisible",
        Namespace="AWS/SQS",
        Dimensions=[{"Name": "QueueName", "Value": queue_name}],
        Statistic="Sum",
        Period=300,
        EvaluationPeriods=1,
        Threshold=0,
        ComparisonOperator="GreaterThanThreshold",
        AlarmActions=[SNS_TOPIC_ARN],
        AlarmDescription=f"EventBridge DLQ has failed event deliveries for {PROJECT_NAME}",
        Tags=[
            {"Key": "Project", "Value": PROJECT_NAME},
            {"Key": "Environment", "Value": ENV},
        ],
    )

    print(f"Created DLQ: {queue_name} (ARN: {queue_arn})")
    return queue_arn

# Create DLQ
dlq_arn = create_event_dlq(f"{PROJECT_NAME}-ml-events-dlq-{ENV}")
```

---

## Integration Points

- **Upstream**: `devops/04` → IAM roles for EventBridge rule targets (Step Functions execution, SNS publish, cross-account PutEvents)
- **Upstream**: `mlops/05` → Drift detection events published to the custom event bus trigger retraining workflows
- **Upstream**: `mlops/13` → Continuous training pipeline triggered by drift→retrain EventBridge rules; training completion events feed back into orchestration
- **Upstream**: `data/01` → Glue ETL job state change events can trigger downstream ML pipelines via EventBridge
- **Downstream**: `mlops/10` → Model Registry events (model package approval) trigger cross-account deployment pipelines
- **Downstream**: `enterprise/05` → Central model registry receives forwarded model approval events from workload accounts
- **Downstream**: `devops/05` → Config rule evaluation events can be published to the ML event bus for compliance monitoring
