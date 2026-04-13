<!-- Template Version: 1.0 | boto3: 1.35+ -->

# Template FinOps 03 — Savings Plans Analysis for ML Workloads

## Purpose
Generate a production-ready Savings Plans analysis and planning framework for ML infrastructure: Savings Plans utilization and coverage reports using the Savings Plans and Cost Explorer APIs, purchase recommendations filtered to SageMaker and compute usage types, right-sizing analysis for over-provisioned endpoints and training instances, a cost projection model comparing on-demand vs 1-year and 3-year savings plan options (no-upfront and partial-upfront), CloudWatch dashboards for utilization/coverage/savings visualization, and a utilization threshold alarm with SNS notification when utilization drops below a configurable percentage.

---

## Role Definition

You are an expert AWS FinOps engineer and ML infrastructure cost planning specialist with expertise in:
- AWS Savings Plans: Compute Savings Plans, SageMaker Savings Plans, plan types (no-upfront, partial-upfront, all-upfront), commitment terms (1-year, 3-year)
- Savings Plans API: `savingsplans.describe_savings_plans()`, `describe_savings_plans_offering_rates()`, plan lifecycle management
- Cost Explorer Savings Plans APIs: `ce.get_savings_plans_utilization()`, `ce.get_savings_plans_coverage()`, `ce.get_savings_plans_purchase_recommendation()`, utilization details and coverage reports
- Right-sizing recommendations: `ce.get_rightsizing_recommendation()`, instance type analysis, over-provisioned resource detection for SageMaker endpoints and training jobs
- Cost projection modeling: on-demand vs reserved pricing comparison, break-even analysis, commitment term optimization, workload profile analysis
- CloudWatch dashboards: `cloudwatch.put_dashboard()`, metric math for utilization percentages, coverage ratios, net savings calculations
- CloudWatch alarms: utilization threshold monitoring, SNS notification integration for FinOps team alerts
- SageMaker cost patterns: endpoint instance hours, training job instance hours, processing job costs, notebook instance costs, inference accelerator costs
- ML workload profiling: steady-state vs burst workloads, training cadence analysis, endpoint utilization patterns

Generate complete, production-deployable cost analysis and planning code.

---

## Context & Inputs

```
PROJECT_NAME:               [REQUIRED]
AWS_REGION:                 [REQUIRED]
AWS_ACCOUNT_ID:             [REQUIRED]
ENV:                        [REQUIRED - dev | stage | prod]

ANALYSIS_PERIOD:            [OPTIONAL: 30 - number of days to analyze for utilization and coverage]
                            Options: 30, 60, 90
                            Longer periods provide more stable recommendations but may
                            miss recent workload changes.

PLAN_TYPES:                 [OPTIONAL: compute,ml - comma-separated Savings Plan types to analyze]
                            Options: compute, ml
                            - compute: Compute Savings Plans (apply to EC2, Fargate, Lambda, SageMaker)
                            - ml: SageMaker Savings Plans (apply only to SageMaker usage)
                            Analyze both to determine optimal plan mix.

UTILIZATION_THRESHOLD:      [OPTIONAL: 80 - percentage below which utilization alarm triggers]
                            Savings Plans utilization below this threshold indicates
                            over-commitment or workload reduction. Range: 1-100.

WORKLOAD_PROFILE:           [OPTIONAL: none - JSON describing ML workload for cost projection]
                            Describes current ML infrastructure usage for projection modeling.
                            Example:
                            {
                              "endpoints": [
                                {"instance_type": "ml.g5.xlarge", "count": 2, "hours_per_day": 24},
                                {"instance_type": "ml.c5.xlarge", "count": 1, "hours_per_day": 12}
                              ],
                              "training": [
                                {"instance_type": "ml.p3.2xlarge", "hours_per_month": 200},
                                {"instance_type": "ml.g5.2xlarge", "hours_per_month": 150}
                              ]
                            }

NOTIFICATION_EMAIL:         [OPTIONAL: none - email for utilization alarm notifications]
                            Example: "finops-team@example.com"

SNS_TOPIC_ARN:              [OPTIONAL - existing SNS topic for alarm notifications]
                            If not provided, a new topic is created:
                            {PROJECT_NAME}-sp-utilization-{ENV}

RECOMMENDATION_TERM:        [OPTIONAL: ONE_YEAR - Savings Plans term for recommendations]
                            Options: ONE_YEAR, THREE_YEARS
                            THREE_YEARS offers deeper discounts but longer commitment.

RECOMMENDATION_PAYMENT:     [OPTIONAL: NO_UPFRONT - payment option for recommendations]
                            Options: NO_UPFRONT, PARTIAL_UPFRONT, ALL_UPFRONT
                            ALL_UPFRONT offers the deepest discount.

LOOKBACK_PERIOD:            [OPTIONAL: SIXTY_DAYS - lookback for purchase recommendations]
                            Options: SEVEN_DAYS, THIRTY_DAYS, SIXTY_DAYS
                            Longer lookback smooths out workload spikes.
```

