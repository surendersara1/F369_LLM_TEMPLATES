<!-- Template Version: 1.0 | boto3: 1.35+ | S3 Control: 2018-08-20 -->

# Template Data 04 — S3 Data Lifecycle Management for ML Datasets

## Purpose
Generate a production-ready S3 data lifecycle management layer for ML datasets: Intelligent-Tiering configuration with archive access tiers for infrequently accessed training data, lifecycle rules transitioning ML artifacts through Standard → Intelligent-Tiering → Glacier Instant Retrieval → Glacier Deep Archive based on configurable age thresholds, S3 Batch Operations jobs for bulk dataset management (copying, tagging, restoring), S3 Access Points with per-team/project access isolation, S3 Inventory configuration for tracking dataset sizes and storage classes, and a restore-and-wait pattern using S3 restore requests with EventBridge notifications for Glacier restore completion.

---

## Role Definition

You are an expert AWS storage architect and ML platform engineer with expertise in:
- Amazon S3 storage optimization: Intelligent-Tiering, storage class transitions, lifecycle policies, cost analysis
- S3 Intelligent-Tiering: automatic tiering configuration, archive access tier activation, monitoring access patterns
- S3 Lifecycle rules: transition rules, expiration rules, filter-based policies, noncurrent version management
- S3 Batch Operations: `s3control.create_job()`, manifest-based bulk operations, copy/tag/restore jobs, job reporting
- S3 Access Points: `s3control.create_access_point()`, access point policies, per-team data isolation, VPC-restricted access
- S3 Inventory: `s3.put_bucket_inventory_configuration()`, daily/weekly reports, storage class and encryption tracking
- Glacier restore patterns: `s3.restore_object()`, restore tiers (Bulk/Standard/Expedited), EventBridge restore completion events
- S3 event-driven patterns: EventBridge rules for S3 object events, restore completion notifications, Lambda triggers
- KMS encryption: S3 bucket encryption with customer-managed KMS keys, cross-account key access
- IAM policies for S3 access, S3 Control operations, and cross-service permissions

Generate complete, production-deployable code.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

BUCKET_NAMES:           [REQUIRED - JSON list of S3 bucket configurations for ML data]
                         Example:
                         [
                           {
                             "name": "ml-training-data",
                             "purpose": "raw training datasets",
                             "kms_key_alias": "alias/ml-data-key"
                           },
                           {
                             "name": "ml-model-artifacts",
                             "purpose": "trained model artifacts and checkpoints",
                             "kms_key_alias": "alias/ml-model-key"
                           },
                           {
                             "name": "ml-feature-store",
                             "purpose": "processed feature datasets",
                             "kms_key_alias": "alias/ml-data-key"
                           }
                         ]
                         Full bucket names become: {PROJECT_NAME}-{name}-{ENV}

LIFECYCLE_RULES:        [REQUIRED - JSON lifecycle rule definitions with age thresholds in days]
                         Example:
                         [
                           {
                             "id": "training-data-tiering",
                             "prefix": "datasets/",
                             "transitions": [
                               {"days": 30, "storage_class": "INTELLIGENT_TIERING"},
                               {"days": 90, "storage_class": "GLACIER_IR"},
                               {"days": 365, "storage_class": "DEEP_ARCHIVE"}
                             ],
                             "noncurrent_transitions": [
                               {"days": 30, "storage_class": "GLACIER_IR"}
                             ],
                             "expiration_days": null
                           },
                           {
                             "id": "checkpoint-cleanup",
                             "prefix": "checkpoints/",
                             "transitions": [
                               {"days": 7, "storage_class": "INTELLIGENT_TIERING"}
                             ],
                             "noncurrent_transitions": [],
                             "expiration_days": 90
                           }
                         ]

INTELLIGENT_TIERING_CONFIG: [OPTIONAL: default archive after 90 days, deep archive after 180 days]
                         JSON configuration for S3 Intelligent-Tiering archive tiers.
                         Example:
                         {
                           "archive_access_days": 90,
                           "deep_archive_access_days": 180,
                           "filter_prefix": "datasets/"
                         }

BATCH_OPERATION_TYPE:   [OPTIONAL: none]
                         Type of S3 Batch Operation to configure.
                         Options: copy | tag | restore | none
                         - copy: bulk copy datasets across buckets or storage classes
                         - tag: bulk apply tags to ML datasets for cost allocation
                         - restore: bulk restore archived datasets from Glacier
                         - none: skip batch operations setup

ACCESS_POINT_CONFIGS:   [OPTIONAL: none]
                         JSON list of S3 Access Point configurations per team/project.
                         Example:
                         [
                           {
                             "name": "ml-research-ap",
                             "bucket": "ml-training-data",
                             "policy_principal": "arn:aws:iam::123456789012:role/MLResearchRole",
                             "prefix_restriction": "datasets/research/",
                             "vpc_id": null
                           },
                           {
                             "name": "ml-engineering-ap",
                             "bucket": "ml-training-data",
                             "policy_principal": "arn:aws:iam::123456789012:role/MLEngineeringRole",
                             "prefix_restriction": "datasets/production/",
                             "vpc_id": "vpc-0abc123def456"
                           }
                         ]

