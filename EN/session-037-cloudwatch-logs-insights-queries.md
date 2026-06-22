# Session 037 — CloudWatch Logs Insights: Query Syntax and Derived Fields

**Estimated duration:** 60 minutes  
**Dependencies:** session-036-cloudwatch-custom-metrics-emf

---

## Objective

By the end of this session, you will be able to write Logs Insights queries to aggregate errors by endpoint, extract fields from structured JSON logs with `parse`, create timeseries visualizations, use automatically discovered fields (`@timestamp`, `@message`, `@requestId`), and save reusable queries in the console.

---

## Context

[FACT] CloudWatch Logs Insights is an interactive log analysis service that accepts its own pipe-oriented query language. You can query multiple log groups simultaneously, visualize results as timeseries or tables, and save queries for reuse. Results are generated in seconds to minutes depending on volume.

[FACT] CloudWatch Logs Insights charges per GB of data scanned (us-east-1: $0.005/GB). The more precise the time filter and log group selection, the lower the cost. Queries have no request cost — only scanning cost.

---

## Core Concepts

### 1. Automatically Discovered Fields

[FACT] Logs Insights automatically discovers and indexes certain fields:

```
Field           Source / Content
────────────────────────────────────────────────────────────────
@timestamp      Log event timestamp (always available)
@message        Full text of the log event
@logStream      Log stream name
@log            Log group identifier (account-id:log-group-name)
@requestId      Present in Lambda REPORT and START/END logs
@duration       Lambda invocation duration (in ms) in REPORT events
@billedDuration Lambda billed duration (ms)
@memorySize     Configured memory (bytes)
@maxMemoryUsed  Maximum memory used (bytes)
@initDuration   Cold start duration (ms) — only in cold starts
@type           Lambda event type: "START", "END", "REPORT"
```

[FACT] For logs in **JSON format**, Logs Insights automatically discovers first-level fields as searchable fields. A log `{"level": "ERROR", "endpoint": "/pay", "latency": 250}` has `level`, `endpoint` and `latency` available directly in queries without needing `parse`.

---

### 2. Main Language Commands

[FACT] The language is pipe-based: each command receives the results of the previous one. Commands are case-insensitive but field names are case-sensitive.

```
Command         Purpose
────────────────────────────────────────────────────────────────
fields          Select fields to display; create derived fields
filter          Filter events (WHERE equivalent)
stats           Aggregate data — count, avg, sum, min, max, percentile
sort            Sort results (must come after the last stats)
limit           Limit number of returned rows
parse           Extract subfields from a text field (glob, regex, logfmt)
dedup           Remove duplicates by field(s)
display         Format display without affecting filters (alias of fields post-stats)
pattern         Group similar messages (ML-based clustering)
diff            Compare metrics with previous period
```

---

### 3. `fields` — Projection and Derived Fields

[FACT] The `fields` command selects which fields to display and can create new fields with expressions:

```sql
-- Simple selection
fields @timestamp, @message, level, endpoint

-- Derived fields with arithmetic operators
fields @timestamp, @duration / 1000 as durationSeconds

-- Conditional derived fields with if()
fields @timestamp,
       if(statusCode >= 400, "error", "ok") as requestStatus

-- coalesce: first non-null
fields @timestamp,
       coalesce(httpMethod, method, "UNKNOWN") as verb
```

---

### 4. `filter` — Event Filtering

[FACT] Supports comparison operators (`=`, `!=`, `<`, `<=`, `>`, `>=`), `like` (substring), `not like`, `in [...]`, `ispresent()`, `not ispresent()`:

```sql
-- Simple filter
filter level = "ERROR"

-- Like (substring, case-sensitive)
filter @message like "TimeoutException"

-- Regex with ~
filter @message =~ /TimeoutException|ConnectionReset/

-- Multiple conditions
filter statusCode >= 400 and endpoint like "/api/payments"

-- Check field existence
filter ispresent(errorCode)

-- IN for list of values
filter statusCode in [400, 401, 403, 404]

-- Combining with NOT
filter not (statusCode = 200 or statusCode = 201)
```

---

### 5. `stats` — Aggregations

[FACT] Functions available in `stats`:

```
Function            Description
────────────────────────────────────────────────────────────────
count(*)            Total events in the group
count(field)        Total events where field is not null
count_distinct(f)   Count of unique values
sum(field)          Sum
avg(field)          Arithmetic mean
min(field)          Minimum
max(field)          Maximum
pct(field, n)       Percentile (e.g.: pct(@duration, 95) = p95)
stddev(field)       Standard deviation
earliest(field)     Oldest value in the group
latest(field)       Most recent value in the group
```

[FACT] `bin(period)` groups by time window — fundamental for timeseries:

```sql
-- Errors by endpoint per minute
filter level = "ERROR"
| stats count(*) as errorCount by endpoint, bin(1m)
| sort bin desc

-- Valid periods: 1s, 10s, 30s, 1m, 5m, 10m, 15m, 30m, 1h, 6h, 1d
```

---

### 6. `parse` — Extracting Fields from Unstructured Text

[FACT] Three modes of `parse`:

**Glob (wildcard `*`):**
```sql
-- Log: "user=alice, method:GET, latency := 45"
parse @message "user=*, method:*, latency := *"
    as user, method, latency
| stats avg(latency) by method
```

**Regex with named groups:**
```sql
-- Log: "[ERROR] POST /api/pay - 503 - 1250ms"
parse @message /\[(?<logLevel>[A-Z]+)\] (?<httpMethod>[A-Z]+) (?<path>[^ ]+) - (?<statusCode>\d+) - (?<durationMs>\d+)ms/
| filter logLevel = "ERROR"
| stats count(*) by path, statusCode
```

**For structured JSON logs — parse is NOT needed:**
```sql
-- JSON log: {"level":"ERROR","endpoint":"/pay","latencyMs":1250,"userId":"u-123"}
-- Logs Insights already discovers the fields automatically

fields @timestamp, endpoint, latencyMs, level
| filter level = "ERROR"
| stats avg(latencyMs) as avgLatency by endpoint
```

---

## Practical Example

### Scenario: Payment API Log Analysis

Logs are emitted in structured JSON by Lambda Powertools Logger:

```json
{
  "@timestamp": "2024-01-15T10:23:45.123Z",
  "level": "INFO",
  "service": "payment-processor",
  "message": "Payment processed",
  "endpoint": "/api/v1/payments",
  "httpMethod": "POST",
  "statusCode": 200,
  "latencyMs": 45,
  "userId": "u-abc123",
  "orderId": "ord-xyz789",
  "cold_start": false,
  "xray_trace_id": "1-abc123-def456"
}
```

#### Query 1: Error Rate by Endpoint in the Last 60 Minutes

```sql
fields @timestamp, endpoint, statusCode, level
| filter level in ["ERROR", "CRITICAL"] or statusCode >= 400
| stats
    count(*) as totalErrors,
    count_distinct(userId) as uniqueUsersAffected,
    max(latencyMs) as maxLatency
  by endpoint, statusCode
| sort totalErrors desc
| limit 20
```

#### Query 2: Latency Percentiles by Endpoint (Timeseries, 5 min)

```sql
filter ispresent(latencyMs) and ispresent(endpoint)
| stats
    avg(latencyMs) as avgLatency,
    pct(latencyMs, 50) as p50,
    pct(latencyMs, 95) as p95,
    pct(latencyMs, 99) as p99,
    count(*) as requestCount
  by endpoint, bin(5m)
| sort bin desc
```

#### Query 3: Lambda Cold Starts — Identify and Measure Impact

```sql
-- Uses Lambda REPORT special fields
filter @type = "REPORT"
| stats
    count(*) as totalInvocations,
    sum(if(ispresent(@initDuration), 1, 0)) as coldStartCount,
    avg(@initDuration) as avgColdStartMs,
    max(@initDuration) as maxColdStartMs,
    avg(@duration) as avgDurationMs,
    avg(@maxMemoryUsed / 1000 / 1000) as avgMemoryMB,
    max(@memorySize / 1000 / 1000) - max(@maxMemoryUsed / 1000 / 1000)
        as overProvisionedMB
  by bin(1h)
| sort bin desc
```

#### Query 4: Trace a User's Journey by orderId

```sql
fields @timestamp, level, message, endpoint, statusCode, latencyMs, orderId
| filter orderId = "ord-xyz789"
| sort @timestamp asc
```

#### Query 5: Top 10 Errors Grouped by Message

```sql
filter level = "ERROR"
| stats count(*) as occurrences by message
| sort occurrences desc
| limit 10
```

