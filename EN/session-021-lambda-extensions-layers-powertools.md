# Session — Lambda: Extensions, Layers and Power Tools

**Estimated duration:** 60 minutes
**Prerequisites:** session-020-lambda-event-source-mappings

---

## Objective

By the end, you will be able to create a Lambda Layer with shared dependencies, configure an external extension to run in the Lambda lifecycle, and instrument a function with Lambda Power Tools (structured logging, tracing, and metrics with one line of code).

---

## Context

[FACT] Three complementary mechanisms solve distinct problems in Lambda functions at scale: **Layers** solve dependency duplication between functions, **Extensions** solve the need for auxiliary code (telemetry agents, security scanners, secrets managers) that runs alongside the handler without modifying application code, and **Powertools** solves the implementation of observability patterns (structured logging, tracing, metrics) without repetitive boilerplate.

[CONSENSUS] These three mechanisms are frequently combined: Powertools is distributed as an AWS-managed Layer, many third-party observability agents (Datadog, Dynatrace, New Relic) are distributed as Extensions inside Layers, and any shared dependency between functions should be extracted to a Layer to avoid each function deploy including hundreds of MB of identical libraries.

---

## Core concepts

### 1. Lambda Layers: anatomy and how they work

[FACT] A Layer is a ZIP file containing code or dependencies that Lambda extracts to `/opt` before executing your function. The runtime adds `/opt` subdirectories to the `PATH` and the language's import path, making layer libraries directly importable without additional configuration.

```
Directory structure extracted in /opt:

Python:
  /opt/python/                     ← added to PYTHONPATH
  /opt/python/lib/python3.12/site-packages/
    requests/
    boto3/
    aws_lambda_powertools/

Node.js:
  /opt/nodejs/node_modules/        ← added to NODE_PATH
    @aws-lambda-powertools/
    axios/

Binaries (any runtime):
  /opt/bin/                        ← added to PATH
    datadog-agent

Shared:
  /opt/lib/                        ← .so libraries
```

[FACT] A function can have up to **5 layers** simultaneously. The total uncompressed size of the function + all layers cannot exceed **250 MB**. Each layer has immutable numbered versions — you always reference a specific version.

```
Layer ARN:
arn:aws:lambda:us-east-1:123456789012:layer:my-deps:7
                          ↑ account   ↑ name   ↑ version
```

[FACT] Layers can be shared between accounts via resource-based policy. AWS and partners publish public layers — for example, the Powertools for Python layer:

```
arn:aws:lambda:us-east-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312:8
               ↑ official AWS account  ↑ name with embedded runtime
```

---

### 2. Creating and publishing a Layer

**Package structure for Python:**

```bash
# 1. Install dependencies in the correct directory
mkdir -p layer/python
pip install requests psycopg2-binary --target layer/python

# 2. Create the ZIP with the correct structure
cd layer
zip -r ../my-deps-layer.zip python/

# 3. Publish the layer
aws lambda publish-layer-version \
  --layer-name my-dependencies \
  --zip-file fileb://../my-deps-layer.zip \
  --compatible-runtimes python3.12 python3.13 \
  --description "Shared dependencies: requests, psycopg2"

# Output: LayerVersionArn with version number
```

**CDK:**

```typescript
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as path from 'path';

// Layer created from a local directory
const depsLayer = new lambda.LayerVersion(this, 'DepsLayer', {
  code: lambda.Code.fromAsset(path.join(__dirname, '../layers/dependencies'), {
    bundling: {
      // Bundling in Docker to ensure correct Linux binaries
      image: lambda.Runtime.PYTHON_3_12.bundlingImage,
      command: [
        'bash', '-c',
        'pip install -r requirements.txt -t /asset-output/python && ' +
        'cp -r /asset-input/python /asset-output/',
      ],
    },
  }),
  compatibleRuntimes: [lambda.Runtime.PYTHON_3_12],
  description: 'Shared Python dependencies',
});

// Using the layer in a function
const fn = new lambda.Function(this, 'MyFunction', {
  runtime: lambda.Runtime.PYTHON_3_12,
  handler: 'index.handler',
  code: lambda.Code.fromAsset('lambda'),
  layers: [depsLayer],
});
```

[FACT] **Runtime compatibility is critical**: a layer built with Python 3.12 may have compiled binaries (C extensions like `psycopg2`, `numpy`) incompatible with Python 3.11 or 3.13. Always build the layer in the same environment as the function — use CDK Docker bundling or Amazon Linux 2023 to ensure compatibility with the Lambda execution environment.

