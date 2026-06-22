# Session 19 — Lambda: execution model, cold starts and provisioned concurrency

**Estimated duration:** 60 minutes
**Prerequisites:** session-004-cdk-v2-setup-bootstrap

---

## Objective

By the end, you will be able to measure the cold start of a function with and without provisioned concurrency, calculate the cost of provisioned concurrency vs on-demand for a specific traffic profile, and identify which languages and package sizes have the greatest impact on cold start.

---

## Context

[FACT] Lambda is a serverless compute service where you pay only for the time your code runs — not for capacity allocated on standby. The corollary of this guarantee is that when there are no active executions, no execution environments are kept warm. The next invocation that arrives when no environment is available must go through the initialization phase before executing the handler. This latency cost is called a **cold start**.

[FACT] Cold starts are not a defect in Lambda — they are a direct consequence of the billing model. The trade-off is: you save by paying zero when your function is not invoked, but the first invocation after a period of inactivity (or any invocation that requires a new execution environment due to horizontal scaling) has additional latency. For most asynchronous workloads, this trade-off is acceptable. For synchronous low-latency APIs with sparse traffic, it can be problematic.

[CONSENSUS] The importance of cold starts is often exaggerated in community discussions. [FACT] According to AWS data, cold starts occur in less than 1% of invocations in most production workloads. The real problem appears in three specific scenarios: APIs with very sparse traffic (one invocation every 15+ minutes), functions with heavy initialization (Spring Boot Java, ML models), and interactive applications where P99 latency matters more than the average.

---

## Core concepts

### 1. The execution environment lifecycle

[FACT] A Lambda execution environment goes through three phases:

```
┌─────────────────────────────────────────────────────────────────┐
│  INIT PHASE (cold start — billed as duration)                  │
│                                                                 │
│  1. Code/layer download (from source: S3, ECR)                 │
│     └─ Often called "cold start" in the strict sense           │
│                                                                 │
│  2. Runtime initialization (Node.js, Python, JVM, etc.)        │
│                                                                 │
│  3. Static initialization code execution                       │
│     └─ Everything OUTSIDE the handler: imports, DB connections,│
│        model loading, SDK initialization                       │
│                                                                 │
│  [signals ready → Next API]                                    │
└─────────────────────────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────────┐
│  INVOKE PHASE (handler execution)                              │
│                                                                 │
│  handler(event, context) { ... }                               │
│                                                                 │
│  Billed by duration × memory                                   │
└─────────────────────────────────────────────────────────────────┘
              │
              │  function returns, environment goes to standby
              ▼
┌─────────────────────────────────────────────────────────────────┐
│  SHUTDOWN PHASE (eventual)                                     │
│                                                                 │
│  Lambda decides to recycle the environment (after inactivity)  │
│  Extensions have up to 2s to finalize                          │
└─────────────────────────────────────────────────────────────────┘
```

[FACT] The INIT phase has a limit of **10 seconds**. If the initialization code (outside the handler) takes more than 10 seconds to complete, Lambda retries on the first invocation using the function's configured timeout. Functions with heavy Spring Boot or multi-GB ML model loading can exceed this limit.

[FACT] What happens between the INVOKE phase of one invocation and the next is important: the execution environment is **frozen** (CPU suspended, memory maintained). Global variables, database connections, and in-memory caches **persist between warm invocations**. This is the mechanism that enables database connection reuse.

```python
# Python: database connection created ONCE during static initialization
# Persists between invocations of the same execution environment
import boto3
import psycopg2

# Code OUTSIDE the handler: executed only on cold start
db_connection = psycopg2.connect(host=os.environ['DB_HOST'], ...)
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table(os.environ['TABLE_NAME'])

def handler(event, context):
    # Reuses db_connection and table (no reconnection overhead)
    result = table.get_item(Key={'id': event['id']})
    return result['Item']
```

---

### 2. Factors influencing cold start duration

[FACT] The main factors, in order of impact:

**Runtime/language:**

```
Typical cold start time (runtime initialization, without app code):

  Python 3.x:      ~100-200ms
  Node.js 20+:     ~100-200ms
  Go (custom):     ~50-100ms   (static binary, no VM)
  .NET 8:          ~300-600ms  (with SnapStart: ~10ms)
  Java 21:         ~500-2000ms (with SnapStart: ~10ms)
  Java Spring Boot: 3000-8000ms (without optimizations)

Note: these values are approximations based on community benchmarks
and vary with allocated memory, package size, and datacenter load.
```

[FACT] **Allocated memory** has an indirect impact on cold start: more memory = more proportional CPU (Lambda allocates CPU proportional to memory). Doubling memory from 512MB to 1024MB can reduce static initialization time by up to 50% for functions with CPU-intensive initialization.

