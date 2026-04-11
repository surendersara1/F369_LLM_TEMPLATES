<!-- Template Version: 1.0 | boto3: 1.35+ | Lake Formation: 2024+ -->

# Template Data 03 — Lake Formation ML Data Governance

## Purpose
Generate a production-ready AWS Lake Formation governed data access layer for ML teams: database and table registration with S3-backed data lakes, LF-Tag-Based Access Control (LF-TBAC) for fine-grained column-level and row-level permissions, cross-account data sharing for multi-team ML environments, Glue Data Catalog table definitions with schema evolution support, SageMaker execution role permission grants scoped to specific tables and columns, and CloudTrail audit logging for tracking data access by ML workloads.

---

## Role Definition

You are an expert AWS data governance architect and ML platform security specialist with expertise in:
- AWS Lake Formation: resource registration, permission grants, LF-TBAC tag-based access control, data filters, hybrid access mode
- Lake Formation fine-grained permissions: column-level filtering, row-level security with data cell filters, tag expressions
- Cross-account data sharing: external account grants, resource links, RAM-based sharing, catalog resource policies
- AWS Glue Data Catalog: database and table management, schema evolution, partition management, table versioning
- SageMaker integration: execution role permission scoping, training data access patterns, notebook data access
- CloudTrail audit logging: Lake Formation API event tracking, data access audit trails, compliance reporting
- IAM and Lake Formation permission model: data lake administrator setup, permission delegation, hybrid IAM and Lake Formation mode
- Data classification: sensitivity tagging, team-based access control, environment-based data segregation

Generate complete, production-deployable code.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

DATABASE_NAME:          [REQUIRED - name suffix for the Glue Data Catalog database]
                         Example: "ml-features"
                         Full name becomes: {PROJECT_NAME}_{DATABASE_NAME}_{ENV}
                         (underscores used for Glue database naming)

TABLE_DEFINITIONS:      [REQUIRED - JSON list of table definitions with columns and S3 locations]
                         Example:
                         [
                           {
                             "name": "customer_features",
                             "s3_location": "s3://my-data-lake/features/customer/",
                             "columns": [
                               {"name": "customer_id", "type": "string"},
                               {"name": "age", "type": "int"},
                               {"name": "income", "type": "double"},
                               {"name": "email", "type": "string"},
                               {"name": "credit_score", "type": "int"},
                               {"name": "segment", "type": "string"}
                             ],
                             "partition_keys": [
                               {"name": "year", "type": "string"},
                               {"name": "month", "type": "string"}
                             ],
                             "input_format": "org.apache.hadoop.hive.ql.io.parquet.MapredParquetInputFormat",
                             "output_format": "org.apache.hadoop.hive.ql.io.parquet.MapredParquetOutputFormat",
                             "serde": "org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe"
                           }
                         ]

LF_TAGS:                [REQUIRED - JSON list of LF-Tag definitions for access control]
                         Example:
                         [
                           {"key": "sensitivity", "values": ["public", "internal", "confidential", "restricted"]},
                           {"key": "team", "values": ["ml-research", "ml-engineering", "data-science", "analytics"]},
                           {"key": "data_domain", "values": ["customer", "transaction", "product", "clickstream"]}
                         ]

CROSS_ACCOUNT_IDS:      [OPTIONAL: none]
                         Comma-separated list of AWS account IDs for cross-account data sharing.
                         Example: "111111111111,222222222222"

SAGEMAKER_ROLE_ARN:     [OPTIONAL - SageMaker execution role ARN to grant data access]
                         Example: "arn:aws:iam::123456789012:role/SageMakerExecutionRole"
                         If provided, grants SELECT on specified tables/columns for ML training.

AUDIT_ENABLED:          [OPTIONAL: true]
                         Enable CloudTrail audit logging for Lake Formation data access events.

DATA_LAKE_ADMIN_ARN:    [OPTIONAL - IAM role/user ARN to register as Lake Formation administrator]
                         Example: "arn:aws:iam::123456789012:role/DataLakeAdmin"
                         If not provided, uses the current caller identity.

S3_REGISTERED_LOCATIONS: [OPTIONAL - comma-separated S3 paths to register with Lake Formation]
                         Example: "s3://my-data-lake/features/,s3://my-data-lake/raw/"
                         If not provided, S3 locations are extracted from TABLE_DEFINITIONS.

ROW_FILTER_EXPRESSIONS: [OPTIONAL - JSON mapping table names to row filter SQL expressions]
                         Example:
                         {
                           "customer_features": "segment IN ('enterprise', 'premium')",
                           "transaction_data": "amount > 0 AND status = 'completed'"
                         }

