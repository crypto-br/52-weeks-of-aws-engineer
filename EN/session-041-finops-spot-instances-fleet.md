# Session 041 — FinOps: Spot Instances, EC2 Fleet, and Interruption Handling

**Estimated duration:** 60 min  
**Prerequisite:** session-040 (Savings Plans and Reserved Instances)

---

## Session Objectives

- Understand the Spot Instances pricing model and risk
- Distinguish modern APIs (CreateFleet / Auto Scaling) from legacy ones (RequestSpotFleet — **do not use**)
- Master allocation strategies: `price-capacity-optimized`, `capacity-optimized`, `diversified`
- Implement complete interruption handling: rebalance recommendation + 2-min notice via IMDS and EventBridge
- Apply instance type diversification and attribute-based selection
- Use Spot Placement Scores for region/AZ selection

---

## 1. Economic Model and Risk

[FACT] Spot Instances offer discounts of **up to 90%** compared to On-Demand pricing, providing access to AWS's idle EC2 capacity. AWS can interrupt a Spot Instance with **2-minute notice** when it needs the capacity back.

[FACT] A **Spot capacity pool** is the set of idle instances with the same instance type **and** the same Availability Zone. The Spot price is defined per pool and varies with supply and demand.

```
Interruption reasons (AWS docs):
┌─────────────────────────────────────────────────────────────────┐
│  CAPACITY  — AWS needs the capacity back (main cause)            │
│  PRICE     — Spot price rose above your maxPrice                 │
│  CONSTRAINT— launch group / AZ group can no longer be satisfied  │
└─────────────────────────────────────────────────────────────────┘

Behavior on interruption (configurable):
  terminate  — default; instance terminates
  stop       — EBS preserved; can restart when capacity returns
  hibernate  — RAM saved to EBS; WITHOUT 2-min warning (immediate hibernation)
```

[FACT] Workloads suitable for Spot: **stateless, fault-tolerant, flexible** — big data, containerized, CI/CD, stateless web servers, HPC, rendering. Workloads **not suitable**: inflexible, stateful, tightly-coupled between nodes, intolerant to any period of partial capacity.

[CONSENSUS] Trying to failover from Spot to On-Demand in response to interruptions can inadvertently cause more interruptions in other Spot Instances. AWS explicitly discourages this pattern.

---

## 2. APIs: What to Use and What to Avoid

[FACT] The official documentation (updated 2026) explicitly classifies Spot APIs as:

```
╔══════════════════════════════════╦══════════════════════════════════════════╗
║ API                              ║ Recommendation                           ║
╠══════════════════════════════════╬══════════════════════════════════════════╣
║ CreateAutoScalingGroup           ║ ✅ YES — managed lifecycle, scaling       ║
║ CreateFleet (instant mode)       ║ ✅ YES — no auto scaling needed           ║
║ RunInstances                     ║ ⚠️  LIMITED — only 1 instance type        ║
║ RequestSpotFleet                 ║ ❌ NO — legacy API, no investment         ║
║ RequestSpotInstances             ║ ❌ NO — legacy API, no investment         ║
╚══════════════════════════════════╩══════════════════════════════════════════╝
```

[FACT] `RequestSpotFleet` and `RequestSpotInstances` are explicitly marked as legacy APIs with "no planned investment" in the AWS documentation. New workloads should use **Auto Scaling Groups** or **EC2 Fleet**.

---

## 3. Allocation Strategies

[FACT] When using multiple capacity pools (EC2 Fleet or Auto Scaling), the **allocation strategy** determines from which pools instances will be launched.

### 3.1 price-capacity-optimized (recommended)

[FACT] Identified as "best choice for most Spot workloads" in AWS documentation. The fleet identifies pools with the **highest available capacity** and, among those, picks the ones with the **lowest price**. Result: lower interruption rate + good cost.

```
Selection: pools with most available capacity → lowest price among those
Use:       stateless containers, microservices, web apps, data/analytics, batch
CLI:       --spot-allocation-strategy price-capacity-optimized (EC2 Fleet)
           --spot-allocation-strategy priceCapacityOptimized  (legacy Spot Fleet)
```

### 3.2 capacity-optimized