---

## Task

Generate complete Savings Plans analysis and planning framework:

```
{PROJECT_NAME}-savings-plans/
├── analysis/
│   ├── utilization_report.py             # Savings Plans utilization report
│   ├── coverage_report.py                # Savings Plans coverage report
│   └── current_plans.py                  # Describe active Savings Plans
├── recommendations/
│   ├── purchase_recommendations.py       # Purchase recommendations filtered to ML usage
│   └── rightsizing_analysis.py           # Right-sizing for endpoints and training instances
├── projections/
│   ├── cost_projection_model.py          # On-demand vs 1yr/3yr cost comparison
│   └── workload_profiler.py              # Analyze workload profile for commitment sizing
├── monitoring/
│   ├── utilization_dashboard.py          # CloudWatch dashboard for SP metrics
│   ├── utilization_alarm.py              # Alarm when utilization drops below threshold
│   └── savings_tracker.py               # Track net savings over time
├── infrastructure/
│   ├── config.py                         # Central configuration
│   └── sns_topics.py                     # SNS topics for utilization alerts
├── run_setup.py                          # CLI orchestrator
└── requirements.txt
```

**config.py**: Central configuration dataclass with all parameters. Load from environment variables or CLI args. Validate required fields. Parse WORKLOAD_PROFILE from JSON string. Validate ANALYSIS_PERIOD is one of 30, 60, 90. Validate UTILIZATION_THRESHOLD is between 1 and 100. Construct resource names using `{PROJECT_NAME}-{component}-{ENV}` convention.

**current_plans.py**: Describe active Savings Plans:
- Call `savingsplans.describe_savings_plans()` to list all active plans
- Filter by plan type (Compute or SageMaker) based on PLAN_TYPES
- For each plan, display: plan ID, plan type, payment option, commitment ($/hr), term, start/end dates, state, utilization
- Call `savingsplans.describe_savings_plans_offering_rates()` to show applicable service/usage type rates
- Summarize total hourly commitment across all active plans
- Output structured JSON report of current plan portfolio

**utilization_report.py**: Generate Savings Plans utilization report:
- Call `ce.get_savings_plans_utilization()` with `TimePeriod` spanning ANALYSIS_PERIOD days
- Use `Granularity="DAILY"` for trend analysis
- Extract `TotalCommitment`, `UsedCommitment`, `UnusedCommitment`, `UtilizationPercentage` for each day
- Call `ce.get_savings_plans_utilization_details()` for per-plan breakdown
- Identify days with utilization below UTILIZATION_THRESHOLD
- Calculate average utilization, min utilization, and trend direction (improving/declining)
- Output daily utilization time series and summary statistics
- Publish utilization metrics to CloudWatch namespace `{PROJECT_NAME}/SavingsPlans`

**coverage_report.py**: Generate Savings Plans coverage report:
- Call `ce.get_savings_plans_coverage()` with `TimePeriod` spanning ANALYSIS_PERIOD days
- Use `Granularity="DAILY"` for trend analysis
- Group by `SERVICE` to show coverage per ML service (SageMaker, EC2, Lambda)
- Extract `CoveragePercentage`, `SpendCoveredBySavingsPlans`, `OnDemandCost` for each period
- Identify services with low coverage (high on-demand spend) as candidates for additional plans
- Calculate total covered vs uncovered spend
- Output coverage summary with per-service breakdown

**purchase_recommendations.py**: Get Savings Plans purchase recommendations filtered to ML:
- Call `ce.get_savings_plans_purchase_recommendation()` with:
  - `SavingsPlanType`: `COMPUTE_SP` or `SAGEMAKER_SP` based on PLAN_TYPES
  - `TermInYears`: from RECOMMENDATION_TERM
  - `PaymentOption`: from RECOMMENDATION_PAYMENT
  - `LookbackPeriodInDays`: from LOOKBACK_PERIOD
- Parse recommendation details: recommended hourly commitment, estimated savings percentage, estimated monthly savings, estimated on-demand cost, estimated SP cost
- For each recommendation, show the break-even point (months to recoup upfront cost)
- Compare recommendations across plan types (Compute vs SageMaker SP)
- Output ranked recommendations by estimated savings percentage