COLUMN_EXCLUSIONS:      [OPTIONAL - JSON mapping table names to excluded column lists]
                         Example:
                         {
                           "customer_features": ["email", "credit_score"]
                         }
                         These columns are excluded from grants to non-privileged roles.
```

---

## Task

Generate complete Lake Formation ML data governance infrastructure:

```
{PROJECT_NAME}-lake-formation/
├── catalog/
│   ├── create_database.py             # Create Glue Data Catalog database
│   ├── create_tables.py               # Create tables with schema and partitions
│   └── schema_evolution.py            # Handle schema evolution for ML datasets
├── registration/
│   ├── register_resources.py          # Register S3 locations with Lake Formation
│   └── set_data_lake_admin.py         # Configure Lake Formation administrator
├── permissions/
│   ├── create_lf_tags.py              # Create LF-Tags for access control
│   ├── assign_lf_tags.py             # Assign LF-Tags to databases, tables, columns
│   ├── grant_tag_permissions.py       # Grant LF-TBAC permissions to principals
│   ├── grant_named_permissions.py     # Grant named resource permissions (column/row level)
│   ├── grant_sagemaker_access.py      # Grant scoped access for SageMaker execution roles
│   └── data_filters.py               # Create data cell filters for row/column filtering
├── cross_account/
│   ├── grant_cross_account.py         # Grant cross-account permissions
│   └── create_resource_links.py       # Create resource links in consumer accounts
├── audit/
│   ├── configure_audit_logging.py     # Configure CloudTrail for Lake Formation audit
│   └── audit_queries.py              # Athena queries for data access audit reports
├── infrastructure/
│   ├── config.py                      # Central configuration
│   └── lf_helpers.py                  # Lake Formation helper utilities
├── run_setup.py                       # CLI orchestrator
└── requirements.txt
```


**config.py**: Central configuration dataclass with all parameters. Load from environment variables or CLI args. Validate required fields. Construct resource names using `{PROJECT_NAME}-{component}-{ENV}` convention. Parse TABLE_DEFINITIONS, LF_TAGS, ROW_FILTER_EXPRESSIONS, and COLUMN_EXCLUSIONS from JSON strings.

**create_database.py**: Create Glue Data Catalog database using `glue.create_database()`:
- Set `Name` to `{PROJECT_NAME}_{DATABASE_NAME}_{ENV}` (underscores for Glue naming)
- Set `LocationUri` to the common S3 prefix for the data lake
- Set `Description` with project, environment, and purpose metadata
- Tag with Project, Environment, and data_domain LF-Tag values

**create_tables.py**: Create Glue Data Catalog tables from TABLE_DEFINITIONS:
- Iterate TABLE_DEFINITIONS and call `glue.create_table()` for each
- Set `StorageDescriptor` with columns, S3 location, input/output format, SerDe info
- Set `PartitionKeys` from table definition
- Set `TableType` to `EXTERNAL_TABLE`
- Set `Parameters` with `classification=parquet`, `compressionType=snappy`
- Enable table versioning for schema evolution tracking

**schema_evolution.py**: Handle schema evolution for ML datasets:
- Use `glue.get_table()` to fetch current schema
- Compare current columns with new column definitions
- Use `glue.update_table()` to add new columns (additive-only schema evolution)
- Log schema changes with version tracking via `glue.get_table_versions()`
- Never remove columns — only add new ones to maintain backward compatibility

**register_resources.py**: Register S3 locations with Lake Formation:
- Extract unique S3 location prefixes from TABLE_DEFINITIONS (or use S3_REGISTERED_LOCATIONS)
- Call `lakeformation.register_resource()` for each S3 location with `UseServiceLinkedRole=True`
- Handle `AlreadyExistsException` gracefully (skip already-registered locations)
- Verify registration with `lakeformation.describe_resource()`

**set_data_lake_admin.py**: Configure Lake Formation administrator:
- Call `lakeformation.put_data_lake_settings()` to register DATA_LAKE_ADMIN_ARN as data lake administrator
- Preserve existing admins when adding new ones
- Optionally configure `CreateDatabaseDefaultPermissions` and `CreateTableDefaultPermissions` to empty lists (disable default IAMAllowedPrincipals grants for new resources)

**create_lf_tags.py**: Create LF-Tags for tag-based access control:
- Iterate LF_TAGS and call `lakeformation.create_lf_tag()` for each tag key with its allowed values
- Handle `EntityNotFoundException` and `AlreadyExistsException`
- Use `lakeformation.update_lf_tag()` to add new values to existing tags

**assign_lf_tags.py**: Assign LF-Tags to databases, tables, and columns:
- Call `lakeformation.add_lf_tags_to_resource()` to tag the database with domain and environment tags
- Call `lakeformation.add_lf_tags_to_resource()` to tag each table with sensitivity and team tags
- Optionally tag individual columns with sensitivity levels (e.g., `email` → `sensitivity=restricted`)
- Use `Resource` parameter with `Database`, `Table`, or `TableWithColumns` resource type

**grant_tag_permissions.py**: Grant LF-TBAC permissions to principals:
- Call `lakeformation.grant_permissions()` with `Resource` containing `LFTagPolicy` for tag-based grants
- Grant `SELECT`, `DESCRIBE` on tables matching tag expressions (e.g., `team=ml-research AND sensitivity IN (public, internal)`)
- Grant `DESCRIBE` on database for discovery
- Support `PermissionsWithGrantOption` for delegated administration

**grant_named_permissions.py**: Grant named resource permissions with column and row filtering:
- Call `lakeformation.grant_permissions()` with `Resource` containing `TableWithColumns` for column-level grants
- Use `ColumnWildcard` with `ExcludedColumnNames` from COLUMN_EXCLUSIONS to exclude sensitive columns
- Create data cell filters using `lakeformation.create_data_cells_filter()` for row-level filtering with ROW_FILTER_EXPRESSIONS

**grant_sagemaker_access.py**: Grant scoped access for SageMaker execution roles:
- Call `lakeformation.grant_permissions()` with `Principal` set to SAGEMAKER_ROLE_ARN
- Grant `SELECT` on specific tables and columns required for ML training
- Apply column exclusions to prevent SageMaker from accessing PII columns
- Apply row filters to limit training data scope (e.g., only completed transactions)
- Grant `DESCRIBE` on database and tables for schema discovery

**data_filters.py**: Create data cell filters for row and column filtering:
- Call `lakeformation.create_data_cells_filter()` for each table with row filter expressions
- Set `RowFilter.FilterExpression` from ROW_FILTER_EXPRESSIONS
- Set `ColumnWildcard` or explicit `ColumnNames` for column-level filtering
- Name filters using `{PROJECT_NAME}-{table_name}-{filter_purpose}-{ENV}` convention

**grant_cross_account.py**: Grant cross-account permissions:
- Iterate CROSS_ACCOUNT_IDS and call `lakeformation.grant_permissions()` for each account
- Set `Principal` with `DataLakePrincipalIdentifier` to the external account ID
- Grant `SELECT`, `DESCRIBE` on shared tables
- Use `PermissionsWithGrantOption` to allow consumer accounts to re-grant to their own principals
- Optionally use `lakeformation.create_lake_formation_opt_in()` for the external account

**create_resource_links.py**: Create resource links in consumer accounts (documentation/example):
- Document the steps consumer accounts must take to create resource links
- Use `glue.create_table()` with `TargetTable` pointing to the shared table in the producer account
- Include example code for consumer account setup with `CatalogId` of the producer account

**configure_audit_logging.py**: Configure CloudTrail for Lake Formation audit:
- Create or update CloudTrail trail to capture Lake Formation data events
- Configure `PutEventSelectors` with `DataResources` for `AWS::Glue::Table` events
- Enable `ReadWriteType=All` to capture both read and write data access
- Configure S3 bucket for audit log storage with lifecycle rules

**audit_queries.py**: Athena queries for data access audit reports:
- Create Athena named queries for common audit patterns:
  - Data access by principal (who accessed what tables)
  - Data access by table (which principals accessed a table)
  - Cross-account access audit (external account data access)
  - Failed permission attempts (access denied events)
  - Data access frequency by time period
- Create Athena table DDL for CloudTrail logs

**run_setup.py**: CLI orchestrator that runs setup steps in order:
1. Set Lake Formation data lake administrator
2. Create Glue Data Catalog database
3. Create tables from TABLE_DEFINITIONS
4. Register S3 locations with Lake Formation
5. Create LF-Tags
6. Assign LF-Tags to database, tables, and columns
7. Grant LF-TBAC permissions
8. Grant SageMaker execution role access (if SAGEMAKER_ROLE_ARN provided)
9. Grant cross-account permissions (if CROSS_ACCOUNT_IDS provided)
10. Configure audit logging (if AUDIT_ENABLED)
11. Print summary with database name, table count, tag count, and granted principals

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**Lake Formation Setup:** Register the caller or DATA_LAKE_ADMIN_ARN as a data lake administrator before granting any permissions. Disable default `IAMAllowedPrincipals` grants on new databases and tables by setting `CreateDatabaseDefaultPermissions` and `CreateTableDefaultPermissions` to empty lists in `put_data_lake_settings()`. This ensures all access goes through Lake Formation permissions, not IAM-only policies.

**Resource Registration:** Register all S3 locations with Lake Formation using `register_resource()` before granting permissions on tables at those locations. Use `UseServiceLinkedRole=True` to let Lake Formation manage the service-linked role for S3 access. If using a custom role, ensure it has `s3:GetObject`, `s3:PutObject`, and `s3:ListBucket` on the registered locations.

**LF-TBAC:** Use tag-based access control as the primary permission model for scalable governance. Define tags along three dimensions: `sensitivity` (public/internal/confidential/restricted), `team` (ml-research/ml-engineering/data-science/analytics), and `data_domain` (customer/transaction/product/clickstream). Tag expressions in grants support AND/OR logic for flexible access policies. LF-TBAC scales better than named resource grants when managing many tables and principals.

**Column-Level Permissions:** Use `ColumnWildcard` with `ExcludedColumnNames` to grant access to all columns except sensitive ones (e.g., email, SSN, credit_score). This is more maintainable than listing included columns — new columns are automatically accessible. For explicit column grants, use `ColumnNames` in the `TableWithColumns` resource.

**Row-Level Security:** Use data cell filters with `RowFilter.FilterExpression` for row-level security. Filter expressions use SQL WHERE clause syntax (e.g., `segment IN ('enterprise', 'premium')`). Combine row and column filters in a single data cell filter for cell-level security. Data cell filters are applied transparently — queries return only permitted rows without modification.

**Cross-Account Sharing:** Grant permissions to external account IDs (not specific roles) using `lakeformation.grant_permissions()`. Consumer accounts must accept the RAM share (if using RAM) and create resource links to access shared tables. Use `PermissionsWithGrantOption` to allow consumer account admins to re-grant to their own principals. Cross-account sharing works with both named resource and LF-TBAC methods.

**Schema Evolution:** Only add columns — never remove or rename existing columns. Use `glue.update_table()` to add new columns to existing tables. Track schema versions using `glue.get_table_versions()`. Downstream consumers (SageMaker, Athena) handle additive schema changes gracefully. Set `Parameters` with `EXTERNAL=TRUE` for external tables.

**Audit Logging:** Enable CloudTrail data events for `AWS::Glue::Table` to capture all data access through Lake Formation. Store audit logs in a dedicated S3 bucket with lifecycle rules (transition to Glacier after 90 days, delete after 365 days). Use Athena to query audit logs for compliance reporting. Monitor for failed access attempts as potential security indicators.

**Security:** Lake Formation permissions are additive — a principal's effective permissions are the union of all grants. Use `REVOKE` to remove specific permissions. Never grant `ALL` or `SUPER` permissions to non-admin principals. Ensure SageMaker execution roles have only the minimum permissions needed for training data access. Use VPC endpoints for Lake Formation and Glue API access in production.

**Naming:** All resources follow `{PROJECT_NAME}-{component}-{ENV}` convention:
- Database: `{PROJECT_NAME}_{DATABASE_NAME}_{ENV}` (underscores for Glue)
- LF-Tags: `sensitivity`, `team`, `data_domain` (global, not environment-scoped)
- Data cell filters: `{PROJECT_NAME}-{table_name}-{filter_purpose}-{ENV}`
- CloudTrail trail: `{PROJECT_NAME}-lf-audit-{ENV}`
- Audit S3 bucket: `{PROJECT_NAME}-lf-audit-logs-{ENV}`

---

## Code Scaffolding Hints


**Register S3 locations with Lake Formation:**
```python
import boto3

