# Session 024 — Lambda Observability: structured logging, X-Ray and Lambda Insights

**Estimated duration:** 60 minutes
**Prerequisites:** session-023-stepfunctions-parallel-map-error

---

## Objective

By the end, you will be able to emit structured logs (JSON) from a Lambda with correlation fields (requestId, userId), enable X-Ray active tracing and add custom subsegments, and enable Lambda Insights to see duration, error, and init time metrics per function.

---

## Context

[FACT] Observability in distributed systems relies on the three classic pillars: **logs** (records of discrete events), **metrics** (numerical time series), and **traces** (request tracing between services). In Lambda, each invocation is ephemeral and potentially distributed across hundreds of simultaneous worker instances — which makes correlation between these three pillars especially critical.

[CONSENSUS] The biggest observability problem in Lambda is not lack of data, but lack of correlation. CloudWatch already captures native metrics and logs by default. What differentiates an observable system from a monitored system is the ability to, given a request ID or traceId, quickly find the function log, the complete X-Ray trace, the performance metrics of the specific worker, and the errors that occurred. Structured logging, X-Ray, and Lambda Insights are the three tools that allow building this correlation systematically in Lambda.

[FACT] Starting in 2023, Lambda began supporting natively the JSON format for system logs (messages that the Lambda service itself emits — like `START`, `END`, `REPORT`), in addition to application logs. This simplifies log ingestion in CloudWatch Logs Insights without the need for custom parsers.

---

## Core concepts

### 1. Structured Logging — logs as objects, not strings

[FACT] The default log format in Lambda is plain text. When the application uses `print()` or `console.log()`, CloudWatch receives a text line that needs to be parsed with regex or glob expressions to extract fields. Structured logging replaces strings with JSON objects, making each field directly queryable.

```
Unstructured log (hard to query):
───────────────────────────────────────────────────────────────
[INFO] 2026-06-24T10:15:32Z - Order P001 processed for user U42 in 245ms

Structured JSON log (CloudWatch Insights auto-discovers fields):
───────────────────────────────────────────────────────────────
{
  "timestamp": "2026-06-24T10:15:32.410Z",
  "level": "INFO",
  "message": "Order processed",
  "requestId": "abc123-def456",
  "traceId": "1-66795-abc...",
  "order_id": "P001",
  "user_id": "U42",
  "duration_ms": 245,
  "service": "orders",
  "version": "2.1.0"
}
```

[FACT] CloudWatch Logs Insights automatically detects fields in JSON lines without configuration. Once logs are in JSON, queries like the one below work directly:

```sql
-- Find all errors from a specific user in the last hour
fields @timestamp, level, message, order_id, duration_ms
| filter level = "ERROR" and user_id = "U42"
| sort @timestamp desc
| limit 50
```

#### Required correlation fields

[CONSENSUS] The practice adopted by most production teams is to include at least four correlation fields in each log:

| Field | Source | Usage |
|---|---|---|
| `requestId` | `context.aws_request_id` | Correlate all logs from one invocation |
| `traceId` | `os.environ["_X_AMZN_TRACE_ID"]` | Correlate with X-Ray trace |
| `service` | Constant in the function | Filter logs by service in aggregated log groups |
| `cold_start` | Initialization variable | Identify invocations with Init phase |

#### Enabling native JSON in Lambda (function log format)

[FACT] Since 2023, it's possible to configure the Lambda function so that **system** messages (START, END, REPORT) are also emitted in JSON. This is separate from the application log format:

```python
# CDK — native log format and log level for the function
from aws_cdk import aws_lambda as lambda_

fn = lambda_.Function(
    self, "MyFunction",
    # ...
    logging_format=lambda_.LoggingFormat.JSON,   # system logs in JSON
    system_log_level=lambda_.SystemLogLevel.INFO,
    application_log_level=lambda_.ApplicationLogLevel.INFO,
    log_retention=logs.RetentionDays.ONE_WEEK,
)
```

