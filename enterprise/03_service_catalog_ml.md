<!-- Template Version: 1.0 | boto3: 1.35+ | cdk: 2.170+ -->

# Template Enterprise 03 — AWS Service Catalog Self-Service ML Platform

## Purpose
Generate a production-ready AWS Service Catalog self-service ML platform: a portfolio of approved ML infrastructure products (SageMaker Domain, training environment, inference endpoint) backed by CloudFormation templates, launch constraints that enforce approved IAM roles for provisioning, tag update constraints that mandate project and cost-center tags, a post-provisioning Lambda function that registers every provisioned product into a DynamoDB inventory table for tracking and chargeback, product versioning for controlled rollout of infrastructure updates, and principal associations that grant access to designated IAM roles.

---

## Role Definition

You are an expert AWS cloud architect and ML platform engineer specializing in self-service infrastructure with expertise in:
- AWS Service Catalog: portfolios, products, provisioning artifacts, launch constraints, tag update constraints, tag options, principal associations, provisioned product lifecycle
- CloudFormation product templates: parameterized templates for SageMaker Domain (VPC, auth mode, default user settings), training environments (SageMaker training job configurations, S3 buckets, IAM roles), and inference endpoints (endpoint configs, auto-scaling, data capture)
- Launch constraints: `servicecatalog.create_constraint()` with `Type='LAUNCH'` to enforce a specific IAM role for CloudFormation stack creation, ensuring least-privilege provisioning regardless of the requesting user's permissions
- Tag update constraints: `servicecatalog.create_constraint()` with `Type='TAG_UPDATE'` to enforce mandatory tags (Project, Environment, CostCenter, Owner) on all provisioned products
- Post-provisioning hooks: CloudFormation custom resources or Service Catalog notification constraints that trigger a Lambda function to register provisioned products in a DynamoDB inventory table
- Product versioning: `servicecatalog.create_provisioning_artifact()` for rolling out updated CloudFormation templates while maintaining backward compatibility for existing provisioned products
- DynamoDB inventory tracking: single-table design for recording provisioned product ID, product type, owner, tags, creation timestamp, stack ARN, and status
- Cost allocation: mandatory tagging strategies that feed into AWS Cost Explorer and FinOps dashboards
- SageMaker Domain provisioning: VPC configuration, IAM auth vs SSO auth, default user settings, app lifecycle management
- SageMaker endpoint deployment: instance type selection, auto-scaling policies, model data capture configuration

Generate complete, production-deployable Service Catalog self-service ML platform code.

---

## Context & Inputs

```
PROJECT_NAME:               [REQUIRED]
AWS_REGION:                 [REQUIRED]
AWS_ACCOUNT_ID:             [REQUIRED]
ENV:                        [REQUIRED - dev | stage | prod]

PORTFOLIO_NAME:             [REQUIRED - display name for the Service Catalog portfolio]
                            Example: "ML Platform Self-Service"
                            The portfolio groups all ML infrastructure products. Users
                            browse this portfolio to discover and provision approved ML
                            resources. Name should be descriptive for end users.

PRODUCTS:                   [REQUIRED - JSON list of products to include in the portfolio]
                            Example: ["sagemaker_domain", "training_env", "inference_endpoint"]
                            Available product types:
                            - "sagemaker_domain": SageMaker Studio Domain with VPC, auth mode,
                              default user settings, and app lifecycle configuration
                            - "training_env": SageMaker training environment with S3 buckets,
                              IAM execution role, VPC config, and approved instance types
                            - "inference_endpoint": SageMaker real-time inference endpoint with
                              endpoint config, auto-scaling policy, and optional data capture

ALLOWED_ROLES:              [REQUIRED - comma-separated list of IAM role ARNs allowed to provision products]
                            Example: "arn:aws:iam::123456789012:role/MLEngineer,arn:aws:iam::123456789012:role/DataScientist"
                            These roles are associated as principals with the portfolio.
                            Only users who can assume these roles can browse and provision
                            products from the portfolio.

MANDATORY_TAGS:             [REQUIRED - JSON object of tag keys and allowed values]
                            Example: {"Project": [], "Environment": ["dev","stage","prod"], "CostCenter": [], "Owner": []}
                            Empty array means any value is accepted. Non-empty array restricts
                            to the listed values. These tags are enforced via tag update
                            constraints on every provisioned product.

INVENTORY_TABLE_NAME:       [REQUIRED - DynamoDB table name for provisioned product inventory]
                            Example: "ml-platform-inventory"
                            Post-provisioning Lambda writes product metadata here for
                            tracking, chargeback, and compliance reporting.

LAUNCH_ROLE_ARN:            [OPTIONAL: auto-created]
                            IAM role ARN used by Service Catalog to provision CloudFormation
                            stacks. If not provided, a launch role is created with permissions
                            scoped to SageMaker, S3, IAM PassRole, and CloudWatch.

VPC_ID:                     [OPTIONAL: none]
                            VPC ID for SageMaker Domain and training environments. Required
                            if provisioning sagemaker_domain or training_env products in
                            VPC-only mode.

SUBNET_IDS:                 [OPTIONAL: none]
                            Comma-separated list of subnet IDs for SageMaker resources.
                            Required when VPC_ID is provided.

SECURITY_GROUP_IDS:         [OPTIONAL: none]
                            Comma-separated list of security group IDs for SageMaker resources.
                            Required when VPC_ID is provided.

APPROVED_INSTANCE_TYPES:    [OPTIONAL: ml.m5.xlarge,ml.m5.2xlarge,ml.g5.xlarge,ml.g5.2xlarge]
                            Comma-separated list of instance types allowed in training and
                            inference products. Enforced via CloudFormation AllowedValues.

KMS_KEY_ARN:                [OPTIONAL: none]
                            KMS key ARN for encrypting SageMaker volumes, S3 buckets, and
                            endpoint storage. When provided, all products enforce encryption.
```

