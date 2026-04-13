<!-- Template Version: 1.0 | boto3: 1.35+ | QuickSight: Enterprise Edition -->

# Template FinOps 05 — FinOps Dashboards for ML Workloads

## Purpose
Generate a production-ready ML cost visualization and reporting framework: CloudWatch dashboards with ML spend widgets (cost by service, spend trend, per-endpoint cost, budget gauge indicators), QuickSight dataset and dashboard for Cost Explorer drill-down analysis, cost-per-inference metric math calculations, chargeback report generator producing CSV reports by business unit, budget status widget integration showing real-time budget utilization, and automatic widget creation for newly deployed SageMaker endpoints and training jobs.

---

## Role Definition

You are an expert AWS FinOps engineer and ML cost visualization specialist with expertise in:
- CloudWatch Dashboards: `put_dashboard()`, JSON dashboard body construction, metric widgets, text widgets, metric math expressions, gauge widgets, period/stat configuration
- Amazon QuickSight: `create_data_set()`, `create_dashboard()`, `create_analysis()`, `create_data_source()`, SPICE datasets, calculated fields, visual types (bar, line, pie, KPI, table)
- Cost Explorer integration: `ce.get_cost_and_usage()` data extraction for QuickSight ingestion, service-level and tag-level cost breakdowns
- CloudWatch Metric Math: `METRICS()`, `SEARCH()`, arithmetic expressions for cost-per-inference, cost-per-training-hour, budget utilization percentages
- Chargeback reporting: CSV generation by business unit/cost center, Cost Explorer tag-based queries, SES email delivery of reports
- AWS Budgets integration: budget status queries via `budgets.describe_budgets()`, budget utilization percentage calculations for dashboard widgets
- SageMaker metrics: `Invocations`, `ModelLatency`, `CPUUtilization`, `GPUUtilization` per endpoint, training job duration and instance hours
- Bedrock metrics: `InputTokenCount`, `OutputTokenCount`, `InvocationLatency`, `Invocations` per model
- Lambda-based automation: EventBridge-triggered Lambda for automatic widget creation when new endpoints or training jobs are detected
- IAM policies for CloudWatch, QuickSight, Cost Explorer, Budgets, SES, and Lambda permissions

Generate complete, production-deployable code.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

DASHBOARD_NAME:         [REQUIRED - name suffix for the CloudWatch dashboard]
                         Example: "ml-finops"
                         Full name becomes: {PROJECT_NAME}-{DASHBOARD_NAME}-{ENV}

SERVICES_TO_TRACK:      [REQUIRED - JSON list of AWS services to include in cost tracking]
                         Example:
                         ["SageMaker", "Bedrock", "Glue", "Lambda", "S3"]
                         Each service gets dedicated spend widgets and trend lines.

QUICKSIGHT_ENABLED:     [OPTIONAL: false - enable QuickSight dashboard creation]
                         Set to true to generate QuickSight dataset, analysis, and dashboard
                         for interactive Cost Explorer drill-down. Requires QuickSight
                         Enterprise Edition with SPICE capacity.

CHARGEBACK_ENABLED:     [OPTIONAL: false - enable chargeback report generation]
                         Set to true to generate CSV chargeback reports by business unit
                         using Cost Explorer tag-based queries. Reports are stored in S3
                         and optionally emailed via SES.

BUDGET_INTEGRATION:     [OPTIONAL: true - integrate budget status widgets]
                         Set to true to add budget utilization gauge widgets to the
                         CloudWatch dashboard. Requires budgets created by finops/01.

REPORT_FREQUENCY:       [OPTIONAL: weekly - frequency for chargeback report generation]
                         Options: daily, weekly, monthly
                         Controls the EventBridge schedule for automated report generation.

COST_CENTER_TAG:        [OPTIONAL: CostCenter - tag key used for chargeback attribution]
                         The cost allocation tag key used to group costs by business unit.

CHARGEBACK_BUCKET:      [OPTIONAL: none - S3 bucket for storing chargeback reports]
                         If not provided, a new bucket is created:
                         {PROJECT_NAME}-chargeback-reports-{ENV}

NOTIFICATION_EMAIL:     [OPTIONAL: none - email for chargeback report delivery via SES]
                         Example: "finops-team@example.com"

QUICKSIGHT_USER_ARN:    [OPTIONAL: none - QuickSight user ARN for dashboard permissions]
                         Required if QUICKSIGHT_ENABLED is true.
                         Example: "arn:aws:quicksight:us-east-1:123456789012:user/default/admin"

ENDPOINT_NAMES:         [OPTIONAL: none - JSON list of SageMaker endpoint names to track]
                         If not provided, auto-discovery via CloudWatch SEARCH() is used.
                         Example: ["text-classifier", "embedding-model", "recommender"]

AUTO_DISCOVER_ENDPOINTS: [OPTIONAL: true - automatically add widgets for new endpoints/jobs]
                          Set to true to deploy a Lambda that listens for SageMaker
                          endpoint and training job creation events via EventBridge and
                          automatically adds corresponding widgets to the dashboard.
