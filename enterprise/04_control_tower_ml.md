<!-- Template Version: 1.0 | boto3: 1.35+ | cdk: 2.170+ -->

# Template Enterprise 04 — AWS Control Tower Landing Zone for ML Account Governance

## Purpose
Generate a production-ready AWS Control Tower governance layer for ML workloads: Customizations for AWS Control Tower (CfCT) manifest and CloudFormation templates that provision ML account baselines (VPC with SageMaker and Bedrock VPC endpoints, IAM execution roles, S3 buckets with encryption, CloudTrail logging, Config recorder), detective guardrails implemented as AWS Config rules (encryption enforcement, VPC-only training, approved instance types), preventive guardrails implemented as SCPs (deny unapproved models, deny unencrypted resources), a Control Tower lifecycle event hook that triggers a Step Functions state machine to deploy ML infrastructure into newly vended accounts, and a Config aggregator with compliance dashboard queries for centralized ML compliance visibility across all accounts in the ML organizational unit.

---

## Role Definition

You are an expert AWS cloud architect and multi-account governance specialist with expertise in:
- AWS Control Tower: landing zone setup, account factory, organizational units, guardrails (detective and preventive), lifecycle events, account baseline customizations
- Customizations for AWS Control Tower (CfCT): manifest.yaml schema, CloudFormation resource sets, SCP policy sets, deployment pipelines, nested OU targeting
- Detective guardrails: AWS Config managed and custom rules deployed via CfCT to detect non-compliant ML resources (unencrypted SageMaker volumes, direct internet access, unapproved instance types)
- Preventive guardrails: SCPs deployed via CfCT that proactively block non-compliant ML resource creation (unapproved Bedrock models, unencrypted training jobs, non-VPC endpoints)
- Account baseline deployment: CloudFormation StackSets for provisioning VPC infrastructure, VPC endpoints for SageMaker and Bedrock services, IAM execution roles, S3 buckets, CloudTrail, and Config recorder in every new ML account
- Control Tower lifecycle events: `CreateManagedAccount` and `UpdateManagedAccount` events published to EventBridge, triggering Step Functions workflows for post-account-creation ML infrastructure deployment
- AWS Config aggregator: multi-account, multi-region compliance aggregation with advanced SQL queries for ML resource compliance dashboards
- Step Functions orchestration: state machines that coordinate cross-service ML account baseline deployment with error handling, retries, and rollback
- CloudFormation StackSets: service-managed StackSets deployed from the management account or delegated administrator to target OUs

Generate complete, production-deployable Control Tower ML governance code.

---

## Context & Inputs

```
PROJECT_NAME:               [REQUIRED]
AWS_REGION:                 [REQUIRED]
AWS_ACCOUNT_ID:             [REQUIRED]
ENV:                        [REQUIRED - dev | stage | prod]

ML_OU_NAME:                 [REQUIRED - name of the organizational unit for ML accounts]
                            Example: "ML-Workloads"
                            The OU under which all ML accounts are organized. CfCT
                            customizations, guardrails, and account baselines target this OU.
                            Must already exist in the Control Tower landing zone.

ML_OU_ID:                   [REQUIRED - organizational unit ID]
                            Example: "ou-abc1-mlworkloads"
                            The OU ID used for SCP attachment and StackSet targeting.

ACCOUNT_BASELINE_CONFIG:    [REQUIRED - JSON object defining baseline infrastructure for ML accounts]
                            Example: {
                              "vpc_cidr": "10.0.0.0/16",
                              "private_subnet_cidrs": ["10.0.1.0/24","10.0.2.0/24"],
                              "enable_sagemaker_endpoint": true,
                              "enable_bedrock_endpoint": true,
                              "enable_s3_endpoint": true,
                              "enable_cloudtrail": true,
                              "enable_config_recorder": true,
                              "kms_key_alias": "alias/ml-workloads",
                              "log_retention_days": 365
                            }
                            Defines the VPC layout, VPC endpoints, logging, and encryption
                            settings deployed into every new ML account via CfCT StackSets.

DETECTIVE_GUARDRAILS:       [REQUIRED - JSON list of detective guardrail Config rule identifiers]
                            Example: ["sagemaker-endpoint-kms","sagemaker-vpc-only","sagemaker-approved-instance-types","bedrock-invocation-logging"]
                            Each identifier maps to a Config rule (managed or custom Lambda
                            evaluator) deployed via CfCT to detect non-compliant ML resources.

PREVENTIVE_GUARDRAILS:      [REQUIRED - JSON list of preventive guardrail SCP identifiers]
                            Example: ["deny-unapproved-instance-types","deny-unencrypted-sagemaker","deny-unapproved-bedrock-models","deny-public-notebook"]
                            Each identifier maps to an SCP deployed via CfCT to proactively
                            block non-compliant ML resource creation.

ACCOUNT_FACTORY_TEMPLATE:   [OPTIONAL: default-ml-account]
                            Name of the Account Factory template used when vending new ML
                            accounts. Maps to a CfCT resource set that deploys the baseline
                            CloudFormation stack.

APPROVED_INSTANCE_TYPES:    [OPTIONAL: ml.m5.xlarge,ml.m5.2xlarge,ml.g5.xlarge,ml.g5.2xlarge]
                            Comma-separated list of SageMaker instance types allowed by
                            detective and preventive guardrails.

APPROVED_BEDROCK_MODELS:    [OPTIONAL: anthropic.claude-3-sonnet-*,anthropic.claude-3-haiku-*,amazon.titan-embed-text-v2:0]
                            Comma-separated list of Bedrock model IDs allowed by preventive
                            guardrails. Supports wildcards.

AGGREGATOR_ACCOUNT_ID:      [OPTIONAL: same as AWS_ACCOUNT_ID]
                            Account ID where the Config aggregator is deployed for
                            centralized compliance visibility. Typically the audit or
                            security tooling account.

NOTIFICATION_EMAIL:         [OPTIONAL: none]
                            Email address for SNS notifications on guardrail violations
                            and account baseline deployment status.

LIFECYCLE_HOOK_ENABLED:     [OPTIONAL: true]
                            When true, deploys an EventBridge rule and Step Functions
                            state machine that automatically provisions ML infrastructure
                            when a new account is vended via Control Tower Account Factory.
```

