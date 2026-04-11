<!-- Template Version: 1.0 | boto3: 1.35+ -->

# Template DevOps 05 — AWS Config Rules for ML Compliance

## Purpose
Generate production-ready AWS Config rules that enforce ML resource compliance: managed Config rules for SageMaker encryption and network access, custom Lambda evaluators for instance type allowlists and Bedrock model restrictions, a conformance pack bundling all ML rules, SSM Automation remediation documents, EventBridge→SNS→Security Hub notification pipeline, and Config advanced query SQL for compliance dashboards.

---

## Role Definition

You are an expert AWS security and compliance engineer specializing in ML workload governance with expertise in:
- AWS Config: managed rules, custom Lambda rules, conformance packs, advanced queries (SQL)
- SageMaker compliance: KMS encryption enforcement, VPC-only mode, internet access restrictions, instance type governance
- Bedrock compliance: model ID restrictions, invocation logging enforcement
- Custom Config rule Lambda evaluators: resource evaluation logic, compliance result reporting
- SSM Automation documents: automated remediation runbooks for non-compliant resources
- EventBridge rules for Config compliance change notifications
- Security Hub integration for centralized compliance findings
- Config advanced queries (SQL) for compliance dashboards and reporting

Generate complete, production-deployable compliance infrastructure code.

---

## Context & Inputs

```
PROJECT_NAME:               [REQUIRED]
AWS_REGION:                 [REQUIRED]
AWS_ACCOUNT_ID:             [REQUIRED]
ENV:                        [REQUIRED - dev | stage | prod]

MANAGED_RULES:              [OPTIONAL: all]
                            Options (comma-separated):
                            - sagemaker-endpoint-configuration-kms-key-configured
                            - sagemaker-notebook-instance-kms-key-configured
                            - sagemaker-notebook-no-direct-internet-access
                            - all (deploys all three)

CUSTOM_RULES:               [OPTIONAL: all]
                            JSON list of custom rule configurations:
                            [
                                {
                                    "name": "approved-instance-types",
                                    "description": "SageMaker endpoints use only approved instance types",
                                    "resource_type": "AWS::SageMaker::EndpointConfig"
                                },
                                {
                                    "name": "vpc-only-training",
                                    "description": "SageMaker training jobs run in VPC-only mode",
                                    "resource_type": "AWS::SageMaker::NotebookInstance"
                                },
                                {
                                    "name": "bedrock-model-restrictions",
                                    "description": "Bedrock access restricted to approved model IDs",
                                    "resource_type": "AWS::Bedrock::Agent"
                                }
                            ]
                            Use "all" to deploy all three custom rules.

APPROVED_INSTANCE_TYPES:    [OPTIONAL: ml.m5.xlarge,ml.m5.2xlarge,ml.g5.xlarge,ml.g5.2xlarge,ml.inf2.xlarge]
                            Comma-separated list of allowed SageMaker instance types.

APPROVED_BEDROCK_MODELS:    [OPTIONAL: anthropic.claude-3-5-sonnet-20241022-v2:0,anthropic.claude-3-haiku-20240307-v1:0,amazon.nova-pro-v1:0]
                            Comma-separated list of allowed Bedrock model IDs.

CONFORMANCE_PACK_NAME:      [OPTIONAL: {PROJECT_NAME}-ml-compliance-{ENV}]
                            Name for the conformance pack bundling all rules.

REMEDIATION_ENABLED:        [OPTIONAL: true]
                            Enable SSM Automation remediation for non-compliant resources.

KMS_KEY_ARN:                [OPTIONAL - KMS key ARN for encryption remediation, from devops/08]
SNS_TOPIC_ARN:              [OPTIONAL - existing SNS topic for notifications]
SECURITY_HUB_ENABLED:       [OPTIONAL: true - send findings to Security Hub]
```

---

## Task

Generate complete Config compliance infrastructure:

```
{PROJECT_NAME}-config-compliance/
├── managed_rules/
│   ├── deploy_managed_rules.py        # Deploy managed Config rules for SageMaker
│   └── config.py                      # Central configuration
├── custom_rules/
│   ├── deploy_custom_rules.py         # Deploy custom Config rules with Lambda evaluators
│   └── evaluators/
│       ├── approved_instance_types/
│       │   └── handler.py             # Lambda: check endpoint instance type allowlist
│       ├── vpc_only_training/
│       │   └── handler.py             # Lambda: check training jobs run in VPC-only mode
│       └── bedrock_model_restrictions/
│           └── handler.py             # Lambda: check Bedrock model ID restrictions
├── conformance_pack/
│   ├── deploy_conformance_pack.py     # Deploy conformance pack bundling all rules
│   └── ml_compliance_pack.yaml        # Conformance pack template (CloudFormation)
├── remediation/
│   ├── deploy_remediation.py          # Attach SSM Automation remediation to rules
│   ├── ssm_documents/
│   │   ├── enable_notebook_kms.yaml   # SSM doc: enable KMS on notebook instance
│   │   └── disable_notebook_internet.yaml  # SSM doc: disable direct internet access
│   └── remediation_role.py            # IAM role for SSM Automation execution
├── notifications/
│   ├── deploy_notifications.py        # EventBridge → SNS → Security Hub pipeline
│   └── securityhub_import/
│       └── handler.py                 # Lambda: import Config findings to Security Hub
├── dashboard/
│   └── compliance_queries.sql         # Config advanced query SQL for compliance reporting
├── cleanup/
│   └── delete_resources.py            # Remove all Config rules and resources
├── run_setup.py                       # CLI orchestrator
└── requirements.txt
```

**config.py**: Central configuration dataclass with all parameters. Load from environment variables or CLI args. Validate required fields. Construct resource names using `{PROJECT_NAME}-{component}-{ENV}` convention. Parse APPROVED_INSTANCE_TYPES and APPROVED_BEDROCK_MODELS into lists.

**deploy_managed_rules.py**: Deploy AWS Config managed rules for SageMaker:
- `sagemaker-endpoint-configuration-kms-key-configured`: Checks that SageMaker endpoint configurations have KMS encryption enabled. Trigger type: configuration change on `AWS::SageMaker::EndpointConfig`.
- `sagemaker-notebook-instance-kms-key-configured`: Checks that SageMaker notebook instances have KMS encryption enabled. Trigger type: configuration change on `AWS::SageMaker::NotebookInstance`.
- `sagemaker-notebook-no-direct-internet-access`: Checks that SageMaker notebook instances do not have direct internet access. Trigger type: configuration change on `AWS::SageMaker::NotebookInstance`.
- Each rule tagged with Project, Environment, ManagedBy.
- Use `config.put_config_rule()` with `Source.Owner = "AWS"` and `Source.SourceIdentifier` set to the managed rule identifier.

**deploy_custom_rules.py**: Deploy custom Config rules with Lambda evaluators:
- For each custom rule, create the Lambda function, then call `config.put_config_rule()` with `Source.Owner = "CUSTOM_LAMBDA"` and `Source.SourceIdentifier` set to the Lambda ARN.
- Add Lambda permission for Config to invoke the function using `lambda_client.add_permission()` with principal `config.amazonaws.com`.
- Set `Source.SourceDetails` with `MessageType = "ConfigurationItemChangeNotification"` for change-triggered rules.

**approved_instance_types/handler.py**: Lambda evaluator that:
- Receives Config event with `invokingEvent` containing the `configurationItem`
- Extracts `ProductionVariants` from the SageMaker EndpointConfig configuration
- Checks each variant's `InstanceType` against APPROVED_INSTANCE_TYPES allowlist
- Returns `COMPLIANT` if all instance types are approved, `NON_COMPLIANT` with annotation listing the offending types otherwise
- Calls `config.put_evaluations()` with the evaluation result

**vpc_only_training/handler.py**: Lambda evaluator that:
- Receives Config event for SageMaker NotebookInstance resources
- Checks `DirectInternetAccess` is set to `Disabled`
- Checks `SubnetId` is present (indicating VPC attachment)
- Returns `COMPLIANT` if both conditions met, `NON_COMPLIANT` otherwise
- Includes annotation describing which condition failed