lf = boto3.client("lakeformation", region_name=AWS_REGION)

def register_s3_location(s3_path, role_arn=None):
    """Register an S3 location with Lake Formation for governed access."""
    try:
        params = {
            "ResourceArn": s3_path,
            "UseServiceLinkedRole": True,
        }
        if role_arn:
            params["UseServiceLinkedRole"] = False
            params["RoleArn"] = role_arn

        lf.register_resource(**params)
        print(f"Registered: {s3_path}")
    except lf.exceptions.AlreadyExistsException:
        print(f"Already registered: {s3_path}")

# Register each S3 location from table definitions
for table_def in TABLE_DEFINITIONS:
    register_s3_location(table_def["s3_location"])

# Verify registration
for table_def in TABLE_DEFINITIONS:
    response = lf.describe_resource(ResourceArn=table_def["s3_location"])
    print(f"Resource: {response['ResourceInfo']['ResourceArn']}, "
          f"Role: {response['ResourceInfo'].get('RoleArn', 'ServiceLinkedRole')}")
```

**Create LF-Tags for tag-based access control:**
```python
def create_lf_tags(lf_tags):
    """Create LF-Tags for data classification and access control."""
    for tag in lf_tags:
        try:
            lf.create_lf_tag(
                CatalogId=AWS_ACCOUNT_ID,
                TagKey=tag["key"],
                TagValues=tag["values"],
            )
            print(f"Created LF-Tag: {tag['key']} = {tag['values']}")
        except lf.exceptions.EntityNotFoundException:
            raise
        except Exception as e:
            if "already exists" in str(e).lower():
                # Update existing tag with new values
                lf.update_lf_tag(
                    CatalogId=AWS_ACCOUNT_ID,
                    TagKey=tag["key"],
                    TagValuesToAdd=[
                        v for v in tag["values"]
                    ],
                )
                print(f"Updated LF-Tag: {tag['key']}")
            else:
                raise