---

## Task

Generate complete AWS Service Catalog self-service ML platform infrastructure:

```
{PROJECT_NAME}-service-catalog-ml/
├── portfolio/
│   ├── create_portfolio.py                 # Create Service Catalog portfolio
│   ├── associate_principals.py             # Associate IAM roles with portfolio
│   └── tag_options.py                      # Create and associate tag options
├── products/
│   ├── sagemaker_domain_product.py         # Register SageMaker Domain product
│   ├── training_env_product.py             # Register training environment product
│   ├── inference_endpoint_product.py       # Register inference endpoint product
│   └── product_versioning.py              # Create new provisioning artifact versions
├── templates/
│   ├── sagemaker_domain.yaml               # CloudFormation template — SageMaker Domain
│   ├── training_env.yaml                   # CloudFormation template — training environment
│   └── inference_endpoint.yaml             # CloudFormation template — inference endpoint
├── constraints/
│   ├── launch_constraints.py               # Launch constraints with approved IAM role
│   ├── tag_update_constraints.py           # Tag update constraints for mandatory tags
│   └── launch_role.py                      # Create launch role if not provided
├── inventory/
│   ├── inventory_table.py                  # Create DynamoDB inventory table
│   ├── inventory_lambda.py                 # Post-provisioning Lambda function code
│   └── notification_constraint.py          # SNS notification constraint for provisioning events
├── infrastructure/
│   └── config.py                           # Central configuration
├── run_setup.py                            # CLI orchestrator
└── requirements.txt
```

**config.py**: Central configuration dataclass with all parameters. Load from environment variables or CLI args. Validate required fields. Construct resource names using `{PROJECT_NAME}-{component}-{ENV}` convention. Parse PRODUCTS JSON list and validate each entry is one of `sagemaker_domain`, `training_env`, `inference_endpoint`. Parse MANDATORY_TAGS JSON object. Parse ALLOWED_ROLES comma-separated list into a Python list. Validate IAM role ARN format. Validate VPC/subnet/security group IDs when provided.

**create_portfolio.py**: Create the Service Catalog portfolio:
- Call `servicecatalog.create_portfolio()` with `DisplayName=PORTFOLIO_NAME`, `ProviderName=PROJECT_NAME`, `Description` describing the self-service ML platform
- Portfolio name follows convention: `{PROJECT_NAME}-ml-portfolio-{ENV}`
- Tag with Project, Environment, ManagedBy
- Store portfolio ID for subsequent product and constraint associations
- Call `servicecatalog.update_portfolio()` if portfolio already exists (idempotent)

**associate_principals.py**: Associate IAM roles with the portfolio:
- For each role ARN in ALLOWED_ROLES, call `servicecatalog.associate_principal_with_portfolio()` with `PrincipalARN=role_arn` and `PrincipalType='IAM'`
- Handle `ResourceNotFoundException` if portfolio does not exist
- Handle `DuplicateResourceException` if principal already associated
- Log each association with role ARN and portfolio ID

**tag_options.py**: Create tag options and associate with portfolio:
- For each tag key in MANDATORY_TAGS, call `servicecatalog.create_tag_option()` with `Key` and `Value` for each allowed value
- If allowed values list is empty (any value accepted), create a single tag option with a placeholder description
- Call `servicecatalog.associate_tag_option_with_resource()` to link each tag option to the portfolio
- Handle `DuplicateResourceException` for existing tag options

**sagemaker_domain.yaml**: CloudFormation template for SageMaker Domain product:
- Parameters: `ProjectName`, `Environment`, `VpcId`, `SubnetIds` (CommaDelimitedList), `SecurityGroupIds` (CommaDelimitedList), `AuthMode` (IAM or SSO, default IAM), `DefaultInstanceType` (AllowedValues from APPROVED_INSTANCE_TYPES), `KmsKeyArn` (optional)
- Resources:
  - `AWS::SageMaker::Domain` with `VpcId`, `SubnetIds`, `AuthMode`, `DefaultUserSettings` (execution role ARN, security groups, Jupyter server app settings with default instance type, kernel gateway app settings), `KmsKeyId` if provided
  - `AWS::IAM::Role` for SageMaker execution role with trust policy for `sagemaker.amazonaws.com`, managed policy `AmazonSageMakerFullAccess` scoped down, S3 access for project bucket, KMS decrypt if key provided
  - `AWS::S3::Bucket` for domain shared storage with encryption, versioning, and lifecycle rules
- Outputs: DomainId, DomainArn, DomainUrl, ExecutionRoleArn, S3BucketName
- Tags: all mandatory tags applied via `Tags` property on every resource

**training_env.yaml**: CloudFormation template for training environment product:
- Parameters: `ProjectName`, `Environment`, `InstanceType` (AllowedValues from APPROVED_INSTANCE_TYPES), `InstanceCount` (default 1), `MaxRuntimeSeconds` (default 86400), `VpcId`, `SubnetIds`, `SecurityGroupIds`, `S3OutputPath`, `KmsKeyArn` (optional)
- Resources:
  - `AWS::IAM::Role` for SageMaker training execution role with trust policy for `sagemaker.amazonaws.com`, permissions for S3 read/write on training data and output paths, ECR pull for training containers, CloudWatch Logs, KMS decrypt
  - `AWS::S3::Bucket` for training output with encryption, versioning, lifecycle (transition to IA after 30 days, Glacier after 90 days)
  - `AWS::CloudWatch::Alarm` for training job duration exceeding MaxRuntimeSeconds
  - `AWS::SNS::Topic` for training job notifications
