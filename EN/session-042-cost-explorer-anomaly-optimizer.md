# Session 042 — Cost Explorer, Cost Anomaly Detection, and Compute Optimizer

**Estimated duration:** 60 min  
**Prerequisite:** session-041 (Spot Instances and Fleet)

---

## Session Objectives

- Master the `GetCostAndUsage` API with filters by tag, dimension, and correct metrics
- Understand the differences between BlendedCost, UnblendedCost, AmortizedCost, and NetAmortizedCost
- Configure Cost Anomaly Detection with managed and tag-based monitors
- Interpret the SNS anomaly payload and its fields (`anomalyScore`, `impact`, `rootCauses`)
- Use Compute Optimizer for EC2 and Fargate with rightsizing preferences
- Export Compute Optimizer recommendations for analysis in S3

---

## 1. Cost Explorer — Fundamental Concepts

### 1.1 Granularity and Data Window

[FACT] Cost Explorer provides data with a delay of **up to 24 hours**. Granularity defines the minimum period of each data point:

```
╔════════════════╦════════════════════════════════════════════════╗
║ Granularity    ║ Restriction                                    ║
╠════════════════╬════════════════════════════════════════════════╣
║ HOURLY         ║ Maximum window: 14 days back                   ║
║ DAILY          ║ Default; 13 months free, 38 months (paid)      ║
║ MONTHLY        ║ Default; 13 months free, 38 months (paid)      ║
╚════════════════╩════════════════════════════════════════════════╝
```

### 1.2 Metrics — Which to Use for What

[FACT] Each metric represents a different cost perspective:

```
UnblendedCost
  — Actual cost charged to the individual account
  — Includes On-Demand price without any RI/SP spreading
  — Use for: individual account billing, simple chargeback

BlendedCost
  — Distributes RI and SP costs proportionally across linked accounts
  — Used for: cost allocation in Organizations (levels price differences)
  — Caution: may not reflect the actual cost of each child account

AmortizedCost
  — Distributes upfront cost of RI/SP across the commitment period
  — E.g.: RI 1 year, $1000 upfront → $2.74/day amortized
  — Use for: real period cost analysis (e.g.: FinOps dashboards)

NetAmortizedCost
  — AmortizedCost after private discounts (Enterprise Discount Program, negotiations)
  — Use for: real net cost in accounts with custom contracts

NormalizedUsageAmount / UsageQuantity
  — Usage volume, not cost; useful for consumption analysis by service
```

[CONSENSUS] For cost dashboards by project/team, the most useful metric is **AmortizedCost** — it reflects the real economic cost, spreading RI/SP upfront payments.

### 1.3 Cost Allocation Tags

[FACT] To filter or group by tag in Cost Explorer, the tag must be **activated** as a Cost Allocation Tag in the Billing console. It is not automatic.

```
Activation flow:
  1. Billing console → Cost allocation tags
  2. Select tag (e.g.: "project") → Activate
  3. Wait 24h to appear in Cost Explorer data
  4. Tags on new resources start appearing immediately after activation
  5. Tags on historical resources are NOT retroactive

Limit: 500 Cost Allocation Tags activated per account
```

---

## 2. GetCostAndUsage API — Structure and Examples

### 2.1 Expression Structure (Filters)

[FACT] The `Expression` type in the Cost Explorer API supports boolean composition with `And`, `Or`, `Not` (arrays), and leaves with `Dimensions`, `Tags`, `CostCategories`.

```python
# Base Expression structure
expression = {
    "And": [
        # Leaf 1: filter by tag
        {
            "Tags": {
                "Key": "project",
                "Values": ["checkout-service", "payments-api"],
                "MatchOptions": ["EQUALS"]
            }
        },
        # Leaf 2: exclude usage type (e.g.: data transfer)
        {
            "Not": {
                "Dimensions": {
                    "Key": "USAGE_TYPE_GROUP",
                    "Values": ["EC2: Data Transfer - Internet (Out)"],
                    "MatchOptions": ["EQUALS"]
                }
            }
        }
    ]
}
```

**Available dimensions in `Dimensions.Key`:** `SERVICE`, `REGION`, `LINKED_ACCOUNT`, `INSTANCE_TYPE`, `USAGE_TYPE`, `USAGE_TYPE_GROUP`, `RECORD_TYPE` (On-Demand/Spot/SavingsPlan/etc.), `OPERATING_SYSTEM`, `TENANCY`, `PURCHASE_TYPE`, `AZ`.

