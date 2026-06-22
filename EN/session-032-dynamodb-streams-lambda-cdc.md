# Session 032 — DynamoDB Streams: Lambda integration for CDC and event-driven

**Estimated duration:** 60 minutes  
**Dependencies:** session-031-dynamodb-gsi-lsi-hot-partitions, session-020-lambda-event-source-mappings

---

## Objective

Understand the anatomy of DynamoDB Streams, their delivery and ordering guarantees, and master the event source mapping configuration with Lambda to implement CDC (Change Data Capture) patterns and reliable event-driven pipelines.

---

## Context

[FACT] DynamoDB Streams is an ordered and durable change log of a DynamoDB table. Each change to an item (INSERT, MODIFY, REMOVE) generates a record in the stream within 24 hours of retention. The stream captures changes in the order they occurred, per item.

CDC (Change Data Capture) is the pattern of capturing and reacting to each data mutation as it occurs — instead of doing periodic polling. [CONSENSUS] DynamoDB Streams is the canonical way to implement CDC in AWS serverless architectures: auditing, cross-region replication, cache invalidation, projection to Elasticsearch/OpenSearch, and fan-out to multiple consumers via SNS/SQS.

---

## Main concepts

### 1. Stream anatomy: view types and retention

[FACT] When enabling a stream, you choose a `StreamViewType` that determines the content of each record:

```
StreamViewType            What each record contains
─────────────────────────────────────────────────────────────────
KEYS_ONLY                 Only PK (+ SK if it exists)
NEW_IMAGE                 Complete item after the change
OLD_IMAGE                 Complete item before the change
NEW_AND_OLD_IMAGES        Both — before and after image
─────────────────────────────────────────────────────────────────
```

[FACT] `StreamViewType` **cannot be changed** after stream creation. To change, you must disable the current stream and create a new one.

[FACT] Retention: 24 hours. Records older than 24h are automatically deleted.

**Special case — PutItem/UpdateItem without actual change:**  
[FACT] If an `UpdateItem` does not change any value (SET attr = attr), **no record is written to the stream**. The stream only captures effective changes.

**Special case — TTL:**  
[FACT] When DynamoDB removes an item by TTL expiration, it generates a REMOVE record with a special `userIdentity` field:

```json
{
  "eventName": "REMOVE",
  "userIdentity": {
    "type": "Service",
    "principalId": "dynamodb.amazonaws.com"
  },
  "dynamodb": { ... }
}
```

This allows distinguishing TTL removal from application removal (which does not have `userIdentity`).

---

### 2. Shards: ephemeral structure and parent-child lineage

[FACT] The stream is divided into **shards**. Each shard contains a subset of records, ordered by `SequenceNumber`.

```
Stream
├── Shard-A (parent) ────── closed, 24h of records
│     └── Shard-C (child) ─ active
└── Shard-B (parent) ────── closed
      └── Shard-D (child) ─ active

Rule: process parent BEFORE child
      (SequenceNumber of parent precedes child)
```

[FACT] Shards are **ephemeral**: automatically created and deleted by DynamoDB in response to throughput changes in the table (physical partition split/merge). You do not control the number of shards.

[FACT] **Critical limit: maximum 2 simultaneous readers per shard.** Above that: `ProvisionedThroughputExceededException` on the stream. If you use Lambda + Kinesis Data Streams as fan-out, you're close to the limit. [CONSENSUS] For multiple independent consumers, use the pattern: DynamoDB Streams → Lambda → SNS → multiple SQS queues (fan-out).

**Formal guarantees:**  
[FACT] 1. **Exactly-once**: each record appears exactly once in the stream.  
[FACT] 2. **In-order per item**: all changes to the same item appear in order on the same shard. Changes to different items may be on different shards, with no guaranteed relative order between them.

---

### 3. Lambda event structure

[FACT] When Lambda consumes a stream, it receives a batch of records. Each record follows this structure:

```json
{
  "eventID": "1",
  "eventVersion": "1.1",
  "eventSource": "aws:dynamodb",
  "awsRegion": "us-east-1",
  "eventName": "INSERT",          // INSERT | MODIFY | REMOVE
  "dynamodb": {
    "StreamViewType": "NEW_AND_OLD_IMAGES",
    "SequenceNumber": "111",
    "ApproximateCreationDateTime": 1428537600,
    "Keys": {
      "PK": {"S": "USER#123"},
      "SK": {"S": "ORDER#2024-01-15T10:00:00Z"}
    },
    "NewImage": {
      "PK":     {"S": "USER#123"},
      "SK":     {"S": "ORDER#2024-01-15T10:00:00Z"},
      "status": {"S": "PENDING"},
      "total":  {"N": "299.90"}
    },
    "OldImage": {
      "PK":     {"S": "USER#123"},
      "SK":     {"S": "ORDER#2024-01-15T10:00:00Z"},
      "status": {"S": "PROCESSING"},
      "total":  {"N": "299.90"}
    }
  }
}
```

[FACT] Values use the **DynamoDB type system** with type discriminators: `{"S": "string"}`, `{"N": "123"}`, `{"BOOL": true}`, `{"NULL": true}`, `{"L": [...]}`, `{"M": {...}}`.

---

### 4. Event Source Mapping: configuration and filtering

[FACT] The event source mapping (ESM) is configured on Lambda, not on the stream. Main parameters:

```
Parameter                         Values / Notes
─────────────────────────────────────────────────────────────────
StartingPosition              TRIM_HORIZON (beginning of stream)
                              LATEST (only new records)
                              AT_TIMESTAMP (specific point)
BatchSize                     1–10,000 (default: 100)
BisectBatchOnFunctionError    True: splits batch in half on error
                              (binary search to isolate poison pill)
MaximumRetryAttempts          -1 (infinite) up to 10,000
DestinationConfig             OnFailure → SQS or SNS (DLQ)
FilterCriteria                Up to 5 filters (logical AND between fields
                              within a filter; logical OR between filters)
─────────────────────────────────────────────────────────────────
```

**Event filtering:**  
[FACT] Filters operate on the `dynamodb` key of the event, not on the external envelope (except `eventName` which is accessed directly).

```
Correct filter structure in ESM:
{
  "Filters": [
    {
      "Pattern": "{\"eventName\": [\"INSERT\"], \"dynamodb\": {\"NewImage\": {\"status\": {\"S\": [{\"prefix\": \"OPEN\"}]}}}}"
    }
  ]
}
```

[FACT] Filtering rules for DynamoDB Streams:
- String values: `["exact_value"]` or `[{"prefix": "val"}]`
- Numeric values: `[{"numeric": [">", 100]}]` — **only** for numeric attributes in `NewImage`/`OldImage`; numeric operators are **not** supported in `Keys`
- Exists: `[{"exists": true}]` or `[{"exists": false}]` — works only on **leaf nodes** (scalar attributes), not on intermediate objects
- Maximum: 5 filters per ESM; AND between conditions within the same filter; OR between different filters

---

### 5. CDC Pattern: replication and fan-out

[CONSENSUS] The CDC pattern with DynamoDB Streams + Lambda for replication follows this topology:

```
DynamoDB Table
      │
      ▼ (stream record)
DynamoDB Streams
      │
      ▼ (event source mapping)
Lambda (CDC processor)
      │
      ├──▶ OpenSearch (projection for search)
      ├──▶ SQS (asynchronous fan-out)
      └──▶ DynamoDB replica (cross-region replication)
```

[FACT] For cross-region replication, AWS offers **DynamoDB Global Tables** as a managed solution. [CONSENSUS] The manual pattern with Streams + Lambda is only preferable when you need data transformation before replication or when the destination is not DynamoDB.

---

## Practical example

### Scenario: order CDC → OpenSearch + S3 auditing

**Table:** `orders-table` with Streams enabled (NEW_AND_OLD_IMAGES)  
**Lambda 1:** CDC processor (indexes to OpenSearch and records audit)

#### CDK Python — Complete stack