- Outputs: ExecutionRoleArn, OutputBucketName, OutputBucketArn, SNSTopicArn
- Tags: all mandatory tags applied

**inference_endpoint.yaml**: CloudFormation template for inference endpoint product:
- Parameters: `ProjectName`, `Environment`, `ModelDataUrl` (S3 URI to model.tar.gz), `ContainerImage` (ECR URI), `InstanceType` (AllowedValues from APPROVED_INSTANCE_TYPES), `InstanceCount` (default 1), `MinCapacity` (default 1), `MaxCapacity` (default 4), `TargetInvocationsPerInstance` (default 100), `EnableDataCapture` (true/false, default false), `KmsKeyArn` (optional)
- Resources:
  - `AWS::IAM::Role` for SageMaker execution role with trust policy for `sagemaker.amazonaws.com`, S3 read for model artifacts, ECR pull, CloudWatch metrics, KMS decrypt
  - `AWS::SageMaker::Model` with `PrimaryContainer` (Image, ModelDataUrl), execution role, VPC config if provided
  - `AWS::SageMaker::EndpointConfig` with production variant (instance type, count, model name), `DataCaptureConfig` if enabled (destination S3, sampling percentage), KMS encryption
  - `AWS::SageMaker::Endpoint` referencing the endpoint config
  - `AWS::ApplicationAutoScaling::ScalableTarget` for endpoint variant with min/max capacity
  - `AWS::ApplicationAutoScaling::ScalingPolicy` with target tracking on `SageMakerVariantInvocationsPerInstance`
  - `AWS::CloudWatch::Alarm` for `Invocation5XXErrors` > 0 for 3 consecutive periods
- Outputs: EndpointName, EndpointArn, EndpointConfigName, ModelName, AutoScalingPolicyArn
- Tags: all mandatory tags applied

**sagemaker_domain_product.py**: Register the SageMaker Domain product in Service Catalog:
- Call `servicecatalog.create_product()` with `Name='{PROJECT_NAME}-sagemaker-domain-{ENV}'`, `Owner=PROJECT_NAME`, `ProductType='CLOUD_FORMATION_TEMPLATE'`, `ProvisioningArtifactParameters` pointing to the `sagemaker_domain.yaml` template uploaded to S3
- Upload `sagemaker_domain.yaml` to S3: `s3://{PROJECT_NAME}-sc-templates-{ENV}/templates/sagemaker_domain.yaml`
- Call `servicecatalog.associate_product_with_portfolio()` to add the product to the portfolio
- Tag with Product, Environment, ProductType=sagemaker_domain

**training_env_product.py**: Register the training environment product:
- Call `servicecatalog.create_product()` with `Name='{PROJECT_NAME}-training-env-{ENV}'`, `ProductType='CLOUD_FORMATION_TEMPLATE'`
- Upload `training_env.yaml` to S3
- Associate with portfolio
- Tag with ProductType=training_env

**inference_endpoint_product.py**: Register the inference endpoint product:
- Call `servicecatalog.create_product()` with `Name='{PROJECT_NAME}-inference-endpoint-{ENV}'`, `ProductType='CLOUD_FORMATION_TEMPLATE'`
- Upload `inference_endpoint.yaml` to S3
- Associate with portfolio
- Tag with ProductType=inference_endpoint

**product_versioning.py**: Create new provisioning artifact versions for existing products:
- Call `servicecatalog.create_provisioning_artifact()` with `ProductId`, `Parameters` (Name=version string, Info.LoadTemplateFromURL=S3 URL to updated template), `Type='CLOUD_FORMATION_TEMPLATE'`
- Version naming: `v{major}.{minor}` (e.g., `v1.0`, `v1.1`, `v2.0`)
- Optionally set `Guidance='DEPRECATED'` on old versions using `servicecatalog.update_provisioning_artifact()` to steer users toward the latest version
- List all versions with `servicecatalog.list_provisioning_artifacts()` and print version history

**launch_role.py**: Create the launch constraint IAM role (if LAUNCH_ROLE_ARN not provided):
- Role name: `{PROJECT_NAME}-sc-launch-role-{ENV}`
- Trust policy: `servicecatalog.amazonaws.com` and `cloudformation.amazonaws.com` as trusted principals
- Permissions: `sagemaker:Create*`, `sagemaker:Describe*`, `sagemaker:Delete*`, `sagemaker:Update*` scoped to resources tagged with Project={PROJECT_NAME}; `s3:CreateBucket`, `s3:PutBucketPolicy`, `s3:PutEncryptionConfiguration` for project buckets; `iam:CreateRole`, `iam:AttachRolePolicy`, `iam:PassRole` scoped to `{PROJECT_NAME}-*` role names; `cloudwatch:PutMetricAlarm`, `sns:CreateTopic`, `sns:Subscribe`; `kms:Decrypt`, `kms:DescribeKey`, `kms:GenerateDataKey` on KMS_KEY_ARN if provided; `application-autoscaling:RegisterScalableTarget`, `application-autoscaling:PutScalingPolicy`
- Tag with Project, Environment, ManagedBy=ServiceCatalog

**launch_constraints.py**: Create launch constraints for each product:
- For each product in the portfolio, call `servicecatalog.create_constraint()` with `Type='LAUNCH'`, `Parameters` containing the launch role ARN (either LAUNCH_ROLE_ARN or the auto-created role)
- This ensures all CloudFormation stacks are created using the launch role, not the end user's permissions
- Handle `DuplicateResourceException` if constraint already exists