[FACT] Focuses exclusively on **maximum capacity availability**, without considering price. Useful when the cost of reprocessing an interruption is very high (long CI, rendering, HPC with hours of computation). Accepts variant `capacity-optimized-prioritized` for ordering instance types.

### 3.3 diversified

[FACT] Distributes instances across **all configured pools** equally. If 10 pools configured and target=100, launches 10 instances in each. Protects against mass interruption of a single pool (only 10% affected).

```
When to use: large fleets or those running for a long time
Limitation:  does not launch in pools where Spot price ≥ On-Demand price
```

### 3.4 lowest-price (NOT recommended)

[FACT] **Highest interruption risk** — only considers price, ignores capacity availability. Pools with higher demand (cheaper) tend to have higher interruption rates. AWS explicitly documents: "We don't recommend the lowest-price allocation strategy."

---

## 4. Instance Type Diversification

[CONSENSUS] AWS's golden rule is to be flexible across at least **10 instance types** per workload, plus enabling **all available AZs** in the VPC.

```
Diversification strategy for a processing workload:
  c family (compute): c5.xlarge, c5a.xlarge, c5n.xlarge, c4.xlarge
  m family (general): m5.xlarge, m5a.xlarge, m5n.xlarge, m4.xlarge
  r family (memory):  r5.large, r5a.large  (if workload accepts it)

Principle: if vertically flexible → include larger instances (more vCPUs)
           if only horizontal scaling → include older generations (lower OD demand)
```

### 4.1 Attribute-based Instance Type Selection (ABITS)

[FACT] Instead of listing specific types, specifies **attributes** (min/max vCPUs, memory, architecture, etc.) and AWS automatically selects all compatible types. Ensures use of new instance types as they are launched.

```python
# CDK Python — attribute-based via CfnAutoScalingGroup
# (see section 7 for complete example)
instance_requirements = autoscaling.CfnAutoScalingGroup.InstanceRequirementsProperty(
    v_cpu_count=autoscaling.CfnAutoScalingGroup.VCpuCountRequestProperty(min=2, max=8),
    memory_mi_b=autoscaling.CfnAutoScalingGroup.MemoryMiBRequestProperty(min=4096, max=16384),
    cpu_manufacturers=["intel", "amd"],
    instance_generations=["current"],
)
```

---

## 5. Spot Placement Scores

[FACT] The **Spot Placement Score** (1–10) indicates the probability of provisioning the requested Spot capacity in a specific region or AZ. Score 10 = highly likely to succeed. It is a **point-in-time** recommendation — it does not guarantee capacity or predict future interruption rate.

```bash
# Get placement score for 100 vCPUs across multiple regions
aws ec2 get-spot-placement-scores \
  --target-capacity 100 \
  --target-capacity-unit-type vcpu \
  --single-availability-zone-flag false \
  --instance-requirements-with-metadata '{
    "ArchitectureTypes": ["x86_64"],
    "VirtualizationTypes": ["hvm"],
    "InstanceRequirements": {
      "VCpuCount": {"Min": 2, "Max": 4},
      "MemoryMiB": {"Min": 4096}
    }
  }'
# Returns: RegionName, Score (1-10), AvailabilityZoneId (if requested)
```

---

## 6. Interruption Handling — Two Signals

### 6.1 Interruption Temporal Architecture

```
Timeline of an interruption:

 T-?min          T-2min                    T
  │               │                         │
  ▼               ▼                         ▼
 [Rebalance      [Interruption Notice    [Instance
  Recommendation] appears in IMDS and     interrupted]
  EventBridge]    EventBridge emits]
  
  ← best-effort → ← guaranteed* 2 min →

* except hibernation: starts immediately, without 2 min notice
```

[FACT] The **Rebalance Recommendation** may arrive **before** the 2-minute notice, giving more time for proactive action. However, it is emitted on a **best-effort** basis — it may arrive together with the 2-minute notice, or not arrive at all.

[FACT] The **Interruption Notice** (2-minute warning) is guaranteed for `terminate` and `stop` actions. For `hibernate`, hibernation starts immediately without a 2-minute advance notice.

### 6.2 Monitoring via IMDS (Instance Metadata Service)

[FACT] Both signals are available as items in IMDS v2. AWS recommends polling every **5 seconds**.