```

---

## Task

Generate complete ML FinOps dashboard and reporting framework:

```
{PROJECT_NAME}-finops-dashboards/
├── cloudwatch/
│   ├── create_dashboard.py               # Main CloudWatch dashboard with ML spend widgets
│   ├── widgets/
│   │   ├── cost_by_service.py            # Widget: ML spend breakdown by service
│   │   ├── spend_trend.py                # Widget: daily/weekly spend trend lines
│   │   ├── per_endpoint_cost.py          # Widget: cost per SageMaker endpoint
│   │   ├── budget_gauges.py              # Widget: budget utilization gauge indicators
│   │   ├── cost_per_inference.py         # Widget: cost-per-inference metric math
│   │   └── bedrock_token_cost.py         # Widget: Bedrock token usage and cost
│   └── auto_discover/
│       ├── endpoint_watcher_lambda.py    # Lambda: auto-add widgets for new endpoints/jobs
│       └── deploy_watcher.py             # Deploy the auto-discovery Lambda + EventBridge rule
├── quicksight/
│   ├── create_data_source.py             # QuickSight data source from Cost Explorer CSV
│   ├── create_dataset.py                 # QuickSight SPICE dataset with calculated fields
│   ├── create_analysis.py                # QuickSight analysis with cost visuals
│   └── create_dashboard.py              # QuickSight published dashboard
├── chargeback/
│   ├── generate_report.py                # CSV chargeback report by business unit
│   ├── schedule_report.py                # EventBridge schedule for automated reports
│   └── email_report.py                   # SES email delivery of chargeback reports
├── infrastructure/
│   ├── config.py                         # Central configuration
│   └── s3_bucket.py                      # S3 bucket for reports and QuickSight data
├── run_setup.py                          # CLI orchestrator
└── requirements.txt
```

**config.py**: Central configuration dataclass with all parameters. Load from environment variables or CLI args. Validate required fields. Parse SERVICES_TO_TRACK and ENDPOINT_NAMES from JSON strings. Validate REPORT_FREQUENCY against allowed values (daily, weekly, monthly). Construct resource names using `{PROJECT_NAME}-{component}-{ENV}` convention. Construct full dashboard name as `{PROJECT_NAME}-{DASHBOARD_NAME}-{ENV}`.

**create_dashboard.py**: Create the main CloudWatch dashboard:
- Build dashboard JSON body by assembling widgets from the `widgets/` modules
- Call `cloudwatch.put_dashboard()` with the assembled JSON body
- Dashboard layout: 3-column grid (width 24 units)
  - Row 1: Title text widget (full width) with dashboard name, last updated timestamp, and legend
  - Row 2: Cost by service pie chart (width 8), spend trend line chart (width 8), budget gauges (width 8)
  - Row 3: Per-endpoint cost bar chart (width 12), cost-per-inference metric math (width 12)
  - Row 4: Bedrock token cost breakdown (width 12), training job cost summary (width 12)
- Dashboard name: `{PROJECT_NAME}-{DASHBOARD_NAME}-{ENV}`
- Set dashboard period to 1 day (86400 seconds) for cost-oriented views
- Tag dashboard via CloudWatch tag API

**cost_by_service.py**: Generate cost-by-service widget definition:
- Use CloudWatch custom metrics published by `finops/01` experiment cost tracker
- Metric namespace: `{PROJECT_NAME}/MLCosts`
- Metrics: one metric per service in SERVICES_TO_TRACK (e.g., `SageMakerDailyCost`, `BedrockDailyCost`)
- Widget type: `metric` with `stat: "Sum"`, `period: 86400` (daily)
- View: `pie` for distribution, with a companion `timeSeries` for trend
- Include `METRICS()` expression to sum all service costs into a total

**spend_trend.py**: Generate spend trend widget definition:
- Line chart showing daily ML spend over the last 30 days
- One line per service in SERVICES_TO_TRACK
- Metric namespace: `{PROJECT_NAME}/MLCosts`
- Period: 86400 (daily), Stat: Sum
- Include metric math expression `e1 = SUM(METRICS())` for total spend line
- Y-axis label: "USD", annotations for budget thresholds if BUDGET_INTEGRATION enabled
- Widget type: `metric` with `view: "timeSeries"`, `stacked: false`

**per_endpoint_cost.py**: Generate per-endpoint cost widget definition:
- If ENDPOINT_NAMES provided, create explicit metrics for each endpoint
- If AUTO_DISCOVER_ENDPOINTS, use `SEARCH('{AWS/SageMaker,EndpointName} MetricName="Invocations"', 'Sum', 86400)` to auto-discover endpoints
- For each endpoint, calculate estimated cost using metric math:
  - `invocations = SageMaker Invocations metric`
  - `instance_hours = (period / 3600) * instance_count` (from custom metric)
  - `estimated_cost = instance_hours * hourly_rate` (from custom metric published by `finops/04`)
- Widget type: `metric` with `view: "bar"`, grouped by endpoint name
- Period: 86400 (daily)

**budget_gauges.py**: Generate budget utilization gauge widgets:
- Query budget status using `budgets.describe_budgets()` for each service budget
- For each budget, create a gauge widget showing:
  - Current spend vs budget limit as a percentage
  - Color coding: green (<50%), yellow (50-80%), red (>80%)
- Use CloudWatch custom metrics: publish budget utilization percentages from a Lambda
- Widget type: `metric` with `view: "gauge"`, `yAxis: {left: {min: 0, max: 100}}`
- One gauge per service in SERVICES_TO_TRACK plus a total ML budget gauge

**cost_per_inference.py**: Generate cost-per-inference metric math widget:
- Use CloudWatch metric math to calculate cost per inference for each endpoint:
  - `m1 = AWS/SageMaker Invocations` (Sum, per endpoint)
  - `m2 = {PROJECT_NAME}/MLCosts EndpointHourlyCost` (custom metric from `devops/13`)
  - `e1 = IF(m1 > 0, (m2 * 3600 / PERIOD(m1)) / m1, 0)` — cost per single inference
  - `e2 = e1 * 1000` — cost per 1000 inferences
- Widget type: `metric` with `view: "timeSeries"`
- Period: 3600 (hourly) for granular cost-per-inference tracking
- Y-axis label: "USD per 1000 inferences"

**bedrock_token_cost.py**: Generate Bedrock token usage and cost widget:
- Metrics from `AWS/Bedrock` namespace:
  - `InputTokenCount` (Sum) per model
  - `OutputTokenCount` (Sum) per model
  - `Invocations` (Sum) per model
- Metric math for estimated cost:
  - `input_cost = InputTokenCount * input_price_per_token` (custom metric from `devops/12`)
  - `output_cost = OutputTokenCount * output_price_per_token`
  - `total_cost = input_cost + output_cost`
- Widget type: `metric` with `view: "timeSeries"`, stacked for input vs output cost
- Period: 86400 (daily)

**endpoint_watcher_lambda.py**: Lambda function for automatic widget creation:
- Triggered by EventBridge rules for:
  - `SageMaker Endpoint State Change` (status = `InService`)
  - `SageMaker Training Job State Change` (status = `Completed`)
- On new endpoint detection:
  - Fetch current dashboard JSON via `cloudwatch.get_dashboard()`
  - Parse the dashboard body JSON
  - Add new metric widgets for the endpoint (invocations, latency, cost-per-inference)
  - Call `cloudwatch.put_dashboard()` with the updated JSON
- On training job completion:
  - Add a training cost annotation to the spend trend widget
- Log all dashboard updates to CloudWatch Logs
- Send SNS notification about the new widget addition

**deploy_watcher.py**: Deploy the auto-discovery Lambda and EventBridge rules:
- Create Lambda function: `{PROJECT_NAME}-endpoint-watcher-{ENV}`
- Create EventBridge rules:
  - Rule 1: `{PROJECT_NAME}-new-endpoint-{ENV}` — matches `SageMaker Endpoint State Change` with `InService` status
  - Rule 2: `{PROJECT_NAME}-training-complete-{ENV}` — matches `SageMaker Training Job State Change` with `Completed` status
- Add Lambda as target for both rules
- IAM role with permissions for CloudWatch dashboards, SageMaker describe, SNS publish

**create_data_source.py**: Create QuickSight data source:
- Export Cost Explorer data to CSV in S3 using `ce.get_cost_and_usage()` for the last 90 days
- Group by SERVICE, INSTANCE_TYPE, and cost allocation tags (Team, CostCenter)
- Write CSV to `s3://{CHARGEBACK_BUCKET}/quicksight/cost_data.csv`
- Create QuickSight data source using `quicksight.create_data_source()`:
  - Type: `S3`
  - S3 manifest pointing to the CSV file
  - Data source name: `{PROJECT_NAME}-cost-data-{ENV}`