**tag_update_constraints.py**: Create tag update constraints for mandatory tags:
- For each product in the portfolio, call `servicecatalog.create_constraint()` with `Type='TAG_UPDATE'`, `Parameters` containing the tag update rules
- Tag update constraint JSON: `{"TagUpdateOnProvisionedProduct": "ALLOWED"}` or `"NOT_ALLOWED"` per tag key
- For mandatory tags, set `"NOT_ALLOWED"` to prevent users from removing required tags after provisioning
- This enforces that mandatory tags set during provisioning cannot be deleted

**inventory_table.py**: Create DynamoDB inventory table:
- Table name: `{INVENTORY_TABLE_NAME}`
- Partition key: `ProvisionedProductId` (String)
- Sort key: `Timestamp` (String, ISO 8601)
- GSI: `ProductType-index` with partition key `ProductType` and sort key `Timestamp` for querying all products of a given type
- GSI: `Owner-index` with partition key `Owner` and sort key `Timestamp` for querying products by owner
- Billing mode: PAY_PER_REQUEST
- TTL attribute: `ExpiresAt` (for auto-cleanup of terminated product records after 365 days)
- Point-in-time recovery enabled
- Tags: Project, Environment

**inventory_lambda.py**: Post-provisioning Lambda function code:
- Triggered by SNS notification from Service Catalog provisioning events
- Parses the SNS message to extract: ProvisionedProductId, ProductId, ProvisioningArtifactId, RecordType (PROVISION/UPDATE/TERMINATE), RecordId, LaunchRoleArn, ProvisionedProductName
- Calls `servicecatalog.describe_provisioned_product()` to get full details including stack ARN, status, tags, and creation time
- Calls `servicecatalog.describe_record()` to get provisioning parameters and outputs
- Writes inventory record to DynamoDB with: ProvisionedProductId, Timestamp, ProductType, ProductName, Owner (from tags), StackArn, Status, Region, AccountId, Tags (all tags as a map), Outputs (CloudFormation outputs as a map), RecordType
- On TERMINATE events, updates the existing record with Status=TERMINATED and sets ExpiresAt TTL
- Logs all inventory operations to CloudWatch Logs

**notification_constraint.py**: Create SNS notification constraint for provisioning events:
- Create SNS topic: `{PROJECT_NAME}-sc-notifications-{ENV}`
- Subscribe the inventory Lambda function to the SNS topic
- For each product in the portfolio, call `servicecatalog.create_constraint()` with `Type='NOTIFICATION'`, `Parameters` containing the SNS topic ARN
- This triggers the Lambda on every provision, update, and terminate event
- Grant `sns:Publish` permission to Service Catalog service principal on the topic

**run_setup.py**: CLI orchestrator that runs setup steps in order:
1. Load and validate configuration
2. Create DynamoDB inventory table
3. Deploy inventory Lambda function and SNS topic
4. Create Service Catalog portfolio
5. Upload CloudFormation templates to S3
6. Register products (based on PRODUCTS list)
7. Associate products with portfolio
8. Create launch role (if needed) and launch constraints
9. Create tag update constraints
10. Create notification constraints
11. Associate principals with portfolio
12. Print summary with portfolio ID, product IDs, constraint IDs, and inventory table name

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**Launch Constraint Security:** Every product in the portfolio must have a launch constraint that specifies an approved IAM role. This role — not the end user's credentials — is used to create the CloudFormation stack. The launch role must follow least-privilege: only the permissions needed to create the specific product's resources. Never use `AdministratorAccess` or `PowerUserAccess` for launch roles. Scope `iam:PassRole` to specific SageMaker execution role name patterns (`{PROJECT_NAME}-*`).

**Mandatory Tagging:** All provisioned products must carry mandatory tags: `Project`, `Environment`, `CostCenter`, and `Owner`. Use tag update constraints to prevent users from removing these tags after provisioning. CloudFormation templates must propagate these tags to every resource via the `Tags` property. Tags feed into Cost Explorer for chargeback reporting via `finops/01`.

**CloudFormation Template Constraints:** CloudFormation product templates must use `AllowedValues` on parameters to restrict instance types to the approved list. Use `Conditions` to optionally enable features (KMS encryption, data capture, VPC mode). All templates must include `DeletionPolicy: Retain` on S3 buckets and `DeletionPolicy: Snapshot` on stateful resources to prevent accidental data loss during product termination.

**Product Versioning:** Use `servicecatalog.create_provisioning_artifact()` to publish new template versions. Set `Guidance='DEPRECATED'` on old versions to steer users toward the latest. Never delete old provisioning artifacts while active provisioned products reference them. Version naming follows `v{major}.{minor}` convention.

**Inventory Tracking:** Every provisioned product must be registered in the DynamoDB inventory table via the post-provisioning Lambda. The Lambda must handle all record types: PROVISION (new record), UPDATE (update existing record), TERMINATE (mark as terminated with TTL). Enable point-in-time recovery on the inventory table. Use GSIs for querying by ProductType and Owner.

**Idempotency:** All setup scripts must be idempotent. Handle `DuplicateResourceException` for portfolios, products, constraints, and principal associations. Use `describe_*` calls to check existence before creation. Use `update_*` calls for modifications rather than delete-and-recreate.

**Encryption:** When KMS_KEY_ARN is provided, all CloudFormation templates must encrypt SageMaker volumes, S3 buckets, and endpoint storage with the specified key. The launch role must have `kms:Decrypt`, `kms:GenerateDataKey`, and `kms:DescribeKey` permissions on the key. Reference `devops/08` for KMS key management patterns.

**SageMaker Domain Configuration:** SageMaker Domain products must enforce VPC-only mode when VPC_ID is provided. Default user settings should restrict instance types to the approved list. App lifecycle management should auto-terminate idle apps after a configurable timeout to reduce costs.