**rightsizing_analysis.py**: Right-sizing analysis for ML resources:
- Call `ce.get_rightsizing_recommendation()` with `Service="AmazonSageMaker"`
- Parse recommendations for over-provisioned resources:
  - Current instance type, recommended instance type, estimated savings
  - Resource details (endpoint name, training job pattern)
- Categorize recommendations by resource type: endpoints (steady-state), training (burst)
- For endpoints: recommend downsizing if average CPU/GPU utilization is low
- For training: recommend right-sizing based on actual resource utilization during jobs
- Calculate total potential savings from all right-sizing recommendations
- Output actionable recommendations with estimated monthly savings per resource

**cost_projection_model.py**: Cost projection comparing on-demand vs Savings Plans:
- If WORKLOAD_PROFILE provided, use it; otherwise estimate from Cost Explorer historical data
- For each instance type in the workload profile, calculate:
  - Monthly on-demand cost = hours × on-demand rate
  - Monthly 1-year no-upfront SP cost = hours × SP rate (typically 20-30% savings)
  - Monthly 1-year partial-upfront SP cost = (upfront / 12) + (hours × reduced rate)
  - Monthly 3-year no-upfront SP cost = hours × SP rate (typically 40-50% savings)
  - Monthly 3-year partial-upfront SP cost = (upfront / 36) + (hours × reduced rate)
- Aggregate across all instance types for total monthly cost per option
- Calculate annual and multi-year total cost of ownership for each option
- Determine optimal commitment level (hourly commitment amount) based on workload profile
- Output comparison table and recommendation with projected savings

**workload_profiler.py**: Analyze workload profile for commitment sizing:
- Query Cost Explorer for SageMaker usage over ANALYSIS_PERIOD days
- Group by `USAGE_TYPE` to identify instance type usage patterns
- Calculate hourly usage patterns: peak hours, off-peak hours, average utilization
- Identify steady-state baseline (minimum consistent usage suitable for SP commitment)
- Identify burst usage (variable usage better served by on-demand)
- Recommend SP commitment level = steady-state baseline hourly cost
- Output workload profile JSON compatible with WORKLOAD_PROFILE parameter format

**utilization_dashboard.py**: CloudWatch dashboard for Savings Plans metrics:
- Create dashboard using `cloudwatch.put_dashboard()` with widgets:
  - Savings Plans utilization percentage over time (line chart, target line at UTILIZATION_THRESHOLD)
  - Savings Plans coverage percentage by service (stacked area chart)
  - Net savings: SP cost vs equivalent on-demand cost (bar chart)
  - Unused commitment amount over time (line chart)
  - Total hourly commitment vs used commitment (dual-axis line chart)
  - Current plan summary (text widget with plan details)
- Dashboard name: `{PROJECT_NAME}-savings-plans-{ENV}`
- Use metric math for derived metrics (utilization %, net savings)

**utilization_alarm.py**: CloudWatch alarm for low utilization:
- Create CloudWatch alarm using `cloudwatch.put_metric_alarm()`:
  - Alarm name: `{PROJECT_NAME}-sp-utilization-low-{ENV}`
  - Metric: custom metric published by `utilization_report.py` or Savings Plans utilization metric
  - Threshold: UTILIZATION_THRESHOLD (default 80%)
  - Comparison: `LessThanThreshold`
  - Evaluation periods: 3 consecutive periods (avoid transient dips)
  - Period: 86400 seconds (1 day)
  - Alarm action: SNS topic for FinOps team notification
- SNS message includes: current utilization %, threshold, unused commitment amount, recommendation to review workload or modify plans

**savings_tracker.py**: Track net savings over time:
- Query `ce.get_savings_plans_utilization()` for the last 90 days
- Calculate daily net savings: `OnDemandCostEquivalent - SPCost`
- Aggregate into weekly and monthly savings totals
- Calculate cumulative savings since plan activation
- Compare actual savings vs projected savings from purchase recommendation
- Publish savings metrics to CloudWatch: `NetSavings`, `CumulativeSavings`, `SavingsRate`
- Output savings report as JSON

**sns_topics.py**: Create SNS topics for utilization alerts:
- Topic name: `{PROJECT_NAME}-sp-utilization-{ENV}`
- Subscribe NOTIFICATION_EMAIL if provided
- Set topic policy allowing CloudWatch Alarms to publish

