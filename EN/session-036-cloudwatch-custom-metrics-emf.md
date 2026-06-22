# Session 036 — CloudWatch: Custom Metrics with EMF (Embedded Metrics Format)

**Estimated duration:** 60 minutes  
**Dependencies:** session-018-ecs-observabilidade-firelens-xray, session-024-lambda-observabilidade-xray-insights

---

## Objective

By the end of this session, you will be able to emit custom metrics from a Lambda function using the EMF format (via `print()` to stdout in structured JSON, without a separate API call), create a CloudWatch dashboard that plots these metrics, explain why EMF is preferable to `put-metric-data` in terms of latency and cost, and use AWS Lambda Powertools to simplify emission.

---

## Context

[FACT] The **Embedded Metrics Format (EMF)** is a CloudWatch JSON specification that instructs the CloudWatch Logs service to automatically extract custom metrics from structured log events. Instead of making a separate API call to CloudWatch, the function writes a specially formatted JSON to stdout — the Lambda Logs Agent captures this and forwards it to CloudWatch Logs, which extracts the metrics asynchronously.

[CONSENSUS] EMF is the canonical approach for custom metrics in Lambda for three reasons: (1) it does not block function execution (asynchronous via logs); (2) it eliminates the cost of `PutMetricData` per API call; (3) the EMF JSON is also available as a log event in CloudWatch Logs Insights for debugging.

---

## Main concepts

### 1. EMF document structure

[FACT] A valid EMF document is a JSON with the `_aws` key as mandatory metadata, plus the metric values and dimensions as fields at the root level:

```json
{
  "_aws": {
    "Timestamp": 1574109732004,
    "CloudWatchMetrics": [
      {
        "Namespace": "MyApp/Payments",
        "Dimensions": [["service", "environment"]],
        "Metrics": [
          { "Name": "ProcessingLatency", "Unit": "Milliseconds", "StorageResolution": 60 },
          { "Name": "SuccessfulPayments",  "Unit": "Count",        "StorageResolution": 60 }
        ]
      }
    ]
  },
  "service":             "payment-processor",
  "environment":         "production",
  "ProcessingLatency":   45.3,
  "SuccessfulPayments":  1,
  "orderId":             "ord-abc-123"
}
```

[FACT] EMF structural rules:
- `_aws.Timestamp`: epoch in **milliseconds** (mandatory)
- `_aws.CloudWatchMetrics`: array of `MetricDirective` (mandatory)
- Each `MetricDirective` must have `Namespace`, `Dimensions` (array of arrays of strings), and `Metrics` (array of `MetricDefinition`)
- Maximum of **100 metrics** per EMF document
- Maximum of **30 dimensions** per DimensionSet
- Metric values: number or array of numbers (maximum 100 elements)
- Extra fields beyond metrics/dimensions (like `orderId` above) are **preserved in the log** but **do not become metrics** — they serve as context for Logs Insights

[FACT] Valid units: `Seconds`, `Microseconds`, `Milliseconds`, `Bytes`, `Kilobytes`, `Megabytes`, `Gigabytes`, `Terabytes`, `Bits`, `Kilobits`, `Megabits`, `Gigabits`, `Terabits`, `Percent`, `Count`, `Bytes/Second`, `Kilobytes/Second`, `Megabytes/Second`, `Gigabytes/Second`, `Terabytes/Second`, `Bits/Second`, `Count/Second`, `None`

---

### 2. EMF vs. PutMetricData: cost and latency comparison

[FACT] There are two mechanisms for ingesting custom metrics into CloudWatch:

```
                    EMF (via CloudWatch Logs)     PutMetricData API
────────────────────────────────────────────────────────────────────
Execution           Asynchronous — print() returns Synchronous — blocks
                    immediately                    until API response
Latency in function None added                    Network latency
                                                  (typically 10-50ms)
Write cost          Log ingestion cost            $0.01 per 1,000 API calls
                    ($0.50/GB — us-east-1)        (regardless of volume)
Metric cost         Same: $0.30/metric/month      Same: $0.30/metric/month
                    (first 10,000)                (first 10,000)
Permission required logs:PutLogEvents             cloudwatch:PutMetricData
Context data        Full log event available      Only metric data
                    in Logs Insights
Batch limit         100 metrics per EMF blob      20 metrics per
                                                  PutMetricData call
```

[CONSENSUS] For Lambda functions with high invocation rate, EMF is financially superior because you don't pay per API call — you only pay for log ingestion (which would occur anyway). The break-even point is low: any function with more than ~1,000 invocations/day typically saves with EMF.

