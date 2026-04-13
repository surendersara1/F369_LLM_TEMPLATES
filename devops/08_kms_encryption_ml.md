<!-- Template Version: 1.0 | boto3: 1.35+ -->

# Template DevOps 08 — KMS Encryption & Key Management for ML Workloads

## Purpose
Generate production-ready AWS KMS encryption infrastructure for ML workloads: separate customer-managed KMS keys per purpose (training data, model artifacts, endpoint volumes, Bedrock custom models), key policies with `kms:ViaService` conditions restricting usage to specific AWS services, automatic key rotation with configurable periods, cross-account key sharing grants for multi-account model deployment, key alias management using `alias/{PROJECT_NAME}/{ENV}/{purpose}` convention, CloudWatch alarm for `ScheduleKeyDeletion` API calls with SNS notification, and working examples of passing KMS key ARNs to SageMaker training jobs, endpoint configurations, and Bedrock model customization jobs.

---

## Role Definition

You are an expert AWS security engineer and encryption specialist with expertise in:
- AWS KMS: customer-managed keys (CMKs), key policies, key rotation, key grants, key aliases, multi-region keys
- KMS key policies: principal-based access, `kms:ViaService` condition keys for service-scoped encryption, `kms:CallerAccount` conditions, `kms:EncryptionContext` conditions
- Key rotation: automatic annual rotation, manual rotation with new key material, rotation period configuration
- Cross-account key sharing: KMS grants for external accounts, key policy statements for cross-account access, grant constraints with encryption context
- SageMaker encryption: training job volume encryption (`VolumeKmsKeyId`), output data encryption (`OutputDataConfig.KmsKeyId`), endpoint volume encryption, notebook instance encryption
- Bedrock encryption: custom model encryption (`customModelKmsKeyId`), model customization job output encryption
- Glue encryption: job bookmark encryption, Data Catalog encryption settings, security configuration with KMS
- S3 encryption: SSE-KMS default bucket encryption, bucket key for cost reduction
- CloudWatch monitoring: CloudTrail event filtering for `ScheduleKeyDeletion`, metric filters, alarms with SNS notification
- IAM integration: key administrator vs key user separation, service-linked role access patterns

Generate complete, production-deployable KMS encryption infrastructure code.

---

## Context & Inputs

```
PROJECT_NAME:               [REQUIRED]
AWS_REGION:                 [REQUIRED]
AWS_ACCOUNT_ID:             [REQUIRED]
ENV:                        [REQUIRED - dev | stage | prod]

KEY_PURPOSES:               [REQUIRED - comma-separated list of key purposes to create]
                            Options: training_data, model_artifacts, endpoint_volumes, bedrock_models
                            Example: "training_data,model_artifacts,endpoint_volumes,bedrock_models"
                            Each purpose creates a separate KMS key with tailored key policy:
                            - training_data: encrypts S3 training datasets, Glue ETL output
                            - model_artifacts: encrypts SageMaker model artifacts in S3, ECR images
                            - endpoint_volumes: encrypts SageMaker endpoint instance volumes
                            - bedrock_models: encrypts Bedrock custom model artifacts and fine-tuning output

KEY_ROTATION_PERIOD:        [OPTIONAL: 365]
                            Key rotation period in days. AWS KMS supports automatic rotation
                            with configurable period (90–2560 days). Default: 365 days (annual).
                            Automatic rotation creates new key material while retaining old
                            material for decryption of previously encrypted data.

CROSS_ACCOUNT_IDS:          [OPTIONAL: none]
                            Comma-separated list of AWS account IDs that need cross-account
                            access to KMS keys for model deployment.
                            Example: "111111111111,222222222222"
                            Used for multi-account ML pipelines where models trained in dev
                            are deployed to staging/prod accounts.

KEY_ALIAS_PREFIX:           [OPTIONAL: alias/{PROJECT_NAME}/{ENV}]
                            Prefix for KMS key aliases. Each key gets an alias:
                            {KEY_ALIAS_PREFIX}/{purpose}
                            Example: alias/myai/prod/training-data

KEY_ADMINISTRATORS:         [OPTIONAL - IAM ARNs for key administrators]
                            Comma-separated list of IAM role/user ARNs that can manage keys.
                            If not provided, defaults to the account root principal.
                            Example: "arn:aws:iam::123456789012:role/SecurityAdmin"

SAGEMAKER_ROLE_ARN:         [OPTIONAL - SageMaker execution role ARN]
                            If provided, grants this role usage permissions on relevant keys.
                            If not provided, reads from SSM: /mlops/{PROJECT_NAME}/{ENV}/sagemaker-role-arn

BEDROCK_ROLE_ARN:           [OPTIONAL - Bedrock service role ARN]
                            If provided, grants this role usage permissions on the bedrock_models key.
                            If not provided, reads from SSM: /mlops/{PROJECT_NAME}/{ENV}/bedrock-role-arn

GLUE_ROLE_ARN:              [OPTIONAL - Glue service role ARN]
                            If provided, grants this role usage permissions on the training_data key.
                            If not provided, reads from SSM: /mlops/{PROJECT_NAME}/{ENV}/glue-role-arn

SNS_TOPIC_ARN:              [OPTIONAL - SNS topic for key deletion alerts]
                            If not provided, a new topic is created:
                            {PROJECT_NAME}-kms-alerts-{ENV}

ENABLE_BUCKET_KEY:          [OPTIONAL: true]
                            Enable S3 Bucket Key to reduce KMS API costs when using SSE-KMS.
                            Reduces calls to KMS by up to 99% for S3 operations.
```

