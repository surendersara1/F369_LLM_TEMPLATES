<!-- Template Version: 1.0 | boto3: 1.35+ -->

# Template FinOps 01 — Cost Allocation & Tagging for ML Workloads

## Purpose
Generate a production-ready ML cost management framework: tag policy enforcement via AWS Organizations for mandatory cost allocation tags on all ML resources, AWS Budgets per ML service (SageMaker, Bedrock, Glue, Lambda) with 50%/80%/100% threshold notifications, Cost Explorer queries by service/instance type/tag for spend analysis, Cost Anomaly Detection monitors for ML-specific anomalies, a per-experiment cost tracking utility that attributes spend to individual ML experiments using tags, and a budget breach IAM restriction action that automatically limits resource creation when budgets are exceeded.

---

## Role Definition

You are an expert AWS FinOps engineer and ML cost management specialist with expertise in:
- AWS Cost Explorer: `get_cost_and_usage()`, `get_cost_forecast()`, filtering by service/tag/instance type, grouping and aggregation
- AWS Budgets: budget creation per service, threshold-based notifications (50%/80%/100%), budget actions for automated cost controls
- AWS Cost Anomaly Detection: anomaly monitors by service/tag/cost category, anomaly subscriptions with SNS/email alerts, threshold configuration
- AWS Organizations tag policies: mandatory tag enforcement, allowed tag values, compliance reporting
- Cost allocation tags: activation, propagation, tag-based cost attribution for ML experiments and teams
- IAM budget actions: automated IAM policy attachment on budget breach to restrict resource creation
- SageMaker cost patterns: training job instance hours, endpoint instance hours, processing job costs, notebook instance costs
- Bedrock cost patterns: input/output token pricing, provisioned throughput costs, model customization job costs
- CloudWatch cost metrics: custom metrics for per-experiment and per-team cost tracking
- Cost optimization: right-sizing recommendations, unused resource detection, reserved capacity analysis

Generate complete, production-deployable cost management code.

---

## Context & Inputs

```
PROJECT_NAME:               [REQUIRED]
AWS_REGION:                 [REQUIRED]
AWS_ACCOUNT_ID:             [REQUIRED]
ENV:                        [REQUIRED - dev | stage | prod]

MANDATORY_TAGS:             [REQUIRED - JSON object defining required tags and allowed values]
                            Tags that must be present on all ML resources for cost allocation.
                            Example:
                            {
                              "Project": {"enforced": true, "allowed_values": ["myai", "research", "platform"]},
                              "Environment": {"enforced": true, "allowed_values": ["dev", "stage", "prod"]},
                              "Team": {"enforced": true, "allowed_values": ["ml-research", "ml-platform", "data-eng"]},
                              "CostCenter": {"enforced": true, "allowed_values": ["CC-100", "CC-200", "CC-300"]},
                              "ExperimentId": {"enforced": false, "allowed_values": ["*"]}
                            }

BUDGET_THRESHOLDS:          [REQUIRED - JSON object with service-level monthly budget limits in USD]
                            Monthly budget limits per ML service. Notifications fire at 50%, 80%, 100%.
                            Example:
                            {
                              "SageMaker": 5000,
                              "Bedrock": 3000,
                              "Glue": 1000,
                              "Lambda": 500,
                              "S3": 500,
                              "Total": 10000
                            }

ANOMALY_THRESHOLD:          [OPTIONAL: 20 - percentage above expected spend that triggers anomaly alert]
                            Cost Anomaly Detection threshold as a percentage.
                            Example: 20 means alert when spend is 20% above expected.

COST_CENTER_MAPPING:        [OPTIONAL: none - JSON mapping cost centers to teams/projects]
                            Maps cost center codes to organizational units for chargeback.
                            Example:
                            {
                              "CC-100": {"team": "ml-research", "budget_owner": "research-lead@example.com"},
                              "CC-200": {"team": "ml-platform", "budget_owner": "platform-lead@example.com"},
                              "CC-300": {"team": "data-eng", "budget_owner": "data-lead@example.com"}
                            }

NOTIFICATION_EMAIL:         [REQUIRED - email address for budget and anomaly notifications]
                            Example: "mlops-finops@example.com"

BUDGET_ACTION_ENABLED:      [OPTIONAL: true]
                            Enable automated IAM restriction when budget exceeds 100%.
                            When true, attaches a deny policy to ML roles on budget breach.

ORGANIZATION_ID:            [OPTIONAL: none - AWS Organization ID for tag policy enforcement]
                            Required for tag policy creation via Organizations.
                            Example: "o-abc123def4"

TARGET_OU_ID:               [OPTIONAL: none - Organizational Unit ID for tag policy attachment]
                            OU where the tag policy is attached.
                            Example: "ou-abc1-23def456"
```

---

## Task

Generate complete ML cost allocation and management framework:

```
{PROJECT_NAME}-cost-allocation/
├── tagging/
│   ├── create_tag_policy.py              # Organizations tag policy for mandatory ML tags
│   ├── activate_cost_tags.py             # Activate cost allocation tags in Billing
│   └── tag_compliance_checker.py         # Scan ML resources for missing/invalid tags
├── budgets/
│   ├── create_service_budgets.py         # AWS Budgets per ML service with notifications
│   ├── create_total_budget.py            # Total ML spend budget with notifications
│   └── budget_action.py                  # IAM restriction action on budget breach
├── cost_explorer/
│   ├── query_by_service.py               # Cost Explorer queries by ML service
│   ├── query_by_instance_type.py         # Cost Explorer queries by instance type
│   ├── query_by_tag.py                   # Cost Explorer queries by cost allocation tag
│   └── cost_forecast.py                  # Cost forecasting for ML workloads
├── anomaly_detection/
│   ├── create_anomaly_monitor.py         # Cost Anomaly Detection monitors for ML
│   └── create_anomaly_subscription.py    # Anomaly alert subscriptions (SNS + email)
├── experiment_tracking/
│   ├── experiment_cost_tracker.py        # Per-experiment cost attribution utility
│   └── chargeback_report.py             # Team/cost-center chargeback report generator
├── infrastructure/
│   ├── config.py                         # Central configuration
│   └── sns_topics.py                     # SNS topics for budget and anomaly notifications
├── run_setup.py                          # CLI orchestrator
└── requirements.txt
```

**config.py**: Central configuration dataclass with all parameters. Load from environment variables or CLI args. Validate required fields. Parse MANDATORY_TAGS, BUDGET_THRESHOLDS, and COST_CENTER_MAPPING from JSON strings. Construct resource names using `{PROJECT_NAME}-{component}-{ENV}` convention.

**create_tag_policy.py**: Create Organizations tag policy for mandatory ML tags:
- Build tag policy JSON document from MANDATORY_TAGS parameter
- For each tag with `enforced: true`, create a tag policy rule that enforces the tag key and restricts to `allowed_values`
- Enforce tags on resource types: `sagemaker:*`, `bedrock:*`, `glue:*`, `lambda:*`, `s3:*`
- Call `organizations.create_policy()` with `Type="TAG_POLICY"` and the policy content
- If TARGET_OU_ID provided, call `organizations.attach_policy()` to attach to the OU
- Tag policy name: `{PROJECT_NAME}-ml-tag-policy-{ENV}`

**activate_cost_tags.py**: Activate cost allocation tags in AWS Billing:
- Call `ce.update_cost_allocation_tags_status()` (via Cost Explorer API) to activate each tag key from MANDATORY_TAGS as a cost allocation tag
- Note: cost allocation tags take up to 24 hours to appear in Cost Explorer after activation
- Verify activation status using `ce.list_cost_allocation_tags()`

**tag_compliance_checker.py**: Scan ML resources for missing or invalid tags:
- Use Resource Groups Tagging API: `resourcegroupstaggingapi.get_resources()` filtered to ML service resource types
- Check each resource against MANDATORY_TAGS requirements
- Report non-compliant resources with missing tags, invalid tag values
- Output compliance summary: total resources, compliant count, non-compliant count, per-tag compliance percentage
- Optionally publish compliance metrics to CloudWatch

**create_service_budgets.py**: Create AWS Budgets per ML service:
- For each service in BUDGET_THRESHOLDS (except "Total"), create a budget using `budgets.create_budget()`:
  - Budget name: `{PROJECT_NAME}-{service}-budget-{ENV}`
  - Budget type: `COST`
  - Budget limit: monthly amount from BUDGET_THRESHOLDS
  - Cost filter: `Service` dimension matching the AWS service name (e.g., `Amazon SageMaker`, `Amazon Bedrock`)
  - Time unit: `MONTHLY`
- Create three notification thresholds per budget:
  - 50% actual spend → SNS notification (informational)
  - 80% actual spend → SNS notification (warning)
  - 100% actual spend → SNS notification (critical) + budget action (if BUDGET_ACTION_ENABLED)
- Each notification targets the SNS topic and NOTIFICATION_EMAIL

**create_total_budget.py**: Create total ML spend budget:
- Create a single budget for total ML spend across all services
- Budget name: `{PROJECT_NAME}-total-ml-budget-{ENV}`
- Budget limit: `Total` value from BUDGET_THRESHOLDS
- Cost filter: filter to ML-related services (SageMaker, Bedrock, Glue, Lambda, S3)
- Same 50%/80%/100% notification thresholds as service budgets
- 100% threshold triggers budget action if BUDGET_ACTION_ENABLED

**budget_action.py**: IAM restriction action on budget breach:
- Create a budget action using `budgets.create_budget_action()` that triggers when budget exceeds 100%
- Action type: `APPLY_IAM_POLICY`
- The action attaches a deny policy to specified IAM roles that prevents creation of new ML resources:
  - Deny `sagemaker:CreateTrainingJob`, `sagemaker:CreateEndpoint`, `sagemaker:CreateProcessingJob`, `sagemaker:CreateNotebookInstance`
  - Deny `bedrock:CreateModelCustomizationJob`, `bedrock:CreateProvisionedModelThroughput`
  - Deny `glue:CreateJob`, `glue:StartJobRun`
- Action role: IAM role that Budgets assumes to attach the deny policy
- Include a manual override mechanism: a Lambda function that removes the deny policy for break-glass scenarios
- Policy name: `{PROJECT_NAME}-budget-breach-deny-{ENV}`