[OPINION — AWS Well-Architected Serverless Lens] EMF is the recommended approach for custom metrics in serverless workloads by eliminating synchronous calls that increase the billable duration of the function.

---

### 3. High-resolution metrics

[FACT] The `StorageResolution` field in MetricDefinition defines storage granularity:

```
StorageResolution = 60  →  Standard resolution: CloudWatch stores at
                            1-minute granularity
                            (default, lower cost)

StorageResolution = 1   →  High resolution: CloudWatch stores at
                            1-second granularity
                            (useful for real-time anomaly detection,
                            sub-minute alerts)
```

[FACT] High-resolution metrics are charged the same as standard in terms of metric cost ($0.30/metric/month), but consume more internal storage. Alarms on high-resolution metrics can have a minimum period of 10 or 30 seconds.

---

### 4. Unique metric = name + namespace + dimensions

[FACT] CloudWatch defines a **unique metric** by the combination of: `metric_name` + `namespace` + `{dimension_key: dimension_value, ...}`. This distinction is critical for understanding cost:

```
Metric A:  Namespace="MyApp", Name="Latency", {service="payment"}
Metric B:  Namespace="MyApp", Name="Latency", {service="checkout"}
→ These are 2 distinct metrics, charged separately.

High cardinality pitfall:
Namespace="MyApp", Name="Latency", {requestId="abc-123"}
Namespace="MyApp", Name="Latency", {requestId="def-456"}
→ Each unique requestId creates a new metric!
   1M requests/day = potentially 1M new metrics/day
   = explosive cost ($0.30 × 1,000,000 = $300,000/month)
```

[FACT] High-cardinality dimensions (`requestId`, `userId`, `sessionId`) **must never be used as dimensions**. Use them as **metadata fields** in EMF (they go in the log, searchable via Logs Insights, without metric cost).

---

### 5. AWS Lambda Powertools — Metrics utility

[FACT] The `aws_lambda_powertools.metrics` module is an abstraction over EMF that: validates the schema, serializes the JSON, does flush in the decorator `@log_metrics`, and prevents accidental creation of metrics with high-cardinality dimensions.

```
Metrics behavior (singleton shared between modules):
  - Accumulates metrics in memory during invocation
  - Automatic flush at end of handler (via @log_metrics)
  - Automatic flush when reaching 100 metrics (EMF limit)
  - Validates units, namespace, dimensions on emission

EphemeralMetrics (non-singleton):
  - Isolated instance — does not share state
  - Useful for multi-tenant or metrics with completely distinct dimensions
```

---

## Practical example

### Scenario: payments API — business and operational metrics

#### CDK Python — Lambda + Powertools Layer + Dashboard