**run_setup.py**: CLI orchestrator that runs setup steps in order:
1. Load and validate configuration
2. Create SNS topics for utilization alerts
3. Describe current Savings Plans portfolio
4. Generate utilization report
5. Generate coverage report
6. Get purchase recommendations
7. Run right-sizing analysis
8. Generate cost projection model
9. Create CloudWatch dashboard
10. Create utilization alarm
11. Print summary with current plans, utilization %, coverage %, top recommendations, and dashboard URL

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**Savings Plans Types:** Two plan types are relevant for ML workloads. Compute Savings Plans apply to EC2, Fargate, Lambda, and SageMaker usage — they offer flexibility across services and instance families. SageMaker Savings Plans apply only to SageMaker usage (training, inference, processing, notebooks) — they offer deeper discounts (up to 64%) but are scoped to SageMaker. Analyze both types to determine the optimal mix. Compute SP is better for diverse workloads; SageMaker SP is better for SageMaker-heavy workloads.

**Utilization vs Coverage:** Utilization measures how much of your committed spend is actually used (target: >80%). Low utilization means you're paying for unused commitment. Coverage measures what percentage of your eligible spend is covered by Savings Plans (target: >70%). Low coverage means you're paying on-demand rates for eligible usage. Optimize both: increase coverage to reduce on-demand spend, maintain high utilization to avoid waste.

**Purchase Recommendations:** Cost Explorer provides purchase recommendations based on historical usage. Use LOOKBACK_PERIOD of 60 days for stable recommendations. Recommendations include estimated savings percentage, monthly savings amount, and hourly commitment. Always compare Compute SP vs SageMaker SP recommendations — sometimes a mix of both is optimal. Consider workload growth when sizing commitments — under-commit slightly to maintain high utilization.

**Right-Sizing:** Right-sizing should be done before purchasing Savings Plans. Over-provisioned resources inflate the baseline, leading to over-commitment. Use `ce.get_rightsizing_recommendation()` to identify SageMaker endpoints and training instances that can be downsized. Right-sizing recommendations consider CPU, memory, and GPU utilization metrics. Apply right-sizing first, then re-analyze for SP purchase.

**Cost Projection:** The cost projection model compares five pricing options: on-demand, 1-year no-upfront, 1-year partial-upfront, 3-year no-upfront, and 3-year partial-upfront. Partial-upfront plans require an upfront payment but offer lower hourly rates. All-upfront plans offer the deepest discount but require full payment at purchase. For ML workloads with uncertain growth, start with 1-year no-upfront plans to maintain flexibility. For stable production workloads, 3-year partial-upfront offers the best savings.

**Security:** Cost Explorer API requires `ce:GetSavingsPlansUtilization`, `ce:GetSavingsPlansCoverage`, `ce:GetSavingsPlansPurchaseRecommendation`, `ce:GetRightsizingRecommendation` permissions. Savings Plans API requires `savingsplans:DescribeSavingsPlans`, `savingsplans:DescribeSavingsPlansOfferingRates` permissions. CloudWatch dashboard and alarm creation requires `cloudwatch:PutDashboard`, `cloudwatch:PutMetricAlarm`, `cloudwatch:PutMetricData` permissions. SNS topic creation requires `sns:CreateTopic`, `sns:Subscribe` permissions. Restrict Savings Plans purchase permissions (`savingsplans:CreateSavingsPlan`) to FinOps leads only.

**Cost:** Cost Explorer API: $0.01 per paginated API request. Savings Plans API: free. CloudWatch dashboards: $3/month per dashboard. CloudWatch alarms: $0.10/month per standard alarm. CloudWatch custom metrics: $0.30/month per metric. Optimize API calls by caching results — SP utilization data changes daily, not hourly.

**Naming:** All resources follow `{PROJECT_NAME}-{component}-{ENV}` convention:
- CloudWatch dashboard: `{PROJECT_NAME}-savings-plans-{ENV}`
- CloudWatch alarm: `{PROJECT_NAME}-sp-utilization-low-{ENV}`
- SNS topic: `{PROJECT_NAME}-sp-utilization-{ENV}`
- CloudWatch namespace: `{PROJECT_NAME}/SavingsPlans`

---

## Code Scaffolding Hints

**Describe active Savings Plans:**
```python
import boto3
import json
from datetime import datetime, timedelta

savingsplans = boto3.client("savingsplans", region_name=AWS_REGION)

def describe_current_plans(plan_types=None):
    """List all active Savings Plans with utilization details."""
    filters = [{"name": "state", "values": ["active"]}]
    if plan_types:
        sp_type_map = {
            "compute": "Compute",
            "ml": "SageMaker",
        }
        type_values = [sp_type_map[t] for t in plan_types if t in sp_type_map]
        if type_values:
            filters.append({"name": "savings-plan-type", "values": type_values})

    response = savingsplans.describe_savings_plans(
        filters=filters,
    )

    plans = response.get("savingsPlans", [])
    total_commitment = 0.0
    for plan in plans:
        commitment = float(plan.get("commitment", "0"))
        total_commitment += commitment
        print(f"  Plan ID: {plan['savingsPlanId']}")
        print(f"    Type: {plan['savingsPlanType']}")
        print(f"    Payment: {plan['paymentOption']}")
        print(f"    Commitment: ${commitment}/hr")
        print(f"    Term: {plan.get('termDurationInSeconds', 0) // (365*24*3600)} year(s)")
        print(f"    State: {plan['state']}")
        print(f"    Start: {plan.get('start', 'N/A')}")
        print(f"    End: {plan.get('end', 'N/A')}")
        print()

    print(f"Total hourly commitment: ${total_commitment:.4f}/hr")
    print(f"Total monthly commitment: ${total_commitment * 730:.2f}/month")
    return plans
```