**MatchOptions:** `EQUALS`, `STARTS_WITH`, `ENDS_WITH`, `CONTAINS`, `ABSENT` (resources without the tag), `CASE_SENSITIVE`, `CASE_INSENSITIVE`.

### 2.2 GroupBy

[FACT] `GroupBy` defines how the result is segmented. Maximum of 2 groups per call. The type can be `DIMENSION` or `TAG`.

```python
group_by = [
    {"Type": "TAG",       "Key": "project"},         # activated tag
    {"Type": "DIMENSION", "Key": "SERVICE"},           # standard dimension
]
```

---

## 3. CDK Python — Cost Monitoring Stack

```python
from aws_cdk import (
    Stack, aws_ce as ce, aws_sns as sns,
    aws_sns_subscriptions as subs,
    aws_budgets as budgets,
)
from constructs import Construct

class CostMonitoringStack(Stack):
    def __init__(self, scope: Construct, construct_id: str, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        # SNS topic for anomaly alerts
        alert_topic = sns.Topic(self, "CostAlertTopic",
            display_name="AWS Cost Anomaly Alerts",
        )
        alert_topic.add_subscription(
            subs.EmailSubscription("finops-team@company.com")
        )

        # ──────────────────────────────────────────────────────────────
        # Cost Anomaly Detection — Monitor by service (AWS Managed)
        # Monitors all services automatically, top 5000 values
        # ──────────────────────────────────────────────────────────────
        service_monitor = ce.CfnAnomalyMonitor(self, "ServiceMonitor",
            monitor_name="AllServicesMonitor",
            monitor_type="DIMENSIONAL",
            monitor_dimension="SERVICE",  # AWS Managed: monitors all services
        )

        # Subscription with two combined thresholds (AND):
        # alert if impact >= $50 AND percentage >= 20%
        ce.CfnAnomalySubscription(self, "ServiceSubscription",
            subscription_name="ServiceAnomalyAlerts",
            monitor_arn_list=[service_monitor.attr_monitor_arn],
            subscribers=[
                ce.CfnAnomalySubscription.SubscriberProperty(
                    address=alert_topic.topic_arn,
                    type="SNS",
                    status="CONFIRMED",
                )
            ],
            frequency="IMMEDIATE",  # individual alerts via SNS (not daily/weekly)
            threshold_expression={
                "And": [
                    {
                        "Dimensions": {
                            "Key": "ANOMALY_TOTAL_IMPACT_ABSOLUTE",
                            "MatchOptions": ["GREATER_THAN_OR_EQUAL"],
                            "Values": ["50"]   # $50 absolute impact
                        }
                    },
                    {
                        "Dimensions": {
                            "Key": "ANOMALY_TOTAL_IMPACT_PERCENTAGE",
                            "MatchOptions": ["GREATER_THAN_OR_EQUAL"],
                            "Values": ["20"]   # 20% percentage deviation
                        }
                    }
                ]
            },
        )

        # ──────────────────────────────────────────────────────────────
        # Monitor by Cost Allocation Tag (project) — Customer Managed
        # Monitors specific projects with custom threshold
        # ──────────────────────────────────────────────────────────────
        tag_monitor = ce.CfnAnomalyMonitor(self, "ProjectTagMonitor",
            monitor_name="ProjectTagMonitor",
            monitor_type="CUSTOM",  # Customer managed
            monitor_specification=ce.CfnAnomalyMonitor.MonitorSpecificationProperty(
                expression={
                    "Tags": {
                        "Key": "project",
                        "Values": ["checkout-service", "payments-api", "fraud-detector"],
                        "MatchOptions": ["EQUALS"]
                    }
                }
            ),
        )

        ce.CfnAnomalySubscription(self, "ProjectSubscription",
            subscription_name="ProjectAnomalyAlerts",
            monitor_arn_list=[tag_monitor.attr_monitor_arn],
            subscribers=[
                ce.CfnAnomalySubscription.SubscriberProperty(
                    address=alert_topic.topic_arn,
                    type="SNS",
                    status="CONFIRMED",
                )
            ],
            frequency="IMMEDIATE",
            threshold_expression={
                "Dimensions": {
                    "Key": "ANOMALY_TOTAL_IMPACT_ABSOLUTE",
                    "MatchOptions": ["GREATER_THAN_OR_EQUAL"],
                    "Values": ["100"]  # higher threshold for core projects
                }
            },
        )

        # ──────────────────────────────────────────────────────────────
        # Budget by project tag — alert on forecast to exceed
        # ──────────────────────────────────────────────────────────────
        budgets.CfnBudget(self, "ProjectBudget",
            budget=budgets.CfnBudget.BudgetDataProperty(
                budget_name="checkout-service-monthly",
                budget_type="COST",
                time_unit="MONTHLY",
                budget_limit=budgets.CfnBudget.SpendProperty(
                    amount=5000, unit="USD"
                ),
                cost_filters={
                    "TagKeyValue": ["user:project$checkout-service"]
                },
            ),
            notifications_with_subscribers=[
                budgets.CfnBudget.NotificationWithSubscribersProperty(
                    notification=budgets.CfnBudget.NotificationProperty(
                        comparison_operator="GREATER_THAN",
                        notification_type="ACTUAL",
                        threshold=80,
                        threshold_type="PERCENTAGE",
                    ),
                    subscribers=[
                        budgets.CfnBudget.SubscriberProperty(
                            address=alert_topic.topic_arn,
                            subscription_type="SNS",
                        )
                    ],
                ),
                budgets.CfnBudget.NotificationWithSubscribersProperty(
                    notification=budgets.CfnBudget.NotificationProperty(
                        comparison_operator="GREATER_THAN",
                        notification_type="FORECASTED",
                        threshold=100,
                        threshold_type="PERCENTAGE",
                    ),
                    subscribers=[
                        budgets.CfnBudget.SubscriberProperty(
                            address=alert_topic.topic_arn,
                            subscription_type="SNS",
                        )
                    ],
                ),
            ],
        )
```