**create_dataset.py**: Create QuickSight SPICE dataset:
- Call `quicksight.create_data_set()` with:
  - Physical table map referencing the S3 data source
  - Logical table map with calculated fields:
    - `CostPerInference`: `cost / invocations` (where applicable)
    - `BudgetUtilization`: `actual_spend / budget_limit * 100`
    - `MonthOverMonthChange`: `(current_month - previous_month) / previous_month * 100`
    - `ServiceCategory`: CASE expression mapping services to categories (Training, Inference, Data, Storage)
  - Import mode: `SPICE` for fast interactive queries
  - Dataset name: `{PROJECT_NAME}-ml-costs-{ENV}`
- Schedule SPICE refresh: daily at 6 AM UTC

**create_analysis.py**: Create QuickSight analysis with cost visuals:
- Call `quicksight.create_analysis()` with:
  - Sheet 1 — Overview: KPI widgets (total spend, budget utilization, cost trend), pie chart (spend by service), line chart (daily spend trend)
  - Sheet 2 — Service Drill-Down: bar chart (cost by instance type), table (top 10 costliest resources), heat map (cost by service × day)
  - Sheet 3 — Chargeback: bar chart (cost by business unit), table (per-team spend with budget utilization), pie chart (cost distribution by cost center)
  - Sheet 4 — Inference: line chart (cost-per-inference trend), bar chart (cost by endpoint), scatter plot (latency vs cost)
- Analysis name: `{PROJECT_NAME}-ml-cost-analysis-{ENV}`

**create_dashboard.py** (quicksight/): Publish QuickSight dashboard from analysis:
- Call `quicksight.create_dashboard()` with:
  - Source entity: the analysis created above
  - Dashboard name: `{PROJECT_NAME}-ml-cost-dashboard-{ENV}`
  - Permissions: grant `quicksight:DescribeDashboard`, `quicksight:ListDashboardVersions`, `quicksight:QueryDashboard` to QUICKSIGHT_USER_ARN
- Print dashboard URL for sharing

**generate_report.py**: Generate CSV chargeback report by business unit:
- Query Cost Explorer using `ce.get_cost_and_usage()` grouped by COST_CENTER_TAG
- For each cost center, break down costs by service
- Calculate: total spend, budget utilization %, month-over-month change %
- Generate CSV with columns: CostCenter, Team, Service, MonthlySpend, BudgetLimit, Utilization%, MoMChange%
- Include summary row with totals
- Upload CSV to `s3://{CHARGEBACK_BUCKET}/reports/{year}/{month}/chargeback_{date}.csv`
- Return S3 path for email delivery

**schedule_report.py**: Create EventBridge schedule for automated report generation:
- Create EventBridge rule based on REPORT_FREQUENCY:
  - daily: `cron(0 7 * * ? *)` — 7 AM UTC daily
  - weekly: `cron(0 7 ? * MON *)` — 7 AM UTC every Monday
  - monthly: `cron(0 7 1 * ? *)` — 7 AM UTC first day of month
- Target: Lambda function that runs `generate_report.py` and `email_report.py`
- Rule name: `{PROJECT_NAME}-chargeback-schedule-{ENV}`
- Deploy the report generation Lambda function

**email_report.py**: Send chargeback report via SES:
- Download CSV from S3
- Build email with:
  - Subject: `[{ENV.upper()}] ML Cost Chargeback Report — {date_range}`
  - Body: summary table (HTML) with per-business-unit totals and budget utilization
  - Attachment: full CSV report
- Send via `ses.send_raw_email()` to NOTIFICATION_EMAIL
- Verify sender identity exists in SES before sending

**s3_bucket.py**: Create S3 bucket for reports and QuickSight data:
- Bucket name: `{PROJECT_NAME}-chargeback-reports-{ENV}`
- Enable server-side encryption (SSE-S3)
- Set lifecycle rule: transition reports to Glacier after 90 days, delete after 365 days
- Block public access
- Bucket policy allowing QuickSight service principal to read (if QUICKSIGHT_ENABLED)

**run_setup.py**: CLI orchestrator that runs setup steps in order:
1. Load and validate configuration
2. Create S3 bucket for reports (if CHARGEBACK_ENABLED or QUICKSIGHT_ENABLED)
3. Create CloudWatch dashboard with all widgets
4. Deploy endpoint watcher Lambda and EventBridge rules (if AUTO_DISCOVER_ENDPOINTS)
5. Create QuickSight data source, dataset, analysis, and dashboard (if QUICKSIGHT_ENABLED)
6. Generate initial chargeback report (if CHARGEBACK_ENABLED)
7. Create chargeback report schedule (if CHARGEBACK_ENABLED)
8. Print summary with dashboard URL, QuickSight dashboard URL, report S3 path, and schedule

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**CloudWatch Dashboards:** Use `cloudwatch.put_dashboard()` with a JSON dashboard body. Dashboard body is a JSON string containing an array of widgets. Each widget has `type` (metric, text, log), `x`/`y` position, `width`/`height` in grid units (max width 24), and `properties` with metric definitions. Use metric math expressions for calculated metrics like cost-per-inference — metric math runs server-side and does not incur additional API costs. Use `SEARCH()` expressions for auto-discovery of new endpoints without dashboard updates. Dashboard names must be unique per account per region.

