<!-- Template Version: 1.0 | boto3: 1.35+ -->

# Template DevOps 07 — Macie PII Detection in Training Data

## Purpose
Generate production-ready Amazon Macie infrastructure for PII detection in ML training data: Macie classification jobs targeting S3 training data buckets with configurable scheduling (one-time or recurring), custom data identifiers for domain-specific sensitive patterns (medical record numbers, internal employee IDs), EventBridge + Step Functions automated remediation workflow (quarantine → notify → sanitize), Lambda-based data sanitization for JSONL and CSV redaction, training job blocking mechanism via S3 bucket policy updates or Lake Formation permission revocation, and Macie findings aggregation queries for PII detection reporting.

---

## Role Definition

You are an expert AWS data privacy engineer and security specialist with expertise in:
- Amazon Macie: classification jobs, managed data identifiers, custom data identifiers, findings analysis, sensitive data discovery
- PII detection patterns: names, addresses, SSNs, credit card numbers, medical record numbers, custom organizational identifiers
- Custom data identifiers: regex patterns, keyword lists, maximum match distance, severity weighting for domain-specific sensitive data
- Automated remediation: EventBridge rules for Macie findings, Step Functions orchestration for multi-step remediation workflows
- Data sanitization: Lambda functions for PII redaction in JSONL and CSV formats, field-level anonymization, tokenization strategies
- S3 security: bucket policies for access blocking, object tagging for quarantine, S3 event notifications
- Lake Formation: permission grants and revocations for governed data access control
- SageMaker integration: blocking training jobs from accessing PII-contaminated data, S3 bucket policy conditions for SageMaker execution roles
- Findings aggregation: Macie statistics API, findings export to Security Hub, compliance reporting

Generate complete, production-deployable data privacy infrastructure code.

---

## Context & Inputs

```
PROJECT_NAME:               [REQUIRED]
AWS_REGION:                 [REQUIRED]
AWS_ACCOUNT_ID:             [REQUIRED]
ENV:                        [REQUIRED - dev | stage | prod]

TARGET_BUCKETS:             [REQUIRED - JSON list of S3 bucket names to scan]
                            Example:
                            [
                                "myai-training-data-prod",
                                "myai-raw-datasets-prod",
                                "myai-feature-store-prod"
                            ]
                            These buckets contain ML training data that must be scanned for PII.

SCAN_SCHEDULE:              [OPTIONAL: one-time]
                            Options: one-time | recurring
                            - one-time: run classification job once
                            - recurring: run on a schedule (daily/weekly/monthly)
                            If recurring, set SCAN_FREQUENCY below.

SCAN_FREQUENCY:             [OPTIONAL: WEEKLY]
                            Options: DAILY | WEEKLY | MONTHLY
                            Only used when SCAN_SCHEDULE=recurring.

CUSTOM_DATA_IDENTIFIERS:    [OPTIONAL: none]
                            JSON list of custom data identifier definitions for domain-specific
                            sensitive data patterns beyond Macie built-in detectors:
                            [
                                {
                                    "name": "medical-record-number",
                                    "description": "Internal MRN format: MRN-XXXXXXXX",
                                    "regex": "MRN-[0-9]{8}",
                                    "keywords": ["medical record", "MRN", "patient"],
                                    "maximumMatchDistance": 50,
                                    "severityLevels": [{"occurrencesThreshold": 1, "severity": "HIGH"}]
                                },
                                {
                                    "name": "employee-id",
                                    "description": "Internal employee ID format: EMP-XXXXX",
                                    "regex": "EMP-[A-Z0-9]{5}",
                                    "keywords": ["employee", "staff", "worker"],
                                    "maximumMatchDistance": 50,
                                    "severityLevels": [{"occurrencesThreshold": 5, "severity": "MEDIUM"}]
                                }
                            ]

REMEDIATION_ACTION:         [OPTIONAL: quarantine]
                            Options: quarantine | sanitize | notify
                            - quarantine: move PII-containing objects to quarantine prefix, block access
                            - sanitize: redact/anonymize PII fields in-place, write sanitized copy
                            - notify: send SNS notification only, no automated data modification
                            Multiple actions execute in order: quarantine → notify → sanitize.

BLOCK_TRAINING_ON_PII:      [OPTIONAL: true]
                            When true, automatically block SageMaker training jobs from accessing
                            buckets where PII is detected until remediation is complete.
                            Mechanism: S3 bucket policy deny statement for SageMaker execution roles
                            or Lake Formation permission revocation.

BLOCKING_MECHANISM:         [OPTIONAL: s3_bucket_policy]
                            Options: s3_bucket_policy | lake_formation
                            - s3_bucket_policy: add Deny statement to bucket policy for SageMaker roles
                            - lake_formation: revoke Lake Formation permissions on affected tables

SNS_TOPIC_ARN:              [OPTIONAL - SNS topic for PII detection notifications]
                            If not provided, a new topic is created:
                            {PROJECT_NAME}-macie-pii-alerts-{ENV}

REDACTION_PATTERNS:         [OPTIONAL: default]
                            JSON mapping of PII types to redaction strategies:
                            {
                                "NAME": "[REDACTED_NAME]",
                                "EMAIL_ADDRESS": "[REDACTED_EMAIL]",
                                "PHONE_NUMBER": "[REDACTED_PHONE]",
                                "AWS_CREDENTIALS": "[REDACTED_CREDENTIALS]",
                                "CREDIT_CARD_NUMBER": "[REDACTED_CC]",
                                "custom:medical-record-number": "[REDACTED_MRN]",
                                "custom:employee-id": "[REDACTED_EMP_ID]"
                            }
                            Use "default" to apply "[REDACTED]" for all PII types.

SAGEMAKER_EXECUTION_ROLE:   [OPTIONAL - SageMaker execution role ARN to block]
                            Used when BLOCK_TRAINING_ON_PII=true with s3_bucket_policy mechanism.
                            If not provided, blocks all SageMaker service principals.
```