---

## Task

Generate complete KMS encryption infrastructure for ML workloads:

```
{PROJECT_NAME}-kms-ml/
├── keys/
│   ├── create_keys.py                    # Create KMS keys per purpose with key policies
│   ├── key_policies.py                   # Key policy document builders with kms:ViaService
│   ├── enable_rotation.py                # Enable automatic key rotation
│   ├── create_aliases.py                 # Create key aliases per naming convention
│   └── create_grants.py                  # Cross-account key sharing grants
├── monitoring/
│   ├── cloudtrail_metric_filter.py       # CloudTrail metric filter for ScheduleKeyDeletion
│   ├── cloudwatch_alarm.py               # CloudWatch alarm + SNS for key deletion events
│   └── key_usage_dashboard.py            # CloudWatch dashboard for KMS key usage metrics
├── integration/
│   ├── sagemaker_encryption.py           # Examples: pass KMS ARNs to SageMaker API calls
│   ├── bedrock_encryption.py             # Examples: pass KMS ARNs to Bedrock API calls
│   └── s3_bucket_encryption.py           # Configure S3 default encryption with KMS
├── infrastructure/
│   ├── create_sns_topic.py               # Create SNS topic for key deletion alerts
│   ├── ssm_outputs.py                    # Write key ARNs and alias ARNs to SSM
│   └── config.py                         # Central configuration
├── run_setup.py                          # CLI orchestrator
└── requirements.txt
```

**config.py**: Central configuration dataclass with all parameters. Load from environment variables or CLI args. Validate required fields. Construct resource names using `{PROJECT_NAME}-{component}-{ENV}` convention. Parse KEY_PURPOSES into a list. Parse CROSS_ACCOUNT_IDS into a list. Resolve role ARNs from SSM if not provided directly.

**create_keys.py**: Create separate KMS keys for each purpose in KEY_PURPOSES:
- Iterate over KEY_PURPOSES list
- For each purpose, call `kms.create_key()` with:
  - `Description`: `{PROJECT_NAME} {ENV} - {purpose} encryption key`
  - `KeyUsage`: `ENCRYPT_DECRYPT`
  - `KeySpec`: `SYMMETRIC_DEFAULT`
  - `Policy`: generated by `key_policies.py` with purpose-specific `kms:ViaService` conditions
  - `Tags`: `Project={PROJECT_NAME}`, `Environment={ENV}`, `Purpose={purpose}`, `ManagedBy=automation`
- Store key IDs and ARNs for downstream use (aliases, rotation, grants, SSM)
- Return mapping of `{purpose: key_arn}`

**key_policies.py**: Build KMS key policy JSON documents per purpose:
- `build_key_policy(purpose, admin_arns, user_role_arns, cross_account_ids)`: construct complete key policy
- Root account statement: allow account root full KMS access (required for key policy management)
- Key administrator statement: allow KEY_ADMINISTRATORS `kms:Create*`, `kms:Describe*`, `kms:Enable*`, `kms:List*`, `kms:Put*`, `kms:Update*`, `kms:Revoke*`, `kms:Disable*`, `kms:Get*`, `kms:Delete*`, `kms:TagResource`, `kms:UntagResource`, `kms:ScheduleKeyDeletion`, `kms:CancelKeyDeletion`
- Key usage statement per purpose with `kms:ViaService` conditions:
  - `training_data`: allow SageMaker role and Glue role; condition `kms:ViaService` = `s3.{region}.amazonaws.com`, `glue.{region}.amazonaws.com`
  - `model_artifacts`: allow SageMaker role and CodeBuild role; condition `kms:ViaService` = `s3.{region}.amazonaws.com`, `sagemaker.{region}.amazonaws.com`
  - `endpoint_volumes`: allow SageMaker role; condition `kms:ViaService` = `sagemaker.{region}.amazonaws.com`
  - `bedrock_models`: allow Bedrock role; condition `kms:ViaService` = `bedrock.{region}.amazonaws.com`
