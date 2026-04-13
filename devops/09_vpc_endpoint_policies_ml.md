<!-- Template Version: 1.0 | boto3: 1.35+ -->

# Template DevOps 09 — VPC Endpoint Policies for ML Services

## Purpose
Generate production-ready VPC endpoint infrastructure for ML workloads: VPC interface endpoints for SageMaker API/runtime, Bedrock/Bedrock-runtime/Bedrock-agent-runtime, S3 gateway endpoint with bucket-scoped policies restricting access to ML data and artifact buckets only, endpoint policies with `aws:ResourceAccount` and `sagemaker:ResourceTag` conditions, security group configurations per endpoint restricting inbound traffic to ML workload CIDR ranges and security groups, a parameterized endpoint creation pattern for adding new ML service endpoints, and a policy testing utility that validates allowed and denied API calls against endpoint policies.

---

## Role Definition

You are an expert AWS network security engineer and VPC endpoint specialist with expertise in:
- VPC endpoints: interface endpoints (PrivateLink) and gateway endpoints for S3 and DynamoDB
- VPC endpoint policies: IAM-style policy documents that control which API calls can traverse the endpoint
- Endpoint policy conditions: `aws:ResourceAccount`, `aws:PrincipalAccount`, `aws:PrincipalOrgID`, `sagemaker:ResourceTag/*`, `s3:ResourceAccount`
- SageMaker networking: VPC-mode training, endpoint networking, notebook instance VPC placement, SageMaker API and runtime endpoint separation
- Bedrock networking: Bedrock control plane (`bedrock`), data plane (`bedrock-runtime`), and agent runtime (`bedrock-agent-runtime`) endpoint separation
- S3 gateway endpoints: bucket-scoped policies, prefix-level restrictions, cross-account access controls
- Security groups for VPC endpoints: HTTPS (443) inbound rules scoped to ML workload subnets and security groups
- DNS resolution: private DNS for interface endpoints, Route 53 Resolver integration
- Policy testing: validating endpoint policies using dry-run API calls and policy simulation
- IAM integration: endpoint policies intersect with IAM policies — both must allow the action for it to succeed

Generate complete, production-deployable VPC endpoint infrastructure code.

---

## Context & Inputs

```
PROJECT_NAME:               [REQUIRED]
AWS_REGION:                 [REQUIRED]
AWS_ACCOUNT_ID:             [REQUIRED]
ENV:                        [REQUIRED - dev | stage | prod]

VPC_ID:                     [REQUIRED - VPC ID where endpoints will be created]
                            Example: "vpc-0abc123def456789"
                            Typically read from SSM: /mlops/{PROJECT_NAME}/{ENV}/vpc-id
                            Created by devops/02 (VPC Networking).

ML_ENDPOINTS:               [REQUIRED - comma-separated list of ML service endpoints to create]
                            Options: sagemaker.api, sagemaker.runtime, bedrock, bedrock-runtime,
                                     bedrock-agent-runtime, s3
                            Example: "sagemaker.api,sagemaker.runtime,bedrock,bedrock-runtime,bedrock-agent-runtime,s3"
                            - sagemaker.api: SageMaker control plane (CreateTrainingJob, CreateEndpoint, etc.)
                            - sagemaker.runtime: SageMaker data plane (InvokeEndpoint)
                            - bedrock: Bedrock control plane (CreateModelCustomizationJob, ListFoundationModels)
                            - bedrock-runtime: Bedrock data plane (InvokeModel, Converse)
                            - bedrock-agent-runtime: Bedrock agent runtime (InvokeAgent, InvokeFlow)
                            - s3: S3 gateway endpoint for data access

ENDPOINT_POLICIES:          [OPTIONAL: default restrictive policies per service]
                            JSON map of service name to custom endpoint policy document.
                            If not provided, generates default policies that restrict access
                            to the owning account and project-tagged resources.
                            Example:
                            {
                              "sagemaker.api": {"Version":"2012-10-17","Statement":[...]},
                              "s3": {"Version":"2012-10-17","Statement":[...]}
                            }

ALLOWED_BUCKETS:            [OPTIONAL: none — restricts S3 endpoint to specific buckets]
                            Comma-separated list of S3 bucket names that the S3 gateway
                            endpoint policy allows access to. If not provided, allows access
                            to all buckets in the owning account.
                            Example: "myai-training-data-prod,myai-model-artifacts-prod,myai-logs-prod"

ALLOWED_SECURITY_GROUPS:    [OPTIONAL: none — creates new SGs per endpoint]
                            Comma-separated list of existing security group IDs that are
                            allowed inbound access to VPC endpoints. If not provided,
                            creates new security groups per endpoint.
                            Example: "sg-0abc123,sg-0def456"

SUBNET_IDS:                 [REQUIRED - comma-separated list of private subnet IDs]
                            Subnets where interface endpoints will be placed.
                            Must be private subnets in the VPC. Use subnets across
                            multiple AZs for high availability.
                            Example: "subnet-0abc123,subnet-0def456"
                            Typically read from SSM: /mlops/{PROJECT_NAME}/{ENV}/private-subnet-ids

ALLOWED_CIDR_BLOCKS:        [OPTIONAL: VPC CIDR — restricts endpoint SG inbound to specific CIDRs]
                            Comma-separated CIDR blocks allowed to access endpoints.
                            Example: "10.0.1.0/24,10.0.2.0/24"

ENABLE_PRIVATE_DNS:         [OPTIONAL: true]
                            Enable private DNS for interface endpoints. When enabled,
                            AWS service DNS names resolve to the endpoint's private IPs
                            within the VPC. Requires VPC DNS resolution and DNS hostnames enabled.

ROUTE_TABLE_IDS:            [OPTIONAL - required only if s3 is in ML_ENDPOINTS]
                            Comma-separated route table IDs for S3 gateway endpoint association.
                            Example: "rtb-0abc123,rtb-0def456"
                            Typically read from SSM: /mlops/{PROJECT_NAME}/{ENV}/private-route-table-ids
```

