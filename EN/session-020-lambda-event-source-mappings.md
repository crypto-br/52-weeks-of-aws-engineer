# Session — Lambda: event source mappings — SQS, Kinesis and DynamoDB Streams with filtering

**Estimated duration:** 60 minutes
**Prerequisites:** session-019-lambda-execution-cold-starts

---

## Objective

By the end, you will be able to configure an event source mapping for SQS with batch size and bisect-on-error, add an event filter to process only events with specific properties (without invoking Lambda unnecessarily), and understand the retry and DLQ behavior for each source.

---

## Context

[FACT] An **Event Source Mapping (ESM)** is a Lambda resource that polls an event source (SQS, Kinesis, DynamoDB Streams, Kafka, MQ) and invokes your function with batches of records. It is Lambda that actively pulls the data — unlike asynchronous triggers (S3, SNS, EventBridge) where the source pushes the event to Lambda. This distinction has profound implications for retry behavior, ordering, and error handling.

[FACT] The three most common sources have very different data models and guarantees:

| | SQS | Kinesis | DynamoDB Streams |
|---|---|---|---|
| Model | Queue (destruction after read) | Stream (retention 24h-365 days) | Table change stream |
| Ordering | Per message group (FIFO) or no guarantee (Standard) | Guaranteed per shard | Guaranteed per partition key |
| Retry | By visibility timeout | Per shard (blocking) | Per shard (blocking) |
| bisectOnError | ❌ Not available | ✅ Available | ✅ Available |
| DLQ | On the SQS queue (native) | On-failure destination (S3, SQS, SNS) | On-failure destination |

[CONSENSUS] The most critical mistake when working with ESMs is not understanding the failure semantics of each source. In SQS, a failure returns the message to the queue (by visibility timeout) and another worker can process it — the shard is not blocked. In Kinesis and DynamoDB Streams, a failure **blocks the entire shard** until resolved. Processing a Kinesis item with an error indefinitely can paralyze processing of all subsequent records in that shard — including those that have no problem.

---

## Core concepts

### 1. Event Source Mapping with SQS

[FACT] The Lambda ESM performs long polling on the SQS queue (up to 20 seconds per request) and invokes the function with a batch of messages. The processing cycle is:

```
Lambda ESM                    SQS Queue
     │                             │
     │─── ReceiveMessage ─────────▶│
     │◀── Batch of N messages ─────│
     │                             │  messages become invisible
     │                             │  during visibility timeout
     │
     │─── Invoke(batch) ──────────▶ Lambda Function
     │                                    │
     │                              processes messages
     │                                    │
     │     ┌─────────── Success ──────────┘
     │     │            Lambda returned without error
     │     ▼
     │  DeleteMessage (all messages in the batch)
     │
     │     ┌─────────── Total failure ────┘
     │     │            Lambda threw exception
     │     ▼
     │  Does NOT call DeleteMessage
     │  After visibility timeout: messages return to queue
     │  If maxReceiveCount reached → messages go to DLQ

     │     ┌─────── Partial failure ──────┘ (with ReportBatchItemFailures)
     │     │        Lambda returned { batchItemFailures: [...] }
     │     ▼
     │  DeleteMessage only for successful messages
     │  Failed messages return to queue by visibility timeout
```

[FACT] Key ESM parameters for SQS:

| Parameter | Default | Range | Description |
|---|---|---|---|
| `batchSize` | 10 | 1–10,000 | Maximum messages per invocation |
| `maximumBatchingWindowInSeconds` | 0 | 0–300 | Waits N seconds to accumulate messages before invoking |
| `functionResponseTypes` | `[]` | `[ReportBatchItemFailures]` | Enables partial batch response |

[FACT] `maximumBatchingWindowInSeconds` is critical for throughput. With `batchSize=100` and `batchingWindow=0`, Lambda is invoked with any number of available messages (up to 100), including batches of 1 message. With `batchingWindow=5`, Lambda accumulates messages for up to 5 seconds before invoking, resulting in larger batches and fewer total invocations — reduces cost and increases efficiency.

---

### 2. Partial Batch Response (ReportBatchItemFailures)

[FACT] By default, if *any* message in the batch fails, *all* return to the queue. This causes reprocessing of messages that were already successfully processed. The solution is `ReportBatchItemFailures`: the function explicitly returns which messages failed, and Lambda only returns those to the queue.

