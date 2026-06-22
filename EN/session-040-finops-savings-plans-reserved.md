# Session 040 — FinOps: Savings Plans vs Reserved Instances — Differences and Flexibility

**Estimated duration:** 60 minutes  
**Dependencies:** none (independent topic)

---

## Objective

By the end of this session, you will be able to compare Compute Savings Plans, EC2 Instance Savings Plans, and Reserved Instances by instance flexibility, regional vs zonal scope, and discount; calculate the ROI of each option for a specific workload profile; and understand the risk of over-commitment.

---

## Context

[FACT] On-Demand pricing is the default model: you pay per hour of usage without commitment. AWS offers substantial discounts in exchange for **1 or 3-year usage commitment**. The two main mechanisms are **Savings Plans** (commitment in $/hour of usage) and **Reserved Instances** (commitment to specific capacity).

[FACT] AWS launched Savings Plans in November 2019 as an evolution of Reserved Instances, with greater flexibility. AWS itself recommends Savings Plans over Reserved Instances for most use cases — but Reserved Instances still offer specific advantages (capacity reservation, zonal discount).

---

## Core Concepts

### 1. Four Types of Savings Plans

[FACT] AWS offers four types of Savings Plans:

```
Type                    Max discount   Covered services
────────────────────────────────────────────────────────────────────
Compute Savings Plans   up to 66%      EC2 (any family/size/
                                       region/OS/tenancy), Fargate,
                                       Lambda

EC2 Instance            up to 72%      EC2 within a specific family
Savings Plans                          in a specific region
                                       (e.g.: m5 in us-east-1)
                                       — any size/OS/tenancy

Database Savings Plans  up to 35%      Aurora, RDS, DynamoDB,
                                       ElastiCache, DocumentDB,
                                       Timestream, Neptune, Keyspaces,
                                       DMS, OpenSearch

SageMaker AI            up to 64%      SageMaker AI — any family/
Savings Plans                          size/region/component
────────────────────────────────────────────────────────────────────
```

[FACT] Savings Plans commit a value in **$/hour** (not a number of instances). If you use more than committed, the excess is charged at On-Demand. If you use less, you pay for the commitment even without usage.

---

### 2. Types of Reserved Instances (RIs) for EC2

[FACT] Reserved Instances for EC2 have two main types by flexibility scope:

```
Type                Max discount   Flexibility
────────────────────────────────────────────────────────────────────
Standard RI         up to 72%      Family, size, OS, and region
                                   fixed. Size flexibility on
                                   regional RIs (within family).
                                   Can sell on Marketplace.

Convertible RI      up to 66%      Can exchange to change
                                   family, size, OS, or tenancy.
                                   Exchange is manual (not automatic).
                                   CANNOT sell on Marketplace.
────────────────────────────────────────────────────────────────────
```

[FACT] **RI Scope** — Regional vs. Zonal:

```
Regional Scope (recommended):
  - Discount applied to any instance in the region
  - Instance size flexibility: discount applies
    automatically to other sizes in the same family
    (e.g.: bought m5.large — applies to m5.xlarge, m5.2xlarge etc.)
  - NO guaranteed capacity reservation

Zonal Scope (AZ-specific):
  - Discount applied only to the specified AZ
  - NO size flexibility (exact instance)
  - WITH guaranteed capacity reservation in that AZ
  - Can be sold on Marketplace (Standard RI)
```

---

### 3. Complete Comparison Table

[FACT — AWS Docs]:

```
                        Compute   EC2 Instance   Convertible   Standard
                        SP        SP             RI            RI
─────────────────────────────────────────────────────────────────────────
Max discount            66%       72%            66%           72%
Commitment              $/hour    $/hour         # instances   # instances
Family flexible         ✓         —              manual        —
Size flexible           ✓         ✓              manual        regional only
OS flexible             ✓         ✓              manual        —
Tenancy flexible        ✓         ✓              manual        —
Cross-region            ✓         —              —             —
Covers Fargate          ✓         —              —             —
Covers Lambda           ✓         —              —             —
Capacity reservation    —         —              zonal only    zonal only
Cancellation            —         —              —             —
Exchange/modification   N/A       N/A            ✓ (manual)    limited
Sell on Marketplace     —         —              —             ✓ (zonal)
─────────────────────────────────────────────────────────────────────────
✓ = automatic; — = not available; manual = requires explicit action
```

---

### 4. Payment Options and Discount Impact

[FACT] All types of Savings Plans and RIs offer three payment options:

```
Payment             Commitment    Discount
────────────────────────────────────────────────────────────────
All Upfront         100% upfront    Maximum (66%, 72% etc.)
Partial Upfront     ~50% upfront    Intermediate
No Upfront          Zero upfront    Lower (still substantial)

3-year term > 1-year term in discount for all options.

Example (m5.xlarge Linux us-east-1):
  On-Demand:              $0.192/hour = $1,684/month
  Standard RI 1y All Up:  $0.110/hour = $965/month  → 43% off
  Standard RI 3y All Up:  $0.069/hour = $605/month  → 64% off
  Compute SP 1y All Up:   $0.116/hour = $1,018/month → 40% off
  Compute SP 3y All Up:   $0.073/hour = $640/month  → 62% off
  EC2 Instance SP 1y:     $0.109/hour = $957/month  → 43% off
```

[UNCERTAIN] The exact discount values vary by region, instance family, and OS — always check the AWS pricing calculator for your specific case, as percentages change periodically.

---

### 5. How Savings Plans Are Applied

[FACT] Savings Plans application follows a hierarchy:

```
Application order:
1. Savings Plans cover first the instances with the LOWEST
   On-Demand price in your portfolio (maximizes coverage)
2. EC2 Instance SP (more specific) is applied BEFORE
   Compute SP (more generic)
3. What exceeds the $/hour commitment → charged On-Demand

Example:
  You have a Compute SP of $1.00/hour (3 years, no upfront)
  Your instances cost:
    - m5.large Linux us-east-1: $0.096/hour On-Demand
    - c5.xlarge Linux eu-west-1: $0.192/hour On-Demand
  
  With Compute SP, you pay the SP price instead of On-Demand
  for the instances covering the commitment:
    ~$1.00/hour → covers these instances with ~60% discount
  Above that → On-Demand
```

[FACT] Savings Plans **do not provide capacity reservation**. If you need to guarantee capacity in a specific AZ (e.g., critical failover), use **On-Demand Capacity Reservation (ODCR)** — Savings Plans apply on top of ODCR.

---

### 6. ROI Analysis and Over-Commitment Risk

[CONSENSUS] The central risk of Savings Plans and RIs is **over-commitment**: committing more than you use. The commitment is irrevocable during the term — you pay even without usage.

```
Decision framework:
─────────────────────────────────────────────────────────────
Baseline usage           → Commit (Savings Plans / RI)
Variable but predictable → Commit baseline, On-Demand for peaks
Unpredictable            → On-Demand (or Spot for interruption-tolerant)

Rule of thumb:
  Commit at most your P10 usage (10th percentile)
  — the usage you have 90% of the time as a guaranteed minimum.
  
  Never commit average usage if there is seasonality,
  because during below-average periods you pay without receiving.
```

**Calculating break-even (Savings Plans 1 year vs. On-Demand):**

```
On-Demand:       $0.192/hour × 8,760h/year = $1,682/year
Compute SP 1y:   $0.116/hour × 8,760h/year = $1,016/year  (66% utilization)

Break-even: you need to use the instance at least X% of the time
for SP to be cheaper than On-Demand:

Break-even % = SP_price / OD_price = 0.116 / 0.192 = 60.4%

→ If instance is active > 60.4% of the time: SP is cheaper
→ If active < 60.4% of the time: On-Demand would be cheaper
```

---

## Practical Example

### Scenario: Evaluating the FinOps Strategy for an E-Commerce Platform

**Workload:**
- 10 instances `m5.xlarge` Linux in us-east-1 running 24/7 (stable web tier)
- 5 instances `c5.2xlarge` Linux in us-east-1 running 70% of the time (batch processing)
- Lambda functions (1,000,000 invocations/month, 512MB, 500ms avg duration)
- Fargate tasks (3,000 vCPU-hours/month, 6,000 GB-hours/month)

#### CDK Python — Cost Anomaly Detection + Budget Alert