[FACT] With `LoggingFormat.JSON`, the `REPORT` record becomes:
```json
{
  "timestamp": "2026-06-24T10:15:32.660Z",
  "type": "platform.report",
  "record": {
    "requestId": "abc123",
    "metrics": {
      "durationMs": 245.12,
      "billedDurationMs": 246,
      "memorySizeMB": 256,
      "maxMemoryUsedMB": 89,
      "initDurationMs": 312.5
    }
  }
}
```

#### Structured logging with pure Python (without Powertools)

```python
import json
import logging
import os

# Configures root logger to emit JSON
class JsonFormatter(logging.Formatter):
    def format(self, record):
        log_entry = {
            "timestamp": self.formatTime(record),
            "level": record.levelname,
            "message": record.getMessage(),
            "logger": record.name,
            "requestId": getattr(record, "requestId", None),
            "traceId": os.environ.get("_X_AMZN_TRACE_ID"),
            "service": "orders",
        }
        # Extra fields passed via extra={}
        for key in vars(record):
            if key not in logging.LogRecord.__dict__ and not key.startswith("_"):
                log_entry[key] = getattr(record, key)
        return json.dumps(log_entry)

logger = logging.getLogger()
logger.setLevel(logging.INFO)
if logger.handlers:
    logger.handlers[0].setFormatter(JsonFormatter())

# Variable to detect cold start
COLD_START = True

def handler(event, context):
    global COLD_START
    cold = COLD_START
    COLD_START = False

    # Enriches all logs from this invocation with requestId
    extra = {"requestId": context.aws_request_id, "cold_start": cold}

    logger.info("Invocation started", extra={**extra, "event_type": event.get("type")})

    try:
        result = process(event, extra)
        logger.info("Invocation completed", extra={**extra, "result": result["status"]})
        return result
    except Exception as e:
        logger.error("Invocation error", extra={**extra, "error_type": type(e).__name__, "error_msg": str(e)})
        raise
```

[CONSENSUS] Using Lambda Powertools Logger (session-021) eliminates the need to implement this boilerplate manually. The `@logger.inject_lambda_context` decorator automatically injects `requestId`, `cold_start`, `xray_trace_id`, and other fields into all invocation logs.

---

### 2. X-Ray Active Tracing — distributed tracing in Lambda

[FACT] AWS X-Ray is the AWS distributed tracing service. In Lambda, tracing works via an X-Ray daemon that runs inside the execution environment and receives data via UDP (port 2000 on loopback). The X-Ray SDK sends segments to this daemon, which forwards them to the X-Ray service.

#### Anatomy of a trace in Lambda

[FACT] With active tracing enabled, Lambda automatically creates two segments per invocation:

```
Trace (X-Amzn-Trace-Id: Root=1-...;Sampled=1)
├── Segment 1: "Lambda" (service)
│   └── Represents the Lambda service receiving the invocation
│       Includes: cold start time, queuing time
│
└── Segment 2: "myFunction" (function)
    ├── Subsegment: Initialization (only on cold starts)
    │   └── Init phase time (module loading)
    ├── Subsegment: Invocation
    │   └── Handler execution time
    │       ├── [your custom subsegments here]
    └── Subsegment: Overhead
        └── Checkpoint/extensions time
```

[FACT] The environment variable `_X_AMZN_TRACE_ID` contains the current invocation's trace ID in the format `Root=1-<timestamp>-<hex>;Parent=<parentId>;Sampled=<0|1>`. This string must be propagated in downstream calls (HTTP headers, SQS messages, etc.) to maintain trace continuity.

#### Enabling active tracing

```python
# CDK
fn = lambda_.Function(
    self, "MyFunction",
    # ...
    tracing=lambda_.Tracing.ACTIVE,   # or PASS_THROUGH to inherit from upstream
)
# CDK automatically adds xray:PutTraceSegments and xray:PutTelemetryRecords
# to the function's execution role
```