**bedrock_model_restrictions/handler.py**: Lambda evaluator that:
- Receives Config event for Bedrock Agent resources
- Extracts `foundationModel` from the agent configuration
- Checks the model ID against APPROVED_BEDROCK_MODELS allowlist
- Returns `COMPLIANT` if model is approved, `NON_COMPLIANT` with annotation listing the unapproved model

**deploy_conformance_pack.py**: Deploy conformance pack using `config.put_conformance_pack()`:
- Bundle all managed and custom rules into a single conformance pack
- Use the `ml_compliance_pack.yaml` CloudFormation template
- Pass APPROVED_INSTANCE_TYPES and KMS_KEY_ARN as conformance pack input parameters

**ml_compliance_pack.yaml**: CloudFormation template defining:
- All three managed Config rules as `AWS::Config::ConfigRule` resources
- Parameters for `ApprovedInstanceTypes`, `KmsKeyArn`
- Conformance pack input parameter mappings

**deploy_remediation.py**: Attach SSM Automation remediation to non-compliant rules:
- Use `config.put_remediation_configurations()` to link each rule to an SSM document
- Set `Automatic = True` if REMEDIATION_ENABLED is true
- Configure `MaximumAutomaticAttempts` and `RetryAttemptSeconds`
- Pass the remediation IAM role ARN for SSM execution

**enable_notebook_kms.yaml**: SSM Automation document that:
- Accepts `NotebookInstanceName` and `KmsKeyId` as parameters
- Stops the notebook instance using `aws:executeAwsApi` with `sagemaker:StopNotebookInstance`
- Waits for stopped state using `aws:waitForAwsResourceProperty`
- Updates the notebook instance with KMS key using `aws:executeAwsApi` with `sagemaker:UpdateNotebookInstance`
- Restarts the notebook instance

**disable_notebook_internet.yaml**: SSM Automation document that:
- Accepts `NotebookInstanceName` as parameter
- Stops the notebook instance
- Updates `DirectInternetAccess` to `Disabled` (note: this requires the notebook to be in a VPC with SubnetId)
- Restarts the notebook instance

**remediation_role.py**: Create IAM role for SSM Automation execution:
- Trust policy for `ssm.amazonaws.com`
- Permissions: `sagemaker:StopNotebookInstance`, `sagemaker:UpdateNotebookInstance`, `sagemaker:StartNotebookInstance`, `sagemaker:DescribeNotebookInstance`
- Scoped to project resources using tag conditions

**deploy_notifications.py**: Set up EventBridge → SNS → Security Hub pipeline:
- Create EventBridge rule matching Config compliance change events:
  - Source: `aws.config`
  - Detail-type: `Config Rules Compliance Change`
  - Detail: `newEvaluationResult.complianceType = "NON_COMPLIANT"`
- Target 1: SNS topic for email/Slack notifications
- Target 2: Lambda function for Security Hub import (if SECURITY_HUB_ENABLED)
- Create SNS topic if SNS_TOPIC_ARN not provided

**securityhub_import/handler.py**: Lambda function that:
- Receives EventBridge event for Config NON_COMPLIANT evaluation
- Transforms the Config finding into ASFF (AWS Security Finding Format)
- Calls `securityhub.batch_import_findings()` with the formatted finding
- Sets `ProductArn`, `GeneratorId`, `Types`, `Severity`, `Resources`, `Compliance.Status`
- Maps Config rule name to Security Hub finding type

**compliance_queries.sql**: Config advanced query SQL statements:
- Query 1: Compliance summary across all ML resources — percentage compliant by resource type
- Query 2: List all NON_COMPLIANT SageMaker resources with rule name and annotation
- Query 3: Encryption compliance — all SageMaker resources missing KMS configuration
- Query 4: Network compliance — notebook instances with direct internet access
- Query 5: Instance type compliance — endpoints using unapproved instance types
- Query 6: Trend query — compliance status changes over time

**delete_resources.py**: Clean up all resources: delete Config rules, conformance pack, Lambda functions, SSM documents, EventBridge rules, SNS topics, IAM roles.

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**Config Recorder:** A Config recorder must be enabled in the target region before deploying rules. The recorder must track `AWS::SageMaker::*` and `AWS::Bedrock::*` resource types. Verify with `config.describe_configuration_recorders()`.

