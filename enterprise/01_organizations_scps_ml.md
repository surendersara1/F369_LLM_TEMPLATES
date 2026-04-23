<!-- Template Version: 1.1 | boto3: 1.35+ | Model IDs: 2026-04-22 refresh -->

# Template Enterprise 01 — AWS Organizations SCPs for ML Governance

## Purpose
Generate production-ready AWS Organizations Service Control Policies (SCPs) for ML governance: SCPs denying unapproved SageMaker instance types using `sagemaker:InstanceTypes` condition key, SCPs enforcing KMS encryption on all SageMaker resources via `sagemaker:VolumeKmsKey` and `sagemaker:OutputKmsKey` condition keys, region restriction SCPs using `aws:RequestedRegion`, Bedrock model ID restriction SCPs using `bedrock:ModelId` condition key, SCP attachment to target organizational units, and a break-glass exception process using IAM role ARN conditions that allow designated roles to bypass restrictions during emergencies.

---

## Role Definition

You are an expert AWS cloud architect and ML governance specialist with expertise in:
- AWS Organizations: organizational units (OUs), service control policies (SCPs), policy inheritance, policy evaluation logic
- SCP authoring: `Deny` effect policies, `Condition` blocks with `StringNotLike`, `StringNotEquals`, `ArnNotLike`, `Null` condition operators
- SageMaker condition keys: `sagemaker:InstanceTypes`, `sagemaker:VolumeKmsKey`, `sagemaker:OutputKmsKey`, `sagemaker:DirectInternetAccess`, `sagemaker:RootAccess`, `sagemaker:VpcSecurityGroupIds`
- Bedrock condition keys: `bedrock:ModelId` for restricting foundation model access
- Region restriction patterns: `aws:RequestedRegion` condition key for limiting ML workloads to approved regions
- KMS encryption enforcement: denying resource creation when encryption keys are not specified
- Break-glass exception patterns: `aws:PrincipalArn` condition key with `ArnNotLike` to exempt emergency roles from SCP restrictions
- SCP size limits: 5,120 bytes per policy, strategies for splitting large policies across multiple SCPs
- Policy testing: Organizations policy simulator, CloudTrail verification of denied actions
- IAM policy evaluation: interaction between SCPs, identity policies, and resource policies

Generate complete, production-deployable SCP governance code.

---

## Context & Inputs

```
PROJECT_NAME:               [REQUIRED]
AWS_REGION:                 [REQUIRED]
AWS_ACCOUNT_ID:             [REQUIRED]
ENV:                        [REQUIRED - dev | stage | prod]

APPROVED_INSTANCE_TYPES:    [REQUIRED - comma-separated list of allowed SageMaker instance types]
                            Example: "ml.m5.xlarge,ml.m5.2xlarge,ml.p3.2xlarge,ml.g5.xlarge,ml.g5.2xlarge,ml.inf2.xlarge"
                            SCP will deny any SageMaker CreateTrainingJob, CreateEndpointConfig,
                            CreateNotebookInstance, or CreateProcessingJob that uses an instance
                            type not in this list. Use wildcards for families: "ml.m5.*,ml.g5.*"

APPROVED_REGIONS:           [REQUIRED - comma-separated list of allowed AWS regions for ML workloads]
                            Example: "us-east-1,us-west-2,eu-west-1"
                            SCP will deny ML service API calls outside these regions.
                            Applies to SageMaker, Bedrock, Glue, and Comprehend actions.

APPROVED_BEDROCK_MODELS:    [REQUIRED - comma-separated list of allowed Bedrock model IDs]
                            Example: "us.anthropic.claude-sonnet-4-7-20260109-v1:0,us.anthropic.claude-haiku-4-5-20251001-v1:0,amazon.titan-embed-text-v2:0"
                            SCP will deny bedrock:InvokeModel and bedrock:InvokeModelWithResponseStream
                            for model IDs not in this list. Supports wildcards: "us.anthropic.claude-sonnet-4-*"

TARGET_OUS:                 [REQUIRED - comma-separated list of OU IDs to attach SCPs]
                            Example: "ou-abc1-mlworkloads,ou-abc1-mlsandbox"
                            SCPs are attached to these OUs. All accounts under these OUs
                            inherit the policy restrictions.

BREAK_GLASS_ROLE_ARNS:      [OPTIONAL: none]
                            Comma-separated list of IAM role ARNs exempt from SCP restrictions.
                            Example: "arn:aws:iam::*:role/BreakGlassAdmin,arn:aws:iam::*:role/EmergencyMLOps"
                            These roles can bypass all ML governance SCPs during emergencies.
                            Use wildcard account ID (*) to apply across all accounts in the OU.

ENCRYPTION_REQUIRED:        [OPTIONAL: true]
                            When true, generates SCPs that deny SageMaker resource creation
                            without KMS encryption keys specified. Set to false for dev/sandbox
                            environments where encryption enforcement is relaxed.

MANAGEMENT_ACCOUNT_ID:      [OPTIONAL: same as AWS_ACCOUNT_ID]
                            AWS Organizations management account ID where SCPs are created.
                            SCPs can only be created from the management account or a
                            delegated administrator account.

DENY_INTERNET_ACCESS:       [OPTIONAL: true]
                            When true, generates SCP denying SageMaker notebook instances
                            with direct internet access enabled.

DENY_ROOT_ACCESS:           [OPTIONAL: true]
                            When true, generates SCP denying SageMaker notebook instances
                            with root access enabled.
```