- Key usage actions: `kms:Encrypt`, `kms:Decrypt`, `kms:ReEncrypt*`, `kms:GenerateDataKey*`, `kms:DescribeKey`, `kms:CreateGrant` (with `GrantIsForAWSResource` condition)
- Cross-account statement (if CROSS_ACCOUNT_IDS provided): allow external account roots `kms:Decrypt`, `kms:DescribeKey`, `kms:CreateGrant` with `kms:ViaService` condition

**enable_rotation.py**: Enable automatic key rotation for all created keys:
- For each key, call `kms.enable_key_rotation()` with `KeyId`
- If KEY_ROTATION_PERIOD is not the default (365), call `kms.update_key_rotation_period()` with `RotationPeriodInDays`
- Verify rotation status using `kms.get_key_rotation_status()`
- Log rotation configuration for each key

**create_aliases.py**: Create key aliases for each purpose:
- For each key, call `kms.create_alias()` with:
  - `AliasName`: `{KEY_ALIAS_PREFIX}/{purpose}` (e.g., `alias/myai/prod/training-data`)
  - `TargetKeyId`: the key ID from create_keys.py
- Aliases provide human-readable references and can be used in place of key ARNs in API calls
- Log alias creation with alias name → key ARN mapping

**create_grants.py**: Create KMS grants for cross-account key sharing:
- For each CROSS_ACCOUNT_ID and each key purpose:
  - Call `kms.create_grant()` with:
    - `KeyId`: key ARN
    - `GranteePrincipal`: `arn:aws:iam::{cross_account_id}:root`
    - `Operations`: `["Decrypt", "DescribeKey", "GenerateDataKey", "ReEncryptFrom"]`
    - `Constraints`: `{"EncryptionContextSubset": {"Project": PROJECT_NAME}}`
  - Grants allow fine-grained, revocable cross-account access without modifying key policies
- Store grant IDs for tracking and potential revocation
- Log grant creation with grantee account, key purpose, and grant ID

**cloudtrail_metric_filter.py**: Create CloudWatch metric filter for key deletion events:
- Create metric filter on CloudTrail log group matching `ScheduleKeyDeletion` API calls
- Filter pattern: `{ ($.eventName = "ScheduleKeyDeletion") && ($.requestParameters.keyId = "*{PROJECT_NAME}*") }`
- Metric name: `{PROJECT_NAME}-kms-key-deletion-{ENV}`
- Metric namespace: `{PROJECT_NAME}/KMS/Security`

**cloudwatch_alarm.py**: Create CloudWatch alarm for key deletion detection:
- Create alarm on the metric filter metric
- Threshold: >= 1 in 1 evaluation period (5 minutes)
- Action: send SNS notification to SNS_TOPIC_ARN
- Alarm name: `{PROJECT_NAME}-kms-deletion-alarm-{ENV}`
- Alarm description includes key purpose and remediation steps

**key_usage_dashboard.py**: Create CloudWatch dashboard for KMS key usage:
- Dashboard name: `{PROJECT_NAME}-kms-usage-{ENV}`
- Widgets: KMS API call counts per key (Encrypt, Decrypt, GenerateDataKey), key deletion alarm status, key rotation status summary

**sagemaker_encryption.py**: Working examples of passing KMS ARNs to SageMaker:
- `create_training_job()` with `ResourceConfig.VolumeKmsKeyId` and `OutputDataConfig.KmsKeyId`
- `create_endpoint_config()` with `KmsKeyId` for endpoint volume encryption
- `create_notebook_instance()` with `KmsKeyId` for notebook storage encryption

**bedrock_encryption.py**: Working examples of passing KMS ARNs to Bedrock:
- `create_model_customization_job()` with `customModelKmsKeyId` and `outputDataConfig.s3Uri` encrypted bucket
- `create_provisioned_model_throughput()` referencing encrypted custom model

**s3_bucket_encryption.py**: Configure S3 default encryption with KMS:
- `put_bucket_encryption()` with SSE-KMS using the training_data key
- Enable S3 Bucket Key if ENABLE_BUCKET_KEY=true for cost reduction

**ssm_outputs.py**: Write key ARNs and alias ARNs to SSM Parameter Store:
- Path: `/mlops/{PROJECT_NAME}/{ENV}/kms-{purpose}-key-arn` for each key
- Path: `/mlops/{PROJECT_NAME}/{ENV}/kms-{purpose}-alias` for each alias
- Other templates read from these SSM paths for encryption configuration

**create_sns_topic.py**: Create SNS topic for key deletion alerts:
- Create topic `{PROJECT_NAME}-kms-alerts-{ENV}` using `sns.create_topic()`
- Configure topic policy for CloudWatch Alarms publishing
- Tag with Project and Environment
- Return topic ARN

