# Session 31 — DynamoDB: GSIs and LSIs, hot partitions and write amplification

**Estimated duration:** 60 minutes
**Prerequisites:** session-030-dynamodb-single-table-adjacency

---

## Objective

By the end, you will be able to design a GSI that solves a secondary access pattern without creating
a hot partition, calculate the cost of write amplification when writing to a table with
multiple GSIs, and decide when an LSI is preferable to a GSI (trade-off of flexibility
vs cost per item).

---

## Context

[FACT] Secondary indexes are the primary tool for supporting access patterns that
cannot be served by the table's primary key. Without them, the only alternative would be `Scan`
— a full table scan that consumes throughput proportional to the total data size,
regardless of how many items are relevant to the query. A well-designed GSI transforms a
query from O(n) to O(log n) with cost proportional only to the result.

[FACT] The cost of secondary indexes is real and measurable: each GSI replicates data (storage) and
consumes additional WCUs on each write to the base table (write amplification). Understanding these costs
is essential for designing schemas that scale without billing surprises or throttling.

---

## Main concepts

### 1. GSI — anatomy and consistency model

[FACT] A **Global Secondary Index (GSI)** is an index with PK and SK that can be completely
different from those of the base table. It is "global" because it can index items from any partition of the
base table — a query on the GSI traverses all partitions.

```
Base table: Orders
  PK = order_id (String)
  SK = customer_id (String)

GSI: OrdersByStatusDate
  PK = status (String)   ← different from base table
  SK = order_date (String)
```

**Fundamental GSI properties:**

```
1. PK and SK independent from base table (can be any top-level String/Number/Binary attribute)
2. EVENTUAL consistency by default (GSI is updated asynchronously after write to base table)
3. No GetItem — only Query and Scan are supported on GSIs
4. Default limit of 20 GSIs per table (increasable via quota request)
5. SEPARATE provisioned throughput from base table (in provisioned mode)
6. GSI inherits the table class from the base table
7. GSI key does not need to be unique — multiple items can have the same PK+SK in the GSI
8. If an item does not have the GSI key attribute, it is NOT projected into the index (sparse index)
```

[FACT] The eventual consistency of the GSI has an important practical implication: right after a write to
the base table, a query on the GSI may not return the updated item. In scenarios where immediate
read after write is critical, code should read from the base table (which supports `ConsistentRead=True`),
not from the GSI.

**Sparse index — when the absence of an attribute is a feature:**

[FACT] If the GSI key attribute does not exist on the item, the item **is not projected into the GSI**. This
allows creating indexes that cover only a subset of the table:

```
Table: Tasks
  PK = task_id
  SK = task_id
  status (optional attribute — only exists when status = "OPEN")

GSI: OpenTasksIndex
  PK = status

Items with status="OPEN" → projected into the GSI
Completed items (without status attribute) → NOT projected into the GSI

Result: the GSI contains ONLY open tasks, without storage cost for completed ones.
Query "list open tasks" → Query GSI, small, efficient.
```

This pattern is called a **sparse index** and is one of the most valuable in DynamoDB: the GSI only stores
items with a given state, reducing storage and read throughput.

### 2. Write amplification — calculating the real cost of multiple GSIs

[FACT] Each write to the base table (PutItem, UpdateItem, DeleteItem) can trigger additional writes
in each GSI. The exact number of writes per GSI depends on what changed:

```
Scenario                                          Additional writes to GSI
─────────────────────────────────────────────────────────────────────────
New item that defines the GSI key attribute       +1 write (inserts into GSI)
Update that changes the VALUE of a GSI key attr   +2 writes (deletes old + inserts new)
Update that deletes a GSI key attribute           +1 write (deletes from GSI)
Item didn't have the attribute and still doesn't +0 writes
Update of projected attribute (not GSI key)       +1 write (updates projection in GSI)
```

**Write amplification calculation with N GSIs:**

```
Table with 3 GSIs, each item has all 3 GSI key attributes defined:

PutItem → 1 write (base table) + 3 writes (1 per GSI) = 4 total writes
UpdateItem changing all 3 GSI key attributes → 1 write + 3×2 writes = 7 total writes
```