**Naming:** All resources follow `{PROJECT_NAME}-{component}-{ENV}` convention:
- Portfolio: `{PROJECT_NAME}-ml-portfolio-{ENV}`
- Products: `{PROJECT_NAME}-{product-type}-{ENV}` (e.g., `myai-sagemaker-domain-prod`)
- Launch role: `{PROJECT_NAME}-sc-launch-role-{ENV}`
- SNS topic: `{PROJECT_NAME}-sc-notifications-{ENV}`
- S3 template bucket: `{PROJECT_NAME}-sc-templates-{ENV}`
- DynamoDB table: `{INVENTORY_TABLE_NAME}` (user-provided)
- CloudFormation stacks: `SC-{ACCOUNT_ID}-pp-{random}` (Service Catalog managed)

**Security:** The portfolio must only be accessible to principals in ALLOWED_ROLES. Do not grant portfolio access to account root or broad IAM groups. The launch role must be assumable only by `servicecatalog.amazonaws.com` and `cloudformation.amazonaws.com` — not by end users directly. The inventory Lambda must have minimal permissions: DynamoDB write to the inventory table, Service Catalog describe calls, and CloudWatch Logs.

---

## Code Scaffolding Hints

**Create Service Catalog portfolio:**
```python
import boto3
import json

sc = boto3.client("servicecatalog", region_name=AWS_REGION)

# Create portfolio
response = sc.create_portfolio(
    DisplayName=PORTFOLIO_NAME,
    ProviderName=PROJECT_NAME,
    Description=f"Self-service ML platform portfolio for {PROJECT_NAME} — {ENV}",
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
        {"Key": "ManagedBy", "Value": "ServiceCatalog"},
    ],
    IdempotencyToken=f"{PROJECT_NAME}-ml-portfolio-{ENV}",
)
portfolio_id = response["PortfolioDetail"]["Id"]
print(f"Created portfolio: {portfolio_id} — {PORTFOLIO_NAME}")
```

**Associate IAM principals with portfolio:**
```python
for role_arn in ALLOWED_ROLES:
    try:
        sc.associate_principal_with_portfolio(
            PortfolioId=portfolio_id,
            PrincipalARN=role_arn,
            PrincipalType="IAM",
        )
        print(f"Associated principal {role_arn} with portfolio {portfolio_id}")
    except sc.exceptions.ResourceNotFoundException:
        print(f"ERROR: Portfolio {portfolio_id} not found")
        raise
    except Exception as e:
        if "DuplicateResource" in str(e):
            print(f"Principal {role_arn} already associated with portfolio")
        else:
            raise
```

**Upload CloudFormation template and create product:**
```python
import boto3

s3 = boto3.client("s3", region_name=AWS_REGION)
sc = boto3.client("servicecatalog", region_name=AWS_REGION)

template_bucket = f"{PROJECT_NAME}-sc-templates-{ENV}"
template_key = "templates/sagemaker_domain.yaml"

# Upload template to S3
with open("templates/sagemaker_domain.yaml", "r") as f:
    s3.put_object(
        Bucket=template_bucket,
        Key=template_key,
        Body=f.read(),
        ServerSideEncryption="aws:kms",
        SSEKMSKeyId=KMS_KEY_ARN,  # optional
    )

template_url = f"https://{template_bucket}.s3.{AWS_REGION}.amazonaws.com/{template_key}"

# Create product with initial provisioning artifact
response = sc.create_product(
    Name=f"{PROJECT_NAME}-sagemaker-domain-{ENV}",
    Owner=PROJECT_NAME,
    Description="Self-service SageMaker Studio Domain with VPC, auth mode, and default user settings",
    ProductType="CLOUD_FORMATION_TEMPLATE",
    ProvisioningArtifactParameters={
        "Name": "v1.0",
        "Description": "Initial release — SageMaker Domain with VPC-only mode",
        "Info": {
            "LoadTemplateFromURL": template_url,
        },
        "Type": "CLOUD_FORMATION_TEMPLATE",
    },
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
        {"Key": "ProductType", "Value": "sagemaker_domain"},
    ],
    IdempotencyToken=f"{PROJECT_NAME}-sagemaker-domain-{ENV}",
)
product_id = response["ProductViewDetail"]["ProductViewSummary"]["ProductId"]
artifact_id = response["ProvisioningArtifactDetail"]["Id"]
print(f"Created product: {product_id}, artifact: {artifact_id}")

# Associate product with portfolio
sc.associate_product_with_portfolio(
    ProductId=product_id,
    PortfolioId=portfolio_id,
)
print(f"Associated product {product_id} with portfolio {portfolio_id}")
```

**Create launch constraint with approved IAM role:**
```python
# Launch constraint ensures CloudFormation stacks are created using the
# launch role, not the end user's credentials
launch_constraint_params = json.dumps({
    "RoleArn": LAUNCH_ROLE_ARN,
    # e.g., "arn:aws:iam::123456789012:role/myai-sc-launch-role-prod"
})

response = sc.create_constraint(
    PortfolioId=portfolio_id,
    ProductId=product_id,
    Type="LAUNCH",
    Parameters=launch_constraint_params,
    Description=f"Launch constraint — use {PROJECT_NAME} approved launch role",
    IdempotencyToken=f"{PROJECT_NAME}-launch-{product_id}-{ENV}",
)
constraint_id = response["ConstraintDetail"]["ConstraintId"]
print(f"Created launch constraint: {constraint_id} for product {product_id}")
```