**Managed Rules:** Use exact managed rule identifiers — these are case-sensitive. Do not modify managed rule evaluation logic. Managed rules are maintained by AWS and automatically updated.

**Custom Rules:** Lambda evaluators must return evaluations within 15 seconds. Use `config.put_evaluations()` with `ResultToken` from the invoking event. Handle `NOT_APPLICABLE` for resources outside scope. Include meaningful annotations for NON_COMPLIANT results.

**Conformance Pack:** Maximum 25 rules per conformance pack. Conformance pack names must be unique per account/region. Use input parameters for values that vary across environments.

**Remediation:** SSM Automation documents must be idempotent — running remediation on an already-compliant resource should be a no-op. Set `MaximumAutomaticAttempts = 3` and `RetryAttemptSeconds = 60`. Remediation role must follow least privilege.

**Security Hub:** ASFF findings must include `SchemaVersion = "2018-10-08"`. Use `ProductArn = "arn:aws:securityhub:{region}:{account}:product/{account}/default"` for custom findings. Set appropriate severity based on rule criticality (KMS = HIGH, internet access = CRITICAL, instance type = MEDIUM).

**Naming:** All resources follow `{PROJECT_NAME}-{component}-{ENV}` convention:
- Config rules: `{PROJECT_NAME}-config-{rule-name}-{ENV}`
- Lambda evaluators: `{PROJECT_NAME}-config-eval-{rule-name}-{ENV}`
- Conformance pack: `{PROJECT_NAME}-ml-compliance-{ENV}`
- SSM documents: `{PROJECT_NAME}-remediate-{action}-{ENV}`
- EventBridge rule: `{PROJECT_NAME}-config-noncompliant-{ENV}`

**Cost:** Config rules: $0.001 per rule evaluation. Custom Lambda rules incur Lambda execution costs. Conformance packs: no additional charge beyond rule evaluations. Config advanced queries: $0.003 per query. Budget for ~1000 evaluations/month in dev, ~10,000 in prod.

---

## Code Scaffolding Hints

**Deploy managed Config rule:**
```python
config_client = boto3.client("config", region_name=AWS_REGION)

# SageMaker endpoint KMS encryption check
config_client.put_config_rule(
    ConfigRule={
        "ConfigRuleName": f"{PROJECT_NAME}-config-sagemaker-endpoint-kms-{ENV}",
        "Description": "Checks that SageMaker endpoint configurations have KMS encryption enabled",
        "Scope": {
            "ComplianceResourceTypes": ["AWS::SageMaker::EndpointConfig"]
        },
        "Source": {
            "Owner": "AWS",
            "SourceIdentifier": "SAGEMAKER_ENDPOINT_CONFIGURATION_KMS_KEY_CONFIGURED",
        },
        "Tags": [
            {"Key": "Project", "Value": PROJECT_NAME},
            {"Key": "Environment", "Value": ENV},
            {"Key": "ManagedBy", "Value": "config-compliance"},
        ],
    }
)

# SageMaker notebook KMS encryption check
config_client.put_config_rule(
    ConfigRule={
        "ConfigRuleName": f"{PROJECT_NAME}-config-sagemaker-notebook-kms-{ENV}",
        "Description": "Checks that SageMaker notebook instances have KMS encryption enabled",
        "Scope": {
            "ComplianceResourceTypes": ["AWS::SageMaker::NotebookInstance"]
        },
        "Source": {
            "Owner": "AWS",
            "SourceIdentifier": "SAGEMAKER_NOTEBOOK_INSTANCE_KMS_KEY_CONFIGURED",
        },
    }
)

# SageMaker notebook no direct internet access
config_client.put_config_rule(
    ConfigRule={
        "ConfigRuleName": f"{PROJECT_NAME}-config-sagemaker-no-internet-{ENV}",
        "Description": "Checks that SageMaker notebook instances do not have direct internet access",
        "Scope": {
            "ComplianceResourceTypes": ["AWS::SageMaker::NotebookInstance"]
        },
        "Source": {
            "Owner": "AWS",
            "SourceIdentifier": "SAGEMAKER_NOTEBOOK_NO_DIRECT_INTERNET_ACCESS",
        },
    }
)
```