[FACT] The WCU cost of each write to the GSI is calculated by the **size of the item projected in the GSI**
(rounded up to the next KB), not by the item size in the base table. This means:
`KEYS_ONLY` projection minimizes WCU per GSI write; `ALL` projection maximizes it.

**Numerical example:**

```
Table with on-demand pricing, 3 KB item, 3 GSIs:
  - GSI1: KEYS_ONLY projection (projected item = 200 bytes → 1 WCU)
  - GSI2: INCLUDE projection (projected item = 1.5 KB → 2 WCUs)
  - GSI3: ALL projection (projected item = 3 KB → 3 WCUs)

PutItem (new item):
  Base table: 3 KB → 3 WCUs
  GSI1: 200 bytes → 1 WCU
  GSI2: 1.5 KB → 2 WCUs
  GSI3: 3 KB → 3 WCUs
  ─────────────────────────
  Total: 9 WCUs per PutItem  (3x the cost without GSIs)
```

[FACT] In **on-demand** mode, there is no explicitly provisioned WCU — DynamoDB scales
automatically. But the cost per write request unit still multiplies by the number of affected GSIs.
Check the cost at `aws.amazon.com/dynamodb/pricing` for the region and table class.

### 3. GSI throttling and back-pressure — the most dangerous cascading effect

[FACT] The back-pressure mechanism is the most counterintuitive aspect of GSI: **if a GSI does not
have enough capacity to process updates, DynamoDB throttles WRITES TO THE BASE TABLE**,
not just queries on the GSI. This means that an undersized GSI can bring down all
write capacity of your table.

```
Base table: 1,000 WCU provisioned (sufficient for the workload)
GSI1: 100 WCU provisioned (undersized)

Workload: 500 writes/s to the table, all affecting GSI1

Result:
  Base table: OK (500 WCU < 1,000)
  GSI1: THROTTLED (500 WCU > 100)
  → Back-pressure: writes to the base table also start to throttle
  → ProvisionedThroughputExceededException on base table writes
     (ResourceArn points to GSI1, not the table — guaranteed confusion)
```

[FACT] There are four distinct types of throttling in GSIs:

```
1. IndexWriteProvisionedThroughputExceeded
   Cause: GSI provisioned WCU insufficient (provisioned mode)
   Fix: increase GSI WCU

2. IndexWriteMaxOnDemandThroughputExceeded
   Cause: configured maximum throughput limit on on-demand GSI exceeded
   Fix: increase the configured max throughput or remove the limit

3. IndexWriteKeyRangeThroughputExceeded
   Cause: HOT PARTITION in the GSI — a single GSI partition exceeds the physical
          limit (~3,000 WCU/s per partition), even with sufficient total WCU
   Fix: redesign the GSI PK for better distribution (write sharding)

4. IndexWriteAccountLimitExceeded
   Cause: table exceeded the regional throughput limit of the account
   Fix: request quota increase via Service Quotas
```

### 4. Hot partition in GSI — diagnosis and solution with write sharding

[FACT] A **hot partition** in a GSI happens when the GSI PK has low cardinality (few
distinct values), concentrating all writes on a few physical partitions of the index. The classic
case is using a status attribute as GSI PK:

```
GSI: OrdersByStatus
  PK = status   → values: "OPEN", "PROCESSING", "CLOSED"

If 80% of orders have status="PROCESSING":
  80% of writes to the GSI go to the "PROCESSING" partition
  That partition may receive 4,000 writes/s → exceeds the ~3,000 WCU/s per partition limit
  → ThrottlingException with IndexWriteKeyRangeThroughputExceeded
```

**Solution: write sharding on the GSI key**

The standard technique is to add a random suffix (shard) to the GSI PK value:

```
Without sharding:     GSI PK = "PROCESSING"
With 10 shards:       GSI PK = "PROCESSING#" + str(random.randint(0, 9))
                      → values: "PROCESSING#0" to "PROCESSING#9"
```

When writing, distribute randomly among shards. When reading, make N parallel queries
(one per shard) and consolidate results on the client:

```python
import asyncio
import boto3
from boto3.dynamodb.conditions import Key

NUM_SHARDS = 10
table = boto3.resource("dynamodb").Table("orders")

# Write: choose random shard
import random
shard = random.randint(0, NUM_SHARDS - 1)
table.put_item(Item={
    "order_id": "ord123",
    "status_shard": f"PROCESSING#{shard}",  # GSI PK with shard
    "order_date": "2024-04-01",
    # ... other attributes
})

# Read: query all shards in parallel
def query_shard(shard_num: int) -> list:
    response = table.query(
        IndexName="OrdersByStatusShard",
        KeyConditionExpression=(
            Key("status_shard").eq(f"PROCESSING#{shard_num}") &
            Key("order_date").begins_with("2024-04")
        ),
    )
    return response.get("Items", [])

# Parallelism with ThreadPoolExecutor (boto3 is not async-native)
from concurrent.futures import ThreadPoolExecutor
with ThreadPoolExecutor(max_workers=NUM_SHARDS) as executor:
    futures = [executor.submit(query_shard, i) for i in range(NUM_SHARDS)]
    all_items = []
    for future in futures:
        all_items.extend(future.result())
```

[FACT] The number of shards should be calibrated: too few shards don't solve the hot partition; too many
shards increase parallel query latency (more roundtrips in parallel). A practical rule:
`num_shards = ceil(peak_writes_per_second / 800)` — reserving margin relative to the
~1,000 WCU/s per partition limit considered safe to avoid sporadic throttling.

### 5. LSI — Local Secondary Index: alternative for an alternative sort key

[FACT] A **Local Secondary Index (LSI)** maintains the same PK as the base table, but with a different
SK. It is "local" because each LSI partition is co-located with the base table partition
with the same PK value. This ensures that a query on the LSI is always **strongly consistent**
when `ConsistentRead=True`.

```
Base table: Thread
  PK = ForumName (String)
  SK = Subject (String)

LSI: LastPostIndex
  PK = ForumName (String)  ← SAME as base table — mandatory
  SK = LastPostDateTime (String)  ← different from base table
```

**LSI restrictions:**

```
1. Can ONLY be created at table creation time (CreateTable)
   Cannot be added later — unlike GSI
2. PK must be identical to the base table PK
3. SK must be a single scalar attribute (String, Number, Binary)
4. Maximum of 5 LSIs per table
5. Item collection limit: 10 GB per PK value (table + all LSIs)
   → If a partition key accumulates > 10 GB between table and LSIs: ItemCollectionSizeLimitExceededException
```

[FACT] The 10 GB per item collection limit is the most critical LSI restriction and the main reason
to prefer GSIs for entities that grow indefinitely. In contrast, the LSI shares
the base table's provisioned throughput (no additional WCU cost per index) and supports strongly
consistent reads — advantages that GSI does not offer.

**Fetching non-projected attributes in LSI:**

[FACT] A unique property of LSI (not available in GSI): if a query on the LSI needs
attributes not projected in the index, DynamoDB does an **automatic fetch** from the base table. This is
transparent to the code, but has cost: each item requiring fetch consumes additional RCUs —
calculated by the item size in the base table (rounded to 4 KB), not by the
item size in the LSI. In GSIs, this is not possible — the GSI cannot access the base table.

```python
# Query on LSI with non-projected attribute
response = table.query(
    IndexName="LastPostIndex",
    ConsistentRead=True,   # supported in LSI, not in GSI
    KeyConditionExpression=(
        Key("ForumName").eq("EC2") &
        Key("LastPostDateTime").between("2024-01-01", "2024-12-31")
    ),
    ProjectionExpression="Subject, Replies, LastPostDateTime, Tags",
    # Tags is not projected in the index → DynamoDB fetches from base table
    # Cost: LSI RCUs + fetch RCUs (per item, rounded to 4 KB)
)
```

### 6. GSI vs LSI — decision table

```
Criterion                        GSI                      LSI
─────────────────────────────────────────────────────────────────────────────
Index PK                         Any attribute            Same as base table
Index SK                         Any attribute            Any attribute
When it can be created           At any time              Only at table creation
When it can be deleted           At any time              Only when deleting the table
Read consistency                 Eventual (only)          Eventual OR Strong
Throughput                       Separate from table      Shared with table
Size limit per PK                No limit                 10 GB (table + all LSIs)
Fetch of non-projected attrs     Not supported            Supported (with extra cost)
Sparse index                     Supported                Supported
Maximum per table                20 (increasable)         5 (fixed)
Storage cost                     Own WCU + storage        Shared storage
```