---

## Task

Generate complete AWS Control Tower ML governance infrastructure:

```
{PROJECT_NAME}-control-tower-ml/
├── cfct/
│   ├── manifest.yaml                       # CfCT manifest — resource sets + policy sets
│   ├── templates/
│   │   ├── ml_account_baseline.yaml        # CloudFormation — VPC, endpoints, IAM, S3, logging
│   │   ├── detective_guardrails.yaml       # CloudFormation — Config rules for ML compliance
│   │   └── config_aggregator.yaml          # CloudFormation — Config aggregator + dashboard
│   └── policies/
│       ├── scp_ml_preventive.json          # SCP — deny unapproved instance types + models
│       ├── scp_encryption_required.json    # SCP — deny unencrypted SageMaker resources
│       └── scp_notebook_security.json      # SCP — deny public notebooks + root access
├── lifecycle/
│   ├── lifecycle_event_rule.py             # EventBridge rule for CreateManagedAccount
│   ├── state_machine_definition.json       # Step Functions — ML account baseline deployment
│   └── deploy_lifecycle_hook.py            # Deploy EventBridge rule + Step Functions
├── compliance/
│   ├── aggregator_queries.py               # Config aggregator SQL queries for ML compliance
│   └── compliance_report.py                # Generate compliance report across ML accounts
├── infrastructure/
│   └── config.py                           # Central configuration
├── run_setup.py                            # CLI orchestrator
└── requirements.txt
```

**config.py**: Central configuration dataclass with all parameters. Load from environment variables or CLI args. Validate required fields. Construct resource names using `{PROJECT_NAME}-{component}-{ENV}` convention. Parse ACCOUNT_BASELINE_CONFIG JSON into a Python dict. Parse DETECTIVE_GUARDRAILS and PREVENTIVE_GUARDRAILS JSON lists. Validate OU ID format (`ou-*`). Validate CIDR ranges in baseline config. Validate approved instance types and Bedrock model ID formats.

**manifest.yaml**: CfCT manifest file defining resource sets and policy sets for the ML OU:
- `region`: AWS_REGION
- `version`: `2021-03-15`
- `resource_sets`:
  - `ml-account-baseline`: deploys `ml_account_baseline.yaml` to all accounts in ML_OU_ID. Parameters include VPC CIDR, subnet CIDRs, endpoint flags, KMS key alias, log retention. Deployment targets: `OrganizationalUnit` with ML_OU_ID.
  - `ml-detective-guardrails`: deploys `detective_guardrails.yaml` to all accounts in ML_OU_ID. Parameters include approved instance types, approved Bedrock models. Deployment targets: `OrganizationalUnit` with ML_OU_ID.
  - `ml-config-aggregator`: deploys `config_aggregator.yaml` to the AGGREGATOR_ACCOUNT_ID only. Parameters include ML_OU_ID, aggregator name.
- `policy_sets`:
  - `ml-preventive-guardrails`: attaches `scp_ml_preventive.json` to ML_OU_ID.
  - `ml-encryption-guardrails`: attaches `scp_encryption_required.json` to ML_OU_ID.
  - `ml-notebook-guardrails`: attaches `scp_notebook_security.json` to ML_OU_ID.

**ml_account_baseline.yaml**: CloudFormation template deployed to every ML account via CfCT StackSet:
- Parameters: `ProjectName`, `Environment`, `VpcCidr`, `PrivateSubnetCidrs` (CommaDelimitedList), `EnableSageMakerEndpoint` (true/false), `EnableBedrockEndpoint` (true/false), `EnableS3Endpoint` (true/false), `KmsKeyAlias`, `LogRetentionDays`
- Resources:
  - `AWS::EC2::VPC` with `VpcCidr`, DNS support and hostnames enabled, tagged with Project and Environment
  - `AWS::EC2::Subnet` (two private subnets across AZs) with `MapPublicIpOnLaunch: false`
  - `AWS::EC2::RouteTable` and `AWS::EC2::SubnetRouteTableAssociation` for private subnets (no internet gateway — private only)
  - `AWS::EC2::SecurityGroup` for VPC endpoints: inbound HTTPS (443) from VPC CIDR, outbound all
  - `AWS::EC2::VPCEndpoint` (Interface) for `com.amazonaws.{REGION}.sagemaker.api` — conditional on `EnableSageMakerEndpoint`
  - `AWS::EC2::VPCEndpoint` (Interface) for `com.amazonaws.{REGION}.sagemaker.runtime` — conditional on `EnableSageMakerEndpoint`
  - `AWS::EC2::VPCEndpoint` (Interface) for `com.amazonaws.{REGION}.bedrock-runtime` — conditional on `EnableBedrockEndpoint`
  - `AWS::EC2::VPCEndpoint` (Interface) for `com.amazonaws.{REGION}.bedrock` — conditional on `EnableBedrockEndpoint`
  - `AWS::EC2::VPCEndpoint` (Gateway) for `com.amazonaws.{REGION}.s3` — conditional on `EnableS3Endpoint`, with route table association
  - `AWS::KMS::Key` with alias `KmsKeyAlias`, key rotation enabled, key policy granting account root full access and SageMaker service `kms:GenerateDataKey` and `kms:Decrypt`
  - `AWS::KMS::Alias` for the key
  - `AWS::IAM::Role` for SageMaker execution: trust policy for `sagemaker.amazonaws.com`, permissions for S3 read/write on project buckets, ECR pull, CloudWatch Logs, KMS decrypt on the baseline key. Role name: `{PROJECT_NAME}-sagemaker-exec-{ENV}`
  - `AWS::S3::Bucket` for ML artifacts: SSE-KMS encryption with the baseline key, versioning enabled, `PublicAccessBlockConfiguration` all true, lifecycle rule (IA after 30 days, Glacier IR after 90 days). Bucket name: `{PROJECT_NAME}-ml-artifacts-{ACCOUNT_ID}-{ENV}`
  - `AWS::S3::BucketPolicy` denying unencrypted uploads (`s3:PutObject` without `s3:x-amz-server-side-encryption`)
  - `AWS::CloudTrail::Trail` logging management events and S3 data events for the ML artifacts bucket, with CloudWatch Logs integration, KMS encryption. Trail name: `{PROJECT_NAME}-ml-trail-{ENV}`
  - `AWS::Config::ConfigurationRecorder` recording all resource types with `IncludeGlobalResourceTypes: true`
  - `AWS::Config::DeliveryChannel` delivering to an S3 bucket with SNS notification