```python
from aws_cdk import (
    Stack, Duration, RemovalPolicy,
    aws_dynamodb as dynamodb,
    aws_lambda as lambda_,
    aws_lambda_event_sources as event_sources,
    aws_sqs as sqs,
    aws_iam as iam,
    aws_logs as logs,
)
from constructs import Construct


class OrdersCdcStack(Stack):
    def __init__(self, scope: Construct, construct_id: str, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        # ── Table with stream enabled ─────────────────────────────────
        orders_table = dynamodb.Table(
            self, "OrdersTable",
            table_name="orders-table",
            partition_key=dynamodb.Attribute(
                name="PK", type=dynamodb.AttributeType.STRING
            ),
            sort_key=dynamodb.Attribute(
                name="SK", type=dynamodb.AttributeType.STRING
            ),
            billing_mode=dynamodb.BillingMode.PAY_PER_REQUEST,
            stream=dynamodb.StreamViewType.NEW_AND_OLD_IMAGES,
            removal_policy=RemovalPolicy.DESTROY,
        )

        # ── DLQ for unrecoverable failures ────────────────────────────
        dlq = sqs.Queue(
            self, "CdcDlq",
            queue_name="orders-cdc-dlq",
            retention_period=Duration.days(14),
        )

        # ── Lambda CDC processor ──────────────────────────────────────
        cdc_fn = lambda_.Function(
            self, "CdcProcessor",
            function_name="orders-cdc-processor",
            runtime=lambda_.Runtime.PYTHON_3_12,
            handler="handler.lambda_handler",
            code=lambda_.Code.from_asset("lambda/cdc_processor"),
            timeout=Duration.seconds(30),
            memory_size=256,
            environment={
                "OPENSEARCH_ENDPOINT": "https://your-domain.us-east-1.es.amazonaws.com",
            },
            log_retention=logs.RetentionDays.ONE_WEEK,
        )

        # ── Event Source Mapping with filtering and DLQ ───────────────
        cdc_fn.add_event_source(
            event_sources.DynamoEventSource(
                table=orders_table,
                starting_position=lambda_.StartingPosition.TRIM_HORIZON,
                batch_size=100,
                bisect_batch_on_error=True,          # isolates poison pills
                retry_attempts=3,
                on_failure=event_sources.SqsDlq(dlq),
                # Process only INSERT and MODIFY of orders
                filters=[
                    lambda_.FilterCriteria.filter({
                        "eventName": lambda_.FilterRule.is_equal("INSERT"),
                        "dynamodb": {
                            "NewImage": {
                                "PK": {"S": lambda_.FilterRule.begins_with("ORDER#")}
                            }
                        }
                    }),
                    lambda_.FilterCriteria.filter({
                        "eventName": lambda_.FilterRule.is_equal("MODIFY"),
                        "dynamodb": {
                            "NewImage": {
                                "PK": {"S": lambda_.FilterRule.begins_with("ORDER#")}
                            }
                        }
                    }),
                ],
            )
        )

        # Permissions to read from stream (already granted by DynamoEventSource)
        # Additional permission for OpenSearch
        cdc_fn.add_to_role_policy(
            iam.PolicyStatement(
                actions=["es:ESHttpPost", "es:ESHttpPut"],
                resources=["arn:aws:es:*:*:domain/orders-search/*"],
            )
        )
```

#### Lambda handler — CDC processor