```python
from aws_cdk import (
    Stack, Duration, RemovalPolicy,
    aws_lambda as lambda_,
    aws_logs as logs,
    aws_cloudwatch as cw,
    aws_cloudwatch_actions as cw_actions,
    aws_sns as sns,
)
from constructs import Construct


class PaymentMetricsStack(Stack):
    def __init__(self, scope: Construct, construct_id: str, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        # ── Lambda Powertools Layer (official ARN per region/version) ──
        # See: https://docs.aws.amazon.com/powertools/python/latest/
        powertools_layer = lambda_.LayerVersion.from_layer_version_arn(
            self, "PowertoolsLayer",
            layer_version_arn=(
                f"arn:aws:lambda:{self.region}:017000801446:"
                "layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:31"
            ),
        )

        # ── Lambda Function ───────────────────────────────────────────
        payment_fn = lambda_.Function(
            self, "PaymentProcessor",
            function_name="payment-processor",
            runtime=lambda_.Runtime.PYTHON_3_12,
            handler="handler.lambda_handler",
            code=lambda_.Code.from_asset("lambda/payment"),
            timeout=Duration.seconds(30),
            memory_size=256,
            layers=[powertools_layer],
            environment={
                "POWERTOOLS_SERVICE_NAME":      "payment-processor",
                "POWERTOOLS_METRICS_NAMESPACE": "MyApp/Payments",
                "ENVIRONMENT":                  "production",
                "LOG_LEVEL":                    "INFO",
            },
            log_retention=logs.RetentionDays.ONE_WEEK,
        )

        # ── CloudWatch Dashboard ──────────────────────────────────────
        dashboard = cw.Dashboard(
            self, "PaymentDashboard",
            dashboard_name="payment-metrics",
        )

        # Widget 1: Success vs failure rate
        dashboard.add_widgets(
            cw.GraphWidget(
                title="Payment Success vs Failure Rate",
                width=12,
                left=[
                    cw.Metric(
                        namespace="MyApp/Payments",
                        metric_name="SuccessfulPayments",
                        dimensions_map={"service": "payment-processor", "environment": "production"},
                        statistic="Sum",
                        period=Duration.minutes(1),
                        color="#2ca02c",
                    ),
                    cw.Metric(
                        namespace="MyApp/Payments",
                        metric_name="FailedPayments",
                        dimensions_map={"service": "payment-processor", "environment": "production"},
                        statistic="Sum",
                        period=Duration.minutes(1),
                        color="#d62728",
                    ),
                ],
            ),
            # Widget 2: Processing latency (p50, p95, p99)
            cw.GraphWidget(
                title="Processing Latency (ms)",
                width=12,
                left=[
                    cw.Metric(
                        namespace="MyApp/Payments",
                        metric_name="ProcessingLatency",
                        dimensions_map={"service": "payment-processor", "environment": "production"},
                        statistic="p50",
                        period=Duration.minutes(1),
                        label="p50",
                        color="#1f77b4",
                    ),
                    cw.Metric(
                        namespace="MyApp/Payments",
                        metric_name="ProcessingLatency",
                        dimensions_map={"service": "payment-processor", "environment": "production"},
                        statistic="p95",
                        period=Duration.minutes(1),
                        label="p95",
                        color="#ff7f0e",
                    ),
                    cw.Metric(
                        namespace="MyApp/Payments",
                        metric_name="ProcessingLatency",
                        dimensions_map={"service": "payment-processor", "environment": "production"},
                        statistic="p99",
                        period=Duration.minutes(1),
                        label="p99",
                        color="#d62728",
                    ),
                ],
            ),
        )

        # ── Alarm on failures ─────────────────────────────────────────
        alarm_topic = sns.Topic(self, "PaymentAlarmTopic")

        cw.Alarm(
            self, "HighFailureRateAlarm",
            alarm_name="payment-high-failure-rate",
            metric=cw.Metric(
                namespace="MyApp/Payments",
                metric_name="FailedPayments",
                dimensions_map={"service": "payment-processor", "environment": "production"},
                statistic="Sum",
                period=Duration.minutes(5),
            ),
            threshold=10,
            evaluation_periods=2,
            comparison_operator=cw.ComparisonOperator.GREATER_THAN_OR_EQUAL_TO_THRESHOLD,
            treat_missing_data=cw.TreatMissingData.NOT_BREACHING,
        ).add_alarm_action(cw_actions.SnsAction(alarm_topic))
```

#### Lambda handler — EMF via Powertools and manual EMF

```python
# lambda/payment/handler.py
import json
import os
import time
from typing import Any

from aws_lambda_powertools import Logger, Metrics
from aws_lambda_powertools.metrics import MetricUnit, MetricResolution
from aws_lambda_powertools.utilities.typing import LambdaContext

# Initialized globally (shared singleton)
# Namespace and service come from env vars:
#   POWERTOOLS_METRICS_NAMESPACE="MyApp/Payments"
#   POWERTOOLS_SERVICE_NAME="payment-processor"
logger = Logger()
metrics = Metrics()

# Additional shared dimension for all metrics
ENVIRONMENT = os.environ.get("ENVIRONMENT", "development")
metrics.set_default_dimensions(environment=ENVIRONMENT)


@metrics.log_metrics(capture_cold_start_metric=True)
@logger.inject_lambda_context
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    """
    @metrics.log_metrics:
      - Serializes and flushes EMF blob to stdout at handler end
      - Captures cold start in separate EMF blob (function_name dimension)
      - If exception occurs, flush is still executed
    """
    start_time = time.perf_counter()

    order_id   = event.get("orderId", "unknown")
    amount_usd = float(event.get("amountUSD", 0))
    currency   = event.get("currency", "USD")

    try:
        # Simulate processing
        result = _process_payment(order_id, amount_usd, currency)

        # ── Business metrics ─────────────────────────────────────────
        metrics.add_metric(
            name="SuccessfulPayments",
            unit=MetricUnit.Count,
            value=1,
        )
        metrics.add_metric(
            name="PaymentValueUSD",
            unit=MetricUnit.Count,      # CloudWatch has no "Currency"
            value=amount_usd,
        )

        # ── High-cardinality metadata (IN log, NOT a dimension) ───────
        # orderId goes to the log event but does NOT create a new metric per invocation
        metrics.add_metadata(key="orderId",   value=order_id)
        metrics.add_metadata(key="currency",  value=currency)
        metrics.add_metadata(key="processor", value=result.get("processor"))

        return {"statusCode": 200, "body": json.dumps(result)}

    except ValueError as e:
        # Order validation error
        metrics.add_metric(name="FailedPayments", unit=MetricUnit.Count, value=1)
        metrics.add_metadata(key="errorType",    value="ValidationError")
        metrics.add_metadata(key="errorMessage", value=str(e))
        metrics.add_metadata(key="orderId",      value=order_id)
        logger.error("Payment validation failed", extra={"orderId": order_id, "error": str(e)})
        return {"statusCode": 400, "body": json.dumps({"error": str(e)})}

    except Exception as e:
        metrics.add_metric(name="FailedPayments", unit=MetricUnit.Count, value=1)
        metrics.add_metadata(key="errorType",    value=type(e).__name__)
        metrics.add_metadata(key="orderId",      value=order_id)
        logger.exception("Unexpected payment error", extra={"orderId": order_id})
        return {"statusCode": 500, "body": json.dumps({"error": "Internal error"})}

    finally:
        # Latency always emitted, regardless of success/failure
        elapsed_ms = (time.perf_counter() - start_time) * 1000
        metrics.add_metric(
            name="ProcessingLatency",
            unit=MetricUnit.Milliseconds,
            value=elapsed_ms,
            resolution=MetricResolution.High,  # StorageResolution=1: sub-minute
        )


def _process_payment(order_id: str, amount: float, currency: str) -> dict:
    if amount <= 0:
        raise ValueError(f"Invalid amount: {amount}")
    if currency not in ("USD", "EUR", "BRL"):
        raise ValueError(f"Unsupported currency: {currency}")
    time.sleep(0.02)  # simulate external call
    return {"orderId": order_id, "status": "approved", "processor": "stripe"}
```