create_lf_tags(LF_TAGS)
# Example LF_TAGS:
# [
#   {"key": "sensitivity", "values": ["public", "internal", "confidential", "restricted"]},
#   {"key": "team", "values": ["ml-research", "ml-engineering", "data-science", "analytics"]},
#   {"key": "data_domain", "values": ["customer", "transaction", "product", "clickstream"]}
# ]
```

**Assign LF-Tags to databases, tables, and columns:**
```python
def add_lf_tags_to_resource(resource, lf_tags_to_assign):
    """Assign LF-Tags to a Lake Formation resource."""
    lf.add_lf_tags_to_resource(
        CatalogId=AWS_ACCOUNT_ID,
        Resource=resource,
        LFTags=[
            {"CatalogId": AWS_ACCOUNT_ID, "TagKey": t["key"], "TagValues": t["values"]}
            for t in lf_tags_to_assign
        ],
    )

database_name = f"{PROJECT_NAME}_{DATABASE_NAME}_{ENV}"

# Tag the database with domain and environment tags
add_lf_tags_to_resource(
    resource={"Database": {"CatalogId": AWS_ACCOUNT_ID, "Name": database_name}},
    lf_tags_to_assign=[
        {"key": "data_domain", "values": ["customer"]},
    ],
)

# Tag a table with sensitivity and team tags
add_lf_tags_to_resource(
    resource={
        "Table": {
            "CatalogId": AWS_ACCOUNT_ID,
            "DatabaseName": database_name,
            "Name": "customer_features",
        }
    },
    lf_tags_to_assign=[
        {"key": "sensitivity", "values": ["confidential"]},
        {"key": "team", "values": ["ml-research", "ml-engineering"]},
    ],
)