```python
# lambda/cdc_processor/handler.py
import json
import os
import boto3
import urllib3
from decimal import Decimal
from typing import Any

http = urllib3.PoolManager()
OPENSEARCH_ENDPOINT = os.environ["OPENSEARCH_ENDPOINT"]


def deserialize_dynamodb_item(item: dict) -> dict:
    """Converts DynamoDB type system → native Python."""
    from boto3.dynamodb.types import TypeDeserializer
    deserializer = TypeDeserializer()
    return {k: deserializer.deserialize(v) for k, v in item.items()}


def index_order(order_data: dict, doc_id: str) -> None:
    """Indexes or updates order in OpenSearch."""
    url = f"{OPENSEARCH_ENDPOINT}/orders/_doc/{doc_id}"
    encoded = json.dumps(order_data, default=str).encode("utf-8")
    response = http.request(
        "PUT", url,
        body=encoded,
        headers={"Content-Type": "application/json"},
    )
    if response.status >= 400:
        raise RuntimeError(
            f"OpenSearch index failed: {response.status} {response.data}"
        )


def delete_order(doc_id: str) -> None:
    """Removes order from OpenSearch."""
    url = f"{OPENSEARCH_ENDPOINT}/orders/_doc/{doc_id}"
    response = http.request("DELETE", url)
    # 404 is acceptable (already removed)
    if response.status >= 400 and response.status != 404:
        raise RuntimeError(
            f"OpenSearch delete failed: {response.status}"
        )


def is_ttl_delete(record: dict) -> bool:
    """Distinguishes TTL removal from application removal."""
    user_identity = record.get("userIdentity", {})
    return (
        user_identity.get("type") == "Service"
        and user_identity.get("principalId") == "dynamodb.amazonaws.com"
    )


def lambda_handler(event: dict, context: Any) -> None:
    records = event.get("Records", [])
    print(f"Processing {len(records)} records")

    for record in records:
        event_name = record["eventName"]          # INSERT | MODIFY | REMOVE
        dynamodb_data = record["dynamodb"]
        keys = deserialize_dynamodb_item(dynamodb_data["Keys"])

        # Canonical ID in OpenSearch = PK + SK (without ambiguous separator)
        doc_id = f"{keys['PK']}_{keys.get('SK', '')}"

        if event_name in ("INSERT", "MODIFY"):
            new_image = deserialize_dynamodb_item(dynamodb_data["NewImage"])
            old_image = (
                deserialize_dynamodb_item(dynamodb_data.get("OldImage", {}))
                if "OldImage" in dynamodb_data else {}
            )

            # For MODIFY: record changed fields for auditing
            if event_name == "MODIFY" and old_image:
                changed_fields = [
                    k for k in new_image
                    if new_image.get(k) != old_image.get(k)
                ]
                print(f"MODIFY {doc_id}: fields changed = {changed_fields}")

            index_order(new_image, doc_id)

        elif event_name == "REMOVE":
            if is_ttl_delete(record):
                # TTL expiration: log and remove from index
                print(f"TTL expiry for {doc_id} — removing from search index")
            else:
                # Explicit removal by application
                print(f"Application delete for {doc_id}")

            delete_order(doc_id)

    print(f"Successfully processed {len(records)} records")
```

#### CLI — Enable stream and create event source mapping with filters

```bash
# 1. Enable stream on existing table
aws dynamodb update-table \
  --table-name orders-table \
  --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES

# 2. Get stream ARN
STREAM_ARN=$(aws dynamodb describe-table \
  --table-name orders-table \
  --query 'Table.LatestStreamArn' \
  --output text)
echo "Stream ARN: $STREAM_ARN"

# 3. Create Event Source Mapping with filtering
aws lambda create-event-source-mapping \
  --function-name orders-cdc-processor \
  --event-source-arn "$STREAM_ARN" \
  --starting-position TRIM_HORIZON \
  --batch-size 100 \
  --bisect-batch-on-function-error \
  --maximum-retry-attempts 3 \
  --destination-config '{"OnFailure":{"Destination":"arn:aws:sqs:us-east-1:123456789012:orders-cdc-dlq"}}' \
  --filter-criteria '{
    "Filters": [
      {
        "Pattern": "{\"eventName\":[\"INSERT\"],\"dynamodb\":{\"NewImage\":{\"PK\":{\"S\":[{\"prefix\":\"ORDER#\"}]}}}}"
      },
      {
        "Pattern": "{\"eventName\":[\"MODIFY\"],\"dynamodb\":{\"NewImage\":{\"PK\":{\"S\":[{\"prefix\":\"ORDER#\"}]}}}}"
      }
    ]
  }'

# 4. Check ESM status
ESM_UUID=$(aws lambda list-event-source-mappings \
  --function-name orders-cdc-processor \
  --query 'EventSourceMappings[0].UUID' \
  --output text)

aws lambda get-event-source-mapping --uuid "$ESM_UUID" \
  --query '{State: State, BatchSize: BatchSize, LastProcessingResult: LastProcessingResult}'

# 5. List active shards of the stream
aws dynamodbstreams describe-stream \
  --stream-arn "$STREAM_ARN" \
  --query 'StreamDescription.{Status:StreamStatus, Shards:Shards[*].{ShardId:ShardId,Parent:ParentShardId}}'

# 6. Monitor errors in CloudWatch
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name Errors \
  --dimensions Name=FunctionName,Value=orders-cdc-processor \
  --start-time "$(date -u -v-1H '+%Y-%m-%dT%H:%M:%SZ')" \
  --end-time "$(date -u '+%Y-%m-%dT%H:%M:%SZ')" \
  --period 300 \
  --statistics Sum

# 7. Check DLQ (messages that definitively failed)
aws sqs get-queue-attributes \
  --queue-url "https://sqs.us-east-1.amazonaws.com/123456789012/orders-cdc-dlq" \
  --attribute-names ApproximateNumberOfMessages
```