---

## Task

Generate complete AWS Organizations SCP governance infrastructure for ML workloads:

```
{PROJECT_NAME}-org-scps-ml/
├── policies/
│   ├── scp_instance_type_restriction.py    # SCP: deny unapproved SageMaker instance types
│   ├── scp_encryption_enforcement.py       # SCP: enforce KMS encryption on SageMaker resources
│   ├── scp_region_restriction.py           # SCP: restrict ML services to approved regions
│   ├── scp_bedrock_model_restriction.py    # SCP: deny unapproved Bedrock model access
│   ├── scp_notebook_security.py            # SCP: deny internet access and root on notebooks
│   └── policy_documents.py                 # Complete SCP JSON policy document definitions
├── attachment/
│   ├── attach_policies.py                  # Attach SCPs to target OUs
│   └── detach_policies.py                  # Detach SCPs (for rollback)
├── break_glass/
│   ├── break_glass_process.py              # Enable/disable break-glass exception
│   └── audit_break_glass.py               # Audit break-glass usage via CloudTrail
├── validation/
│   ├── validate_policies.py               # Validate SCP JSON syntax and size limits
│   └── test_policy_effects.py             # Test SCP effects with simulated API calls
├── infrastructure/
│   └── config.py                          # Central configuration
├── run_setup.py                           # CLI orchestrator
└── requirements.txt
```

**config.py**: Central configuration dataclass with all parameters. Load from environment variables or CLI args. Validate required fields. Construct resource names using `{PROJECT_NAME}-{component}-{ENV}` convention. Parse comma-separated lists into Python lists. Validate OU ID format (`ou-*`). Validate role ARN format.

**policy_documents.py**: Define all SCP JSON policy documents as Python dictionaries:
- `get_instance_type_restriction_policy(approved_types, break_glass_arns)`: Returns SCP JSON that denies `sagemaker:CreateTrainingJob`, `sagemaker:CreateEndpointConfig`, `sagemaker:CreateProcessingJob`, `sagemaker:CreateNotebookInstance` when `sagemaker:InstanceTypes` is not in the approved list. Uses `StringNotLike` condition with `ForAnyValue` set operator. Includes `ArnNotLike` condition on `aws:PrincipalArn` for break-glass exemption.
- `get_encryption_enforcement_policy(break_glass_arns)`: Returns SCP JSON that denies `sagemaker:CreateTrainingJob`, `sagemaker:CreateEndpointConfig`, `sagemaker:CreateProcessingJob` when `sagemaker:VolumeKmsKey` is null (using `Null` condition operator). Separate statement denying when `sagemaker:OutputKmsKey` is null for training jobs. Includes break-glass exemption.
- `get_region_restriction_policy(approved_regions, break_glass_arns)`: Returns SCP JSON that denies `sagemaker:*`, `bedrock:*`, `glue:*`, `comprehend:*` when `aws:RequestedRegion` is not in the approved list. Uses `StringNotEquals` condition with `ForAnyValue`. Excludes global services (IAM, Organizations, STS). Includes break-glass exemption.
- `get_bedrock_model_restriction_policy(approved_models, break_glass_arns)`: Returns SCP JSON that denies `bedrock:InvokeModel`, `bedrock:InvokeModelWithResponseStream`, `bedrock:Converse`, `bedrock:ConverseStream` when `bedrock:ModelId` is not in the approved list. Uses `StringNotLike` condition for wildcard model ID matching. Includes break-glass exemption.
- `get_notebook_security_policy(break_glass_arns)`: Returns SCP JSON that denies `sagemaker:CreateNotebookInstance` when `sagemaker:DirectInternetAccess` is `Enabled` and when `sagemaker:RootAccess` is `Enabled`. Includes break-glass exemption.