# Tag specific columns with sensitivity (e.g., PII columns)
add_lf_tags_to_resource(
    resource={
        "TableWithColumns": {
            "CatalogId": AWS_ACCOUNT_ID,
            "DatabaseName": database_name,
            "Name": "customer_features",
            "ColumnNames": ["email", "credit_score"],
        }
    },
    lf_tags_to_assign=[
        {"key": "sensitivity", "values": ["restricted"]},
    ],
)
```

**Grant LF-TBAC permissions to principals:**
```python
def grant_tag_based_permissions(principal_arn, tag_expression, permissions, database_name):
    """Grant permissions using LF-Tag-Based Access Control (LF-TBAC)."""
    # Grant table-level permissions via tag expression
    lf.grant_permissions(
        CatalogId=AWS_ACCOUNT_ID,
        Principal={"DataLakePrincipalIdentifier": principal_arn},
        Resource={
            "LFTagPolicy": {
                "CatalogId": AWS_ACCOUNT_ID,
                "ResourceType": "TABLE",
                "Expression": tag_expression,
            }
        },
        Permissions=permissions,
    )

    # Grant database DESCRIBE for discovery
    lf.grant_permissions(
        CatalogId=AWS_ACCOUNT_ID,
        Principal={"DataLakePrincipalIdentifier": principal_arn},
        Resource={
            "Database": {
                "CatalogId": AWS_ACCOUNT_ID,
                "Name": database_name,
            }
        },
        Permissions=["DESCRIBE"],
    )

# Example: ML Research team can SELECT on public + internal data in customer domain
grant_tag_based_permissions(
    principal_arn="arn:aws:iam::123456789012:role/MLResearchRole",
    tag_expression=[
        {"TagKey": "sensitivity", "TagValues": ["public", "internal"]},
        {"TagKey": "team", "TagValues": ["ml-research"]},
        {"TagKey": "data_domain", "TagValues": ["customer", "transaction"]},
    ],
    permissions=["SELECT", "DESCRIBE"],
    database_name=f"{PROJECT_NAME}_{DATABASE_NAME}_{ENV}",
)
```

**Grant named resource permissions with column-level filtering:**
```python
def grant_column_filtered_permissions(principal_arn, database_name, table_name,
                                       excluded_columns=None, included_columns=None):
    """Grant SELECT with column-level filtering using named resource method."""
    if excluded_columns:
        # Grant all columns EXCEPT excluded ones
        resource = {
            "TableWithColumns": {
                "CatalogId": AWS_ACCOUNT_ID,
                "DatabaseName": database_name,
                "Name": table_name,
                "ColumnWildcard": {
                    "ExcludedColumnNames": excluded_columns,
                },
            }
        }
    elif included_columns:
        # Grant only specific columns
        resource = {
            "TableWithColumns": {
                "CatalogId": AWS_ACCOUNT_ID,
                "DatabaseName": database_name,
                "Name": table_name,
                "ColumnNames": included_columns,
            }
        }
    else:
        # Grant all columns
        resource = {
            "Table": {
                "CatalogId": AWS_ACCOUNT_ID,
                "DatabaseName": database_name,
                "Name": table_name,
            }
        }

    lf.grant_permissions(
        CatalogId=AWS_ACCOUNT_ID,
        Principal={"DataLakePrincipalIdentifier": principal_arn},
        Resource=resource,
        Permissions=["SELECT"],
    )