---

## Common pitfalls

**1. Immutable StreamViewType causes loss of historical data**  
[FACT] Changing `StreamViewType` requires disabling the current stream (losing unprocessed records) and creating a new one. Plan the needed type before deploying to production.

**2. Limit of 2 readers per shard**  
[FACT] Lambda consumes 1 reader per shard. If you add a second consumer (e.g., Kinesis Data Streams consumer, or second Lambda ESM), you use 2/2 readers. A third will cause throttling. Use fan-out via SNS/SQS if you need more consumers.

**3. FilterExpression on ESM filters before Lambda, but you still pay for the invoked Lambda**  
[CONSENSUS] Correct: filtering prevents Lambda *invocations* for records that don't pass the filter. But you **still pay** for stream read capacity (GetRecords). The savings are on Lambda invocations, not on stream reads.

**4. Exists does not work on intermediate nodes**  
[FACT] The `{"exists": true}` filter only works on leaf nodes. Correct example:
```json
{"dynamodb": {"NewImage": {"discount": {"N": [{"exists": true}]}}}}
```
**Incorrect** example (does not work):
```json
{"dynamodb": {"NewImage": {"discount": {"exists": true}}}}
```

**5. Reprocessing when switching StartingPosition to TRIM_HORIZON**  
[CONSENSUS] `TRIM_HORIZON` processes all records from the last 24h. On tables with high write rate, this can mean millions of records. Use `LATEST` for new consumers that don't need history, or `AT_TIMESTAMP` for a specific point.

**6. Poison pill without BisectBatchOnFunctionError**  
[CONSENSUS] A single malformed record can indefinitely block shard processing (Lambda reprocesses the batch until `MaximumRetryAttempts`). Always enable `BisectBatchOnFunctionError=True` + DLQ to isolate the problematic record and continue processing.

**7. TTL deletes arrive with up to 48h delay**  
[FACT] DynamoDB does not remove TTL-expired items instantly. Removal can occur up to 48h after expiration. The stream record will have `ApproximateCreationDateTime` reflecting *when the removal was processed*, not when the TTL expired. Do not use stream record timestamps from TTL as a reliable temporal reference.

---

## Reflection exercise

You have a `users-table` with DynamoDB Streams enabled (NEW_AND_OLD_IMAGES). The CDC processor indexes users in OpenSearch. A developer reports that some users appear in OpenSearch with outdated data — the status was updated to `PREMIUM` in the table, but OpenSearch still shows `FREE`.

Analyze the scenario and answer:

1. The CDC code correctly receives `NEW_IMAGE` on MODIFY. What would be the most likely cause of outdated data in OpenSearch?

2. The same developer suggests adding a second Lambda ESM pointing to the same stream, to keep a Redis cache updated. What risk does this introduce and how would you architect the solution correctly?

3. To reduce costs, the team wants to process only status changes to `PREMIUM`. Write the correct `FilterCriteria` JSON for this filter, knowing that the attribute is: `{"status": {"S": "PREMIUM"}}` in `NewImage`, and the event is MODIFY.

---

## Resources for further study

- [FACT] Official documentation — DynamoDB Streams: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Streams.html
- [FACT] Lambda event filtering for DynamoDB: https://docs.aws.amazon.com/lambda/latest/dg/with-ddb-filtering.html
- [FACT] Lambda — DynamoDB event source mapping: https://docs.aws.amazon.com/lambda/latest/dg/with-ddb.html
- [OPINION] AWS Blog — CDC patterns with DynamoDB Streams: https://aws.amazon.com/blogs/database/dynamodb-streams-use-cases-and-design-patterns/

---