**Metric Math for Cost:** CloudWatch metric math enables server-side calculations without custom Lambda functions. Key patterns: `SUM(METRICS())` to aggregate across services, `IF(m1 > 0, m2/m1, 0)` for safe division in cost-per-inference, `PERIOD(m1)` to normalize rates across different periods. Metric math expressions can reference up to 500 metrics. Use `id` labels (e.g., `e1`, `m1`) to reference metrics in expressions. Metric math does not support string operations — use separate widgets for categorical breakdowns.

**QuickSight:** Requires QuickSight Enterprise Edition for SPICE datasets and scheduled refreshes. SPICE capacity is billed per GB ($0.25/GB/month). Create a manifest file in S3 that points to the cost data CSV for the S3 data source. Use calculated fields for derived metrics (cost-per-inference, budget utilization). Schedule SPICE refresh daily to keep data current. Dashboard permissions control who can view — grant to specific QuickSight users or groups. QuickSight dashboards support embedding in internal portals via `GetDashboardEmbedUrl`.

**Chargeback Reports:** Query Cost Explorer grouped by the COST_CENTER_TAG to attribute costs to business units. Cost Explorer data has a 24-hour delay — reports reflect spend through the previous day. Generate CSV with standard columns for finance team consumption. Store reports in S3 with date-partitioned paths for historical access. Use SES for email delivery — verify sender identity before sending. Schedule reports via EventBridge + Lambda for automated delivery.

**Budget Integration:** Query budget status using `budgets.describe_budgets()` to get current spend vs limit. Publish budget utilization percentages as custom CloudWatch metrics for dashboard gauge widgets. Budget data updates approximately every 8-12 hours — gauges show near-real-time but not instant values. Color-code gauges: green (<50%), yellow (50-80%), red (>80%), critical (>100%). Reference budgets created by `finops/01` using the naming convention `{PROJECT_NAME}-{service}-budget-{ENV}`.

**Auto-Discovery:** Use EventBridge rules to detect new SageMaker endpoints and training jobs. The watcher Lambda fetches the current dashboard JSON, adds new widgets, and updates the dashboard. This avoids manual dashboard maintenance as the ML platform grows. Use `SEARCH()` metric expressions where possible — they automatically include new endpoints matching the search pattern without dashboard updates. For training jobs, add cost annotations to the spend trend widget showing job cost and duration.

**Security:** CloudWatch dashboard access is controlled by IAM policies (`cloudwatch:GetDashboard`, `cloudwatch:PutDashboard`). QuickSight dashboard access is controlled by QuickSight permissions (separate from IAM). Cost Explorer API requires `ce:GetCostAndUsage` permissions. Budgets API requires `budgets:DescribeBudgets` permissions. SES requires `ses:SendRawEmail` and verified sender identity. The watcher Lambda needs `cloudwatch:GetDashboard`, `cloudwatch:PutDashboard`, `sagemaker:DescribeEndpoint`, and `sns:Publish` permissions. Chargeback reports may contain sensitive cost data — encrypt S3 bucket and restrict access.

**Cost:** CloudWatch dashboards: first 3 dashboards free, then $3/month per dashboard. CloudWatch metrics: custom metrics $0.30/metric/month. QuickSight: $18/month per author, $0.25/GB SPICE. Cost Explorer API: $0.01 per request. SES: $0.10 per 1000 emails. Lambda: negligible for scheduled report generation. Optimize Cost Explorer API calls by caching results in S3 and querying only incremental data.

**Naming:** All resources follow `{PROJECT_NAME}-{component}-{ENV}` convention:
- CloudWatch dashboard: `{PROJECT_NAME}-{DASHBOARD_NAME}-{ENV}`
- QuickSight data source: `{PROJECT_NAME}-cost-data-{ENV}`
- QuickSight dataset: `{PROJECT_NAME}-ml-costs-{ENV}`
- QuickSight analysis: `{PROJECT_NAME}-ml-cost-analysis-{ENV}`
- QuickSight dashboard: `{PROJECT_NAME}-ml-cost-dashboard-{ENV}`
- Chargeback S3 bucket: `{PROJECT_NAME}-chargeback-reports-{ENV}`
- Chargeback report path: `s3://{bucket}/reports/{year}/{month}/chargeback_{date}.csv`
- EventBridge rules: `{PROJECT_NAME}-new-endpoint-{ENV}`, `{PROJECT_NAME}-training-complete-{ENV}`, `{PROJECT_NAME}-chargeback-schedule-{ENV}`
- Watcher Lambda: `{PROJECT_NAME}-endpoint-watcher-{ENV}`
- Report Lambda: `{PROJECT_NAME}-chargeback-reporter-{ENV}`
- CloudWatch namespace: `{PROJECT_NAME}/MLCosts`

---

## Code Scaffolding Hints