---

### 3. Lambda Extensions: internal vs external

[FACT] Extensions are processes that run **in the same execution environment** as the Lambda function, but with direct access to the environment lifecycle via the **Extensions API**. They register to receive event notifications (`INVOKE`, `SHUTDOWN`) and can execute code before, during, and after each invocation.

**Two types:**

```
Internal Extensions:
  → Run in the SAME PROCESS as the runtime
  → Implemented via wrapper scripts (PYTHON_EXEC_WRAPPER, NODE_OPTIONS, JAVA_TOOL_OPTIONS)
  → Access to the function process
  → Example: SDK call interceptor for security

External Extensions (more common):
  → Run as SEPARATE PROCESSES in the execution environment
  → Register via the Extensions API via HTTP
  → Continue running after the function returns
  → Have access to the /tmp filesystem
  → Example: Datadog agent, secrets scanner, telemetry agent
```

[FACT] The lifecycle of an external extension:

```
INIT phase:
  1. Extension bootstrap (executable /opt/extensions/my-extension)
  2. POST /register → registers for INVOKE and/or SHUTDOWN events
  3. GET /event/next → blocks waiting for the next event
     (all extensions must reach here before the function executes)

INVOKE phase:
  4. Lambda receives invocation
  5. Extension receives INVOKE event via /event/next
  6. Extension and function execute CONCURRENTLY
  7. Function returns response to the caller
  8. Extension completes its post-invocation work
  9. Extension does GET /event/next → waits for next invocation

SHUTDOWN phase:
  10. Lambda decides to terminate the environment
  11. Extension receives SHUTDOWN event via /event/next
  12. Extension has 2 seconds to finalize (flush buffers, etc.)
  13. Environment is destroyed
```

[FACT] Extensions **increase cold start time**: Lambda waits for all registered extensions to reach the "ready" state (first GET /event/next) before executing the handler. A heavy extension can add 100-500ms to cold start. Partners like Datadog and Dynatrace have optimized their extensions to minimize this impact, but it's something to measure in production.

[FACT] Extensions reside in the `/opt/extensions/` directory when distributed via Layer. The file must be executable (`chmod +x`). Lambda automatically executes all binaries in this directory.

---

### 4. Lambda Powertools: observability without boilerplate

[FACT] Lambda Powertools is an open-source library maintained by AWS that implements the three observability pillars for Lambda with minimal code: **Logger** (structured logging), **Tracer** (X-Ray tracing), and **Metrics** (CloudWatch EMF). Available for Python and TypeScript (Node.js).

**Installation as AWS-managed Layer (without including in the package):**

```python
# Python — use the public AWS layer:
# arn:aws:lambda:us-east-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312:8
# (most recent version at docs.powertools.aws.dev/lambda/python)

# TypeScript — use the public layer:
# arn:aws:lambda:us-east-1:094274105915:layer:AWSLambdaPowertoolsTypeScriptV2:27
```

**Logger — structured logging with automatic Lambda context:**

```python
from aws_lambda_powertools import Logger

logger = Logger(service="order-service")  # initialization OUTSIDE the handler

@logger.inject_lambda_context(log_event=True)  # decorator: injects requestId, etc.
def handler(event, context):
    logger.info("Processing order", order_id=event["orderId"], amount=event["amount"])
    
    try:
        result = process_order(event)
        logger.info("Order processed successfully", result=result)
        return result
    except Exception as e:
        logger.exception("Failed to process order")  # includes stack trace
        raise
```

Structured JSON output in CloudWatch:
```json
{
  "level": "INFO",
  "location": "handler:12",
  "message": "Processing order",
  "order_id": "ord-123",
  "amount": 99.90,
  "service": "order-service",
  "timestamp": "2026-06-17T10:30:00.123Z",
  "xray_trace_id": "1-abc123",
  "cold_start": true,
  "function_request_id": "req-456",
  "function_arn": "arn:aws:lambda:..."
}
```

**Tracer — X-Ray tracing with automatic capture:**

```python
from aws_lambda_powertools import Tracer

tracer = Tracer(service="order-service")

@tracer.capture_lambda_handler    # creates subsegment for the entire handler
def handler(event, context):
    return process_order(event)

@tracer.capture_method            # creates subsegment for any method
def process_order(event):
    # X-Ray automatically captures errors and duration of this method
    charge_card(event["paymentInfo"])
    update_inventory(event["items"])
    return {"status": "ok"}
```

