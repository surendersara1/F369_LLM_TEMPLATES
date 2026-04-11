<!-- Template Version: 1.0 | boto3: 1.35+ -->

# Template DevOps 06 — GuardDuty + Security Hub for ML Workloads

## Purpose
Generate production-ready GuardDuty and Security Hub infrastructure for ML workload threat detection: GuardDuty detector with S3 and Lambda protection features, Security Hub automation rules for auto-resolving low-severity and escalating high-severity findings, Lambda-based automated remediation workflows (isolate notebooks, revoke credentials, quarantine S3 objects), custom Security Hub insights for ML resource types, EventBridge rules for anomalous SageMaker and Bedrock API calls, and compliance framework mapping for SOC 2, HIPAA, and GDPR.

---

## Role Definition

You are an expert AWS security operations engineer specializing in ML workload threat detection and incident response with expertise in:
- Amazon GuardDuty: detector configuration, S3 protection, Lambda protection, finding types, suppression filters, threat intelligence
- AWS Security Hub: automation rules, custom actions, custom insights, ASFF (AWS Security Finding Format), compliance standards
- Automated remediation: Lambda-based incident response for SageMaker notebook isolation, IAM credential revocation, S3 object quarantine
- EventBridge: security event routing, anomalous API call detection patterns for SageMaker and Bedrock services
- Security Hub insights: custom grouping and filtering of findings by ML resource types (SageMaker, Bedrock, Glue)
- Compliance framework mapping: SOC 2 trust service criteria, HIPAA safeguards, GDPR data protection requirements for ML workloads
- Incident response orchestration: Step Functions workflows for multi-step remediation with approval gates

Generate complete, production-deployable security operations infrastructure code.

---

## Context & Inputs

```
PROJECT_NAME:               [REQUIRED]
AWS_REGION:                 [REQUIRED]
AWS_ACCOUNT_ID:             [REQUIRED]
ENV:                        [REQUIRED - dev | stage | prod]

GUARDDUTY_FEATURES:         [OPTIONAL: s3_protection,lambda_protection]
                            Comma-separated list of GuardDuty protection features:
                            - s3_protection: monitor S3 data events for ML data buckets
                            - lambda_protection: monitor Lambda network activity for ML functions
                            - all (enables both)

AUTOMATION_RULES:           [OPTIONAL: default]
                            JSON list of Security Hub automation rule configurations:
                            [
                                {
                                    "name": "auto-resolve-low",
                                    "description": "Auto-resolve LOW severity findings for ML resources",
                                    "criteria_severity": ["LOW", "INFORMATIONAL"],
                                    "action": "SUPPRESSED"
                                },
                                {
                                    "name": "escalate-high-critical",
                                    "description": "Escalate HIGH/CRITICAL findings with SNS notification",
                                    "criteria_severity": ["HIGH", "CRITICAL"],
                                    "action": "NEW"
                                }
                            ]
                            Use "default" to deploy both auto-resolve and escalate rules.

CUSTOM_INSIGHTS:            [OPTIONAL: default]
                            JSON list of Security Hub custom insight configurations:
                            [
                                {
                                    "name": "ml-findings-by-resource",
                                    "group_by": "ResourceType",
                                    "filters": {
                                        "resource_types": [
                                            "AwsSageMakerNotebookInstance",
                                            "AwsSageMakerEndpoint"
                                        ]
                                    }
                                },
                                {
                                    "name": "ml-findings-by-severity",
                                    "group_by": "SeverityLabel",
                                    "filters": {
                                        "resource_types": [
                                            "AwsSageMakerNotebookInstance",
                                            "AwsSageMakerEndpoint"
                                        ]
                                    }
                                }
                            ]
                            Use "default" to deploy both ML resource insights.

REMEDIATION_ACTIONS:        [OPTIONAL: isolate_notebook,revoke_credentials,quarantine_s3]
                            Comma-separated list of automated remediation actions:
                            - isolate_notebook: stop and isolate compromised SageMaker notebooks
                            - revoke_credentials: deactivate IAM access keys for compromised roles
                            - quarantine_s3: move suspicious S3 objects to quarantine bucket
                            - all (enables all three)

COMPLIANCE_FRAMEWORKS:      [OPTIONAL: soc2,hipaa]
                            Comma-separated list of compliance frameworks to map:
                            - soc2: SOC 2 Type II trust service criteria
                            - hipaa: HIPAA security and privacy safeguards
                            - gdpr: GDPR data protection requirements
                            - all (maps all three)

SNS_TOPIC_ARN:              [OPTIONAL - existing SNS topic for security notifications]
QUARANTINE_BUCKET:          [OPTIONAL - S3 bucket for quarantined objects, created if not provided]
```

---

## Task

Generate complete GuardDuty + Security Hub infrastructure:

```
{PROJECT_NAME}-security-hub-ml/
├── guardduty/
│   ├── deploy_detector.py             # Enable GuardDuty with S3+Lambda protection
│   └── config.py                      # Central configuration
├── securityhub/
│   ├── deploy_automation_rules.py     # Create automation rules (auto-resolve/escalate)
│   ├── deploy_insights.py             # Create custom insights for ML resources
│   └── enable_standards.py            # Enable compliance standards
├── remediation/
│   ├── deploy_remediation.py          # Deploy remediation Lambda + EventBridge triggers
│   ├── handlers/
│   │   ├── isolate_notebook/
│   │   │   └── handler.py             # Lambda: stop notebook, remove network access
│   │   ├── revoke_credentials/
│   │   │   └── handler.py             # Lambda: deactivate IAM access keys
│   │   └── quarantine_s3/
│   │       └── handler.py             # Lambda: move S3 objects to quarantine bucket
│   └── remediation_role.py            # IAM role for remediation Lambdas
├── eventbridge/
│   ├── deploy_rules.py                # EventBridge rules for GuardDuty + anomalous API calls
│   └── anomaly_patterns.py            # Event patterns for SageMaker/Bedrock anomalies
├── compliance/
│   └── framework_mapping.py           # Map findings to SOC 2/HIPAA/GDPR controls
├── cleanup/
│   └── delete_resources.py            # Remove all security resources
├── run_setup.py                       # CLI orchestrator
└── requirements.txt
```

**config.py**: Central configuration dataclass with all parameters. Load from environment variables or CLI args. Validate required fields. Construct resource names using `{PROJECT_NAME}-{component}-{ENV}` convention. Parse GUARDDUTY_FEATURES, REMEDIATION_ACTIONS, and COMPLIANCE_FRAMEWORKS into lists.

**deploy_detector.py**: Enable GuardDuty detector with ML-relevant protection features:
- Call `guardduty.create_detector()` with `Enable=True` and `Features` list.
- If `s3_protection` in GUARDDUTY_FEATURES: include `{"Name": "S3_DATA_EVENTS", "Status": "ENABLED"}` in Features.
- If `lambda_protection` in GUARDDUTY_FEATURES: include `{"Name": "LAMBDA_NETWORK_LOGS", "Status": "ENABLED"}` in Features.
- Include `FindingPublishingFrequency` set to `FIFTEEN_MINUTES` for prod, `ONE_HOUR` for dev/stage.
- Tag detector with Project, Environment, ManagedBy.
- Check for existing detector using `guardduty.list_detectors()` before creating; if exists, update with `guardduty.update_detector()`.

**deploy_automation_rules.py**: Create Security Hub automation rules:
- **Auto-resolve low-severity rule**: Call `securityhub.create_automation_rule()` with:
  - `RuleName`: `{PROJECT_NAME}-auto-resolve-low-{ENV}`
  - `RuleOrder`: 100
  - `Criteria`: filter on `SeverityLabel` = `LOW` or `INFORMATIONAL`, and `ResourceType` containing SageMaker/Bedrock resource types
  - `Actions`: set `Workflow.Status` to `SUPPRESSED`, add `Note` with auto-resolution reason
- **Escalate high-severity rule**: Call `securityhub.create_automation_rule()` with:
  - `RuleName`: `{PROJECT_NAME}-escalate-high-{ENV}`
  - `RuleOrder`: 50
  - `Criteria`: filter on `SeverityLabel` = `HIGH` or `CRITICAL`
  - `Actions`: set `Workflow.Status` to `NEW`, set `Note` with escalation message
  - `IsTerminal`: True (stop processing further rules)

**deploy_insights.py**: Create Security Hub custom insights for ML resources:
- **ML findings by resource type**: Call `securityhub.create_insight()` with:
  - `Name`: `{PROJECT_NAME}-ml-findings-by-resource-{ENV}`
  - `Filters`: `ResourceType` matching `AwsSageMakerNotebookInstance`, `AwsSageMakerEndpoint`, `AwsSageMakerEndpointConfig`
  - `GroupByAttribute`: `ResourceType`
- **ML findings by severity**: Call `securityhub.create_insight()` with:
  - `Name`: `{PROJECT_NAME}-ml-findings-by-severity-{ENV}`
  - `Filters`: `ResourceType` matching ML resource types
  - `GroupByAttribute`: `SeverityLabel`
- **ML findings by account** (for multi-account): Call `securityhub.create_insight()` with:
  - `Name`: `{PROJECT_NAME}-ml-findings-by-account-{ENV}`
  - `Filters`: `ResourceType` matching ML resource types
  - `GroupByAttribute`: `AwsAccountId`

**enable_standards.py**: Enable Security Hub compliance standards:
- Call `securityhub.batch_enable_standards()` with ARNs for selected COMPLIANCE_FRAMEWORKS:
  - SOC 2: `arn:aws:securityhub:{region}::standards/service-managed-aws-control-tower/v/1.0.0`
  - HIPAA: `arn:aws:securityhub:{region}::standards/hipaa-security/v/1.0.0`
  - CIS: `arn:aws:securityhub:{region}::standards/cis-aws-foundations-benchmark/v/3.0.0`