**Create CloudWatch dashboard with metric math widgets:**
```python
import boto3
import json

cloudwatch = boto3.client("cloudwatch", region_name=AWS_REGION)

dashboard_name = f"{PROJECT_NAME}-{DASHBOARD_NAME}-{ENV}"

dashboard_body = {
    "widgets": [
        # Row 1: Title
        {
            "type": "text",
            "x": 0, "y": 0, "width": 24, "height": 2,
            "properties": {
                "markdown": f"# {PROJECT_NAME} ML FinOps Dashboard ({ENV})\nLast updated: {{timestamp}}. Costs are approximate and based on CloudWatch metrics and Cost Explorer data."
            },
        },
        # Row 2: Cost by service (pie chart)
        {
            "type": "metric",
            "x": 0, "y": 2, "width": 8, "height": 6,
            "properties": {
                "metrics": [
                    [f"{PROJECT_NAME}/MLCosts", "SageMakerDailyCost", {"id": "m1", "label": "SageMaker"}],
                    [f"{PROJECT_NAME}/MLCosts", "BedrockDailyCost", {"id": "m2", "label": "Bedrock"}],
                    [f"{PROJECT_NAME}/MLCosts", "GlueDailyCost", {"id": "m3", "label": "Glue"}],
                    [f"{PROJECT_NAME}/MLCosts", "LambdaDailyCost", {"id": "m4", "label": "Lambda"}],
                    [f"{PROJECT_NAME}/MLCosts", "S3DailyCost", {"id": "m5", "label": "S3"}],
                ],
                "view": "pie",
                "title": "ML Spend by Service",
                "period": 86400,
                "stat": "Sum",
                "region": AWS_REGION,
            },
        },
        # Row 2: Spend trend (line chart)
        {
            "type": "metric",
            "x": 8, "y": 2, "width": 8, "height": 6,
            "properties": {
                "metrics": [
                    [f"{PROJECT_NAME}/MLCosts", "SageMakerDailyCost", {"id": "m1", "label": "SageMaker"}],
                    [f"{PROJECT_NAME}/MLCosts", "BedrockDailyCost", {"id": "m2", "label": "Bedrock"}],
                    [f"{PROJECT_NAME}/MLCosts", "GlueDailyCost", {"id": "m3", "label": "Glue"}],
                    [f"{PROJECT_NAME}/MLCosts", "LambdaDailyCost", {"id": "m4", "label": "Lambda"}],
                    [f"{PROJECT_NAME}/MLCosts", "S3DailyCost", {"id": "m5", "label": "S3"}],
                    [{"expression": "SUM(METRICS())", "id": "e1", "label": "Total ML Spend", "color": "#FF0000"}],
                ],
                "view": "timeSeries",
                "stacked": False,
                "title": "Daily ML Spend Trend (30 days)",
                "period": 86400,
                "stat": "Sum",
                "region": AWS_REGION,
                "yAxis": {"left": {"label": "USD", "showUnits": False}},
            },
        },
        # Row 2: Budget gauges
        {
            "type": "metric",
            "x": 16, "y": 2, "width": 8, "height": 6,
            "properties": {
                "metrics": [
                    [f"{PROJECT_NAME}/MLCosts", "BudgetUtilization", "Service", "Total",
                     {"id": "m1", "label": "Total Budget"}],
                    [f"{PROJECT_NAME}/MLCosts", "BudgetUtilization", "Service", "SageMaker",
                     {"id": "m2", "label": "SageMaker Budget"}],
                    [f"{PROJECT_NAME}/MLCosts", "BudgetUtilization", "Service", "Bedrock",
                     {"id": "m3", "label": "Bedrock Budget"}],
                ],
                "view": "gauge",
                "title": "Budget Utilization (%)",
                "period": 86400,
                "stat": "Maximum",
                "region": AWS_REGION,
                "yAxis": {"left": {"min": 0, "max": 100}},
                "annotations": {
                    "horizontal": [
                        {"value": 50, "color": "#2ca02c", "label": "On Track"},
                        {"value": 80, "color": "#ff7f0e", "label": "Warning"},
                        {"value": 100, "color": "#d62728", "label": "Over Budget"},
                    ]
                },
            },
        },
        # Row 3: Per-endpoint cost (bar chart with SEARCH)
        {
            "type": "metric",
            "x": 0, "y": 8, "width": 12, "height": 6,
            "properties": {
                "metrics": [
                    [{"expression": f"SEARCH('{{AWS/SageMaker,EndpointName}} MetricName=\"Invocations\"', 'Sum', 86400)", "id": "e1", "label": ""}],
                ],
                "view": "bar",
                "title": "Invocations per Endpoint (Daily)",
                "period": 86400,
                "stat": "Sum",
                "region": AWS_REGION,
            },
        },
        # Row 3: Cost per inference (metric math)
        {
            "type": "metric",
            "x": 12, "y": 8, "width": 12, "height": 6,
            "properties": {
                "metrics": [
                    ["AWS/SageMaker", "Invocations", "EndpointName", f"{PROJECT_NAME}-*-{ENV}",
                     "VariantName", "AllTraffic", {"id": "m1", "visible": False, "stat": "Sum"}],
                    [f"{PROJECT_NAME}/MLCosts", "EndpointHourlyCost", "EndpointName", f"{PROJECT_NAME}-*-{ENV}",
                     {"id": "m2", "visible": False, "stat": "Average"}],
                    [{"expression": "IF(m1 > 0, (m2 * 3600 / PERIOD(m1)) / m1 * 1000, 0)",
                      "id": "e1", "label": "Cost per 1K Inferences (USD)", "color": "#1f77b4"}],
                ],
                "view": "timeSeries",
                "title": "Cost per 1000 Inferences",
                "period": 3600,
                "region": AWS_REGION,
                "yAxis": {"left": {"label": "USD per 1K inferences", "showUnits": False}},
            },
        },
        # Row 4: Bedrock token cost
        {
            "type": "metric",
            "x": 0, "y": 14, "width": 12, "height": 6,
            "properties": {
                "metrics": [
                    [{"expression": f"SEARCH('{{AWS/Bedrock,ModelId}} MetricName=\"InputTokenCount\"', 'Sum', 86400)", "id": "e1", "label": "Input Tokens"}],
                    [{"expression": f"SEARCH('{{AWS/Bedrock,ModelId}} MetricName=\"OutputTokenCount\"', 'Sum', 86400)", "id": "e2", "label": "Output Tokens"}],
                ],
                "view": "timeSeries",
                "stacked": True,
                "title": "Bedrock Token Usage (Daily)",
                "period": 86400,
                "stat": "Sum",
                "region": AWS_REGION,
                "yAxis": {"left": {"label": "Token Count", "showUnits": False}},
            },
        },
        # Row 4: Training job cost summary
        {
            "type": "metric",
            "x": 12, "y": 14, "width": 12, "height": 6,
            "properties": {
                "metrics": [
                    [f"{PROJECT_NAME}/MLCosts", "TrainingJobCost", {"id": "m1", "label": "Training Cost", "stat": "Sum"}],
                    [f"{PROJECT_NAME}/MLCosts", "ProcessingJobCost", {"id": "m2", "label": "Processing Cost", "stat": "Sum"}],
                    [{"expression": "m1 + m2", "id": "e1", "label": "Total Compute Cost", "color": "#d62728"}],
                ],
                "view": "timeSeries",
                "title": "Training & Processing Job Costs",
                "period": 86400,
                "region": AWS_REGION,
                "yAxis": {"left": {"label": "USD", "showUnits": False}},
            },
        },
    ],
}

cloudwatch.put_dashboard(
    DashboardName=dashboard_name,
    DashboardBody=json.dumps(dashboard_body),
)
print(f"Dashboard created: {dashboard_name}")
print(f"  URL: https://{AWS_REGION}.console.aws.amazon.com/cloudwatch/home?region={AWS_REGION}#dashboards:name={dashboard_name}")
```