- Outputs: VpcId, SubnetIds, SecurityGroupId, KmsKeyArn, KmsKeyAlias, SageMakerExecutionRoleArn, ArtifactBucketName, TrailArn

**detective_guardrails.yaml**: CloudFormation template deploying Config rules as detective guardrails:
- Parameters: `ProjectName`, `Environment`, `ApprovedInstanceTypes` (comma-separated), `ApprovedBedrockModels` (comma-separated)
- Resources:
  - `AWS::Config::ConfigRule` — `sagemaker-endpoint-configuration-kms-key-configured` (managed rule): detects SageMaker endpoint configs without KMS encryption
  - `AWS::Config::ConfigRule` — `sagemaker-notebook-instance-kms-key-configured` (managed rule): detects notebook instances without KMS encryption
  - `AWS::Config::ConfigRule` — `sagemaker-notebook-no-direct-internet-access` (managed rule): detects notebooks with direct internet access
  - `AWS::Config::ConfigRule` — custom rule `{PROJECT_NAME}-sagemaker-approved-instance-types`: Lambda evaluator that checks SageMaker endpoint configs and training jobs use only approved instance types from `ApprovedInstanceTypes` parameter
  - `AWS::Config::ConfigRule` — custom rule `{PROJECT_NAME}-sagemaker-vpc-only-training`: Lambda evaluator that checks SageMaker training jobs have `VpcConfig` specified (subnets and security groups)
  - `AWS::Config::ConfigRule` — custom rule `{PROJECT_NAME}-bedrock-invocation-logging`: Lambda evaluator that checks Bedrock model invocation logging is enabled in the account
  - `AWS::Lambda::Function` for each custom Config rule evaluator with inline Python code
  - `AWS::IAM::Role` for Lambda evaluators with `config.amazonaws.com` trust and `config:PutEvaluations`, `sagemaker:Describe*`, `bedrock:GetModelInvocationLoggingConfiguration` permissions
  - `AWS::SNS::Topic` for Config rule violation notifications
  - `AWS::Events::Rule` matching `Config Rules Compliance Change` events with `NON_COMPLIANT` status, targeting the SNS topic
- Outputs: ConfigRuleArns (comma-separated), SNSTopicArn

**config_aggregator.yaml**: CloudFormation template for centralized Config aggregator:
- Parameters: `ProjectName`, `Environment`, `MlOuId`
- Resources:
  - `AWS::Config::ConfigurationAggregator` with `OrganizationAggregationSource` targeting ML_OU_ID, all regions. Aggregator name: `{PROJECT_NAME}-ml-compliance-aggregator-{ENV}`
  - `AWS::IAM::Role` for the aggregator with `config.amazonaws.com` trust and `organizations:ListAccounts`, `organizations:ListAWSServiceAccessForOrganization`, `config:BatchGetAggregateResourceConfig` permissions
- Outputs: AggregatorName, AggregatorArn

**scp_ml_preventive.json**: SCP policy document denying unapproved ML resource creation:
- Statement 1: Deny `sagemaker:CreateTrainingJob`, `sagemaker:CreateEndpointConfig`, `sagemaker:CreateProcessingJob`, `sagemaker:CreateNotebookInstance` when `sagemaker:InstanceTypes` is not in APPROVED_INSTANCE_TYPES. Uses `StringNotLike` with `ForAnyValue` set operator.
- Statement 2: Deny `bedrock:InvokeModel`, `bedrock:InvokeModelWithResponseStream`, `bedrock:Converse`, `bedrock:ConverseStream` when `bedrock:ModelId` is not in APPROVED_BEDROCK_MODELS. Uses `StringNotLike` condition.
- Both statements include `ArnNotLike` condition on `aws:PrincipalArn` for break-glass role exemption.