---

## Task

Generate complete Macie PII detection and remediation infrastructure:

```
{PROJECT_NAME}-macie-pii/
├── classification/
│   ├── create_classification_job.py       # Create Macie classification job for target buckets
│   ├── create_custom_identifiers.py       # Create custom data identifiers for domain patterns
│   └── get_findings_statistics.py         # Aggregate and report PII findings
├── remediation/
│   ├── step_functions_workflow.py         # Create Step Functions remediation state machine
│   ├── quarantine_handler.py             # Lambda: quarantine PII-containing S3 objects
│   ├── notify_handler.py                # Lambda: send SNS notification with finding details
│   ├── sanitize_handler.py              # Lambda: redact PII from JSONL/CSV files
│   └── eventbridge_rule.py              # EventBridge rule for Macie finding events
├── blocking/
│   ├── block_training_access.py          # Block SageMaker access to PII-contaminated buckets
│   ├── unblock_training_access.py        # Restore access after remediation
│   └── bucket_policy_manager.py          # S3 bucket policy update utilities
├── infrastructure/
│   ├── enable_macie.py                   # Enable Macie and configure account settings
│   ├── create_sns_topic.py               # Create SNS topic for PII alerts
│   └── config.py                         # Central configuration
├── run_setup.py                          # CLI orchestrator
└── requirements.txt
```

**config.py**: Central configuration dataclass with all parameters. Load from environment variables or CLI args. Validate required fields. Construct resource names using `{PROJECT_NAME}-{component}-{ENV}` convention.

**enable_macie.py**: Enable Amazon Macie for the account:
- Call `macie2.enable_macie()` to activate the service
- Configure `macie2.put_classification_export_configuration()` to export discovery results to S3
- Set export bucket to `{PROJECT_NAME}-macie-results-{ENV}`
- Enable automated sensitive data discovery if available
- Tag Macie resources with Project and Environment

**create_classification_job.py**: Create Macie classification job using `macie2.create_classification_job()`:
- Set `jobType` to `ONE_TIME` or `SCHEDULED` based on SCAN_SCHEDULE
- Configure `s3JobDefinition` with `bucketDefinitions` targeting TARGET_BUCKETS
- If SCAN_SCHEDULE=recurring, set `scheduleFrequencyType` to SCAN_FREQUENCY
- Include `managedDataIdentifierSelector` set to `ALL` for comprehensive PII detection
- If CUSTOM_DATA_IDENTIFIERS provided, include `customDataIdentifierIds` referencing created identifiers
- Set `samplingPercentage` to 100 for complete scanning (configurable for large datasets)
- Tag job with `{PROJECT_NAME}-macie-scan-{ENV}`

**create_custom_identifiers.py**: Create custom data identifiers using `macie2.create_custom_data_identifier()`:
- Iterate over CUSTOM_DATA_IDENTIFIERS JSON list
- For each identifier: set `name`, `description`, `regex`, `keywords`, `maximumMatchDistance`
- Configure `severityLevels` for finding severity based on occurrence thresholds
- Return list of custom data identifier IDs for use in classification job
- Tag each identifier with Project and Environment

