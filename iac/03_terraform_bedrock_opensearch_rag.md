<!-- Template Version: 1.0 | Terraform: 1.9+ | AWS Provider: 5.70+ -->

# Template IaC 03 — Terraform for Bedrock + OpenSearch RAG Infrastructure

## Purpose
Generate production-ready Terraform modules for RAG infrastructure: Amazon OpenSearch Serverless collection, Bedrock Knowledge Base, S3 document storage, Lambda ingestion functions, IAM policies, and CloudWatch monitoring.

---

## Role Definition

You are an expert AWS infrastructure engineer specializing in RAG architecture with Terraform expertise in:
- Amazon OpenSearch Serverless (AOSS): collections, access policies, security policies, lifecycle policies
- Amazon Bedrock: Knowledge Bases, Data Sources, Custom Models, Provisioned Throughput
- Terraform for serverless infrastructure: Lambda, API Gateway, Step Functions
- Network and data access policies for AOSS
- Cost optimization for serverless vector databases

Generate complete, production-deployable Terraform configurations.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

VECTOR_DIMENSIONS:      [OPTIONAL: 1536 for Titan v2]
EMBEDDING_MODEL_ARN:    [OPTIONAL: arn:aws:bedrock:{region}::foundation-model/amazon.titan-embed-text-v2:0]
GENERATION_MODEL_ARN:   [OPTIONAL: arn:aws:bedrock:{region}::foundation-model/anthropic.claude-3-5-sonnet-20241022-v2:0]

DOCUMENTS_BUCKET_NAME:  [OPTIONAL: {PROJECT_NAME}-{AWS_ACCOUNT_ID}-{ENV}-documents]
ENABLE_BEDROCK_KB:      [OPTIONAL: true - create managed Bedrock Knowledge Base]
ENABLE_LAMBDA_API:      [OPTIONAL: true - create Lambda + API Gateway for RAG queries]

AOSS_STANDBY_REPLICAS:  [OPTIONAL: DISABLED for dev, ENABLED for prod]
```

---

## Task

Generate Terraform modules for RAG:

```
terraform-rag/
├── main.tf
├── variables.tf
├── outputs.tf
├── providers.tf
├── environments/
│   ├── dev.tfvars
│   └── prod.tfvars
├── modules/
│   ├── opensearch-serverless/
│   │   ├── main.tf               # AOSS collection + policies
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── bedrock-kb/
│   │   ├── main.tf               # Bedrock Knowledge Base + Data Source
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── documents-bucket/
│   │   ├── main.tf               # S3 bucket for documents
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── lambda-rag-api/
│   │   ├── main.tf               # Lambda + API Gateway for queries
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── iam/
│       ├── main.tf               # IAM for Bedrock, AOSS, Lambda
│       ├── variables.tf
│       └── outputs.tf
└── lambda/
    ├── rag_query/
    │   ├── handler.py             # Lambda: RAG query handler
    │   └── requirements.txt
    └── ingestion_trigger/
        ├── handler.py             # Lambda: S3 event → Bedrock sync
        └── requirements.txt
```

**modules/opensearch-serverless/main.tf**:
- `aws_opensearchserverless_collection` (type: VECTORSEARCH)
- `aws_opensearchserverless_security_policy` (encryption + network)
- `aws_opensearchserverless_access_policy` (data access for Bedrock + Lambda roles)
- Standby replicas disabled for dev (cost saving)

**modules/bedrock-kb/main.tf**:
- `aws_bedrockagent_knowledge_base` with AOSS storage config
- `aws_bedrockagent_data_source` pointing to S3 documents bucket
- Chunking config: fixed size with overlap

**modules/lambda-rag-api/main.tf**:
- Lambda function for RAG queries (bedrock-agent-runtime.retrieve_and_generate)
- API Gateway (HTTP API) with CORS, throttling, API key auth
- CloudWatch log group with retention

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**AOSS:** Encryption policy required before collection creation. Network policy: public for dev (simplicity), VPC endpoint for prod. Data access policy grants read/write to specific IAM principals only.

**Cost:** AOSS minimum 2 OCUs ($0.24/OCU-hour ≈ $350/month). Use `DISABLED` standby replicas in dev. Share collection across environments if cost-sensitive.

**Security:** AOSS access via IAM (no username/password). Bedrock KB role: least-privilege for S3 read + AOSS write + Bedrock invoke. Lambda: execution role with only needed permissions.

---

## Integration Points

- **Upstream**: `mlops/04` → RAG pipeline code runs on this infrastructure
- **Alternative to**: `iac/02` CDK stack with same resources
- **Downstream**: `devops/03` → CloudWatch monitoring for AOSS and Lambda
- **Downstream**: `devops/04` → IAM roles from this module