**Get Savings Plans utilization report:**
```python
ce = boto3.client("ce", region_name=AWS_REGION)

def get_utilization_report(analysis_period=30):
    """Generate Savings Plans utilization report over the analysis period."""
    end_date = datetime.utcnow().strftime("%Y-%m-%d")
    start_date = (datetime.utcnow() - timedelta(days=analysis_period)).strftime("%Y-%m-%d")

    response = ce.get_savings_plans_utilization(
        TimePeriod={"Start": start_date, "End": end_date},
        Granularity="DAILY",
    )

    total = response.get("Total", {})
    utilization = total.get("Utilization", {})
    print(f"Savings Plans Utilization Summary ({analysis_period} days):")
    print(f"  Total Commitment:  ${float(utilization.get('TotalCommitment', '0')):.2f}")
    print(f"  Used Commitment:   ${float(utilization.get('UsedCommitment', '0')):.2f}")
    print(f"  Unused Commitment: ${float(utilization.get('UnusedCommitment', '0')):.2f}")
    print(f"  Utilization:       {utilization.get('UtilizationPercentage', '0')}%")

    # Daily breakdown for trend analysis
    daily_data = []
    for period in response.get("SavingsPlansUtilizationsByTime", []):
        day_util = period.get("Utilization", {})
        daily_data.append({
            "date": period["TimePeriod"]["Start"],
            "utilization_pct": float(day_util.get("UtilizationPercentage", "0")),
            "total_commitment": float(day_util.get("TotalCommitment", "0")),
            "used_commitment": float(day_util.get("UsedCommitment", "0")),
            "unused_commitment": float(day_util.get("UnusedCommitment", "0")),
        })

    # Identify low-utilization days
    low_util_days = [d for d in daily_data if d["utilization_pct"] < UTILIZATION_THRESHOLD]
    if low_util_days:
        print(f"\n  WARNING: {len(low_util_days)} days below {UTILIZATION_THRESHOLD}% threshold")

    return {"total": utilization, "daily": daily_data}
```

**Get Savings Plans coverage report:**
```python
def get_coverage_report(analysis_period=30):
    """Generate Savings Plans coverage report by service."""
    end_date = datetime.utcnow().strftime("%Y-%m-%d")
    start_date = (datetime.utcnow() - timedelta(days=analysis_period)).strftime("%Y-%m-%d")

    response = ce.get_savings_plans_coverage(
        TimePeriod={"Start": start_date, "End": end_date},
        Granularity="MONTHLY",
        GroupBy=[{"Type": "DIMENSION", "Key": "SERVICE"}],
        Filter={
            "Dimensions": {
                "Key": "SERVICE",
                "Values": [
                    "Amazon SageMaker",
                    "Amazon Elastic Compute Cloud - Compute",
                    "AWS Lambda",
                ],
                "MatchOptions": ["EQUALS"],
            }
        },
    )

    print(f"Savings Plans Coverage Summary ({analysis_period} days):")
    for period in response.get("SavingsPlansCoverages", []):
        coverage = period.get("Coverage", {})
        attrs = period.get("Attributes", {})
        service = attrs.get("SERVICE", "Unknown")
        pct = coverage.get("CoveragePercentage", "0")
        covered = float(coverage.get("SpendCoveredBySavingsPlans", "0"))
        on_demand = float(coverage.get("OnDemandCost", "0"))
        print(f"  {service}:")
        print(f"    Coverage: {pct}%")
        print(f"    Covered spend: ${covered:.2f}")
        print(f"    On-demand spend: ${on_demand:.2f}")

    return response
```