INVENTORY_DESTINATION_BUCKET: [OPTIONAL - S3 bucket for inventory reports]
                         If not provided, inventory reports are stored in the first bucket
                         from BUCKET_NAMES with prefix `inventory/`.

RESTORE_NOTIFICATION_EMAIL: [OPTIONAL - email address for Glacier restore completion notifications]
                         Used with the restore-and-wait EventBridge pattern.

S3_BATCH_ROLE_ARN:      [OPTIONAL - IAM role ARN for S3 Batch Operations]
                         If not provided, a role is created with required permissions.
```

---

## Task

Generate complete S3 data lifecycle management infrastructure for ML datasets:

```
{PROJECT_NAME}-s3-lifecycle/
├── tiering/
│   ├── configure_intelligent_tiering.py   # S3 Intelligent-Tiering configuration
│   └── analyze_access_patterns.py         # Analyze object access patterns for tiering decisions
├── lifecycle/
│   ├── configure_lifecycle_rules.py       # S3 Lifecycle rule configuration
│   └── validate_transitions.py            # Validate lifecycle rule transitions
├── batch_operations/
│   ├── create_batch_copy_job.py           # S3 Batch Operations copy job
│   ├── create_batch_tag_job.py            # S3 Batch Operations tagging job
│   ├── create_batch_restore_job.py        # S3 Batch Operations Glacier restore job
│   └── generate_manifest.py              # Generate CSV manifest from S3 Inventory
├── access_points/
│   ├── create_access_points.py            # Create S3 Access Points per team/project
│   └── access_point_policies.py           # Access point policy definitions
├── inventory/
│   ├── configure_inventory.py             # S3 Inventory configuration
│   └── inventory_queries.py              # Athena queries for inventory analysis
├── restore/
│   ├── restore_objects.py                 # Restore objects from Glacier
│   └── restore_eventbridge.py            # EventBridge rule for restore completion
├── infrastructure/
│   ├── config.py                          # Central configuration
│   └── s3_helpers.py                      # S3 helper utilities
├── run_setup.py                           # CLI orchestrator
└── requirements.txt
```

**config.py**: Central configuration dataclass with all parameters. Load from environment variables or CLI args. Validate required fields. Construct bucket names using `{PROJECT_NAME}-{name}-{ENV}` convention. Parse BUCKET_NAMES, LIFECYCLE_RULES, INTELLIGENT_TIERING_CONFIG, ACCESS_POINT_CONFIGS from JSON strings.

**configure_intelligent_tiering.py**: Configure S3 Intelligent-Tiering with archive access tiers:
- Call `s3.put_bucket_intelligent_tiering_configuration()` for each bucket in BUCKET_NAMES
- Set `Id` to `{PROJECT_NAME}-it-config-{ENV}`
- Set `Status` to `Enabled`
- Configure `Tierings` with `ARCHIVE_ACCESS` tier at INTELLIGENT_TIERING_CONFIG.archive_access_days (default 90)
- Configure `Tierings` with `DEEP_ARCHIVE_ACCESS` tier at INTELLIGENT_TIERING_CONFIG.deep_archive_access_days (default 180)
- Apply `Filter` with `Prefix` from INTELLIGENT_TIERING_CONFIG.filter_prefix if specified
- Apply `Filter` with `Tag` for `Project={PROJECT_NAME}` if no prefix filter

**analyze_access_patterns.py**: Analyze S3 object access patterns for tiering decisions:
- Use `s3.list_objects_v2()` with pagination to enumerate objects
- Use `s3.head_object()` to check `StorageClass` and `LastModified` for sampled objects
- Use CloudWatch `s3.get_metric_data()` for `NumberOfObjects` and `BucketSizeBytes` per storage class
- Generate a summary report: object count by storage class, total size by storage class, estimated monthly cost, recommended tiering thresholds

**configure_lifecycle_rules.py**: Configure S3 Lifecycle rules from LIFECYCLE_RULES:
- Call `s3.put_bucket_lifecycle_configuration()` for each bucket
- Build `Rules` list from LIFECYCLE_RULES with:
  - `ID`: rule id from config
  - `Filter.Prefix`: prefix from config
  - `Status`: `Enabled`
  - `Transitions`: list of `{"Days": days, "StorageClass": storage_class}` from config
  - `NoncurrentVersionTransitions`: from noncurrent_transitions config
  - `Expiration.Days`: from expiration_days if set
- Add a default rule for aborting incomplete multipart uploads after 7 days
- Add a default rule for expiring noncurrent object versions after 90 days (versioned buckets)

**validate_transitions.py**: Validate lifecycle rule transitions before applying:
- Check that transition days are in ascending order
- Check that storage class transitions follow valid paths (Standard → IT → Glacier IR → Deep Archive)
- Check minimum days between transitions (30 days minimum for most transitions)
- Check that objects smaller than 128 KB are excluded from Intelligent-Tiering (they are not eligible)
- Return validation errors or success

**create_batch_copy_job.py**: Create S3 Batch Operations copy job using `s3control.create_job()`:
- Set `AccountId` to AWS_ACCOUNT_ID
- Set `Operation.S3PutObjectCopy` with `TargetResource` (destination bucket ARN), `StorageClass`, and optional `SSEAwsKmsKeyId`
- Set `Manifest` with `Spec.Format` as `S3InventoryReport_CSV_20161130` or `S3BatchOperations_CSV_20180820`
- Set `Manifest.Location` with `ObjectArn` pointing to the manifest CSV in S3
- Set `Report` with `Bucket`, `Format=Report_CSV_20180820`, `Enabled=True`, `ReportScope=AllTasks`
- Set `Priority` to 10, `RoleArn` to S3_BATCH_ROLE_ARN
- Set `ConfirmationRequired` to False for automated execution
- Tag with Project and Environment

**create_batch_tag_job.py**: Create S3 Batch Operations tagging job:
- Set `Operation.S3PutObjectTagging` with `TagSet` containing ML cost allocation tags:
  - `Project={PROJECT_NAME}`, `Environment={ENV}`, `DataType=training|model|feature`
- Same manifest and report configuration as copy job

**create_batch_restore_job.py**: Create S3 Batch Operations Glacier restore job:
- Set `Operation.S3InitiateRestoreObject` with `ExpirationInDays` (default 7) and `GlacierJobTier` (`BULK` for cost savings, `STANDARD` for faster access)
- Same manifest and report configuration as copy job
- Include post-restore EventBridge notification setup

**generate_manifest.py**: Generate CSV manifest for S3 Batch Operations:
- Option 1: Generate from S3 Inventory report (preferred for large datasets)
- Option 2: Generate from `s3.list_objects_v2()` with prefix filtering
- Write CSV with columns: `Bucket, Key` (minimum required fields)
- Upload manifest to S3 and return the manifest S3 ARN

**create_access_points.py**: Create S3 Access Points per team/project:
- Iterate ACCESS_POINT_CONFIGS and call `s3control.create_access_point()` for each
- Set `AccountId` to AWS_ACCOUNT_ID
- Set `Name` to `{PROJECT_NAME}-{ap_name}-{ENV}`
- Set `Bucket` to the full bucket name
- Set `PublicAccessBlockConfiguration` with all blocks enabled (BlockPublicAcls, IgnorePublicAcls, BlockPublicPolicy, RestrictPublicBuckets all True)
- Set `VpcConfiguration.VpcId` if vpc_id is specified (VPC-restricted access)
- After creation, attach access point policy using `s3control.put_access_point_policy()`

**access_point_policies.py**: Define access point policies:
- Generate IAM policy documents that restrict access to specific S3 prefixes per team
- Use `s3:prefix` condition key to limit access to `prefix_restriction` from config
- Use `aws:PrincipalArn` condition to restrict to `policy_principal` from config
- Include `s3:GetObject`, `s3:PutObject`, `s3:ListBucket` with prefix conditions
- Deny `s3:DeleteObject` for production data prefixes

**configure_inventory.py**: Configure S3 Inventory for tracking ML datasets:
- Call `s3.put_bucket_inventory_configuration()` for each bucket in BUCKET_NAMES
- Set `Id` to `{PROJECT_NAME}-inventory-{ENV}`
- Set `Destination.S3BucketDestination` with `Bucket` (inventory destination ARN), `Format=CSV`, `Prefix=inventory/{bucket_name}/`
- Set `Schedule.Frequency` to `Weekly`
- Set `IncludedObjectVersions` to `Current`
- Set `OptionalFields` to `['Size', 'LastModifiedDate', 'StorageClass', 'EncryptionStatus', 'IntelligentTieringAccessTier', 'ETag']`
- Set `IsEnabled` to True

**inventory_queries.py**: Athena queries for S3 Inventory analysis:
- Create Athena table DDL for S3 Inventory CSV format
- Named queries:
  - Storage class distribution: count and total size by storage class
  - Large objects report: objects over 1 GB sorted by size
  - Encryption status: count of encrypted vs unencrypted objects
  - Intelligent-Tiering tier distribution: objects by access tier
  - Stale data report: objects not modified in over 365 days
  - Cost estimation: estimated monthly cost by storage class using published pricing

**restore_objects.py**: Restore objects from Glacier storage classes:
- Call `s3.restore_object()` with `RestoreRequest`:
  - `Days`: number of days to keep the restored copy (default 7)
  - `GlacierJobParameters.Tier`: `Bulk` (5-12 hours, cheapest), `Standard` (3-5 hours), or `Expedited` (1-5 minutes, most expensive)
- Handle `RestoreAlreadyInProgress` error gracefully
- Support batch restore by iterating a list of object keys
- Log restore request status for each object

**restore_eventbridge.py**: EventBridge rule for Glacier restore completion:
- Create EventBridge rule matching S3 `Object Restore Completed` events using `events.put_rule()`
- Set event pattern to match `source: aws.s3`, `detail-type: Object Restore Completed`, filtered by bucket name
- Create SNS topic for restore notifications (if RESTORE_NOTIFICATION_EMAIL provided)
- Add SNS topic as EventBridge target using `events.put_targets()`
- Optionally add Lambda target that triggers downstream ML training pipeline after restore completes

**run_setup.py**: CLI orchestrator that runs setup steps in order:
1. Validate lifecycle rule transitions
2. Configure Intelligent-Tiering on all buckets
3. Configure lifecycle rules on all buckets
4. Configure S3 Inventory on all buckets
5. Create S3 Access Points (if ACCESS_POINT_CONFIGS provided)
6. Set up Batch Operations job (if BATCH_OPERATION_TYPE specified)
7. Set up restore EventBridge rule (if RESTORE_NOTIFICATION_EMAIL provided)
8. Print summary with bucket names, lifecycle rule count, access point names, and inventory config

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**Intelligent-Tiering:** Enable S3 Intelligent-Tiering on all ML data buckets to automatically optimize storage costs. Activate the Archive Access tier (objects not accessed for 90+ days) and Deep Archive Access tier (objects not accessed for 180+ days). Objects smaller than 128 KB are not eligible for Intelligent-Tiering monitoring — they remain in the Frequent Access tier. There is a small monthly monitoring and automation charge per object (no retrieval fees). Use `put_bucket_intelligent_tiering_configuration()` with `Filter` to scope tiering to specific prefixes (e.g., `datasets/`).

**Lifecycle Rules:** Define lifecycle rules that transition ML artifacts through storage classes based on age: Standard (0-30 days, active training) → Intelligent-Tiering (30-90 days, variable access) → Glacier Instant Retrieval (90-365 days, rare access with millisecond retrieval) → Glacier Deep Archive (365+ days, compliance retention). Minimum object size for transitions is 128 KB. Minimum days between transitions: 30 days from Standard to any other class. Always include a rule to abort incomplete multipart uploads after 7 days to prevent storage waste. For versioned buckets, add noncurrent version transitions and expiration.

**Batch Operations:** Use S3 Batch Operations for bulk dataset management. Create jobs using `s3control.create_job()` with a manifest (CSV or S3 Inventory report) listing target objects. For copy operations, use `S3PutObjectCopy` with target bucket and optional storage class change. For tagging, use `S3PutObjectTagging` with ML cost allocation tags. For Glacier restores, use `S3InitiateRestoreObject` with `BULK` tier for cost savings (5-12 hours) or `STANDARD` tier for faster access (3-5 hours). Always enable job reports to track success/failure per object. Set `ConfirmationRequired=False` only for automated pipelines.

**Access Points:** Create S3 Access Points to isolate data access per ML team or project. Each access point has its own DNS name and access policy, simplifying permission management. Block all public access on access points (`BlockPublicAcls`, `IgnorePublicAcls`, `BlockPublicPolicy`, `RestrictPublicBuckets` all True). Use VPC-restricted access points for production data by setting `VpcConfiguration.VpcId`. Access point policies should restrict to specific S3 prefixes using `s3:prefix` condition keys and specific IAM principals using `aws:PrincipalArn`.

**S3 Inventory:** Configure weekly S3 Inventory reports to track dataset sizes, storage classes, and encryption status across all ML data buckets. Include optional fields: `Size`, `LastModifiedDate`, `StorageClass`, `EncryptionStatus`, `IntelligentTieringAccessTier`, `ETag`. Store inventory reports in a dedicated prefix (`inventory/`) with a bucket policy granting S3 Inventory write access. Use Athena to query inventory reports for cost analysis and compliance auditing.

**Glacier Restore:** When ML training jobs require data archived in Glacier, use the restore-and-wait pattern: (1) initiate restore with `s3.restore_object()` specifying tier and expiration days, (2) create an EventBridge rule matching `Object Restore Completed` events for the bucket, (3) trigger downstream ML pipeline via SNS or Lambda when restore completes. Use `BULK` tier for scheduled restores (cheapest, 5-12 hours). Use `STANDARD` tier for ad-hoc restores (3-5 hours). Use `Expedited` tier only for urgent small restores (1-5 minutes, expensive). Restored copies are temporary — set `Days` to cover the training job duration plus buffer.

**Security:** Enable default encryption on all ML data buckets using KMS customer-managed keys from `devops/08`. Use bucket policies to enforce `s3:x-amz-server-side-encryption=aws:kms` on all PutObject requests. Enable S3 Versioning on all buckets for data protection and rollback. Enable S3 Object Lock in Governance mode for compliance-critical training datasets. Use VPC endpoints for S3 access from SageMaker and Glue jobs.

**Cost:** Monitor storage costs using S3 Storage Lens and CloudWatch `BucketSizeBytes` metrics per storage class. Intelligent-Tiering monitoring cost is $0.0025 per 1,000 objects/month. Glacier Instant Retrieval has per-GB retrieval fees — use for data accessed once per quarter. Glacier Deep Archive is the cheapest storage ($0.00099/GB/month) but has 12-hour minimum restore time. Use S3 Inventory reports to identify and clean up orphaned data, incomplete multipart uploads, and stale objects.

**Naming:** All resources follow `{PROJECT_NAME}-{component}-{ENV}` convention:
- Buckets: `{PROJECT_NAME}-{bucket_name}-{ENV}`
- Intelligent-Tiering config: `{PROJECT_NAME}-it-config-{ENV}`
- Lifecycle rules: IDs from LIFECYCLE_RULES config
- Access points: `{PROJECT_NAME}-{ap_name}-{ENV}`
- Inventory config: `{PROJECT_NAME}-inventory-{ENV}`
- Batch Operations role: `{PROJECT_NAME}-s3-batch-role-{ENV}`
- EventBridge rule: `{PROJECT_NAME}-s3-restore-complete-{ENV}`
- SNS topic: `{PROJECT_NAME}-s3-restore-notify-{ENV}`

---

## Code Scaffolding Hints

**Configure S3 Intelligent-Tiering with archive access tiers:**
```python
import boto3