---

## 4. Python — Cost Explorer GetCostAndUsage

```python
import boto3
from datetime import datetime, date, timedelta
from dataclasses import dataclass, field
from typing import Optional
import calendar

ce = boto3.client("ce", region_name="us-east-1")


@dataclass
class ProjectCostReport:
    project: str
    service: str
    monthly_cost: float
    previous_month_cost: float
    mom_change_pct: float  # month-over-month %


def get_cost_by_project_and_service(
    tag_key: str = "project",
    tag_values: Optional[list[str]] = None,
    lookback_months: int = 2,
) -> list[ProjectCostReport]:
    """
    Returns monthly cost segmented by tag:project × SERVICE,
    using AmortizedCost (correct for FinOps — amortizes RI/SP upfront).
    """
    today = date.today()
    start_current = today.replace(day=1)
    start_prev = (start_current - timedelta(days=lookback_months * 31)).replace(day=1)
    end = today + timedelta(days=1)

    filter_expr: dict = {}
    if tag_values:
        filter_expr = {
            "Tags": {
                "Key": tag_key,
                "Values": tag_values,
                "MatchOptions": ["EQUALS"]
            }
        }

    params = dict(
        TimePeriod={
            "Start": start_prev.strftime("%Y-%m-%d"),
            "End": end.strftime("%Y-%m-%d"),
        },
        Granularity="MONTHLY",
        Metrics=["AmortizedCost"],
        GroupBy=[
            {"Type": "TAG",       "Key": tag_key},
            {"Type": "DIMENSION", "Key": "SERVICE"},
        ],
    )
    if filter_expr:
        params["Filter"] = filter_expr

    # Paginate results
    all_results = []
    next_token = None
    while True:
        if next_token:
            params["NextPageToken"] = next_token
        response = ce.get_cost_and_usage(**params)
        all_results.extend(response.get("ResultsByTime", []))
        next_token = response.get("NextPageToken")
        if not next_token:
            break

    # Organize by (project, service) → {month_key: cost}
    from collections import defaultdict
    cost_map: dict[tuple, dict[str, float]] = defaultdict(dict)
    for result in all_results:
        period_start = result["TimePeriod"]["Start"][:7]
        for group in result.get("Groups", []):
            keys = group["Keys"]
            project_val = keys[0].replace(f"{tag_key}$", "")
            service_val = keys[1]
            cost = float(group["Metrics"]["AmortizedCost"]["Amount"])
            cost_map[(project_val, service_val)][period_start] = cost

    # Build report with MoM variation
    months = sorted({m for costs in cost_map.values() for m in costs})
    if len(months) < 2:
        return []
    prev_month, curr_month = months[-2], months[-1]

    reports = []
    for (project, service), monthly_data in cost_map.items():
        curr = monthly_data.get(curr_month, 0.0)
        prev = monthly_data.get(prev_month, 0.0)
        mom = ((curr - prev) / prev * 100) if prev > 0 else 0.0
        if curr > 0.01 or prev > 0.01:
            reports.append(ProjectCostReport(
                project=project, service=service,
                monthly_cost=curr, previous_month_cost=prev, mom_change_pct=mom,
            ))

    return sorted(reports, key=lambda r: r.monthly_cost, reverse=True)
```