**Deploy custom Config rule with Lambda evaluator:**
```python
import json

lambda_client = boto3.client("lambda", region_name=AWS_REGION)
config_client = boto3.client("config", region_name=AWS_REGION)

evaluator_arn = f"arn:aws:lambda:{AWS_REGION}:{AWS_ACCOUNT_ID}:function:{PROJECT_NAME}-config-eval-approved-instance-types-{ENV}"

# Grant Config permission to invoke the Lambda
lambda_client.add_permission(
    FunctionName=evaluator_arn,
    StatementId="AllowConfigInvoke",
    Action="lambda:InvokeFunction",
    Principal="config.amazonaws.com",
    SourceAccount=AWS_ACCOUNT_ID,
)

# Create the custom Config rule
config_client.put_config_rule(
    ConfigRule={
        "ConfigRuleName": f"{PROJECT_NAME}-config-approved-instance-types-{ENV}",
        "Description": "Checks SageMaker endpoints use only approved instance types",
        "Scope": {
            "ComplianceResourceTypes": ["AWS::SageMaker::EndpointConfig"]
        },
        "Source": {
            "Owner": "CUSTOM_LAMBDA",
            "SourceIdentifier": evaluator_arn,
            "SourceDetails": [
                {
                    "EventSource": "aws.config",
                    "MessageType": "ConfigurationItemChangeNotification",
                }
            ],
        },
        "InputParameters": json.dumps({
            "approvedInstanceTypes": APPROVED_INSTANCE_TYPES,
        }),
    }
)
```

**Lambda evaluator function pattern (approved instance types):**
```python
import json
import boto3
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)

config_client = boto3.client("config")

def lambda_handler(event, context):
    """Evaluate SageMaker EndpointConfig for approved instance types."""
    invoking_event = json.loads(event["invokingEvent"])
    rule_parameters = json.loads(event.get("ruleParameters", "{}"))
    configuration_item = invoking_event.get("configurationItem", {})

    # Parse approved instance types from rule parameters
    approved_types = [
        t.strip()
        for t in rule_parameters.get("approvedInstanceTypes", "").split(",")
        if t.strip()
    ]

    resource_type = configuration_item.get("resourceType", "")
    resource_id = configuration_item.get("resourceId", "")

    # Skip if resource was deleted
    if configuration_item.get("configurationItemStatus") == "ResourceDeleted":
        compliance_type = "NOT_APPLICABLE"
        annotation = "Resource has been deleted"
    elif resource_type != "AWS::SageMaker::EndpointConfig":
        compliance_type = "NOT_APPLICABLE"
        annotation = "Resource type not in scope"
    else:
        configuration = configuration_item.get("configuration", {})
        production_variants = configuration.get("productionVariants", [])

        non_compliant_types = []
        for variant in production_variants:
            instance_type = variant.get("instanceType", "")
            if instance_type and instance_type not in approved_types:
                non_compliant_types.append(instance_type)

        if non_compliant_types:
            compliance_type = "NON_COMPLIANT"
            annotation = f"Unapproved instance types: {', '.join(non_compliant_types)}. Approved: {', '.join(approved_types)}"
        else:
            compliance_type = "COMPLIANT"
            annotation = "All instance types are approved"

    config_client.put_evaluations(
        Evaluations=[
            {
                "ComplianceResourceType": resource_type,
                "ComplianceResourceId": resource_id,
                "ComplianceType": compliance_type,
                "Annotation": annotation[:255],
                "OrderingTimestamp": configuration_item.get(
                    "configurationItemCaptureTime",
                    invoking_event.get("notificationCreationTime"),
                ),
            }
        ],
        ResultToken=event["resultToken"],
    )

    return {"compliance_type": compliance_type, "annotation": annotation}
```

**Deploy conformance pack:**
```python
config_client = boto3.client("config", region_name=AWS_REGION)

with open("conformance_pack/ml_compliance_pack.yaml", "r") as f:
    template_body = f.read()

config_client.put_conformance_pack(
    ConformancePackName=f"{PROJECT_NAME}-ml-compliance-{ENV}",
    TemplateBody=template_body,
    ConformancePackInputParameters=[
        {
            "ParameterName": "ApprovedInstanceTypes",
            "ParameterValue": APPROVED_INSTANCE_TYPES,
        },
        {
            "ParameterName": "KmsKeyArn",
            "ParameterValue": KMS_KEY_ARN,
        },
    ],
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)
```