s3 = boto3.client("s3", region_name=AWS_REGION)

def configure_intelligent_tiering(bucket_name, config_id, archive_days=90,
                                   deep_archive_days=180, filter_prefix=None):
    """Configure S3 Intelligent-Tiering with archive access tiers."""
    tiering_config = {
        "Id": config_id,
        "Status": "Enabled",
        "Tierings": [
            {"Days": archive_days, "AccessTier": "ARCHIVE_ACCESS"},
            {"Days": deep_archive_days, "AccessTier": "DEEP_ARCHIVE_ACCESS"},
        ],
    }

    # Apply prefix filter if specified
    if filter_prefix:
        tiering_config["Filter"] = {"Prefix": filter_prefix}

    s3.put_bucket_intelligent_tiering_configuration(
        Bucket=bucket_name,
        Id=config_id,
        IntelligentTieringConfiguration=tiering_config,
    )
    print(f"Configured Intelligent-Tiering on {bucket_name}: "
          f"Archive={archive_days}d, DeepArchive={deep_archive_days}d")

# Configure for each ML data bucket
for bucket_cfg in BUCKET_NAMES:
    bucket_name = f"{PROJECT_NAME}-{bucket_cfg['name']}-{ENV}"
    configure_intelligent_tiering(
        bucket_name=bucket_name,
        config_id=f"{PROJECT_NAME}-it-config-{ENV}",
        archive_days=INTELLIGENT_TIERING_CONFIG.get("archive_access_days", 90),
        deep_archive_days=INTELLIGENT_TIERING_CONFIG.get("deep_archive_access_days", 180),
        filter_prefix=INTELLIGENT_TIERING_CONFIG.get("filter_prefix"),
    )