**run_setup.py**: CLI orchestrator that runs setup steps in order:
1. Create SNS topic for key deletion alerts
2. Create KMS keys per purpose with key policies
3. Create key aliases
4. Enable automatic key rotation
5. Create cross-account grants (if CROSS_ACCOUNT_IDS provided)
6. Configure CloudTrail metric filter and CloudWatch alarm
7. Write key ARNs to SSM Parameter Store
8. Print summary with key ARNs, aliases, rotation status, and grant IDs

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**Key Separation:** Create separate KMS keys per purpose (training data, model artifacts, endpoint volumes, Bedrock models). Key separation limits blast radius — if one key is compromised, only that category of data is affected. Each key has a tailored key policy with `kms:ViaService` conditions restricting which AWS services can use it.

**Key Policies:** Every key policy must include: (1) root account access statement for policy management, (2) key administrator statement for key lifecycle operations, (3) key user statement with `kms:ViaService` conditions scoped to the specific AWS services that need the key. Use `kms:ViaService` to ensure keys can only be used through their intended services (e.g., the training data key can only be used via S3 and Glue, not directly). Include `kms:CallerAccount` condition to restrict usage to the owning account (plus explicitly granted cross-account IDs).

**Key Rotation:** Enable automatic key rotation on all keys. AWS KMS generates new cryptographic material annually (or per KEY_ROTATION_PERIOD) while retaining old material for decryption. Rotation is transparent — existing encrypted data does not need re-encryption. The key ARN and alias remain the same after rotation. Monitor rotation via `kms:GetKeyRotationStatus`.

**Cross-Account Access:** Use KMS grants (not key policy modifications) for cross-account access when possible. Grants are revocable, auditable, and scoped to specific operations. Include `EncryptionContextSubset` constraints on grants to ensure cross-account usage is limited to project resources. For persistent cross-account access, add key policy statements with external account root principals.

**Alias Management:** Use aliases following `alias/{PROJECT_NAME}/{ENV}/{purpose}` convention. Aliases provide human-readable references and enable key rotation without updating consumer configurations. Never delete an alias without first verifying no resources reference it. Aliases are unique per account per region.

**Monitoring:** Create CloudWatch alarm for `ScheduleKeyDeletion` API calls to detect accidental or malicious key deletion attempts. Key deletion has a mandatory 7–30 day waiting period — the alarm provides early warning to cancel deletion if unintended. Monitor KMS API usage (Encrypt, Decrypt, GenerateDataKey) per key for anomaly detection.

**Security:** Key administrators should not be key users (separation of duties). Use `kms:GrantIsForAWSResource` condition on `kms:CreateGrant` to ensure grants are only created by AWS services on behalf of the user, not directly. Never use `kms:*` in key policies — enumerate specific actions. Tag all keys with Project, Environment, and Purpose for cost allocation and governance.

**Cost:** KMS charges per API call ($0.03 per 10,000 requests). Enable S3 Bucket Key to reduce KMS calls by up to 99% for S3 SSE-KMS operations. Each customer-managed key costs $1/month. Monitor `AWS/KMS` CloudWatch metrics to track API call volume and optimize. Use key aliases in API calls to avoid hardcoding key ARNs.

**Naming:** All resources follow `{PROJECT_NAME}-{component}-{ENV}` convention:
- KMS key description: `{PROJECT_NAME} {ENV} - {purpose} encryption key`
- Key alias: `alias/{PROJECT_NAME}/{ENV}/{purpose}`
- CloudWatch alarm: `{PROJECT_NAME}-kms-deletion-alarm-{ENV}`
- SNS topic: `{PROJECT_NAME}-kms-alerts-{ENV}`
- SSM parameter: `/mlops/{PROJECT_NAME}/{ENV}/kms-{purpose}-key-arn`
- Dashboard: `{PROJECT_NAME}-kms-usage-{ENV}`

---

## Code Scaffolding Hints

**Create KMS key with purpose-specific key policy:**
```python
import boto3
import json

kms = boto3.client("kms", region_name=AWS_REGION)

def create_ml_kms_key(purpose, key_policy):
    """Create a customer-managed KMS key for a specific ML purpose."""
    response = kms.create_key(
        Description=f"{PROJECT_NAME} {ENV} - {purpose} encryption key",
        KeyUsage="ENCRYPT_DECRYPT",
        KeySpec="SYMMETRIC_DEFAULT",
        Policy=json.dumps(key_policy),
        Tags=[
            {"TagKey": "Project", "TagValue": PROJECT_NAME},
            {"TagKey": "Environment", "TagValue": ENV},
            {"TagKey": "Purpose", "TagValue": purpose},
            {"TagKey": "ManagedBy", "TagValue": "automation"},
        ],
    )
    key_id = response["KeyMetadata"]["KeyId"]
    key_arn = response["KeyMetadata"]["Arn"]
    print(f"KMS key created for {purpose}: {key_arn}")
    return key_id, key_arn
```

