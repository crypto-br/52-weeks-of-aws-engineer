# Session 038 — CloudWatch: Composite Alarms, Anomaly Detection, and Automated Actions

**Estimated duration:** 60 minutes  
**Dependencies:** session-037-cloudwatch-logs-insights-queries

---

## Objective

By the end of this session, you will be able to create a Composite Alarm that only fires when errors **and** high latency occur simultaneously (reducing false positives), enable anomaly detection on a metric and adjust the tolerance band, configure alarm actions that invoke Lambda or SSM Automation (beyond SNS), and understand alarm states and `MISSING_DATA` semantics.

---

## Context

[FACT] A standard CloudWatch alarm monitors **a single metric** with a fixed threshold. Two advanced features extend this: **Composite Alarms** combine multiple alarms via logical expressions (AND, OR, NOT), and **Anomaly Detection** replaces the fixed threshold with a dynamic band calculated by ML based on the metric's history.

[CONSENSUS] The main motivation for Composite Alarms in production is to reduce **false positives**: a latency spike at 3 AM without an increase in errors rarely requires waking someone up. Composite Alarms allow modeling this reasoning: `ALARM(HighLatency) AND ALARM(HighErrorRate)` — only fires if both are true simultaneously.

---

## Core Concepts

### 1. Alarm States and Evaluation Semantics

[FACT] Every CloudWatch alarm (simple or composite) can be in one of three states:

```
State               Meaning
────────────────────────────────────────────────────────────────
OK                  Metric is within the threshold / band
ALARM               Metric has breached the threshold / band
INSUFFICIENT_DATA   Insufficient data to evaluate
                    (common in newly created alarms or metrics
                    with low-cardinality data)
```

[FACT] The **Datapoints to alarm (M out of N)** parameter controls sensitivity:

```
M of N — example: 3 of 5
  ┌─────────────────────────────────────────────────────┐
  │ Evaluation periods (N): 5                           │
  │ Datapoints to alarm (M): 3                          │
  │                                                     │
  │ Alarm only fires if 3 of the last 5 datapoints      │
  │ breach the threshold — reduces false positives       │
  │ on noisy metrics                                    │
  └─────────────────────────────────────────────────────┘

M = N (e.g.: 3 of 3): N consecutive periods breaching → alarm fires
M < N (e.g.: 2 of 5): less sensitive to isolated outliers
```

[FACT] **Missing data treatment** — behavior when data points are absent:

```
Value                   Behavior
────────────────────────────────────────────────────────────────
notBreaching            Absence = OK (missing data is normal)
breaching               Absence = ALARM (e.g.: heartbeat — absence is failure)
ignore                  Maintains current alarm state
missing                 Alarm goes to INSUFFICIENT_DATA
```

---

### 2. Composite Alarms: Rule Expression

[FACT] A Composite Alarm evaluates an **AlarmRule** — a boolean expression over the states of other alarms. Supports `ALARM()`, `OK()`, `INSUFFICIENT_DATA()` as state functions, and `AND`, `OR`, `NOT`, parentheses as logical operators.

```
AlarmRule examples:

-- Fires only if BOTH conditions are in ALARM
ALARM("payment-high-error-rate") AND ALARM("payment-high-latency")

-- Fires if either is in ALARM
ALARM("payment-high-error-rate") OR ALARM("payment-high-latency")

-- Fires for backend errors, unless maintenance is scheduled
ALARM("payment-high-error-rate") AND NOT ALARM("maintenance-window-active")

-- Complex expression with parentheses
(ALARM("payment-high-error-rate") OR ALARM("payment-high-latency"))
AND NOT OK("health-check-endpoint")

-- Composite can reference other Composite Alarms
ALARM("payment-composite") AND ALARM("database-composite")
```

[FACT] Composite Alarms have a cost: $0.50/alarm/month (us-east-1, in addition to the child alarm costs).

[FACT] Composite Alarms **do not evaluate metrics directly** — they only aggregate states from other alarms. The expression `ALARM("alarm-name")` checks whether the named alarm is in the ALARM state.

[FACT] **Dependency cycles**: two Composite Alarms depending on each other stop being evaluated. To break the cycle, change the `AlarmRule` of one of them to `FALSE`.

---

### 3. Anomaly Detection: Dynamic ML-Based Band

