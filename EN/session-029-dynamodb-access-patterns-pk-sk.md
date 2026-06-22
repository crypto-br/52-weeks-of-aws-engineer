# Session 29 — DynamoDB: access patterns first and generic PK/SK

**Estimated duration:** 60 minutes
**Prerequisites:** session-020-lambda-event-source-mappings

---

## Objective

By the end, you will be able to list all access patterns of an application before writing
any schema, design generic PK and SK to support multiple entity types
(e.g., PK = `USER#123`, SK = `PROFILE`), and implement GetItem by PK and Query by SK prefix in
code.

---

## Context

[FACT] DynamoDB is a NoSQL key-value and document database, fully managed
by AWS, that provides single-digit millisecond latencies at any scale. It was
originally described in the paper "Dynamo: Amazon's Highly Available Key-value Store" (2007) and
became one of the most used services in AWS for applications that need predictable throughput without
the operational costs of a relational database.

[FACT] The fundamental difference between modeling for SQL and modeling for DynamoDB is not technical — it's
philosophical. In SQL, you normalize data first and write queries afterwards ("queries follow
the data"). In DynamoDB, it's the reverse: you document the queries (access patterns) first and
design the schema to serve exactly those queries ("data follows the queries"). Starting with the
schema in DynamoDB almost always results in designs that require expensive Scans or multiple queries
where one would suffice.

[OPINION] Rick Houlihan (ex-AWS, principal DynamoDB evangelist) articulates this as: "If you
don't know your access patterns before you design your DynamoDB table, you're going to have a bad
time." — this position is consensus in the DynamoDB community, documented in numerous re:Invent
talks and in The DynamoDB Book by Alex DeBrie.

---

## Main concepts

### 1. The "access patterns first" paradigm — why queries before schema

In a relational database, the design process is: entities → normalization → tables → indexes.
Queries are written afterwards, and the query optimizer (query planner) decides how to execute them
efficiently using indexes, joins and dynamic execution plans.

DynamoDB has no query planner. Each read operation must specify exactly how
to reach the data: via primary key (GetItem), via partition with sort key filter (Query), or
via full table scan (Scan). Scan is the only operation that "finds" data without knowing
where it is — and it is prohibitively expensive on large tables.

[FACT] The correct process for DynamoDB is:

```
Step 1: Identify domain entities
        (e.g., User, Order, Product, Review, Session)

Step 2: Identify relationships between entities
        (e.g., User has many Orders; Order has many OrderItems)

Step 3: Document ALL necessary access patterns
        (e.g., "get user by ID", "list orders for a user",
         "get user orders by date", "get items of an order")

Step 4: ONLY THEN design PK, SK and secondary indexes
        to serve each access pattern with a single operation
```

[FACT] An access pattern documents: **who** is reading, **what** is being read, and **which
filters/sorts** are needed. A table typically has 10–30 access patterns. Each
pattern becomes a DynamoDB operation — GetItem, Query or (rarely) Scan.

**Example access pattern list for e-commerce:**

| # | Access Pattern | DynamoDB Operation |
|---|---------------|-------------------|
| AP1 | Get user by ID | GetItem |
| AP2 | List all orders for a user | Query |
| AP3 | List user orders in the last 30 days | Query (BETWEEN on SK) |
| AP4 | Get specific order by ID | GetItem |
| AP5 | List items of an order | Query (begins_with on SK) |
| AP6 | Get product by ID | GetItem |
| AP7 | List reviews for a product | Query |
| AP8 | Get specific review from a user on a product | GetItem |

This list must be **complete and stable** before starting the design. Adding a new access pattern
after creating the table usually requires creating a new secondary index (Global Secondary
Index — GSI), which has storage and write throughput costs.

### 2. Partition key and sort key — how DynamoDB stores and accesses data

[FACT] DynamoDB has two types of primary key:

```
Simple primary key:
  Partition key (PK) only
  → uniquely identifies an item
  → only supports GetItem (exact value lookup)

Composite primary key:
  Partition key (PK) + Sort key (SK)
  → PK + SK together uniquely identify an item
  → multiple items can have the same PK, with different SKs
  → supports GetItem (exact PK + SK) and Query (exact PK + condition on SK)
```

[FACT] DynamoDB uses the partition key value as input to an internal hash function to
determine which **physical partition** the item will be stored on. Items with the same PK value are
stored on the same partition, sorted by the SK value. This grouping is called an
**item collection**.

```
Physical partition for PK = "USER#123":
  ┌──────────────┬──────────────────────────────────────────┐
  │ PK           │ SK                   │ other attributes  │
  ├──────────────┼──────────────────────┼───────────────────┤
  │ USER#123     │ ORDER#2024-01-15#001  │ total: 150.00    │
  │ USER#123     │ ORDER#2024-01-20#002  │ total: 89.90     │
  │ USER#123     │ ORDER#2024-03-05#003  │ total: 230.00    │
  │ USER#123     │ PROFILE              │ name: João Silva  │
  │ USER#123     │ SESSION#abc123       │ expires: ...      │
  └──────────────┴──────────────────────┴───────────────────┘
  ↑ all items with PK="USER#123" are stored together, sorted by SK
```

[FACT] The **Query** operation allows fetching all items from a partition (fixed PK) and
optionally filtering by SK using range conditions. Since items are stored sorted
by SK, these operations are O(log n) in the number of items in the partition — efficient and predictable.

[FACT] The sort key has a type — String, Number, or Binary. For Strings, the ordering is **lexicographic
(UTF-8)**. This means `"2024-01-15"` sorts correctly as a date if in
ISO 8601 format (year-month-day), but `"15/01/2024"` (day/month/year) does not sort correctly as a date.
Using timestamps in ISO 8601 or Unix epoch format in the SK is a canonical practice.

### 3. Generic PK and SK — the pattern of entity name prefixes

[FACT] In single-table designs, it is standard practice to use generic names `PK` and `SK` for the
primary key attributes, instead of semantic names like `userId` or `orderId`. This is because
the same `PK` column will store values of completely different types (`USER#123`,
`ORDER#456`, `PRODUCT#789`) — a semantic name would be misleading.

**Entity prefixes** serve to:

```
1. Distinguish different entity types in the same table
   USER#123 vs ORDER#123  →  avoids ID collisions between types

2. Allow Query by type using begins_with
   SK begins_with "ORDER#"  →  returns only user orders

3. Document the schema intent for other developers
   PK = "USER#abc123"  →  immediately readable what this item represents

4. Prevent "wrong type" bugs
   If GetItem returns an item with PK="ORDER#456" but you expected USER,
   the prefix makes the error obvious
```

**Prefix convention:**

```
Entity      PK value          SK value
─────────────────────────────────────────────────────────────
User        USER#<userId>     PROFILE
Order       USER#<userId>     ORDER#<orderId>
OrderItem   ORDER#<orderId>   ITEM#<productId>
Product     PRODUCT#<pid>     METADATA
Review      PRODUCT#<pid>     REVIEW#<userId>
Session     USER#<userId>     SESSION#<sessionId>
```

[FACT] The `#` separator is conventional but arbitrary — you can use any character that does not
appear in the IDs. The `#` is widely used because it is easy to read and does not cause problems in URLs or
JSON.

[FACT] When PK and SK have the same value (e.g., `PK = "USER#123"`, `SK = "USER#123"`), the item is
called the **root item** of the partition. This pattern is used when you want to ensure
that a "header" item always exists for each entity, even when other items in the collection
(orders, sessions) have not yet been created.

### 4. Item collections — grouping and data locality

[FACT] An **item collection** is the set of all items that share the same
partition key value. It is the fundamental unit of "data locality" in DynamoDB — all items in
a collection reside on the same physical partition and can be retrieved with a single Query operation.

```
Item collection for PK = "USER#123":
  → PROFILE (user data)
  → ORDER#2024-01-15#001 (January order)
  → ORDER#2024-01-20#002 (January order)
  → ORDER#2024-03-05#003 (March order)
  → SESSION#abc123 (active session)

Cost of retrieving profile + all orders:
  SQL: JOIN between users and orders tables → 2 queries or 1 with JOIN
  DynamoDB: 1 Query with PK="USER#123" → 1 operation, 0.5 RCU (if < 4KB)
```

[FACT] The size limit of an item collection is **10 GB per partition key value** (only
for tables with LSI — Local Secondary Index). For tables without LSI, there is no size limit per
partition key.

[FACT] An individual item in DynamoDB has a limit of **400 KB**. Very large attributes (e.g.,
images, PDFs, blobs) should be stored in S3, with DynamoDB storing only the reference
(URL or S3 key).

### 5. GetItem and Query operations — implementation and semantics

[FACT] **GetItem** is DynamoDB's most efficient operation — O(1), always reads exactly 1 item
given PK + SK. Never does a Scan. Cost: 0.5 RCU per 4KB in eventual consistency or 1 RCU in strong
consistency.

[FACT] **Query** requires an exact PK and optionally a condition on the SK. Supports the following
operators on the sort key:

```
=           → exact value
<, <=       → less than (or equal to) a value
>, >=       → greater than (or equal to) a value
BETWEEN x AND y → value between x and y (inclusive)
begins_with(SK, "prefix") → SK starts with the prefix
```

[FACT] Query returns results **sorted by SK** in ascending order by default (lexicographic
for String, numeric for Number). For descending order, use `ScanIndexForward=False`.

[FACT] A single Query call returns at most **1 MB of data**. If the collection has more than 1 MB,
the response includes `LastEvaluatedKey` — a pagination token. To fetch all results, you need
to make paginated calls until `LastEvaluatedKey` is `null`.

**FilterExpression vs KeyConditionExpression:**

[FACT] This distinction is critical for understanding cost in DynamoDB:

```
KeyConditionExpression:
  → applied BEFORE reading the data
  → uses PK and SK to identify which items to read
  → you pay only for data that matches the key condition

FilterExpression:
  → applied AFTER reading the data (on the DynamoDB side, but after consuming RCUs)
  → filters non-key attributes
  → you PAY for data read, even the filtered-out ones
  → useful for refining results, but does NOT reduce read cost
```

Example: Query with `PK = "USER#123"` returns 100 items (500 KB). If a `FilterExpression` filters
90 of them, you paid for 500 KB but received only 50 KB of results. To avoid this, the
filter should always be on the SK (via `begins_with` or `BETWEEN`).

---

## Practical example

**Scenario:** blog platform with users, posts and comments. Access patterns identified:

```
AP1: Get user by userId
AP2: Get post by postId
AP3: List all posts from a user (most recent first)
AP4: List posts from a user in the last month
AP5: List comments on a post
AP6: Get specific comment (postId + userId)
AP7: Get profile + last 5 posts from user (transaction)
```

**Designed schema:**

```
Table: blog-single-table
Composite primary key: PK (String) + SK (String)

PK value             SK value                    Represents
──────────────────────────────────────────────────────────────────────
USER#u123            PROFILE                     User u123's profile
USER#u123            POST#2024-03-15T10:00:00    Post from 03/15 by u123
USER#u123            POST#2024-03-20T14:30:00    Post from 03/20 by u123
USER#u123            POST#2024-04-01T09:15:00    Post from 04/01 by u123
POST#p456            METADATA                    Post p456 metadata
POST#p456            COMMENT#u789                Comment from user u789
POST#p456            COMMENT#u321                Comment from user u321
```

**How each access pattern is served:**

```
AP1 (get user):
  GetItem(PK="USER#u123", SK="PROFILE")

AP2 (get post):
  GetItem(PK="POST#p456", SK="METADATA")

AP3 (list user posts, most recent first):
  Query(PK="USER#u123",
        KeyConditionExpression="PK = :pk AND begins_with(SK, :prefix)",
        ExpressionAttributeValues={":pk": "USER#u123", ":prefix": "POST#"},
        ScanIndexForward=False)   ← descending order (most recent first)

AP4 (posts from last month — April 2024):
  Query(PK="USER#u123",
        KeyConditionExpression="PK = :pk AND SK BETWEEN :start AND :end",
        ExpressionAttributeValues={
          ":pk": "USER#u123",
          ":start": "POST#2024-04-01",
          ":end": "POST#2024-04-30T23:59:59"
        })

AP5 (comments on a post):
  Query(PK="POST#p456",
        KeyConditionExpression="PK = :pk AND begins_with(SK, :prefix)",
        ExpressionAttributeValues={":pk": "POST#p456", ":prefix": "COMMENT#"})

AP6 (specific comment):
  GetItem(PK="POST#p456", SK="COMMENT#u789")
```

### Python implementation with boto3

```python
import boto3
from boto3.dynamodb.conditions import Key
from datetime import datetime, timedelta, timezone

dynamodb = boto3.resource("dynamodb", region_name="us-east-1")
table = dynamodb.Table("blog-single-table")


# AP1: Get user by ID (GetItem — O(1), no Scan)
def get_user(user_id: str) -> dict | None:
    response = table.get_item(
        Key={
            "PK": f"USER#{user_id}",
            "SK": "PROFILE",
        },
        ConsistentRead=False,  # eventual consistency (0.5 RCU) — adequate for profile reads
    )
    return response.get("Item")


# AP3: List all posts from a user, most recent to oldest
def list_user_posts(user_id: str, limit: int = 20) -> list[dict]:
    response = table.query(
        KeyConditionExpression=Key("PK").eq(f"USER#{user_id}") & Key("SK").begins_with("POST#"),
        ScanIndexForward=False,   # descending order → most recent first
        Limit=limit,
    )
    return response.get("Items", [])


# AP4: User posts within a date range
def list_user_posts_in_range(user_id: str, start: datetime, end: datetime) -> list[dict]:
    # ISO 8601 with UTC timezone for correct lexicographic ordering
    start_sk = "POST#" + start.strftime("%Y-%m-%dT%H:%M:%S")
    end_sk = "POST#" + end.strftime("%Y-%m-%dT%H:%M:%S")

    response = table.query(
        KeyConditionExpression=(
            Key("PK").eq(f"USER#{user_id}") &
            Key("SK").between(start_sk, end_sk)
        ),
        ScanIndexForward=False,
    )
    return response.get("Items", [])


# AP5: Comments on a post with pagination
def list_post_comments(post_id: str) -> list[dict]:
    all_items = []
    last_key = None

    while True:
        kwargs = {
            "KeyConditionExpression": (
                Key("PK").eq(f"POST#{post_id}") &
                Key("SK").begins_with("COMMENT#")
            ),
        }
        if last_key:
            kwargs["ExclusiveStartKey"] = last_key

        response = table.query(**kwargs)
        all_items.extend(response.get("Items", []))
        last_key = response.get("LastEvaluatedKey")

        if not last_key:
            break

    return all_items


# Create a user (PutItem)
def create_user(user_id: str, name: str, email: str) -> None:
    table.put_item(
        Item={
            "PK": f"USER#{user_id}",
            "SK": "PROFILE",
            "entity_type": "USER",         # auxiliary attribute for clarity
            "user_id": user_id,
            "name": name,
            "email": email,
            "created_at": datetime.now(timezone.utc).isoformat(),
        },
        ConditionExpression="attribute_not_exists(PK)",  # prevents overwriting existing user
    )


# Create a post (SK includes timestamp for chronological ordering)
def create_post(user_id: str, post_id: str, title: str, body: str) -> None:
    now = datetime.now(timezone.utc)
    timestamp = now.strftime("%Y-%m-%dT%H:%M:%S")

    # Store in user's partition (for AP3/AP4)
    table.put_item(
        Item={
            "PK": f"USER#{user_id}",
            "SK": f"POST#{timestamp}",
            "entity_type": "POST",
            "post_id": post_id,
            "title": title,
            "body": body,
            "created_at": now.isoformat(),
        }
    )

    # Also store in post's partition (for AP2 and AP5)
    table.put_item(
        Item={
            "PK": f"POST#{post_id}",
            "SK": "METADATA",
            "entity_type": "POST",
            "post_id": post_id,
            "user_id": user_id,
            "title": title,
            "created_at": now.isoformat(),
        }
    )
```

### CLI — basic queries

```bash
# AP1: GetItem — get user
aws dynamodb get-item \
  --table-name blog-single-table \
  --key '{"PK": {"S": "USER#u123"}, "SK": {"S": "PROFILE"}}' \
  --consistent-read

# AP3: Query — user posts, most recent first
aws dynamodb query \
  --table-name blog-single-table \
  --key-condition-expression "PK = :pk AND begins_with(SK, :prefix)" \
  --expression-attribute-values '{":pk":{"S":"USER#u123"}, ":prefix":{"S":"POST#"}}' \
  --scan-index-forward false \
  --limit 10

# AP4: Query — posts between two dates
aws dynamodb query \
  --table-name blog-single-table \
  --key-condition-expression "PK = :pk AND SK BETWEEN :start AND :end" \
  --expression-attribute-values '{
    ":pk":{"S":"USER#u123"},
    ":start":{"S":"POST#2024-04-01"},
    ":end":{"S":"POST#2024-04-30T23:59:59"}
  }'

# AP5: Query — comments on a post
aws dynamodb query \
  --table-name blog-single-table \
  --key-condition-expression "PK = :pk AND begins_with(SK, :prefix)" \
  --expression-attribute-values '{":pk":{"S":"POST#p456"}, ":prefix":{"S":"COMMENT#"}}'
```

---

## Common pitfalls

### Pitfall 1: using FilterExpression as a substitute for KeyConditionExpression

One of the most frequent confusions: developers create a schema without a discriminating SK and then
use `FilterExpression` to filter by entity type. For example: table with `PK = userId`
and an attribute `type = "ORDER"` or `type = "PROFILE"`, then query with
`FilterExpression="type = ORDER"`. DynamoDB reads **all items in the partition**, pays for all of them,
and discards those that don't pass the filter. In a partition with 10,000 items, you pay for 10,000
reads and receive 500. The fix is to use `begins_with(SK, "ORDER#")` in the `KeyConditionExpression`
— DynamoDB then only reads items with the correct SK, and you pay only for what you receive.

### Pitfall 2: timestamps in SK with inconsistent timezone

If part of the code stores timestamps in UTC and another part stores in local time, the
lexicographic ordering breaks: `"POST#2024-01-15T10:00:00-03:00"` and `"POST#2024-01-15T13:00:00Z"` are the
same instant, but lexicographically different, and will appear separated in a Query with
`BETWEEN`. The rule is: **always UTC, always ISO 8601, always with consistent precision**. Using
`datetime.now(timezone.utc).strftime("%Y-%m-%dT%H:%M:%S")` in Python ensures uniformity.
Alternatively, store Unix timestamps as Number — `Key("SK").between(1705316400, 1705320000)`
works correctly with the Number type.

### Pitfall 3: item too large from storing embedded collections in the item

The 400 KB per item limit is generous for most cases, but it's easy to hit when
storing growing arrays as attributes of an item: comments embedded in a post,
state history embedded in an order, tags embedded in a product. The correct solution is
to model each element as a **separate item** in the table (using discriminating SK), not as
a list attribute on the parent item. Separate items = unlimited growth, granular reads,
updates without rewriting the entire item. List attributes = convenient for small
and fixed lists (e.g., list of 3-5 tags), problematic for lists that grow indefinitely.

---

## Reflection exercise

You are designing a DynamoDB schema for a customer support system. The domain
entities are: Customer, Ticket (support case), Message (message within a
ticket), and Agent (support agent who handles tickets).

The access patterns identified by the team are:
- Get customer data by ID
- List all open tickets for a customer
- List all messages in a ticket, in chronological order
- Get ticket by ID (for any customer)
- List tickets assigned to a specific agent today
- Get the most recent ticket for a customer (for the service dashboard)

Describe how you would organize the table: what would be the PK and SK values for each entity
type, how the choice of timestamp in the SK solves the requirement for "list in chronological order"
and "get the most recent", and which of the access patterns above **cannot** be served just with
PK + SK (requires a secondary index) and why. Also explain why "list tickets
assigned to an agent today" is different from the other patterns in terms of modeling.

---

## Resources for further study

**1. Data Modeling foundations in DynamoDB**
URL: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/data-modeling-foundations.html
What you'll find: structured comparison between single table design and multiple table design,
with trade-offs of each approach (backup, encryption, streams, GraphQL, cost) and guidance
on when to use each.
Why it's the right source: AWS primary documentation with the current position (2024+) on single vs
multi-table, which shifted slightly from previous years when AWS was more dogmatic about
single-table.

**2. First steps for modeling relational data in DynamoDB**
URL: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-modeling-nosql.html
What you'll find: the access pattern identification process with a concrete example of
15 patterns for an order system, and the mental framework for "query first, schema last".
Why it's the right source: the access patterns table in this document is the canonical AWS
example for the DynamoDB modeling process.

**3. Best practices for using sort keys to organize data in DynamoDB**
URL: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-sort-keys.html
What you'll find: composite sort key pattern for hierarchical data
(`[country]#[region]#[state]#[city]`), version control pattern with sort key prefixes
(`v0_` for current version, `v1_`... for history).
Why it's the right source: primary documentation with the two most cited sort key patterns in
DynamoDB literature.

**4. Key condition expressions for the Query operation in DynamoDB**
URL: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Query.KeyConditionExpressions.html
What you'll find: complete list of valid operators on the sort key (`=`, `<`, `<=`, `>`,
`>=`, `BETWEEN`, `begins_with`), with CLI examples for each.
Why it's the right source: direct reference for implementing each access pattern; avoids confusion
between operators available in KeyCondition vs FilterExpression.

**5. The DynamoDB Book — Alex DeBrie**
URL: https://www.dynamodbbook.com
What you'll find: the most complete technical book on DynamoDB modeling, covering all
design patterns (adjacency list, GSI overloading, sparse indexes, write sharding) with
complete examples of real applications.
Why it's the right source: [OPINION] Alex DeBrie is considered by the community as the most
accessible and practical reference on DynamoDB — more didactic than the official documentation, and regularly
updated. Not free, but widely cited in AWS re:Invent talks.

---