**get_findings_statistics.py**: Aggregate Macie findings using `macie2.get_finding_statistics()`:
- Query findings grouped by `classificationDetails.result.sensitiveData.category` (PII types)
- Query findings grouped by `resourcesAffected.s3Bucket.name` (per-bucket breakdown)
- Query findings grouped by `severity.description` (severity distribution)
- Generate summary report: total findings, PII types detected, affected buckets, severity breakdown
- Optionally export findings to Security Hub via `macie2.get_findings()` for centralized view

**step_functions_workflow.py**: Create Step Functions state machine for remediation:
- Define state machine with states: `QuarantineObject` → `NotifyOwner` → `SanitizeData` → `UnblockAccess`
- `QuarantineObject`: invoke quarantine_handler Lambda to move object to quarantine prefix
- `NotifyOwner`: invoke notify_handler Lambda to send SNS notification
- `SanitizeData`: invoke sanitize_handler Lambda to redact PII from the file
- `UnblockAccess`: invoke unblock_training_access to restore SageMaker access
- Include error handling with `Catch` blocks and retry with exponential backoff
- Conditional branching based on REMEDIATION_ACTION (skip steps not selected)
- Create state machine using `sfn.create_state_machine()` with `{PROJECT_NAME}-macie-remediation-{ENV}`

**quarantine_handler.py**: Lambda function for quarantining PII-containing objects:
- Receive Macie finding event from Step Functions
- Extract affected S3 bucket and object key from finding
- Copy object to quarantine prefix: `s3://{bucket}/quarantine/{original_key}`
- Add S3 object tag `pii-status=quarantined` and `quarantine-timestamp`
- If BLOCK_TRAINING_ON_PII=true, invoke block_training_access
- Log quarantine action to CloudWatch with finding ID and object details
- Return quarantine status for Step Functions workflow

**notify_handler.py**: Lambda function for SNS notification:
- Receive finding details from Step Functions
- Format notification message with: finding ID, affected bucket/object, PII types detected, severity, recommended action
- Publish to SNS_TOPIC_ARN using `sns.publish()`
- Include deep link to Macie console for the finding
- Return notification status

**sanitize_handler.py**: Lambda function for PII redaction in training data:
- Receive quarantined object details from Step Functions
- Download object from S3 quarantine prefix
- Detect file format (JSONL or CSV) from extension or content
- For JSONL: parse each line as JSON, apply REDACTION_PATTERNS to string fields matching PII types
- For CSV: parse with csv module, apply REDACTION_PATTERNS to columns identified in Macie finding
- Write sanitized file to `s3://{bucket}/sanitized/{original_key}`
- Add S3 object tag `pii-status=sanitized` and `sanitized-timestamp`
- Return sanitization summary: fields redacted, records processed, output path

**eventbridge_rule.py**: Create EventBridge rule for Macie findings:
- Create rule matching `source: "aws.macie"` and `detail-type: "Macie Finding"`
- Filter on `detail.severity.description` for MEDIUM, HIGH, CRITICAL findings
- Set target to Step Functions state machine ARN for automated remediation
- Configure input transformer to pass finding details to state machine
- Create rule using `events.put_rule()` and `events.put_targets()`
- Rule name: `{PROJECT_NAME}-macie-finding-remediation-{ENV}`

**block_training_access.py**: Block SageMaker training access to PII-contaminated buckets:
- If BLOCKING_MECHANISM=s3_bucket_policy:
  - Read current bucket policy using `s3.get_bucket_policy()`
  - Add Deny statement for SageMaker execution role on `s3:GetObject` and `s3:ListBucket`
  - Condition: `StringLike: {"s3:ExistingObjectTag/pii-status": "quarantined"}`
  - Update policy using `s3.put_bucket_policy()`
- If BLOCKING_MECHANISM=lake_formation:
  - Revoke Lake Formation permissions using `lakeformation.revoke_permissions()` for affected tables
  - Record revoked permissions for later restoration
- Log blocking action with timestamp and affected resources

**unblock_training_access.py**: Restore SageMaker access after remediation:
- If BLOCKING_MECHANISM=s3_bucket_policy:
  - Remove the Deny statement added by block_training_access
  - Update policy using `s3.put_bucket_policy()`
- If BLOCKING_MECHANISM=lake_formation:
  - Re-grant Lake Formation permissions using `lakeformation.grant_permissions()`