- Verify Security Hub is enabled using `securityhub.describe_hub()` before enabling standards.

**isolate_notebook/handler.py**: Lambda remediation function that:
- Receives Security Hub finding event via EventBridge
- Extracts SageMaker notebook instance name from finding `Resources[].Id`
- Calls `sagemaker.stop_notebook_instance()` to stop the notebook
- Calls `sagemaker.update_notebook_instance()` with `DirectInternetAccess='Disabled'` (if in VPC)
- Tags the notebook with `SecurityIncident=true` and `IsolatedAt={timestamp}`
- Publishes remediation result to SNS topic
- Logs remediation action to CloudWatch with finding ID

**revoke_credentials/handler.py**: Lambda remediation function that:
- Receives Security Hub finding event via EventBridge
- Extracts IAM principal ARN from finding `Resources[].Id`
- Lists access keys using `iam.list_access_keys()` for the affected user/role
- Deactivates all access keys using `iam.update_access_key()` with `Status='Inactive'`
- Attaches a deny-all inline policy using `iam.put_user_policy()` as an immediate block
- Publishes remediation result to SNS topic
- Logs all revoked keys and policy changes to CloudWatch

**quarantine_s3/handler.py**: Lambda remediation function that:
- Receives Security Hub finding event via EventBridge
- Extracts S3 bucket and object key from finding `Resources[].Id`
- Copies the object to QUARANTINE_BUCKET with metadata preserving original location
- Deletes the object from the source bucket using `s3.delete_object()`
- Adds a tag `QuarantineReason={finding_type}` to the quarantined copy
- Publishes remediation result to SNS topic

**remediation_role.py**: Create IAM role for remediation Lambda functions:
- Trust policy for `lambda.amazonaws.com`
- Permissions: `sagemaker:StopNotebookInstance`, `sagemaker:UpdateNotebookInstance`, `sagemaker:DescribeNotebookInstance`, `sagemaker:AddTags`
- Permissions: `iam:ListAccessKeys`, `iam:UpdateAccessKey`, `iam:PutUserPolicy`
- Permissions: `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject`, `s3:PutObjectTagging`
- Permissions: `sns:Publish` for notification topic
- Permissions: `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents`
- Scoped to project resources using tag conditions where possible

**deploy_rules.py**: Create EventBridge rules for security events:
- **GuardDuty finding rule**: Match `source: aws.guardduty`, `detail-type: GuardDuty Finding`, filter on ML-related finding types. Target: remediation Lambda functions based on finding type.
- **Anomalous SageMaker API rule**: Match `source: aws.cloudtrail`, `detail-type: AWS API Call via CloudTrail`, filter on `detail.eventSource: sagemaker.amazonaws.com` with anomalous actions (`CreatePresignedNotebookInstanceUrl`, `CreateEndpoint` from unusual IPs). Target: SNS notification + remediation Lambda.
- **Anomalous Bedrock API rule**: Match `source: aws.cloudtrail`, `detail-type: AWS API Call via CloudTrail`, filter on `detail.eventSource: bedrock.amazonaws.com` with high-volume `InvokeModel` or `InvokeModelWithResponseStream` calls. Target: SNS notification.
- Create SNS topic if SNS_TOPIC_ARN not provided.

**anomaly_patterns.py**: Define EventBridge event patterns:
- Pattern for GuardDuty findings related to ML resources (S3 data exfiltration, Lambda network anomalies)
- Pattern for CloudTrail anomalous SageMaker API calls (unusual `CreatePresignedNotebookInstanceUrl`, `CreateEndpoint`, `DeleteModel`)
- Pattern for CloudTrail anomalous Bedrock API calls (high-frequency `InvokeModel`, `InvokeModelWithResponseStream`, `CreateModelCustomizationJob`)
- Each pattern as a reusable JSON dict for EventBridge rule creation

**framework_mapping.py**: Map Security Hub findings to compliance frameworks:
- SOC 2 mapping: map finding types to CC6 (Logical and Physical Access Controls), CC7 (System Operations), CC8 (Change Management) trust service criteria
- HIPAA mapping: map finding types to §164.312 (Technical Safeguards), §164.308 (Administrative Safeguards)
- GDPR mapping: map finding types to Article 32 (Security of Processing), Article 33 (Notification of Breach)
- Generate compliance report JSON grouping findings by framework control
- Output format compatible with Security Hub custom actions for compliance dashboards

**delete_resources.py**: Clean up all resources: delete GuardDuty detector (or disable features), delete Security Hub automation rules, delete custom insights, delete EventBridge rules, delete Lambda functions, delete SNS topics, delete IAM roles.

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**GuardDuty Detector:** Only one detector per account per region. Check for existing detectors before creating. Use `update_detector()` to modify features on existing detectors. S3 protection monitors S3 data events — ensure CloudTrail S3 data event logging is enabled. Lambda protection monitors network activity logs — no additional configuration required.