**Publish budget utilization as custom CloudWatch metrics:**
```python
budgets_client = boto3.client("budgets", region_name=AWS_REGION)

def publish_budget_utilization(account_id, project_name, env, services):
    """Query budget status and publish utilization % to CloudWatch."""
    response = budgets_client.describe_budgets(AccountId=account_id)

    metric_data = []
    for budget in response.get("Budgets", []):
        budget_name = budget["BudgetName"]
        # Match budgets created by finops/01
        if not budget_name.startswith(f"{project_name}-"):
            continue

        limit = float(budget["BudgetLimit"]["Amount"])
        actual = float(budget.get("CalculatedSpend", {}).get("ActualSpend", {}).get("Amount", "0"))
        utilization = (actual / limit * 100) if limit > 0 else 0

        # Determine service name from budget name
        service = "Total"
        for svc in services:
            if svc.lower() in budget_name.lower():
                service = svc
                break

        metric_data.append({
            "MetricName": "BudgetUtilization",
            "Dimensions": [{"Name": "Service", "Value": service}],
            "Value": utilization,
            "Unit": "Percent",
        })
        metric_data.append({
            "MetricName": "BudgetActualSpend",
            "Dimensions": [{"Name": "Service", "Value": service}],
            "Value": actual,
            "Unit": "None",
        })

    if metric_data:
        cloudwatch.put_metric_data(
            Namespace=f"{project_name}/MLCosts",
            MetricData=metric_data,
        )
        print(f"Published {len(metric_data)} budget utilization metrics")
```

**QuickSight dataset creation with calculated fields:**
```python
quicksight = boto3.client("quicksight", region_name=AWS_REGION)

# Create data source from S3 CSV
quicksight.create_data_source(
    AwsAccountId=AWS_ACCOUNT_ID,
    DataSourceId=f"{PROJECT_NAME}-cost-data-{ENV}",
    Name=f"{PROJECT_NAME}-cost-data-{ENV}",
    Type="S3",
    DataSourceParameters={
        "S3Parameters": {
            "ManifestFileLocation": {
                "Bucket": f"{PROJECT_NAME}-chargeback-reports-{ENV}",
                "Key": "quicksight/manifest.json",
            }
        }
    },
    Permissions=[
        {
            "Principal": QUICKSIGHT_USER_ARN,
            "Actions": [
                "quicksight:DescribeDataSource",
                "quicksight:DescribeDataSourcePermissions",
                "quicksight:PassDataSource",
                "quicksight:UpdateDataSource",
                "quicksight:DeleteDataSource",
                "quicksight:UpdateDataSourcePermissions",
            ],
        }
    ],
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)

# Create SPICE dataset with calculated fields
quicksight.create_data_set(
    AwsAccountId=AWS_ACCOUNT_ID,
    DataSetId=f"{PROJECT_NAME}-ml-costs-{ENV}",
    Name=f"{PROJECT_NAME}-ml-costs-{ENV}",
    PhysicalTableMap={
        "cost-table": {
            "S3Source": {
                "DataSourceArn": f"arn:aws:quicksight:{AWS_REGION}:{AWS_ACCOUNT_ID}:datasource/{PROJECT_NAME}-cost-data-{ENV}",
                "InputColumns": [
                    {"Name": "Date", "Type": "STRING"},
                    {"Name": "Service", "Type": "STRING"},
                    {"Name": "CostCenter", "Type": "STRING"},
                    {"Name": "Team", "Type": "STRING"},
                    {"Name": "UnblendedCost", "Type": "DECIMAL"},
                    {"Name": "UsageQuantity", "Type": "DECIMAL"},
                    {"Name": "InstanceType", "Type": "STRING"},
                    {"Name": "BudgetLimit", "Type": "DECIMAL"},
                ],
                "UploadSettings": {
                    "Format": "CSV",
                    "StartFromRow": 1,
                    "ContainsHeader": True,
                    "Delimiter": ",",
                },
            }
        }
    },
    LogicalTableMap={
        "cost-logical": {
            "Alias": "ML Cost Data",
            "Source": {"PhysicalTableId": "cost-table"},
            "DataTransforms": [
                {
                    "CreateColumnsOperation": {
                        "Columns": [
                            {
                                "ColumnName": "BudgetUtilization",
                                "ColumnId": "budget_util",
                                "Expression": "ifelse({BudgetLimit} > 0, {UnblendedCost} / {BudgetLimit} * 100, 0)",
                            },
                            {
                                "ColumnName": "ServiceCategory",
                                "ColumnId": "svc_cat",
                                "Expression": "ifelse({Service} = 'Amazon SageMaker', 'ML Compute', {Service} = 'Amazon Bedrock', 'GenAI', {Service} = 'AWS Glue', 'Data Processing', 'Other')",
                            },
                        ]
                    }
                },
                {
                    "CastColumnTypeOperation": {
                        "ColumnName": "Date",
                        "NewColumnType": "DATETIME",
                        "Format": "yyyy-MM-dd",
                    }
                },
            ],
        }
    },
    ImportMode="SPICE",
    Permissions=[
        {
            "Principal": QUICKSIGHT_USER_ARN,
            "Actions": [
                "quicksight:DescribeDataSet",
                "quicksight:DescribeDataSetPermissions",
                "quicksight:PassDataSet",
                "quicksight:DescribeIngestion",
                "quicksight:ListIngestions",
                "quicksight:UpdateDataSet",
                "quicksight:DeleteDataSet",
                "quicksight:CreateIngestion",
                "quicksight:CancelIngestion",
                "quicksight:UpdateDataSetPermissions",
            ],
        }
    ],
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)
print(f"QuickSight dataset created: {PROJECT_NAME}-ml-costs-{ENV}")
```