**scp_encryption_required.json**: SCP policy document enforcing encryption:
- Statement 1: Deny `sagemaker:CreateTrainingJob`, `sagemaker:CreateEndpointConfig`, `sagemaker:CreateProcessingJob` when `sagemaker:VolumeKmsKey` is null (using `Null` condition operator set to `true`).
- Statement 2: Deny `sagemaker:CreateTrainingJob` when `sagemaker:OutputKmsKey` is null.
- Includes break-glass role exemption.

**scp_notebook_security.json**: SCP policy document for notebook security:
- Statement 1: Deny `sagemaker:CreateNotebookInstance` when `sagemaker:DirectInternetAccess` equals `Enabled`.
- Statement 2: Deny `sagemaker:CreateNotebookInstance` when `sagemaker:RootAccess` equals `Enabled`.
- Includes break-glass role exemption.

**lifecycle_event_rule.py**: Deploy EventBridge rule for Control Tower lifecycle events:
- Create EventBridge rule matching `AWS Service Event via CloudTrail` with detail-type `AWS Service Event via CloudTrail` and source `aws.controltower`
- Event pattern filters for `CreateManagedAccount` and `UpdateManagedAccount` events with `serviceEventDetails.createManagedAccountStatus.state` equal to `SUCCEEDED`
- Target: Step Functions state machine ARN for ML account baseline deployment
- Rule name: `{PROJECT_NAME}-ct-lifecycle-ml-{ENV}`

**state_machine_definition.json**: Step Functions state machine for ML account baseline deployment:
- Input: Control Tower lifecycle event payload containing new account ID, account name, OU ID
- Step 1 — `ExtractAccountInfo`: Pass state extracting `accountId`, `accountName` from the event payload
- Step 2 — `WaitForAccountReady`: Wait 60 seconds for account to stabilize after creation
- Step 3 — `AssumeRoleInNewAccount`: Lambda task that assumes `AWSControlTowerExecution` role in the new account and returns temporary credentials
- Step 4 — `DeployVPCEndpoints`: Lambda task that creates VPC endpoints for SageMaker and Bedrock in the new account using the assumed role credentials (supplements the CfCT baseline if additional endpoints are needed)
- Step 5 — `ValidateBaseline`: Lambda task that verifies the CfCT StackSet instance deployed successfully by checking CloudFormation stack status in the new account
- Step 6 — `EnableConfigRecorder`: Lambda task that verifies Config recorder is active in the new account
- Step 7 — `NotifySuccess`: SNS publish task notifying NOTIFICATION_EMAIL of successful ML account baseline deployment
- Error handling: `Catch` on each step with `States.ALL` routing to `NotifyFailure` state that publishes error details to SNS
- Retry: each Lambda task retries 3 times with exponential backoff (2, 4, 8 seconds)

**deploy_lifecycle_hook.py**: Deploy the EventBridge rule and Step Functions state machine:
- Create IAM role for Step Functions with trust policy for `states.amazonaws.com`, permissions for `lambda:InvokeFunction`, `sns:Publish`, `sts:AssumeRole`
- Create IAM role for EventBridge with trust policy for `events.amazonaws.com`, permissions for `states:StartExecution`
- Upload state machine definition and create Step Functions state machine using `stepfunctions.create_state_machine()`
- Create EventBridge rule and target using `events.put_rule()` and `events.put_targets()`
- State machine name: `{PROJECT_NAME}-ml-account-baseline-{ENV}`

**aggregator_queries.py**: Config aggregator SQL queries for ML compliance:
- `get_ml_compliance_summary()`: Query returning compliance percentage across all ML accounts: `SELECT accountId, configRuleName, complianceType, COUNT(*) FROM aws_config_compliance_summary WHERE configRuleName LIKE '{PROJECT_NAME}%' GROUP BY accountId, configRuleName, complianceType`
- `get_noncompliant_sagemaker_resources()`: Query returning all non-compliant SageMaker resources: `SELECT resourceId, resourceType, accountId, awsRegion, configuration FROM aws_config_resource WHERE resourceType LIKE 'AWS::SageMaker::%' AND complianceType = 'NON_COMPLIANT'`
- `get_unencrypted_endpoints()`: Query returning SageMaker endpoints without KMS encryption
- `get_vpc_compliance()`: Query returning SageMaker training jobs not running in VPC
- Execute queries using `config.select_aggregate_resource_config()` with the aggregator name

**compliance_report.py**: Generate compliance report across ML accounts:
- Call each query function from `aggregator_queries.py`
- Aggregate results into a JSON report with: timestamp, aggregator name, total accounts, compliant accounts, non-compliant accounts, per-rule compliance percentages, list of non-compliant resources with account ID, resource ID, and rule name
- Optionally publish report to S3 and send summary via SNS
- Output report to stdout in JSON format

**run_setup.py**: CLI orchestrator that runs setup steps in order:
1. Load and validate configuration
2. Generate CfCT manifest.yaml with resource sets and policy sets
3. Generate CloudFormation templates (baseline, detective guardrails, aggregator)
4. Generate SCP policy documents
5. Deploy lifecycle event hook (EventBridge rule + Step Functions) if LIFECYCLE_HOOK_ENABLED
6. Deploy Config aggregator (if AGGREGATOR_ACCOUNT_ID provided)
7. Print summary with manifest path, template paths, policy paths, state machine ARN, and aggregator name

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**CfCT Manifest Compliance:** The `manifest.yaml` must conform to the CfCT v2 schema (`version: 2021-03-15`). Resource sets must specify `deployment_targets` with `organizational_units` targeting ML_OU_ID. Policy sets must specify `type: scp` and target the same OU. All CloudFormation templates referenced in resource sets must be relative paths under the `templates/` directory. CfCT deploys via CodePipeline — the manifest and templates are committed to a CodeCommit repository.