**SSM Automation document for notebook KMS remediation:**
```yaml
description: Enable KMS encryption on a SageMaker notebook instance
schemaVersion: "0.3"
assumeRole: "{{ AutomationAssumeRole }}"
parameters:
  NotebookInstanceName:
    type: String
    description: Name of the SageMaker notebook instance
  KmsKeyId:
    type: String
    description: KMS key ID or ARN for encryption
  AutomationAssumeRole:
    type: String
    description: IAM role ARN for SSM Automation execution
mainSteps:
  - name: StopNotebookInstance
    action: aws:executeAwsApi
    inputs:
      Service: sagemaker
      Api: StopNotebookInstance
      NotebookInstanceName: "{{ NotebookInstanceName }}"
    onFailure: Abort

  - name: WaitForStopped
    action: aws:waitForAwsResourceProperty
    timeoutSeconds: 600
    inputs:
      Service: sagemaker
      Api: DescribeNotebookInstance
      NotebookInstanceName: "{{ NotebookInstanceName }}"
      PropertySelector: "$.NotebookInstanceStatus"
      DesiredValues:
        - Stopped

  - name: UpdateWithKms
    action: aws:executeAwsApi
    inputs:
      Service: sagemaker
      Api: UpdateNotebookInstance
      NotebookInstanceName: "{{ NotebookInstanceName }}"
      KmsKeyId: "{{ KmsKeyId }}"

  - name: StartNotebookInstance
    action: aws:executeAwsApi
    inputs:
      Service: sagemaker
      Api: StartNotebookInstance
      NotebookInstanceName: "{{ NotebookInstanceName }}"

  - name: WaitForInService
    action: aws:waitForAwsResourceProperty
    timeoutSeconds: 600
    inputs:
      Service: sagemaker
      Api: DescribeNotebookInstance
      NotebookInstanceName: "{{ NotebookInstanceName }}"
      PropertySelector: "$.NotebookInstanceStatus"
      DesiredValues:
        - InService
```

**Config advanced query SQL for compliance dashboard:**
```sql
-- Query 1: Compliance summary by resource type
SELECT
  resourceType,
  COUNT(*) AS totalResources,
  SUM(CASE WHEN complianceType = 'COMPLIANT' THEN 1 ELSE 0 END) AS compliantCount,
  SUM(CASE WHEN complianceType = 'NON_COMPLIANT' THEN 1 ELSE 0 END) AS nonCompliantCount
WHERE
  resourceType LIKE 'AWS::SageMaker::%'
GROUP BY
  resourceType

-- Query 2: All NON_COMPLIANT SageMaker resources
SELECT
  resourceId,
  resourceType,
  resourceName,
  configuration.kmsKeyId,
  tags
WHERE
  resourceType LIKE 'AWS::SageMaker::%'
  AND complianceType = 'NON_COMPLIANT'

-- Query 3: SageMaker resources missing KMS encryption
SELECT
  resourceId,
  resourceType,
  resourceName,
  configuration
WHERE
  resourceType IN (
    'AWS::SageMaker::EndpointConfig',
    'AWS::SageMaker::NotebookInstance'
  )
  AND configuration.kmsKeyId IS NULL

-- Query 4: Notebook instances with direct internet access
SELECT
  resourceId,
  resourceName,
  configuration.directInternetAccess,
  configuration.subnetId
WHERE
  resourceType = 'AWS::SageMaker::NotebookInstance'
  AND configuration.directInternetAccess = 'Enabled'
```