```python
# Python: partial batch response for SQS
def handler(event, context):
    failures = []
    
    for record in event['Records']:
        try:
            process_message(record['body'])
        except Exception as e:
            print(f"Failed to process {record['messageId']}: {e}")
            failures.append({'itemIdentifier': record['messageId']})
    
    # Returns only the messages that failed
    # Messages not listed here are deleted successfully
    return {'batchItemFailures': failures}
```

```typescript
// TypeScript: partial batch response
import { SQSHandler, SQSBatchResponse } from 'aws-lambda';

export const handler: SQSHandler = async (event): Promise<SQSBatchResponse> => {
  const failures: SQSBatchResponse['batchItemFailures'] = [];

  await Promise.allSettled(
    event.Records.map(async (record) => {
      try {
        await processMessage(JSON.parse(record.body));
      } catch (err) {
        failures.push({ itemIdentifier: record.messageId });
      }
    })
  );

  return { batchItemFailures: failures };
};
```

[FACT] For `ReportBatchItemFailures` to work, the ESM must be configured with `functionResponseTypes: [ReportBatchItemFailures]`. If the ESM doesn't have this configuration, Lambda ignores the `batchItemFailures` field in the return and treats any response as total success.

---

### 3. Event Source Mapping with Kinesis and DynamoDB Streams

[FACT] Kinesis and DynamoDB Streams are different from SQS in one fundamental aspect: records are organized in **shards**, and processing of each shard is **sequential and blocking**. If a batch fails, Lambda reprocesses the same batch (or a subset, with bisect) before advancing to the next record in the shard.

```
Kinesis Shard
  ├── Record 1 (position: 1000)  ← processed successfully
  ├── Record 2 (position: 1001)  ← processed successfully
  ├── Record 3 (position: 1002)  ← FAILURE → retry batch containing 3+
  ├── Record 4 (position: 1003)  ← BLOCKED waiting for resolution of 3
  ├── Record 5 (position: 1004)  ← BLOCKED
  └── ...

Without bisect: retries the entire batch {3, 4, 5} until resolved
With bisect:
  1st attempt: batch {3, 4, 5} fails
  2nd attempt: bisect → {3} and {4, 5}
    - {3} fails → bisect → {3} (batch of 1, no bisect possible)
    - {4, 5} success
  3rd attempt: retries {3} until maxRetry or expiration
```

[FACT] Key parameters for Kinesis/DynamoDB Streams:

| Parameter | Default | Range | Description |
|---|---|---|---|
| `batchSize` | 100 | 1–10,000 | Records per invocation |
| `bisectBatchOnFunctionError` | false | boolean | Splits batch in half on error |
| `maximumRetryAttempts` | -1 (infinite) | -1, 0–10,000 | Attempts before discarding |
| `maximumRecordAgeInSeconds` | -1 (infinite) | -1, 60–604,800 | Discards records older than N seconds |
| `destinationConfig.onFailure` | none | SQS/SNS/S3/Lambda ARN | Destination for discarded records |
| `startingPosition` | required | `LATEST`, `TRIM_HORIZON`, `AT_TIMESTAMP` | Reading start point |

[FACT] `maximumRetryAttempts: -1` (infinite) + `maximumRecordAgeInSeconds: -1` (infinite) means a problematic record blocks the shard **forever** until manually resolved. In production, always configure at least one of these to avoid eternally blocked shards.

[FACT] `startingPosition` for new ESMs:
- `TRIM_HORIZON`: reads from the oldest available record. Use when setting up an ESM for the first time on a stream with existing data that should be processed.
- `LATEST`: ignores existing records and starts only with new ones. Use when historical data should not be processed.
- `AT_TIMESTAMP`: starts from a specific timestamp. Useful for replaying a specific period.

---

### 4. Event Filtering