- Verify access is restored by checking policy/permissions
- Log unblocking action with timestamp

**bucket_policy_manager.py**: Utility functions for S3 bucket policy management:
- `get_current_policy(bucket)`: retrieve and parse current bucket policy JSON
- `add_deny_statement(policy, principal, actions, condition)`: add Deny statement to policy
- `remove_deny_statement(policy, sid)`: remove specific Deny statement by SID
- `update_bucket_policy(bucket, policy)`: write updated policy back to S3
- Statement SID convention: `{PROJECT_NAME}-macie-block-{ENV}`

**create_sns_topic.py**: Create SNS topic for PII alerts:
- Create topic `{PROJECT_NAME}-macie-pii-alerts-{ENV}` using `sns.create_topic()`
- Configure topic policy for Macie and Step Functions publishing
- Tag with Project and Environment
- Return topic ARN for use in other components

**run_setup.py**: CLI orchestrator that runs setup steps in order:
1. Enable Macie for the account
2. Create SNS topic for PII alerts
3. Create custom data identifiers (if configured)
4. Create classification job targeting training data buckets
5. Create Step Functions remediation workflow
6. Create EventBridge rule for Macie findings
7. Print summary with job ID, custom identifier IDs, state machine ARN, and rule name

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**Macie Configuration:** Enable Macie before creating classification jobs. Use `managedDataIdentifierSelector=ALL` for comprehensive PII detection covering names, addresses, SSNs, credit cards, API keys, and 100+ built-in identifiers. Set `samplingPercentage=100` for production scans; use lower values (10-25%) for initial assessment of large datasets. Classification jobs scan S3 objects at the time of job execution — new objects added after job start are not scanned until the next run.