**query_by_service.py**: Cost Explorer queries by ML service:
- Query last 30 days of cost data using `ce.get_cost_and_usage()` grouped by `SERVICE` dimension
- Filter to ML services: Amazon SageMaker, Amazon Bedrock, AWS Glue, AWS Lambda, Amazon S3
- Return daily and monthly aggregated costs per service
- Include both blended and unblended cost metrics
- Output formatted table with service name, daily average, monthly total, month-over-month change

**query_by_instance_type.py**: Cost Explorer queries by instance type:
- Query SageMaker costs grouped by `INSTANCE_TYPE` dimension
- Identify top-spending instance types for training and inference
- Calculate cost-per-hour for each instance type
- Flag underutilized instance types (high cost, low usage hours)
- Output recommendations for right-sizing

**query_by_tag.py**: Cost Explorer queries by cost allocation tag:
- Query costs grouped by tag keys from MANDATORY_TAGS (Team, CostCenter, ExperimentId)
- Attribute costs to teams, cost centers, and individual experiments
- Calculate percentage of untagged spend (cost attribution gap)
- Output per-team and per-cost-center spend summaries

**cost_forecast.py**: Cost forecasting for ML workloads:
- Call `ce.get_cost_forecast()` for each ML service for the next 30/60/90 days
- Compare forecast against BUDGET_THRESHOLDS to predict budget breaches
- Output forecast summary with projected monthly spend, budget utilization percentage, and breach risk assessment

**create_anomaly_monitor.py**: Create Cost Anomaly Detection monitors:
- Create a service-level anomaly monitor using `ce.create_anomaly_monitor()`:
  - Monitor name: `{PROJECT_NAME}-ml-service-anomaly-{ENV}`
  - Monitor type: `DIMENSIONAL`
  - Monitor dimension: `SERVICE`
- Create a tag-based anomaly monitor for cost center tracking:
  - Monitor name: `{PROJECT_NAME}-ml-costcenter-anomaly-{ENV}`
  - Monitor type: `CUSTOM`
  - Monitor specification: filter by cost allocation tag `CostCenter`
- Both monitors detect spending anomalies using ML-based detection

**create_anomaly_subscription.py**: Create anomaly alert subscriptions:
- Create subscription using `ce.create_anomaly_subscription()`:
  - Subscription name: `{PROJECT_NAME}-ml-anomaly-alerts-{ENV}`
  - Monitor ARN list: both monitors from `create_anomaly_monitor.py`
  - Threshold expression: alert when anomaly impact exceeds ANOMALY_THRESHOLD percentage of expected spend
  - Subscribers: SNS topic ARN and NOTIFICATION_EMAIL
  - Frequency: `IMMEDIATE` for real-time alerts

**experiment_cost_tracker.py**: Per-experiment cost attribution utility:
- Given an `ExperimentId` tag value, query Cost Explorer for all costs attributed to that experiment
- Break down costs by resource type (training jobs, endpoints, processing jobs, storage)
- Calculate total experiment cost, cost per training hour, cost per inference request
- Compare against team budget allocation
- Output experiment cost report as JSON for dashboard consumption
- Publish per-experiment cost metrics to CloudWatch custom namespace `{PROJECT_NAME}/ExperimentCosts`

**chargeback_report.py**: Team/cost-center chargeback report generator:
- Query Cost Explorer grouped by `CostCenter` tag for the specified billing period
- Join with COST_CENTER_MAPPING to add team names and budget owner emails
- Calculate per-team spend, budget utilization, and month-over-month trend
- Generate CSV chargeback report with columns: CostCenter, Team, BudgetOwner, MonthlySpend, BudgetLimit, Utilization%
- Optionally send report via SES to budget owners

**sns_topics.py**: Create SNS topics for notifications:
- Budget notification topic: `{PROJECT_NAME}-budget-alerts-{ENV}`
- Anomaly notification topic: `{PROJECT_NAME}-anomaly-alerts-{ENV}`
- Subscribe NOTIFICATION_EMAIL to both topics
- Set topic policies to allow AWS Budgets and Cost Anomaly Detection to publish

**run_setup.py**: CLI orchestrator that runs setup steps in order:
1. Load and validate configuration
2. Create SNS topics for notifications
3. Create tag policy and attach to OU (if ORGANIZATION_ID provided)
4. Activate cost allocation tags
5. Create per-service budgets with notifications
6. Create total ML budget with notifications
7. Create budget breach action (if BUDGET_ACTION_ENABLED)
8. Create anomaly detection monitors
9. Create anomaly alert subscriptions
10. Run tag compliance check
11. Print summary with budget names, monitor IDs, SNS topic ARNs, and compliance report

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**Tag Policy Enforcement:** Use AWS Organizations tag policies to enforce mandatory tags on all ML resources. Tag policies define which tag keys are required and what values are allowed. Enforce tags on SageMaker, Bedrock, Glue, Lambda, and S3 resource types. Tag policies are preventive — they block resource creation with missing or invalid tags. Requires AWS Organizations with tag policies enabled. Tag policies apply to the OU where they are attached and all child accounts.