```bash
# CLI
aws lambda update-function-configuration \
  --function-name MyFunction \
  --tracing-config Mode=Active
```

#### Custom subsegments

[FACT] The X-Ray SDK allows creating subsegments for any operation within the handler — database calls, external APIs, heavy processing:

```python
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.core import patch_all

# Automatically patches boto3, requests, httplib, pymongo, etc.
patch_all()

def handler(event, context):
    order_id = event["order_id"]

    # Manual subsegment with context manager
    with xray_recorder.in_subsegment("validate-order") as subseg:
        subseg.put_annotation("order_id", order_id)   # indexed — filterable
        subseg.put_annotation("amount", event["amount"])
        subseg.put_metadata("full_event", event)       # not indexed — only stored
        result = validate_order(order_id)

    # Decorator on internal functions
    result_db = save_to_database(order_id, result)

    return {"status": "ok"}

@xray_recorder.capture("save-to-database")
def save_to_database(order_id, data):
    # boto3 is already patched — DynamoDB calls appear as
    # automatic subsegments inside "save-to-database"
    table.put_item(Item={"id": order_id, **data})
    return True
```

#### Annotations vs Metadata

[FACT] The distinction between `put_annotation` and `put_metadata` is critical for X-Ray usage:

```
Annotations                          Metadata
────────────────────────────────     ────────────────────────────────
Types: string, number, boolean       Types: any JSON serializable
Indexed by X-Ray                     NOT indexed
Appear in filter expressions         Only visible in trace detail
Limit: 50 annotations per trace      Limit: 64KB per segment
Usage: grouping, filters, alerts     Usage: debug, context data
```

[FACT] Filter expressions in the X-Ray console use annotations:
```
# Find all traces with error for a specific order
annotation.order_id = "P001" AND error = true

# Slow traces (>2s) from a specific service
annotation.service = "orders" AND responsetime > 2
```

#### Sampling rules

[FACT] By default, X-Ray samples 5% of requests (or 1 req/s, whichever is greater). In high-volume production, this is essential for cost control. Custom rules can be configured:

```bash
aws xray create-sampling-rule --cli-input-json '{
  "SamplingRule": {
    "RuleName": "HighValueOrders",
    "Priority": 1,
    "FixedRate": 1.0,
    "ReservoirSize": 5,
    "ServiceName": "orders",
    "ServiceType": "AWS::Lambda::Function",
    "Host": "*",
    "HTTPMethod": "*",
    "URLPath": "*",
    "ResourceARN": "*",
    "Attributes": { "order_value": "high" }
  }
}'
```

---

### 3. Lambda Insights — system metrics per invocation

[FACT] Lambda Insights is implemented as a **Lambda Extension** (internal), distributed as an AWS-managed Lambda Layer. When enabled, the extension collects system metrics from each invocation and sends them to CloudWatch Logs in the `/aws/lambda/insights` group using the EMF (Embedded Metric Format) format, which CloudWatch interprets to create time series metrics.

#### Collected metrics

[FACT] Lambda Insights collects the following metrics per invocation:

```
Performance metrics:
┌─────────────────────────┬──────────────────────────────────────────────────┐
│ Metric                  │ Description                                      │
├─────────────────────────┼──────────────────────────────────────────────────┤
│ duration                │ Invocation duration in ms                        │
│ billed_duration         │ Billed duration (rounded to 1ms)                 │
│ init_duration           │ Init phase time (cold start only)                │
│ memory_utilization      │ % of configured memory utilized                  │
│ used_memory_max         │ Peak memory usage in MB                          │
│ cpu_total_time          │ Total CPU time in ms                             │
├─────────────────────────┼──────────────────────────────────────────────────┤
│ I/O metrics:            │                                                  │
│ rx_bytes                │ Bytes received via network                       │
│ tx_bytes                │ Bytes sent via network                           │
│ disk_used               │ /tmp usage in MB                                 │
│ disk_total              │ Total /tmp space in MB                           │
├─────────────────────────┼──────────────────────────────────────────────────┤
│ Diagnostic:             │                                                  │
│ cold_start              │ 1 if cold start, 0 otherwise                    │
│ out_of_memory           │ 1 if function exceeded memory                   │
│ timeout                 │ 1 if function hit timeout                        │
│ errors                  │ 1 if unhandled error occurred                    │
└─────────────────────────┴──────────────────────────────────────────────────┘
```