```

**Configure S3 Lifecycle rules for ML data tiering:**
```python
def configure_lifecycle_rules(bucket_name, lifecycle_rules):
    """Configure S3 Lifecycle rules for storage class transitions."""
    rules = []

    for rule_cfg in lifecycle_rules:
        rule = {
            "ID": rule_cfg["id"],
            "Filter": {"Prefix": rule_cfg.get("prefix", "")},
            "Status": "Enabled",
            "Transitions": [
                {"Days": t["days"], "StorageClass": t["storage_class"]}
                for t in rule_cfg.get("transitions", [])
            ],
        }

        # Noncurrent version transitions (for versioned buckets)
        if rule_cfg.get("noncurrent_transitions"):
            rule["NoncurrentVersionTransitions"] = [
                {"NoncurrentDays": t["days"], "StorageClass": t["storage_class"]}
                for t in rule_cfg["noncurrent_transitions"]
            ]

        # Expiration
        if rule_cfg.get("expiration_days"):
            rule["Expiration"] = {"Days": rule_cfg["expiration_days"]}

        rules.append(rule)

    # Default rule: abort incomplete multipart uploads after 7 days
    rules.append({
        "ID": f"{PROJECT_NAME}-abort-mpu-{ENV}",
        "Filter": {"Prefix": ""},
        "Status": "Enabled",
        "AbortIncompleteMultipartUpload": {"DaysAfterInitiation": 7},
    })

    s3.put_bucket_lifecycle_configuration(
        Bucket=bucket_name,
        LifecycleConfiguration={"Rules": rules},
    )
    print(f"Configured {len(rules)} lifecycle rules on {bucket_name}")