**scp_instance_type_restriction.py**: Create SCP for instance type governance:
- Call `organizations.create_policy()` with `Type='SERVICE_CONTROL_POLICY'`
- Policy name: `{PROJECT_NAME}-scp-instance-types-{ENV}`
- Content from `get_instance_type_restriction_policy()`
- Validate policy size < 5,120 bytes before creation
- Tag with Project and Environment

**scp_encryption_enforcement.py**: Create SCP for KMS encryption enforcement:
- Call `organizations.create_policy()` with encryption enforcement policy
- Policy name: `{PROJECT_NAME}-scp-encryption-{ENV}`
- Only created when ENCRYPTION_REQUIRED is true
- Content from `get_encryption_enforcement_policy()`
- Validate policy size < 5,120 bytes

**scp_region_restriction.py**: Create SCP for region restriction:
- Call `organizations.create_policy()` with region restriction policy
- Policy name: `{PROJECT_NAME}-scp-region-restrict-{ENV}`
- Content from `get_region_restriction_policy()`
- Validate policy size < 5,120 bytes

**scp_bedrock_model_restriction.py**: Create SCP for Bedrock model governance:
- Call `organizations.create_policy()` with Bedrock model restriction policy
- Policy name: `{PROJECT_NAME}-scp-bedrock-models-{ENV}`
- Content from `get_bedrock_model_restriction_policy()`
- Validate policy size < 5,120 bytes

**scp_notebook_security.py**: Create SCP for notebook security:
- Call `organizations.create_policy()` with notebook security policy
- Policy name: `{PROJECT_NAME}-scp-notebook-security-{ENV}`
- Only created when DENY_INTERNET_ACCESS or DENY_ROOT_ACCESS is true
- Content from `get_notebook_security_policy()`

**attach_policies.py**: Attach all created SCPs to target OUs:
- For each policy ID and each OU in TARGET_OUS, call `organizations.attach_policy()`
- Verify attachment with `organizations.list_policies_for_target()`
- Log attachment results with policy name, OU ID, and status
- Handle `DuplicatePolicyAttachmentException` gracefully (already attached)

**detach_policies.py**: Detach SCPs from OUs for rollback:
- List all SCPs matching `{PROJECT_NAME}-scp-*-{ENV}` pattern
- For each policy and OU, call `organizations.detach_policy()`
- Optionally delete policies after detachment with `organizations.delete_policy()`
- Confirm detachment with `organizations.list_policies_for_target()`

**break_glass_process.py**: Break-glass exception management:
- `enable_break_glass(role_arn)`: Update SCP condition blocks to add a new exempt role ARN. Uses `organizations.update_policy()` to modify the policy content.
- `disable_break_glass(role_arn)`: Remove the role ARN from SCP exemptions. Restores full enforcement.
- `list_break_glass_roles()`: Parse current SCP documents and list all exempt role ARNs.
- Log all break-glass changes to CloudWatch Logs with timestamp, actor, and justification.

**audit_break_glass.py**: Audit break-glass usage via CloudTrail:
- Query CloudTrail for API calls made by break-glass roles using `cloudtrail.lookup_events()` with `LookupAttributes` filtering on the break-glass role ARN
- Filter for ML-related API calls (SageMaker, Bedrock, Glue)
- Generate audit report with: timestamp, role ARN, API action, resource ARN, source IP
- Publish audit summary to SNS for security team review