**Metrics — CloudWatch EMF without synchronous API calls:**

```python
from aws_lambda_powertools import Metrics
from aws_lambda_powertools.metrics import MetricUnit

metrics = Metrics(namespace="OrderService", service="order-service")

@metrics.log_metrics(capture_cold_start_metric=True)  # flushed automatically
def handler(event, context):
    metrics.add_metric(name="OrdersProcessed", unit=MetricUnit.Count, value=1)
    metrics.add_metric(name="OrderAmount", unit=MetricUnit.Dollars, value=event["amount"])
    metrics.add_dimension(name="PaymentMethod", value=event["paymentMethod"])
    
    result = process_order(event)
    return result
```

[FACT] Powertools Metrics uses **CloudWatch Embedded Metric Format (EMF)**: metrics are embedded in logs as structured JSON and extracted by CloudWatch automatically. This avoids synchronous calls to the CloudWatch API during invocation, which would add latency and extra cost.

**TypeScript (Middy middleware — functional approach):**

```typescript
import { Logger } from '@aws-lambda-powertools/logger';
import { Tracer } from '@aws-lambda-powertools/tracer';
import { Metrics, MetricUnit } from '@aws-lambda-powertools/metrics';
import { injectLambdaContext } from '@aws-lambda-powertools/logger/middleware';
import { captureLambdaHandler } from '@aws-lambda-powertools/tracer/middleware';
import { logMetrics } from '@aws-lambda-powertools/metrics/middleware';
import middy from '@middy/core';

const logger = new Logger({ serviceName: 'order-service' });
const tracer = new Tracer({ serviceName: 'order-service' });
const metrics = new Metrics({ namespace: 'OrderService', serviceName: 'order-service' });

const lambdaHandler = async (event: APIGatewayEvent) => {
  logger.info('Processing order', { orderId: event.pathParameters?.id });
  
  metrics.addMetric('OrdersProcessed', MetricUnit.Count, 1);
  
  const result = await processOrder(event);
  return { statusCode: 200, body: JSON.stringify(result) };
};

// Applies all middlewares in one line
export const handler = middy(lambdaHandler)
  .use(injectLambdaContext(logger, { logEvent: true }))
  .use(captureLambdaHandler(tracer))
  .use(logMetrics(metrics, { captureColdStartMetric: true }));
```

---

## Practical example

**Complete scenario:** A Lambda for order processing with:
- Layer for shared Python dependencies (`requests`, `psycopg2`).
- AWS-managed Powertools Layer.
- Complete observability: Logger + Tracer + Metrics.
- Datadog Extension via Layer (simulated with extension structure).

### Complete CDK

```typescript
import { Stack, StackProps, Duration } from 'aws-cdk-lib';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as logs from 'aws-cdk-lib/aws-logs';
import * as iam from 'aws-cdk-lib/aws-iam';
import { Construct } from 'constructs';

export class InstrumentedLambdaStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    // Layer 1: Custom Python dependencies
    const depsLayer = new lambda.LayerVersion(this, 'DepsLayer', {
      layerVersionName: 'order-service-deps',
      code: lambda.Code.fromAsset('layers/deps', {
        bundling: {
          image: lambda.Runtime.PYTHON_3_12.bundlingImage,
          command: [
            'bash', '-c',
            'pip install requests==2.31.0 psycopg2-binary==2.9.9 ' +
            '-t /asset-output/python --no-cache-dir',
          ],
        },
      }),
      compatibleRuntimes: [lambda.Runtime.PYTHON_3_12],
      description: 'requests + psycopg2 — pinned versions',
    });

    // Layer 2: Powertools (public AWS-managed layer)
    const powertoolsLayer = lambda.LayerVersion.fromLayerVersionArn(
      this,
      'PowertoolsLayer',
      // Public Powertools Python v3 layer ARN for Python 3.12
      `arn:aws:lambda:${this.region}:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312:8`
    );

    // Log Group with retention
    const logGroup = new logs.LogGroup(this, 'FnLogs', {
      logGroupName: '/aws/lambda/order-processor',
      retention: logs.RetentionDays.ONE_MONTH,
    });

    // Function with both layers
    const fn = new lambda.Function(this, 'OrderProcessor', {
      functionName: 'order-processor',
      runtime: lambda.Runtime.PYTHON_3_12,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambda/order-processor'),
      layers: [depsLayer, powertoolsLayer],
      timeout: Duration.seconds(30),
      memorySize: 512,
      environment: {
        // Powertools reads these variables automatically
        POWERTOOLS_SERVICE_NAME: 'order-service',
        POWERTOOLS_METRICS_NAMESPACE: 'OrderService',
        LOG_LEVEL: 'INFO',
        // X-Ray tracing active (required with tracer.capture_lambda_handler)
        POWERTOOLS_TRACE_ENABLED: 'true',
      },
      tracing: lambda.Tracing.ACTIVE,  // enables X-Ray
      logGroup,
    });

    // Permission for X-Ray
    fn.addToRolePolicy(new iam.PolicyStatement({
      actions: ['xray:PutTraceSegments', 'xray:PutTelemetryRecords'],
      resources: ['*'],
    }));
  }
}
```