**Build key policy with `kms:ViaService` conditions:**
```python
def build_training_data_key_policy(
    account_id, admin_arns, sagemaker_role_arn, glue_role_arn, region, cross_account_ids=None
):
    """Build key policy for training data encryption key."""
    policy = {
        "Version": "2012-10-17",
        "Id": f"{PROJECT_NAME}-training-data-key-policy",
        "Statement": [
            # Root account access (required for key policy management)
            {
                "Sid": "EnableRootAccountAccess",
                "Effect": "Allow",
                "Principal": {"AWS": f"arn:aws:iam::{account_id}:root"},
                "Action": "kms:*",
                "Resource": "*",
            },
            # Key administrators
            {
                "Sid": "AllowKeyAdministration",
                "Effect": "Allow",
                "Principal": {"AWS": admin_arns},
                "Action": [
                    "kms:Create*",
                    "kms:Describe*",
                    "kms:Enable*",
                    "kms:List*",
                    "kms:Put*",
                    "kms:Update*",
                    "kms:Revoke*",
                    "kms:Disable*",
                    "kms:Get*",
                    "kms:Delete*",
                    "kms:TagResource",
                    "kms:UntagResource",
                    "kms:ScheduleKeyDeletion",
                    "kms:CancelKeyDeletion",
                ],
                "Resource": "*",
            },
            # Key usage for SageMaker and Glue via S3 and Glue services
            {
                "Sid": "AllowMLServiceEncryption",
                "Effect": "Allow",
                "Principal": {
                    "AWS": [sagemaker_role_arn, glue_role_arn],
                },
                "Action": [
                    "kms:Encrypt",
                    "kms:Decrypt",
                    "kms:ReEncrypt*",
                    "kms:GenerateDataKey*",
                    "kms:DescribeKey",
                ],
                "Resource": "*",
                "Condition": {
                    "StringEquals": {
                        "kms:ViaService": [
                            f"s3.{region}.amazonaws.com",
                            f"glue.{region}.amazonaws.com",
                        ],
                        "kms:CallerAccount": account_id,
                    }
                },
            },
            # Allow grant creation for AWS services
            {
                "Sid": "AllowGrantsForAWSServices",
                "Effect": "Allow",
                "Principal": {
                    "AWS": [sagemaker_role_arn, glue_role_arn],
                },
                "Action": "kms:CreateGrant",
                "Resource": "*",
                "Condition": {
                    "Bool": {"kms:GrantIsForAWSResource": "true"},
                },
            },
        ],
    }

    # Add cross-account access if configured
    if cross_account_ids:
        policy["Statement"].append({
            "Sid": "AllowCrossAccountDecrypt",
            "Effect": "Allow",
            "Principal": {
                "AWS": [f"arn:aws:iam::{acct}:root" for acct in cross_account_ids],
            },
            "Action": [
                "kms:Decrypt",
                "kms:DescribeKey",
                "kms:CreateGrant",
            ],
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "kms:ViaService": f"s3.{region}.amazonaws.com",
                },
            },
        })

    return policy


def build_endpoint_volumes_key_policy(account_id, admin_arns, sagemaker_role_arn, region):
    """Build key policy for SageMaker endpoint volume encryption."""
    return {
        "Version": "2012-10-17",
        "Id": f"{PROJECT_NAME}-endpoint-volumes-key-policy",
        "Statement": [
            {
                "Sid": "EnableRootAccountAccess",
                "Effect": "Allow",
                "Principal": {"AWS": f"arn:aws:iam::{account_id}:root"},
                "Action": "kms:*",
                "Resource": "*",
            },
            {
                "Sid": "AllowKeyAdministration",
                "Effect": "Allow",
                "Principal": {"AWS": admin_arns},
                "Action": [
                    "kms:Create*", "kms:Describe*", "kms:Enable*", "kms:List*",
                    "kms:Put*", "kms:Update*", "kms:Revoke*", "kms:Disable*",
                    "kms:Get*", "kms:Delete*", "kms:TagResource", "kms:UntagResource",
                    "kms:ScheduleKeyDeletion", "kms:CancelKeyDeletion",
                ],
                "Resource": "*",
            },
            {
                "Sid": "AllowSageMakerVolumeEncryption",
                "Effect": "Allow",
                "Principal": {"AWS": sagemaker_role_arn},
                "Action": [
                    "kms:Encrypt", "kms:Decrypt", "kms:ReEncrypt*",
                    "kms:GenerateDataKey*", "kms:DescribeKey",
                ],
                "Resource": "*",
                "Condition": {
                    "StringEquals": {
                        "kms:ViaService": f"sagemaker.{region}.amazonaws.com",
                        "kms:CallerAccount": account_id,
                    }
                },
            },
            {
                "Sid": "AllowGrantsForAWSServices",
                "Effect": "Allow",
                "Principal": {"AWS": sagemaker_role_arn},
                "Action": "kms:CreateGrant",
                "Resource": "*",
                "Condition": {
                    "Bool": {"kms:GrantIsForAWSResource": "true"},
                },
            },
        ],
    }


def build_bedrock_models_key_policy(account_id, admin_arns, bedrock_role_arn, region):
    """Build key policy for Bedrock custom model encryption."""
    return {
        "Version": "2012-10-17",
        "Id": f"{PROJECT_NAME}-bedrock-models-key-policy",
        "Statement": [
            {
                "Sid": "EnableRootAccountAccess",
                "Effect": "Allow",
                "Principal": {"AWS": f"arn:aws:iam::{account_id}:root"},
                "Action": "kms:*",
                "Resource": "*",
            },
            {
                "Sid": "AllowKeyAdministration",
                "Effect": "Allow",
                "Principal": {"AWS": admin_arns},
                "Action": [
                    "kms:Create*", "kms:Describe*", "kms:Enable*", "kms:List*",
                    "kms:Put*", "kms:Update*", "kms:Revoke*", "kms:Disable*",
                    "kms:Get*", "kms:Delete*", "kms:TagResource", "kms:UntagResource",
                    "kms:ScheduleKeyDeletion", "kms:CancelKeyDeletion",
                ],
                "Resource": "*",
            },
            {
                "Sid": "AllowBedrockModelEncryption",
                "Effect": "Allow",
                "Principal": {"AWS": bedrock_role_arn},
                "Action": [
                    "kms:Encrypt", "kms:Decrypt", "kms:ReEncrypt*",
                    "kms:GenerateDataKey*", "kms:DescribeKey",
                ],
                "Resource": "*",
                "Condition": {
                    "StringEquals": {
                        "kms:ViaService": f"bedrock.{region}.amazonaws.com",
                        "kms:CallerAccount": account_id,
                    }
                },
            },
            {
                "Sid": "AllowGrantsForAWSServices",
                "Effect": "Allow",
                "Principal": {"AWS": bedrock_role_arn},
                "Action": "kms:CreateGrant",
                "Resource": "*",
                "Condition": {
                    "Bool": {"kms:GrantIsForAWSResource": "true"},
                },
            },
        ],
    }
```