**Get Savings Plans purchase recommendations:**
```python
def get_purchase_recommendations(
    plan_type="COMPUTE_SP",
    term="ONE_YEAR",
    payment_option="NO_UPFRONT",
    lookback="SIXTY_DAYS",
):
    """Get Savings Plans purchase recommendations filtered to ML usage."""
    response = ce.get_savings_plans_purchase_recommendation(
        SavingsPlanType=plan_type,
        TermInYears=term,
        PaymentOption=payment_option,
        LookbackPeriodInDays=lookback,
    )

    meta = response.get("SavingsPlansPurchaseRecommendation", {})
    summary = meta.get("SavingsPlansPurchaseRecommendationSummary", {})
    details = meta.get("SavingsPlansPurchaseRecommendationDetails", [])

    print(f"Purchase Recommendation ({plan_type}, {term}, {payment_option}):")
    print(f"  Estimated savings: {summary.get('EstimatedSavingsPercentage', '0')}%")
    print(f"  Estimated monthly savings: ${float(summary.get('EstimatedMonthlySavingsAmount', '0')):.2f}")
    print(f"  Recommended hourly commitment: ${float(summary.get('HourlyCommitmentToPurchase', '0')):.4f}/hr")
    print(f"  Current on-demand spend: ${float(summary.get('CurrentOnDemandSpend', '0')):.2f}/month")
    print(f"  Estimated SP cost: ${float(summary.get('EstimatedSavingsAmount', '0')):.2f}/month")

    for detail in details[:5]:
        print(f"\n  Detail:")
        print(f"    Hourly commitment: ${float(detail.get('HourlyCommitmentToPurchase', '0')):.4f}/hr")
        print(f"    Estimated savings: {detail.get('EstimatedSavingsPercentage', '0')}%")
        print(f"    Estimated on-demand cost: ${float(detail.get('EstimatedOnDemandCost', '0')):.2f}")
        print(f"    Estimated SP cost: ${float(detail.get('EstimatedSPCost', '0')):.2f}")
        print(f"    Upfront cost: ${float(detail.get('UpfrontCost', '0')):.2f}")

    return {"summary": summary, "details": details}
```

**Get right-sizing recommendations for SageMaker:**
```python
def get_rightsizing_recommendations():
    """Get right-sizing recommendations for SageMaker resources."""
    response = ce.get_rightsizing_recommendation(
        Service="AmazonSageMaker",
        Configuration={
            "RecommendationTarget": "SAME_INSTANCE_FAMILY",
            "BenefitsConsidered": True,
        },
    )

    summary = response.get("Summary", {})
    recommendations = response.get("RightsizingRecommendations", [])

    print(f"Right-Sizing Recommendations for SageMaker:")
    print(f"  Total recommendations: {summary.get('TotalRecommendationCount', '0')}")
    print(f"  Estimated monthly savings: ${float(summary.get('EstimatedTotalMonthlySavingsAmount', '0')):.2f}")
    print(f"  Savings currency: {summary.get('SavingsCurrencyCode', 'USD')}")

    for rec in recommendations[:10]:
        current = rec.get("CurrentInstance", {})
        action = rec.get("RightsizingType", "Unknown")
        tags = current.get("Tags", [])
        resource_id = current.get("ResourceId", "Unknown")
        monthly_cost = float(current.get("MonthlyCost", "0"))

        print(f"\n  Resource: {resource_id}")
        print(f"    Action: {action}")
        print(f"    Current monthly cost: ${monthly_cost:.2f}")

        if action == "MODIFY":
            targets = rec.get("ModifyRecommendationDetail", {}).get("TargetInstances", [])
            for target in targets:
                est_savings = float(target.get("EstimatedMonthlySavings", "0"))
                new_cost = float(target.get("EstimatedMonthlyCost", "0"))
                instance_type = target.get("ResourceDetails", {}).get(
                    "EC2ResourceDetails", {}
                ).get("InstanceType", "Unknown")
                print(f"    Recommended: {instance_type}")
                print(f"    New monthly cost: ${new_cost:.2f}")
                print(f"    Estimated savings: ${est_savings:.2f}/month")
        elif action == "TERMINATE":
            print(f"    Recommendation: Terminate (unused resource)")
            print(f"    Estimated savings: ${monthly_cost:.2f}/month")

    return {"summary": summary, "recommendations": recommendations}
```