### Function code (`lambda/order-processor/index.py`)

```python
import os
import json
import requests  # from DepsLayer
from aws_lambda_powertools import Logger, Tracer, Metrics
from aws_lambda_powertools.metrics import MetricUnit
from aws_lambda_powertools.utilities.typing import LambdaContext

# Initialization outside the handler (reused in warm invocations)
logger = Logger()       # reads POWERTOOLS_SERVICE_NAME from env
tracer = Tracer()       # reads POWERTOOLS_TRACE_ENABLED from env
metrics = Metrics()     # reads POWERTOOLS_METRICS_NAMESPACE from env

@logger.inject_lambda_context(log_event=True)
@tracer.capture_lambda_handler
@metrics.log_metrics(capture_cold_start_metric=True)
def handler(event: dict, context: LambdaContext) -> dict:
    order_id = event.get("orderId")
    amount = event.get("amount", 0)

    logger.info("Starting order processing", order_id=order_id)
    
    metrics.add_metric("OrdersReceived", MetricUnit.Count, 1)
    metrics.add_dimension("Environment", os.environ.get("ENV", "prod"))

    try:
        result = _process_order(order_id, amount)
        metrics.add_metric("OrdersSucceeded", MetricUnit.Count, 1)
        metrics.add_metric("OrderAmount", MetricUnit.NoUnit, amount)
        logger.info("Order processed", order_id=order_id, result=result)
        return {"statusCode": 200, "body": json.dumps(result)}
    except Exception as e:
        metrics.add_metric("OrdersFailed", MetricUnit.Count, 1)
        logger.exception("Processing failure", order_id=order_id)
        raise

@tracer.capture_method   # creates subsegment "## _process_order" in X-Ray
def _process_order(order_id: str, amount: float) -> dict:
    # Simulates external service call
    response = requests.post(
        "https://payment-api.internal/charge",
        json={"orderId": order_id, "amount": amount},
        timeout=5,
    )
    response.raise_for_status()
    return response.json()
```

### Validating the instrumentation

```bash
# 1. Verify structured logs in CloudWatch
aws logs tail /aws/lambda/order-processor --follow \
  | python3 -m json.tool

# 2. Verify EMF metrics in the OrderService namespace
aws cloudwatch list-metrics --namespace OrderService

# 3. Verify trace in X-Ray
aws xray get-trace-summaries \
  --start-time $(date -d '10 minutes ago' +%s) \
  --end-time $(date +%s) \
  --query 'TraceSummaries[?HasError==`false`].[Id,Duration]'

# 4. Verify layers configured on the function
aws lambda get-function-configuration \
  --function-name order-processor \
  --query 'Layers[*].Arn'
```

---

## Common pitfalls

### Pitfall 1: Layer built on macOS with binaries incompatible with Linux

**The mistake:** You install Python dependencies with `pip install -t ./python` on macOS and package as a layer. The layer works locally (tests with moto/localstack) but fails in production with `ImportError: /opt/python/psycopg2/_psycopg.cpython-312-darwin.so: cannot execute binary file`.

**Why it happens:** Packages with C extensions (`psycopg2`, `cryptography`, `numpy`, `Pillow`) compile binaries specific to the operating system. macOS binaries (darwin) are incompatible with Linux (the Lambda execution environment).

**How to recognize:** `ImportError` with `.so` or `.dylib` in the path, or message `cannot execute binary file`. The problem only appears in production — local tests on the same macOS pass.