# Example: Grant SageMaker role access excluding PII columns
grant_column_filtered_permissions(
    principal_arn=SAGEMAKER_ROLE_ARN,
    database_name=f"{PROJECT_NAME}_{DATABASE_NAME}_{ENV}",
    table_name="customer_features",
    excluded_columns=["email", "credit_score"],  # Exclude PII
)
```

**Create data cell filters for row-level security:**
```python
def create_row_filter(database_name, table_name, filter_name, row_expression,
                      column_names=None, excluded_columns=None):
    """Create a data cell filter for row-level and cell-level security."""
    table_data = {
        "TableCatalogId": AWS_ACCOUNT_ID,
        "DatabaseName": database_name,
        "TableName": table_name,
        "Name": filter_name,
        "RowFilter": {
            "FilterExpression": row_expression,
        },
    }

    # Column filtering within the data cell filter
    if excluded_columns:
        table_data["ColumnWildcard"] = {
            "ExcludedColumnNames": excluded_columns,
        }
    elif column_names:
        table_data["ColumnNames"] = column_names
    else:
        table_data["ColumnWildcard"] = {}

    lf.create_data_cells_filter(TableData=table_data)

# Example: Filter to only enterprise/premium customers, excluding PII columns
create_row_filter(
    database_name=f"{PROJECT_NAME}_{DATABASE_NAME}_{ENV}",
    table_name="customer_features",
    filter_name=f"{PROJECT_NAME}-customer-enterprise-{ENV}",
    row_expression="segment IN ('enterprise', 'premium')",
    excluded_columns=["email", "credit_score"],
)

# Grant permissions using the data cell filter
lf.grant_permissions(
    CatalogId=AWS_ACCOUNT_ID,
    Principal={"DataLakePrincipalIdentifier": SAGEMAKER_ROLE_ARN},
    Resource={
        "DataCellsFilter": {
            "TableCatalogId": AWS_ACCOUNT_ID,
            "DatabaseName": f"{PROJECT_NAME}_{DATABASE_NAME}_{ENV}",
            "TableName": "customer_features",
            "Name": f"{PROJECT_NAME}-customer-enterprise-{ENV}",
        }
    },
    Permissions=["SELECT"],
)
```

**Cross-account data sharing:**
```python
def grant_cross_account_access(external_account_id, database_name, table_name,
                                permissions=None, grant_option=False):
    """Grant cross-account access to a Lake Formation table."""
    perms = permissions or ["SELECT", "DESCRIBE"]

    lf.grant_permissions(
        CatalogId=AWS_ACCOUNT_ID,
        Principal={"DataLakePrincipalIdentifier": external_account_id},
        Resource={
            "Table": {
                "CatalogId": AWS_ACCOUNT_ID,
                "DatabaseName": database_name,
                "Name": table_name,
            }
        },
        Permissions=perms,
        PermissionsWithGrantOption=perms if grant_option else [],
    )
    print(f"Granted {perms} to account {external_account_id} on {database_name}.{table_name}")

# Grant cross-account access for each external account
if CROSS_ACCOUNT_IDS:
    for account_id in CROSS_ACCOUNT_IDS.split(","):
        account_id = account_id.strip()
        for table_def in TABLE_DEFINITIONS:
            grant_cross_account_access(
                external_account_id=account_id,
                database_name=f"{PROJECT_NAME}_{DATABASE_NAME}_{ENV}",
                table_name=table_def["name"],
                grant_option=True,  # Allow consumer to re-grant
            )
```

**Create Glue Data Catalog table with schema evolution support:**
```python
glue = boto3.client("glue", region_name=AWS_REGION)