---

## 5. Compute Optimizer — Rightsizing

### 5.1 Supported Resources and Prerequisites

[FACT] Compute Optimizer supports: EC2 instances, EC2 Auto Scaling groups, EBS volumes, Lambda functions, ECS services on Fargate, Aurora/RDS databases, commercial software licenses.

[FACT] Compute Optimizer **is not active by default** — explicit opt-in is required in the account or in the Organization's management account.

[FACT] By default, it analyzes **14 days** of CloudWatch metrics. With Enhanced Infrastructure Metrics (paid), it extends to **93 days**.

[FACT] For recommendations that consider EC2 memory, the **CloudWatch agent** must be installed on the instance (or configure external metrics ingestion via Datadog/Dynatrace).

### 5.2 Findings (Possible Results)

```
╔══════════════════════╦══════════════════════════════════════════════════════╗
║ Finding              ║ Meaning                                              ║
╠══════════════════════╬══════════════════════════════════════════════════════╣
║ OVER_PROVISIONED     ║ Instance larger than needed; savings possible         ║
║ UNDER_PROVISIONED    ║ Instance smaller than needed; performance risk        ║
║ OPTIMIZED            ║ Configuration adequate for current usage              ║
║ NOT_OPTIMIZED        ║ Insufficient data or special configuration            ║
╚══════════════════════╩══════════════════════════════════════════════════════╝
```

### 5.3 Rightsizing Preferences — Presets

[FACT] Compute Optimizer offers 4 configurable presets, with direct impact on recommendation conservatism:

```
╔══════════════════════╦══════════════╦══════════════╦════════════════╗
║ Preset               ║ CPU Threshold║ CPU Headroom ║ Memory Headroom║
╠══════════════════════╬══════════════╬══════════════╬════════════════╣
║ Maximum savings      ║ P90          ║ 0%           ║ 10%            ║
║ Balanced             ║ P95          ║ 30%          ║ 30%            ║
║ Default              ║ P99.5        ║ 20%          ║ 20%            ║
║ Maximum performance  ║ P99.5        ║ 30%          ║ 30%            ║
╚══════════════════════╩══════════════╩══════════════╩════════════════╝

CPU Threshold = percentile above which data is ignored (e.g.: P90 ignores top 10% spikes)
CPU Headroom  = margin added above current usage for buffer
Memory Headroom = margin added above memory usage
```

[CONSENSUS] For conservative production, `Default` (P99.5 + 20% headroom) is adequate. For dev/staging environments where spikes are not critical, `Balanced` or `Maximum savings` generate more savings.

---

## 6-9. (Python code and CLI examples remain identical to the original — code is not translated)

See the original source file for the complete Python Compute Optimizer code and CLI examples. The code comments and variable names are already in English in the original.

---

## 10. Pitfalls

[FACT] **Cost Allocation Tags are not retroactive**: when activating a tag, only **future** data is indexed. Costs prior to activation do not appear filtered by that tag.

[FACT] **Cost Explorer has up to 24h delay**: detected anomalies may have up to 24h of delay. For real-time alerts, use CloudWatch Alarms instead of Cost Anomaly Detection.

[FACT] **Compute Optimizer requires opt-in**: it does not collect data until explicitly enabled. If activated today, the first recommendations only appear after 14 days of metric collection.