[FACT] **Deployment package size** impacts download/extraction time during cold start. Practical references:

```
Small package (< 5 MB):   minimal impact (< 50ms additional)
Medium package (5-50 MB):  50-200ms additional
Large package (> 50 MB):   200ms+ additional
Container image (> 500 MB): can add 1-3s on first pull

Mitigation: Lambda maintains a code cache for an undisclosed period of time.
Subsequent pulls of the same code are much faster.
```

[FACT] **VPC** was historically the largest cold start multiplier (added 1-3 seconds to provision ENI). [FACT] Since 2019, AWS changed the VPC architecture to pre-provision ENIs. Cold starts in VPC functions are now comparable to non-VPC functions in most cases. Residual impact still exists in accounts with few recently created VPC execution environments.

---

### 3. Provisioned Concurrency

[FACT] Provisioned Concurrency (PC) maintains a configured number of **pre-initialized and warm** execution environments, eliminating cold starts for those environments. When an invocation reaches a PC environment, it enters the INVOKE phase directly without going through INIT.

```
Without PC:
  Invocation 1: [INIT 800ms] + [INVOKE 50ms] = 850ms latency
  Invocation 2: [INVOKE 50ms] = 50ms (warm)

With PC (2 provisioned environments):
  Invocation 1: [INVOKE 50ms] (environment already initialized) = 50ms
  Invocation 2: [INVOKE 50ms] = 50ms
  Invocation 3: [INIT 800ms] + [INVOKE 50ms] = 850ms (PC exhausted → on-demand)
```

[FACT] PC **must be configured on a specific version or alias**, not on `$LATEST`:

```bash
# ❌ Wrong: $LATEST does not support PC
aws lambda put-provisioned-concurrency-config \
  --function-name my-function \
  --qualifier '$LATEST' \
  --provisioned-concurrent-executions 10

# ✅ Correct: numbered version
aws lambda put-provisioned-concurrency-config \
  --function-name my-function \
  --qualifier 3           # version 3
  --provisioned-concurrent-executions 10

# ✅ Correct: alias
aws lambda put-provisioned-concurrency-config \
  --function-name my-function \
  --qualifier prod        # alias 'prod'
  --provisioned-concurrent-executions 10
```

[FACT] **Provisioned Concurrency cost:**

```
PC billing (us-east-1, reference):
  $0.0000646234 per GB-second of allocated PC
  $0.0000097656 per GB-second of invocation on PC

On-demand billing (for comparison):
  $0.0000200000 per GB-second of invocation

Example: 10 PC × 1GB × 24h × 30 days
  PC allocated:    10 × 1 × 86400 × 30 × $0.0000646234 = $1,679.08/month
  Invocation on PC: depends on actual traffic
  
Total PC for 10 environments 1GB: ~$1,679/month in allocation alone!

→ PC has HIGH fixed cost. Only justifiable if the cost of cold start
  (SLA loss, impacted users) is greater than this value.
```

[FACT] A more cost-controlled alternative is to use **Application Auto Scaling** to dynamically adjust PC based on schedule:

```bash
# Scale PC to 10 from 8am to 6pm (business hours), 2 outside hours
aws application-autoscaling register-scalable-target \
  --service-namespace lambda \
  --resource-id function:my-function:prod \
  --scalable-dimension lambda:function:ProvisionedConcurrency \
  --min-capacity 2 \
  --max-capacity 10
```

---

### 4. SnapStart: cold starts for Java and .NET

[FACT] SnapStart is a Lambda feature that captures a **snapshot** of the execution environment after the INIT phase and reuses it for new invocations, eliminating runtime and static code initialization time. Instead of initializing the JVM and loading all classes on each cold start, Lambda restores the memory and disk state from the snapshot in milliseconds.

```
Without SnapStart (Java Spring Boot):
  Deploy new version → [JVM init: 1s] + [Spring init: 5s] + [handler: 100ms]
  Total cold start: ~6.1 seconds

With SnapStart:
  Deploy new version → Lambda does INIT and takes snapshot
  Cold invocation → [restore snapshot: ~10ms] + [handler: 100ms]
  Total cold start: ~110ms
```

[FACT] SnapStart limitations (May 2026):
- Available only for Java (managed runtimes) and .NET 8+.
- Does not support container image functions (only deployment package).
- Does not support functions with `ephemeralStorage > 512MB`.
- The snapshot is taken per version — each `PublishVersion` generates a new snapshot.
- Code that uses timestamps or randoms in INIT may have unexpected behavior when restored (time "freezes" in the snapshot).

[FACT] SnapStart has zero configuration cost. Billing is for normal invocation time (there is no "allocation" cost like PC).

---

## Practical example

**Scenario:** You have a REST API with Lambda + API Gateway. The function uses Node.js and connects to RDS PostgreSQL. Traffic is irregular: peaks of 500 req/s during business hours and almost zero at night. The P99 latency must be < 200ms.