```python
from aws_cdk import (
    Stack,
    aws_budgets as budgets,
    aws_ce as ce,  # Cost Explorer
)
from constructs import Construct


class FinOpsCostControlStack(Stack):
    def __init__(self, scope: Construct, construct_id: str, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        # ── Budget: alert when monthly cost exceeds $10,000 ───────────
        monthly_budget = budgets.CfnBudget(
            self, "MonthlyComputeBudget",
            budget=budgets.CfnBudget.BudgetDataProperty(
                budget_name="monthly-compute-budget",
                budget_type="COST",
                time_unit="MONTHLY",
                budget_limit=budgets.CfnBudget.SpendProperty(
                    amount=10000,
                    unit="USD",
                ),
                cost_filters={
                    "Service": ["Amazon Elastic Compute Cloud - Compute",
                                "AWS Lambda",
                                "AWS Fargate"],
                },
                cost_types=budgets.CfnBudget.CostTypesProperty(
                    include_tax=True,
                    include_subscription=True,
                    use_blended=False,
                ),
            ),
            notifications_with_subscribers=[
                budgets.CfnBudget.NotificationWithSubscribersProperty(
                    notification=budgets.CfnBudget.NotificationProperty(
                        comparison_operator="GREATER_THAN",
                        notification_type="ACTUAL",   # actual cost (not forecast)
                        threshold=80,                 # 80% of budget
                        threshold_type="PERCENTAGE",
                    ),
                    subscribers=[
                        budgets.CfnBudget.SubscriberProperty(
                            address="finops@company.com",
                            subscription_type="EMAIL",
                        ),
                    ],
                ),
                budgets.CfnBudget.NotificationWithSubscribersProperty(
                    notification=budgets.CfnBudget.NotificationProperty(
                        comparison_operator="GREATER_THAN",
                        notification_type="FORECASTED",  # forecast exceeds budget
                        threshold=100,
                        threshold_type="PERCENTAGE",
                    ),
                    subscribers=[
                        budgets.CfnBudget.SubscriberProperty(
                            address="finops@company.com",
                            subscription_type="EMAIL",
                        ),
                    ],
                ),
            ],
        )

        # ── Cost Anomaly Detection ────────────────────────────────────
        # Detects cost anomalies automatically (ML-based)
        anomaly_monitor = ce.CfnAnomalyMonitor(
            self, "ComputeCostMonitor",
            monitor_name="compute-cost-monitor",
            monitor_type="DIMENSIONAL",
            monitor_dimension="SERVICE",
        )

        anomaly_subscription = ce.CfnAnomalySubscription(
            self, "ComputeAnomalyAlert",
            subscription_name="compute-anomaly-alert",
            monitor_arn_list=[anomaly_monitor.attr_monitor_arn],
            subscribers=[
                ce.CfnAnomalySubscription.SubscriberProperty(
                    address="finops@company.com",
                    type="EMAIL",
                    status="CONFIRMED",
                ),
            ],
            # Alert when total impact exceeds $100
            threshold_expression=ce.CfnAnomalySubscription.ExpressionProperty(
                dimensions=ce.CfnAnomalySubscription.DimensionValuesProperty(
                    key="ANOMALY_TOTAL_IMPACT_ABSOLUTE",
                    values=["100"],
                    match_options=["GREATER_THAN_OR_EQUAL"],
                )
            ),
            frequency="DAILY",
        )
```

#### ROI Calculation — Python (Offline Analysis)