**Security Hub:** Security Hub must be enabled in the target region before creating automation rules or insights. Use `securityhub.describe_hub()` to verify. Automation rules are limited to 100 per account. Rule order determines processing sequence — lower numbers execute first. `IsTerminal=True` stops further rule processing for matched findings.

**Automation Rules:** Criteria support `EQUALS`, `NOT_EQUALS`, `CONTAINS`, `NOT_CONTAINS`, `PREFIX`, `PREFIX_NOT_EQUALS` comparisons. Actions can update `Workflow.Status`, `Severity.Label`, `Note`, `VerificationState`. Rules apply to new and updated findings — not retroactively to existing findings.

**Custom Insights:** Maximum 100 custom insights per account. `GroupByAttribute` must be a valid ASFF field (e.g., `ResourceType`, `SeverityLabel`, `AwsAccountId`, `ComplianceStatus`). Filters use the same ASFF field syntax as automation rules.

**Remediation:** Lambda remediation functions must complete within 5 minutes. Implement idempotent remediation — running on an already-remediated resource should be a no-op. Log all remediation actions for audit trail. Include error handling for resources that are already stopped/deleted.

**ASFF Compliance:** All custom findings must use `SchemaVersion = "2018-10-08"`. Use `ProductArn = "arn:aws:securityhub:{region}:{account}:product/{account}/default"` for custom findings. Severity labels: `INFORMATIONAL`, `LOW`, `MEDIUM`, `HIGH`, `CRITICAL`.

**Naming:** All resources follow `{PROJECT_NAME}-{component}-{ENV}` convention:
- GuardDuty: detector tagged with `{PROJECT_NAME}-guardduty-{ENV}`
- Automation rules: `{PROJECT_NAME}-securityhub-{rule-name}-{ENV}`
- Custom insights: `{PROJECT_NAME}-insight-{insight-name}-{ENV}`
- Lambda remediation: `{PROJECT_NAME}-remediate-{action}-{ENV}`
- EventBridge rules: `{PROJECT_NAME}-security-{event-type}-{ENV}`

**Cost:** GuardDuty: per-volume pricing for CloudTrail events, S3 data events, Lambda network logs. Security Hub: $0.0010 per finding ingestion event (first 10,000/month/account free). Lambda remediation: standard Lambda pricing. Budget for ~5,000 findings/month in dev, ~50,000 in prod.

---

## Code Scaffolding Hints

**Enable GuardDuty detector with S3 and Lambda protection:**
```python
import boto3

guardduty = boto3.client("guardduty", region_name=AWS_REGION)

# Check for existing detector
existing = guardduty.list_detectors()
if existing["DetectorIds"]:
    detector_id = existing["DetectorIds"][0]
    # Update existing detector with ML protection features
    guardduty.update_detector(
        DetectorId=detector_id,
        Enable=True,
        FindingPublishingFrequency="FIFTEEN_MINUTES",  # FIFTEEN_MINUTES | ONE_HOUR | SIX_HOURS
        Features=[
            {"Name": "S3_DATA_EVENTS", "Status": "ENABLED"},
            {"Name": "LAMBDA_NETWORK_LOGS", "Status": "ENABLED"},
        ],
    )
else:
    # Create new detector
    response = guardduty.create_detector(
        Enable=True,
        FindingPublishingFrequency="FIFTEEN_MINUTES",
        Features=[
            {"Name": "S3_DATA_EVENTS", "Status": "ENABLED"},
            {"Name": "LAMBDA_NETWORK_LOGS", "Status": "ENABLED"},
        ],
        Tags={
            "Project": PROJECT_NAME,
            "Environment": ENV,
            "ManagedBy": "security-hub-ml",
        },
    )
    detector_id = response["DetectorId"]
```

**Create Security Hub automation rule (auto-resolve low severity):**
```python
securityhub = boto3.client("securityhub", region_name=AWS_REGION)

securityhub.create_automation_rule(
    RuleName=f"{PROJECT_NAME}-securityhub-auto-resolve-low-{ENV}",
    Description="Auto-resolve LOW and INFORMATIONAL severity findings for ML resources",
    RuleOrder=100,
    RuleStatus="ENABLED",
    Criteria={
        "SeverityLabel": [
            {"Value": "LOW", "Comparison": "EQUALS"},
            {"Value": "INFORMATIONAL", "Comparison": "EQUALS"},
        ],
        "ResourceType": [
            {"Value": "AwsSageMaker", "Comparison": "PREFIX"},
        ],
    },
    Actions=[
        {
            "Type": "FINDING_FIELDS_UPDATE",
            "FindingFieldsUpdate": {
                "Workflow": {"Status": "SUPPRESSED"},
                "Note": {
                    "Text": f"Auto-resolved by {PROJECT_NAME} automation rule — low severity ML finding",
                    "UpdatedBy": f"{PROJECT_NAME}-automation",
                },
            },
        }
    ],
)
```