**EventBridge rule for Config NON_COMPLIANT notifications:**
```python
events_client = boto3.client("events", region_name=AWS_REGION)
sns_client = boto3.client("sns", region_name=AWS_REGION)

# Create SNS topic if not provided
if not SNS_TOPIC_ARN:
    topic_response = sns_client.create_topic(
        Name=f"{PROJECT_NAME}-config-compliance-{ENV}",
        Tags=[
            {"Key": "Project", "Value": PROJECT_NAME},
            {"Key": "Environment", "Value": ENV},
        ],
    )
    sns_topic_arn = topic_response["TopicArn"]
else:
    sns_topic_arn = SNS_TOPIC_ARN

# Create EventBridge rule for NON_COMPLIANT Config evaluations
events_client.put_rule(
    Name=f"{PROJECT_NAME}-config-noncompliant-{ENV}",
    Description="Trigger on AWS Config NON_COMPLIANT evaluations for ML resources",
    EventPattern=json.dumps({
        "source": ["aws.config"],
        "detail-type": ["Config Rules Compliance Change"],
        "detail": {
            "messageType": ["ComplianceChangeNotification"],
            "newEvaluationResult": {
                "complianceType": ["NON_COMPLIANT"]
            },
        },
    }),
    State="ENABLED",
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)

# Add SNS target
events_client.put_targets(
    Rule=f"{PROJECT_NAME}-config-noncompliant-{ENV}",
    Targets=[
        {
            "Id": "sns-notification",
            "Arn": sns_topic_arn,
        },
    ],
)
```

**Security Hub finding import Lambda:**
```python
import json
import boto3
from datetime import datetime

securityhub = boto3.client("securityhub")

def lambda_handler(event, context):
    """Import Config NON_COMPLIANT finding into Security Hub as ASFF."""
    detail = event.get("detail", {})
    resource_type = detail.get("resourceType", "")
    resource_id = detail.get("resourceId", "")
    rule_name = detail.get("configRuleName", "")
    account_id = event.get("account", "")
    region = event.get("region", "")
    new_result = detail.get("newEvaluationResult", {})
    annotation = new_result.get("annotation", "No annotation provided")

    # Map rule criticality to Security Hub severity
    severity_map = {
        "sagemaker-no-internet": {"Label": "CRITICAL", "Normalized": 90},
        "sagemaker-kms": {"Label": "HIGH", "Normalized": 70},
        "approved-instance-types": {"Label": "MEDIUM", "Normalized": 50},
        "vpc-only-training": {"Label": "HIGH", "Normalized": 70},
        "bedrock-model-restrictions": {"Label": "MEDIUM", "Normalized": 50},
    }

    # Determine severity from rule name
    severity = {"Label": "MEDIUM", "Normalized": 50}
    for key, sev in severity_map.items():
        if key in rule_name.lower():
            severity = sev
            break

    finding = {
        "SchemaVersion": "2018-10-08",
        "Id": f"{rule_name}/{resource_id}/{datetime.utcnow().isoformat()}",
        "ProductArn": f"arn:aws:securityhub:{region}:{account_id}:product/{account_id}/default",
        "GeneratorId": f"aws-config-{rule_name}",
        "AwsAccountId": account_id,
        "Types": ["Software and Configuration Checks/AWS Config Analysis"],
        "CreatedAt": datetime.utcnow().strftime("%Y-%m-%dT%H:%M:%S.%fZ"),
        "UpdatedAt": datetime.utcnow().strftime("%Y-%m-%dT%H:%M:%S.%fZ"),
        "Severity": severity,
        "Title": f"Config Rule Violation: {rule_name}",
        "Description": annotation[:1024],
        "Resources": [
            {
                "Type": resource_type,
                "Id": resource_id,
                "Region": region,
            }
        ],
        "Compliance": {"Status": "FAILED"},
        "Workflow": {"Status": "NEW"},
    }

    securityhub.batch_import_findings(Findings=[finding])

    return {"statusCode": 200, "findingId": finding["Id"]}
```

---

## Integration Points

- **Upstream**: `devops/04` → IAM roles for Lambda evaluators, SSM Automation execution role, Config service-linked role
- **Upstream**: `devops/08` → KMS key ARN used in encryption remediation and as input parameter for conformance pack
- **Downstream**: `devops/06` → Security Hub receives Config compliance findings via the EventBridge→Lambda→ASFF pipeline
- **Downstream**: `enterprise/01` → SCPs enforce preventive controls; Config rules provide detective controls for the same policies
- **Downstream**: `enterprise/04` → Control Tower guardrails complement Config rules; conformance pack aligns with Control Tower detective guardrails
- **Related**: `data/05` → EventBridge ML orchestration can consume Config compliance change events to gate ML workflows