```
Rebalance Recommendation endpoint:
  GET http://169.254.169.254/latest/meta-data/events/recommendations/rebalance
  Response when present: {"noticeTime": "2024-01-15T14:22:00Z"}
  Response when absent:  HTTP 404

Interruption Notice endpoint:
  GET http://169.254.169.254/latest/meta-data/spot/instance-action
  Response when present: {"action": "terminate", "time": "2024-01-15T14:25:00Z"}
                         {"action": "stop",      "time": "2024-01-15T14:25:00Z"}
  Response when absent:  HTTP 404
```

### 6.3 Monitoring via EventBridge

[FACT] Both signals are emitted as events in EventBridge in the instance's account/region. `detail-type` allows filtering:

```json
// Event: Rebalance Recommendation
{
  "detail-type": "EC2 Instance Rebalance Recommendation",
  "source": "aws.ec2",
  "detail": { "instance-id": "i-0abcdef1234567890" }
}

// Event: Interruption Warning (2 min)
{
  "detail-type": "EC2 Spot Instance Interruption Warning",
  "source": "aws.ec2",
  "detail": {
    "instance-id": "i-0abcdef1234567890",
    "instance-action": "terminate"   // or "stop" or "hibernate"
  }
}
```

### 6.4 Capacity Rebalancing (Auto Scaling / EC2 Fleet)

[FACT] Auto Scaling Groups and EC2 Fleet have a native **Capacity Rebalancing** feature: when they receive the Rebalance Recommendation signal, they automatically launch replacement instances **before** terminating instances at risk. This maintains the target capacity during transitions.

---

## 7. CDK Python — Auto Scaling Group with Spot (Mixed Instances)