**validate_policies.py**: Validate SCP documents before creation:
- Check JSON syntax validity
- Check policy size < 5,120 bytes (SCP limit)
- Validate `Effect` is `Deny` (SCPs should use deny-list pattern for ML governance)
- Validate condition keys are valid for the specified actions
- Warn if policy approaches size limit (>4,500 bytes)

**test_policy_effects.py**: Test SCP effects:
- Use `organizations.simulate_custom_policy()` or `iam.simulate_principal_policy()` to test policy effects
- Test cases: approved instance type (should be allowed), unapproved instance type (should be denied), break-glass role with unapproved type (should be allowed), missing KMS key (should be denied), unapproved region (should be denied), unapproved Bedrock model (should be denied)
- Output pass/fail results for each test case

**run_setup.py**: CLI orchestrator that runs setup steps in order:
1. Load and validate configuration
2. Generate and validate all SCP policy documents
3. Create SCPs in Organizations
4. Attach SCPs to target OUs
5. Run policy effect tests
6. Print summary with policy IDs, attached OUs, and test results

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**SCP Size Limit:** Each SCP has a maximum size of 5,120 bytes. If a single policy exceeds this limit, split it into multiple SCPs. For instance type restrictions with many allowed types, use wildcard patterns (e.g., `ml.m5.*` instead of listing every m5 variant). Monitor policy sizes during creation and warn when approaching the limit.

**SCP Evaluation Logic:** SCPs are deny-list guardrails — they restrict what actions accounts in the OU can perform. SCPs do not grant permissions; they only limit existing permissions. An explicit `Deny` in an SCP overrides any `Allow` in identity or resource policies. The management account is never affected by SCPs.

**Break-Glass Pattern:** Break-glass roles must be tightly controlled. Use `aws:PrincipalArn` with `ArnNotLike` in the SCP condition to exempt specific roles. Break-glass roles should exist in every account under the OU but should only be assumable via a controlled process (e.g., approval workflow, MFA requirement). Audit all break-glass usage via CloudTrail.

**Condition Key Accuracy:** Use the correct condition keys for each service:
- SageMaker: `sagemaker:InstanceTypes` (list), `sagemaker:VolumeKmsKey` (string), `sagemaker:OutputKmsKey` (string), `sagemaker:DirectInternetAccess` (string), `sagemaker:RootAccess` (string)
- Bedrock: `bedrock:ModelId` (string) — applies to `InvokeModel`, `InvokeModelWithResponseStream`, `Converse`, `ConverseStream`
- Global: `aws:RequestedRegion` (string), `aws:PrincipalArn` (string)

**Region Restriction:** When restricting regions, exclude global services that must operate in `us-east-1` regardless of restriction: IAM, Organizations, STS, CloudFront, Route 53, WAF Global. The SCP should only target ML-specific service actions.

**Encryption Enforcement:** The `Null` condition operator checks whether a condition key is present in the request. `"Null": {"sagemaker:VolumeKmsKey": "true"}` matches when the key is NOT provided, enabling a deny when encryption is missing. This is the correct pattern for enforcing encryption via SCPs.

**Testing:** Always test SCPs in a sandbox OU before attaching to production OUs. Use `organizations.simulate_custom_policy()` to verify expected behavior. Monitor CloudTrail for `OrganizationsSCPFailure` events after attachment to detect unintended blocks.

**Security:** The management account or delegated administrator account must be used to create and manage SCPs. Store SCP policy documents in version control. Use `organizations.update_policy()` for modifications rather than delete-and-recreate to maintain policy ID stability. Tag all SCPs with Project and Environment.

**Naming:** All resources follow `{PROJECT_NAME}-{component}-{ENV}` convention:
- SCP policies: `{PROJECT_NAME}-scp-{purpose}-{ENV}` (e.g., `myai-scp-instance-types-prod`)
- CloudWatch log group: `/aws/organizations/{PROJECT_NAME}-scp-audit-{ENV}`

---

## Code Scaffolding Hints