**Enable automatic key rotation:**
```python
def enable_rotation(key_id, rotation_period_days=365):
    """Enable automatic key rotation with configurable period."""
    kms.enable_key_rotation(KeyId=key_id)

    # Configure rotation period if non-default
    if rotation_period_days != 365:
        kms.update_key_rotation_period(
            KeyId=key_id,
            RotationPeriodInDays=rotation_period_days,
        )

    # Verify rotation is enabled
    status = kms.get_key_rotation_status(KeyId=key_id)
    print(f"Key {key_id}: rotation enabled = {status['KeyRotationEnabled']}")
    if "RotationPeriodInDays" in status:
        print(f"  Rotation period: {status['RotationPeriodInDays']} days")
    return status["KeyRotationEnabled"]
```

**Create key aliases:**
```python
def create_key_alias(key_id, purpose, alias_prefix=None):
    """Create a human-readable alias for a KMS key."""
    alias_prefix = alias_prefix or f"alias/{PROJECT_NAME}/{ENV}"
    alias_name = f"{alias_prefix}/{purpose}"

    # Delete existing alias if present (aliases are unique)
    try:
        kms.delete_alias(AliasName=alias_name)
    except kms.exceptions.NotFoundException:
        pass

    kms.create_alias(
        AliasName=alias_name,
        TargetKeyId=key_id,
    )
    print(f"Alias created: {alias_name} -> {key_id}")
    return alias_name
```

**Create cross-account grants:**
```python
def create_cross_account_grant(key_id, key_purpose, grantee_account_id):
    """Create a KMS grant for cross-account key access."""
    response = kms.create_grant(
        KeyId=key_id,
        GranteePrincipal=f"arn:aws:iam::{grantee_account_id}:root",
        Operations=[
            "Decrypt",
            "DescribeKey",
            "GenerateDataKey",
            "ReEncryptFrom",
        ],
        Constraints={
            "EncryptionContextSubset": {
                "Project": PROJECT_NAME,
            }
        },
        Name=f"{PROJECT_NAME}-{key_purpose}-grant-{grantee_account_id}-{ENV}",
    )
    grant_id = response["GrantId"]
    grant_token = response["GrantToken"]
    print(f"Grant created: {key_purpose} -> account {grantee_account_id} (grant ID: {grant_id})")
    return grant_id, grant_token
```