---

## Task

Generate complete VPC endpoint infrastructure for ML services:

```
{PROJECT_NAME}-vpc-endpoints-ml/
├── endpoints/
│   ├── create_interface_endpoint.py      # Parameterized interface endpoint creation
│   ├── create_s3_gateway_endpoint.py     # S3 gateway endpoint with bucket-scoped policy
│   ├── create_sagemaker_endpoints.py     # SageMaker API + runtime endpoints
│   ├── create_bedrock_endpoints.py       # Bedrock + Bedrock-runtime + Bedrock-agent-runtime
│   └── endpoint_policies.py             # Endpoint policy document builders per service
├── security/
│   ├── create_security_groups.py         # Security groups per endpoint type
│   └── sg_rules.py                       # Inbound/outbound rule definitions
├── testing/
│   ├── test_endpoint_policies.py         # Policy testing utility (allowed/denied calls)
│   └── validate_connectivity.py          # Endpoint DNS resolution and connectivity checks
├── infrastructure/
│   ├── ssm_outputs.py                    # Write endpoint IDs and DNS names to SSM
│   └── config.py                         # Central configuration
├── run_setup.py                          # CLI orchestrator
└── requirements.txt
```

**config.py**: Central configuration dataclass with all parameters. Load from environment variables or CLI args. Validate required fields. Construct resource names using `{PROJECT_NAME}-{component}-{ENV}` convention. Parse ML_ENDPOINTS into a list. Parse ALLOWED_BUCKETS, SUBNET_IDS, ALLOWED_SECURITY_GROUPS, ALLOWED_CIDR_BLOCKS, and ROUTE_TABLE_IDS into lists. Resolve VPC_ID and SUBNET_IDS from SSM if not provided directly.

**create_interface_endpoint.py**: Parameterized interface endpoint creation pattern:
- `create_interface_endpoint(service_name, policy_document, subnet_ids, security_group_ids, enable_private_dns)`: generic function that creates any VPC interface endpoint
- Call `ec2.create_vpc_endpoint()` with:
  - `VpcEndpointType`: `"Interface"`
  - `VpcId`: VPC_ID
  - `ServiceName`: `com.amazonaws.{region}.{service_name}`
  - `SubnetIds`: SUBNET_IDS list
  - `SecurityGroupIds`: security group IDs for this endpoint
  - `PrivateDnsEnabled`: ENABLE_PRIVATE_DNS
  - `PolicyDocument`: JSON string of the endpoint policy
  - `TagSpecifications`: `Name={PROJECT_NAME}-vpce-{service_name}-{ENV}`, `Project={PROJECT_NAME}`, `Environment={ENV}`
- Return endpoint ID and DNS entries
- This function is reused by `create_sagemaker_endpoints.py` and `create_bedrock_endpoints.py`

**create_s3_gateway_endpoint.py**: S3 gateway endpoint with bucket-scoped policy:
- Call `ec2.create_vpc_endpoint()` with:
  - `VpcEndpointType`: `"Gateway"`
  - `VpcId`: VPC_ID
  - `ServiceName`: `com.amazonaws.{region}.s3`
  - `RouteTableIds`: ROUTE_TABLE_IDS list
  - `PolicyDocument`: S3 endpoint policy from `endpoint_policies.py` scoped to ALLOWED_BUCKETS
  - `TagSpecifications`: `Name={PROJECT_NAME}-vpce-s3-{ENV}`, `Project={PROJECT_NAME}`, `Environment={ENV}`
- S3 gateway endpoints do not use subnets or security groups — they use route table associations
- Return endpoint ID

**create_sagemaker_endpoints.py**: Create SageMaker VPC endpoints:
- If `sagemaker.api` in ML_ENDPOINTS: create interface endpoint for `com.amazonaws.{region}.sagemaker.api` with SageMaker API endpoint policy (restricts to project-tagged resources)
- If `sagemaker.runtime` in ML_ENDPOINTS: create interface endpoint for `com.amazonaws.{region}.sagemaker.runtime` with SageMaker runtime endpoint policy (restricts InvokeEndpoint to project-tagged endpoints)
- Uses `create_interface_endpoint()` from `create_interface_endpoint.py`
- Passes SageMaker-specific security group from `create_security_groups.py`