```python
from aws_cdk import (
    Stack, Duration, aws_ec2 as ec2,
    aws_autoscaling as autoscaling,
    aws_iam as iam, aws_sns as sns,
    aws_events as events, aws_events_targets as targets,
    aws_lambda as _lambda,
)
from constructs import Construct

class SpotFleetStack(Stack):
    def __init__(self, scope: Construct, construct_id: str, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        vpc = ec2.Vpc(self, "VPC",
            max_azs=3,  # all AZs for diversification
            nat_gateways=1,
        )

        # Security Group for the processor
        sg = ec2.SecurityGroup(self, "WorkerSG", vpc=vpc, allow_all_outbound=True)

        # IAM Role for the instances
        role = iam.Role(self, "WorkerRole",
            assumed_by=iam.ServicePrincipal("ec2.amazonaws.com"),
            managed_policies=[
                iam.ManagedPolicy.from_aws_managed_policy_name(
                    "AmazonSSMManagedInstanceCore"
                ),
            ],
        )

        # Launch Template with user data for graceful shutdown
        user_data = ec2.UserData.for_linux()
        user_data.add_commands(
            "#!/bin/bash",
            "yum install -y aws-cli jq",
            # Install and start the interruption handling daemon
            "cat > /usr/local/bin/spot-handler.sh << 'EOF'",
            "#!/bin/bash",
            "while true; do",
            "  TOKEN=$(curl -s -X PUT http://169.254.169.254/latest/api/token \\",
            "    -H 'X-aws-ec2-metadata-token-ttl-seconds: 21600')",
            "  ACTION=$(curl -s -H \"X-aws-ec2-metadata-token: $TOKEN\" \\",
            "    http://169.254.169.254/latest/meta-data/spot/instance-action 2>/dev/null)",
            "  if echo \"$ACTION\" | grep -q 'terminate\\|stop'; then",
            "    echo 'INTERRUPTION NOTICE: '$ACTION",
            "    systemctl stop worker.service",
            "    aws sqs send-message --queue-url $DRAIN_QUEUE_URL \\",
            "      --message-body \"{\\\"instance_id\\\":\\\"$(curl -s -H \\\"X-aws-ec2-metadata-token: $TOKEN\\\" \\",
            "      http://169.254.169.254/latest/meta-data/instance-id)\\\"}\"",
            "    break",
            "  fi",
            "  REBALANCE=$(curl -s -H \"X-aws-ec2-metadata-token: $TOKEN\" \\",
            "    http://169.254.169.254/latest/meta-data/events/recommendations/rebalance 2>/dev/null)",
            "  if echo \"$REBALANCE\" | grep -q 'noticeTime'; then",
            "    echo 'REBALANCE RECOMMENDATION: '$REBALANCE",
            "    # Stop new jobs but don't terminate existing ones",
            "    touch /var/run/spot-draining",
            "  fi",
            "  sleep 5",
            "done",
            "EOF",
            "chmod +x /usr/local/bin/spot-handler.sh",
            "nohup /usr/local/bin/spot-handler.sh &",
        )

        launch_template = ec2.LaunchTemplate(self, "WorkerLT",
            machine_image=ec2.MachineImage.latest_amazon_linux2(),
            role=role,
            security_group=sg,
            user_data=user_data,
            # DO NOT define instance_type here when using mixed instances policy
        )

        # Auto Scaling Group with mixed instances policy
        # Uses CfnAutoScalingGroup (L1) for access to the full MixedInstancesPolicy
        asg = autoscaling.CfnAutoScalingGroup(self, "WorkerASG",
            min_size="2",
            max_size="20",
            desired_capacity="6",
            vpc_zone_identifier=vpc.select_subnets(
                subnet_type=ec2.SubnetType.PRIVATE_WITH_EGRESS
            ).subnet_ids,
            # Capacity Rebalancing — launches replacement before terminating at-risk instance
            capacity_rebalance=True,
            mixed_instances_policy=autoscaling.CfnAutoScalingGroup.MixedInstancesPolicyProperty(
                launch_template=autoscaling.CfnAutoScalingGroup.LaunchTemplateProperty(
                    launch_template_specification=autoscaling.CfnAutoScalingGroup.LaunchTemplateSpecificationProperty(
                        launch_template_id=launch_template.launch_template_id,
                        version=launch_template.latest_version_number,
                    ),
                    overrides=[
                        # Diversification: ≥ 10 instance types
                        autoscaling.CfnAutoScalingGroup.LaunchTemplateOverridesProperty(
                            instance_type="c5.xlarge"),
                        autoscaling.CfnAutoScalingGroup.LaunchTemplateOverridesProperty(
                            instance_type="c5a.xlarge"),
                        autoscaling.CfnAutoScalingGroup.LaunchTemplateOverridesProperty(
                            instance_type="c5n.xlarge"),
                        autoscaling.CfnAutoScalingGroup.LaunchTemplateOverridesProperty(
                            instance_type="c4.xlarge"),
                        autoscaling.CfnAutoScalingGroup.LaunchTemplateOverridesProperty(
                            instance_type="m5.xlarge"),
                        autoscaling.CfnAutoScalingGroup.LaunchTemplateOverridesProperty(
                            instance_type="m5a.xlarge"),
                        autoscaling.CfnAutoScalingGroup.LaunchTemplateOverridesProperty(
                            instance_type="m5n.xlarge"),
                        autoscaling.CfnAutoScalingGroup.LaunchTemplateOverridesProperty(
                            instance_type="m4.xlarge"),
                        autoscaling.CfnAutoScalingGroup.LaunchTemplateOverridesProperty(
                            instance_type="r5.large"),
                        autoscaling.CfnAutoScalingGroup.LaunchTemplateOverridesProperty(
                            instance_type="r5a.large"),
                    ],
                ),
                instances_distribution=autoscaling.CfnAutoScalingGroup.InstancesDistributionProperty(
                    # 0% On-Demand base, 100% Spot
                    on_demand_base_capacity=0,
                    on_demand_percentage_above_base_capacity=0,
                    spot_allocation_strategy="price-capacity-optimized",  # RECOMMENDED
                ),
            ),
        )

        # --- EventBridge: Lambda for Interruption Warning ---
        interrupt_fn = _lambda.Function(self, "SpotInterruptHandler",
            runtime=_lambda.Runtime.PYTHON_3_12,
            handler="index.handler",
            code=_lambda.Code.from_inline("""
import boto3, json, os

ec2 = boto3.client('ec2')
asg = boto3.client('autoscaling')

def handler(event, context):
    instance_id = event['detail']['instance-id']
    action = event['detail']['instance-action']
    print(f"INTERRUPTION WARNING: {instance_id} will be {action}")
    
    # Detach from ASG gracefully (allows ASG to replace the instance)
    try:
        response = asg.describe_auto_scaling_instances(InstanceIds=[instance_id])
        if response['AutoScalingInstances']:
            asg_name = response['AutoScalingInstances'][0]['AutoScalingGroupName']
            asg.detach_instances(
                InstanceIds=[instance_id],
                AutoScalingGroupName=asg_name,
                ShouldDecrementDesiredCapacity=False,  # ASG provisions replacement
            )
            print(f"Detached {instance_id} from ASG {asg_name}")
    except Exception as e:
        print(f"ASG detach failed (may be OK): {e}")
    
    return {"statusCode": 200, "instanceId": instance_id, "action": action}
"""),
        )
        interrupt_fn.add_to_role_policy(iam.PolicyStatement(
            actions=["autoscaling:DescribeAutoScalingInstances",
                     "autoscaling:DetachInstances",
                     "ec2:DescribeInstances"],
            resources=["*"],
        ))

        # EventBridge rule: Interruption Warning → Lambda
        events.Rule(self, "SpotInterruptRule",
            event_pattern=events.EventPattern(
                source=["aws.ec2"],
                detail_type=["EC2 Spot Instance Interruption Warning"],
            ),
            targets=[targets.LambdaFunction(interrupt_fn)],
        )

        # EventBridge rule: Rebalance Recommendation → Lambda (same or different function)
        events.Rule(self, "SpotRebalanceRule",
            event_pattern=events.EventPattern(
                source=["aws.ec2"],
                detail_type=["EC2 Instance Rebalance Recommendation"],
            ),
            targets=[targets.LambdaFunction(interrupt_fn)],
        )
```