#### Query 6: `parse` on Unstructured Logs — Lambda REPORT Latency

```sql
-- Lambda REPORT logs: "REPORT RequestId: abc Duration: 123.45 ms Billed Duration: 200 ms ..."
filter @type = "REPORT"
| parse @message "Duration: * ms" as rawDuration
| stats
    avg(rawDuration) as avgDuration,
    pct(rawDuration, 99) as p99Duration,
    max(rawDuration) as maxDuration
  by bin(5m)
| sort bin desc
```

#### Query 7: `dedup` — Last Error Occurrence per User

```sql
fields @timestamp, userId, message, endpoint, statusCode
| filter level = "ERROR" and ispresent(userId)
| sort @timestamp desc
| dedup userId
| limit 50
```

#### Query 8: Slow Requests with Retry Deduplication

```sql
fields @timestamp, @requestId, endpoint, latencyMs, statusCode, userId
| filter latencyMs > 1000
| sort @timestamp desc
| dedup @requestId
| limit 20
```

#### CLI — Execute Queries via AWS CLI

```bash
# 1. Start asynchronous query
QUERY_ID=$(aws logs start-query \
  --log-group-names "/aws/lambda/payment-processor" \
  --start-time "$(date -u -d '1 hour ago' +%s 2>/dev/null || date -u -v-1H +%s)" \
  --end-time "$(date -u +%s)" \
  --query-string '
    filter level = "ERROR" or statusCode >= 400
    | stats count(*) as errorCount by endpoint, statusCode
    | sort errorCount desc
    | limit 10
  ' \
  --query 'queryId' \
  --output text)

echo "Query ID: $QUERY_ID"

# 2. Wait for completion and get results
sleep 5
aws logs get-query-results \
  --query-id "$QUERY_ID" \
  --query '{Status: status, Results: results}'

# 3. Query multiple log groups simultaneously
QUERY_ID2=$(aws logs start-query \
  --log-group-names \
      "/aws/lambda/payment-processor" \
      "/aws/lambda/order-service" \
      "/aws/lambda/notification-service" \
  --start-time "$(date -u -d '1 hour ago' +%s 2>/dev/null || date -u -v-1H +%s)" \
  --end-time "$(date -u +%s)" \
  --query-string '
    filter level = "ERROR"
    | stats count(*) as errors by @log, bin(5m)
    | sort bin desc
  ' \
  --query 'queryId' \
  --output text)

# 4. List saved queries in the console
aws logs describe-query-definitions \
  --query 'queryDefinitions[*].{Name:name,QueryString:queryString}'

# 5. Save a reusable query
aws logs put-query-definition \
  --name "payment-error-rate-by-endpoint" \
  --log-group-names "/aws/lambda/payment-processor" \
  --query-string '
    filter level = "ERROR" or statusCode >= 400
    | stats count(*) as errors, count(*) / 1 as errorRate by endpoint, bin(5m)
    | sort bin desc
  '

# 6. Check approximate scanning cost
# CloudWatch Logs → Insights → Query history in the console
# Or via CloudWatch Metrics:
aws cloudwatch get-metric-statistics \
  --namespace AWS/Logs \
  --metric-name DataScannedInBytes \
  --start-time "$(date -u -d '1 day ago' '+%Y-%m-%dT%H:%M:%SZ' 2>/dev/null || date -u -v-1d '+%Y-%m-%dT%H:%M:%SZ')" \
  --end-time "$(date -u '+%Y-%m-%dT%H:%M:%SZ')" \
  --period 86400 \
  --statistics Sum \
  --query 'Datapoints[0].Sum'
```

#### Dashboard CDK — Widget with Logs Insights Query