**Guardrail Layering:** Detective guardrails (Config rules) detect existing non-compliant resources and report findings. Preventive guardrails (SCPs) proactively block non-compliant resource creation. Both layers must be deployed together for defense-in-depth. Detective guardrails catch resources created before preventive guardrails were active. Preventive guardrails prevent new violations. Never rely on only one layer.

**Account Baseline Idempotency:** The `ml_account_baseline.yaml` CloudFormation template must be idempotent — deploying it multiple times to the same account must not fail or create duplicate resources. Use `Conditions` to skip resource creation when resources already exist. Use `DeletionPolicy: Retain` on VPC, S3 buckets, and KMS keys to prevent accidental deletion during stack updates.

**Lifecycle Event Handling:** The Step Functions state machine triggered by Control Tower lifecycle events must handle both `CreateManagedAccount` and `UpdateManagedAccount` events. The state machine must wait for the account to stabilize (60-second wait) before attempting cross-account operations. All Lambda tasks must assume the `AWSControlTowerExecution` role in the target account. Error handling must catch all failures and notify via SNS without leaving the account in a partially configured state.

**Config Aggregator Permissions:** The Config aggregator requires `organizations:ListAccounts` and `organizations:ListAWSServiceAccessForOrganization` permissions. The aggregator IAM role must be in the AGGREGATOR_ACCOUNT_ID. AWS Config must be enabled as a trusted service in Organizations via `organizations.enable_aws_service_access(ServicePrincipal='config-multiaccountsetup.amazonaws.com')`.

**SCP Size Limits:** Each SCP policy document must be under 5,120 bytes. If a policy exceeds this limit, split it into multiple SCPs. The CfCT manifest supports multiple policy sets targeting the same OU. Validate policy size before deployment.

**Encryption:** All baseline resources must use KMS encryption. The KMS key created in `ml_account_baseline.yaml` must have key rotation enabled. The key policy must grant the SageMaker execution role `kms:GenerateDataKey` and `kms:Decrypt`. S3 bucket policies must deny unencrypted uploads. Reference `devops/08` for KMS key management patterns.

**VPC Endpoint Security:** VPC endpoints created in the baseline must have security groups restricting inbound traffic to HTTPS (port 443) from the VPC CIDR only. Interface endpoints must have `PrivateDnsEnabled: true` for seamless SDK integration. The S3 gateway endpoint must be associated with all private route tables. Reference `devops/09` for VPC endpoint policy patterns.

**Naming:** All resources follow `{PROJECT_NAME}-{component}-{ENV}` convention:
- VPC: `{PROJECT_NAME}-ml-vpc-{ENV}`
- Subnets: `{PROJECT_NAME}-ml-private-{AZ}-{ENV}`
- Security group: `{PROJECT_NAME}-ml-vpce-sg-{ENV}`
- KMS key alias: `alias/{PROJECT_NAME}-ml-{ENV}`
- S3 bucket: `{PROJECT_NAME}-ml-artifacts-{ACCOUNT_ID}-{ENV}`
- IAM role: `{PROJECT_NAME}-sagemaker-exec-{ENV}`
- CloudTrail: `{PROJECT_NAME}-ml-trail-{ENV}`
- Config rules: `{PROJECT_NAME}-{rule-name}-{ENV}`
- SCPs: `{PROJECT_NAME}-scp-{purpose}-{ENV}`
- Step Functions: `{PROJECT_NAME}-ml-account-baseline-{ENV}`
- Config aggregator: `{PROJECT_NAME}-ml-compliance-aggregator-{ENV}`

**Security:** The CfCT CodePipeline must run in the management account or delegated administrator account. SCP policies must include break-glass role exemptions for emergency access. The lifecycle event Step Functions role must have minimal permissions — only `sts:AssumeRole` on `AWSControlTowerExecution` in target accounts, `lambda:InvokeFunction` on baseline Lambdas, and `sns:Publish` on the notification topic. Config rule Lambda evaluators must have read-only permissions on the resources they evaluate.

---

## Code Scaffolding Hints

**CfCT manifest.yaml:**
```yaml
---
region: us-east-1
version: 2021-03-15

resources:
  - name: ml-account-baseline
    description: "ML account baseline — VPC, endpoints, IAM, S3, logging"
    resource_file: templates/ml_account_baseline.yaml
    parameters:
      - parameter_key: ProjectName
        parameter_value: myai
      - parameter_key: Environment
        parameter_value: prod
      - parameter_key: VpcCidr
        parameter_value: "10.0.0.0/16"
      - parameter_key: PrivateSubnetCidrs
        parameter_value: "10.0.1.0/24,10.0.2.0/24"
      - parameter_key: EnableSageMakerEndpoint
        parameter_value: "true"
      - parameter_key: EnableBedrockEndpoint
        parameter_value: "true"
      - parameter_key: EnableS3Endpoint
        parameter_value: "true"
      - parameter_key: KmsKeyAlias
        parameter_value: "alias/myai-ml-prod"
      - parameter_key: LogRetentionDays
        parameter_value: "365"
    deployment_targets:
      organizational_units:
        - ou-abc1-mlworkloads

  - name: ml-detective-guardrails
    description: "Detective guardrails — Config rules for ML compliance"
    resource_file: templates/detective_guardrails.yaml
    parameters:
      - parameter_key: ProjectName
        parameter_value: myai
      - parameter_key: Environment
        parameter_value: prod
      - parameter_key: ApprovedInstanceTypes
        parameter_value: "ml.m5.xlarge,ml.m5.2xlarge,ml.g5.xlarge,ml.g5.2xlarge"
      - parameter_key: ApprovedBedrockModels
        parameter_value: "anthropic.claude-3-sonnet-*,anthropic.claude-3-haiku-*"
    deployment_targets:
      organizational_units:
        - ou-abc1-mlworkloads

  - name: ml-config-aggregator
    description: "Centralized Config aggregator for ML compliance"
    resource_file: templates/config_aggregator.yaml
    parameters:
      - parameter_key: ProjectName
        parameter_value: myai
      - parameter_key: Environment
        parameter_value: prod
      - parameter_key: MlOuId
        parameter_value: ou-abc1-mlworkloads
    deployment_targets:
      accounts:
        - "123456789012"  # Audit/security tooling account

organization_policies:
  - name: ml-preventive-guardrails
    description: "Preventive guardrails — deny unapproved instance types and models"
    policy_file: policies/scp_ml_preventive.json
    type: scp
    deployment_targets:
      organizational_units:
        - ou-abc1-mlworkloads

  - name: ml-encryption-guardrails
    description: "Preventive guardrails — enforce KMS encryption"
    policy_file: policies/scp_encryption_required.json
    type: scp
    deployment_targets:
      organizational_units:
        - ou-abc1-mlworkloads

  - name: ml-notebook-guardrails
    description: "Preventive guardrails — deny public notebooks and root access"
    policy_file: policies/scp_notebook_security.json
    type: scp
    deployment_targets:
      organizational_units:
        - ou-abc1-mlworkloads
```