---

## 8. Python — Spot Price History and Savings Analysis

```python
import boto3
from datetime import datetime, timedelta, timezone
from statistics import mean, stdev
from dataclasses import dataclass
from typing import Optional

ec2 = boto3.client("ec2", region_name="us-east-1")

@dataclass
class SpotPoolAnalysis:
    instance_type: str
    az: str
    current_price: float
    avg_price_7d: float
    price_stdev: float
    on_demand_price: float
    discount_pct: float
    volatility_coefficient: float  # stdev / mean — how much it varies relatively


def analyze_spot_pools(
    instance_types: list[str],
    on_demand_prices: dict[str, float],  # {instance_type: price}
    lookback_days: int = 7,
) -> list[SpotPoolAnalysis]:
    """
    Analyzes Spot price history to find the most stable and cheap pools.
    Note: low price + low volatility = pool with good consistent availability.
    """
    end = datetime.now(timezone.utc)
    start = end - timedelta(days=lookback_days)

    results = []
    for instance_type in instance_types:
        paginator = ec2.get_paginator("describe_spot_price_history")
        pages = paginator.paginate(
            InstanceTypes=[instance_type],
            ProductDescriptions=["Linux/UNIX"],
            StartTime=start,
            EndTime=end,
        )

        # Group by AZ
        prices_by_az: dict[str, list[float]] = {}
        for page in pages:
            for entry in page["SpotPriceHistory"]:
                az = entry["AvailabilityZone"]
                price = float(entry["SpotPrice"])
                prices_by_az.setdefault(az, []).append(price)

        od_price = on_demand_prices.get(instance_type, 0.0)

        for az, prices in prices_by_az.items():
            if len(prices) < 2:
                continue
            avg = mean(prices)
            sd = stdev(prices)
            cv = sd / avg if avg > 0 else 0  # coefficient of variation

            results.append(SpotPoolAnalysis(
                instance_type=instance_type,
                az=az,
                current_price=prices[0],  # most recent
                avg_price_7d=avg,
                price_stdev=sd,
                on_demand_price=od_price,
                discount_pct=((od_price - avg) / od_price * 100) if od_price > 0 else 0,
                volatility_coefficient=cv,
            ))

    # Sort: lowest volatility first, then lowest price
    return sorted(results, key=lambda r: (r.volatility_coefficient, r.avg_price_7d))


def print_pool_ranking(pools: list[SpotPoolAnalysis], top_n: int = 5):
    print(f"\n{'Type':15} {'AZ':20} {'Current':8} {'Avg7d':8} {'Discount':9} {'Volatility':13}")
    print("-" * 75)
    for p in pools[:top_n]:
        print(
            f"{p.instance_type:15} {p.az:20} "
            f"${p.current_price:.4f} ${p.avg_price_7d:.4f} "
            f"{p.discount_pct:6.1f}%    CV={p.volatility_coefficient:.3f}"
        )


# Usage example
if __name__ == "__main__":
    instance_types = [
        "c5.xlarge", "c5a.xlarge", "c5n.xlarge",
        "m5.xlarge", "m5a.xlarge",
    ]
    # On-Demand prices us-east-1 (reference — check at ec2.aws.amazon.com/pricing)
    od_prices = {
        "c5.xlarge":  0.170,
        "c5a.xlarge": 0.154,
        "c5n.xlarge": 0.216,
        "m5.xlarge":  0.192,
        "m5a.xlarge": 0.172,
    }

    pools = analyze_spot_pools(instance_types, od_prices, lookback_days=7)
    print_pool_ranking(pools, top_n=10)

    # Identify pools with discount > 50% and low volatility (CV < 0.10)
    prime_pools = [p for p in pools if p.discount_pct > 50 and p.volatility_coefficient < 0.10]
    print(f"\n'Prime' pools (>50% discount, CV<0.10): {len(prime_pools)}")
    for p in prime_pools:
        print(f"  → {p.instance_type} @ {p.az}: ${p.avg_price_7d:.4f}/h ({p.discount_pct:.1f}% off)")
```