**Cost Allocation Tags:** Activate all mandatory tag keys as cost allocation tags in AWS Billing. Cost allocation tags take up to 24 hours to appear in Cost Explorer after activation. Only activated tags can be used for grouping and filtering in Cost Explorer queries. User-defined tags (as opposed to AWS-generated tags) must be explicitly activated. Tag keys are case-sensitive in Cost Explorer.

**AWS Budgets:** Create separate budgets per ML service to track spending granularly. Use `COST` budget type with `MONTHLY` time unit. Set three notification thresholds: 50% (informational), 80% (warning), 100% (critical). Budget notifications support both SNS and email subscribers. Budget actions can automatically apply IAM policies on breach — use this to prevent runaway costs by denying new resource creation. Budget actions require a service-linked role for Budgets to assume.

**Budget Actions:** When BUDGET_ACTION_ENABLED is true, create a budget action that attaches a deny IAM policy to ML roles when the budget exceeds 100%. The deny policy blocks creation of new training jobs, endpoints, processing jobs, and provisioned throughput. Include a break-glass mechanism (Lambda function) that can temporarily remove the deny policy for critical workloads. Budget actions execute automatically — no human intervention required. Test budget actions in dev before enabling in prod.

**Cost Anomaly Detection:** Create anomaly monitors that use ML to detect unusual spending patterns. Service-level monitors detect anomalies per AWS service. Custom monitors with tag filters detect anomalies per cost center or team. Set the anomaly threshold (ANOMALY_THRESHOLD) to balance sensitivity — 20% is a good starting point for ML workloads which can have variable spend. Anomaly alerts are near-real-time (within hours of the anomalous spend).

**Cost Explorer Queries:** Use Cost Explorer API for programmatic cost analysis. Filter by `SERVICE` dimension for per-service costs, `INSTANCE_TYPE` for instance-level analysis, and tag keys for cost attribution. Use `get_cost_and_usage()` for historical data and `get_cost_forecast()` for projections. Cost Explorer data has a 24-hour delay. Group by multiple dimensions for drill-down analysis. Use both `BlendedCost` and `UnblendedCost` metrics — unblended is more accurate for single-account analysis.

**Per-Experiment Tracking:** Use the `ExperimentId` tag to attribute costs to individual ML experiments. Query Cost Explorer filtered by `ExperimentId` tag values to calculate per-experiment costs. Publish per-experiment cost metrics to CloudWatch for dashboard integration. This enables data scientists to understand the cost of their experiments and make cost-aware decisions.

**Security:** Budget action IAM roles need `iam:AttachRolePolicy` and `iam:DetachRolePolicy` permissions scoped to the deny policy ARN only. SNS topics need resource policies allowing `budgets.amazonaws.com` and `ce.amazonaws.com` to publish. Cost Explorer API requires `ce:GetCostAndUsage`, `ce:GetCostForecast`, `ce:CreateAnomalyMonitor`, `ce:CreateAnomalySubscription` permissions. Organizations tag policy management requires `organizations:CreatePolicy`, `organizations:AttachPolicy` permissions.

**Cost:** AWS Budgets: first 2 budgets per account are free, then $0.02/day per budget. Cost Anomaly Detection: free. Cost Explorer API: $0.01 per paginated API request. Tag policies: free (part of Organizations). Budget actions: free. Optimize Cost Explorer API calls by caching results and using appropriate granularity (DAILY vs MONTHLY).

**Naming:** All resources follow `{PROJECT_NAME}-{component}-{ENV}` convention:
- Budget: `{PROJECT_NAME}-{service}-budget-{ENV}`
- Total budget: `{PROJECT_NAME}-total-ml-budget-{ENV}`
- Anomaly monitor: `{PROJECT_NAME}-ml-service-anomaly-{ENV}`
- Anomaly subscription: `{PROJECT_NAME}-ml-anomaly-alerts-{ENV}`
- Tag policy: `{PROJECT_NAME}-ml-tag-policy-{ENV}`
- SNS topic: `{PROJECT_NAME}-budget-alerts-{ENV}`, `{PROJECT_NAME}-anomaly-alerts-{ENV}`
- Deny policy: `{PROJECT_NAME}-budget-breach-deny-{ENV}`
- CloudWatch namespace: `{PROJECT_NAME}/ExperimentCosts`

---

## Code Scaffolding Hints

**Create Organizations tag policy for mandatory ML tags:**
```python
import boto3
import json

organizations = boto3.client("organizations", region_name=AWS_REGION)

def create_tag_policy(mandatory_tags, project_name, env):
    """Create an Organizations tag policy enforcing mandatory ML tags."""
    tag_policy = {"tags": {}}

    for tag_key, tag_config in mandatory_tags.items():
        tag_rule = {
            "tag_key": {"@@assign": tag_key},
        }
        if tag_config.get("enforced", False):
            tag_rule["enforced_for"] = {
                "@@assign": [
                    "sagemaker:*",
                    "bedrock:*",
                    "glue:*",
                    "lambda:*",
                    "s3:*",
                ]
            }
        if tag_config.get("allowed_values") and tag_config["allowed_values"] != ["*"]:
            tag_rule["tag_value"] = {
                "@@assign": tag_config["allowed_values"]
            }
        tag_policy["tags"][tag_key] = tag_rule

    response = organizations.create_policy(
        Content=json.dumps(tag_policy),
        Description=f"Mandatory ML resource tags for {project_name} ({env})",
        Name=f"{project_name}-ml-tag-policy-{env}",
        Type="TAG_POLICY",
        Tags=[
            {"Key": "Project", "Value": project_name},
            {"Key": "Environment", "Value": env},
        ],
    )
    policy_id = response["Policy"]["PolicySummary"]["Id"]
    print(f"Tag policy created: {policy_id}")
    return policy_id


def attach_tag_policy(policy_id, target_id):
    """Attach tag policy to an OU or account."""
    organizations.attach_policy(
        PolicyId=policy_id,
        TargetId=target_id,
    )
    print(f"Tag policy {policy_id} attached to {target_id}")
```