**Custom Data Identifiers:** Regex patterns must follow RE2 syntax (Macie's regex engine). Set `maximumMatchDistance` to control how close keywords must be to the regex match (default 50 characters). Define `severityLevels` with occurrence thresholds — a single PII match may be LOW severity, but 100+ matches in one file should be HIGH. Test custom identifiers against sample data before deploying to production.

**Remediation Workflow:** The Step Functions workflow must be idempotent — re-running on the same finding should not duplicate quarantine copies or notifications. Use S3 object tags (`pii-status`) to track remediation state and prevent duplicate processing. Set Lambda timeout to 15 minutes for sanitization of large files. Configure Step Functions with `TimeoutSeconds` of 3600 (1 hour) for the full workflow.

**Data Sanitization:** The sanitization Lambda must handle JSONL files (one JSON object per line) and CSV files (with headers). For JSONL: iterate fields recursively and replace values matching PII patterns. For CSV: identify columns from Macie finding details and replace cell values. Never modify the original file — always write to a `sanitized/` prefix. Preserve file encoding (UTF-8) and line endings. For files larger than Lambda's /tmp storage (10GB), use streaming processing or S3 Select.

**Training Job Blocking:** When using S3 bucket policy blocking, add a Deny statement with a unique SID (`{PROJECT_NAME}-macie-block-{ENV}`) so it can be cleanly removed after remediation. The Deny statement should target the SageMaker execution role principal and deny `s3:GetObject` on objects tagged `pii-status=quarantined`. When using Lake Formation blocking, revoke `SELECT` permission on affected tables and record the original grants for restoration.

**Security:** Macie service-linked role is created automatically. Lambda execution roles need `macie2:GetFindings`, `s3:GetObject`, `s3:PutObject`, `s3:PutObjectTagging`, `s3:GetBucketPolicy`, `s3:PutBucketPolicy`, `sns:Publish`, and `states:StartExecution`. Use least-privilege IAM policies scoped to TARGET_BUCKETS. Encrypt Macie results bucket with KMS from `devops/08`.

**Cost:** Macie pricing is based on S3 objects evaluated and data inspected (per GB). For large training datasets, use `samplingPercentage` to control costs during initial assessment. Recurring jobs re-scan all objects unless using `S3JobDefinition.scoping` to limit by object age or prefix. Monitor `macie2:ClassificationJobStatus` CloudWatch metrics for job duration and cost tracking.

**Naming:** All resources follow `{PROJECT_NAME}-{component}-{ENV}` convention:
- Classification job: `{PROJECT_NAME}-macie-scan-{ENV}`
- Custom identifier: `{PROJECT_NAME}-macie-{identifier_name}-{ENV}`
- Step Functions: `{PROJECT_NAME}-macie-remediation-{ENV}`
- EventBridge rule: `{PROJECT_NAME}-macie-finding-remediation-{ENV}`
- SNS topic: `{PROJECT_NAME}-macie-pii-alerts-{ENV}`
- Quarantine prefix: `s3://{bucket}/quarantine/`
- Sanitized prefix: `s3://{bucket}/sanitized/`

---

## Code Scaffolding Hints


**Enable Macie and configure export:**
```python
import boto3

macie = boto3.client("macie2", region_name=AWS_REGION)

# Enable Macie for the account
macie.enable_macie(
    status="ENABLED",
)

# Configure classification export to S3
macie.put_classification_export_configuration(
    configuration={
        "s3Destination": {
            "bucketName": f"{PROJECT_NAME}-macie-results-{ENV}",
            "keyPrefix": "macie-findings/",
            "kmsKeyArn": KMS_KEY_ARN,  # From devops/08
        }
    }
)
```

**Create Macie classification job:**
```python
import json

# Build bucket definitions from TARGET_BUCKETS
bucket_definitions = [
    {
        "accountId": AWS_ACCOUNT_ID,
        "buckets": TARGET_BUCKETS,  # List of bucket names
    }
]

# Build custom data identifier list (if configured)
custom_identifier_ids = []  # Populated by create_custom_identifiers.py

# Create classification job
job_response = macie.create_classification_job(
    name=f"{PROJECT_NAME}-macie-scan-{ENV}",
    description=f"PII detection scan for {PROJECT_NAME} ML training data",
    jobType="ONE_TIME" if SCAN_SCHEDULE == "one-time" else "SCHEDULED",
    s3JobDefinition={
        "bucketDefinitions": bucket_definitions,
        "scoping": {
            "includes": {
                "and": [
                    {
                        "simpleScopeTerm": {
                            "comparator": "STARTS_WITH",
                            "key": "OBJECT_EXTENSION",
                            "values": ["jsonl", "csv", "json", "parquet", "txt"],
                        }
                    }
                ]
            }
        },
    },
    managedDataIdentifierSelector="ALL",
    customDataIdentifierIds=custom_identifier_ids,
    samplingPercentage=100,
    scheduleFrequencyType=SCAN_FREQUENCY if SCAN_SCHEDULE == "recurring" else None,
    tags={
        "Project": PROJECT_NAME,
        "Environment": ENV,
    },
)
job_id = job_response["jobId"]
print(f"Classification job created: {job_id}")
```

**Create custom data identifier:**
```python
def create_custom_data_identifier(identifier_config):
    """Create a custom Macie data identifier for domain-specific patterns."""
    response = macie.create_custom_data_identifier(
        name=f"{PROJECT_NAME}-macie-{identifier_config['name']}-{ENV}",
        description=identifier_config.get("description", ""),
        regex=identifier_config["regex"],
        keywords=identifier_config.get("keywords", []),
        maximumMatchDistance=identifier_config.get("maximumMatchDistance", 50),
        severityLevels=identifier_config.get("severityLevels", [
            {"occurrencesThreshold": 1, "severity": "MEDIUM"},
            {"occurrencesThreshold": 10, "severity": "HIGH"},
        ]),
        tags={
            "Project": PROJECT_NAME,
            "Environment": ENV,
        },
    )
    return response["customDataIdentifierId"]

# Create all custom identifiers
custom_identifier_ids = []
for identifier in CUSTOM_DATA_IDENTIFIERS:
    cid = create_custom_data_identifier(identifier)
    custom_identifier_ids.append(cid)
    print(f"Custom identifier created: {identifier['name']} -> {cid}")
```

**Get finding statistics (aggregation queries):**
```python
def get_findings_by_bucket():
    """Get PII finding counts grouped by S3 bucket."""
    response = macie.get_finding_statistics(
        groupBy="resourcesAffected.s3Bucket.name",
        findingCriteria={
            "criterion": {
                "category": {
                    "eq": ["CLASSIFICATION"],
                },
            }
        },
    )
    return response["countsByGroup"]

def get_findings_by_pii_type():
    """Get PII finding counts grouped by sensitive data category."""
    response = macie.get_finding_statistics(
        groupBy="classificationDetails.result.sensitiveData.category",
        findingCriteria={
            "criterion": {
                "category": {
                    "eq": ["CLASSIFICATION"],
                },
            }
        },
    )
    return response["countsByGroup"]

def get_findings_by_severity():
    """Get PII finding counts grouped by severity."""
    response = macie.get_finding_statistics(
        groupBy="severity.description",
        findingCriteria={
            "criterion": {
                "category": {
                    "eq": ["CLASSIFICATION"],
                },
            }
        },
    )
    return response["countsByGroup"]

# Generate summary report
bucket_stats = get_findings_by_bucket()
pii_type_stats = get_findings_by_pii_type()
severity_stats = get_findings_by_severity()

report = {
    "project": PROJECT_NAME,
    "environment": ENV,
    "findings_by_bucket": {g["groupKey"]: g["count"] for g in bucket_stats},
    "findings_by_pii_type": {g["groupKey"]: g["count"] for g in pii_type_stats},
    "findings_by_severity": {g["groupKey"]: g["count"] for g in severity_stats},
    "total_findings": sum(g["count"] for g in severity_stats),
}
print(json.dumps(report, indent=2))
```

**S3 bucket policy update to block SageMaker training access:**
```python
import json

s3 = boto3.client("s3", region_name=AWS_REGION)

def block_training_access(bucket_name, sagemaker_role_arn):
    """Add Deny statement to bucket policy blocking SageMaker access to PII data."""
    deny_sid = f"{PROJECT_NAME}-macie-block-{ENV}"

    # Get current policy
    try:
        current_policy = json.loads(s3.get_bucket_policy(Bucket=bucket_name)["Policy"])
    except s3.exceptions.from_code("NoSuchBucketPolicy"):
        current_policy = {"Version": "2012-10-17", "Statement": []}

    # Remove existing block statement if present (idempotent)
    current_policy["Statement"] = [
        stmt for stmt in current_policy["Statement"]
        if stmt.get("Sid") != deny_sid
    ]

    # Add Deny statement for SageMaker execution role
    deny_statement = {
        "Sid": deny_sid,
        "Effect": "Deny",
        "Principal": {
            "AWS": sagemaker_role_arn or f"arn:aws:iam::{AWS_ACCOUNT_ID}:root",
        },
        "Action": [
            "s3:GetObject",
            "s3:ListBucket",
        ],
        "Resource": [
            f"arn:aws:s3:::{bucket_name}",
            f"arn:aws:s3:::{bucket_name}/*",
        ],
        "Condition": {
            "StringEquals": {
                "s3:ExistingObjectTag/pii-status": "quarantined",
            }
        },
    }
    current_policy["Statement"].append(deny_statement)

    # Update bucket policy
    s3.put_bucket_policy(
        Bucket=bucket_name,
        Policy=json.dumps(current_policy),
    )
    print(f"Training access blocked on {bucket_name} for role {sagemaker_role_arn}")

def unblock_training_access(bucket_name):
    """Remove the Deny statement to restore SageMaker access."""
    deny_sid = f"{PROJECT_NAME}-macie-block-{ENV}"

    try:
        current_policy = json.loads(s3.get_bucket_policy(Bucket=bucket_name)["Policy"])
    except Exception:
        return  # No policy to modify

    current_policy["Statement"] = [
        stmt for stmt in current_policy["Statement"]
        if stmt.get("Sid") != deny_sid
    ]

    if current_policy["Statement"]:
        s3.put_bucket_policy(
            Bucket=bucket_name,
            Policy=json.dumps(current_policy),
        )
    else:
        s3.delete_bucket_policy(Bucket=bucket_name)

    print(f"Training access restored on {bucket_name}")
```

**Lambda sanitization function for JSONL and CSV:**
```python
import csv
import io
import json
import re

s3_resource = boto3.resource("s3", region_name=AWS_REGION)

# Default redaction patterns
DEFAULT_REDACTION = "[REDACTED]"

def sanitize_jsonl(content, redaction_patterns):
    """Redact PII from JSONL content (one JSON object per line)."""
    sanitized_lines = []
    records_processed = 0
    fields_redacted = 0

    for line in content.strip().split("\n"):
        if not line.strip():
            continue
        record = json.loads(line)
        record, count = redact_dict(record, redaction_patterns)
        sanitized_lines.append(json.dumps(record, ensure_ascii=False))
        records_processed += 1
        fields_redacted += count

    return "\n".join(sanitized_lines), records_processed, fields_redacted

def sanitize_csv(content, redaction_patterns, pii_columns=None):
    """Redact PII from CSV content."""
    reader = csv.DictReader(io.StringIO(content))
    output = io.StringIO()
    writer = None
    records_processed = 0
    fields_redacted = 0

    for row in reader:
        if writer is None:
            writer = csv.DictWriter(output, fieldnames=reader.fieldnames)
            writer.writeheader()

        for col in (pii_columns or row.keys()):
            if col in row and row[col]:
                for pii_type, replacement in redaction_patterns.items():
                    if contains_pii_pattern(row[col], pii_type):
                        row[col] = replacement
                        fields_redacted += 1
                        break

        writer.writerow(row)
        records_processed += 1

    return output.getvalue(), records_processed, fields_redacted

def redact_dict(obj, redaction_patterns):
    """Recursively redact PII values in a dictionary."""
    count = 0
    if isinstance(obj, dict):
        for key, value in obj.items():
            if isinstance(value, str):
                for pii_type, replacement in redaction_patterns.items():
                    if contains_pii_pattern(value, pii_type):
                        obj[key] = replacement
                        count += 1
                        break
            elif isinstance(value, (dict, list)):
                obj[key], sub_count = redact_dict(value, redaction_patterns)
                count += sub_count
    elif isinstance(obj, list):
        for i, item in enumerate(obj):
            obj[i], sub_count = redact_dict(item, redaction_patterns)
            count += sub_count
    return obj, count

def contains_pii_pattern(value, pii_type):
    """Check if a string value matches a known PII pattern."""
    patterns = {
        "EMAIL_ADDRESS": r"[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}",
        "PHONE_NUMBER": r"\b\d{3}[-.]?\d{3}[-.]?\d{4}\b",
        "CREDIT_CARD_NUMBER": r"\b\d{4}[-\s]?\d{4}[-\s]?\d{4}[-\s]?\d{4}\b",
        "AWS_CREDENTIALS": r"(AKIA|ASIA)[A-Z0-9]{16}",
        "NAME": r"\b[A-Z][a-z]+ [A-Z][a-z]+\b",
    }
    if pii_type in patterns:
        return bool(re.search(patterns[pii_type], value))
    return False

def lambda_handler(event, context):
    """Lambda handler for data sanitization."""
    bucket = event["bucket"]
    quarantine_key = event["quarantine_key"]
    original_key = event["original_key"]

    # Download quarantined file
    obj = s3_resource.Object(bucket, quarantine_key)
    content = obj.get()["Body"].read().decode("utf-8")

    # Detect format and sanitize
    if original_key.endswith(".jsonl") or original_key.endswith(".json"):
        sanitized, records, fields = sanitize_jsonl(content, REDACTION_PATTERNS)
    elif original_key.endswith(".csv"):
        sanitized, records, fields = sanitize_csv(content, REDACTION_PATTERNS)
    else:
        return {"status": "SKIPPED", "reason": f"Unsupported format: {original_key}"}

    # Write sanitized file
    sanitized_key = f"sanitized/{original_key}"
    s3_resource.Object(bucket, sanitized_key).put(
        Body=sanitized.encode("utf-8"),
        Tagging="pii-status=sanitized",
    )

    return {
        "status": "SANITIZED",
        "records_processed": records,
        "fields_redacted": fields,
        "output_path": f"s3://{bucket}/{sanitized_key}",
    }
```

**EventBridge rule for Macie findings:**
```python
events = boto3.client("events", region_name=AWS_REGION)

# Create EventBridge rule for Macie findings
events.put_rule(
    Name=f"{PROJECT_NAME}-macie-finding-remediation-{ENV}",
    Description=f"Trigger remediation on Macie PII findings for {PROJECT_NAME}",
    EventPattern=json.dumps({
        "source": ["aws.macie"],
        "detail-type": ["Macie Finding"],
        "detail": {
            "severity": {
                "description": ["Medium", "High", "Critical"],
            },
            "type": [{
                "prefix": "SensitiveData",
            }],
        },
    }),
    State="ENABLED",
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)

# Set Step Functions state machine as target
events.put_targets(
    Rule=f"{PROJECT_NAME}-macie-finding-remediation-{ENV}",
    Targets=[
        {
            "Id": "macie-remediation-sfn",
            "Arn": STEP_FUNCTIONS_ARN,  # State machine ARN from step_functions_workflow.py
            "RoleArn": EVENTBRIDGE_ROLE_ARN,
            "InputTransformer": {
                "InputPathsMap": {
                    "findingId": "$.detail.id",
                    "bucket": "$.detail.resourcesAffected.s3Bucket.name",
                    "objectKey": "$.detail.resourcesAffected.s3Object.key",
                    "severity": "$.detail.severity.description",
                    "piiTypes": "$.detail.classificationDetails.result.sensitiveData",
                },
                "InputTemplate": json.dumps({
                    "findingId": "<findingId>",
                    "bucket": "<bucket>",
                    "objectKey": "<objectKey>",
                    "severity": "<severity>",
                    "piiTypes": "<piiTypes>",
                    "project": PROJECT_NAME,
                    "environment": ENV,
                }),
            },
        }
    ],
)
```

**Step Functions state machine definition:**
```python
sfn = boto3.client("stepfunctions", region_name=AWS_REGION)

state_machine_definition = {
    "Comment": f"Macie PII remediation workflow for {PROJECT_NAME}",
    "StartAt": "QuarantineObject",
    "States": {
        "QuarantineObject": {
            "Type": "Task",
            "Resource": QUARANTINE_LAMBDA_ARN,
            "Parameters": {
                "bucket.$": "$.bucket",
                "objectKey.$": "$.objectKey",
                "findingId.$": "$.findingId",
            },
            "ResultPath": "$.quarantineResult",
            "Retry": [{"ErrorEquals": ["States.TaskFailed"], "MaxAttempts": 2, "BackoffRate": 2}],
            "Catch": [{"ErrorEquals": ["States.ALL"], "Next": "NotifyOwner", "ResultPath": "$.error"}],
            "Next": "NotifyOwner",
        },
        "NotifyOwner": {
            "Type": "Task",
            "Resource": NOTIFY_LAMBDA_ARN,
            "Parameters": {
                "findingId.$": "$.findingId",
                "bucket.$": "$.bucket",
                "objectKey.$": "$.objectKey",
                "severity.$": "$.severity",
                "piiTypes.$": "$.piiTypes",
                "snsTopicArn": SNS_TOPIC_ARN,
            },
            "ResultPath": "$.notifyResult",
            "Retry": [{"ErrorEquals": ["States.TaskFailed"], "MaxAttempts": 2, "BackoffRate": 2}],
            "Next": "CheckSanitizeEnabled",
        },
        "CheckSanitizeEnabled": {
            "Type": "Choice",
            "Choices": [
                {
                    "Variable": "$.remediationAction",
                    "StringEquals": "sanitize",
                    "Next": "SanitizeData",
                }
            ],
            "Default": "CheckBlockTraining",
        },
        "SanitizeData": {
            "Type": "Task",
            "Resource": SANITIZE_LAMBDA_ARN,
            "Parameters": {
                "bucket.$": "$.bucket",
                "quarantine_key.$": "$.quarantineResult.quarantine_key",
                "original_key.$": "$.objectKey",
            },
            "ResultPath": "$.sanitizeResult",
            "TimeoutSeconds": 900,
            "Retry": [{"ErrorEquals": ["States.TaskFailed"], "MaxAttempts": 1, "BackoffRate": 2}],
            "Next": "CheckBlockTraining",
        },
        "CheckBlockTraining": {
            "Type": "Choice",
            "Choices": [
                {
                    "Variable": "$.blockTraining",
                    "BooleanEquals": False,
                    "Next": "RemediationComplete",
                }
            ],
            "Default": "UnblockAccess",
        },
        "UnblockAccess": {
            "Type": "Task",
            "Resource": UNBLOCK_LAMBDA_ARN,
            "Parameters": {
                "bucket.$": "$.bucket",
            },
            "ResultPath": "$.unblockResult",
            "Next": "RemediationComplete",
        },
        "RemediationComplete": {
            "Type": "Succeed",
        },
    },
}

sfn.create_state_machine(
    name=f"{PROJECT_NAME}-macie-remediation-{ENV}",
    definition=json.dumps(state_machine_definition),
    roleArn=SFN_ROLE_ARN,
    tags=[
        {"key": "Project", "value": PROJECT_NAME},
        {"key": "Environment", "value": ENV},
    ],
)
```

---

## Integration Points

- **Upstream**: `devops/04` → IAM roles for Macie service role, Lambda execution roles, Step Functions execution role, EventBridge invocation role
- **Upstream**: `data/03` → Lake Formation permissions for blocking/restoring access to governed training data tables
- **Downstream**: `data/01` → Glue ETL jobs consume sanitized data from the `sanitized/` S3 prefix after PII remediation
- **Downstream**: `devops/06` → Security Hub receives Macie findings for centralized security monitoring and compliance dashboards
- **Downstream**: `enterprise/01` → SCPs can enforce Macie enablement across all accounts in the ML organization unit