**SCP policy document — deny unapproved instance types and Bedrock models:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnapprovedSageMakerInstanceTypes",
      "Effect": "Deny",
      "Action": [
        "sagemaker:CreateTrainingJob",
        "sagemaker:CreateEndpointConfig",
        "sagemaker:CreateProcessingJob",
        "sagemaker:CreateNotebookInstance"
      ],
      "Resource": "*",
      "Condition": {
        "ForAnyValue:StringNotLike": {
          "sagemaker:InstanceTypes": [
            "ml.m5.xlarge",
            "ml.m5.2xlarge",
            "ml.g5.xlarge",
            "ml.g5.2xlarge"
          ]
        },
        "ArnNotLike": {
          "aws:PrincipalArn": "arn:aws:iam::*:role/BreakGlassAdmin"
        }
      }
    },
    {
      "Sid": "DenyUnapprovedBedrockModels",
      "Effect": "Deny",
      "Action": [
        "bedrock:InvokeModel",
        "bedrock:InvokeModelWithResponseStream",
        "bedrock:Converse",
        "bedrock:ConverseStream"
      ],
      "Resource": "*",
      "Condition": {
        "ForAnyValue:StringNotLike": {
          "bedrock:ModelId": [
            "anthropic.claude-3-sonnet-*",
            "anthropic.claude-3-haiku-*",
            "amazon.titan-embed-text-v2:0"
          ]
        },
        "ArnNotLike": {
          "aws:PrincipalArn": "arn:aws:iam::*:role/BreakGlassAdmin"
        }
      }
    }
  ]
}
```

**Custom Config rule Lambda evaluator — approved instance types:**
```python
import boto3
import json

config_client = boto3.client("config")
sagemaker_client = boto3.client("sagemaker")

APPROVED_INSTANCE_TYPES = [
    "ml.m5.xlarge", "ml.m5.2xlarge", "ml.g5.xlarge", "ml.g5.2xlarge"
]


def handler(event, context):
    """Config rule evaluator: check SageMaker resources use approved instance types."""
    invoking_event = json.loads(event["invokingEvent"])
    configuration_item = invoking_event.get("configurationItem", {})
    resource_type = configuration_item.get("resourceType", "")
    resource_id = configuration_item.get("resourceId", "")
    configuration = configuration_item.get("configuration", {})

    compliance_type = "COMPLIANT"
    annotation = "Resource uses approved instance types"

    if resource_type == "AWS::SageMaker::EndpointConfig":
        # Check production variants for unapproved instance types
        variants = configuration.get("productionVariants", [])
        for variant in variants:
            instance_type = variant.get("instanceType", "")
            if instance_type and instance_type not in APPROVED_INSTANCE_TYPES:
                compliance_type = "NON_COMPLIANT"
                annotation = f"Unapproved instance type: {instance_type}"
                break

    elif resource_type == "AWS::SageMaker::NotebookInstance":
        instance_type = configuration.get("instanceType", "")
        if instance_type and instance_type not in APPROVED_INSTANCE_TYPES:
            compliance_type = "NON_COMPLIANT"
            annotation = f"Unapproved instance type: {instance_type}"

    config_client.put_evaluations(
        Evaluations=[
            {
                "ComplianceResourceType": resource_type,
                "ComplianceResourceId": resource_id,
                "ComplianceType": compliance_type,
                "Annotation": annotation,
                "OrderingTimestamp": configuration_item.get(
                    "configurationItemCaptureTime"
                ),
            }
        ],
        ResultToken=event["resultToken"],
    )
    return {"compliance_type": compliance_type, "annotation": annotation}
```

**CloudFormation snippet — VPC endpoints for SageMaker and Bedrock:**
```yaml
Parameters:
  VpcId:
    Type: AWS::EC2::VPC::Id
  SubnetIds:
    Type: List<AWS::EC2::Subnet::Id>
  VpcCidr:
    Type: String
  EnableSageMakerEndpoint:
    Type: String
    Default: "true"
    AllowedValues: ["true", "false"]
  EnableBedrockEndpoint:
    Type: String
    Default: "true"
    AllowedValues: ["true", "false"]