#### Manual EMF without Powertools (to understand the mechanism)

```python
import json
import time

def emit_emf_manually(metric_name: str, value: float, unit: str,
                      namespace: str, dimensions: dict) -> None:
    """
    Emits an EMF metric via print() — without any external dependency.
    The Lambda Logs Agent captures stdout and sends to CloudWatch Logs.
    CloudWatch Logs extracts the metric automatically.
    """
    emf_document = {
        "_aws": {
            "Timestamp": int(time.time() * 1000),   # epoch in milliseconds
            "CloudWatchMetrics": [
                {
                    "Namespace": namespace,
                    "Dimensions": [list(dimensions.keys())],  # array of arrays
                    "Metrics": [
                        {"Name": metric_name, "Unit": unit, "StorageResolution": 60}
                    ],
                }
            ],
        },
        metric_name: value,
        **dimensions,   # dimensions at root level
    }
    # One line per EMF document (no internal line breaks)
    print(json.dumps(emf_document))
```

#### CLI — Create metrics manually and create dashboard

```bash
# 1. Emit a metric via PutMetricData (for comparison with EMF)
aws cloudwatch put-metric-data \
  --namespace "MyApp/Payments" \
  --metric-name "ManualTestMetric" \
  --value 42 \
  --unit Count \
  --dimensions service=payment-processor,environment=production

# 2. Verify if EMF metrics were created
aws cloudwatch list-metrics \
  --namespace "MyApp/Payments" \
  --query 'Metrics[*].{Name:MetricName,Dims:Dimensions}'

# 3. Get latency statistics (last 15 minutes)
aws cloudwatch get-metric-statistics \
  --namespace "MyApp/Payments" \
  --metric-name "ProcessingLatency" \
  --dimensions Name=service,Value=payment-processor \
               Name=environment,Value=production \
  --start-time "$(date -u -d '15 minutes ago' '+%Y-%m-%dT%H:%M:%SZ' 2>/dev/null || date -u -v-15M '+%Y-%m-%dT%H:%M:%SZ')" \
  --end-time "$(date -u '+%Y-%m-%dT%H:%M:%SZ')" \
  --period 60 \
  --statistics Average Minimum Maximum p95 p99 \
  --extended-statistics p95 p99

# 4. Create alarm on high error rate
aws cloudwatch put-metric-alarm \
  --alarm-name "payment-high-failure-rate" \
  --alarm-description "More than 10 payment failures in 5 minutes" \
  --namespace "MyApp/Payments" \
  --metric-name "FailedPayments" \
  --dimensions Name=service,Value=payment-processor Name=environment,Value=production \
  --statistic Sum \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 10 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --treat-missing-data notBreaching

# 5. Check EMF JSON generated by Lambda in logs
LOG_GROUP="/aws/lambda/payment-processor"
LATEST_STREAM=$(aws logs describe-log-streams \
  --log-group-name "$LOG_GROUP" \
  --order-by LastEventTime \
  --descending \
  --limit 1 \
  --query 'logStreams[0].logStreamName' \
  --output text)

aws logs get-log-events \
  --log-group-name "$LOG_GROUP" \
  --log-stream-name "$LATEST_STREAM" \
  --limit 20 \
  --query 'events[*].message' \
  --output text | grep -A5 '"_aws"' | head -50

# 6. Query metrics and orderId via Logs Insights
# (metadata fields are searchable even though they are not dimensions)
aws logs start-query \
  --log-group-name "/aws/lambda/payment-processor" \
  --start-time "$(date -u -d '1 hour ago' +%s 2>/dev/null || date -u -v-1H +%s)" \
  --end-time "$(date -u +%s)" \
  --query-string '
    fields @timestamp, orderId, ProcessingLatency, errorType
    | filter ispresent(FailedPayments)
    | sort @timestamp desc
    | limit 20
  '
```