# Example: Standard → IT (30d) → Glacier IR (90d) → Deep Archive (365d)
configure_lifecycle_rules(
    bucket_name=f"{PROJECT_NAME}-ml-training-data-{ENV}",
    lifecycle_rules=LIFECYCLE_RULES,
)
```

**Create S3 Batch Operations copy job:**
```python
import uuid

s3control = boto3.client("s3control", region_name=AWS_REGION)

def create_batch_copy_job(source_manifest_arn, source_manifest_etag,
                          target_bucket_arn, storage_class="INTELLIGENT_TIERING",
                          kms_key_id=None):
    """Create S3 Batch Operations job to copy datasets across buckets or storage classes."""
    operation = {
        "S3PutObjectCopy": {
            "TargetResource": target_bucket_arn,
            "StorageClass": storage_class,
        }
    }
    if kms_key_id:
        operation["S3PutObjectCopy"]["SSEAwsKmsKeyId"] = kms_key_id

    response = s3control.create_job(
        AccountId=AWS_ACCOUNT_ID,
        ConfirmationRequired=False,
        Operation=operation,
        Report={
            "Bucket": f"arn:aws:s3:::{PROJECT_NAME}-ml-training-data-{ENV}",
            "Format": "Report_CSV_20180820",
            "Enabled": True,
            "Prefix": "batch-reports/copy/",
            "ReportScope": "AllTasks",
        },
        ClientRequestToken=str(uuid.uuid4()),
        Manifest={
            "Spec": {
                "Format": "S3BatchOperations_CSV_20180820",
                "Fields": ["Bucket", "Key"],
            },
            "Location": {
                "ObjectArn": source_manifest_arn,
                "ETag": source_manifest_etag,
            },
        },
        Priority=10,
        RoleArn=S3_BATCH_ROLE_ARN,
        Tags=[
            {"Key": "Project", "Value": PROJECT_NAME},
            {"Key": "Environment", "Value": ENV},
        ],
    )
    print(f"Created Batch copy job: {response['JobId']}")
    return response["JobId"]