**Cost projection model (on-demand vs Savings Plans):**
```python
def generate_cost_projection(workload_profile):
    """Compare on-demand vs Savings Plans costs for a workload profile."""
    # Savings Plans discount rates (approximate, varies by instance type)
    discount_rates = {
        "1yr_no_upfront": 0.25,       # ~25% savings
        "1yr_partial_upfront": 0.30,   # ~30% savings
        "3yr_no_upfront": 0.40,        # ~40% savings
        "3yr_partial_upfront": 0.50,   # ~50% savings
    }

    total_on_demand_monthly = 0.0
    projections = []

    # Process endpoints (steady-state)
    for endpoint in workload_profile.get("endpoints", []):
        instance_type = endpoint["instance_type"]
        count = endpoint["count"]
        hours_per_day = endpoint["hours_per_day"]
        monthly_hours = hours_per_day * 30 * count

        # Get on-demand rate from Pricing API (simplified)
        on_demand_rate = get_sagemaker_on_demand_rate(instance_type)
        if on_demand_rate is None:
            continue

        monthly_cost = monthly_hours * on_demand_rate
        total_on_demand_monthly += monthly_cost

        projections.append({
            "resource_type": "endpoint",
            "instance_type": instance_type,
            "monthly_hours": monthly_hours,
            "on_demand_rate": on_demand_rate,
            "on_demand_monthly": monthly_cost,
        })

    # Process training jobs (burst)
    for training in workload_profile.get("training", []):
        instance_type = training["instance_type"]
        hours_per_month = training["hours_per_month"]

        on_demand_rate = get_sagemaker_on_demand_rate(instance_type)
        if on_demand_rate is None:
            continue

        monthly_cost = hours_per_month * on_demand_rate
        total_on_demand_monthly += monthly_cost

        projections.append({
            "resource_type": "training",
            "instance_type": instance_type,
            "monthly_hours": hours_per_month,
            "on_demand_rate": on_demand_rate,
            "on_demand_monthly": monthly_cost,
        })

    # Calculate SP costs for each option
    comparison = {"on_demand_monthly": round(total_on_demand_monthly, 2)}
    for option, discount in discount_rates.items():
        sp_monthly = total_on_demand_monthly * (1 - discount)
        savings_monthly = total_on_demand_monthly - sp_monthly
        comparison[option] = {
            "monthly_cost": round(sp_monthly, 2),
            "monthly_savings": round(savings_monthly, 2),
            "annual_savings": round(savings_monthly * 12, 2),
            "discount_pct": round(discount * 100, 1),
        }

    print(f"\nCost Projection Summary:")
    print(f"  On-demand monthly: ${comparison['on_demand_monthly']:.2f}")
    for option in discount_rates:
        opt = comparison[option]
        print(f"  {option}: ${opt['monthly_cost']:.2f}/mo "
              f"(save ${opt['monthly_savings']:.2f}/mo, {opt['discount_pct']}%)")

    return comparison


def get_sagemaker_on_demand_rate(instance_type):
    """Get SageMaker on-demand hourly rate from Pricing API."""
    pricing = boto3.client("pricing", region_name="us-east-1")
    try:
        response = pricing.get_products(
            ServiceCode="AmazonSageMaker",
            Filters=[
                {"Type": "TERM_MATCH", "Field": "instanceType", "Value": instance_type},
                {"Type": "TERM_MATCH", "Field": "location", "Value": "US East (N. Virginia)"},
                {"Type": "TERM_MATCH", "Field": "component", "Value": "Hosting Instance"},
            ],
            MaxResults=5,
        )
        for price_item in response.get("PriceList", []):
            item = json.loads(price_item)
            terms = item.get("terms", {}).get("OnDemand", {})
            for term in terms.values():
                for dimension in term.get("priceDimensions", {}).values():
                    price = float(dimension["pricePerUnit"]["USD"])
                    if price > 0:
                        return price
    except Exception as e:
        print(f"  Warning: Could not get pricing for {instance_type}: {e}")
    return None
```

**CloudWatch dashboard for Savings Plans metrics:**
```python
cloudwatch = boto3.client("cloudwatch", region_name=AWS_REGION)

def create_utilization_dashboard(project_name, env):
    """Create CloudWatch dashboard for Savings Plans metrics."""
    dashboard_name = f"{project_name}-savings-plans-{env}"
    namespace = f"{project_name}/SavingsPlans"

    dashboard_body = {
        "widgets": [
            {
                "type": "metric",
                "x": 0, "y": 0, "width": 12, "height": 6,
                "properties": {
                    "title": "Savings Plans Utilization %",
                    "metrics": [
                        [namespace, "UtilizationPercentage", "Environment", env],
                    ],
                    "period": 86400,
                    "stat": "Average",
                    "region": AWS_REGION,
                    "yAxis": {"left": {"min": 0, "max": 100}},
                    "annotations": {
                        "horizontal": [
                            {"label": "Threshold", "value": UTILIZATION_THRESHOLD, "color": "#d62728"}
                        ]
                    },
                },
            },
            {
                "type": "metric",
                "x": 12, "y": 0, "width": 12, "height": 6,
                "properties": {
                    "title": "Coverage % by Service",
                    "metrics": [
                        [namespace, "CoveragePercentage", "Service", "SageMaker", "Environment", env],
                        [namespace, "CoveragePercentage", "Service", "EC2", "Environment", env],
                        [namespace, "CoveragePercentage", "Service", "Lambda", "Environment", env],
                    ],
                    "period": 86400,
                    "stat": "Average",
                    "region": AWS_REGION,
                    "yAxis": {"left": {"min": 0, "max": 100}},
                },
            },
            {
                "type": "metric",
                "x": 0, "y": 6, "width": 12, "height": 6,
                "properties": {
                    "title": "Net Savings ($)",
                    "metrics": [
                        [namespace, "NetSavings", "Environment", env],
                        [namespace, "CumulativeSavings", "Environment", env],
                    ],
                    "period": 86400,
                    "stat": "Sum",
                    "region": AWS_REGION,
                },
            },
            {
                "type": "metric",
                "x": 12, "y": 6, "width": 12, "height": 6,
                "properties": {
                    "title": "Commitment: Used vs Unused ($)",
                    "metrics": [
                        [namespace, "UsedCommitment", "Environment", env],
                        [namespace, "UnusedCommitment", "Environment", env],
                    ],
                    "period": 86400,
                    "stat": "Sum",
                    "region": AWS_REGION,
                    "view": "timeSeries",
                    "stacked": True,
                },
            },
        ],
    }

    cloudwatch.put_dashboard(
        DashboardName=dashboard_name,
        DashboardBody=json.dumps(dashboard_body),
    )
    print(f"Dashboard created: {dashboard_name}")
    return dashboard_name
```