Conditions:
  CreateSageMakerEndpoint: !Equals [!Ref EnableSageMakerEndpoint, "true"]
  CreateBedrockEndpoint: !Equals [!Ref EnableBedrockEndpoint, "true"]

Resources:
  VPCEndpointSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: "Security group for ML VPC endpoints"
      VpcId: !Ref VpcId
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: !Ref VpcCidr
          Description: "HTTPS from VPC"
      Tags:
        - Key: Name
          Value: !Sub "${ProjectName}-ml-vpce-sg-${Environment}"

  SageMakerAPIEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Condition: CreateSageMakerEndpoint
    Properties:
      VpcId: !Ref VpcId
      ServiceName: !Sub "com.amazonaws.${AWS::Region}.sagemaker.api"
      VpcEndpointType: Interface
      PrivateDnsEnabled: true
      SubnetIds: !Ref SubnetIds
      SecurityGroupIds:
        - !Ref VPCEndpointSecurityGroup

  SageMakerRuntimeEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Condition: CreateSageMakerEndpoint
    Properties:
      VpcId: !Ref VpcId
      ServiceName: !Sub "com.amazonaws.${AWS::Region}.sagemaker.runtime"
      VpcEndpointType: Interface
      PrivateDnsEnabled: true
      SubnetIds: !Ref SubnetIds
      SecurityGroupIds:
        - !Ref VPCEndpointSecurityGroup

  BedrockEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Condition: CreateBedrockEndpoint
    Properties:
      VpcId: !Ref VpcId
      ServiceName: !Sub "com.amazonaws.${AWS::Region}.bedrock"
      VpcEndpointType: Interface
      PrivateDnsEnabled: true
      SubnetIds: !Ref SubnetIds
      SecurityGroupIds:
        - !Ref VPCEndpointSecurityGroup

  BedrockRuntimeEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Condition: CreateBedrockEndpoint
    Properties:
      VpcId: !Ref VpcId
      ServiceName: !Sub "com.amazonaws.${AWS::Region}.bedrock-runtime"
      VpcEndpointType: Interface
      PrivateDnsEnabled: true
      SubnetIds: !Ref SubnetIds
      SecurityGroupIds:
        - !Ref VPCEndpointSecurityGroup
```

**Step Functions state machine for ML account baseline deployment:**
```json
{
  "Comment": "ML account baseline deployment triggered by Control Tower lifecycle events",
  "StartAt": "ExtractAccountInfo",
  "States": {
    "ExtractAccountInfo": {
      "Type": "Pass",
      "Parameters": {
        "accountId.$": "$.detail.serviceEventDetails.createManagedAccountStatus.account.accountId",
        "accountName.$": "$.detail.serviceEventDetails.createManagedAccountStatus.account.accountName",
        "ouId.$": "$.detail.serviceEventDetails.createManagedAccountStatus.organizationalUnit.organizationalUnitId"
      },
      "Next": "WaitForAccountReady"
    },
    "WaitForAccountReady": {
      "Type": "Wait",
      "Seconds": 60,
      "Next": "ValidateBaseline"
    },
    "ValidateBaseline": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT:function:PROJECT-validate-baseline-ENV",
      "Parameters": {
        "accountId.$": "$.accountId",
        "stackName": "StackSet-ml-account-baseline"
      },
      "Retry": [
        {
          "ErrorEquals": ["States.ALL"],
          "IntervalSeconds": 30,
          "MaxAttempts": 5,
          "BackoffRate": 2.0
        }
      ],
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "Next": "NotifyFailure",
          "ResultPath": "$.error"
        }
      ],
      "Next": "EnableConfigRecorder"
    },
    "EnableConfigRecorder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT:function:PROJECT-enable-config-ENV",
      "Parameters": {
        "accountId.$": "$.accountId"
      },
      "Retry": [
        {
          "ErrorEquals": ["States.ALL"],
          "IntervalSeconds": 10,
          "MaxAttempts": 3,
          "BackoffRate": 2.0
        }
      ],
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "Next": "NotifyFailure",
          "ResultPath": "$.error"
        }
      ],
      "Next": "NotifySuccess"
    },
    "NotifySuccess": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:REGION:ACCOUNT:PROJECT-ct-lifecycle-notifications-ENV",
        "Subject": "ML Account Baseline Deployed Successfully",
        "Message.$": "States.Format('ML baseline deployed for account {} ({})', $.accountId, $.accountName)"
      },
      "End": true
    },
    "NotifyFailure": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:REGION:ACCOUNT:PROJECT-ct-lifecycle-notifications-ENV",
        "Subject": "ML Account Baseline Deployment FAILED",
        "Message.$": "States.Format('FAILED: ML baseline for account {} — Error: {}', $.accountId, $.error.Cause)"
      },
      "End": true
    }
  }
}
```

**Deploy EventBridge lifecycle event rule:**
```python
import boto3
import json

events = boto3.client("events", region_name=AWS_REGION)

# EventBridge rule for Control Tower lifecycle events
rule_response = events.put_rule(
    Name=f"{PROJECT_NAME}-ct-lifecycle-ml-{ENV}",
    EventPattern=json.dumps({
        "source": ["aws.controltower"],
        "detail-type": ["AWS Service Event via CloudTrail"],
        "detail": {
            "eventName": ["CreateManagedAccount", "UpdateManagedAccount"],
            "serviceEventDetails": {
                "createManagedAccountStatus": {
                    "state": ["SUCCEEDED"]
                }
            }
        }
    }),
    State="ENABLED",
    Description=f"Trigger ML account baseline on new Control Tower account creation",
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)
print(f"Created EventBridge rule: {rule_response['RuleArn']}")