**SCP: Deny unapproved SageMaker instance types:**
```python
import boto3
import json

organizations = boto3.client("organizations")

# SCP policy document — deny unapproved instance types
instance_type_policy = {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "DenyUnapprovedSageMakerInstanceTypes",
            "Effect": "Deny",
            "Action": [
                "sagemaker:CreateTrainingJob",
                "sagemaker:CreateHyperParameterTuningJob",
                "sagemaker:CreateProcessingJob",
                "sagemaker:CreateEndpointConfig",
                "sagemaker:CreateNotebookInstance",
                "sagemaker:CreateTransformJob",
            ],
            "Resource": "*",
            "Condition": {
                "ForAnyValue:StringNotLike": {
                    "sagemaker:InstanceTypes": APPROVED_INSTANCE_TYPES
                    # Example: ["ml.m5.*", "ml.g5.*", "ml.p3.2xlarge", "ml.inf2.*"]
                },
                "ArnNotLike": {
                    "aws:PrincipalArn": BREAK_GLASS_ROLE_ARNS
                    # Example: ["arn:aws:iam::*:role/BreakGlassAdmin"]
                },
            },
        }
    ],
}

# Create the SCP
response = organizations.create_policy(
    Content=json.dumps(instance_type_policy),
    Description=f"Deny unapproved SageMaker instance types for {PROJECT_NAME} {ENV}",
    Name=f"{PROJECT_NAME}-scp-instance-types-{ENV}",
    Type="SERVICE_CONTROL_POLICY",
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)
policy_id = response["Policy"]["PolicySummary"]["Id"]
print(f"Created SCP: {policy_id}")
```

**SCP: Enforce KMS encryption on SageMaker resources:**
```python
encryption_policy = {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "DenySageMakerWithoutVolumeEncryption",
            "Effect": "Deny",
            "Action": [
                "sagemaker:CreateTrainingJob",
                "sagemaker:CreateEndpointConfig",
                "sagemaker:CreateProcessingJob",
                "sagemaker:CreateNotebookInstance",
            ],
            "Resource": "*",
            "Condition": {
                "Null": {
                    "sagemaker:VolumeKmsKey": "true"
                },
                "ArnNotLike": {
                    "aws:PrincipalArn": BREAK_GLASS_ROLE_ARNS
                },
            },
        },
        {
            "Sid": "DenySageMakerWithoutOutputEncryption",
            "Effect": "Deny",
            "Action": [
                "sagemaker:CreateTrainingJob",
                "sagemaker:CreateProcessingJob",
                "sagemaker:CreateTransformJob",
            ],
            "Resource": "*",
            "Condition": {
                "Null": {
                    "sagemaker:OutputKmsKey": "true"
                },
                "ArnNotLike": {
                    "aws:PrincipalArn": BREAK_GLASS_ROLE_ARNS
                },
            },
        },
    ],
}

response = organizations.create_policy(
    Content=json.dumps(encryption_policy),
    Description=f"Enforce KMS encryption on SageMaker resources for {PROJECT_NAME} {ENV}",
    Name=f"{PROJECT_NAME}-scp-encryption-{ENV}",
    Type="SERVICE_CONTROL_POLICY",
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)
```

**SCP: Restrict ML workloads to approved regions:**
```python
region_restriction_policy = {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "DenyMLServicesOutsideApprovedRegions",
            "Effect": "Deny",
            "Action": [
                "sagemaker:*",
                "bedrock:*",
                "glue:*",
                "comprehend:*",
            ],
            "Resource": "*",
            "Condition": {
                "StringNotEquals": {
                    "aws:RequestedRegion": APPROVED_REGIONS
                    # Example: ["us-east-1", "us-west-2", "eu-west-1"]
                },
                "ArnNotLike": {
                    "aws:PrincipalArn": BREAK_GLASS_ROLE_ARNS
                },
            },
        }
    ],
}

response = organizations.create_policy(
    Content=json.dumps(region_restriction_policy),
    Description=f"Restrict ML services to approved regions for {PROJECT_NAME} {ENV}",
    Name=f"{PROJECT_NAME}-scp-region-restrict-{ENV}",
    Type="SERVICE_CONTROL_POLICY",
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)
```