[FACT] Anomaly Detection analyzes a metric's history and creates a model of expected values, considering hourly, daily, and weekly patterns. The result is a **band** (lower/upper bound) around expected values.

```
Metric: RequestLatency (ms)
                                        ┌─ fixed threshold (rigid)
  ─── ─── ─── ─── ─── ─── ─── ─── ─── ┤
                                        │
     ╭─────────────────────────────╮   ← upper band (normal)
     │    expected values           │
  ───┤        (ML model)           ├─── ← expected values
     │                             │
     ╰─────────────────────────────╯   ← lower band (normal)
  
  Weekend: wider band (more variable traffic)
  Business hours: narrower band (more consistent pattern)
```

[FACT] The **Anomaly Detection Threshold** parameter (positive number, doesn't need to be integer) controls band width:
- Threshold = 1: narrow band, more sensitive to deviations
- Threshold = 2: moderate band (common default)
- Threshold = 3+: wide band, more tolerant

[FACT] The model takes **up to 2 weeks** to be fully trained. In the first 3 days, the band is an approximation. Excluding abnormal periods from training (deployments, incidents, holidays) improves model accuracy.

[FACT] You can configure the alarm to fire when the metric is:
- **Above the band** (useful for detecting anomalous high latency)
- **Below the band** (useful for detecting anomalous traffic drops)
- **Outside the band** (any deviation, up or down)

[FACT] Anomaly Detection accumulates cost per monitored metric: $0.30/month for detector creation (in addition to the normal metric cost) — us-east-1.

---

### 4. Alarm Actions Beyond SNS

[FACT] Available alarm actions by resource type:

```
Action Type         Trigger States      Usage
────────────────────────────────────────────────────────────────
SNS notification    ALARM, OK,          Fan-out to email, SMS,
                    INSUFFICIENT_DATA   Lambda, SQS, HTTP endpoint

Lambda invocation   ALARM, OK,          Automated remediation,
                    INSUFFICIENT_DATA   alert enrichment

EC2 action          ALARM, OK           Stop, Start, Reboot,
                    (EC2 alarms only)   Terminate the instance

SSM OpsItem         ALARM only          Creates item in OpsCenter
                                        for incident management

SSM Incident Manager ALARM only         Creates incident in AWS
                                        Incident Manager

Auto Scaling        ALARM, OK           Scale-out / scale-in
(via ScalingPolicy)                     (configured in the policy,
                                        not directly in the alarm)
```

[FACT] To invoke Lambda directly from an alarm (without SNS as intermediary): the alarm needs permission to call `lambda:InvokeFunction`. The permission is granted automatically by the console; via CDK/CLI, you need to create an explicit `lambda.Permission`.

---

## Practical Example

### Scenario: Payment System — Composite Alert + Automated Remediation

**Observability architecture:**
- Alarm 1: `HighErrorRate` — errors > 5% in 2 of 3 periods of 1 min
- Alarm 2: `HighLatency` (Anomaly Detection) — latency outside expected band
- Alarm 3: `MaintenanceWindow` — manual alarm to suppress notifications during maintenance
- Composite Alarm: fires when `HighErrorRate AND HighLatency AND NOT MaintenanceWindow`
- Action: Lambda that enriches the alert and creates an OpsItem in SSM

#### CDK Python — Complete Stack

```python
from aws_cdk import (
    Stack, Duration, RemovalPolicy,
    aws_cloudwatch as cw,
    aws_cloudwatch_actions as cwa,
    aws_lambda as lambda_,
    aws_sns as sns,
    aws_sns_subscriptions as sns_subs,
    aws_iam as iam,
)
from constructs import Construct


class PaymentAlarmsStack(Stack):
    def __init__(self, scope: Construct, construct_id: str, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        # ── Base metrics (emitted via EMF from session 036) ───────────
        error_metric = cw.Metric(
            namespace="MyApp/Payments",
            metric_name="FailedPayments",
            dimensions_map={"service": "payment-processor", "environment": "production"},
            statistic="Sum",
            period=Duration.minutes(1),
        )
        total_metric = cw.Metric(
            namespace="MyApp/Payments",
            metric_name="SuccessfulPayments",
            dimensions_map={"service": "payment-processor", "environment": "production"},
            statistic="Sum",
            period=Duration.minutes(1),
        )
        latency_metric = cw.Metric(
            namespace="MyApp/Payments",
            metric_name="ProcessingLatency",
            dimensions_map={"service": "payment-processor", "environment": "production"},
            statistic="p95",
            period=Duration.minutes(1),
        )

        # ── Metric Math: error rate (%) ───────────────────────────────
        error_rate_metric = cw.MathExpression(
            expression="100 * errors / (errors + successes)",
            using_metrics={
                "errors":    error_metric,
                "successes": total_metric,
            },
            period=Duration.minutes(1),
            label="Error Rate (%)",
        )

        # ── Alarm 1: High error rate (fixed threshold) ────────────────
        high_error_alarm = cw.Alarm(
            self, "HighErrorRateAlarm",
            alarm_name="payment-high-error-rate",
            alarm_description=(
                "Payment error rate > 5% for 2 of 3 consecutive minutes.\n"
                "**Runbook:** https://wiki.internal/runbooks/payment-errors"
            ),
            metric=error_rate_metric,
            threshold=5.0,
            evaluation_periods=3,
            datapoints_to_alarm=2,          # 2 of 3: tolerant to outliers
            comparison_operator=cw.ComparisonOperator.GREATER_THAN_THRESHOLD,
            treat_missing_data=cw.TreatMissingData.NOT_BREACHING,
        )

        # ── Alarm 2: Anomalous latency (Anomaly Detection) ───────────
        # CfnAnomalyDetector configures the ML model
        anomaly_detector = cw.CfnAnomalyDetector(
            self, "LatencyAnomalyDetector",
            namespace="MyApp/Payments",
            metric_name="ProcessingLatency",
            stat="p95",
            dimensions=[
                cw.CfnAnomalyDetector.DimensionProperty(
                    name="service",  value="payment-processor"
                ),
                cw.CfnAnomalyDetector.DimensionProperty(
                    name="environment", value="production"
                ),
            ],
            # Exclude maintenance windows from model training
            # configuration=cw.CfnAnomalyDetector.ConfigurationProperty(
            #     excluded_time_ranges=[
            #         cw.CfnAnomalyDetector.RangeProperty(
            #             start_time="2024-01-01T02:00:00",
            #             end_time="2024-01-01T04:00:00",
            #         )
            #     ]
            # ),
        )

        # Alarm based on the anomaly detector band
        high_latency_alarm = cw.CfnAlarm(
            self, "HighLatencyAnomalyAlarm",
            alarm_name="payment-high-latency-anomaly",
            alarm_description=(
                "Payment p95 latency is outside the expected band "
                "(anomaly detection threshold=2)."
            ),
            metrics=[
                cw.CfnAlarm.MetricDataQueryProperty(
                    id="m1",
                    metric_stat=cw.CfnAlarm.MetricStatProperty(
                        metric=cw.CfnAlarm.MetricProperty(
                            namespace="MyApp/Payments",
                            metric_name="ProcessingLatency",
                            dimensions=[
                                cw.CfnAlarm.DimensionProperty(
                                    name="service", value="payment-processor"
                                ),
                                cw.CfnAlarm.DimensionProperty(
                                    name="environment", value="production"
                                ),
                            ],
                        ),
                        stat="p95",
                        period=60,
                    ),
                    return_data=True,
                ),
                cw.CfnAlarm.MetricDataQueryProperty(
                    id="ad1",
                    expression="ANOMALY_DETECTION_BAND(m1, 2)",  # threshold=2
                    label="Anomaly Band",
                    return_data=True,
                ),
            ],
            comparison_operator="GreaterThanUpperThreshold",
            threshold_metric_id="ad1",
            evaluation_periods=3,
            datapoints_to_alarm=2,
            treat_missing_data="notBreaching",
        )

        # ── Alarm 3: Maintenance window (manual control) ─────────────
        # This alarm is manually set to ALARM during maintenance
        # to suppress notifications from the Composite Alarm
        maintenance_alarm = cw.Alarm(
            self, "MaintenanceWindowAlarm",
            alarm_name="payment-maintenance-window",
            alarm_description=(
                "Set this alarm to ALARM manually during planned maintenance "
                "to suppress composite alarm notifications."
            ),
            metric=cw.Metric(
                namespace="MyApp/Payments",
                metric_name="MaintenanceWindowActive",
                dimensions_map={"service": "payment-processor"},
                statistic="Sum",
                period=Duration.minutes(1),
            ),
            threshold=0,
            evaluation_periods=1,
            comparison_operator=cw.ComparisonOperator.GREATER_THAN_THRESHOLD,
            treat_missing_data=cw.TreatMissingData.NOT_BREACHING,
        )

        # ── Remediation Lambda ────────────────────────────────────────
        remediation_fn = lambda_.Function(
            self, "AlarmRemediationFn",
            function_name="payment-alarm-remediation",
            runtime=lambda_.Runtime.PYTHON_3_12,
            handler="handler.lambda_handler",
            code=lambda_.Code.from_asset("lambda/alarm_remediation"),
            timeout=Duration.seconds(30),
        )

        # Permission for CloudWatch to invoke the Lambda
        remediation_fn.add_permission(
            "CloudWatchInvoke",
            principal=iam.ServicePrincipal("lambda.alarms.cloudwatch.amazonaws.com"),
            source_arn=f"arn:aws:cloudwatch:{self.region}:{self.account}:alarm:payment-critical-composite",
        )

        # ── SNS Topic for notifications ───────────────────────────────
        ops_topic = sns.Topic(
            self, "OpsTopic",
            topic_name="payment-ops-alerts",
        )
        ops_topic.add_subscription(
            sns_subs.EmailSubscription("ops-team@company.com")
        )

        # ── Composite Alarm ───────────────────────────────────────────
        composite_alarm = cw.CfnCompositeAlarm(
            self, "PaymentCriticalComposite",
            alarm_name="payment-critical-composite",
            alarm_description=(
                "Critical: payment errors AND latency anomaly detected simultaneously.\n"
                "Suppressed during maintenance window.\n"
                "**Runbook:** https://wiki.internal/runbooks/payment-critical"
            ),
            alarm_rule=(
                f'ALARM("{high_error_alarm.alarm_name}") '
                f'AND ALARM("{high_latency_alarm.alarm_name}") '
                f'AND NOT ALARM("{maintenance_alarm.alarm_name}")'
            ),
            alarm_actions=[
                ops_topic.topic_arn,
                remediation_fn.function_arn,
            ],
            ok_actions=[ops_topic.topic_arn],
            # actions_suppressor: another alarm that suppresses actions (advanced feature)
            # actions_suppressor="payment-maintenance-window",
        )
```

#### Lambda Handler — Automated Remediation

```python
# lambda/alarm_remediation/handler.py
import json
import boto3
from typing import Any

ssm = boto3.client("ssm")
sns = boto3.client("sns")


def lambda_handler(event: dict, context: Any) -> None:
    """
    Invoked directly by CloudWatch Alarm when it changes state.
    The event contains: AlarmName, NewStateValue, NewStateReason, StateChangeTime,
                        OldStateValue, Trigger (threshold/band info)
    """
    print(json.dumps(event))

    alarm_name  = event.get("alarmData", {}).get("alarmName", event.get("AlarmName", "unknown"))
    new_state   = event.get("alarmData", {}).get("state", {}).get("value", event.get("NewStateValue", "unknown"))
    reason      = event.get("alarmData", {}).get("state", {}).get("reason", event.get("NewStateReason", ""))
    change_time = event.get("alarmData", {}).get("state", {}).get("timestamp", event.get("StateChangeTime", ""))

    if new_state != "ALARM":
        # Remediation action only makes sense when entering ALARM
        print(f"Alarm {alarm_name} changed to {new_state} — no action needed")
        return

    print(f"ALARM triggered: {alarm_name} at {change_time}")
    print(f"Reason: {reason}")

    # 1. Create OpsItem in SSM OpsCenter for tracking
    try:
        ssm.create_ops_item(
            Title=f"Payment Critical Alarm: {alarm_name}",
            Description=(
                f"CloudWatch Composite Alarm triggered at {change_time}.\n\n"
                f"Reason: {reason}\n\n"
                "Runbook: https://wiki.internal/runbooks/payment-critical"
            ),
            Source="cloudwatch",
            OperationalData={
                "/aws/resources": {
                    "Value": json.dumps([{
                        "arn": f"arn:aws:cloudwatch::alarm:{alarm_name}"
                    }]),
                    "Type": "SearchableString",
                }
            },
            Severity="2",   # 1=Critical, 2=High, 3=Medium, 4=Low
            Category="Availability",
            Tags=[
                {"Key": "Service",     "Value": "payment-processor"},
                {"Key": "AlarmName",   "Value": alarm_name},
                {"Key": "Environment", "Value": "production"},
            ],
        )
        print("SSM OpsItem created successfully")
    except Exception as e:
        print(f"Failed to create SSM OpsItem: {e}")

    # 2. Additional enrichment: e.g. check if there's a recent deploy in progress
    # (call to CodeDeploy, Deployment, etc. omitted for brevity)
```

#### CLI — Create and Manage Alarms

```bash
# 1. Create Composite Alarm via CLI
aws cloudwatch put-composite-alarm \
  --alarm-name "payment-critical-composite" \
  --alarm-description "Fires when errors AND anomalous latency occur simultaneously" \
  --alarm-rule 'ALARM("payment-high-error-rate") AND ALARM("payment-high-latency-anomaly") AND NOT ALARM("payment-maintenance-window")' \
  --alarm-actions \
      "arn:aws:sns:us-east-1:123456789012:payment-ops-alerts" \
      "arn:aws:lambda:us-east-1:123456789012:function:payment-alarm-remediation" \
  --ok-actions "arn:aws:sns:us-east-1:123456789012:payment-ops-alerts"

# 2. Create anomaly detector for latency
aws cloudwatch put-anomaly-detector \
  --namespace "MyApp/Payments" \
  --metric-name "ProcessingLatency" \
  --stat "p95" \
  --dimensions Name=service,Value=payment-processor Name=environment,Value=production

# 3. Create alarm based on the anomaly detector
aws cloudwatch put-metric-alarm \
  --alarm-name "payment-high-latency-anomaly" \
  --metrics '[
    {
      "Id": "m1",
      "MetricStat": {
        "Metric": {
          "Namespace": "MyApp/Payments",
          "MetricName": "ProcessingLatency",
          "Dimensions": [
            {"Name": "service", "Value": "payment-processor"},
            {"Name": "environment", "Value": "production"}
          ]
        },
        "Stat": "p95",
        "Period": 60
      },
      "ReturnData": true
    },
    {
      "Id": "ad1",
      "Expression": "ANOMALY_DETECTION_BAND(m1, 2)",
      "Label": "ProcessingLatency (expected)",
      "ReturnData": true
    }
  ]' \
  --comparison-operator GreaterThanUpperThreshold \
  --threshold-metric-id ad1 \
  --evaluation-periods 3 \
  --datapoints-to-alarm 2 \
  --treat-missing-data notBreaching

# 4. Simulate maintenance: force alarm to ALARM state manually
# (puts maintenance alarm in ALARM to suppress composite)
aws cloudwatch set-alarm-state \
  --alarm-name "payment-maintenance-window" \
  --state-value ALARM \
  --state-reason "Planned maintenance window 02:00-04:00 UTC"

# 5. Restore after maintenance
aws cloudwatch set-alarm-state \
  --alarm-name "payment-maintenance-window" \
  --state-value OK \
  --state-reason "Maintenance window completed"

# 6. List states of all payment alarms
aws cloudwatch describe-alarms \
  --alarm-name-prefix "payment-" \
  --query 'MetricAlarms[*].{Name:AlarmName,State:StateValue,Reason:StateReason}'

aws cloudwatch describe-alarms \
  --alarm-types CompositeAlarm \
  --alarm-name-prefix "payment-" \
  --query 'CompositeAlarms[*].{Name:AlarmName,State:StateValue,Rule:AlarmRule}'

# 7. Exclude anomalous period from anomaly detection model
# (e.g.: exclude yesterday's incident window from training)
YESTERDAY_START=$(date -u -d 'yesterday 02:00' '+%Y-%m-%dT%H:%M:%S' 2>/dev/null || date -u -v-1d '+%Y-%m-%dT02:00:00')
YESTERDAY_END=$(date -u -d 'yesterday 06:00' '+%Y-%m-%dT%H:%M:%S' 2>/dev/null || date -u -v-1d '+%Y-%m-%dT06:00:00')

aws cloudwatch put-anomaly-detector \
  --namespace "MyApp/Payments" \
  --metric-name "ProcessingLatency" \
  --stat "p95" \
  --dimensions Name=service,Value=payment-processor Name=environment,Value=production \
  --configuration "{
    \"ExcludedTimeRanges\": [
      {\"StartTime\": \"${YESTERDAY_START}\", \"EndTime\": \"${YESTERDAY_END}\"}
    ]
  }"
```

---

## Common Pitfalls

**1. Dependency cycles in Composite Alarms**  
[FACT] Two Composite Alarms referencing each other create a cycle that paralyzes the evaluation of both. It is not possible to delete alarms in a cycle without first breaking the dependency — set `AlarmRule = FALSE` on one of them via `set-alarm-state` or `put-composite-alarm` and then delete.

**2. Anomaly Detection takes up to 2 weeks for maximum accuracy**  
[FACT] In the first hours after detector creation, the band is a rough estimate. Creating alarms based on anomaly detection immediately after deployment can generate frequent false alarms. Wait at least 3 days for the band to stabilize; 2 weeks for ideal accuracy.

**3. `set-alarm-state` is temporary — CloudWatch overwrites on the next cycle**  
[FACT] `set-alarm-state` forces the alarm state but that state is **overwritten on the next metric evaluation** (in the next evaluation period). It is useful for tests and temporary manual maintenance, but does not persist. For suppression during maintenance, use the Maintenance Window Alarm pattern described above.

**4. Lambda action requires explicit permission — console handles it, CDK does not**  
[FACT] The CloudWatch console automatically adds the `lambda:InvokeFunction` permission for the principal `lambda.alarms.cloudwatch.amazonaws.com`. Via CDK or CLI, you need to explicitly create a `lambda.Permission` with `source_arn` pointing to the alarm ARN. Without this, the alarm creates the action but invocation fails silently (no visible error in the alarm — the error only appears in the Lambda's CloudWatch Logs).

**5. Actions Suppressor vs. NOT in AlarmRule**  
[CONSENSUS] There are two mechanisms to suppress Composite Alarm actions: (1) include `NOT ALARM("maintenance-alarm")` in the AlarmRule — the alarm changes state but does not execute actions; (2) use `ActionsSuppressor` in `put-composite-alarm` — the alarm changes state AND suppresses actions with a configurable "wait" mechanism. The second is more robust for long maintenance windows as it allows configuring `ActionsSuppressorWaitPeriod` and `ActionsSuppressorExtensionPeriod`.

**6. Anomaly detection cost is non-trivial at scale**  
[FACT] Each detector costs $0.30/month (in addition to the metric cost). 100 metrics with anomaly detection = $30/month in detectors alone. For workloads with hundreds of microservices and multiple metrics per service, cost scales quickly. Prioritize anomaly detection for critical SLO metrics.

---

## Reflection Exercise

You have three microservices — `checkout`, `inventory`, and `notification` — each with individual `HighErrorRate` and `HighLatency` alarms. You want to create an observability architecture that:

1. Alerts distinctly when only one service is degraded vs. when multiple services are degraded simultaneously (indicating a shared infrastructure problem, not an individual service issue).

2. Model the correct AlarmRule for a "CriticalSystemDegradation" Composite Alarm that fires when **two or more** of the three services have `HighErrorRate` in ALARM simultaneously. Write the logical expression — hint: `A AND B OR A AND C OR B AND C` is equivalent to "2 or more of 3".

3. The `notification` service has very low traffic at 3h-6h UTC (fewer than 10 requests/minute). During this period, `HighErrorRate` frequently goes to `INSUFFICIENT_DATA`. How would you configure `treat_missing_data` for this alarm, and what impact would it have on the Composite Alarm? Is there any risk in this choice?

---

## Resources for Further Study

- [FACT] Create a Composite Alarm: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Create_Composite_Alarm.html
- [FACT] Create an alarm based on anomaly detection: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Create_Anomaly_Detection_Alarm.html
- [FACT] Using CloudWatch Anomaly Detection: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Anomaly_Detection.html
- [FACT] Alarm evaluation and missing data: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/alarms-and-missing-data.html
- [FACT] PutCompositeAlarm API (AlarmRule syntax): https://docs.aws.amazon.com/AmazonCloudWatch/latest/APIReference/API_PutCompositeAlarm.html

---
