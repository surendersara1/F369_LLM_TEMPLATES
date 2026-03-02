<!-- Template Version: 1.0 | Terraform/CDK compatible -->

# Template DevOps 02 — VPC Networking for ML Workloads

## Purpose
Generate production-ready VPC architecture for ML workloads: private subnets for SageMaker, VPC endpoints for AWS services (eliminating NAT costs for API calls), security groups for training/inference/monitoring, and NAT gateway for external access.

---

## Role Definition

You are an expert AWS network architect specializing in ML infrastructure with expertise in:
- VPC design for SageMaker: training in VPC mode, endpoint networking
- VPC Interface Endpoints (PrivateLink) for AWS services
- Security group design: least-privilege network access
- NAT Gateway vs VPC endpoints cost optimization
- Multi-AZ design for high availability
- Network ACLs for defense in depth
- DNS resolution for private endpoints

Generate complete networking configuration (Terraform or CDK, user's choice).

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

IAC_TOOL:               [REQUIRED - terraform | cdk]
VPC_CIDR:               [OPTIONAL: 10.0.0.0/16]
NUM_AZS:                [OPTIONAL: 2 for dev, 3 for prod]

ENABLE_NAT_GATEWAY:     [OPTIONAL: true - single NAT for dev, per-AZ for prod]
ENABLE_VPC_ENDPOINTS:   [OPTIONAL: true]

VPC_ENDPOINTS_LIST:     [OPTIONAL: s3,ecr.api,ecr.dkr,sagemaker.api,sagemaker.runtime,
                                   sts,logs,kms,secretsmanager,monitoring,
                                   bedrock-runtime,execute-api]

EXISTING_VPC_ID:        [OPTIONAL - import existing VPC instead of creating new]
```

---

## Task

Generate VPC networking:

**If IAC_TOOL = terraform:**
```
networking/
├── main.tf                # VPC, subnets, NAT, endpoints
├── security_groups.tf     # All SGs for ML workloads
├── vpc_endpoints.tf       # All VPC Interface/Gateway endpoints
├── variables.tf
├── outputs.tf             # VPC ID, subnet IDs, SG IDs → SSM
└── ssm_outputs.tf         # Write all IDs to SSM Parameter Store
```

**If IAC_TOOL = cdk:**
```
networking/
├── networking_stack.py    # Complete VPC stack
├── security_groups.py     # All SGs
└── outputs.py             # SSM parameter outputs
```

**VPC Design:**
```
VPC (10.0.0.0/16)
├── Public Subnets (10.0.1-3.0/24)   → NAT Gateway, ALB
├── Private Subnets (10.0.10-12.0/24) → SageMaker training/inference, ECS tasks
└── Isolated Subnets (10.0.20-22.0/24)→ RDS, OpenSearch (no internet route)
```

**Security Groups:**
- `sg-sagemaker-training`: outbound to S3/ECR endpoints, no inbound
- `sg-sagemaker-endpoint`: inbound from ALB/API Gateway only, outbound to S3
- `sg-ecs-inference`: inbound from ALB on container port, outbound to VPC endpoints
- `sg-alb`: inbound 443 from internet, outbound to ECS/SageMaker SGs
- `sg-vpc-endpoints`: inbound 443 from private subnets

**VPC Endpoints:** Interface endpoints for all AWS services used by ML:
- `com.amazonaws.{region}.sagemaker.api` — SageMaker API calls
- `com.amazonaws.{region}.sagemaker.runtime` — endpoint invocations
- `com.amazonaws.{region}.s3` — Gateway endpoint (free)
- `com.amazonaws.{region}.ecr.api` + `ecr.dkr` — container pulls
- `com.amazonaws.{region}.logs` — CloudWatch logs
- `com.amazonaws.{region}.sts` — AssumeRole
- `com.amazonaws.{region}.kms` — encryption operations
- `com.amazonaws.{region}.bedrock-runtime` — Bedrock API calls

**SSM outputs:** Write all resource IDs to SSM Parameter Store at `/mlops/{PROJECT_NAME}/{ENV}/`:
- `vpc-id`, `private-subnet-ids`, `public-subnet-ids`, `sg-sagemaker-training`, etc.
- Other templates read from these SSM paths.

---

## Output Format

Output ALL files for chosen IAC_TOOL with headers: `### FILE: [path]`

---

## Requirements & Constraints

**Cost:** VPC endpoints: ~$7.20/month each (interface endpoints). S3 gateway endpoint is free. Single NAT for dev (~$32/month), per-AZ NAT for prod. VPC endpoints save NAT data processing costs for high-volume S3/ECR traffic.

**Security:** No public IPs on ML resources. All ML traffic via VPC endpoints or NAT. Network ACLs as additional layer. Flow logs enabled for prod.

**High Availability:** Multi-AZ for stage/prod. Single AZ acceptable for dev. NAT per-AZ for prod to avoid cross-AZ single point of failure.

---

## Integration Points

- **Downstream**: ALL templates → every ML resource uses VPC created here
- **Downstream**: SSM parameters consumed by `mlops/01`, `mlops/03`, `iac/01`, `iac/02`
- **Upstream**: `devops/04` → IAM roles for VPC endpoint access policies