**SCP: Deny unapproved Bedrock model access:**
```python
bedrock_model_policy = {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "DenyUnapprovedBedrockModels",
            "Effect": "Deny",
            "Action": [
                "bedrock:InvokeModel",
                "bedrock:InvokeModelWithResponseStream",
                "bedrock:Converse",
                "bedrock:ConverseStream",
            ],
            "Resource": "*",
            "Condition": {
                "ForAnyValue:StringNotLike": {
                    "bedrock:ModelId": APPROVED_BEDROCK_MODELS
                    # Example: ["us.anthropic.claude-sonnet-4-7*", "us.anthropic.claude-haiku-4-5*", "amazon.titan-embed*"]
                },
                "ArnNotLike": {
                    "aws:PrincipalArn": BREAK_GLASS_ROLE_ARNS
                },
            },
        }
    ],
}

response = organizations.create_policy(
    Content=json.dumps(bedrock_model_policy),
    Description=f"Restrict Bedrock model access to approved models for {PROJECT_NAME} {ENV}",
    Name=f"{PROJECT_NAME}-scp-bedrock-models-{ENV}",
    Type="SERVICE_CONTROL_POLICY",
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)
```

**SCP: Deny notebook internet access and root access:**
```python
notebook_security_policy = {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "DenyNotebookDirectInternetAccess",
            "Effect": "Deny",
            "Action": [
                "sagemaker:CreateNotebookInstance",
                "sagemaker:UpdateNotebookInstance",
            ],
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "sagemaker:DirectInternetAccess": "Enabled"
                },
                "ArnNotLike": {
                    "aws:PrincipalArn": BREAK_GLASS_ROLE_ARNS
                },
            },
        },
        {
            "Sid": "DenyNotebookRootAccess",
            "Effect": "Deny",
            "Action": [
                "sagemaker:CreateNotebookInstance",
                "sagemaker:UpdateNotebookInstance",
            ],
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "sagemaker:RootAccess": "Enabled"
                },
                "ArnNotLike": {
                    "aws:PrincipalArn": BREAK_GLASS_ROLE_ARNS
                },
            },
        },
    ],
}

response = organizations.create_policy(
    Content=json.dumps(notebook_security_policy),
    Description=f"Deny notebook internet and root access for {PROJECT_NAME} {ENV}",
    Name=f"{PROJECT_NAME}-scp-notebook-security-{ENV}",
    Type="SERVICE_CONTROL_POLICY",
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)
```

**Attach SCPs to target OUs:**
```python
def attach_policy_to_ous(policy_id, policy_name, target_ous):
    """Attach an SCP to all target organizational units."""
    for ou_id in target_ous:
        try:
            organizations.attach_policy(
                PolicyId=policy_id,
                TargetId=ou_id,
            )
            print(f"Attached {policy_name} to OU {ou_id}")
        except organizations.exceptions.DuplicatePolicyAttachmentException:
            print(f"Policy {policy_name} already attached to OU {ou_id}")
        except organizations.exceptions.PolicyNotFoundException:
            print(f"ERROR: Policy {policy_name} ({policy_id}) not found")
            raise

# Attach all ML governance SCPs
for policy_id, policy_name in created_policies:
    attach_policy_to_ous(policy_id, policy_name, TARGET_OUS)

# Verify attachments
for ou_id in TARGET_OUS:
    attached = organizations.list_policies_for_target(
        TargetId=ou_id,
        Filter="SERVICE_CONTROL_POLICY",
    )
    policy_names = [p["Name"] for p in attached["Policies"]]
    print(f"OU {ou_id} has SCPs: {policy_names}")
```

**Detach and rollback SCPs:**
```python
def detach_and_delete_policies(project_name, env, target_ous):
    """Detach all project SCPs from OUs and optionally delete them."""
    # List all SCPs in the organization
    paginator = organizations.get_paginator("list_policies")
    for page in paginator.paginate(Filter="SERVICE_CONTROL_POLICY"):
        for policy in page["Policies"]:
            if policy["Name"].startswith(f"{project_name}-scp-") and policy["Name"].endswith(f"-{env}"):
                policy_id = policy["Id"]
                # Detach from all target OUs
                for ou_id in target_ous:
                    try:
                        organizations.detach_policy(
                            PolicyId=policy_id,
                            TargetId=ou_id,
                        )
                        print(f"Detached {policy['Name']} from OU {ou_id}")
                    except organizations.exceptions.PolicyNotAttachedException:
                        pass
                # Delete the policy
                organizations.delete_policy(PolicyId=policy_id)
                print(f"Deleted SCP: {policy['Name']} ({policy_id})")
```