# Add Step Functions state machine as target
events.put_targets(
    Rule=f"{PROJECT_NAME}-ct-lifecycle-ml-{ENV}",
    Targets=[
        {
            "Id": "ml-baseline-state-machine",
            "Arn": STATE_MACHINE_ARN,
            "RoleArn": EVENTS_ROLE_ARN,  # Role with states:StartExecution permission
        }
    ],
)
print(f"Added target: {STATE_MACHINE_ARN}")
```

**Config aggregator SQL queries for ML compliance:**
```python
import boto3
import json

config = boto3.client("config", region_name=AWS_REGION)

AGGREGATOR_NAME = f"{PROJECT_NAME}-ml-compliance-aggregator-{ENV}"


def get_ml_compliance_summary():
    """Get compliance summary across all ML accounts."""
    response = config.select_aggregate_resource_config(
        Expression="""
            SELECT
                accountId,
                configRuleName,
                complianceType,
                COUNT(*)
            WHERE
                resourceType LIKE 'AWS::SageMaker::%'
            GROUP BY
                accountId,
                configRuleName,
                complianceType
        """,
        ConfigurationAggregatorName=AGGREGATOR_NAME,
    )
    results = [json.loads(r) for r in response.get("Results", [])]
    print(f"ML compliance summary: {len(results)} entries")
    return results


def get_noncompliant_resources():
    """Get all non-compliant SageMaker resources across ML accounts."""
    response = config.select_aggregate_resource_config(
        Expression="""
            SELECT
                resourceId,
                resourceType,
                accountId,
                awsRegion,
                complianceType
            WHERE
                resourceType LIKE 'AWS::SageMaker::%'
                AND complianceType = 'NON_COMPLIANT'
        """,
        ConfigurationAggregatorName=AGGREGATOR_NAME,
    )
    results = [json.loads(r) for r in response.get("Results", [])]
    print(f"Non-compliant ML resources: {len(results)}")
    for r in results:
        print(f"  {r['accountId']} | {r['resourceType']} | {r['resourceId']}")
    return results


def get_compliance_percentage():
    """Calculate overall ML compliance percentage."""
    response = config.select_aggregate_resource_config(
        Expression="""
            SELECT
                complianceType,
                COUNT(*) as count
            WHERE
                resourceType LIKE 'AWS::SageMaker::%'
            GROUP BY
                complianceType
        """,
        ConfigurationAggregatorName=AGGREGATOR_NAME,
    )
    results = [json.loads(r) for r in response.get("Results", [])]
    compliant = sum(r["count"] for r in results if r["complianceType"] == "COMPLIANT")
    total = sum(r["count"] for r in results)
    pct = (compliant / total * 100) if total > 0 else 0
    print(f"ML compliance: {pct:.1f}% ({compliant}/{total} resources)")
    return {"compliant": compliant, "total": total, "percentage": round(pct, 1)}
```

**Create Config aggregator:**
```python
import boto3

config = boto3.client("config", region_name=AWS_REGION)

response = config.put_configuration_aggregator(
    ConfigurationAggregatorName=f"{PROJECT_NAME}-ml-compliance-aggregator-{ENV}",
    OrganizationAggregationSource={
        "RoleArn": AGGREGATOR_ROLE_ARN,
        "AwsRegions": [AWS_REGION],  # or omit for all regions
        "AllAwsRegions": False,
    },
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)
print(f"Created Config aggregator: {response['ConfigurationAggregator']['ConfigurationAggregatorName']}")
```

---

## Integration Points

- **Upstream**: `enterprise/01` (Organizations SCPs) → Provides SCP policy patterns and condition key references reused by the preventive guardrails in this template; the CfCT policy sets deploy SCPs following the same deny-list patterns defined in `enterprise/01`
- **Upstream**: `devops/05` (Config Rules ML Compliance) → Provides Config rule definitions and Lambda evaluator patterns reused by the detective guardrails in this template; the CfCT resource sets deploy Config rules following the same evaluation logic
- **Upstream**: `devops/09` (VPC Endpoint Policies) → Provides VPC endpoint configuration patterns and endpoint policy documents used by the account baseline template when creating SageMaker and Bedrock VPC endpoints
- **Upstream**: `devops/04` (IAM Roles & Policies) → Provides IAM role patterns for the SageMaker execution role, Step Functions execution role, Config rule Lambda evaluator roles, and Config aggregator role created in the account baseline
- **Upstream**: `devops/08` (KMS Encryption) → Provides KMS key management patterns used by the account baseline template for creating per-account ML encryption keys with rotation and scoped key policies
- **Downstream**: `enterprise/03` (Service Catalog) → Control Tower account baselines can include Service Catalog portfolio sharing to automatically grant new ML accounts access to the self-service ML portfolio; the lifecycle event hook can trigger portfolio association after account creation
- **Downstream**: `enterprise/02` (Cross-Account Model Deployment) → Account baselines provision the IAM roles and VPC infrastructure required for cross-account model deployment pipelines; the SageMaker execution role and KMS key created in each ML account are consumed by cross-account CodePipeline actions
- **Downstream**: `enterprise/05` (Centralized Model Registry) → Config aggregator compliance data feeds into centralized governance dashboards; account baselines ensure each ML account has the IAM permissions and network connectivity to publish models to the central registry
- **Downstream**: `devops/03` (CloudWatch Monitoring) → Config rule violation events and lifecycle event notifications feed into CloudWatch dashboards for operational visibility across ML accounts