**CloudWatch alarm for ScheduleKeyDeletion:**
```python
import boto3

logs = boto3.client("logs", region_name=AWS_REGION)
cloudwatch = boto3.client("cloudwatch", region_name=AWS_REGION)
sns = boto3.client("sns", region_name=AWS_REGION)

# Create metric filter on CloudTrail log group
logs.put_metric_filter(
    logGroupName="aws-cloudtrail-logs",  # Your CloudTrail log group name
    filterName=f"{PROJECT_NAME}-kms-key-deletion-{ENV}",
    filterPattern='{ ($.eventName = "ScheduleKeyDeletion") }',
    metricTransformations=[
        {
            "metricName": f"{PROJECT_NAME}-kms-key-deletion-{ENV}",
            "metricNamespace": f"{PROJECT_NAME}/KMS/Security",
            "metricValue": "1",
            "defaultValue": 0,
        }
    ],
)

# Create CloudWatch alarm
cloudwatch.put_metric_alarm(
    AlarmName=f"{PROJECT_NAME}-kms-deletion-alarm-{ENV}",
    AlarmDescription=(
        f"CRITICAL: A KMS key in {PROJECT_NAME} {ENV} has been scheduled for deletion. "
        "Verify this is intentional. Use kms:CancelKeyDeletion to reverse if needed."
    ),
    Namespace=f"{PROJECT_NAME}/KMS/Security",
    MetricName=f"{PROJECT_NAME}-kms-key-deletion-{ENV}",
    Statistic="Sum",
    Period=300,  # 5 minutes
    EvaluationPeriods=1,
    Threshold=1,
    ComparisonOperator="GreaterThanOrEqualToThreshold",
    TreatMissingData="notBreaching",
    AlarmActions=[SNS_TOPIC_ARN],
    OKActions=[SNS_TOPIC_ARN],
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)
print(f"CloudWatch alarm created: {PROJECT_NAME}-kms-deletion-alarm-{ENV}")
```

**Pass KMS ARNs to SageMaker training job:**
```python
import boto3

sagemaker = boto3.client("sagemaker", region_name=AWS_REGION)

# Read KMS key ARNs from SSM (written by ssm_outputs.py)
ssm = boto3.client("ssm", region_name=AWS_REGION)
training_data_key_arn = ssm.get_parameter(
    Name=f"/mlops/{PROJECT_NAME}/{ENV}/kms-training_data-key-arn"
)["Parameter"]["Value"]
model_artifacts_key_arn = ssm.get_parameter(
    Name=f"/mlops/{PROJECT_NAME}/{ENV}/kms-model_artifacts-key-arn"
)["Parameter"]["Value"]
endpoint_volumes_key_arn = ssm.get_parameter(
    Name=f"/mlops/{PROJECT_NAME}/{ENV}/kms-endpoint_volumes-key-arn"
)["Parameter"]["Value"]

# Create training job with KMS encryption
sagemaker.create_training_job(
    TrainingJobName=f"{PROJECT_NAME}-training-{ENV}",
    AlgorithmSpecification={
        "TrainingImage": "763104351884.dkr.ecr.us-east-1.amazonaws.com/pytorch-training:2.0-gpu-py310",
        "TrainingInputMode": "File",
    },
    RoleArn=SAGEMAKER_ROLE_ARN,
    InputDataConfig=[
        {
            "ChannelName": "training",
            "DataSource": {
                "S3DataSource": {
                    "S3DataType": "S3Prefix",
                    "S3Uri": f"s3://{PROJECT_NAME}-training-data-{ENV}/datasets/",
                }
            },
        }
    ],
    OutputDataConfig={
        "S3OutputPath": f"s3://{PROJECT_NAME}-model-artifacts-{ENV}/output/",
        "KmsKeyId": model_artifacts_key_arn,  # Encrypt model output
    },
    ResourceConfig={
        "InstanceType": "ml.p3.2xlarge",
        "InstanceCount": 1,
        "VolumeSizeInGB": 50,
        "VolumeKmsKeyId": endpoint_volumes_key_arn,  # Encrypt training volume
    },
    StoppingCondition={"MaxRuntimeInSeconds": 86400},
)

# Create endpoint config with volume encryption
sagemaker.create_endpoint_config(
    EndpointConfigName=f"{PROJECT_NAME}-endpoint-config-{ENV}",
    ProductionVariants=[
        {
            "VariantName": "primary",
            "ModelName": f"{PROJECT_NAME}-model-{ENV}",
            "InstanceType": "ml.g5.xlarge",
            "InitialInstanceCount": 1,
        }
    ],
    KmsKeyId=endpoint_volumes_key_arn,  # Encrypt endpoint storage volume
)
```