---

## 9. CLI — Essential Examples

```bash
# 1. Spot price history — last 24h for specific types
aws ec2 describe-spot-price-history \
  --instance-types c5.xlarge m5.xlarge \
  --product-descriptions "Linux/UNIX" \
  --start-time $(date -u -d '24 hours ago' +%Y-%m-%dT%H:%M:%SZ) \
  --query 'SpotPriceHistory[*].{Type:InstanceType,AZ:AvailabilityZone,Price:SpotPrice,Time:Timestamp}' \
  --output table

# 2. Spot Placement Score — identify best region for 100 vCPUs
aws ec2 get-spot-placement-scores \
  --target-capacity 100 \
  --target-capacity-unit-type vcpu \
  --single-availability-zone-flag false \
  --instance-requirements-with-metadata '{
    "ArchitectureTypes": ["x86_64"],
    "VirtualizationTypes": ["hvm"],
    "InstanceRequirements": {
      "VCpuCount": {"Min": 2, "Max": 8},
      "MemoryMiB": {"Min": 4096, "Max": 32768},
      "InstanceGenerations": ["current"]
    }
  }' \
  --query 'SpotPlacementScores | sort_by(@, &Score) | reverse(@)' \
  --output table

# 3. Create EC2 Fleet (instant mode) with price-capacity-optimized
aws ec2 create-fleet \
  --type instant \
  --target-capacity-specification TotalTargetCapacity=10,DefaultTargetCapacityType=spot \
  --spot-options AllocationStrategy=price-capacity-optimized,MaxTotalPrice=1.50 \
  --launch-template-configs '[
    {
      "LaunchTemplateSpecification": {
        "LaunchTemplateId": "lt-0abcdef1234567890",
        "Version": "$Latest"
      },
      "Overrides": [
        {"InstanceType": "c5.xlarge",  "SubnetId": "subnet-aaa"},
        {"InstanceType": "c5a.xlarge", "SubnetId": "subnet-aaa"},
        {"InstanceType": "m5.xlarge",  "SubnetId": "subnet-bbb"},
        {"InstanceType": "m5a.xlarge", "SubnetId": "subnet-bbb"},
        {"InstanceType": "c5.xlarge",  "SubnetId": "subnet-ccc"},
        {"InstanceType": "m5.xlarge",  "SubnetId": "subnet-ccc"}
      ]
    }
  ]' \
  --query 'FleetId'

# 4. Describe fleet and its instances
aws ec2 describe-fleets \
  --fleet-ids fleet-0abc1234def56789 \
  --query 'Fleets[0].{State:FleetState,Capacity:TargetCapacitySpecification}'

aws ec2 describe-fleet-instances \
  --fleet-id fleet-0abc1234def56789

# 5. ASG — check Spot vs On-Demand instances in the mixed fleet
aws autoscaling describe-auto-scaling-instances \
  --query 'AutoScalingInstances[?AutoScalingGroupName==`WorkerASG`].[InstanceId,InstanceType,LifecycleState,HealthStatus]' \
  --output table

# 6. Cancel EC2 Fleet and its instances
aws ec2 delete-fleets \
  --fleet-ids fleet-0abc1234def56789 \
  --terminate-instances

# 7. Simulate interruption with AWS FIS (Fault Injection Service)
#    — useful for testing graceful shutdown in staging environments
aws fis create-experiment-template \
  --description "Test Spot interruption handling" \
  --actions '{"InterruptSpot": {
    "actionId": "aws:ec2:send-spot-instance-interruptions",
    "parameters": {"durationBeforeInterruption": "PT2M"},
    "targets": {"SpotInstances": "targetInstances"}
  }}' \
  --targets '{"targetInstances": {
    "resourceType": "aws:ec2:spot-instance",
    "resourceArns": ["arn:aws:ec2:us-east-1:123456789012:instance/i-0abc123"],
    "selectionMode": "ALL"
  }}' \
  --role-arn arn:aws:iam::123456789012:role/FISRole \
  --stop-conditions '[{"source":"none"}]'
```