**CloudWatch alarm for low Savings Plans utilization:**
```python
sns = boto3.client("sns", region_name=AWS_REGION)

def create_utilization_alarm(project_name, env, threshold, sns_topic_arn):
    """Create alarm when SP utilization drops below threshold."""
    alarm_name = f"{project_name}-sp-utilization-low-{env}"
    namespace = f"{project_name}/SavingsPlans"

    cloudwatch.put_metric_alarm(
        AlarmName=alarm_name,
        AlarmDescription=(
            f"Savings Plans utilization dropped below {threshold}%. "
            f"Review workload changes or consider modifying plan commitments."
        ),
        Namespace=namespace,
        MetricName="UtilizationPercentage",
        Dimensions=[{"Name": "Environment", "Value": env}],
        Statistic="Average",
        Period=86400,  # 1 day
        EvaluationPeriods=3,  # 3 consecutive days below threshold
        Threshold=float(threshold),
        ComparisonOperator="LessThanThreshold",
        TreatMissingData="missing",
        AlarmActions=[sns_topic_arn],
        OKActions=[sns_topic_arn],
        Tags=[
            {"Key": "Project", "Value": project_name},
            {"Key": "Environment", "Value": env},
        ],
    )
    print(f"Utilization alarm created: {alarm_name} (threshold: {threshold}%)")
    return alarm_name
```

**Publish Savings Plans metrics to CloudWatch:**
```python
def publish_sp_metrics(project_name, env, utilization_data):
    """Publish Savings Plans utilization and savings metrics to CloudWatch."""
    namespace = f"{project_name}/SavingsPlans"

    metric_data = [
        {
            "MetricName": "UtilizationPercentage",
            "Dimensions": [{"Name": "Environment", "Value": env}],
            "Value": float(utilization_data.get("utilization_pct", 0)),
            "Unit": "Percent",
        },
        {
            "MetricName": "UsedCommitment",
            "Dimensions": [{"Name": "Environment", "Value": env}],
            "Value": float(utilization_data.get("used_commitment", 0)),
            "Unit": "None",
        },
        {
            "MetricName": "UnusedCommitment",
            "Dimensions": [{"Name": "Environment", "Value": env}],
            "Value": float(utilization_data.get("unused_commitment", 0)),
            "Unit": "None",
        },
    ]

    # Add net savings if available
    if "net_savings" in utilization_data:
        metric_data.append({
            "MetricName": "NetSavings",
            "Dimensions": [{"Name": "Environment", "Value": env}],
            "Value": float(utilization_data["net_savings"]),
            "Unit": "None",
        })

    cloudwatch.put_metric_data(
        Namespace=namespace,
        MetricData=metric_data,
    )
    print(f"Published {len(metric_data)} metrics to {namespace}")
```

---

## Integration Points

- **Upstream**: `finops/01` → Cost allocation tags and Cost Explorer data provide the spend baseline for Savings Plans analysis and workload profiling
- **Upstream**: `devops/04` → IAM roles with Cost Explorer, Savings Plans, and CloudWatch permissions
- **Downstream**: `finops/05` → FinOps dashboards consume Savings Plans utilization, coverage, and savings metrics for executive reporting
- **Downstream**: `enterprise/01` → SCPs can enforce Savings Plans purchase approval workflows and restrict plan modifications to authorized roles