**Create Security Hub automation rule (escalate high/critical):**
```python
securityhub.create_automation_rule(
    RuleName=f"{PROJECT_NAME}-securityhub-escalate-high-{ENV}",
    Description="Escalate HIGH and CRITICAL severity findings for immediate response",
    RuleOrder=50,
    RuleStatus="ENABLED",
    IsTerminal=True,
    Criteria={
        "SeverityLabel": [
            {"Value": "HIGH", "Comparison": "EQUALS"},
            {"Value": "CRITICAL", "Comparison": "EQUALS"},
        ],
    },
    Actions=[
        {
            "Type": "FINDING_FIELDS_UPDATE",
            "FindingFieldsUpdate": {
                "Workflow": {"Status": "NEW"},
                "Note": {
                    "Text": f"ESCALATED by {PROJECT_NAME} — HIGH/CRITICAL finding requires immediate investigation",
                    "UpdatedBy": f"{PROJECT_NAME}-automation",
                },
                "Severity": {"Label": "CRITICAL"},
            },
        }
    ],
)
```

**Create Security Hub custom insight for ML resources:**
```python
# Insight: ML findings grouped by resource type
securityhub.create_insight(
    Name=f"{PROJECT_NAME}-insight-ml-findings-by-resource-{ENV}",
    Filters={
        "ResourceType": [
            {"Value": "AwsSageMakerNotebookInstance", "Comparison": "EQUALS"},
            {"Value": "AwsSageMakerEndpoint", "Comparison": "EQUALS"},
            {"Value": "AwsSageMakerEndpointConfig", "Comparison": "EQUALS"},
        ],
    },
    GroupByAttribute="ResourceType",
)

# Insight: ML findings grouped by severity
securityhub.create_insight(
    Name=f"{PROJECT_NAME}-insight-ml-findings-by-severity-{ENV}",
    Filters={
        "ResourceType": [
            {"Value": "AwsSageMakerNotebookInstance", "Comparison": "EQUALS"},
            {"Value": "AwsSageMakerEndpoint", "Comparison": "EQUALS"},
        ],
    },
    GroupByAttribute="SeverityLabel",
)

# Insight: ML findings grouped by account (multi-account)
securityhub.create_insight(
    Name=f"{PROJECT_NAME}-insight-ml-findings-by-account-{ENV}",
    Filters={
        "ResourceType": [
            {"Value": "AwsSageMaker", "Comparison": "PREFIX"},
        ],
    },
    GroupByAttribute="AwsAccountId",
)
```

**Lambda remediation function — isolate SageMaker notebook:**
```python
import json
import boto3
import logging
from datetime import datetime

logger = logging.getLogger()
logger.setLevel(logging.INFO)

sagemaker = boto3.client("sagemaker")
sns = boto3.client("sns")

SNS_TOPIC_ARN = os.environ.get("SNS_TOPIC_ARN", "")

def lambda_handler(event, context):
    """Isolate a compromised SageMaker notebook instance."""
    detail = event.get("detail", {})
    findings = detail.get("findings", [event.get("detail", {})])

    for finding in findings:
        resources = finding.get("Resources", [])
        finding_id = finding.get("Id", "unknown")
        severity = finding.get("Severity", {}).get("Label", "UNKNOWN")

        for resource in resources:
            resource_id = resource.get("Id", "")
            resource_type = resource.get("Type", "")

            if "NotebookInstance" not in resource_type and "NotebookInstance" not in resource_id:
                continue

            # Extract notebook instance name from ARN
            notebook_name = resource_id.split("/")[-1] if "/" in resource_id else resource_id

            try:
                # Check current status
                desc = sagemaker.describe_notebook_instance(
                    NotebookInstanceName=notebook_name
                )
                status = desc.get("NotebookInstanceStatus", "")

                if status == "InService":
                    # Stop the notebook instance
                    sagemaker.stop_notebook_instance(
                        NotebookInstanceName=notebook_name
                    )
                    logger.info(f"Stopped notebook: {notebook_name}")

                # Tag with security incident metadata
                notebook_arn = desc.get("NotebookInstanceArn", "")
                sagemaker.add_tags(
                    ResourceArn=notebook_arn,
                    Tags=[
                        {"Key": "SecurityIncident", "Value": "true"},
                        {"Key": "IsolatedAt", "Value": datetime.utcnow().isoformat()},
                        {"Key": "FindingId", "Value": finding_id[:255]},
                        {"Key": "Severity", "Value": severity},
                    ],
                )

                # Notify via SNS
                if SNS_TOPIC_ARN:
                    sns.publish(
                        TopicArn=SNS_TOPIC_ARN,
                        Subject=f"[{severity}] Notebook Isolated: {notebook_name}",
                        Message=json.dumps({
                            "action": "isolate_notebook",
                            "notebook": notebook_name,
                            "finding_id": finding_id,
                            "severity": severity,
                            "timestamp": datetime.utcnow().isoformat(),
                        }, indent=2),
                    )

            except sagemaker.exceptions.ClientError as e:
                logger.error(f"Failed to isolate {notebook_name}: {e}")
                raise

    return {"statusCode": 200, "action": "isolate_notebook"}
```