**Create AWS Budget per ML service with 50/80/100% notifications:**
```python
budgets_client = boto3.client("budgets", region_name=AWS_REGION)

def create_service_budget(service_name, monthly_limit, account_id, project_name, env,
                          sns_topic_arn, notification_email):
    """Create a monthly cost budget for a specific ML service."""
    # Map friendly names to AWS service dimension values
    service_map = {
        "SageMaker": "Amazon SageMaker",
        "Bedrock": "Amazon Bedrock",
        "Glue": "AWS Glue",
        "Lambda": "AWS Lambda",
        "S3": "Amazon Simple Storage Service",
    }
    aws_service_name = service_map.get(service_name, service_name)
    budget_name = f"{project_name}-{service_name.lower()}-budget-{env}"

    budgets_client.create_budget(
        AccountId=account_id,
        Budget={
            "BudgetName": budget_name,
            "BudgetLimit": {
                "Amount": str(monthly_limit),
                "Unit": "USD",
            },
            "BudgetType": "COST",
            "TimeUnit": "MONTHLY",
            "CostFilters": {
                "Service": [aws_service_name],
            },
            "CostTypes": {
                "IncludeTax": True,
                "IncludeSubscription": True,
                "UseBlended": False,
                "IncludeRefund": False,
                "IncludeCredit": False,
                "IncludeUpfront": True,
                "IncludeRecurring": True,
                "IncludeOtherSubscription": True,
                "IncludeSupport": False,
                "IncludeDiscount": True,
                "UseAmortized": False,
            },
        },
        NotificationsWithSubscribers=[
            {
                "Notification": {
                    "NotificationType": "ACTUAL",
                    "ComparisonOperator": "GREATER_THAN",
                    "Threshold": 50.0,
                    "ThresholdType": "PERCENTAGE",
                    "NotificationState": "ALARM",
                },
                "Subscribers": [
                    {"SubscriptionType": "SNS", "Address": sns_topic_arn},
                    {"SubscriptionType": "EMAIL", "Address": notification_email},
                ],
            },
            {
                "Notification": {
                    "NotificationType": "ACTUAL",
                    "ComparisonOperator": "GREATER_THAN",
                    "Threshold": 80.0,
                    "ThresholdType": "PERCENTAGE",
                    "NotificationState": "ALARM",
                },
                "Subscribers": [
                    {"SubscriptionType": "SNS", "Address": sns_topic_arn},
                    {"SubscriptionType": "EMAIL", "Address": notification_email},
                ],
            },
            {
                "Notification": {
                    "NotificationType": "ACTUAL",
                    "ComparisonOperator": "GREATER_THAN",
                    "Threshold": 100.0,
                    "ThresholdType": "PERCENTAGE",
                    "NotificationState": "ALARM",
                },
                "Subscribers": [
                    {"SubscriptionType": "SNS", "Address": sns_topic_arn},
                    {"SubscriptionType": "EMAIL", "Address": notification_email},
                ],
            },
        ],
    )
    print(f"Budget created: {budget_name} (${monthly_limit}/month for {service_name})")
    return budget_name
```

**Cost Explorer query by service:**
```python
from datetime import datetime, timedelta

ce = boto3.client("ce", region_name=AWS_REGION)

def query_cost_by_service(days=30):
    """Query ML service costs for the last N days."""
    end_date = datetime.utcnow().strftime("%Y-%m-%d")
    start_date = (datetime.utcnow() - timedelta(days=days)).strftime("%Y-%m-%d")

    response = ce.get_cost_and_usage(
        TimePeriod={"Start": start_date, "End": end_date},
        Granularity="MONTHLY",
        Metrics=["UnblendedCost", "UsageQuantity"],
        Filter={
            "Dimensions": {
                "Key": "SERVICE",
                "Values": [
                    "Amazon SageMaker",
                    "Amazon Bedrock",
                    "AWS Glue",
                    "AWS Lambda",
                    "Amazon Simple Storage Service",
                ],
                "MatchOptions": ["EQUALS"],
            }
        },
        GroupBy=[
            {"Type": "DIMENSION", "Key": "SERVICE"},
        ],
    )

    for period in response["ResultsByTime"]:
        print(f"\nPeriod: {period['TimePeriod']['Start']} to {period['TimePeriod']['End']}")
        for group in period["Groups"]:
            service = group["Keys"][0]
            cost = float(group["Metrics"]["UnblendedCost"]["Amount"])
            print(f"  {service}: ${cost:.2f}")

    return response
```