### Measuring cold start in CloudWatch

```bash
# Filter invocations with Init Duration in Lambda logs
# (Init Duration appears only on cold starts)
aws logs filter-log-events \
  --log-group-name /aws/lambda/my-api-function \
  --filter-pattern '"Init Duration"' \
  --start-time $(date -d '1 hour ago' +%s)000 \
  | jq '.events[].message' \
  | grep -o 'Init Duration: [0-9.]*'

# Typical output:
# Init Duration: 823.45
# Init Duration: 791.12
# Init Duration: 1203.78
```

CloudWatch Insights for cold start analysis at scale:

```sql
-- CloudWatch Insights query: cold start distribution by hour
filter @message like /Init Duration/
| parse @message "Init Duration: * ms" as initDuration
| stats
    count(*) as coldStarts,
    avg(initDuration) as avgInit,
    pct(initDuration, 95) as p95Init,
    max(initDuration) as maxInit
  by bin(1h)
| sort by bin(1h) desc
```

### Calculating cost: PC vs on-demand for a traffic profile

```
Traffic profile:
  8am-6pm (10h/day × 22 business days = 220h/month): 100 invocations/s
  Rest (530h/month): 2 invocations/s

Function: 1GB memory, 100ms average execution

On-demand (without PC):
  Total invocations/month:
    100 req/s × 220h × 3600s = 79,200,000 invocations at peak
    2 req/s × 530h × 3600s  =  3,816,000 invocations off-peak
    Total: 83,016,000 invocations
  
  GB-seconds cost: 83,016,000 × 0.1s × 1GB = 8,301,600 GB-s
  Invocations cost: $0.20/M × 83.016M = $16.60
  GB-s cost: $0.0000200000 × 8,301,600 = $166.03
  Total on-demand: ~$182.63/month
  (+ cold starts in ~1% of invocations = ~830,000 cold starts)

With PC of 40 environments (peak of 100 req/s × 100ms = 10 concurrent + margin):
  PC allocation: 40 × 1GB × 720h = 28,800 GB-h = 103,680,000 GB-s
  PC allocated cost: 103,680,000 × $0.0000646234 = $6,698/month
  
  → PC is MUCH more expensive for this profile.
  
With PC of 5 environments (guarantees fast response for low traffic):
  PC allocation: 5 × 1GB × 720h × 3600s/h = 12,960,000 GB-s
  PC cost: 12,960,000 × $0.0000646234 = $837/month
  
  → Still more expensive than on-demand. PC only pays off if the SLA
    requires that ALL cold starts are eliminated, not just
    those during low traffic.

Conclusion for this profile:
  Best strategy: optimize static code to reduce cold start
  (lazy DB connection, remove unused dependencies)
  + accept occasional cold starts in exchange for 10x lower cost.
```

### CDK with Provisioned Concurrency via alias

```typescript
import { Stack, Duration } from 'aws-cdk-lib';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as appscaling from 'aws-cdk-lib/aws-applicationautoscaling';

// Function
const fn = new lambda.Function(this, 'ApiFunction', {
  runtime: lambda.Runtime.NODEJS_22_X,
  handler: 'index.handler',
  code: lambda.Code.fromAsset('lambda'),
  memorySize: 1024,
  timeout: Duration.seconds(30),
});

// Publishes immutable version (required for PC)
const version = fn.currentVersion;

// Alias 'prod' points to the current version
const prodAlias = new lambda.Alias(this, 'ProdAlias', {
  aliasName: 'prod',
  version,
  provisionedConcurrentExecutions: 5,  // PC configured on alias
});

// Auto Scaling of PC based on schedule
const target = prodAlias.addAutoScaling({
  minCapacity: 2,
  maxCapacity: 20,
});

// Scale to 10 at 8am, back to 2 at 6pm (UTC-3 = UTC+3 inverted)
target.scaleOnSchedule('ScaleUpMorning', {
  schedule: appscaling.Schedule.cron({ hour: '11', minute: '0' }), // 8am BRT
  minCapacity: 10,
});
target.scaleOnSchedule('ScaleDownEvening', {
  schedule: appscaling.Schedule.cron({ hour: '21', minute: '0' }), // 6pm BRT
  minCapacity: 2,
});
```

---

## Common pitfalls

### Pitfall 1: Database connections created inside the handler

**The mistake:** The code creates a new database connection on every invocation:

```python
def handler(event, context):
    # ❌ Connection created INSIDE the handler
    conn = psycopg2.connect(host=DB_HOST, ...)
    result = conn.execute("SELECT ...")
    conn.close()
    return result
```

**Why it happens:** The developer doesn't know the execution environment model or is worried about "stale" connections in warm environments.