[FACT] Event filters allow the ESM to discard records *before* invoking Lambda — at no invocation cost. For SQS, records discarded by the filter are **automatically deleted from the queue** (they don't return for reprocessing). For Kinesis and DynamoDB Streams, the iterator advances past filtered records (does not block).

[FACT] The filter syntax is based on EventBridge pattern matching. Each filter is a JSON object describing the properties that the event **must have** to be processed. Multiple filters are combined with logical OR.

**For SQS** — filters on the `body` field (which must be valid JSON):

```json
{
  "Filters": [
    {
      "Pattern": "{\"body\": {\"eventType\": [\"ORDER_PLACED\", \"ORDER_UPDATED\"]}}"
    }
  ]
}
```

Only SQS messages whose `body` contains `eventType` equal to `ORDER_PLACED` or `ORDER_UPDATED` reach Lambda.

**For DynamoDB Streams** — filters on the `dynamodb` field:

```json
{
  "Filters": [
    {
      "Pattern": "{\"dynamodb\": {\"NewImage\": {\"status\": {\"S\": [\"ACTIVE\"]}}}}"
    },
    {
      "Pattern": "{\"eventName\": [\"INSERT\"]}"
    }
  ]
}
```

This filter processes records where: `status = 'ACTIVE'` in the new image **OR** the event type is `INSERT`.

**For Kinesis** — filters on the `data` field (base64 decoded, must be JSON):

```json
{
  "Filters": [
    {
      "Pattern": "{\"data\": {\"source\": [\"mobile-app\"], \"priority\": [{\"numeric\": [\">=\", 3]}]}}"
    }
  ]
}
```

[FACT] Supported operators in filters:

```
Value comparison:
  ["value"]                    → exact equality
  [{"prefix": "ord-"}]        → starts with
  [{"suffix": "-v2"}]         → ends with (for strings)
  [{"anything-but": ["X"]}]   → any value except X

Numeric (SQS and Kinesis only, not DynamoDB Streams):
  [{"numeric": ["=", 100]}]
  [{"numeric": [">", 0, "<", 100]}]
  [{"numeric": [">=", 0]}]

Existence:
  [{"exists": true}]          → field exists in the event
  [{"exists": false}]         → field does not exist
```

---

## Practical example

**Scenario:** An e-commerce system with three processing pipelines:

1. **SQS**: order queue, processes only orders with `status: "PLACED"` with partial batch response.
2. **DynamoDB Streams**: reacts to changes in the inventory table, processes only when `quantity` drops to zero (stock depletion alert).
3. **Kinesis**: user event stream, processes only high-priority click events.

### Pipeline 1: SQS with filter and partial batch response (CDK)

```typescript
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as lambdaEventSources from 'aws-cdk-lib/aws-lambda-event-sources';
import * as sqs from 'aws-cdk-lib/aws-sqs';
import { Duration } from 'aws-cdk-lib';

const orderQueue = new sqs.Queue(this, 'OrderQueue', {
  visibilityTimeout: Duration.seconds(180),  // >= 6x Lambda timeout
  deadLetterQueue: {
    queue: new sqs.Queue(this, 'OrderDLQ'),
    maxReceiveCount: 5,   // after 5 failures → DLQ
  },
});

const orderProcessor = new lambda.Function(this, 'OrderProcessor', {
  runtime: lambda.Runtime.NODEJS_22_X,
  handler: 'index.handler',
  code: lambda.Code.fromAsset('lambda/order-processor'),
  timeout: Duration.seconds(30),
});

orderProcessor.addEventSource(
  new lambdaEventSources.SqsEventSource(orderQueue, {
    batchSize: 50,
    maxBatchingWindow: Duration.seconds(10),
    // Enables partial batch response
    reportBatchItemFailures: true,
    // Filter: processes only orders with status PLACED
    filters: [
      lambda.FilterCriteria.filter({
        body: {
          status: lambda.FilterRule.isEqual('PLACED'),
        },
      }),
    ],
  })
);
```

### Pipeline 2: DynamoDB Streams with stock depletion filter

```typescript
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';

const inventoryTable = new dynamodb.Table(this, 'InventoryTable', {
  partitionKey: { name: 'productId', type: dynamodb.AttributeType.STRING },
  stream: dynamodb.StreamViewType.NEW_AND_OLD_IMAGES,  // needs both images
  billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
});

const stockAlertFn = new lambda.Function(this, 'StockAlert', {
  runtime: lambda.Runtime.PYTHON_3_12,
  handler: 'index.handler',
  code: lambda.Code.fromAsset('lambda/stock-alert'),
  timeout: Duration.seconds(30),
});

stockAlertFn.addEventSource(
  new lambdaEventSources.DynamoEventSource(inventoryTable, {
    startingPosition: lambda.StartingPosition.LATEST,
    batchSize: 100,
    bisectBatchOnError: true,            // splits problematic batch
    retryAttempts: 3,                    // maximum of 3 attempts
    maxRecordAge: Duration.hours(1),     // discards records older than 1h
    // On-failure: saves discarded records to SQS for manual analysis
    onFailure: new lambdaEventSources.SqsDlq(
      new sqs.Queue(this, 'StreamDLQ')
    ),
    // Filter: only MODIFY where quantity changed to 0
    filters: [
      lambda.FilterCriteria.filter({
        eventName: lambda.FilterRule.isEqual('MODIFY'),
        dynamodb: {
          NewImage: {
            quantity: {
              N: lambda.FilterRule.isEqual('0'),
            },
          },
        },
      }),
    ],
  })
);
```

### Pipeline 3: Kinesis with priority filter

```typescript
import * as kinesis from 'aws-cdk-lib/aws-kinesis';

const eventStream = new kinesis.Stream(this, 'UserEventStream', {
  shardCount: 4,
  retentionPeriod: Duration.days(7),
});

const highPriorityProcessor = new lambda.Function(this, 'HighPriorityProcessor', {
  runtime: lambda.Runtime.NODEJS_22_X,
  handler: 'index.handler',
  code: lambda.Code.fromAsset('lambda/high-priority'),
  timeout: Duration.seconds(60),
});

highPriorityProcessor.addEventSource(
  new lambdaEventSources.KinesisEventSource(eventStream, {
    startingPosition: lambda.StartingPosition.LATEST,
    batchSize: 200,
    maxBatchingWindow: Duration.seconds(5),
    bisectBatchOnError: true,
    retryAttempts: 2,
    maxRecordAge: Duration.minutes(30),
    reportBatchItemFailures: true,
    // On-failure: saves to S3 for later analysis
    onFailure: new lambdaEventSources.S3OnFailureDestination(
      new s3.Bucket(this, 'FailedRecords')
    ),
    // Filter: only click events with priority >= 3
    filters: [
      lambda.FilterCriteria.filter({
        data: {
          eventType: lambda.FilterRule.isEqual('CLICK'),
          priority: lambda.FilterRule.between(3, 10),
        },
      }),
    ],
  })
);
```

### Handler with partial batch response for Kinesis

```typescript
// lambda/high-priority/index.ts
import { KinesisStreamHandler, KinesisStreamBatchResponse } from 'aws-lambda';

export const handler: KinesisStreamHandler = async (
  event
): Promise<KinesisStreamBatchResponse> => {
  const failures: KinesisStreamBatchResponse['batchItemFailures'] = [];

  for (const record of event.Records) {
    try {
      const data = JSON.parse(
        Buffer.from(record.kinesis.data, 'base64').toString('utf-8')
      );
      await processHighPriorityClick(data);
    } catch (err) {
      console.error(`Failed on record ${record.kinesis.sequenceNumber}:`, err);
      // For Kinesis, the identifier is sequenceNumber
      failures.push({ itemIdentifier: record.kinesis.sequenceNumber });
    }
  }

  return { batchItemFailures: failures };
};
```

---

## Common pitfalls

### Pitfall 1: visibilityTimeout shorter than the Lambda timeout (SQS)

**The mistake:** The SQS queue has `visibilityTimeout: 30s` and the Lambda has `timeout: 30s`. The Lambda processes a batch that takes exactly 30 seconds. At that point, the visibility timeout expires, and the queue makes the messages visible again — the ESM or another consumer picks them up and starts reprocessing. The Lambda is still executing (the 30s is the timeout, not the actual time). Result: messages processed in duplicate.

**Why it happens:** The visibility timeout must be greater than the maximum time Lambda can take to process a batch. AWS recommends `visibilityTimeout >= 6 × functionTimeout` to allow margin for retries.

**How to avoid:** `visibilityTimeout = max(6 × functionTimeout, 30s)`. For a Lambda with a 5-minute timeout processing large batches, the visibility timeout should be at least 30 minutes.

---

### Pitfall 2: Kinesis/DynamoDB Streams shard blocked by unrecoverable record

**The mistake:** A corrupted record (invalid JSON, unexpected schema) enters the stream. Lambda fails to process it. With `maximumRetryAttempts: -1` (infinite) and `maximumRecordAgeInSeconds: -1` (infinite), the ESM retries the same record forever. All subsequent records in that shard are blocked. Kinesis lag grows indefinitely. The system stops processing events from that shard.

**Why it happens:** Records in streams are processed in order per shard. A permanent failure without discard configuration or retry limit locks the shard indefinitely.

**How to recognize:** The `IteratorAgeMilliseconds` metric for Kinesis grows continuously without stabilizing. The CloudWatch Alarm for `IteratorAge` fires. In the Lambda console, the ESM shows very high `IteratorAge` for some shards and zero for others.

**How to avoid:** Always configure:
- `maximumRetryAttempts: 3` (or another finite value)
- `maximumRecordAgeInSeconds: 3600` (or a value appropriate for the SLA)
- `destinationConfig.onFailure`: to not lose discarded records
- `bisectBatchOnError: true`: to isolate problematic records quickly

---

### Pitfall 3: SQS filter deleting messages unintentionally

**The mistake:** You configure a filter on the SQS ESM to process only messages with `eventType: "ORDER_PLACED"`. Other messages (e.g., `eventType: "INVENTORY_UPDATE"`) were previously processed by another Lambda via a separate ESM, but you forgot to remove the filter from the new Lambda. When the new Lambda is deployed with the filter, messages of type `INVENTORY_UPDATE` are automatically **deleted from the queue** by the ESM, without being processed by anyone.

**Why it happens:** [FACT] For SQS, messages filtered by the ESM are automatically deleted from the queue — they don't return for reprocessing. This behavior is different from Kinesis/DynamoDB Streams, where the iterator simply advances.

**How to recognize:** Messages disappearing from the queue without appearing in Lambda logs (no invocation was made). The queue's `NumberOfMessagesSent` metric is greater than `NumberOfMessagesDeleted` via Lambda + DLQ — there's a delta of messages that disappeared without processing.

**How to avoid:** Understand that filters on SQS ESMs are destructive. Use SQS filters only when you are certain that filtered messages should be permanently discarded, not merely ignored. For routing (a message should go to Lambda A or Lambda B), use SNS fan-out with subscription filters or EventBridge rules instead of ESM SQS filters.

---

## Reflection exercise

You are designing the order processing pipeline for an e-commerce system. The SQS queue receives three types of messages: `ORDER_PLACED`, `ORDER_CANCELLED`, and `PAYMENT_CONFIRMED`. Each type should be processed by a different Lambda.

If you create a single ESM for each Lambda with filters for the respective message type, what fundamental problem occurs with messages of the non-filtered types? How would you solve this while maintaining loose coupling? Also consider the following: processing `PAYMENT_CONFIRMED` is critical and cannot have duplicates; processing `ORDER_PLACED` can have slight duplication if necessary. How would you configure the visibility timeout, the DLQ's maxReceiveCount, and the use of `ReportBatchItemFailures` differently for each Lambda, given these distinct requirements?

---

## Resources for further study

### 1. Using Lambda with Amazon SQS
**URL:** https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html
**What to find:** Complete guide for the SQS ESM: batch size and batching window configuration, partial batch response with code examples, retry behavior, and DLQ configuration. Includes the `visibilityTimeout >= 6 × functionTimeout` recommendation.
**Why it's the right source:** It's the primary reference for SQS + Lambda — covers all configuration details that affect reliability.

### 2. Control which events Lambda sends to your function (event filtering)
**URL:** https://docs.aws.amazon.com/lambda/latest/dg/invocation-eventfiltering.html
**What to find:** Complete reference for filter syntax, available operators, examples for each source (SQS, Kinesis, DynamoDB Streams), and discard behavior by source (SQS deletes, streams advance the iterator).
**Why it's the right source:** It's the canonical documentation for filters — essential for understanding the behavior differences between sources.

### 3. How Lambda processes records from stream and queue-based event sources
**URL:** https://docs.aws.amazon.com/lambda/latest/dg/invocation-eventsourcemapping.html
**What to find:** Unified view of how the ESM works for all sources: polling, batch window, concurrency per shard, and all error handling parameters (`bisectBatchOnFunctionError`, `maximumRetryAttempts`, `maximumRecordAgeInSeconds`, `destinationConfig`).
**Why it's the right source:** It's the most complete reference document on ESM behavior — the right place to understand the mechanics before debugging production problems.

---