**Cost Explorer query by instance type:**
```python
def query_cost_by_instance_type(days=30):
    """Query SageMaker costs grouped by instance type."""
    end_date = datetime.utcnow().strftime("%Y-%m-%d")
    start_date = (datetime.utcnow() - timedelta(days=days)).strftime("%Y-%m-%d")

    response = ce.get_cost_and_usage(
        TimePeriod={"Start": start_date, "End": end_date},
        Granularity="MONTHLY",
        Metrics=["UnblendedCost", "UsageQuantity"],
        Filter={
            "Dimensions": {
                "Key": "SERVICE",
                "Values": ["Amazon SageMaker"],
                "MatchOptions": ["EQUALS"],
            }
        },
        GroupBy=[
            {"Type": "DIMENSION", "Key": "INSTANCE_TYPE"},
        ],
    )

    for period in response["ResultsByTime"]:
        print(f"\nPeriod: {period['TimePeriod']['Start']} to {period['TimePeriod']['End']}")
        for group in period["Groups"]:
            instance_type = group["Keys"][0]
            cost = float(group["Metrics"]["UnblendedCost"]["Amount"])
            usage = float(group["Metrics"]["UsageQuantity"]["Amount"])
            print(f"  {instance_type}: ${cost:.2f} ({usage:.1f} hours)")

    return response
```

**Cost Explorer query by tag:**
```python
def query_cost_by_tag(tag_key, days=30):
    """Query costs grouped by a cost allocation tag."""
    end_date = datetime.utcnow().strftime("%Y-%m-%d")
    start_date = (datetime.utcnow() - timedelta(days=days)).strftime("%Y-%m-%d")

    response = ce.get_cost_and_usage(
        TimePeriod={"Start": start_date, "End": end_date},
        Granularity="MONTHLY",
        Metrics=["UnblendedCost"],
        Filter={
            "Dimensions": {
                "Key": "SERVICE",
                "Values": [
                    "Amazon SageMaker",
                    "Amazon Bedrock",
                    "AWS Glue",
                    "AWS Lambda",
                    "Amazon Simple Storage Service",
                ],
                "MatchOptions": ["EQUALS"],
            }
        },
        GroupBy=[
            {"Type": "TAG", "Key": tag_key},
        ],
    )

    total_tagged = 0.0
    total_untagged = 0.0
    for period in response["ResultsByTime"]:
        for group in period["Groups"]:
            tag_value = group["Keys"][0]
            cost = float(group["Metrics"]["UnblendedCost"]["Amount"])
            if tag_value.startswith(f"{tag_key}$"):
                tag_value = tag_value.split("$", 1)[1] or "(untagged)"
            if tag_value == "(untagged)" or not tag_value:
                total_untagged += cost
            else:
                total_tagged += cost
            print(f"  {tag_key}={tag_value}: ${cost:.2f}")

    total = total_tagged + total_untagged
    if total > 0:
        attribution_rate = (total_tagged / total) * 100
        print(f"\nCost attribution rate: {attribution_rate:.1f}% ({total_untagged:.2f} untagged)")

    return response
```

**Create Cost Anomaly Detection monitor:**
```python
def create_service_anomaly_monitor(project_name, env):
    """Create a Cost Anomaly Detection monitor for ML services."""
    response = ce.create_anomaly_monitor(
        AnomalyMonitor={
            "MonitorName": f"{project_name}-ml-service-anomaly-{env}",
            "MonitorType": "DIMENSIONAL",
            "MonitorDimension": "SERVICE",
        },
        ResourceTags=[
            {"Key": "Project", "Value": project_name},
            {"Key": "Environment", "Value": env},
        ],
    )
    monitor_arn = response["MonitorArn"]
    print(f"Service anomaly monitor created: {monitor_arn}")
    return monitor_arn


def create_costcenter_anomaly_monitor(project_name, env):
    """Create a Cost Anomaly Detection monitor filtered by CostCenter tag."""
    response = ce.create_anomaly_monitor(
        AnomalyMonitor={
            "MonitorName": f"{project_name}-ml-costcenter-anomaly-{env}",
            "MonitorType": "CUSTOM",
            "MonitorSpecification": {
                "Tags": {
                    "Key": "CostCenter",
                    "Values": [],  # Empty = monitor all CostCenter tag values
                    "MatchOptions": ["ABSENT", "EQUALS"],
                },
            },
        },
        ResourceTags=[
            {"Key": "Project", "Value": project_name},
            {"Key": "Environment", "Value": env},
        ],
    )
    monitor_arn = response["MonitorArn"]
    print(f"CostCenter anomaly monitor created: {monitor_arn}")
    return monitor_arn
```