**Lambda remediation function — revoke IAM credentials:**
```python
import json
import boto3
import logging
from datetime import datetime

logger = logging.getLogger()
logger.setLevel(logging.INFO)

iam = boto3.client("iam")
sns = boto3.client("sns")

SNS_TOPIC_ARN = os.environ.get("SNS_TOPIC_ARN", "")

def lambda_handler(event, context):
    """Revoke IAM credentials for a compromised principal."""
    detail = event.get("detail", {})
    findings = detail.get("findings", [event.get("detail", {})])

    for finding in findings:
        resources = finding.get("Resources", [])
        finding_id = finding.get("Id", "unknown")

        for resource in resources:
            resource_id = resource.get("Id", "")

            # Extract IAM user name from ARN
            if ":user/" in resource_id:
                user_name = resource_id.split(":user/")[-1]
            else:
                logger.info(f"Skipping non-user resource: {resource_id}")
                continue

            try:
                # List and deactivate all access keys
                keys = iam.list_access_keys(UserName=user_name)
                revoked_keys = []
                for key_meta in keys.get("AccessKeyMetadata", []):
                    key_id = key_meta["AccessKeyId"]
                    if key_meta["Status"] == "Active":
                        iam.update_access_key(
                            UserName=user_name,
                            AccessKeyId=key_id,
                            Status="Inactive",
                        )
                        revoked_keys.append(key_id)
                        logger.info(f"Deactivated key {key_id} for user {user_name}")

                # Attach deny-all inline policy as immediate block
                deny_policy = {
                    "Version": "2012-10-17",
                    "Statement": [
                        {
                            "Sid": "SecurityIncidentDenyAll",
                            "Effect": "Deny",
                            "Action": "*",
                            "Resource": "*",
                        }
                    ],
                }
                iam.put_user_policy(
                    UserName=user_name,
                    PolicyName="SecurityIncident-DenyAll",
                    PolicyDocument=json.dumps(deny_policy),
                )
                logger.info(f"Attached deny-all policy to user {user_name}")

                # Notify via SNS
                if SNS_TOPIC_ARN:
                    sns.publish(
                        TopicArn=SNS_TOPIC_ARN,
                        Subject=f"[CRITICAL] Credentials Revoked: {user_name}",
                        Message=json.dumps({
                            "action": "revoke_credentials",
                            "user": user_name,
                            "revoked_keys": revoked_keys,
                            "deny_policy_attached": True,
                            "finding_id": finding_id,
                            "timestamp": datetime.utcnow().isoformat(),
                        }, indent=2),
                    )

            except iam.exceptions.NoSuchEntityException:
                logger.warning(f"IAM user not found: {user_name}")
            except Exception as e:
                logger.error(f"Failed to revoke credentials for {user_name}: {e}")
                raise

    return {"statusCode": 200, "action": "revoke_credentials"}
```