```python
"""
FinOps analysis script: calculates ROI of Savings Plans vs On-Demand
for the workload profile described in the scenario.

Approximate prices us-east-1 (always check the pricing calculator):
https://aws.amazon.com/ec2/pricing/reserved-instances/pricing/
"""

from dataclasses import dataclass
from typing import Literal


@dataclass
class WorkloadComponent:
    name: str
    on_demand_hourly: float
    utilization_pct: float      # 0.0 – 1.0
    count: int
    hours_per_year: float = 8760.0


def annual_on_demand_cost(w: WorkloadComponent) -> float:
    return w.on_demand_hourly * w.utilization_pct * w.hours_per_year * w.count


def annual_savings_plan_cost(
    w: WorkloadComponent,
    sp_hourly_rate: float,
) -> float:
    """
    Savings Plan is always charged by the commitment (sp_hourly_rate * count),
    regardless of utilization — that's why over-commitment is dangerous.
    """
    committed = sp_hourly_rate * w.hours_per_year * w.count
    return committed


def break_even_utilization(
    on_demand_rate: float,
    sp_rate: float,
) -> float:
    """Minimum utilization for SP to be cheaper than On-Demand."""
    return sp_rate / on_demand_rate


def analyze_workload():
    # ── Workload components ───────────────────────────────────────────
    web_tier = WorkloadComponent(
        name="Web tier (m5.xlarge, 10x)",
        on_demand_hourly=0.192,     # m5.xlarge Linux us-east-1
        utilization_pct=1.0,        # 100% — always on
        count=10,
    )
    batch_tier = WorkloadComponent(
        name="Batch tier (c5.2xlarge, 5x)",
        on_demand_hourly=0.340,     # c5.2xlarge Linux us-east-1
        utilization_pct=0.70,       # 70% of the time
        count=5,
    )

    # ── Approximate Savings Plans prices 1y No Upfront ────────────────
    # NOTE: always check current pricing at aws.amazon.com
    WEB_SP_RATE   = 0.116   # Compute SP ~40% off On-Demand (m5.xlarge)
    BATCH_SP_RATE = 0.204   # Compute SP ~40% off On-Demand (c5.2xlarge)

    print("=" * 65)
    print("FINOPS ANALYSIS — Savings Plans vs On-Demand")
    print("=" * 65)

    for workload, sp_rate in [(web_tier, WEB_SP_RATE), (batch_tier, BATCH_SP_RATE)]:
        od_annual = annual_on_demand_cost(workload)
        sp_annual = annual_savings_plan_cost(workload, sp_rate)
        savings   = od_annual - sp_annual
        be_util   = break_even_utilization(workload.on_demand_hourly, sp_rate)

        print(f"\n{workload.name}")
        print(f"  Annual On-Demand:         ${od_annual:,.0f}")
        print(f"  Annual Savings Plan:      ${sp_annual:,.0f}")
        print(f"  Savings:                  ${savings:,.0f}  ({savings/od_annual*100:.1f}%)")
        print(f"  Break-even utilization:   {be_util*100:.1f}%")
        print(f"  Current utilization:      {workload.utilization_pct*100:.0f}%")
        print(f"  → {'✓ SP RECOMMENDED' if workload.utilization_pct > be_util else '✗ On-Demand cheaper'}")

    # ── Over-commitment risk on batch tier ────────────────────────────
    print("\n--- RISK ANALYSIS (Batch tier) ---")
    print("Scenario: utilization drops to 40% (half of expected)")
    batch_worst = WorkloadComponent(
        name="Batch tier — worst case",
        on_demand_hourly=0.340,
        utilization_pct=0.40,   # worst case
        count=5,
    )
    od_worst  = annual_on_demand_cost(batch_worst)
    sp_annual = annual_savings_plan_cost(batch_tier, BATCH_SP_RATE)
    waste     = sp_annual - od_worst
    print(f"  On-Demand (40% util):   ${od_worst:,.0f}")
    print(f"  Savings Plan (fixed):   ${sp_annual:,.0f}")
    print(f"  Waste:                  ${waste:,.0f}  (paying without using)")
    print(f"  → RECOMMENDATION: commit only the safe baseline (P10)")


if __name__ == "__main__":
    analyze_workload()
```

#### CLI — Check Recommendations and Purchase Savings Plan

```bash
# 1. Get Savings Plans recommendations from Cost Explorer (AWS CLI)
aws ce get-savings-plans-purchase-recommendation \
  --savings-plans-type COMPUTE_SP \
  --term-in-years ONE_YEAR \
  --payment-option NO_UPFRONT \
  --lookback-period-in-days THIRTY_DAYS \
  --query 'SavingsPlansPurchaseRecommendation.{
    Summary:SavingsPlansPurchaseRecommendationSummary,
    Details:SavingsPlansPurchaseRecommendationDetails[0:3]
  }'

# 2. View current utilization of existing Savings Plans
aws ce get-savings-plans-utilization \
  --time-period '{"Start":"2026-06-01","End":"2026-06-30"}' \
  --query 'Total.{
    Utilized:Utilization.UtilizationPercentage,
    UnusedHours:Savings.NetSavings,
    TotalCommitment:Utilization.TotalCommitment
  }'

# 3. View Savings Plans coverage (% of On-Demand covered)
aws ce get-savings-plans-coverage \
  --time-period '{"Start":"2026-06-01","End":"2026-06-30"}' \
  --granularity MONTHLY \
  --query 'SavingsPlansCoverages[*].Coverage.{
    CoverageHoursPercentage:CoverageHoursPercentage,
    SpendCoveredBySavingsPlans:SpendCoveredBySavingsPlans,
    OnDemandCost:OnDemandCost
  }'

# 4. Purchase Savings Plan (CAUTION — irrevocable commitment)
# Always check Cost Explorer recommendation first
aws savingsplans create-savings-plan \
  --savings-plan-offering-id "offering-id-from-describe" \
  --commitment 1.00 \
  --purchase-time "2026-07-01T00:00:00Z" \
  --client-token "unique-idempotency-key-$(uuidgen)"

# 5. List active Savings Plans
aws savingsplans describe-savings-plans \
  --states ACTIVE \
  --query 'savingsPlans[*].{
    Id:savingsPlanId,
    Type:savingsPlanType,
    Commitment:commitment,
    Start:start,
    End:end,
    State:state,
    PaymentOption:paymentOption,
    Term:termDurationInSeconds
  }'

# 6. View Reserved Instances recommendations
aws ce get-reservation-purchase-recommendation \
  --service "Amazon EC2" \
  --lookback-period-in-days THIRTY_DAYS \
  --term-in-years ONE_YEAR \
  --payment-option NO_UPFRONT \
  --query 'Recommendations[0:3].{
    Summary:RecommendationSummary,
    Details:RecommendationDetails[0:2]
  }'

# 7. View Reserved Instances utilization
aws ce get-reservation-utilization \
  --time-period '{"Start":"2026-06-01","End":"2026-06-30"}' \
  --query 'Total.UtilizationsByTime[*].{
    UtilizationPct:Utilization.UtilizationPercentage,
    UnusedHours:Utilization.UnusedHours
  }'
```