---

## 10. Billing for Interrupted Instances

[FACT] As per official AWS documentation:

```
╔═══════════════════════════════════════════════════════════════════╗
║ Who interrupted          │ Partial hour billing                    ║
╠═══════════════════════════════════════════════════════════════════╣
║ AWS (Spot interruption)  │ First hour: NOT charged                 ║
║                          │ Subsequent partial hours: CHARGED       ║
╠═══════════════════════════════════════════════════════════════════╣
║ You (manual terminate)   │ Partial hour: CHARGED normally          ║
╚═══════════════════════════════════════════════════════════════════╝

Example: instance runs 2h 23min, AWS terminates → pay 2h (full hours)
         instance runs 0h 47min, AWS terminates → pay $0 (first hour)
         instance runs 1h 23min, you terminate → pay 1h 23min (pro-rated)
```

[FACT] For Spot Instances with `stop` and `hibernate` behavior: the instance stops accruing compute charges when stopped, but EBS and Elastic IPs continue to be charged.

---

## 11. Diagram: Interruption Handling Pipeline

```
Complete interruption flow in production:

 AWS decides         IMDS/EventBridge        Application
 to interrupt
     │
     ├──[early]──► EC2 Instance        ──► ASG launches replacement
     │             Rebalance Rec.           instance proactively
     │             (best-effort)
     │
     ├──[T-2min]─► EC2 Spot Instance   ──► Lambda EventBridge Handler:
     │             Interruption Warning      • Detach from ASG
     │             (guaranteed*)             • Notify SQS/SNS
     │                                       • Context logging
     │
     ├──[T-2min]─► IMDS polls (5s)     ──► spot-handler.sh:
     │             /spot/instance-action     • systemctl stop worker
     │                                       • flush pending work
     │                                       • upload logs to S3
     │
     └──[T=0]────► Instance terminates ──► ASG maintains desired capacity
                   (or stops/hibernates)    with new healthy instance

* Except hibernate: no 2-min notice (hibernates immediately)
```

---

## 12. Pitfalls

[FACT] **RequestSpotFleet is legacy**: new projects should use CreateFleet or Auto Scaling Groups. AWS documentation marks RequestSpotFleet as "no planned investment" — there is no guarantee of future support.

[FACT] **lowest-price has highest interruption risk**: the strategy focuses exclusively on price, without considering availability. Cheap pools tend to have high On-Demand demand → higher interruptions. Use `price-capacity-optimized`.

[FACT] **Misconfigured maxPrice increases interruptions**: specifying `maxPrice` causes more interruptions than not specifying it. If the Spot price exceeds your limit, the instance is interrupted. Most workloads should not set maxPrice.

[FACT] **Hibernate has no 2-min notice**: hibernation starts immediately upon receiving the signal — there is no 2-minute window for graceful shutdown. Use `terminate` or `stop` if you need time for cleanup.

[CONSENSUS] **Spot Instances are not suitable for primary databases**: stateful, tightly coupled, and interruption-intolerant. Use On-Demand or Reserved Instances for databases.