**Create QuickSight dashboard from analysis:**
```python
# Create analysis
quicksight.create_analysis(
    AwsAccountId=AWS_ACCOUNT_ID,
    AnalysisId=f"{PROJECT_NAME}-ml-cost-analysis-{ENV}",
    Name=f"{PROJECT_NAME}-ml-cost-analysis-{ENV}",
    SourceEntity={
        "SourceTemplate": {
            "DataSetReferences": [
                {
                    "DataSetPlaceholder": "ml_costs",
                    "DataSetArn": f"arn:aws:quicksight:{AWS_REGION}:{AWS_ACCOUNT_ID}:dataset/{PROJECT_NAME}-ml-costs-{ENV}",
                }
            ],
            "Arn": QUICKSIGHT_TEMPLATE_ARN,  # Use a pre-built cost analysis template
        }
    },
    Permissions=[
        {
            "Principal": QUICKSIGHT_USER_ARN,
            "Actions": [
                "quicksight:DescribeAnalysis",
                "quicksight:UpdateAnalysis",
                "quicksight:DeleteAnalysis",
                "quicksight:QueryAnalysis",
                "quicksight:DescribeAnalysisPermissions",
                "quicksight:UpdateAnalysisPermissions",
            ],
        }
    ],
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)

# Publish dashboard from analysis
quicksight.create_dashboard(
    AwsAccountId=AWS_ACCOUNT_ID,
    DashboardId=f"{PROJECT_NAME}-ml-cost-dashboard-{ENV}",
    Name=f"{PROJECT_NAME}-ml-cost-dashboard-{ENV}",
    SourceEntity={
        "SourceTemplate": {
            "DataSetReferences": [
                {
                    "DataSetPlaceholder": "ml_costs",
                    "DataSetArn": f"arn:aws:quicksight:{AWS_REGION}:{AWS_ACCOUNT_ID}:dataset/{PROJECT_NAME}-ml-costs-{ENV}",
                }
            ],
            "Arn": QUICKSIGHT_TEMPLATE_ARN,
        }
    },
    Permissions=[
        {
            "Principal": QUICKSIGHT_USER_ARN,
            "Actions": [
                "quicksight:DescribeDashboard",
                "quicksight:ListDashboardVersions",
                "quicksight:UpdateDashboardPermissions",
                "quicksight:QueryDashboard",
                "quicksight:UpdateDashboard",
                "quicksight:DeleteDashboard",
                "quicksight:DescribeDashboardPermissions",
                "quicksight:UpdateDashboardPublishedVersion",
            ],
        }
    ],
    DashboardPublishOptions={
        "AdHocFilteringOption": {"AvailabilityStatus": "ENABLED"},
        "ExportToCSVOption": {"AvailabilityStatus": "ENABLED"},
        "SheetControlsOption": {"VisibilityState": "EXPANDED"},
    },
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)
print(f"QuickSight dashboard created: {PROJECT_NAME}-ml-cost-dashboard-{ENV}")
print(f"  URL: https://{AWS_REGION}.quicksight.aws.amazon.com/sn/dashboards/{PROJECT_NAME}-ml-cost-dashboard-{ENV}")
```

**Generate CSV chargeback report by business unit:**
```python
import csv
import io
from datetime import datetime, timedelta

ce = boto3.client("ce", region_name=AWS_REGION)
s3 = boto3.client("s3", region_name=AWS_REGION)

def generate_chargeback_report(cost_center_tag, services, project_name, env, bucket):
    """Generate CSV chargeback report grouped by business unit."""
    end_date = datetime.utcnow().strftime("%Y-%m-%d")
    start_date = (datetime.utcnow().replace(day=1)).strftime("%Y-%m-%d")

    # Query Cost Explorer by cost center tag
    response = ce.get_cost_and_usage(
        TimePeriod={"Start": start_date, "End": end_date},
        Granularity="MONTHLY",
        Metrics=["UnblendedCost", "UsageQuantity"],
        Filter={
            "Dimensions": {
                "Key": "SERVICE",
                "Values": [
                    "Amazon SageMaker", "Amazon Bedrock", "AWS Glue",
                    "AWS Lambda", "Amazon Simple Storage Service",
                ],
                "MatchOptions": ["EQUALS"],
            }
        },
        GroupBy=[
            {"Type": "TAG", "Key": cost_center_tag},
            {"Type": "DIMENSION", "Key": "SERVICE"},
        ],
    )

    # Build report rows
    rows = []
    for period in response["ResultsByTime"]:
        for group in period["Groups"]:
            tag_value = group["Keys"][0].split("$", 1)[-1] or "(untagged)"
            service = group["Keys"][1]
            cost = float(group["Metrics"]["UnblendedCost"]["Amount"])
            usage = float(group["Metrics"]["UsageQuantity"]["Amount"])
            rows.append({
                "CostCenter": tag_value,
                "Service": service,
                "MonthlySpend": round(cost, 2),
                "UsageQuantity": round(usage, 2),
                "Period": f"{start_date} to {end_date}",
            })

    # Write CSV
    output = io.StringIO()
    fieldnames = ["CostCenter", "Service", "MonthlySpend", "UsageQuantity", "Period"]
    writer = csv.DictWriter(output, fieldnames=fieldnames)
    writer.writeheader()
    writer.writerows(rows)

    # Add summary row
    total_cost = sum(r["MonthlySpend"] for r in rows)
    writer.writerow({
        "CostCenter": "TOTAL",
        "Service": "All Services",
        "MonthlySpend": round(total_cost, 2),
        "UsageQuantity": "",
        "Period": f"{start_date} to {end_date}",
    })

    # Upload to S3
    now = datetime.utcnow()
    s3_key = f"reports/{now.year}/{now.month:02d}/chargeback_{now.strftime('%Y%m%d')}.csv"
    s3.put_object(
        Bucket=bucket,
        Key=s3_key,
        Body=output.getvalue().encode("utf-8"),
        ContentType="text/csv",
        ServerSideEncryption="AES256",
        Tagging=f"Project={project_name}&Environment={env}",
    )
    print(f"Chargeback report uploaded: s3://{bucket}/{s3_key}")
    print(f"  Total ML spend: ${total_cost:.2f}")
    print(f"  Business units: {len(set(r['CostCenter'] for r in rows))}")
    return f"s3://{bucket}/{s3_key}"
```