def create_catalog_table(database_name, table_def):
    """Create a Glue Data Catalog table from a table definition."""
    glue.create_table(
        CatalogId=AWS_ACCOUNT_ID,
        DatabaseName=database_name,
        TableInput={
            "Name": table_def["name"],
            "Description": f"ML feature table: {table_def['name']} for {PROJECT_NAME}",
            "TableType": "EXTERNAL_TABLE",
            "Parameters": {
                "classification": "parquet",
                "compressionType": "snappy",
                "EXTERNAL": "TRUE",
                "Project": PROJECT_NAME,
                "Environment": ENV,
            },
            "StorageDescriptor": {
                "Columns": [
                    {"Name": col["name"], "Type": col["type"]}
                    for col in table_def["columns"]
                ],
                "Location": table_def["s3_location"],
                "InputFormat": table_def.get(
                    "input_format",
                    "org.apache.hadoop.hive.ql.io.parquet.MapredParquetInputFormat",
                ),
                "OutputFormat": table_def.get(
                    "output_format",
                    "org.apache.hadoop.hive.ql.io.parquet.MapredParquetOutputFormat",
                ),
                "SerdeInfo": {
                    "SerializationLibrary": table_def.get(
                        "serde",
                        "org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe",
                    ),
                    "Parameters": {"serialization.format": "1"},
                },
                "Compressed": True,
            },
            "PartitionKeys": [
                {"Name": pk["name"], "Type": pk["type"]}
                for pk in table_def.get("partition_keys", [])
            ],
        },
    )
    print(f"Created table: {database_name}.{table_def['name']}")

def evolve_schema(database_name, table_name, new_columns):
    """Add new columns to an existing table (additive-only schema evolution)."""
    response = glue.get_table(
        CatalogId=AWS_ACCOUNT_ID,
        DatabaseName=database_name,
        Name=table_name,
    )
    current_table = response["Table"]
    current_columns = current_table["StorageDescriptor"]["Columns"]
    current_col_names = {col["Name"] for col in current_columns}

    # Only add columns that don't already exist
    columns_to_add = [
        col for col in new_columns if col["name"] not in current_col_names
    ]

    if not columns_to_add:
        print(f"No new columns to add to {table_name}")
        return

    updated_columns = current_columns + [
        {"Name": col["name"], "Type": col["type"]} for col in columns_to_add
    ]

    # Build TableInput from current table (remove metadata fields)
    table_input = {
        "Name": current_table["Name"],
        "Description": current_table.get("Description", ""),
        "TableType": current_table.get("TableType", "EXTERNAL_TABLE"),
        "Parameters": current_table.get("Parameters", {}),
        "StorageDescriptor": {
            **current_table["StorageDescriptor"],
            "Columns": updated_columns,
        },
        "PartitionKeys": current_table.get("PartitionKeys", []),
    }

    glue.update_table(
        CatalogId=AWS_ACCOUNT_ID,
        DatabaseName=database_name,
        TableInput=table_input,
    )
    print(f"Added {len(columns_to_add)} columns to {table_name}: "
          f"{[c['name'] for c in columns_to_add]}")
```

**Configure CloudTrail audit logging for Lake Formation:**
```python
cloudtrail = boto3.client("cloudtrail", region_name=AWS_REGION)

def configure_lf_audit_logging(trail_name, s3_bucket):
    """Configure CloudTrail to capture Lake Formation data access events."""
    # Create or update trail
    try:
        cloudtrail.create_trail(
            Name=trail_name,
            S3BucketName=s3_bucket,
            IsMultiRegionTrail=False,
            EnableLogFileValidation=True,
            Tags=[
                {"Key": "Project", "Value": PROJECT_NAME},
                {"Key": "Environment", "Value": ENV},
            ],
        )
    except cloudtrail.exceptions.TrailAlreadyExistsException:
        pass

    # Enable data events for Glue tables (captures Lake Formation access)
    cloudtrail.put_event_selectors(
        TrailName=trail_name,
        AdvancedEventSelectors=[
            {
                "Name": "LakeFormationDataAccess",
                "FieldSelectors": [
                    {"Field": "eventCategory", "Equals": ["Data"]},
                    {"Field": "resources.type", "Equals": ["AWS::Glue::Table"]},
                ],
            },
        ],
    )

    # Start logging
    cloudtrail.start_logging(Name=trail_name)
    print(f"Audit logging enabled: {trail_name} → s3://{s3_bucket}/")