**When to prefer LSI:**
- Alternative sort key access with the **same partition** (e.g., sort posts by date instead of title, within the same forum)
- Need for **strongly consistent reads** on the index
- Item collection provably small (< 5 GB) and no forecast of growth to 10 GB
- Application already exists with table created and LSI planned from the start

**When to prefer GSI:**
- Access by a different entity attribute (e.g., find orders by customer email)
- Item collection that may grow indefinitely
- Need to add the index after the table is already in production

---

## Practical example

**Scenario:** e-commerce platform with `Orders` table. Access patterns:
1. Get order by ID (table PK)
2. List orders for a customer by date (GSI: customer_id + order_date)
3. List open orders by date (sparse GSI: status + order_date, only OPEN items)
4. List orders by sales representative in the last month (GSI: rep_id + order_date)

### CDK Python — table with 3 GSIs

```python
from aws_cdk import (
    Stack,
    aws_dynamodb as dynamodb,
    RemovalPolicy,
)
from constructs import Construct

class OrdersTableStack(Stack):
    def __init__(self, scope: Construct, id: str, **kwargs):
        super().__init__(scope, id, **kwargs)

        table = dynamodb.Table(
            self, "OrdersTable",
            table_name="orders",
            partition_key=dynamodb.Attribute(
                name="order_id",
                type=dynamodb.AttributeType.STRING,
            ),
            billing_mode=dynamodb.BillingMode.PAY_PER_REQUEST,
            removal_policy=RemovalPolicy.DESTROY,
        )

        # GSI 1: orders by customer (AP2)
        # PK = customer_id, SK = order_date → good distribution (many customers)
        table.add_global_secondary_index(
            index_name="OrdersByCustomerDate",
            partition_key=dynamodb.Attribute(
                name="customer_id",
                type=dynamodb.AttributeType.STRING,
            ),
            sort_key=dynamodb.Attribute(
                name="order_date",
                type=dynamodb.AttributeType.STRING,
            ),
            projection_type=dynamodb.ProjectionType.INCLUDE,
            non_key_attributes=["status", "total_amount", "items_count"],
        )

        # GSI 2: open orders (AP3) — sparse index
        # status only exists when = "OPEN"; closed orders are not projected
        # WARNING: PK = status has low cardinality → use sharding at high volume
        table.add_global_secondary_index(
            index_name="OpenOrdersByDate",
            partition_key=dynamodb.Attribute(
                name="open_status_shard",  # "OPEN#0" to "OPEN#9"
                type=dynamodb.AttributeType.STRING,
            ),
            sort_key=dynamodb.Attribute(
                name="order_date",
                type=dynamodb.AttributeType.STRING,
            ),
            projection_type=dynamodb.ProjectionType.INCLUDE,
            non_key_attributes=["customer_id", "total_amount", "rep_id"],
        )

        # GSI 3: orders by sales rep (AP4)
        table.add_global_secondary_index(
            index_name="OrdersByRepDate",
            partition_key=dynamodb.Attribute(
                name="rep_id",
                type=dynamodb.AttributeType.STRING,
            ),
            sort_key=dynamodb.Attribute(
                name="order_date",
                type=dynamodb.AttributeType.STRING,
            ),
            projection_type=dynamodb.ProjectionType.INCLUDE,
            non_key_attributes=["customer_id", "status", "total_amount"],
        )
```

### boto3 — write with sharding on sparse GSI and amplification calculation