**Lambda remediation function — quarantine S3 objects:**
```python
import json
import boto3
import logging
import os
from datetime import datetime

logger = logging.getLogger()
logger.setLevel(logging.INFO)

s3 = boto3.client("s3")
sns = boto3.client("sns")

QUARANTINE_BUCKET = os.environ.get("QUARANTINE_BUCKET", "")
SNS_TOPIC_ARN = os.environ.get("SNS_TOPIC_ARN", "")

def lambda_handler(event, context):
    """Quarantine suspicious S3 objects by moving to quarantine bucket."""
    detail = event.get("detail", {})
    findings = detail.get("findings", [event.get("detail", {})])

    for finding in findings:
        resources = finding.get("Resources", [])
        finding_id = finding.get("Id", "unknown")
        finding_type = finding.get("Types", ["Unknown"])[0]

        for resource in resources:
            resource_id = resource.get("Id", "")

            # Parse S3 ARN: arn:aws:s3:::bucket/key
            if not resource_id.startswith("arn:aws:s3:::"):
                continue

            s3_path = resource_id.replace("arn:aws:s3:::", "")
            if "/" in s3_path:
                source_bucket = s3_path.split("/")[0]
                object_key = "/".join(s3_path.split("/")[1:])
            else:
                logger.info(f"Skipping bucket-level resource: {resource_id}")
                continue

            try:
                # Copy to quarantine bucket with metadata
                quarantine_key = f"quarantine/{datetime.utcnow().strftime('%Y/%m/%d')}/{source_bucket}/{object_key}"
                s3.copy_object(
                    Bucket=QUARANTINE_BUCKET,
                    Key=quarantine_key,
                    CopySource={"Bucket": source_bucket, "Key": object_key},
                    Metadata={
                        "original-bucket": source_bucket,
                        "original-key": object_key,
                        "quarantine-reason": finding_type[:255],
                        "finding-id": finding_id[:255],
                        "quarantined-at": datetime.utcnow().isoformat(),
                    },
                    MetadataDirective="REPLACE",
                )

                # Tag the quarantined copy
                s3.put_object_tagging(
                    Bucket=QUARANTINE_BUCKET,
                    Key=quarantine_key,
                    Tagging={
                        "TagSet": [
                            {"Key": "QuarantineReason", "Value": finding_type[:255]},
                            {"Key": "OriginalBucket", "Value": source_bucket},
                            {"Key": "FindingId", "Value": finding_id[:128]},
                        ]
                    },
                )

                # Delete from source bucket
                s3.delete_object(Bucket=source_bucket, Key=object_key)
                logger.info(f"Quarantined {source_bucket}/{object_key} → {QUARANTINE_BUCKET}/{quarantine_key}")

                # Notify via SNS
                if SNS_TOPIC_ARN:
                    sns.publish(
                        TopicArn=SNS_TOPIC_ARN,
                        Subject=f"[HIGH] S3 Object Quarantined: {source_bucket}/{object_key}",
                        Message=json.dumps({
                            "action": "quarantine_s3",
                            "source_bucket": source_bucket,
                            "object_key": object_key,
                            "quarantine_bucket": QUARANTINE_BUCKET,
                            "quarantine_key": quarantine_key,
                            "finding_type": finding_type,
                            "finding_id": finding_id,
                            "timestamp": datetime.utcnow().isoformat(),
                        }, indent=2),
                    )

            except Exception as e:
                logger.error(f"Failed to quarantine {source_bucket}/{object_key}: {e}")
                raise

    return {"statusCode": 200, "action": "quarantine_s3"}
```

**EventBridge rule for GuardDuty ML findings:**
```python
import json
import boto3

events_client = boto3.client("events", region_name=AWS_REGION)
sns_client = boto3.client("sns", region_name=AWS_REGION)

# Create SNS topic if not provided
if not SNS_TOPIC_ARN:
    topic_response = sns_client.create_topic(
        Name=f"{PROJECT_NAME}-security-alerts-{ENV}",
        Tags=[
            {"Key": "Project", "Value": PROJECT_NAME},
            {"Key": "Environment", "Value": ENV},
        ],
    )
    sns_topic_arn = topic_response["TopicArn"]
else:
    sns_topic_arn = SNS_TOPIC_ARN

# GuardDuty finding rule — triggers remediation for ML-related findings
events_client.put_rule(
    Name=f"{PROJECT_NAME}-security-guardduty-ml-{ENV}",
    Description="Route GuardDuty findings related to ML resources to remediation",
    EventPattern=json.dumps({
        "source": ["aws.guardduty"],
        "detail-type": ["GuardDuty Finding"],
        "detail": {
            "severity": [{"numeric": [">=", 4]}],
        },
    }),
    State="ENABLED",
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)

# Add SNS + Lambda targets
events_client.put_targets(
    Rule=f"{PROJECT_NAME}-security-guardduty-ml-{ENV}",
    Targets=[
        {
            "Id": "sns-security-alert",
            "Arn": sns_topic_arn,
        },
        {
            "Id": "remediation-lambda",
            "Arn": f"arn:aws:lambda:{AWS_REGION}:{AWS_ACCOUNT_ID}:function:{PROJECT_NAME}-remediate-isolate-notebook-{ENV}",
        },
    ],
)
```

**EventBridge rule for anomalous SageMaker/Bedrock API calls:**
```python
# Anomalous SageMaker API calls via CloudTrail
events_client.put_rule(
    Name=f"{PROJECT_NAME}-security-sagemaker-anomaly-{ENV}",
    Description="Detect anomalous SageMaker API calls (presigned URLs, endpoint creation)",
    EventPattern=json.dumps({
        "source": ["aws.sagemaker"],
        "detail-type": ["AWS API Call via CloudTrail"],
        "detail": {
            "eventSource": ["sagemaker.amazonaws.com"],
            "eventName": [
                "CreatePresignedNotebookInstanceUrl",
                "CreateEndpoint",
                "DeleteModel",
                "DeleteEndpoint",
                "StopTrainingJob",
            ],
        },
    }),
    State="ENABLED",
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)

events_client.put_targets(
    Rule=f"{PROJECT_NAME}-security-sagemaker-anomaly-{ENV}",
    Targets=[
        {"Id": "sns-sagemaker-anomaly", "Arn": sns_topic_arn},
    ],
)

# Anomalous Bedrock API calls via CloudTrail
events_client.put_rule(
    Name=f"{PROJECT_NAME}-security-bedrock-anomaly-{ENV}",
    Description="Detect anomalous Bedrock API calls (high-volume invocations, customization jobs)",
    EventPattern=json.dumps({
        "source": ["aws.bedrock"],
        "detail-type": ["AWS API Call via CloudTrail"],
        "detail": {
            "eventSource": ["bedrock.amazonaws.com"],
            "eventName": [
                "InvokeModel",
                "InvokeModelWithResponseStream",
                "CreateModelCustomizationJob",
                "DeleteCustomModel",
                "CreateProvisionedModelThroughput",
            ],
        },
    }),
    State="ENABLED",
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)

events_client.put_targets(
    Rule=f"{PROJECT_NAME}-security-bedrock-anomaly-{ENV}",
    Targets=[
        {"Id": "sns-bedrock-anomaly", "Arn": sns_topic_arn},
    ],
)
```