#### Enabling Lambda Insights via CDK

```python
# CDK
fn = lambda_.Function(
    self, "MyFunction",
    # ...
    tracing=lambda_.Tracing.ACTIVE,
    insights_version=lambda_.LambdaInsightsVersion.VERSION_1_0_229_0,
    # CDK automatically adds:
    # - The managed layer arn:aws:lambda:<region>:580247275435:layer:LambdaInsightsExtension:...
    # - The CloudWatchLambdaInsightsExecutionRolePolicy to the execution role
)
```

[FACT] The layer ARN changes per region. `LambdaInsightsVersion.VERSION_1_0_229_0` is the most recent version as of May 2026 — check [docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Lambda-Insights.html](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Lambda-Insights.html) for available versions in your region.

#### Dashboard and Log Insights

[FACT] The CloudWatch console automatically creates a dashboard at `/LambdaInsights` when Lambda Insights is enabled. Metrics become available in the `LambdaInsights` namespace.

```sql
-- CloudWatch Logs Insights: slowest invocations with cold start
-- Log group: /aws/lambda/insights
fields @timestamp, function_name, duration, init_duration, memory_utilization, cold_start
| filter cold_start = 1
| sort duration desc
| limit 20

-- Correlate application log with Insights metrics
-- Log group: /aws/lambda/MyFunction
fields @timestamp, @message, @requestId
| filter level = "ERROR"
| join insights on requestId = @requestId   -- correlation via requestId
```

---

### 4. Correlation between the three pillars