**Validate SCP policy size and syntax:**
```python
import json
import sys

SCP_MAX_SIZE = 5120  # bytes

def validate_policy(policy_document, policy_name):
    """Validate SCP JSON document before creation."""
    policy_json = json.dumps(policy_document, separators=(",", ":"))
    policy_size = len(policy_json.encode("utf-8"))

    if policy_size > SCP_MAX_SIZE:
        print(f"ERROR: {policy_name} exceeds SCP size limit: {policy_size} > {SCP_MAX_SIZE} bytes")
        print("Consider using wildcard patterns or splitting into multiple SCPs.")
        sys.exit(1)

    if policy_size > 4500:
        print(f"WARNING: {policy_name} approaching SCP size limit: {policy_size}/{SCP_MAX_SIZE} bytes")

    # Validate structure
    assert "Version" in policy_document, "Missing Version field"
    assert "Statement" in policy_document, "Missing Statement field"
    for stmt in policy_document["Statement"]:
        assert stmt.get("Effect") == "Deny", f"SCP statement should use Deny effect, got: {stmt.get('Effect')}"
        assert "Action" in stmt, "Missing Action field in statement"
        assert "Condition" in stmt, "Missing Condition field — SCPs without conditions deny all principals"

    print(f"VALID: {policy_name} ({policy_size} bytes)")
    return True
```

**Audit break-glass role usage via CloudTrail:**
```python
import boto3
from datetime import datetime, timedelta

cloudtrail = boto3.client("cloudtrail", region_name=AWS_REGION)
sns = boto3.client("sns", region_name=AWS_REGION)

def audit_break_glass_usage(break_glass_role_arns, lookback_hours=24):
    """Query CloudTrail for API calls made by break-glass roles."""
    end_time = datetime.utcnow()
    start_time = end_time - timedelta(hours=lookback_hours)

    ml_services = ["sagemaker", "bedrock", "glue", "comprehend"]
    audit_events = []

    for role_arn in break_glass_role_arns:
        # Extract role name from ARN for CloudTrail lookup
        role_name = role_arn.split("/")[-1]
        events = cloudtrail.lookup_events(
            LookupAttributes=[
                {"AttributeKey": "Username", "AttributeValue": role_name},
            ],
            StartTime=start_time,
            EndTime=end_time,
            MaxResults=50,
        )

        for event in events.get("Events", []):
            event_source = event.get("EventSource", "")
            if any(svc in event_source for svc in ml_services):
                audit_events.append({
                    "timestamp": event["EventTime"].isoformat(),
                    "role_arn": role_arn,
                    "action": event["EventName"],
                    "service": event_source,
                    "source_ip": event.get("SourceIPAddress", "unknown"),
                    "resources": [
                        r.get("ResourceName", "")
                        for r in event.get("Resources", [])
                    ],
                })

    # Publish audit report if any break-glass usage detected
    if audit_events:
        sns.publish(
            TopicArn=f"arn:aws:sns:{AWS_REGION}:{AWS_ACCOUNT_ID}:{PROJECT_NAME}-scp-audit-{ENV}",
            Subject=f"[{ENV.upper()}] Break-Glass Role Usage Detected — {PROJECT_NAME}",
            Message=json.dumps({"events": audit_events}, indent=2, default=str),
        )
        print(f"ALERT: {len(audit_events)} break-glass ML API calls detected")
    else:
        print("No break-glass ML API calls detected in the audit window")

    return audit_events
```

---

## Integration Points

- **Upstream**: `devops/04` → IAM roles referenced in break-glass exception patterns and SCP condition keys (`aws:PrincipalArn`)
- **Downstream**: `enterprise/04` → Control Tower guardrails consume SCPs as preventive controls for ML workload OUs
- **Downstream**: `devops/05` → Config rules provide detective controls that complement SCP preventive controls (e.g., Config detects non-compliant resources that SCPs didn't block due to pre-existing state)
- **Downstream**: `devops/08` → KMS encryption keys referenced by encryption enforcement SCPs; KMS key ARNs must exist before SCP enforcement is enabled
- **Related**: `enterprise/03` → Service Catalog launch constraints work alongside SCPs to enforce approved ML environment configurations
- **Related**: `finops/01` → Cost allocation tag policies in Organizations complement ML governance SCPs for budget enforcement