**Create tag update constraint for mandatory tags:**
```python
# Tag update constraint prevents users from removing mandatory tags
tag_update_params = json.dumps({
    "TagUpdateOnProvisionedProduct": "NOT_ALLOWED",
})

response = sc.create_constraint(
    PortfolioId=portfolio_id,
    ProductId=product_id,
    Type="TAG_UPDATE",
    Parameters=tag_update_params,
    Description="Prevent removal of mandatory tags on provisioned products",
    IdempotencyToken=f"{PROJECT_NAME}-tagupdate-{product_id}-{ENV}",
)
print(f"Created tag update constraint: {response['ConstraintDetail']['ConstraintId']}")
```

**Create notification constraint for inventory tracking:**
```python
sns = boto3.client("sns", region_name=AWS_REGION)

# Create SNS topic for provisioning notifications
topic_response = sns.create_topic(
    Name=f"{PROJECT_NAME}-sc-notifications-{ENV}",
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)
topic_arn = topic_response["TopicArn"]

# Allow Service Catalog to publish to the topic
sns.set_topic_attributes(
    TopicArn=topic_arn,
    AttributeName="Policy",
    AttributeValue=json.dumps({
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "AllowServiceCatalogPublish",
                "Effect": "Allow",
                "Principal": {"Service": "servicecatalog.amazonaws.com"},
                "Action": "sns:Publish",
                "Resource": topic_arn,
            }
        ],
    }),
)

# Create notification constraint
notification_params = json.dumps({
    "NotificationArns": [topic_arn],
})

response = sc.create_constraint(
    PortfolioId=portfolio_id,
    ProductId=product_id,
    Type="NOTIFICATION",
    Parameters=notification_params,
    Description="Notify inventory Lambda on provisioning events",
    IdempotencyToken=f"{PROJECT_NAME}-notify-{product_id}-{ENV}",
)
print(f"Created notification constraint: {response['ConstraintDetail']['ConstraintId']}")
```

**Create new provisioning artifact version:**
```python
# Publish a new version of an existing product template
response = sc.create_provisioning_artifact(
    ProductId=product_id,
    Parameters={
        "Name": "v1.1",
        "Description": "Added auto-scaling policy and data capture support",
        "Info": {
            "LoadTemplateFromURL": f"https://{template_bucket}.s3.{AWS_REGION}.amazonaws.com/templates/inference_endpoint_v1.1.yaml",
        },
        "Type": "CLOUD_FORMATION_TEMPLATE",
    },
    IdempotencyToken=f"{PROJECT_NAME}-artifact-v1.1-{product_id}",
)
new_artifact_id = response["ProvisioningArtifactDetail"]["Id"]
print(f"Created provisioning artifact v1.1: {new_artifact_id}")

# Deprecate the old version to steer users toward v1.1
sc.update_provisioning_artifact(
    ProductId=product_id,
    ProvisioningArtifactId=artifact_id,  # old v1.0 artifact
    Guidance="DEPRECATED",
)
print(f"Deprecated old artifact {artifact_id} — users will see deprecation warning")

# List all versions
artifacts = sc.list_provisioning_artifacts(ProductId=product_id)
for art in artifacts["ProvisioningArtifactDetails"]:
    print(f"  {art['Name']} ({art['Id']}) — Guidance: {art.get('Guidance', 'DEFAULT')}")
```

**Create launch role for Service Catalog:**
```python
iam = boto3.client("iam")

launch_role_trust = {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": [
                    "servicecatalog.amazonaws.com",
                    "cloudformation.amazonaws.com",
                ]
            },
            "Action": "sts:AssumeRole",
        }
    ],
}

response = iam.create_role(
    RoleName=f"{PROJECT_NAME}-sc-launch-role-{ENV}",
    AssumeRolePolicyDocument=json.dumps(launch_role_trust),
    Description=f"Service Catalog launch role for {PROJECT_NAME} ML products",
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
        {"Key": "ManagedBy", "Value": "ServiceCatalog"},
    ],
)
launch_role_arn = response["Role"]["Arn"]

# Attach scoped permissions for ML product provisioning
launch_policy = {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "SageMakerProvisioning",
            "Effect": "Allow",
            "Action": [
                "sagemaker:CreateDomain",
                "sagemaker:DeleteDomain",
                "sagemaker:DescribeDomain",
                "sagemaker:UpdateDomain",
                "sagemaker:CreateModel",
                "sagemaker:DeleteModel",
                "sagemaker:DescribeModel",
                "sagemaker:CreateEndpointConfig",
                "sagemaker:DeleteEndpointConfig",
                "sagemaker:DescribeEndpointConfig",
                "sagemaker:CreateEndpoint",
                "sagemaker:DeleteEndpoint",
                "sagemaker:DescribeEndpoint",
                "sagemaker:UpdateEndpoint",
                "sagemaker:AddTags",
                "sagemaker:DeleteTags",
                "sagemaker:ListTags",
            ],
            "Resource": f"arn:aws:sagemaker:{AWS_REGION}:{AWS_ACCOUNT_ID}:*",
        },
        {
            "Sid": "S3BucketProvisioning",
            "Effect": "Allow",
            "Action": [
                "s3:CreateBucket",
                "s3:DeleteBucket",
                "s3:PutBucketPolicy",
                "s3:PutEncryptionConfiguration",
                "s3:PutBucketVersioning",
                "s3:PutLifecycleConfiguration",
                "s3:PutBucketTagging",
                "s3:GetBucketLocation",
            ],
            "Resource": f"arn:aws:s3:::{PROJECT_NAME}-*",
        },
        {
            "Sid": "IAMRoleProvisioning",
            "Effect": "Allow",
            "Action": [
                "iam:CreateRole",
                "iam:DeleteRole",
                "iam:AttachRolePolicy",
                "iam:DetachRolePolicy",
                "iam:PutRolePolicy",
                "iam:DeleteRolePolicy",
                "iam:GetRole",
                "iam:TagRole",
                "iam:PassRole",
            ],
            "Resource": f"arn:aws:iam::{AWS_ACCOUNT_ID}:role/{PROJECT_NAME}-*",
        },
        {
            "Sid": "CloudWatchAlarms",
            "Effect": "Allow",
            "Action": [
                "cloudwatch:PutMetricAlarm",
                "cloudwatch:DeleteAlarms",
                "cloudwatch:DescribeAlarms",
            ],
            "Resource": f"arn:aws:cloudwatch:{AWS_REGION}:{AWS_ACCOUNT_ID}:alarm:{PROJECT_NAME}-*",
        },
        {
            "Sid": "SNSTopics",
            "Effect": "Allow",
            "Action": [
                "sns:CreateTopic",
                "sns:DeleteTopic",
                "sns:Subscribe",
                "sns:TagResource",
            ],
            "Resource": f"arn:aws:sns:{AWS_REGION}:{AWS_ACCOUNT_ID}:{PROJECT_NAME}-*",
        },
        {
            "Sid": "AutoScaling",
            "Effect": "Allow",
            "Action": [
                "application-autoscaling:RegisterScalableTarget",
                "application-autoscaling:DeregisterScalableTarget",
                "application-autoscaling:PutScalingPolicy",
                "application-autoscaling:DeleteScalingPolicy",
            ],
            "Resource": "*",
        },
        {
            "Sid": "CloudFormationStacks",
            "Effect": "Allow",
            "Action": [
                "cloudformation:CreateStack",
                "cloudformation:DeleteStack",
                "cloudformation:UpdateStack",
                "cloudformation:DescribeStacks",
                "cloudformation:DescribeStackEvents",
                "cloudformation:GetTemplate",
            ],
            "Resource": f"arn:aws:cloudformation:{AWS_REGION}:{AWS_ACCOUNT_ID}:stack/SC-*",
        },
    ],
}

# Add KMS permissions if key provided
if KMS_KEY_ARN:
    launch_policy["Statement"].append({
        "Sid": "KMSAccess",
        "Effect": "Allow",
        "Action": [
            "kms:Decrypt",
            "kms:DescribeKey",
            "kms:GenerateDataKey",
        ],
        "Resource": KMS_KEY_ARN,
    })

iam.put_role_policy(
    RoleName=f"{PROJECT_NAME}-sc-launch-role-{ENV}",
    PolicyName=f"{PROJECT_NAME}-sc-launch-policy",
    PolicyDocument=json.dumps(launch_policy),
)
print(f"Created launch role: {launch_role_arn}")
```