[FACT] The field that unites logs, traces, and metrics in Lambda is the `requestId` (also called `aws_request_id` in Python's context object). The correlation flow is:

```
Invocation received
       │
       ▼
Lambda Service generates requestId ────────────────────────────────────┐
       │                                                                │
       ▼                                                                ▼
┌──────────────────────┐    ┌────────────────────────┐    ┌────────────────────────┐
│    LOGS              │    │      X-RAY             │    │  LAMBDA INSIGHTS       │
│                      │    │                        │    │                        │
│ Structured log with  │    │ Trace ID generated by  │    │ EMF metrics emitted    │
│ "requestId": "abc"   │    │ X-Ray daemon           │    │ with requestId and     │
│ "traceId": "1-..."   │    │                        │    │ function_name          │
│ "cold_start": true   │    │ Function segment has   │    │                        │
│ "user_id": "U42"     │    │ requestId annotation   │    │ init_duration: 312ms   │
│                      │    │                        │    │ memory_utilization: 35% │
└──────────┬───────────┘    └───────────┬────────────┘    └────────────┬───────────┘
           │                            │                              │
           └────────────────────────────┴──────────────────────────────┘
                         requestId as correlation key

CloudWatch console → ServiceLens: unites logs + traces in a single view
```

[FACT] **CloudWatch ServiceLens** (tab in the CloudWatch console) automatically consumes the correlation between logs and X-Ray traces when:
1. The function has active tracing enabled.
2. The logs include the `@xrayTraceId` field (Lambda Powertools injects this automatically; without Powertools, use `os.environ["_X_AMZN_TRACE_ID"]`).

---

## Practical example

**Scenario:** Order processing function with complete structured logging, X-Ray with custom subsegments, and Lambda Insights.

### Python handler with all three pillars

```python
import json
import logging
import os
import time
import boto3
from aws_xray_sdk.core import xray_recorder, patch_all

# Patches boto3 clients automatically for X-Ray
patch_all()

# ── Structured Logger ──────────────────────────────────────────────────────────
class StructuredLogger:
    def __init__(self, service_name: str, level: str = "INFO"):
        self.service = service_name
        self.level = getattr(logging, level)
        self._base_fields: dict = {}

    def set_invocation_context(self, request_id: str, cold_start: bool):
        self._base_fields = {
            "requestId": request_id,
            "cold_start": cold_start,
            "traceId": os.environ.get("_X_AMZN_TRACE_ID", ""),
            "service": self.service,
        }

    def _emit(self, level: str, message: str, **kwargs):
        entry = {
            "timestamp": time.strftime("%Y-%m-%dT%H:%M:%S.000Z", time.gmtime()),
            "level": level,
            "message": message,
            **self._base_fields,
            **kwargs,
        }
        # Lambda captures stdout; using print ensures immediate flush
        print(json.dumps(entry))

    def info(self, msg: str, **kwargs):  self._emit("INFO", msg, **kwargs)
    def warn(self, msg: str, **kwargs):  self._emit("WARN", msg, **kwargs)
    def error(self, msg: str, **kwargs): self._emit("ERROR", msg, **kwargs)
    def debug(self, msg: str, **kwargs):
        if self.level <= logging.DEBUG:
            self._emit("DEBUG", msg, **kwargs)

logger = StructuredLogger("orders")

# ── Clients (initialized outside handler = reused in warm starts) ──────────────
dynamodb = boto3.resource("dynamodb")
table = dynamodb.Table(os.environ["ORDERS_TABLE"])

# Detects cold start
_COLD_START = True

# ── Handler ───────────────────────────────────────────────────────────────────
def handler(event, context):
    global _COLD_START
    cold = _COLD_START
    _COLD_START = False

    logger.set_invocation_context(context.aws_request_id, cold)
    logger.info("Invocation started", order_id=event.get("order_id"))

    start = time.time()

    try:
        result = process_order(event)
        duration_ms = int((time.time() - start) * 1000)
        logger.info(
            "Order processed",
            order_id=event["order_id"],
            status=result["status"],
            duration_ms=duration_ms,
        )
        return result

    except ValueError as e:
        logger.error(
            "Validation error",
            order_id=event.get("order_id"),
            error_type="ValueError",
            error_msg=str(e),
        )
        raise
    except Exception as e:
        logger.error(
            "Unexpected error",
            order_id=event.get("order_id"),
            error_type=type(e).__name__,
            error_msg=str(e),
        )
        raise


def process_order(event: dict) -> dict:
    order_id = event["order_id"]

    # ── Subsegment: validation ─────────────────────────────────────────────────
    with xray_recorder.in_subsegment("validate-order") as seg:
        seg.put_annotation("order_id", order_id)
        seg.put_annotation("amount", event.get("amount", 0))
        seg.put_metadata("full_event", event, namespace="orders")

        if not order_id or not isinstance(event.get("amount"), (int, float)):
            raise ValueError(f"Invalid order: required fields missing")

        if event["amount"] <= 0:
            raise ValueError(f"Order amount must be positive: {event['amount']}")

    # ── Subsegment: persistence ────────────────────────────────────────────────
    with xray_recorder.in_subsegment("persist-order") as seg:
        seg.put_annotation("order_id", order_id)
        # patched boto3 → DynamoDB call appears as sub-subsegment
        table.put_item(Item={
            "order_id": order_id,
            "amount": str(event["amount"]),
            "status": "PROCESSED",
            "request_id": xray_recorder.current_segment().id,
        })

    return {"status": "PROCESSED", "order_id": order_id}
```

### CDK — function with all three pillars enabled

```python
from aws_cdk import (
    Stack, Duration, RemovalPolicy,
    aws_lambda as lambda_,
    aws_logs as logs,
    aws_iam as iam,
)

class OrdersObservabilityStack(Stack):

    def __init__(self, scope, construct_id, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        # Layer with aws-xray-sdk (built via Docker for Linux compatibility)
        xray_layer = lambda_.LayerVersion(
            self, "XRayLayer",
            code=lambda_.Code.from_asset(
                "layers/xray",
                bundling={
                    "image": lambda_.Runtime.PYTHON_3_12.bundling_image,
                    "command": [
                        "bash", "-c",
                        "pip install aws-xray-sdk -t /asset-output/python"
                    ],
                }
            ),
            compatible_runtimes=[lambda_.Runtime.PYTHON_3_12],
            description="aws-xray-sdk for custom instrumentation",
        )

        fn = lambda_.Function(
            self, "ProcessOrder",
            runtime=lambda_.Runtime.PYTHON_3_12,
            handler="handler.handler",
            code=lambda_.Code.from_asset("src/orders"),
            memory_size=256,
            timeout=Duration.seconds(30),
            layers=[xray_layer],
            environment={
                "ORDERS_TABLE": "orders",
                "POWERTOOLS_SERVICE_NAME": "orders",
            },
            # Pillar 1: Native structured logging
            logging_format=lambda_.LoggingFormat.JSON,
            system_log_level=lambda_.SystemLogLevel.INFO,
            application_log_level=lambda_.ApplicationLogLevel.INFO,
            log_retention=logs.RetentionDays.ONE_WEEK,
            # Pillar 2: X-Ray active tracing
            tracing=lambda_.Tracing.ACTIVE,
            # Pillar 3: Lambda Insights
            insights_version=lambda_.LambdaInsightsVersion.VERSION_1_0_229_0,
        )

        # Additional permission for DynamoDB (X-Ray already added by CDK)
        fn.add_to_role_policy(iam.PolicyStatement(
            actions=["dynamodb:PutItem", "dynamodb:GetItem"],
            resources=["arn:aws:dynamodb:*:*:table/orders"],
        ))
```

### CloudWatch Logs Insights queries for diagnostics

```sql
-- 1. Errors from the last 3 hours grouped by type
fields @timestamp, message, error_type, order_id
| filter level = "ERROR"
| stats count(*) as total by error_type
| sort total desc

-- 2. p95 and p99 latency per hour (using duration_ms field from log)
fields @timestamp, duration_ms
| filter ispresent(duration_ms)
| stats
    pct(duration_ms, 95) as p95,
    pct(duration_ms, 99) as p99,
    avg(duration_ms) as average
  by bin(1h)

-- 3. Cold starts and their requestIds (to cross-reference with X-Ray)
fields @timestamp, requestId, cold_start, message
| filter cold_start = true
| sort @timestamp desc
| limit 100

-- 4. In log group /aws/lambda/insights — functions with memory > 80%
fields @timestamp, function_name, memory_utilization, duration, cold_start
| filter memory_utilization > 80
| sort memory_utilization desc
| limit 50
```

---

## Common pitfalls

### Pitfall 1 — `print()` with JSON object is not the same as true structured logging

**The mistake:** The developer does `print(json.dumps({"level": "INFO", "message": "ok"}))` and assumes CloudWatch Logs Insights will parse it as JSON. It works — but the timestamp generated by Lambda for the log line isn't inside the JSON, making sorting difficult. Additionally, if the JSON object contains line breaks, CloudWatch may interpret it as multiple log events.

**Why it happens:** CloudWatch Logs captures each line (`\n`) as a separate event. If `json.dumps` doesn't have `separators=(',', ':')` and produces multi-line JSON, the event is fragmented.

**How to avoid:**
- Always use `json.dumps(obj, separators=(',', ':'))` (no spaces) to ensure the JSON is a single line.
- Or use `json.dumps(obj)` without `indent` (which is the default — without indent produces one line).
- For timestamps, rely on the `@timestamp` field that CloudWatch adds automatically — it's not necessary to include a timestamp in the JSON (but including it doesn't hurt and facilitates correlation).