**Create anomaly subscription with threshold:**
```python
def create_anomaly_subscription(monitor_arns, anomaly_threshold, sns_topic_arn,
                                 notification_email, project_name, env):
    """Create anomaly alert subscription for ML cost monitors."""
    response = ce.create_anomaly_subscription(
        AnomalySubscription={
            "SubscriptionName": f"{project_name}-ml-anomaly-alerts-{env}",
            "MonitorArnList": monitor_arns,
            "Subscribers": [
                {"Type": "SNS", "Address": sns_topic_arn},
                {"Type": "EMAIL", "Address": notification_email},
            ],
            "ThresholdExpression": {
                "Dimensions": {
                    "Key": "ANOMALY_TOTAL_IMPACT_PERCENTAGE",
                    "Values": [str(anomaly_threshold)],
                    "MatchOptions": ["GREATER_THAN_OR_EQUAL"],
                },
            },
            "Frequency": "IMMEDIATE",
        },
        ResourceTags=[
            {"Key": "Project", "Value": project_name},
            {"Key": "Environment", "Value": env},
        ],
    )
    subscription_arn = response["SubscriptionArn"]
    print(f"Anomaly subscription created: {subscription_arn}")
    return subscription_arn
```

**Budget breach IAM restriction action:**
```python
iam = boto3.client("iam", region_name=AWS_REGION)

def create_budget_breach_deny_policy(project_name, env):
    """Create IAM deny policy for budget breach restriction."""
    policy_document = {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "DenyMLResourceCreationOnBudgetBreach",
                "Effect": "Deny",
                "Action": [
                    "sagemaker:CreateTrainingJob",
                    "sagemaker:CreateEndpoint",
                    "sagemaker:CreateEndpointConfig",
                    "sagemaker:CreateProcessingJob",
                    "sagemaker:CreateTransformJob",
                    "sagemaker:CreateNotebookInstance",
                    "bedrock:CreateModelCustomizationJob",
                    "bedrock:CreateProvisionedModelThroughput",
                    "glue:CreateJob",
                    "glue:StartJobRun",
                ],
                "Resource": "*",
                "Condition": {
                    "StringEquals": {
                        "aws:RequestedRegion": AWS_REGION,
                    }
                },
            }
        ],
    }

    response = iam.create_policy(
        PolicyName=f"{project_name}-budget-breach-deny-{env}",
        PolicyDocument=json.dumps(policy_document),
        Description=f"Deny ML resource creation when budget is breached ({project_name} {env})",
        Tags=[
            {"Key": "Project", "Value": project_name},
            {"Key": "Environment", "Value": env},
            {"Key": "ManagedBy", "Value": "budget-action"},
        ],
    )
    policy_arn = response["Policy"]["Arn"]
    print(f"Budget breach deny policy created: {policy_arn}")
    return policy_arn


def create_budget_action(budget_name, account_id, deny_policy_arn, action_role_arn,
                          project_name, env):
    """Create budget action that applies deny policy on 100% breach."""
    response = budgets_client.create_budget_action(
        AccountId=account_id,
        BudgetName=budget_name,
        NotificationType="ACTUAL",
        ActionType="APPLY_IAM_POLICY",
        ActionThreshold={
            "ActionThresholdValue": 100.0,
            "ActionThresholdType": "PERCENTAGE",
        },
        Definition={
            "IamActionDefinition": {
                "PolicyArn": deny_policy_arn,
                "Roles": [f"{project_name}-sagemaker-execution-{env}"],
            },
        },
        ExecutionRoleArn=action_role_arn,
        ApprovalModel="AUTOMATIC",
        Subscribers=[
            {"SubscriptionType": "EMAIL", "Address": NOTIFICATION_EMAIL},
        ],
    )
    action_id = response["ActionId"]
    print(f"Budget action created: {action_id} for budget {budget_name}")
    return action_id
```

**Per-experiment cost tracking utility:**
```python
cloudwatch = boto3.client("cloudwatch", region_name=AWS_REGION)

def track_experiment_cost(experiment_id, days=30):
    """Query and track costs for a specific ML experiment."""
    end_date = datetime.utcnow().strftime("%Y-%m-%d")
    start_date = (datetime.utcnow() - timedelta(days=days)).strftime("%Y-%m-%d")

    response = ce.get_cost_and_usage(
        TimePeriod={"Start": start_date, "End": end_date},
        Granularity="DAILY",
        Metrics=["UnblendedCost"],
        Filter={
            "Tags": {
                "Key": "ExperimentId",
                "Values": [experiment_id],
                "MatchOptions": ["EQUALS"],
            }
        },
        GroupBy=[
            {"Type": "DIMENSION", "Key": "SERVICE"},
        ],
    )

    total_cost = 0.0
    service_costs = {}
    for period in response["ResultsByTime"]:
        for group in period["Groups"]:
            service = group["Keys"][0]
            cost = float(group["Metrics"]["UnblendedCost"]["Amount"])
            service_costs[service] = service_costs.get(service, 0.0) + cost
            total_cost += cost

    # Publish per-experiment cost metric to CloudWatch
    cloudwatch.put_metric_data(
        Namespace=f"{PROJECT_NAME}/ExperimentCosts",
        MetricData=[
            {
                "MetricName": "ExperimentTotalCost",
                "Dimensions": [
                    {"Name": "ExperimentId", "Value": experiment_id},
                    {"Name": "Environment", "Value": ENV},
                ],
                "Value": total_cost,
                "Unit": "None",
            },
        ],
    )

    report = {
        "experiment_id": experiment_id,
        "period": {"start": start_date, "end": end_date},
        "total_cost": round(total_cost, 2),
        "cost_by_service": {k: round(v, 2) for k, v in service_costs.items()},
    }
    print(json.dumps(report, indent=2))
    return report
```