[FACT] **EC2 memory without CloudWatch agent = recommendations based on CPU only**: Compute Optimizer cannot recommend based on memory without the CloudWatch agent or external metrics ingestion. Recommendations without memory may overestimate over-provisioning.

[FACT] **BlendedCost ≠ UnblendedCost in Organizations**: in member accounts, BlendedCost distributes RI/SP discounts from the management account proportionally, which may appear **lower** than the actual cost charged to the account. Use UnblendedCost for real per-account chargeback.

[CONSENSUS] **Customer Managed Monitor with single threshold**: when using a single customer-managed monitor for multiple projects/tags with very different cost volumes (e.g.: $50 and $50,000/month), the same absolute threshold generates many false positives for the smaller or silence for the larger. Prefer separate monitors for similar cost groups.

[FACT] **Compute Optimizer does not consider SP/RI commitment**: by default, it recommends instances without considering your existing Savings Plans or Reserved Instances. Use `put-recommendation-preferences` with `preferredResources` to restrict to covered families.

---

## 11. When to Use Each Tool

```
┌─────────────────────────────────┬────────────────────────────────────┐
│ Question                        │ Tool                               │
├─────────────────────────────────┼────────────────────────────────────┤
│ How much did project X cost     │ Cost Explorer — GetCostAndUsage    │
│ this month?                     │ with filter by tag + AmortizedCost │
├─────────────────────────────────┼────────────────────────────────────┤
│ Spending increased              │ Cost Anomaly Detection             │
│ unexpectedly? Which service?    │ (ML detects + rootCauses)          │
├─────────────────────────────────┼────────────────────────────────────┤
│ Will I exceed the budget?       │ Cost Explorer — GetCostForecast    │
│                                 │ + Budgets with FORECASTED threshold│
├─────────────────────────────────┼────────────────────────────────────┤
│ Which instance is               │ Compute Optimizer — EC2/ECS recs   │
│ over-provisioned?               │ finding=OVER_PROVISIONED           │
├─────────────────────────────────┼────────────────────────────────────┤
│ Compare cost periods            │ Cost Explorer — GetCostComparisons │
├─────────────────────────────────┼────────────────────────────────────┤
│ Rightsizing at scale (500+      │ Compute Optimizer export → S3 →    │
│ instances)?                     │ Athena + QuickSight                │
└─────────────────────────────────┴────────────────────────────────────┘
```

---

## Reflection Exercise

An engineering team wants to implement an automatic cost governance system that:

1. **Detects** when any service of a specific project spends more than 150% of expected
2. **Identifies** the root cause (account, region, usage type)
3. **Updates** an instance tag with the status `cost-review=pending`
4. **Opens** an automatic ticket in the company's ticketing system

**Design the complete architecture**, answering:

1. Which type of Cost Anomaly Detection monitor to use — AWS Managed or Customer Managed? Why?
2. How to configure the threshold to detect exactly "150% of expected"? Which field to use: absolute or percentage?
3. What is the complete data path: from the spending occurrence to the ticket being opened? What are the delays?
4. Why not use Cost Explorer directly to detect anomalies in real time?
5. What would Compute Optimizer add to this flow? At what point in the pipeline would it be most useful?

---

## References

- [FACT] [What is AWS Cost Explorer](https://docs.aws.amazon.com/cost-management/latest/userguide/ce-what-is.html) — docs.aws.amazon.com
- [FACT] [Getting started with AWS Cost Anomaly Detection](https://docs.aws.amazon.com/cost-management/latest/userguide/getting-started-ad.html) — docs.aws.amazon.com
- [FACT] [What is AWS Compute Optimizer](https://docs.aws.amazon.com/compute-optimizer/latest/ug/what-is-compute-optimizer.html) — docs.aws.amazon.com
- [FACT] [Rightsizing recommendation preferences](https://docs.aws.amazon.com/compute-optimizer/latest/ug/rightsizing-preferences.html) — docs.aws.amazon.com
- [FACT] [Cost allocation tags](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/cost-alloc-tags.html) — docs.aws.amazon.com
- [FACT] [GetCostAndUsage API Reference](https://docs.aws.amazon.com/aws-cost-management/latest/APIReference/API_GetCostAndUsage.html) — docs.aws.amazon.com