**create_bedrock_endpoints.py**: Create Bedrock VPC endpoints:
- If `bedrock` in ML_ENDPOINTS: create interface endpoint for `com.amazonaws.{region}.bedrock` with Bedrock control plane policy (restricts to owning account)
- If `bedrock-runtime` in ML_ENDPOINTS: create interface endpoint for `com.amazonaws.{region}.bedrock-runtime` with Bedrock runtime policy (restricts InvokeModel/Converse to owning account)
- If `bedrock-agent-runtime` in ML_ENDPOINTS: create interface endpoint for `com.amazonaws.{region}.bedrock-agent-runtime` with Bedrock agent runtime policy (restricts InvokeAgent/InvokeFlow to owning account)
- Uses `create_interface_endpoint()` from `create_interface_endpoint.py`
- Passes Bedrock-specific security group from `create_security_groups.py`

**endpoint_policies.py**: Endpoint policy document builders per service:
- `build_sagemaker_api_policy(account_id, project_name)`: allow SageMaker API actions only for resources tagged with `Project={PROJECT_NAME}` using `sagemaker:ResourceTag/Project` condition, restrict to `aws:PrincipalAccount` matching the owning account
- `build_sagemaker_runtime_policy(account_id, project_name)`: allow `sagemaker:InvokeEndpoint` and `sagemaker:InvokeEndpointWithResponseStream` only for endpoints tagged with `Project={PROJECT_NAME}`
- `build_bedrock_policy(account_id)`: allow Bedrock control plane actions restricted to `aws:ResourceAccount` matching the owning account
- `build_bedrock_runtime_policy(account_id)`: allow `bedrock:InvokeModel`, `bedrock:InvokeModelWithResponseStream`, `bedrock:Converse`, `bedrock:ConverseStream` restricted to `aws:ResourceAccount`
- `build_bedrock_agent_runtime_policy(account_id)`: allow `bedrock:InvokeAgent`, `bedrock:InvokeFlow` restricted to `aws:ResourceAccount`
- `build_s3_gateway_policy(account_id, allowed_buckets)`: allow S3 actions only on specified bucket ARNs and their objects; if ALLOWED_BUCKETS is empty, allow all buckets in the owning account using `s3:ResourceAccount` condition
- Each policy includes an explicit `Deny` for actions outside the allowed scope where appropriate

**create_security_groups.py**: Create security groups per endpoint type:
- `create_endpoint_security_group(endpoint_type)`: create SG with:
  - Name: `{PROJECT_NAME}-vpce-{endpoint_type}-sg-{ENV}`
  - Description: `Security group for {endpoint_type} VPC endpoint`
  - VpcId: VPC_ID
  - Tags: `Project={PROJECT_NAME}`, `Environment={ENV}`, `EndpointType={endpoint_type}`
- Create separate security groups for: `sagemaker`, `bedrock`, or a shared group if ALLOWED_SECURITY_GROUPS is provided
- Apply inbound rules from `sg_rules.py`

**sg_rules.py**: Security group rule definitions:
- All interface endpoints require HTTPS (port 443) inbound
- If ALLOWED_CIDR_BLOCKS provided: create inbound rules allowing TCP 443 from each CIDR block
- If ALLOWED_SECURITY_GROUPS provided: create inbound rules allowing TCP 443 from each referenced security group
- If neither provided: create inbound rule allowing TCP 443 from the VPC CIDR (looked up via `ec2.describe_vpcs()`)
- Outbound: allow all traffic (default) — endpoints initiate no outbound connections
- `apply_inbound_rules(security_group_id, allowed_cidrs, allowed_sgs)`: call `ec2.authorize_security_group_ingress()` with the appropriate rules

**test_endpoint_policies.py**: Policy testing utility:
- `test_sagemaker_endpoint_policy()`: attempt `sagemaker.list_training_jobs()` through the endpoint (should succeed), attempt to access resources in another account (should fail based on policy)
- `test_s3_endpoint_policy()`: attempt `s3.list_objects_v2()` on an allowed bucket (should succeed), attempt on a non-allowed bucket (should fail)
- `test_bedrock_endpoint_policy()`: attempt `bedrock_runtime.invoke_model()` (should succeed for owning account models)
- Report results as pass/fail per test case with error details
- Uses `try/except` to catch `AccessDeniedException` and `ClientError` for denied calls

**validate_connectivity.py**: Endpoint DNS resolution and connectivity checks:
- Resolve endpoint DNS names using `socket.getaddrinfo()` to verify private DNS is working
- Verify endpoint state is `available` using `ec2.describe_vpc_endpoints()`
- Check security group rules are correctly applied using `ec2.describe_security_groups()`
- Report connectivity status per endpoint

**ssm_outputs.py**: Write endpoint IDs and DNS names to SSM Parameter Store:
- Path: `/mlops/{PROJECT_NAME}/{ENV}/vpce-{service_name}-id` for each endpoint
- Path: `/mlops/{PROJECT_NAME}/{ENV}/vpce-{service_name}-dns` for each endpoint's DNS name
- Other templates read from these SSM paths for VPC endpoint configuration