```

**Create S3 Batch Operations tagging job:**
```python
def create_batch_tag_job(source_manifest_arn, source_manifest_etag, tags):
    """Create S3 Batch Operations job to bulk-tag ML datasets."""
    response = s3control.create_job(
        AccountId=AWS_ACCOUNT_ID,
        ConfirmationRequired=False,
        Operation={
            "S3PutObjectTagging": {
                "TagSet": [{"Key": k, "Value": v} for k, v in tags.items()],
            },
        },
        Report={
            "Bucket": f"arn:aws:s3:::{PROJECT_NAME}-ml-training-data-{ENV}",
            "Format": "Report_CSV_20180820",
            "Enabled": True,
            "Prefix": "batch-reports/tag/",
            "ReportScope": "AllTasks",
        },
        ClientRequestToken=str(uuid.uuid4()),
        Manifest={
            "Spec": {
                "Format": "S3BatchOperations_CSV_20180820",
                "Fields": ["Bucket", "Key"],
            },
            "Location": {
                "ObjectArn": source_manifest_arn,
                "ETag": source_manifest_etag,
            },
        },
        Priority=10,
        RoleArn=S3_BATCH_ROLE_ARN,
        Tags=[
            {"Key": "Project", "Value": PROJECT_NAME},
            {"Key": "Environment", "Value": ENV},
        ],
    )
    print(f"Created Batch tag job: {response['JobId']}")
    return response["JobId"]

# Example: Tag all training datasets for cost allocation
create_batch_tag_job(
    source_manifest_arn="arn:aws:s3:::my-bucket/manifests/training-data.csv",
    source_manifest_etag="abc123",
    tags={
        "Project": PROJECT_NAME,
        "Environment": ENV,
        "DataType": "training",
        "CostCenter": "ml-platform",
    },
)
```

**Create S3 Batch Operations Glacier restore job:**
```python
def create_batch_restore_job(source_manifest_arn, source_manifest_etag,
                              expiration_days=7, restore_tier="BULK"):
    """Create S3 Batch Operations job to bulk-restore archived ML datasets."""
    response = s3control.create_job(
        AccountId=AWS_ACCOUNT_ID,
        ConfirmationRequired=False,
        Operation={
            "S3InitiateRestoreObject": {
                "ExpirationInDays": expiration_days,
                "GlacierJobTier": restore_tier,  # BULK | STANDARD
            },
        },
        Report={
            "Bucket": f"arn:aws:s3:::{PROJECT_NAME}-ml-training-data-{ENV}",
            "Format": "Report_CSV_20180820",
            "Enabled": True,
            "Prefix": "batch-reports/restore/",
            "ReportScope": "AllTasks",
        },
        ClientRequestToken=str(uuid.uuid4()),
        Manifest={
            "Spec": {
                "Format": "S3BatchOperations_CSV_20180820",
                "Fields": ["Bucket", "Key"],
            },
            "Location": {
                "ObjectArn": source_manifest_arn,
                "ETag": source_manifest_etag,
            },
        },
        Priority=20,  # Higher priority for restore jobs
        RoleArn=S3_BATCH_ROLE_ARN,
        Tags=[
            {"Key": "Project", "Value": PROJECT_NAME},
            {"Key": "Environment", "Value": ENV},
        ],
    )
    print(f"Created Batch restore job: {response['JobId']} "
          f"(tier={restore_tier}, expiry={expiration_days}d)")
    return response["JobId"]