**How to avoid:** Always build layers with C extensions using Docker with the correct Lambda runtime image (`public.ecr.aws/lambda/python:3.12`), or use CDK Docker bundling (`lambda.Runtime.PYTHON_3_12.bundlingImage`). For psycopg2 specifically, there's the `psycopg2-binary` package that already includes the necessary libraries.

---

### Pitfall 2: Extension increasing cold start beyond acceptable

**The mistake:** You add a security extension (vulnerability scanner) from a vendor that starts a heavy Go process on cold start. The cold start of a Node.js function that was 200ms becomes 1.8 seconds after adding the extension via Layer. The service latency SLAs are violated.

**Why it happens:** External extensions block the INIT phase: Lambda waits for all extensions to reach the "ready" state (GET /event/next) before executing the handler. A slow extension blocks the entire INIT phase.

**How to recognize:** The `Init Duration` in Lambda logs increases significantly after adding the extension. The CloudWatch dashboard shows a spike in P99 duration after the deploy.

**How to avoid:** Measure the impact of each extension on cold start with and without it before going to production. Check if the vendor has lazy or asynchronous initialization options. Consider using Provisioned Concurrency to absorb the cold start if the extension is mandatory but the impact is unacceptable.

---

### Pitfall 3: Powertools Metrics without `@log_metrics` decorator — metrics never sent

**The mistake:** The developer uses Powertools `Metrics` but forgets the `@metrics.log_metrics` decorator on the handler:

```python
# ❌ Metrics are never sent
metrics = Metrics()

def handler(event, context):
    metrics.add_metric("OrdersProcessed", MetricUnit.Count, 1)
    return {"statusCode": 200}
    # → metrics added but never flushed to CloudWatch
```

**Why it happens:** Powertools Metrics uses EMF — metrics are emitted by writing a special JSON to stdout at the end of invocation. Without the decorator (or manual call to `metrics.flush_metrics()`), the JSON is never written, and metrics are silently discarded.

**How to recognize:** No metrics appear in the configured CloudWatch namespace, even with `add_metric` code visible in the logs. There's no error — metrics simply vanish.

**How to avoid:** Always use `@metrics.log_metrics` on the main handler. For cases where the decorator isn't viable (complex async handlers, web frameworks), call `metrics.flush_metrics()` explicitly in a `finally` block to ensure metrics are sent even in case of error.

---

## Reflection exercise

You have 15 Lambda functions in a payments microservice. All use: `requests==2.31.0`, `boto3==1.34.0`, `cryptography==42.0.0`, `aws-lambda-powertools==3.x`. The deployment package of each function includes these dependencies — each ZIP is ~45 MB.

How would you structure layers to reduce deployment sizes and simplify dependency updates? Would you create a single layer with everything or multiple separate layers by category — and what are the trade-offs of each approach? When a vulnerability is discovered in `cryptography` and you need to update to `42.0.1`, what is the layer update process and how do you ensure all 15 functions use the updated version without downtime? And if two functions need different versions of `boto3` due to incompatibility with other dependencies — can layers solve this problem, or is there a fundamental limitation that makes it impossible?

---

## Resources for further study

### 1. Managing Lambda dependencies with layers
**URL:** https://docs.aws.amazon.com/lambda/latest/dg/chapter-layers.html
**What to find:** Complete guide to creating and managing layers: directory structure per runtime, how to publish, how to share between accounts, limits (250MB, 5 layers), and the behavior of deleted layers on existing functions.
**Why it's the right source:** It's the primary reference for layers — covers all details that affect compatibility and maintenance.

### 2. Augment Lambda functions using Lambda extensions
**URL:** https://docs.aws.amazon.com/lambda/latest/dg/lambda-extensions.html
**What to find:** Extension types (internal vs external), how the Extensions API works, the complete lifecycle with diagram, how extensions are distributed via layers, and the list of partner extensions (Datadog, Dynatrace, New Relic, etc.).
**Why it's the right source:** It's the official entry point for the topic — includes the lifecycle sequence diagram that clarifies the cold start impact.

### 3. Lambda Powertools for Python — official documentation
**URL:** https://docs.powertools.aws.dev/lambda/python/latest/
**What to find:** Complete documentation of Logger, Tracer, Metrics with advanced examples: log sampling, X-Ray annotation, high-resolution metrics, middleware pattern, and additional utilities (idempotency, batch processing, feature flags).
**Why it's the right source:** It's the primary source — more up-to-date than AWS documentation and includes advanced pattern examples that don't appear in the basic guide.

---