```python
import boto3
import random
from boto3.dynamodb.conditions import Key
from concurrent.futures import ThreadPoolExecutor
from datetime import datetime, timezone

dynamodb = boto3.resource("dynamodb", region_name="us-east-1")
table = dynamodb.Table("orders")

NUM_SHARDS = 10  # calibrate based on peak writes/s


def create_order(order_id: str, customer_id: str, rep_id: str, total: float) -> None:
    """
    Write amplification for this PutItem:
      - Base table: 1 write
      - GSI OrdersByCustomerDate: +1 write (customer_id and order_date defined)
      - GSI OpenOrdersByDate: +1 write (open_status_shard defined → order is open)
      - GSI OrdersByRepDate: +1 write (rep_id and order_date defined)
    Total: 4 writes (4x amplification)
    """
    now = datetime.now(timezone.utc).strftime("%Y-%m-%dT%H:%M:%S")
    shard = random.randint(0, NUM_SHARDS - 1)

    table.put_item(Item={
        "order_id": order_id,
        "customer_id": customer_id,
        "rep_id": rep_id,
        "order_date": now,
        "status": "OPEN",
        "open_status_shard": f"OPEN#{shard}",  # sparse GSI attribute with sharding
        "total_amount": str(total),
        "items_count": 0,
    })


def close_order(order_id: str) -> None:
    """
    Update amplification for closing an order:
      - Base table: 1 write
      - GSI OpenOrdersByDate:
          +1 write to DELETE open_status_shard from GSI (item leaves sparse index)
          → REMOVE open_status_shard from item in base table
      - GSIs OrdersByCustomerDate and OrdersByRepDate: 0 additional writes (key unchanged)
    Total: 2 writes

    Note: by removing the open_status_shard attribute, the item ceases to exist in the sparse GSI.
    """
    table.update_item(
        Key={"order_id": order_id},
        UpdateExpression="SET #s = :closed REMOVE open_status_shard",
        ExpressionAttributeNames={"#s": "status"},
        ExpressionAttributeValues={":closed": "CLOSED"},
    )


# AP3: list open orders in parallel across shards
def list_open_orders(date_prefix: str = "2024-04") -> list[dict]:
    def query_shard(shard_num: int) -> list:
        response = table.query(
            IndexName="OpenOrdersByDate",
            KeyConditionExpression=(
                Key("open_status_shard").eq(f"OPEN#{shard_num}") &
                Key("order_date").begins_with(date_prefix)
            ),
            ScanIndexForward=False,
        )
        return response.get("Items", [])

    with ThreadPoolExecutor(max_workers=NUM_SHARDS) as executor:
        results = list(executor.map(query_shard, range(NUM_SHARDS)))

    # Consolidate and sort by date (each shard returns sorted, manual merge)
    all_items = [item for shard_items in results for item in shard_items]
    return sorted(all_items, key=lambda x: x["order_date"], reverse=True)
```

### CLI — check throttling and item collection size

```bash
# Check consumed throughput per GSI (CloudWatch)
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB \
  --metric-name ConsumedWriteCapacityUnits \
  --dimensions Name=TableName,Value=orders Name=GlobalSecondaryIndexName,Value=OpenOrdersByDate \
  --start-time 2024-04-01T00:00:00Z \
  --end-time 2024-04-01T01:00:00Z \
  --period 60 \
  --statistics Sum

# Check throttling per GSI
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB \
  --metric-name WriteThrottleEvents \
  --dimensions Name=TableName,Value=orders Name=GlobalSecondaryIndexName,Value=OpenOrdersByDate \
  --start-time 2024-04-01T00:00:00Z \
  --end-time 2024-04-01T01:00:00Z \
  --period 60 \
  --statistics Sum

# Monitor item collection size (for tables with LSI)
# Use ReturnItemCollectionMetrics=SIZE on PutItem/UpdateItem
aws dynamodb put-item \
  --table-name forum-threads \
  --item '{"ForumName":{"S":"EC2"}, "Subject":{"S":"test"}, ...}' \
  --return-item-collection-metrics SIZE

# Describe GSIs of a table
aws dynamodb describe-table \
  --table-name orders \
  --query 'Table.GlobalSecondaryIndexes[*].{Name:IndexName, Status:IndexStatus, WCU:ProvisionedThroughput.WriteCapacityUnits}'
```

---

## Common pitfalls

### Pitfall 1: undersized GSI throttles the base table (silent back-pressure)

The most insidious pitfall: you provision 1,000 WCU on the base table and only 50 WCU on the GSI.
The workload increases to 200 writes/s, each affecting the GSI. The GSI gets throttled (200 > 50),
and DynamoDB starts throttling writes to the **base table** to protect GSI consistency.
The `ResourceArn` in the exception points to the GSI, but the failing code is writing to the
base table — the developer gets confused. The AWS-documented rule: the GSI provisioned WCU should
be **equal to or greater than** the base table WCU (because each base table write can generate a
write to the GSI). In on-demand mode, back-pressure still exists — the GSI has its own
configurable maximum throughput limit.