**DynamoDB inventory table and post-provisioning Lambda:**
```python
import boto3

dynamodb = boto3.client("dynamodb", region_name=AWS_REGION)

# Create inventory table
dynamodb.create_table(
    TableName=INVENTORY_TABLE_NAME,
    KeySchema=[
        {"AttributeName": "ProvisionedProductId", "KeyType": "HASH"},
        {"AttributeName": "Timestamp", "KeyType": "RANGE"},
    ],
    AttributeDefinitions=[
        {"AttributeName": "ProvisionedProductId", "AttributeType": "S"},
        {"AttributeName": "Timestamp", "AttributeType": "S"},
        {"AttributeName": "ProductType", "AttributeType": "S"},
        {"AttributeName": "Owner", "AttributeType": "S"},
    ],
    GlobalSecondaryIndexes=[
        {
            "IndexName": "ProductType-index",
            "KeySchema": [
                {"AttributeName": "ProductType", "KeyType": "HASH"},
                {"AttributeName": "Timestamp", "KeyType": "RANGE"},
            ],
            "Projection": {"ProjectionType": "ALL"},
        },
        {
            "IndexName": "Owner-index",
            "KeySchema": [
                {"AttributeName": "Owner", "KeyType": "HASH"},
                {"AttributeName": "Timestamp", "KeyType": "RANGE"},
            ],
            "Projection": {"ProjectionType": "ALL"},
        },
    ],
    BillingMode="PAY_PER_REQUEST",
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)

# Enable point-in-time recovery
dynamodb.update_continuous_backups(
    TableName=INVENTORY_TABLE_NAME,
    PointInTimeRecoverySpecification={"PointInTimeRecoveryEnabled": True},
)

# Enable TTL for auto-cleanup of terminated records
dynamodb.update_time_to_live(
    TableName=INVENTORY_TABLE_NAME,
    TimeToLiveSpecification={
        "Enabled": True,
        "AttributeName": "ExpiresAt",
    },
)
print(f"Created inventory table: {INVENTORY_TABLE_NAME}")
```