**run_setup.py**: CLI orchestrator that runs setup steps in order:
1. Load and validate configuration
2. Create security groups for endpoint types
3. Apply security group inbound rules
4. Create S3 gateway endpoint (if `s3` in ML_ENDPOINTS)
5. Create SageMaker interface endpoints (if `sagemaker.*` in ML_ENDPOINTS)
6. Create Bedrock interface endpoints (if `bedrock*` in ML_ENDPOINTS)
7. Write endpoint IDs and DNS names to SSM
8. Run connectivity validation
9. Print summary with endpoint IDs, DNS names, security group IDs, and policy summaries

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**Endpoint Separation:** Create separate VPC endpoints per ML service (SageMaker API, SageMaker runtime, Bedrock, Bedrock-runtime, Bedrock-agent-runtime, S3). Separation enables per-service endpoint policies — the SageMaker runtime endpoint can have a different policy than the SageMaker API endpoint. Interface endpoints cost $0.01/hour per AZ plus data processing charges; consolidate to fewer AZs in dev to reduce cost.

**Endpoint Policies:** VPC endpoint policies are IAM-style policy documents that act as an additional access control layer. They intersect with IAM policies — both the endpoint policy AND the IAM policy must allow the action. Use endpoint policies to enforce: (1) only the owning account can use the endpoint (`aws:PrincipalAccount`), (2) only project-tagged resources can be accessed (`sagemaker:ResourceTag/Project`), (3) only specific S3 buckets are reachable (`s3:prefix` and bucket ARN conditions). Endpoint policies cannot grant permissions — they can only restrict what IAM already allows.

**S3 Gateway Endpoint:** S3 uses a gateway endpoint (free, no hourly charge) instead of an interface endpoint. Gateway endpoints are associated with route tables, not subnets. The S3 endpoint policy should restrict access to only the ML data and artifact buckets listed in ALLOWED_BUCKETS. Include `s3:GetObject`, `s3:PutObject`, `s3:ListBucket`, `s3:DeleteObject`, and `s3:GetBucketLocation` actions. Always allow access to S3 buckets required by SageMaker and Bedrock services (e.g., SageMaker-managed buckets for model artifacts).

**Security Groups:** Each interface endpoint gets a security group allowing inbound HTTPS (TCP 443) only from ML workload subnets or security groups. Do not use `0.0.0.0/0` for endpoint security groups — restrict to the VPC CIDR at minimum, and to specific ML workload CIDRs or security groups when possible. Outbound rules can be permissive (endpoints do not initiate connections). Tag security groups with `Project`, `Environment`, and `EndpointType`.

**Private DNS:** Enable private DNS on interface endpoints so that standard AWS SDK calls (e.g., `boto3.client("sagemaker")`) automatically route through the VPC endpoint without code changes. Private DNS requires the VPC to have `enableDnsSupport` and `enableDnsHostnames` set to `true`. Verify DNS resolution in the testing utility.

**Policy Conditions:** Use these condition keys in endpoint policies:
- `aws:PrincipalAccount`: restrict to principals in the owning account
- `aws:ResourceAccount`: restrict to resources in the owning account
- `sagemaker:ResourceTag/Project`: restrict SageMaker API calls to resources tagged with the project name
- `aws:PrincipalOrgID`: restrict to principals in the AWS Organization (for multi-account setups)
- S3 bucket ARN conditions: restrict S3 access to specific bucket ARNs and prefixes

**Cost:** Interface endpoints cost $0.01/hour per AZ (~$7.20/month per AZ). In dev, deploy to 1-2 AZs; in prod, deploy to all AZs for HA. S3 gateway endpoints are free. Data processed through interface endpoints costs $0.01/GB. Monitor `AWS/PrivateLinkEndpoints` CloudWatch metrics for `BytesProcessed` and `ActiveConnections`.

**Naming:** All resources follow `{PROJECT_NAME}-{component}-{ENV}` convention:
- VPC endpoint: `{PROJECT_NAME}-vpce-{service_name}-{ENV}`
- Security group: `{PROJECT_NAME}-vpce-{service_type}-sg-{ENV}`
- SSM parameter: `/mlops/{PROJECT_NAME}/{ENV}/vpce-{service_name}-id`
- SSM parameter: `/mlops/{PROJECT_NAME}/{ENV}/vpce-{service_name}-dns`

---

## Code Scaffolding Hints

**Parameterized interface endpoint creation:**
```python
import boto3
import json

ec2 = boto3.client("ec2", region_name=AWS_REGION)

def create_interface_endpoint(service_name, policy_document, subnet_ids, security_group_ids, enable_private_dns=True):
    """Create a VPC interface endpoint for any AWS service."""
    response = ec2.create_vpc_endpoint(
        VpcEndpointType="Interface",
        VpcId=VPC_ID,
        ServiceName=f"com.amazonaws.{AWS_REGION}.{service_name}",
        SubnetIds=subnet_ids,
        SecurityGroupIds=security_group_ids,
        PrivateDnsEnabled=enable_private_dns,
        PolicyDocument=json.dumps(policy_document),
        TagSpecifications=[
            {
                "ResourceType": "vpc-endpoint",
                "Tags": [
                    {"Key": "Name", "Value": f"{PROJECT_NAME}-vpce-{service_name}-{ENV}"},
                    {"Key": "Project", "Value": PROJECT_NAME},
                    {"Key": "Environment", "Value": ENV},
                    {"Key": "Service", "Value": service_name},
                ],
            }
        ],
    )
    endpoint_id = response["VpcEndpoint"]["VpcEndpointId"]
    dns_entries = response["VpcEndpoint"].get("DnsEntries", [])
    print(f"VPC endpoint created: {endpoint_id} for {service_name}")
    return endpoint_id, dns_entries
```