---

## Common pitfalls

**1. High-cardinality dimensions = cost explosion**  
[FACT] Each unique combination of (namespace + metric_name + dimensions) is a distinct metric charged at $0.30/month (for the first 10,000). Using `requestId`, `userId` or `sessionId` as a dimension with 1M unique values/month generates 1M new metrics = $300,000/month. Use `add_metadata()` for high-cardinality data — it stays in the log, doesn't create metrics.

**2. Multi-line EMF is not processed**  
[FACT] The EMF document must be a single JSON object on **a single line** in stdout. If the JSON is pretty-printed (with line breaks), CloudWatch Logs does not recognize the format and silently discards metric extraction. Powertools guarantees this; in manual mode, use `json.dumps(doc)` without `indent`.

**3. Metrics or dimensions defined outside the handler (global scope)**  
[CONSENSUS] Dimensions or metrics added at global scope (`import` time) are only applied during cold start. In subsequent invocations, global state persists between calls on the same instance, but is not reinitialized. Powertools warns about this. Permanent dimensions should be configured via `set_default_dimensions()`.

**4. Missing flush on exception without the decorator**  
[FACT] If you use `metrics.flush_metrics()` manually (without the `@log_metrics` decorator), an uncaught exception in an intermediate block can leave metrics un-emitted. Always use `try/finally` or use the decorator, which guarantees flush even in case of exception.

**5. `StorageResolution=1` (high-resolution) without real need**  
[CONSENSUS] High-resolution metrics allow alarms with 10/30-second periods, but don't cost more in metrics — the extra cost is marginal in storage. The problem is that alarms on high-resolution metrics consume more evaluation reads, slightly increasing alarm cost. Use high-resolution only when sub-minute granularity is truly necessary for the SLA.

**6. Namespace with high granularity creates visibility silos**  
[CONSENSUS] Using very specific namespaces (e.g., `MyApp/Payments/OrderType/Subscription`) fragments metrics into silos that are difficult to correlate in dashboards. Use one namespace per service/application and use dimensions to segment — dimensions are filterable in the console and in queries.

---

## Reflection exercise

You have a Lambda function that processes events from an SQS queue. For each message, you want to monitor:
- `MessageProcessed` (Count) — by message type (`messageType`: order, refund, notification)
- `ProcessingDuration` (Milliseconds) — processing latency
- The original `messageId` for correlation with logs

1. How would you structure the dimensions for `MessageProcessed` considering there are 3 message types (`order`, `refund`, `notification`)? How many distinct metrics does this create in CloudWatch? What would be the problem if you used `messageId` as a dimension?

2. Write the Python code snippet with Powertools that emits `MessageProcessed` and `ProcessingDuration` with the `messageType` dimension, keeping `messageId` as metadata (not a dimension). Use `@metrics.log_metrics`.

3. The EMF blob generated by Powertools is also a log event in CloudWatch Logs. Write a Logs Insights query that lists the last 10 `messageId` of messages of type `refund` with `ProcessingDuration > 500ms`.

---

## Resources for further study

- [FACT] EMF Specification (JSON structure, limits, schema): https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Embedded_Metric_Format_Specification.html
- [FACT] Embedding metrics within logs (overview): https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Embedded_Metric_Format.html
- [FACT] Powertools for AWS Lambda (Python) — Metrics: https://docs.aws.amazon.com/powertools/python/latest/core/metrics/
- [FACT] CloudWatch Pricing (custom metrics, ingestion): https://aws.amazon.com/cloudwatch/pricing/

---