---

## Common Pitfalls

**1. Over-commitment: committing the average, not the baseline**  
[CONSENSUS] The most common FinOps pitfall is buying Savings Plans or RIs based on the average usage over the last 30 days, without considering seasonality. A workload with a peak of 100 instances on Black Friday and a minimum of 20 instances in July should commit only the 20 baseline instances — not the average. Over-commitment wastes capital on idle capacity.

**2. Savings Plans do not provide capacity reservation**  
[FACT] Unlike zonal Reserved Instances, Savings Plans do not guarantee capacity will be available. In regional scarcity scenarios (e.g., during large-scale events or outages), On-Demand and Savings Plans compete for available capacity. For critical workloads with availability SLAs, combine Savings Plans with On-Demand Capacity Reservation (ODCR).

**3. Savings Plans do not cover Spot usage**  
[FACT] Savings Plans and RIs are applied only to On-Demand and ODCR usage. Spot hours do not receive additional Savings Plans discounts — they already have their own discounted price. Planning Savings Plans coverage for workloads you intend to move to Spot results in unnecessary commitment.

**4. Convertible RIs require manual exchange — not automatic**  
[FACT] Unlike EC2 Instance Savings Plans (which automatically apply to any size in the family), Convertible RIs require you to **manually execute the exchange** via console or API when you want to change family, size, or OS. Forgetting to do the exchange means the old RI remains unused while you pay On-Demand for the new instances.

**5. Standard RI zonal: size flexibility absent**  
[FACT] Standard RIs with **Zonal** scope do not have size flexibility — the discount only applies to exactly the size, OS, and tenancy specified at purchase. RIs with **Regional** scope have size flexibility within the family. To get size flexibility with RI, always use Regional scope.

**6. Savings Plans are applied sequentially, not in parallel**  
[FACT] If you have multiple Savings Plans (e.g., EC2 Instance SP and Compute SP), they are applied first with the EC2 Instance SP (more specific), then the Compute SP for the remainder. Buying both without analyzing overlap can result in redundant coverage paying for the Compute SP without using it.

---

## Reflection Exercise

Your company has the following EC2 usage profile in us-east-1 (data from the last 90 days):

- **Stable instances:** 20 x `m5.2xlarge` Linux running 100% of the time (web application)
- **Processing instances:** 10 x `c5.4xlarge` Linux running on average 60% of the time, with minimum 40% and maximum 90% (daytime jobs)
- **ML instances:** 3 x `p3.2xlarge` Linux running on average 30% of the time (weekly training)

Answer:

1. For stable instances, what is the annual financial difference between Compute Savings Plans 1 year All Upfront (discount ~60%) and Standard RI 1 year All Upfront (discount ~64%), considering `m5.2xlarge` On-Demand at $0.384/hour? When does the Compute SP flexibility justify the discount difference?

2. For processing instances, what would be the safe Savings Plans commitment considering the over-commitment risk? Justify using the P10 concept (10th percentile of utilization = 40%).

3. For ML instances (`p3.2xlarge`, 30% average utilization): does the Compute SP break-even (discount ~60%) justify the commitment? What would be the most suitable alternative for this type of workload?

---

## Resources for Further Study

- [FACT] Savings Plans types (Compute, EC2 Instance, Database, SageMaker): https://docs.aws.amazon.com/savingsplans/latest/userguide/plan-types.html
- [FACT] Compute Savings Plans vs Reserved Instances (comparison table): https://docs.aws.amazon.com/savingsplans/latest/userguide/sp-ris.html
- [FACT] How Savings Plans apply to usage: https://docs.aws.amazon.com/savingsplans/latest/userguide/sp-applying.html
- [FACT] Reserved Instances for EC2 (regional vs zonal scope, Standard vs Convertible): https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-reserved-instances.html
- [UNCERTAIN] AWS Pricing Calculator (current prices — check before committing): https://calculator.aws/pricing/2/home

---