**Create SageMaker API interface endpoint:**
```python
sagemaker_api_policy = build_sagemaker_api_policy(AWS_ACCOUNT_ID, PROJECT_NAME)
sagemaker_api_endpoint_id, sagemaker_api_dns = create_interface_endpoint(
    service_name="sagemaker.api",
    policy_document=sagemaker_api_policy,
    subnet_ids=SUBNET_IDS,
    security_group_ids=[sagemaker_sg_id],
    enable_private_dns=ENABLE_PRIVATE_DNS,
)
```

**Create SageMaker runtime interface endpoint:**
```python
sagemaker_runtime_policy = build_sagemaker_runtime_policy(AWS_ACCOUNT_ID, PROJECT_NAME)
sagemaker_runtime_endpoint_id, sagemaker_runtime_dns = create_interface_endpoint(
    service_name="sagemaker.runtime",
    policy_document=sagemaker_runtime_policy,
    subnet_ids=SUBNET_IDS,
    security_group_ids=[sagemaker_sg_id],
    enable_private_dns=ENABLE_PRIVATE_DNS,
)
```

**Create Bedrock interface endpoints:**
```python
# Bedrock control plane
bedrock_policy = build_bedrock_policy(AWS_ACCOUNT_ID)
bedrock_endpoint_id, bedrock_dns = create_interface_endpoint(
    service_name="bedrock",
    policy_document=bedrock_policy,
    subnet_ids=SUBNET_IDS,
    security_group_ids=[bedrock_sg_id],
    enable_private_dns=ENABLE_PRIVATE_DNS,
)

# Bedrock runtime (data plane)
bedrock_runtime_policy = build_bedrock_runtime_policy(AWS_ACCOUNT_ID)
bedrock_runtime_endpoint_id, bedrock_runtime_dns = create_interface_endpoint(
    service_name="bedrock-runtime",
    policy_document=bedrock_runtime_policy,
    subnet_ids=SUBNET_IDS,
    security_group_ids=[bedrock_sg_id],
    enable_private_dns=ENABLE_PRIVATE_DNS,
)

# Bedrock agent runtime
bedrock_agent_runtime_policy = build_bedrock_agent_runtime_policy(AWS_ACCOUNT_ID)
bedrock_agent_runtime_endpoint_id, bedrock_agent_runtime_dns = create_interface_endpoint(
    service_name="bedrock-agent-runtime",
    policy_document=bedrock_agent_runtime_policy,
    subnet_ids=SUBNET_IDS,
    security_group_ids=[bedrock_sg_id],
    enable_private_dns=ENABLE_PRIVATE_DNS,
)
```

**Create S3 gateway endpoint with bucket-scoped policy:**
```python
s3_policy = build_s3_gateway_policy(AWS_ACCOUNT_ID, ALLOWED_BUCKETS)

s3_response = ec2.create_vpc_endpoint(
    VpcEndpointType="Gateway",
    VpcId=VPC_ID,
    ServiceName=f"com.amazonaws.{AWS_REGION}.s3",
    RouteTableIds=ROUTE_TABLE_IDS,
    PolicyDocument=json.dumps(s3_policy),
    TagSpecifications=[
        {
            "ResourceType": "vpc-endpoint",
            "Tags": [
                {"Key": "Name", "Value": f"{PROJECT_NAME}-vpce-s3-{ENV}"},
                {"Key": "Project", "Value": PROJECT_NAME},
                {"Key": "Environment", "Value": ENV},
            ],
        }
    ],
)
s3_endpoint_id = s3_response["VpcEndpoint"]["VpcEndpointId"]
print(f"S3 gateway endpoint created: {s3_endpoint_id}")
```

**SageMaker API endpoint policy with resource tag conditions:**
```python
def build_sagemaker_api_policy(account_id, project_name):
    """Build endpoint policy for SageMaker API restricting to project-tagged resources."""
    return {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "AllowSageMakerAPIForProject",
                "Effect": "Allow",
                "Principal": {"AWS": "*"},
                "Action": [
                    "sagemaker:CreateTrainingJob",
                    "sagemaker:CreateEndpoint",
                    "sagemaker:CreateEndpointConfig",
                    "sagemaker:CreateModel",
                    "sagemaker:CreateProcessingJob",
                    "sagemaker:CreateTransformJob",
                    "sagemaker:DescribeTrainingJob",
                    "sagemaker:DescribeEndpoint",
                    "sagemaker:DescribeEndpointConfig",
                    "sagemaker:DescribeModel",
                    "sagemaker:ListTrainingJobs",
                    "sagemaker:ListEndpoints",
                    "sagemaker:ListModels",
                    "sagemaker:UpdateEndpoint",
                    "sagemaker:DeleteEndpoint",
                    "sagemaker:DeleteModel",
                    "sagemaker:StopTrainingJob",
                    "sagemaker:AddTags",
                    "sagemaker:ListTags",
                ],
                "Resource": f"arn:aws:sagemaker:{AWS_REGION}:{account_id}:*",
                "Condition": {
                    "StringEquals": {
                        "aws:PrincipalAccount": account_id,
                    }
                },
            },
            {
                "Sid": "RestrictToProjectTaggedResources",
                "Effect": "Deny",
                "Principal": {"AWS": "*"},
                "Action": [
                    "sagemaker:CreateTrainingJob",
                    "sagemaker:CreateEndpoint",
                    "sagemaker:CreateProcessingJob",
                    "sagemaker:CreateTransformJob",
                ],
                "Resource": f"arn:aws:sagemaker:{AWS_REGION}:{account_id}:*",
                "Condition": {
                    "StringNotEquals": {
                        "sagemaker:ResourceTag/Project": project_name,
                    }
                },
            },
        ],
    }
```