### Pitfall 2: ALL projection in GSI on table with large items

`ALL` projection on a GSI means every item from the base table is completely replicated in the index.
For a table with 10 KB items and 10 million items, a GSI with `ALL` adds ~100 GB of
storage and multiplies the cost of each write by the item size (10 KB → 10 WCUs write per
GSI, per base table write). The alternative is `INCLUDE` with only the attributes needed
for the query. If the code does `ProjectionExpression` on the GSI to fetch only 3 attributes, those
3 attributes should be projected — it makes no sense to use `ALL` and then select 3 fields.

### Pitfall 3: creating LSI after discovering the table already exists in production

LSI can only be created at `CreateTable`. If you realize 6 months after launch that
you need an alternative sort key with strongly consistent reads on the same partition, it is
not possible to add an LSI — the only option is to create a new table, migrate data, and update
the code. The available alternative is a GSI (which can be added at any time), but
GSI does not support strongly consistent reads. To avoid this situation, the correct approach is to plan
all needed LSIs **before** creating the table in production, even if the corresponding
access patterns are not yet critical. The cost of an unused LSI is only storage — much
less than a table migration.

---

## Reflection exercise

You are working with an `Events` table for a live events platform. The table
has:
- PK = `event_id`
- Attributes: `venue_id`, `start_time`, `status` (`UPCOMING`/`LIVE`/`ENDED`), `category`, `ticket_price`

The access patterns are:
- AP1: get event by ID → GetItem on base table
- AP2: list all events for a venue by date → GSI
- AP3: list all `LIVE` events now (can have 5-500 simultaneous events, peak of 2,000 writes/s to table) → sparse GSI with sharding?
- AP4: list events in a category by ascending price → GSI
- AP5: list events for a venue sorted by price → alternative to the default sort key of AP2

For AP3, calculate: with peak 2,000 writes/s, each write affecting the status GSI, how many
shards are needed to avoid hot partition (assuming 800 WCU/s per partition limit as
safe margin)? For AP5, discuss whether an LSI or a second GSI would be more appropriate, and what
the implications are of choosing LSI given that the platform may have venues with thousands of events
over the years.

---

## Resources for further study

**1. Using Global Secondary Indexes in DynamoDB**
URL: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GSI.html
What you'll find: complete GSI model (projections, eventual consistency, asynchronous
synchronization), exact WCU calculation per operation on the GSI (with scenario table), storage
considerations (100 bytes overhead per item in the index).
Why it's the right source: primary documentation with the write cost scenario table by operation
type — exactly what is needed to calculate write amplification.

**2. Understanding Global Secondary Index (GSI) write throttling and back pressure in DynamoDB**
URL: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/gsi-throttling.html
What you'll find: the 4 types of GSI throttling with specific error codes
(`IndexWriteProvisionedThroughputExceeded`, `IndexWriteKeyRangeThroughputExceeded`, etc.) and the
back-pressure mechanism explained with the example of `status` as GSI PK.
Why it's the right source: it is the only page in official documentation that explains back-pressure
explicitly — the most counterintuitive concept of GSIs.

**3. Local secondary indexes**
URL: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/LSI.html
What you'll find: creation-only-at-CreateTable limitation, 10 GB per item collection limit,
behavior of fetching non-projected attributes (with cost calculated by the
full item from the base table), `ReturnItemCollectionMetrics` to monitor collection size.
Why it's the right source: primary documentation with the exact cost calculation for fetching non-projected
attributes — one of the most overlooked nuances of LSI.

**4. General guidelines for secondary indexes in DynamoDB**
URL: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-indexes-general.html
What you'll find: practical guidelines for projection (when to use KEYS_ONLY vs INCLUDE vs
ALL), recommendations on when to create indexes vs when to use FilterExpression, and when sparse
indexes are the correct solution.
Why it's the right source: it is the synthesis of indexing best practices — more practical than the
individual GSI/LSI reference pages.

---