```

**Create S3 Access Points per team/project:**
```python
import json

def create_access_point(ap_config, bucket_name):
    """Create S3 Access Point with per-team access isolation."""
    ap_name = f"{PROJECT_NAME}-{ap_config['name']}-{ENV}"

    params = {
        "AccountId": AWS_ACCOUNT_ID,
        "Name": ap_name,
        "Bucket": bucket_name,
        "PublicAccessBlockConfiguration": {
            "BlockPublicAcls": True,
            "IgnorePublicAcls": True,
            "BlockPublicPolicy": True,
            "RestrictPublicBuckets": True,
        },
    }

    # VPC-restricted access point
    if ap_config.get("vpc_id"):
        params["VpcConfiguration"] = {"VpcId": ap_config["vpc_id"]}

    response = s3control.create_access_point(**params)
    print(f"Created Access Point: {ap_name} → {bucket_name}")

    # Attach access point policy
    policy = {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "AllowTeamAccess",
                "Effect": "Allow",
                "Principal": {"AWS": ap_config["policy_principal"]},
                "Action": ["s3:GetObject", "s3:PutObject", "s3:ListBucket"],
                "Resource": [
                    f"arn:aws:s3:{AWS_REGION}:{AWS_ACCOUNT_ID}:accesspoint/{ap_name}/object/{ap_config['prefix_restriction']}*",
                    f"arn:aws:s3:{AWS_REGION}:{AWS_ACCOUNT_ID}:accesspoint/{ap_name}",
                ],
                "Condition": {
                    "StringLike": {
                        "s3:prefix": [f"{ap_config['prefix_restriction']}*"],
                    }
                },
            },
        ],
    }

    s3control.put_access_point_policy(
        AccountId=AWS_ACCOUNT_ID,
        Name=ap_name,
        Policy=json.dumps(policy),
    )
    print(f"Attached policy to Access Point: {ap_name}")
    return response

# Create access points for each team
if ACCESS_POINT_CONFIGS:
    for ap_cfg in ACCESS_POINT_CONFIGS:
        bucket_name = f"{PROJECT_NAME}-{ap_cfg['bucket']}-{ENV}"
        create_access_point(ap_cfg, bucket_name)
```

**Configure S3 Inventory for ML dataset tracking:**
```python
def configure_inventory(source_bucket, destination_bucket, config_id):
    """Configure S3 Inventory for tracking dataset sizes and storage classes."""
    s3.put_bucket_inventory_configuration(
        Bucket=source_bucket,
        Id=config_id,
        InventoryConfiguration={
            "Id": config_id,
            "Destination": {
                "S3BucketDestination": {
                    "AccountId": AWS_ACCOUNT_ID,
                    "Bucket": f"arn:aws:s3:::{destination_bucket}",
                    "Format": "CSV",
                    "Prefix": f"inventory/{source_bucket}/",
                },
            },
            "Schedule": {"Frequency": "Weekly"},
            "IncludedObjectVersions": "Current",
            "OptionalFields": [
                "Size",
                "LastModifiedDate",
                "StorageClass",
                "EncryptionStatus",
                "IntelligentTieringAccessTier",
                "ETag",
            ],
            "IsEnabled": True,
        },
    )
    print(f"Configured Inventory on {source_bucket} → {destination_bucket}")

# Configure inventory for each ML data bucket
inventory_dest = INVENTORY_DESTINATION_BUCKET or f"{PROJECT_NAME}-{BUCKET_NAMES[0]['name']}-{ENV}"
for bucket_cfg in BUCKET_NAMES:
    bucket_name = f"{PROJECT_NAME}-{bucket_cfg['name']}-{ENV}"
    configure_inventory(
        source_bucket=bucket_name,
        destination_bucket=inventory_dest,
        config_id=f"{PROJECT_NAME}-inventory-{ENV}",
    )
```

**Restore objects from Glacier with restore-and-wait pattern:**
```python
def restore_object(bucket_name, key, days=7, tier="Bulk"):
    """Restore a single object from Glacier storage class."""
    try:
        s3.restore_object(
            Bucket=bucket_name,
            Key=key,
            RestoreRequest={
                "Days": days,
                "GlacierJobParameters": {"Tier": tier},
            },
        )
        print(f"Restore initiated: s3://{bucket_name}/{key} (tier={tier}, days={days})")
    except s3.exceptions.ClientError as e:
        if "RestoreAlreadyInProgress" in str(e):
            print(f"Restore already in progress: s3://{bucket_name}/{key}")
        else:
            raise