**SageMaker runtime endpoint policy:**
```python
def build_sagemaker_runtime_policy(account_id, project_name):
    """Build endpoint policy for SageMaker runtime restricting InvokeEndpoint to project endpoints."""
    return {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "AllowInvokeProjectEndpoints",
                "Effect": "Allow",
                "Principal": {"AWS": "*"},
                "Action": [
                    "sagemaker:InvokeEndpoint",
                    "sagemaker:InvokeEndpointWithResponseStream",
                    "sagemaker:InvokeEndpointAsync",
                ],
                "Resource": f"arn:aws:sagemaker:{AWS_REGION}:{account_id}:endpoint/*",
                "Condition": {
                    "StringEquals": {
                        "aws:PrincipalAccount": account_id,
                        "sagemaker:ResourceTag/Project": project_name,
                    }
                },
            },
        ],
    }
```

**Bedrock control plane endpoint policy:**
```python
def build_bedrock_policy(account_id):
    """Build endpoint policy for Bedrock control plane restricting to owning account."""
    return {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "AllowBedrockControlPlane",
                "Effect": "Allow",
                "Principal": {"AWS": "*"},
                "Action": [
                    "bedrock:ListFoundationModels",
                    "bedrock:GetFoundationModel",
                    "bedrock:CreateModelCustomizationJob",
                    "bedrock:GetModelCustomizationJob",
                    "bedrock:ListModelCustomizationJobs",
                    "bedrock:CreateProvisionedModelThroughput",
                    "bedrock:GetProvisionedModelThroughput",
                    "bedrock:ListProvisionedModelThroughputs",
                    "bedrock:CreateGuardrail",
                    "bedrock:GetGuardrail",
                    "bedrock:ListGuardrails",
                    "bedrock:CreateEvaluationJob",
                    "bedrock:GetEvaluationJob",
                    "bedrock:PutModelInvocationLoggingConfiguration",
                    "bedrock:GetModelInvocationLoggingConfiguration",
                ],
                "Resource": "*",
                "Condition": {
                    "StringEquals": {
                        "aws:PrincipalAccount": account_id,
                        "aws:ResourceAccount": account_id,
                    }
                },
            },
        ],
    }
```

**Bedrock runtime endpoint policy:**
```python
def build_bedrock_runtime_policy(account_id):
    """Build endpoint policy for Bedrock runtime restricting to owning account."""
    return {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "AllowBedrockInference",
                "Effect": "Allow",
                "Principal": {"AWS": "*"},
                "Action": [
                    "bedrock:InvokeModel",
                    "bedrock:InvokeModelWithResponseStream",
                    "bedrock:Converse",
                    "bedrock:ConverseStream",
                ],
                "Resource": [
                    f"arn:aws:bedrock:{AWS_REGION}::foundation-model/*",
                    f"arn:aws:bedrock:{AWS_REGION}:{account_id}:provisioned-model/*",
                    f"arn:aws:bedrock:{AWS_REGION}:{account_id}:inference-profile/*",
                ],
                "Condition": {
                    "StringEquals": {
                        "aws:PrincipalAccount": account_id,
                    }
                },
            },
        ],
    }
```

**Bedrock agent runtime endpoint policy:**
```python
def build_bedrock_agent_runtime_policy(account_id):
    """Build endpoint policy for Bedrock agent runtime restricting to owning account."""
    return {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "AllowBedrockAgentRuntime",
                "Effect": "Allow",
                "Principal": {"AWS": "*"},
                "Action": [
                    "bedrock:InvokeAgent",
                    "bedrock:InvokeFlow",
                    "bedrock:Retrieve",
                    "bedrock:RetrieveAndGenerate",
                ],
                "Resource": [
                    f"arn:aws:bedrock:{AWS_REGION}:{account_id}:agent/*",
                    f"arn:aws:bedrock:{AWS_REGION}:{account_id}:agent-alias/*",
                    f"arn:aws:bedrock:{AWS_REGION}:{account_id}:flow/*",
                    f"arn:aws:bedrock:{AWS_REGION}:{account_id}:flow-alias/*",
                    f"arn:aws:bedrock:{AWS_REGION}:{account_id}:knowledge-base/*",
                ],
                "Condition": {
                    "StringEquals": {
                        "aws:PrincipalAccount": account_id,
                    }
                },
            },
        ],
    }
```