---

### Pitfall 2 — `patch_all()` from X-Ray SDK outside the handler causes errors in test environment

**The mistake:** The developer calls `patch_all()` at the module's global scope. In unit tests without the X-Ray daemon running, the SDK tries to register the trace and fails with `SegmentNotFoundException: cannot find the current segment/subsegment`.

**Why it happens:** `patch_all()` monkeypatches boto3 clients globally. In tests, there's no active X-Ray context — the daemon isn't running and there's no open segment.

**How to avoid:**
- Configure the SDK to ignore errors when there's no context: `xray_recorder.configure(context_missing='LOG_ERROR')` (default in Lambda) or `'IGNORE_ERROR'`.
- In tests, configure via environment variable: `AWS_XRAY_CONTEXT_MISSING=LOG_ERROR`.
- CDK/Lambda already configures this automatically when tracing is enabled, but local test environments may not have this variable.

```python
from aws_xray_sdk.core import xray_recorder, patch_all
xray_recorder.configure(context_missing='IGNORE_ERROR')
patch_all()
```

---

### Pitfall 3 — Lambda Insights without `CloudWatchLambdaInsightsExecutionRolePolicy` makes the extension fail silently

**The mistake:** Lambda Insights is enabled (layer added), but metrics don't appear in `/aws/lambda/insights`. The function runs normally, but no data arrives.