[FACT] **Capacity Rebalancing can cause temporary scale-out**: when launching a replacement before terminating the at-risk instance, the ASG is temporarily above desired capacity. This is expected and is not a bug.

[CONSENSUS] **Spot + Savings Plans do not stack**: Savings Plans cover On-Demand usage and Spot for Fargate/Lambda, but **do not cover Spot EC2**. For Spot EC2, the discount is already built into the Spot price.

[UNCERTAIN] **Interruption rate per pool**: the Spot Instance Advisor (console) shows historical interruption frequencies categorized (<5%, 5-10%, 10-15%, >20%). These categories are indicative but do not guarantee future behavior.

---

## 13. When NOT to Use Spot Instances

[FACT] AWS documentation is explicit — Spot Instances **are not suitable** for:

```
DO NOT use Spot for:
  ✗ Inflexible workloads (require exact instance type)
  ✗ Stateful workloads without checkpointing
  ✗ Tightly-coupled applications between nodes (HPC MPI without checkpoint)
  ✗ Primary database (stateful, intolerant)
  ✗ Workloads that cannot tolerate any period without full capacity
  ✗ Applications that depend on failover to On-Demand*

USE Spot for:
  ✓ Big data / ETL batch
  ✓ Stateless containers (ECS, EKS)
  ✓ CI/CD runners
  ✓ Stateless web servers (with ALB)
  ✓ HPC with checkpointing
  ✓ Rendering / encoding
  ✓ ML training (with checkpointing to S3)

* Failover Spot→OD can cause more interruptions in other Spot
```

---

## 14. Visual Summary

```
                    SPOT INSTANCES — CONCEPT MAP

  Price                                   Interruption
  ─────                                   ───────────
  Up to 90% discount                      AWS warns ~2 min before
  Varies by pool (type + AZ)              Rebalance Rec: early warning
  History: describe-spot-price-history    Polling IMDS every 5s
  Placement Score: 1-10 per region/AZ     EventBridge: 2 event types
                  │                                     │
                  ▼                                     ▼
         Diversification                      Handling Actions
         ─────────────                       ────────────────
         ≥ 10 instance types                 Graceful shutdown
         All AZs enabled                     Checkpoint to S3/DynamoDB
         ABITS: attributes, not fixed types  Detach from ASG
                  │                          Capacity Rebalancing
                  ▼
         Allocation Strategy
         ───────────────────
         price-capacity-optimized ← RECOMMENDED
         capacity-optimized       ← high interruption cost
         diversified              ← large/long-running fleets
         lowest-price             ← DO NOT USE
                  │
                  ▼
         Modern APIs
         ─────────────
         CreateAutoScalingGroup   ← with lifecycle
         CreateFleet (instant)    ← without auto scaling
         RequestSpotFleet         ← LEGACY, do not use
         RequestSpotInstances     ← LEGACY, do not use
```

---

## Reflection Exercise

An ML team uses an Auto Scaling Group with `lowest-price` policy and 3 instance types (`p3.2xlarge` only). Training takes 6 hours and has no checkpointing. The interruption rate is high (>20% of jobs are interrupted before completion), causing high reprocessing cost.

**Identify all design problems and propose a complete architectural solution**, considering:

1. Which allocation strategy should be used and why?
2. How should instance type diversification be done for GPUs?
3. What needs to change in the training job design to tolerate interruptions?
4. How does Capacity Rebalancing help (or not) in this specific scenario?
5. Is there any situation where Spot would be unsuitable even with all the above improvements?

---

## References

- [FACT] [Best practices for Amazon EC2 Spot](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-best-practices.html) — docs.aws.amazon.com (updated 2026)
- [FACT] [Spot Instance interruption notices](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-instance-termination-notices.html) — docs.aws.amazon.com
- [FACT] [EC2 instance rebalance recommendations](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/rebalance-recommendations.html) — docs.aws.amazon.com
- [FACT] [Allocation strategies for EC2 Fleet or Spot Fleet](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-fleet-allocation-strategy.html) — docs.aws.amazon.com
- [FACT] [Prepare for Spot Instance interruptions](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/prepare-for-interruptions.html) — docs.aws.amazon.com
- [FACT] [Spot placement score](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-placement-score.html) — docs.aws.amazon.com