**Pass KMS ARNs to Bedrock model customization job:**
```python
bedrock = boto3.client("bedrock", region_name=AWS_REGION)

bedrock_key_arn = ssm.get_parameter(
    Name=f"/mlops/{PROJECT_NAME}/{ENV}/kms-bedrock_models-key-arn"
)["Parameter"]["Value"]

# Create Bedrock model customization (fine-tuning) job with KMS encryption
bedrock.create_model_customization_job(
    jobName=f"{PROJECT_NAME}-finetune-{ENV}",
    customModelName=f"{PROJECT_NAME}-custom-model-{ENV}",
    roleArn=BEDROCK_ROLE_ARN,
    baseModelIdentifier="amazon.titan-text-express-v1",
    customizationType="FINE_TUNING",
    trainingDataConfig={
        "s3Uri": f"s3://{PROJECT_NAME}-training-data-{ENV}/bedrock/train.jsonl",
    },
    validationDataConfig={
        "validators": [
            {"s3Uri": f"s3://{PROJECT_NAME}-training-data-{ENV}/bedrock/validation.jsonl"}
        ]
    },
    outputDataConfig={
        "s3Uri": f"s3://{PROJECT_NAME}-model-artifacts-{ENV}/bedrock/output/",
    },
    customModelKmsKeyId=bedrock_key_arn,  # Encrypt custom model artifacts
    hyperParameters={
        "epochCount": "3",
        "batchSize": "8",
        "learningRate": "0.00001",
    },
)
```

**Configure S3 default encryption with KMS and Bucket Key:**
```python
s3 = boto3.client("s3", region_name=AWS_REGION)

def configure_bucket_encryption(bucket_name, kms_key_arn, enable_bucket_key=True):
    """Configure S3 bucket default encryption with KMS and optional Bucket Key."""
    s3.put_bucket_encryption(
        Bucket=bucket_name,
        ServerSideEncryptionConfiguration={
            "Rules": [
                {
                    "ApplyServerSideEncryptionByDefault": {
                        "SSEAlgorithm": "aws:kms",
                        "KMSMasterKeyID": kms_key_arn,
                    },
                    "BucketKeyEnabled": enable_bucket_key,
                }
            ]
        },
    )
    print(f"Bucket {bucket_name}: SSE-KMS enabled with key {kms_key_arn}")
    if enable_bucket_key:
        print(f"  S3 Bucket Key enabled (reduces KMS API calls by up to 99%)")

# Configure encryption on ML data buckets
configure_bucket_encryption(
    f"{PROJECT_NAME}-training-data-{ENV}",
    training_data_key_arn,
)
configure_bucket_encryption(
    f"{PROJECT_NAME}-model-artifacts-{ENV}",
    model_artifacts_key_arn,
)
```

**Write key ARNs to SSM Parameter Store:**
```python
ssm = boto3.client("ssm", region_name=AWS_REGION)

def write_key_to_ssm(purpose, key_arn, alias_name):
    """Write KMS key ARN and alias to SSM for cross-template consumption."""
    ssm.put_parameter(
        Name=f"/mlops/{PROJECT_NAME}/{ENV}/kms-{purpose}-key-arn",
        Description=f"KMS key ARN for {purpose} encryption",
        Value=key_arn,
        Type="String",
        Overwrite=True,
        Tags=[
            {"Key": "Project", "Value": PROJECT_NAME},
            {"Key": "Environment", "Value": ENV},
        ],
    )
    ssm.put_parameter(
        Name=f"/mlops/{PROJECT_NAME}/{ENV}/kms-{purpose}-alias",
        Description=f"KMS key alias for {purpose} encryption",
        Value=alias_name,
        Type="String",
        Overwrite=True,
    )
    print(f"SSM: /mlops/{PROJECT_NAME}/{ENV}/kms-{purpose}-key-arn = {key_arn}")
```

---

## Integration Points

- **Upstream**: `devops/04` → IAM roles (SageMaker execution role, Bedrock role, Glue role, CodeBuild role) that are granted key usage permissions in key policies
- **Downstream**: `devops/05` → Config rules that verify SageMaker resources have KMS encryption enabled (`sagemaker-endpoint-configuration-kms-key-configured`, `sagemaker-notebook-instance-kms-key-configured`)
- **Downstream**: `enterprise/02` → Cross-account model deployment pipeline uses cross-account KMS grants to decrypt model artifacts in target accounts
- **Downstream**: `data/03` → Lake Formation governed tables use KMS keys for S3 data encryption at rest
- **Downstream**: `data/04` → S3 lifecycle management configures SSE-KMS default encryption on ML data buckets using keys from this template
- **Downstream**: `data/01` → Glue ETL jobs use the training_data key for encrypting processed feature output in S3
- **Downstream**: `mlops/01` → SageMaker training pipeline reads KMS key ARNs from SSM for volume and output encryption
- **Downstream**: `mlops/09` → Bedrock fine-tuning jobs use the bedrock_models key for custom model encryption
- **Downstream**: `devops/07` → Macie results bucket encrypted with KMS key from this template