**Post-provisioning Lambda handler:**
```python
import json
import boto3
from datetime import datetime, timedelta

dynamodb = boto3.resource("dynamodb")
sc = boto3.client("servicecatalog")
table = dynamodb.Table(INVENTORY_TABLE_NAME)


def handler(event, context):
    """Process Service Catalog provisioning notifications from SNS."""
    for record in event["Records"]:
        message = json.loads(record["Sns"]["Message"])
        provisioned_product_id = message.get("ProvisionedProductId")
        record_type = message.get("RecordType")  # PROVISION, UPDATE, TERMINATE

        # Get full provisioned product details
        pp_detail = sc.describe_provisioned_product(Id=provisioned_product_id)
        pp = pp_detail["ProvisionedProductDetail"]

        # Get provisioning record for outputs and parameters
        record_detail = sc.describe_record(Id=message.get("RecordId"))

        # Extract tags
        tags = {t["Key"]: t["Value"] for t in pp.get("Tags", [])}

        now = datetime.utcnow().isoformat() + "Z"

        if record_type == "TERMINATE":
            # Mark as terminated with TTL for auto-cleanup
            expires_at = int((datetime.utcnow() + timedelta(days=365)).timestamp())
            table.put_item(Item={
                "ProvisionedProductId": provisioned_product_id,
                "Timestamp": now,
                "ProductType": tags.get("ProductType", "unknown"),
                "ProductName": pp.get("Name", ""),
                "Owner": tags.get("Owner", "unknown"),
                "Status": "TERMINATED",
                "RecordType": record_type,
                "Region": AWS_REGION,
                "AccountId": AWS_ACCOUNT_ID,
                "Tags": tags,
                "ExpiresAt": expires_at,
            })
        else:
            # Extract CloudFormation outputs
            outputs = {
                o["OutputKey"]: o["OutputValue"]
                for o in record_detail.get("RecordOutputs", [])
            }
            table.put_item(Item={
                "ProvisionedProductId": provisioned_product_id,
                "Timestamp": now,
                "ProductType": tags.get("ProductType", "unknown"),
                "ProductName": pp.get("Name", ""),
                "Owner": tags.get("Owner", "unknown"),
                "StackArn": pp.get("PhysicalId", ""),
                "Status": pp.get("Status", ""),
                "RecordType": record_type,
                "Region": AWS_REGION,
                "AccountId": AWS_ACCOUNT_ID,
                "Tags": tags,
                "Outputs": outputs,
            })

        print(f"Inventory {record_type}: {provisioned_product_id} — {pp.get('Name', '')}")

    return {"statusCode": 200}
```

**CloudFormation template snippet — SageMaker Domain product:**
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: SageMaker Studio Domain — Service Catalog Product

Parameters:
  ProjectName:
    Type: String
    Description: Project name for resource naming
  Environment:
    Type: String
    AllowedValues: [dev, stage, prod]
  VpcId:
    Type: AWS::EC2::VPC::Id
    Description: VPC for SageMaker Domain
  SubnetIds:
    Type: List<AWS::EC2::Subnet::Id>
    Description: Subnets for SageMaker Domain
  SecurityGroupIds:
    Type: List<AWS::EC2::SecurityGroup::Id>
    Description: Security groups for SageMaker Domain
  AuthMode:
    Type: String
    Default: IAM
    AllowedValues: [IAM, SSO]
  DefaultInstanceType:
    Type: String
    Default: ml.m5.xlarge
    AllowedValues:
      - ml.m5.xlarge
      - ml.m5.2xlarge
      - ml.g5.xlarge
      - ml.g5.2xlarge
  KmsKeyArn:
    Type: String
    Default: ''
    Description: Optional KMS key ARN for encryption

Conditions:
  HasKmsKey: !Not [!Equals [!Ref KmsKeyArn, '']]

Resources:
  SageMakerExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub '${ProjectName}-sm-exec-${Environment}'
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: sagemaker.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AmazonSageMakerFullAccess
      Tags:
        - Key: Project
          Value: !Ref ProjectName
        - Key: Environment
          Value: !Ref Environment

  SageMakerDomain:
    Type: AWS::SageMaker::Domain
    Properties:
      DomainName: !Sub '${ProjectName}-domain-${Environment}'
      AuthMode: !Ref AuthMode
      VpcId: !Ref VpcId
      SubnetIds: !Ref SubnetIds
      DefaultUserSettings:
        ExecutionRole: !GetAtt SageMakerExecutionRole.Arn
        SecurityGroups: !Ref SecurityGroupIds
        JupyterServerAppSettings:
          DefaultResourceSpec:
            InstanceType: system
        KernelGatewayAppSettings:
          DefaultResourceSpec:
            InstanceType: !Ref DefaultInstanceType
      KmsKeyId: !If [HasKmsKey, !Ref KmsKeyArn, !Ref 'AWS::NoValue']
      Tags:
        - Key: Project
          Value: !Ref ProjectName
        - Key: Environment
          Value: !Ref Environment

Outputs:
  DomainId:
    Value: !Ref SageMakerDomain
  DomainArn:
    Value: !GetAtt SageMakerDomain.DomainArn
  DomainUrl:
    Value: !GetAtt SageMakerDomain.Url
  ExecutionRoleArn:
    Value: !GetAtt SageMakerExecutionRole.Arn
```

---

## Integration Points

- **Upstream**: `devops/04` (IAM Roles & Policies) → Provides IAM roles referenced by ALLOWED_ROLES for portfolio principal association and the SageMaker execution roles used within CloudFormation product templates
- **Upstream**: `enterprise/01` (Organizations SCPs) → SCPs restrict which instance types, regions, and Bedrock models can be used; Service Catalog product templates must only offer parameters that comply with active SCPs to avoid provisioning failures
- **Upstream**: `devops/08` (KMS Encryption) → Provides KMS_KEY_ARN used by CloudFormation product templates for encrypting SageMaker volumes, S3 buckets, and endpoint storage
- **Upstream**: `devops/02` (VPC Networking) → Provides VPC_ID, SUBNET_IDS, and SECURITY_GROUP_IDS used by SageMaker Domain and training environment products for VPC-only mode
- **Downstream**: `enterprise/04` (Control Tower) → Control Tower account baselines can include Service Catalog portfolio sharing to automatically grant new ML accounts access to the self-service portfolio
- **Downstream**: `finops/01` (Cost Allocation) → Mandatory tags (Project, CostCenter, Owner) enforced by tag update constraints feed into Cost Explorer and Budgets for per-team and per-project cost tracking
- **Downstream**: `finops/05` (FinOps Dashboards) → DynamoDB inventory table provides provisioned product metadata for cost attribution dashboards and chargeback reports
- **Downstream**: `devops/05` (Config Rules) → Config rules can validate that all SageMaker resources were provisioned through Service Catalog (checking for required tags and launch role usage)