**The cost:** A TCP + TLS connection to RDS takes 50-200ms. For a function that executes in 10ms, this is 500-2000% overhead. With 1000 invocations/minute, this creates and destroys 1000 connections/minute on the database, potentially exhausting the RDS max_connections.

**How to avoid:** Create connections in global scope (outside the handler). Use RDS Proxy to manage the connection pool on the database side — it aggregates connections from multiple execution environments into a smaller pool for RDS.

```python
# ✅ Connection created OUTSIDE the handler (static initialization)
conn = None

def get_connection():
    global conn
    if conn is None or conn.closed:
        conn = psycopg2.connect(host=DB_HOST, ...)
    return conn

def handler(event, context):
    c = get_connection()
    result = c.execute("SELECT ...")
    return result
```

---

### Pitfall 2: Provisioned Concurrency on `$LATEST`

**The mistake:** The pipeline configures PC on the function's `$LATEST` version:

```bash
aws lambda put-provisioned-concurrency-config \
  --function-name my-function \
  --qualifier '$LATEST' \   # ← this fails!
  --provisioned-concurrent-executions 5
```

**Why it happens:** `$LATEST` is a special alias that points to the most recent code. It's convenient for development but doesn't support PC because it's mutable — Lambda cannot maintain a snapshot of something that changes with every deploy.

**How to recognize:** Error `InvalidParameterValueException: Provisioned Concurrency Configurations are not supported on unpublished versions`. The cold start continues happening even after configuring PC.

**How to avoid:** Always publish a version (`PublishVersion`) before configuring PC, and apply PC to the numbered version or to an alias that points to it. In CDK, use `fn.currentVersion` which automatically publishes an immutable version.

---

### Pitfall 3: Unnecessary global imports increasing cold start

**The mistake:** The function imports complete libraries when it only uses one function from each:

```python
# ❌ Imports the entire SDK during initialization
import boto3
import pandas as pd
import numpy as np
from PIL import Image

def handler(event, context):
    # Only uses S3 and one JSON operation
    s3 = boto3.client('s3')
    # pandas, numpy, PIL are never used in this function
```

**Why it happens:** Copying code from elsewhere without reviewing imports, or keeping dependencies from a previous version of the function that was simplified.

**The cost:** `pandas` + `numpy` together can add 300-500ms to cold start from the import alone. A function that executes in 50ms can have an 800ms cold start because of unused imports.

**How to avoid:** Audit imports regularly. Use lazy imports (inside the handler) for rarely used libraries. For Node.js, use bundlers (esbuild via CDK's `NodejsFunction`) that perform tree-shaking and eliminate unused code from the final bundle.

---

## Reflection exercise

You have three Lambda functions with distinct profiles:

1. **auth-validator**: Node.js, 128MB, executes in 5ms, invoked 10,000 times/minute constantly 24h/day. Current cold start: 200ms.

2. **report-generator**: Java Spring Boot, 3GB, executes in 8s, invoked 2 times/hour, only during business hours. Current cold start: 6s.

3. **image-resizer**: Python, 512MB, executes in 300ms, invoked in bursts from 0 to 500 req/s in less than 1 minute (unpredictable marketing campaigns). Current cold start: 400ms.

For each function, decide: is the cold start a real problem in the usage context? If yes, which strategy would you use — code optimization, Provisioned Concurrency, SnapStart, or another approach? Calculate (approximately) the monthly cost of each chosen strategy and compare with the cost of doing nothing (how many users affected, what impact on SLA).

---

## Resources for further study

### 1. Understanding the Lambda execution environment lifecycle
**URL:** https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtime-environment.html
**What to find:** Detailed description of the Init, Invoke, and Shutdown phases, the 10-second limit of the Init phase, execution environment reuse behavior, and how extensions interact with the lifecycle.
**Why it's the right source:** It's the primary documentation of the execution model — the foundation for understanding all other concepts in this session.

### 2. Configuring provisioned concurrency for a function
**URL:** https://docs.aws.amazon.com/lambda/latest/dg/provisioned-concurrency.html
**What to find:** How to configure PC on versions and aliases, the billing difference between allocated PC and PC invocation, how behavior changes when PC is exhausted (fallback to on-demand), and how to use Application Auto Scaling to dynamically adjust PC.
**Why it's the right source:** It's the official feature reference, with all configuration and cost details.

### 3. Optimizing static initialization
**URL:** https://docs.aws.amazon.com/lambda/latest/dg/static-initialization.html
**What to find:** Best practices for reducing static initialization time (outside the handler): lazy initialization, caching heavy objects, measuring with `Init Duration` in logs, and examples per language.
**Why it's the right source:** It's the practical optimization guide for cold start without needing PC — the first line of defense before considering provisioned concurrency.

---