**Why it happens:** The Lambda Insights extension needs permission to write logs to the `/aws/lambda/insights` log group with specific permissions: `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents`. Without these permissions, the extension fails to initialize and is ignored — it doesn't throw an error on the main invocation.

**How to avoid:**
- With CDK: `insights_version=lambda_.LambdaInsightsVersion.*` adds the managed policy automatically.
- Manually: add `CloudWatchLambdaInsightsExecutionRolePolicy` (AWS managed) to the function's execution role.
- To verify: check the extension logs in the `/aws/lambda/insights` log group or enable `LAMBDA_INSIGHTS_LOG_LEVEL=info` in environment variables.

```python
# CDK does this automatically, but if you need to do it manually:
fn.role.add_managed_policy(
    iam.ManagedPolicy.from_aws_managed_policy_name(
        "CloudWatchLambdaInsightsExecutionRolePolicy"
    )
)
```

---

## Reflection exercise

You have a Lambda function that processes payments and is receiving user complaints that "some payments don't process". The system has no observability configured beyond standard Lambda logs (plain text, no correlation). You need to propose an observability solution that allows, given a transaction ID reported by the user, to find in less than 5 minutes: (a) the complete log of the invocation that processed that transaction, (b) whether there was a retry or cold start, (c) which external calls (database, payment API) were made and which was slowest, and (d) whether the problem is systemic (affects X% of transactions) or isolated.

**Question:** Which fields would you include in structured logs? How would you configure X-Ray to trace the payment API call (which is external HTTP, not AWS)? What CloudWatch Logs Insights query would you use to identify if the problem is systemic? Where would Lambda Insights help (or not help) in this diagnosis?

---

## Resources for further study

1. **Monitor function performance with Amazon CloudWatch Lambda Insights**
   URL: https://docs.aws.amazon.com/lambda/latest/dg/monitoring-insights.html
   Official guide for enabling Lambda Insights with layer ARNs per region, step-by-step via console/CDK/CLI, and complete list of collected metrics. Includes how to interpret the `/LambdaInsights` dashboard.

2. **Visualize Lambda function invocations using AWS X-Ray**
   URL: https://docs.aws.amazon.com/lambda/latest/dg/services-xray.html
   Explains the native X-Ray integration with Lambda: how segments are created automatically, how to enable active tracing, and how to use the X-Ray Python SDK within Lambda functions. Includes subsegment examples and sampling configuration.

3. **Configuring JSON and plain text log formats**
   URL: https://docs.aws.amazon.com/lambda/latest/dg/monitoring-cloudwatchlogs-logformat.html
   Documentation of the new native JSON format for system logs (START, END, REPORT). Describes fields emitted in each event type, how to configure via console/CLI/CDK, and how to use with CloudWatch Logs Insights.

---