**Auto-discovery Lambda for new endpoints:**
```python
import json

cloudwatch = boto3.client("cloudwatch")
sagemaker = boto3.client("sagemaker")
sns = boto3.client("sns")

def lambda_handler(event, context):
    """Auto-add CloudWatch dashboard widgets for new SageMaker endpoints/jobs."""
    detail_type = event.get("detail-type", "")
    detail = event.get("detail", {})

    dashboard_name = f"{PROJECT_NAME}-{DASHBOARD_NAME}-{ENV}"

    # Get current dashboard
    try:
        response = cloudwatch.get_dashboard(DashboardName=dashboard_name)
        dashboard_body = json.loads(response["DashboardBody"])
    except cloudwatch.exceptions.DashboardNotFoundError:
        print(f"Dashboard {dashboard_name} not found")
        return

    if "Endpoint" in detail_type:
        endpoint_name = detail.get("EndpointName", "")
        status = detail.get("EndpointStatus", "")
        if status != "InService":
            return

        # Add endpoint metrics widget
        new_widget = {
            "type": "metric",
            "x": 0, "y": 20, "width": 12, "height": 6,
            "properties": {
                "metrics": [
                    ["AWS/SageMaker", "Invocations", "EndpointName", endpoint_name,
                     "VariantName", "AllTraffic", {"id": "m1", "stat": "Sum"}],
                    ["AWS/SageMaker", "ModelLatency", "EndpointName", endpoint_name,
                     "VariantName", "AllTraffic", {"id": "m2", "stat": "Average"}],
                ],
                "view": "timeSeries",
                "title": f"Endpoint: {endpoint_name}",
                "period": 300,
                "region": AWS_REGION,
            },
        }
        dashboard_body["widgets"].append(new_widget)
        action = f"Added widgets for endpoint: {endpoint_name}"

    elif "Training Job" in detail_type:
        job_name = detail.get("TrainingJobName", "")
        status = detail.get("TrainingJobStatus", "")
        if status != "Completed":
            return

        # Add training cost annotation to spend trend
        job_details = sagemaker.describe_training_job(TrainingJobName=job_name)
        duration_sec = (job_details["TrainingEndTime"] - job_details["TrainingStartTime"]).total_seconds()
        action = f"Training job completed: {job_name} ({duration_sec/3600:.1f} hours)"

    else:
        return

    # Update dashboard
    cloudwatch.put_dashboard(
        DashboardName=dashboard_name,
        DashboardBody=json.dumps(dashboard_body),
    )

    # Notify
    sns.publish(
        TopicArn=SNS_TOPIC_ARN,
        Subject=f"[{ENV}] Dashboard Updated — {dashboard_name}",
        Message=action,
    )
    print(action)
    return {"statusCode": 200, "body": action}
```

**Schedule chargeback report with EventBridge:**
```python
events = boto3.client("events", region_name=AWS_REGION)
lambda_client = boto3.client("lambda", region_name=AWS_REGION)

# Map frequency to cron expression
schedule_map = {
    "daily": "cron(0 7 * * ? *)",
    "weekly": "cron(0 7 ? * MON *)",
    "monthly": "cron(0 7 1 * ? *)",
}
schedule_expression = schedule_map[REPORT_FREQUENCY]

rule_name = f"{PROJECT_NAME}-chargeback-schedule-{ENV}"

events.put_rule(
    Name=rule_name,
    ScheduleExpression=schedule_expression,
    State="ENABLED",
    Description=f"Generate {REPORT_FREQUENCY} ML chargeback report for {PROJECT_NAME} ({ENV})",
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)

events.put_targets(
    Rule=rule_name,
    Targets=[
        {
            "Id": "chargeback-reporter",
            "Arn": f"arn:aws:lambda:{AWS_REGION}:{AWS_ACCOUNT_ID}:function:{PROJECT_NAME}-chargeback-reporter-{ENV}",
            "Input": json.dumps({
                "cost_center_tag": COST_CENTER_TAG,
                "bucket": CHARGEBACK_BUCKET,
                "notification_email": NOTIFICATION_EMAIL,
            }),
        }
    ],
)
print(f"Chargeback schedule created: {rule_name}")
print(f"  Frequency: {REPORT_FREQUENCY} ({schedule_expression})")
```

---

## Integration Points

- **Upstream**: `finops/01` → Cost allocation tags and budget definitions; budget names follow `{PROJECT_NAME}-{service}-budget-{ENV}` convention for budget gauge widgets; Cost Explorer queries use the same tag keys (Team, CostCenter, ExperimentId) for chargeback attribution
- **Upstream**: `finops/04` → Inference cost optimization data; per-endpoint cost metrics (`EndpointHourlyCost` custom metric) used in cost-per-inference metric math calculations; auto-scaling configuration informs capacity-based cost estimates
- **Upstream**: `devops/12` → Bedrock invocation logging; token count metrics (`InputTokenCount`, `OutputTokenCount`) from CloudWatch `AWS/Bedrock` namespace used for Bedrock cost widgets; invocation logs in S3 feed QuickSight analysis
- **Upstream**: `devops/13` → Cost-per-inference dashboard data; custom CloudWatch metrics for per-endpoint and per-model cost tracking; metric math patterns for cost/invocation calculations
- **Downstream**: `enterprise/01` → Governance reporting; chargeback CSV reports and QuickSight dashboards provide cost visibility for organizational governance; budget utilization data supports SCP enforcement decisions