**Tag compliance checker using Resource Groups Tagging API:**
```python
tagging = boto3.client("resourcegroupstaggingapi", region_name=AWS_REGION)

def check_tag_compliance(mandatory_tags):
    """Scan ML resources for missing or invalid tags."""
    resource_types = [
        "sagemaker:training-job",
        "sagemaker:endpoint",
        "sagemaker:notebook-instance",
        "sagemaker:processing-job",
        "lambda:function",
        "glue:job",
        "s3:bucket",
    ]

    non_compliant = []
    total_resources = 0

    paginator = tagging.get_paginator("get_resources")
    for page in paginator.paginate(ResourceTypeFilters=resource_types):
        for resource in page["ResourceTagMappingList"]:
            total_resources += 1
            arn = resource["ResourceARN"]
            tags = {t["Key"]: t["Value"] for t in resource.get("Tags", [])}

            missing_tags = []
            invalid_tags = []
            for tag_key, tag_config in mandatory_tags.items():
                if tag_config.get("enforced", False):
                    if tag_key not in tags:
                        missing_tags.append(tag_key)
                    elif (tag_config.get("allowed_values")
                          and tag_config["allowed_values"] != ["*"]
                          and tags[tag_key] not in tag_config["allowed_values"]):
                        invalid_tags.append(
                            {"key": tag_key, "value": tags[tag_key],
                             "allowed": tag_config["allowed_values"]}
                        )

            if missing_tags or invalid_tags:
                non_compliant.append({
                    "arn": arn,
                    "missing_tags": missing_tags,
                    "invalid_tags": invalid_tags,
                })

    compliant = total_resources - len(non_compliant)
    compliance_rate = (compliant / total_resources * 100) if total_resources > 0 else 100.0

    print(f"Tag compliance: {compliant}/{total_resources} ({compliance_rate:.1f}%)")
    for item in non_compliant:
        print(f"  NON-COMPLIANT: {item['arn']}")
        if item["missing_tags"]:
            print(f"    Missing: {', '.join(item['missing_tags'])}")
        for inv in item["invalid_tags"]:
            print(f"    Invalid: {inv['key']}={inv['value']} (allowed: {inv['allowed']})")

    return {"total": total_resources, "compliant": compliant,
            "non_compliant": non_compliant, "compliance_rate": compliance_rate}
```

**Cost forecast for budget breach prediction:**
```python
def forecast_ml_costs(budget_thresholds, days_ahead=30):
    """Forecast ML costs and predict budget breaches."""
    start_date = datetime.utcnow().strftime("%Y-%m-%d")
    end_date = (datetime.utcnow() + timedelta(days=days_ahead)).strftime("%Y-%m-%d")

    service_map = {
        "SageMaker": "Amazon SageMaker",
        "Bedrock": "Amazon Bedrock",
        "Glue": "AWS Glue",
        "Lambda": "AWS Lambda",
        "S3": "Amazon Simple Storage Service",
    }

    forecasts = {}
    for service_name, budget_limit in budget_thresholds.items():
        if service_name == "Total":
            continue
        aws_service = service_map.get(service_name)
        if not aws_service:
            continue

        try:
            response = ce.get_cost_forecast(
                TimePeriod={"Start": start_date, "End": end_date},
                Metric="UNBLENDED_COST",
                Granularity="MONTHLY",
                Filter={
                    "Dimensions": {
                        "Key": "SERVICE",
                        "Values": [aws_service],
                        "MatchOptions": ["EQUALS"],
                    }
                },
            )
            forecast_amount = float(response["Total"]["Amount"])
            utilization = (forecast_amount / budget_limit) * 100 if budget_limit > 0 else 0
            breach_risk = "HIGH" if utilization > 90 else "MEDIUM" if utilization > 70 else "LOW"

            forecasts[service_name] = {
                "forecast": round(forecast_amount, 2),
                "budget": budget_limit,
                "utilization_pct": round(utilization, 1),
                "breach_risk": breach_risk,
            }
            print(f"  {service_name}: ${forecast_amount:.2f} / ${budget_limit} "
                  f"({utilization:.1f}%) — Risk: {breach_risk}")
        except ce.exceptions.DataUnavailableException:
            print(f"  {service_name}: Forecast unavailable (insufficient historical data)")

    return forecasts
```

---

## Integration Points

- **Upstream**: `devops/04` → IAM roles for Budgets service-linked role, Cost Explorer API permissions, Organizations policy management permissions
- **Upstream**: `devops/12` → Bedrock invocation logging provides detailed per-invocation cost data for token-level cost attribution
- **Downstream**: `finops/05` → FinOps dashboards consume cost allocation data, budget status, and anomaly alerts for visualization
- **Downstream**: `finops/03` → Savings Plans analysis uses Cost Explorer historical data from this template's queries to recommend purchase plans
- **Downstream**: `devops/13` → Cost-per-inference dashboards use per-experiment and per-service cost data for inference cost tracking
- **Related**: `enterprise/01` → SCPs can enforce tag requirements at the Organizations level, complementing tag policies from this template
- **Related**: `enterprise/03` → Service Catalog products can enforce mandatory tags via launch constraints, ensuring cost attribution from provisioning