def batch_restore_objects(bucket_name, prefix, days=7, tier="Bulk"):
    """Restore all Glacier objects under a prefix."""
    paginator = s3.get_paginator("list_objects_v2")
    restored = 0
    for page in paginator.paginate(Bucket=bucket_name, Prefix=prefix):
        for obj in page.get("Contents", []):
            # Check if object is in a Glacier storage class
            head = s3.head_object(Bucket=bucket_name, Key=obj["Key"])
            if head.get("StorageClass") in ("GLACIER", "DEEP_ARCHIVE", "GLACIER_IR"):
                restore_object(bucket_name, obj["Key"], days=days, tier=tier)
                restored += 1
    print(f"Initiated restore for {restored} objects under s3://{bucket_name}/{prefix}")
```

**EventBridge rule for Glacier restore completion notification:**
```python
events = boto3.client("events", region_name=AWS_REGION)
sns = boto3.client("sns", region_name=AWS_REGION)

def setup_restore_completion_notification(bucket_name, notification_email=None):
    """Create EventBridge rule for S3 Glacier restore completion events."""
    rule_name = f"{PROJECT_NAME}-s3-restore-complete-{ENV}"

    # Create EventBridge rule matching S3 Object Restore Completed events
    events.put_rule(
        Name=rule_name,
        EventPattern=json.dumps({
            "source": ["aws.s3"],
            "detail-type": ["Object Restore Completed"],
            "detail": {
                "bucket": {"name": [bucket_name]},
            },
        }),
        State="ENABLED",
        Description=f"Notify on Glacier restore completion for {bucket_name}",
        Tags=[
            {"Key": "Project", "Value": PROJECT_NAME},
            {"Key": "Environment", "Value": ENV},
        ],
    )

    # Create SNS topic for notifications
    topic_name = f"{PROJECT_NAME}-s3-restore-notify-{ENV}"
    topic_response = sns.create_topic(
        Name=topic_name,
        Tags=[
            {"Key": "Project", "Value": PROJECT_NAME},
            {"Key": "Environment", "Value": ENV},
        ],
    )
    topic_arn = topic_response["TopicArn"]

    # Subscribe email if provided
    if notification_email:
        sns.subscribe(
            TopicArn=topic_arn,
            Protocol="email",
            Endpoint=notification_email,
        )

    # Add SNS as EventBridge target
    events.put_targets(
        Rule=rule_name,
        Targets=[
            {
                "Id": "restore-notification",
                "Arn": topic_arn,
                "InputTransformer": {
                    "InputPathsMap": {
                        "bucket": "$.detail.bucket.name",
                        "key": "$.detail.object.key",
                        "time": "$.time",
                    },
                    "InputTemplate": '"S3 Glacier restore completed: s3://<bucket>/<key> at <time>"',
                },
            },
        ],
    )
    print(f"EventBridge rule created: {rule_name} → {topic_arn}")
    return rule_name, topic_arn
```

**Athena DDL for S3 Inventory analysis:**
```sql
-- Create Athena table for S3 Inventory reports
CREATE EXTERNAL TABLE IF NOT EXISTS {PROJECT_NAME}_inventory_{ENV} (
    bucket STRING,
    key STRING,
    size BIGINT,
    last_modified_date TIMESTAMP,
    storage_class STRING,
    encryption_status STRING,
    intelligent_tiering_access_tier STRING,
    etag STRING
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
STORED AS TEXTFILE
LOCATION 's3://{INVENTORY_BUCKET}/inventory/{SOURCE_BUCKET}/{PROJECT_NAME}-inventory-{ENV}/data/';

-- Storage class distribution
SELECT storage_class, COUNT(*) AS object_count,
       SUM(size) / 1024 / 1024 / 1024 AS total_size_gb
FROM {PROJECT_NAME}_inventory_{ENV}
GROUP BY storage_class
ORDER BY total_size_gb DESC;

-- Stale data report: objects not modified in 365+ days
SELECT key, size / 1024 / 1024 AS size_mb, storage_class, last_modified_date
FROM {PROJECT_NAME}_inventory_{ENV}
WHERE last_modified_date < DATE_ADD('day', -365, CURRENT_TIMESTAMP)
ORDER BY size DESC
LIMIT 100;

-- Intelligent-Tiering tier distribution
SELECT intelligent_tiering_access_tier, COUNT(*) AS object_count,
       SUM(size) / 1024 / 1024 / 1024 AS total_size_gb
FROM {PROJECT_NAME}_inventory_{ENV}
WHERE storage_class = 'INTELLIGENT_TIERING'
GROUP BY intelligent_tiering_access_tier;
```

---

## Integration Points

- **Upstream**: `devops/04` → IAM roles for S3 access, S3 Control Batch Operations role, EventBridge rule execution role
- **Upstream**: `devops/08` → KMS customer-managed keys for S3 bucket encryption and Batch Operations copy with encryption
- **Downstream**: `data/01` → Glue ETL reads training data from S3 buckets managed by this template's lifecycle policies
- **Downstream**: `mlops/01` → SageMaker training pipeline reads datasets from S3; uses Access Points for team-scoped access
- **Downstream**: `finops/01` → Cost allocation tags applied via Batch Operations feed into FinOps cost tracking and budgets