```python
from aws_cdk import (
    Stack, Duration,
    aws_cloudwatch as cw,
)
from constructs import Construct


class LogsInsightsDashboardStack(Stack):
    def __init__(self, scope: Construct, construct_id: str, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        dashboard = cw.Dashboard(
            self, "ApiDashboard",
            dashboard_name="api-insights",
        )

        # Logs Insights widget — timeseries of errors by endpoint
        dashboard.add_widgets(
            cw.LogQueryWidget(
                title="Error Rate by Endpoint (5m)",
                log_group_names=["/aws/lambda/payment-processor"],
                view=cw.LogQueryVisualizationType.LINE,
                width=24,
                height=6,
                query_lines=[
                    "filter level = 'ERROR' or statusCode >= 400",
                    "| stats count(*) as errors by endpoint, bin(5m)",
                    "| sort bin desc",
                ],
            ),
        )

        dashboard.add_widgets(
            # Table widget — top errors
            cw.LogQueryWidget(
                title="Top Error Messages (last 1h)",
                log_group_names=["/aws/lambda/payment-processor"],
                view=cw.LogQueryVisualizationType.TABLE,
                width=12,
                height=6,
                query_lines=[
                    "filter level = 'ERROR'",
                    "| stats count(*) as occurrences by message",
                    "| sort occurrences desc",
                    "| limit 10",
                ],
            ),
            # Bar widget — p99 latency by endpoint
            cw.LogQueryWidget(
                title="p99 Latency by Endpoint",
                log_group_names=["/aws/lambda/payment-processor"],
                view=cw.LogQueryVisualizationType.BAR,
                width=12,
                height=6,
                query_lines=[
                    "filter ispresent(latencyMs)",
                    "| stats pct(latencyMs, 99) as p99 by endpoint",
                    "| sort p99 desc",
                    "| limit 10",
                ],
            ),
        )
```

---

## Common Pitfalls

**1. Cost per GB scanned — queries on large log groups**  
[FACT] Logs Insights charges $0.005/GB scanned (us-east-1). A 100 GB log group with 30-day retention can cost $0.50 per query if the time range is not restricted. Always use the smallest time range that answers your question. For frequent historical analyses, consider exporting logs to S3 and using Athena (cheaper for large volumes).

**2. `parse` on JSON logs is unnecessary and slow**  
[FACT] Logs Insights already automatically indexes first-level fields from JSON logs. Using `parse @message "\"latency\": *" as latency` on a JSON log is redundant and uses more CPU. For JSON logs, use the fields directly.

**3. `sort` and `limit` must come after the last `stats`**  
[FACT] The language requires that `sort` and `limit` appear after the last `stats` command. Placing `sort` before `stats` generates a syntax error or incorrect results.

**4. Fields with special characters require backticks**  
[FACT] Fields containing hyphens, dots, or starting with a number need backticks. Example: a log with `"detail-type"` must be referenced as `` `detail-type` `` in the query. Fields like `requestParameters.userName` (nested) are accessed with dot-notation directly.

**5. Queries on multiple log groups return the `@log` field**  
[FACT] When querying multiple log groups, the `@log` field identifies which log group each event came from — `account-id:log-group-name`. Without filtering by `@log`, you may inadvertently mix events from different services in the same aggregation.

**6. `count_distinct` is approximate for high cardinality**  
[FACT] For fields with many unique values (e.g., `userId`), `count_distinct` uses HyperLogLog internally and may have an estimation error of ~2-3%. For exact counts of high cardinality, export the data to Athena.

---

## Reflection Exercise

You have a log group `/aws/lambda/checkout-service` with structured JSON logs following the Powertools Logger pattern. The relevant fields are: `level`, `endpoint`, `statusCode`, `latencyMs`, `userId`, `cartId`, `errorCode`.

Write queries for:

1. **Availability SLA by endpoint:** Return, for each endpoint, the total requests, total errors (statusCode >= 400 or level = ERROR), and the success percentage — grouped by 1-hour windows over the last 24 hours. Sort by error rate descending.

2. **Affected user diagnosis:** Given a specific `userId` (`u-suspicious-123`), list in chronological order all error events from the last 6 hours, showing `@timestamp`, `endpoint`, `statusCode`, `errorCode`, and `cartId`.

3. **Parse on legacy logs:** The legacy service emits logs in text format: `"[2024-01-15 10:23:45] WARN /checkout/confirm - user=u-abc123 cart=cart-456 error=STOCK_DEPLETED"`. Write a query with `parse` (glob or regex) that extracts `warnUser`, `warnCart` and `warnError`, and aggregates by `warnError`.

---

## Resources for Further Study

- [FACT] CloudWatch Logs Insights query syntax: https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax.html
- [FACT] Sample queries (Lambda, VPC Flow, CloudTrail, API Gateway): https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax-examples.html
- [FACT] Supported logs and discovered fields: https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_AnalyzeLogData-discoverable-fields.html
- [FACT] Functions reference (boolean, datetime, numeric): https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax-operations-functions.html

---