**S3 gateway endpoint policy with bucket-scoped access:**
```python
def build_s3_gateway_policy(account_id, allowed_buckets=None):
    """Build S3 gateway endpoint policy restricting to specific ML buckets."""
    if allowed_buckets:
        # Restrict to specific buckets
        bucket_arns = [f"arn:aws:s3:::{b}" for b in allowed_buckets]
        object_arns = [f"arn:aws:s3:::{b}/*" for b in allowed_buckets]
        return {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Sid": "AllowMLBucketAccess",
                    "Effect": "Allow",
                    "Principal": {"AWS": "*"},
                    "Action": [
                        "s3:GetObject",
                        "s3:PutObject",
                        "s3:DeleteObject",
                        "s3:ListBucket",
                        "s3:GetBucketLocation",
                        "s3:ListBucketMultipartUploads",
                        "s3:AbortMultipartUpload",
                        "s3:ListMultipartUploadParts",
                    ],
                    "Resource": bucket_arns + object_arns,
                    "Condition": {
                        "StringEquals": {
                            "aws:PrincipalAccount": account_id,
                        }
                    },
                },
                {
                    "Sid": "AllowSageMakerManagedBuckets",
                    "Effect": "Allow",
                    "Principal": {"AWS": "*"},
                    "Action": [
                        "s3:GetObject",
                        "s3:PutObject",
                        "s3:ListBucket",
                        "s3:GetBucketLocation",
                    ],
                    "Resource": [
                        "arn:aws:s3:::sagemaker-*",
                        "arn:aws:s3:::sagemaker-*/*",
                        "arn:aws:s3:::jumpstart-cache-prod-*",
                        "arn:aws:s3:::jumpstart-cache-prod-*/*",
                    ],
                    "Condition": {
                        "StringEquals": {
                            "aws:PrincipalAccount": account_id,
                        }
                    },
                },
            ],
        }
    else:
        # Allow all buckets in the owning account
        return {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Sid": "AllowAccountBucketAccess",
                    "Effect": "Allow",
                    "Principal": {"AWS": "*"},
                    "Action": "s3:*",
                    "Resource": "*",
                    "Condition": {
                        "StringEquals": {
                            "s3:ResourceAccount": account_id,
                        }
                    },
                },
                {
                    "Sid": "AllowSageMakerManagedBuckets",
                    "Effect": "Allow",
                    "Principal": {"AWS": "*"},
                    "Action": [
                        "s3:GetObject",
                        "s3:ListBucket",
                        "s3:GetBucketLocation",
                    ],
                    "Resource": [
                        "arn:aws:s3:::sagemaker-*",
                        "arn:aws:s3:::sagemaker-*/*",
                        "arn:aws:s3:::jumpstart-cache-prod-*",
                        "arn:aws:s3:::jumpstart-cache-prod-*/*",
                    ],
                },
            ],
        }
```

**Security group creation for VPC endpoints:**
```python
def create_endpoint_security_group(endpoint_type, vpc_id, allowed_cidrs=None, allowed_sgs=None):
    """Create a security group for VPC endpoints with HTTPS inbound rules."""
    # Create security group
    sg_response = ec2.create_security_group(
        GroupName=f"{PROJECT_NAME}-vpce-{endpoint_type}-sg-{ENV}",
        Description=f"Security group for {endpoint_type} VPC endpoints - {PROJECT_NAME} {ENV}",
        VpcId=vpc_id,
        TagSpecifications=[
            {
                "ResourceType": "security-group",
                "Tags": [
                    {"Key": "Name", "Value": f"{PROJECT_NAME}-vpce-{endpoint_type}-sg-{ENV}"},
                    {"Key": "Project", "Value": PROJECT_NAME},
                    {"Key": "Environment", "Value": ENV},
                    {"Key": "EndpointType", "Value": endpoint_type},
                ],
            }
        ],
    )
    sg_id = sg_response["GroupId"]

    # Build inbound rules for HTTPS (443)
    ip_permissions = []

    if allowed_cidrs:
        for cidr in allowed_cidrs:
            ip_permissions.append({
                "IpProtocol": "tcp",
                "FromPort": 443,
                "ToPort": 443,
                "IpRanges": [{"CidrIp": cidr, "Description": f"HTTPS from {cidr}"}],
            })
    elif allowed_sgs:
        for ref_sg in allowed_sgs:
            ip_permissions.append({
                "IpProtocol": "tcp",
                "FromPort": 443,
                "ToPort": 443,
                "UserIdGroupPairs": [{"GroupId": ref_sg, "Description": f"HTTPS from {ref_sg}"}],
            })
    else:
        # Default: allow from VPC CIDR
        vpc_info = ec2.describe_vpcs(VpcIds=[vpc_id])
        vpc_cidr = vpc_info["Vpcs"][0]["CidrBlock"]
        ip_permissions.append({
            "IpProtocol": "tcp",
            "FromPort": 443,
            "ToPort": 443,
            "IpRanges": [{"CidrIp": vpc_cidr, "Description": "HTTPS from VPC CIDR"}],
        })

    # Apply inbound rules
    ec2.authorize_security_group_ingress(
        GroupId=sg_id,
        IpPermissions=ip_permissions,
    )
    print(f"Security group created: {sg_id} for {endpoint_type} endpoints")
    return sg_id
```