configure_lf_audit_logging(
    trail_name=f"{PROJECT_NAME}-lf-audit-{ENV}",
    s3_bucket=f"{PROJECT_NAME}-lf-audit-logs-{ENV}",
)
```

**Athena audit queries for data access reporting:**
```python
# Athena DDL for CloudTrail logs table
CLOUDTRAIL_TABLE_DDL = f"""
CREATE EXTERNAL TABLE IF NOT EXISTS {PROJECT_NAME}_audit_{ENV}.cloudtrail_logs (
    eventVersion STRING,
    userIdentity STRUCT<
        type: STRING,
        principalId: STRING,
        arn: STRING,
        accountId: STRING,
        invokedBy: STRING,
        accessKeyId: STRING,
        userName: STRING,
        sessionContext: STRUCT<
            attributes: STRUCT<mfaAuthenticated: STRING, creationDate: STRING>,
            sessionIssuer: STRUCT<type: STRING, principalId: STRING, arn: STRING, accountId: STRING, userName: STRING>
        >
    >,
    eventTime STRING,
    eventSource STRING,
    eventName STRING,
    awsRegion STRING,
    sourceIPAddress STRING,
    userAgent STRING,
    requestParameters STRING,
    responseElements STRING,
    resources ARRAY<STRUCT<arn: STRING, accountId: STRING, type: STRING>>,
    readOnly STRING,
    errorCode STRING,
    errorMessage STRING
)
ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'
LOCATION 's3://{PROJECT_NAME}-lf-audit-logs-{ENV}/AWSLogs/{AWS_ACCOUNT_ID}/CloudTrail/{AWS_REGION}/'
"""

# Query: Data access by principal
DATA_ACCESS_BY_PRINCIPAL = f"""
SELECT
    userIdentity.arn AS principal,
    eventName,
    JSON_EXTRACT_SCALAR(requestParameters, '$.databaseName') AS database_name,
    JSON_EXTRACT_SCALAR(requestParameters, '$.tableName') AS table_name,
    COUNT(*) AS access_count,
    MIN(eventTime) AS first_access,
    MAX(eventTime) AS last_access
FROM {PROJECT_NAME}_audit_{ENV}.cloudtrail_logs
WHERE eventSource = 'lakeformation.amazonaws.com'
    AND eventTime >= DATE_FORMAT(DATE_ADD('day', -30, NOW()), '%Y-%m-%dT%H:%i:%sZ')
GROUP BY userIdentity.arn, eventName,
    JSON_EXTRACT_SCALAR(requestParameters, '$.databaseName'),
    JSON_EXTRACT_SCALAR(requestParameters, '$.tableName')
ORDER BY access_count DESC
"""

# Query: Failed access attempts (potential security indicators)
FAILED_ACCESS_ATTEMPTS = f"""
SELECT
    userIdentity.arn AS principal,
    eventName,
    errorCode,
    errorMessage,
    eventTime,
    JSON_EXTRACT_SCALAR(requestParameters, '$.databaseName') AS database_name,
    JSON_EXTRACT_SCALAR(requestParameters, '$.tableName') AS table_name
FROM {PROJECT_NAME}_audit_{ENV}.cloudtrail_logs
WHERE eventSource = 'lakeformation.amazonaws.com'
    AND errorCode IS NOT NULL
    AND eventTime >= DATE_FORMAT(DATE_ADD('day', -7, NOW()), '%Y-%m-%dT%H:%i:%sZ')
ORDER BY eventTime DESC
"""
```

**Set Lake Formation data lake administrator and default permissions:**
```python
def configure_data_lake_settings(admin_arn):
    """Configure Lake Formation data lake settings with admin and secure defaults."""
    # Get current settings to preserve existing admins
    current = lf.get_data_lake_settings(CatalogId=AWS_ACCOUNT_ID)
    current_admins = current["DataLakeSettings"].get("DataLakeAdmins", [])

    # Add new admin if not already present
    admin_entry = {"DataLakePrincipalIdentifier": admin_arn}
    if admin_entry not in current_admins:
        current_admins.append(admin_entry)

    lf.put_data_lake_settings(
        CatalogId=AWS_ACCOUNT_ID,
        DataLakeSettings={
            "DataLakeAdmins": current_admins,
            # Disable default IAMAllowedPrincipals grants for new resources
            "CreateDatabaseDefaultPermissions": [],
            "CreateTableDefaultPermissions": [],
            "Parameters": {
                "CROSS_ACCOUNT_VERSION": "4",
            },
        },
    )
    print(f"Data lake admin configured: {admin_arn}")
    print("Default permissions disabled for new databases and tables")

configure_data_lake_settings(DATA_LAKE_ADMIN_ARN)
```

---

## Integration Points

- **Upstream**: `devops/04` → IAM roles for Lake Formation service role, data lake administrator role, and SageMaker execution role with `lakeformation:GetDataAccess` permission
- **Upstream**: `data/01` → Glue ETL jobs write feature data to S3 locations governed by Lake Formation; Glue crawlers populate the Data Catalog tables managed here
- **Downstream**: `mlops/01` → SageMaker training pipeline reads governed training data via Lake Formation permissions granted to the SageMaker execution role
- **Downstream**: `enterprise/01` → SCPs enforce Lake Formation usage by denying direct S3 access and requiring Lake Formation permission checks
- **Downstream**: `devops/05` → Config rules validate that Lake Formation is enabled on all data lake S3 locations and that no `IAMAllowedPrincipals` grants exist