**Compliance framework mapping:**
```python
import json
from datetime import datetime

# SOC 2 Trust Service Criteria mapping
SOC2_MAPPING = {
    "Software and Configuration Checks": "CC6.1 — Logical Access Security",
    "TTPs/Initial Access": "CC6.6 — Restriction of Logical Access",
    "Effects/Data Exfiltration": "CC6.7 — Restriction of Data Transmission",
    "Unusual Behaviors/User": "CC7.2 — Monitoring of System Components",
    "TTPs/Credential Access": "CC6.1 — Logical Access Security",
    "TTPs/Persistence": "CC8.1 — Change Management",
}

# HIPAA Security Rule mapping
HIPAA_MAPPING = {
    "Software and Configuration Checks": "§164.312(a)(1) — Access Control",
    "TTPs/Initial Access": "§164.312(a)(1) — Access Control",
    "Effects/Data Exfiltration": "§164.312(e)(1) — Transmission Security",
    "Unusual Behaviors/User": "§164.312(b) — Audit Controls",
    "TTPs/Credential Access": "§164.308(a)(5)(ii)(C) — Log-in Monitoring",
    "TTPs/Persistence": "§164.308(a)(5)(ii)(D) — Password Management",
}

# GDPR mapping
GDPR_MAPPING = {
    "Software and Configuration Checks": "Article 32 — Security of Processing",
    "TTPs/Initial Access": "Article 32 — Security of Processing",
    "Effects/Data Exfiltration": "Article 33 — Notification of Breach to Authority",
    "Unusual Behaviors/User": "Article 32 — Security of Processing",
    "TTPs/Credential Access": "Article 32 — Security of Processing",
    "TTPs/Persistence": "Article 32 — Security of Processing",
}


def map_finding_to_frameworks(finding, frameworks):
    """Map a Security Hub finding to compliance framework controls."""
    finding_types = finding.get("Types", [])
    mapped = {}

    framework_maps = {
        "soc2": SOC2_MAPPING,
        "hipaa": HIPAA_MAPPING,
        "gdpr": GDPR_MAPPING,
    }

    for framework in frameworks:
        mapping = framework_maps.get(framework, {})
        controls = []
        for finding_type in finding_types:
            for pattern, control in mapping.items():
                if pattern in finding_type:
                    controls.append(control)
        mapped[framework] = list(set(controls)) if controls else ["No mapping available"]

    return mapped


def generate_compliance_report(findings, frameworks):
    """Generate compliance report grouping findings by framework control."""
    report = {
        "generated_at": datetime.utcnow().isoformat(),
        "total_findings": len(findings),
        "frameworks": {},
    }

    for framework in frameworks:
        report["frameworks"][framework] = {}

    for finding in findings:
        mapping = map_finding_to_frameworks(finding, frameworks)
        for framework, controls in mapping.items():
            for control in controls:
                if control not in report["frameworks"][framework]:
                    report["frameworks"][framework][control] = []
                report["frameworks"][framework][control].append({
                    "finding_id": finding.get("Id", ""),
                    "severity": finding.get("Severity", {}).get("Label", ""),
                    "title": finding.get("Title", ""),
                    "resource": finding.get("Resources", [{}])[0].get("Id", ""),
                })

    return report
```

---

## Integration Points

- **Upstream**: `devops/04` → IAM roles for remediation Lambda functions, EventBridge rule targets, GuardDuty service-linked role
- **Upstream**: `devops/05` → Config compliance findings feed into Security Hub via the EventBridge→Lambda→ASFF pipeline established in devops/05
- **Downstream**: `enterprise/04` → Control Tower guardrails complement GuardDuty threat detection; Security Hub aggregates findings across Control Tower managed accounts
- **Downstream**: `devops/03` → CloudWatch dashboards consume Security Hub finding metrics and GuardDuty severity trends for security operations visibility
- **Related**: `data/05` → EventBridge ML orchestration can consume GuardDuty finding events to gate ML workflows when security incidents are detected
- **Related**: `devops/07` → Macie PII findings feed into the same Security Hub insights and automation rules for unified ML security posture