**Policy testing utility:**
```python
import boto3
from botocore.exceptions import ClientError

def test_endpoint_policies():
    """Test VPC endpoint policies by attempting allowed and denied API calls."""
    results = []

    # Test SageMaker API endpoint
    sagemaker = boto3.client("sagemaker", region_name=AWS_REGION)
    try:
        sagemaker.list_training_jobs(MaxResults=1)
        results.append({"test": "SageMaker ListTrainingJobs", "result": "PASS", "detail": "Allowed"})
    except ClientError as e:
        code = e.response["Error"]["Code"]
        results.append({"test": "SageMaker ListTrainingJobs", "result": "FAIL", "detail": code})

    # Test SageMaker runtime endpoint
    sagemaker_runtime = boto3.client("sagemaker-runtime", region_name=AWS_REGION)
    # Note: InvokeEndpoint requires an actual endpoint — skip if none exist

    # Test Bedrock runtime endpoint
    bedrock_runtime = boto3.client("bedrock-runtime", region_name=AWS_REGION)
    try:
        bedrock_runtime.converse(
            modelId="amazon.titan-text-lite-v1",
            messages=[{"role": "user", "content": [{"text": "test"}]}],
            inferenceConfig={"maxTokens": 1},
        )
        results.append({"test": "Bedrock Converse", "result": "PASS", "detail": "Allowed"})
    except ClientError as e:
        code = e.response["Error"]["Code"]
        if code == "AccessDeniedException":
            results.append({"test": "Bedrock Converse", "result": "POLICY_DENIED", "detail": code})
        else:
            results.append({"test": "Bedrock Converse", "result": "PASS", "detail": f"API reachable ({code})"})

    # Test S3 endpoint — allowed bucket
    s3 = boto3.client("s3", region_name=AWS_REGION)
    if ALLOWED_BUCKETS:
        try:
            s3.list_objects_v2(Bucket=ALLOWED_BUCKETS[0], MaxKeys=1)
            results.append({"test": f"S3 ListObjects ({ALLOWED_BUCKETS[0]})", "result": "PASS", "detail": "Allowed"})
        except ClientError as e:
            code = e.response["Error"]["Code"]
            results.append({"test": f"S3 ListObjects ({ALLOWED_BUCKETS[0]})", "result": "FAIL", "detail": code})

    # Report results
    print("\n=== VPC Endpoint Policy Test Results ===")
    for r in results:
        status = "✅" if r["result"] == "PASS" else "❌"
        print(f"  {status} {r['test']}: {r['result']} — {r['detail']}")

    return results
```

**Write endpoint outputs to SSM:**
```python
ssm = boto3.client("ssm", region_name=AWS_REGION)

def write_endpoint_to_ssm(service_name, endpoint_id, dns_entries):
    """Write VPC endpoint ID and DNS name to SSM for cross-template consumption."""
    ssm.put_parameter(
        Name=f"/mlops/{PROJECT_NAME}/{ENV}/vpce-{service_name}-id",
        Description=f"VPC endpoint ID for {service_name}",
        Value=endpoint_id,
        Type="String",
        Overwrite=True,
        Tags=[
            {"Key": "Project", "Value": PROJECT_NAME},
            {"Key": "Environment", "Value": ENV},
        ],
    )
    if dns_entries:
        primary_dns = dns_entries[0].get("DnsName", "")
        ssm.put_parameter(
            Name=f"/mlops/{PROJECT_NAME}/{ENV}/vpce-{service_name}-dns",
            Description=f"VPC endpoint DNS name for {service_name}",
            Value=primary_dns,
            Type="String",
            Overwrite=True,
        )
    print(f"SSM: /mlops/{PROJECT_NAME}/{ENV}/vpce-{service_name}-id = {endpoint_id}")
```

---

## Integration Points

- **Upstream**: `devops/02` → VPC ID, private subnet IDs, and route table IDs created by the VPC networking template; this template creates endpoints within that VPC
- **Upstream**: `devops/04` → IAM roles (SageMaker execution role, Bedrock role) whose API calls traverse these VPC endpoints; endpoint policies must allow actions performed by these roles
- **Downstream**: `devops/05` → Config compliance rules that verify VPC endpoints exist for required ML services and that endpoint policies are not overly permissive
- **Downstream**: `enterprise/04` → Control Tower account baselines that deploy SageMaker and Bedrock VPC endpoints in every new ML workload account using this template's patterns
- **Downstream**: `mlops/01` → SageMaker training jobs running in VPC mode use the SageMaker API and runtime endpoints for API calls
- **Downstream**: `mlops/03` → SageMaker inference endpoints in VPC use the SageMaker runtime endpoint for InvokeEndpoint calls
- **Downstream**: `mlops/14` → Bedrock Agents running in VPC use the Bedrock-agent-runtime endpoint for InvokeAgent calls
- **Downstream**: `devops/08` → KMS VPC endpoint may be needed alongside ML endpoints for encryption operations; coordinate with KMS key management
