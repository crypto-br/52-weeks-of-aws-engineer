# Session 035 — DynamoDB Global Tables: eventual consistency and conflict resolution

**Estimated duration:** 60 minutes  
**Dependencies:** session-034-dynamodb-transacoes-condicionais

---

## Objective

By the end of this session, you will be able to enable Global Tables v2019 in multiple regions, explain the conflict resolution mechanism (last-writer-wins based on internal timestamp), identify patterns where Global Tables adds real value (read latency by region, active-active DR) vs where it is overkill, and calculate the additional replication cost with rWRU/rWCU.

---

## Context

[FACT] DynamoDB Global Tables is a managed multi-region, multi-active replication feature: each replica accepts both reads **and** writes. When an application writes to a replica, DynamoDB automatically replicates the change to all other replicas. There is no "primary" replica — all are active.

[FACT] Two versions exist: **2019.11.21 (Current)** and 2017.11.29 (Legacy). Always use the 2019 version, which is more cost-efficient, supports more features (tables with existing data, on-demand mode) and has a simplified billing model.

---

## Main concepts

### 1. Consistency modes: MREC vs. MRSC

[FACT] Global Tables supports two consistency modes, defined at creation and **immutable** afterwards:

```
                    MREC                         MRSC
                    (Multi-Region Eventual        (Multi-Region Strong
                    Consistency) — default        Consistency)
────────────────────────────────────────────────────────────────────────
Replication         Asynchronous                 Synchronous (≥ 1 region
                                                 before confirming)
Write latency       Low                          Higher
Strongly consistent Returns local data           Always returns most recent
read                (may be stale if written     data from any region
                    in another region)
RPO                 > 0 (typically seconds)      0
TransactWriteItems  Supported (only in origin    NOT supported
                    region — not ACID cross)
TTL                 Supported                    NOT supported
LSI                 Supported                    NOT supported
Regions             Any available region         Only fixed sets:
                                                 US (N.Virginia+Ohio+Oregon),
                                                 EU (Ireland+London+Paris+Frankfurt),
                                                 AP (Tokyo+Seoul+Osaka)
Topology            Any number of replicas       Exactly 3 regions
                                                 (2 replicas + 1 witness,
                                                 or 3 replicas)
```

[FACT] The mode **cannot be changed** after creation. To switch from MREC to MRSC would require recreating the table.

---

### 2. Conflict resolution: last-writer-wins

[FACT] In MREC mode, if the same item is modified simultaneously in multiple regions, DynamoDB resolves the conflict with **last-writer-wins based on internal timestamp**:

```
Region us-east-1                    Region eu-west-1
──────────────────────              ──────────────────────
t=100ms: Write item X               t=101ms: Write item X
         {status: "A"}                       {status: "B"}
         (internal timestamp: T1)            (internal timestamp: T2)

Cross replication:
  us-east-1 receives write from eu-west-1 (T2)
  eu-west-1 receives write from us-east-1 (T1)

Resolution:
  T2 > T1 → version "B" wins in BOTH regions
  Final result: item X = {status: "B"} in both replicas

IMPORTANT: the application in us-east-1 is NOT notified that its
write was overwritten. It received "success" on the PutItem.
```

[FACT] The timestamp used is **internal to DynamoDB** — it is not a visible attribute in the application nor is it the `createdAt`/`updatedAt` you define. It is not possible to control or influence the conflict resolution result.

[CONSENSUS] For workloads where simultaneous conflicts are possible and silent loss of writes is unacceptable (e.g., financial transactions, inventory), Global Tables MREC **is not suitable**. Use MRSC or design the application to write to a single region.

---

### 3. Billing model: rWRU and rWCU

[FACT] When a table becomes part of a Global Table, write units change from WRU/WCU to **rWRU/rWCU (replicated)**, charged in **all regions containing a replica**:

```
Scenario: on-demand, 2 regions (us-east-1 + eu-west-1)
         1 KB item written in us-east-1

─────────────────────────────────────────────────────────────
Single-region table:   1 WRU (in us-east-1)
─────────────────────────────────────────────────────────────
Global table (2 regions):
  us-east-1: 1 rWRU  (where it was written)
  eu-west-1: 1 rWRU  (destination replica)
  Total: 2 rWRU
─────────────────────────────────────────────────────────────
Global table (3 regions):
  us-east-1: 1 rWRU
  eu-west-1: 1 rWRU
  ap-southeast-1: 1 rWRU
  Total: 3 rWRU
─────────────────────────────────────────────────────────────

Price: rWRU ≈ same price as WRU (us-east-1: $1.25 / million)
       → with 2 regions: write cost ~2×
       → with 3 regions: write cost ~3×

GSI updates: charged in WRU (not rWRU) in all regions
             → a write with 1 GSI in 2 regions:
             2 rWRU (base table) + 2 WRU (GSI) = 4 units total

Reads: charged normally in RRU/RCU per replica
       (no multiplier — you pay only for reads performed)

Cross-region data transfer: additional charge per GB transferred
                            between regions
```

[FACT] MRSC mode with **witness**: replication to the witness does **not** generate rWRU, storage, or data transfer costs. The witness is transparent in terms of billing.

---

### 4. Synchronized vs. non-synchronized settings

[FACT] Not all configurations are synchronized between replicas:

```
ALWAYS synchronized (change in one replica → changes all):
  - Capacity mode (provisioned ↔ on-demand)
  - Write capacity (provisioned) and write auto scaling
  - Key schema and GSI definitions
  - SSE type (KMS key type)
  - TTL (configuration, not item values)
  - Streams definition (MREC)

SYNCHRONIZED but overridable per replica:
  - Read capacity and read auto scaling (override per replica)
  - Table class
  - On-demand max read throughput

NEVER synchronized (manage manually per replica):
  - Deletion protection
  - Point-in-time recovery (PITR)
  - Tags
  - CloudWatch Contributor Insights
  - Kinesis Data Streams
  - Resource Policies
```

[CONSENSUS] Non-synchronized PITR is a common pitfall: enabling PITR on the original table does not protect replicas. You must enable PITR **on each replica individually**.

---

### 5. DAX with Global Tables: critical behavior

[FACT] Writes replicated from other regions arrive **directly to DynamoDB**, **bypassing DAX**. DAX in the destination region is not updated when a replication arrives — only when the cache TTL expires or when the local application writes through DAX.

```
Region eu-west-1:
  Application writes via DAX → DynamoDB → replicated to us-east-1

Region us-east-1:
  Replication arrives → DynamoDB updated
  DAX in us-east-1: STALE until TTL expires
  Application reads via DAX in us-east-1: may return old data
  (even after replication has arrived at DynamoDB)
```

[CONSENSUS] If you use DAX with Global Tables and data is written in multiple regions, adjust the item cache TTL to reflect acceptable staleness tolerance — or avoid DAX for those access patterns.

---

## Practical example

### Scenario: global content catalog — low read latency by region

**Use case:** streaming platform with users in North America, Europe and Asia Pacific. Content is published centrally (us-east-1), read in all regions. User writes (favorites, history) occur locally.

#### CDK Python — MREC Global Table with 3 regions

```python
from aws_cdk import (
    Stack, RemovalPolicy,
    aws_dynamodb as dynamodb,
)
from constructs import Construct


class GlobalContentTableStack(Stack):
    """
    Stack deployed in us-east-1.
    CfnGlobalTable creates replicas in specified regions automatically.
    """

    def __init__(self, scope: Construct, construct_id: str, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        global_table = dynamodb.CfnGlobalTable(
            self, "ContentGlobalTable",
            table_name="content-table",
            billing_mode="PAY_PER_REQUEST",
            attribute_definitions=[
                dynamodb.CfnGlobalTable.AttributeDefinitionProperty(
                    attribute_name="PK", attribute_type="S"
                ),
                dynamodb.CfnGlobalTable.AttributeDefinitionProperty(
                    attribute_name="SK", attribute_type="S"
                ),
                dynamodb.CfnGlobalTable.AttributeDefinitionProperty(
                    attribute_name="contentType", attribute_type="S"
                ),
                dynamodb.CfnGlobalTable.AttributeDefinitionProperty(
                    attribute_name="publishedAt", attribute_type="S"
                ),
            ],
            key_schema=[
                dynamodb.CfnGlobalTable.KeySchemaProperty(
                    attribute_name="PK", key_type="HASH"
                ),
                dynamodb.CfnGlobalTable.KeySchemaProperty(
                    attribute_name="SK", key_type="RANGE"
                ),
            ],
            global_secondary_indexes=[
                dynamodb.CfnGlobalTable.GlobalSecondaryIndexProperty(
                    index_name="ContentByTypeDate",
                    key_schema=[
                        dynamodb.CfnGlobalTable.KeySchemaProperty(
                            attribute_name="contentType", key_type="HASH"
                        ),
                        dynamodb.CfnGlobalTable.KeySchemaProperty(
                            attribute_name="publishedAt", key_type="RANGE"
                        ),
                    ],
                    projection=dynamodb.CfnGlobalTable.ProjectionProperty(
                        projection_type="INCLUDE",
                        non_key_attributes=["title", "thumbnail", "duration"],
                    ),
                )
            ],
            # Streams enabled (mandatory for MREC — managed by DynamoDB)
            stream_specification=dynamodb.CfnGlobalTable.StreamSpecificationProperty(
                stream_view_type="NEW_AND_OLD_IMAGES"
            ),
            sse_specification=dynamodb.CfnGlobalTable.SSESpecificationProperty(
                sse_enabled=True,
            ),
            time_to_live_specification=dynamodb.CfnGlobalTable.TimeToLiveSpecificationProperty(
                attribute_name="expiresAt",
                enabled=True,
            ),
            replicas=[
                dynamodb.CfnGlobalTable.ReplicaSpecificationProperty(
                    region="us-east-1",
                    point_in_time_recovery_specification=dynamodb.CfnGlobalTable.PointInTimeRecoverySpecificationProperty(
                        point_in_time_recovery_enabled=True
                    ),
                    tags=[{"key": "Environment", "value": "production"}],
                ),
                dynamodb.CfnGlobalTable.ReplicaSpecificationProperty(
                    region="eu-west-1",
                    point_in_time_recovery_specification=dynamodb.CfnGlobalTable.PointInTimeRecoverySpecificationProperty(
                        point_in_time_recovery_enabled=True
                    ),
                    tags=[{"key": "Environment", "value": "production"}],
                ),
                dynamodb.CfnGlobalTable.ReplicaSpecificationProperty(
                    region="ap-southeast-1",
                    point_in_time_recovery_specification=dynamodb.CfnGlobalTable.PointInTimeRecoverySpecificationProperty(
                        point_in_time_recovery_enabled=True
                    ),
                    tags=[{"key": "Environment", "value": "production"}],
                ),
            ],
        )
```


#### Python — Regional write and read with fallback

```python
import boto3
from botocore.exceptions import ClientError
from datetime import datetime, timezone

# Each region has its own client
REGIONS = ["us-east-1", "eu-west-1", "ap-southeast-1"]
HOME_REGION = "us-east-1"  # region where content is published

clients = {
    region: boto3.resource("dynamodb", region_name=region)
    for region in REGIONS
}

def get_local_table(region: str):
    return clients[region].Table("content-table")


def publish_content(content_id: str, title: str, content_type: str, body: str) -> dict:
    """
    Publishes content in home region (us-east-1).
    DynamoDB automatically replicates to eu-west-1 and ap-southeast-1.

    WARNING: reads in other regions after this write may be
    eventually consistent — may return stale data for some milliseconds
    to seconds depending on ReplicationLatency.
    """
    table = get_local_table(HOME_REGION)
    now = datetime.now(timezone.utc).isoformat()

    table.put_item(
        Item={
            "PK": f"CONTENT#{content_id}",
            "SK": "METADATA",
            "contentId": content_id,
            "title": title,
            "contentType": content_type,
            "body": body,
            "publishedAt": now,
            "status": "PUBLISHED",
            # Origin region — useful for filtering Stream records by region
            "originRegion": HOME_REGION,
        },
        # put-if-not-exists
        ConditionExpression="attribute_not_exists(PK)",
    )
    return {"contentId": content_id, "publishedAt": now}


def get_content(content_id: str, preferred_region: str) -> dict | None:
    """
    Reads content from the replica local to the user.
    If the local replica is unavailable, tries home region.

    For recently published content, may return stale data
    (ReplicationLatency typically < 1 second for nearby regions).
    """
    regions_to_try = [preferred_region]
    if preferred_region != HOME_REGION:
        regions_to_try.append(HOME_REGION)

    for region in regions_to_try:
        try:
            response = get_local_table(region).get_item(
                Key={"PK": f"CONTENT#{content_id}", "SK": "METADATA"},
            )
            item = response.get("Item")
            if item:
                return item
        except ClientError as e:
            if e.response["Error"]["Code"] in (
                "ProvisionedThroughputExceededException",
                "ServiceUnavailable",
            ):
                # Try next region
                continue
            raise

    return None


def record_user_favorite(
    user_id: str,
    content_id: str,
    user_region: str,
) -> None:
    """
    User favorite is written to the replica local to the user.
    No expected conflict (a user operates in one region at a time).

    In rare cases (same user in multiple simultaneous sessions
    in different regions), last-writer-wins resolves.
    """
    table = get_local_table(user_region)
    table.put_item(
        Item={
            "PK": f"USER#{user_id}",
            "SK": f"FAVORITE#{content_id}",
            "userId": user_id,
            "contentId": content_id,
            "createdAt": datetime.now(timezone.utc).isoformat(),
            "originRegion": user_region,
        }
    )
```

#### CLI — Create Global Table and monitor replication

```bash
# 1. Create single-region table first (base for Global Table)
aws dynamodb create-table \
  --table-name content-table \
  --attribute-definitions \
      AttributeName=PK,AttributeType=S \
      AttributeName=SK,AttributeType=S \
  --key-schema \
      AttributeName=PK,KeyType=HASH \
      AttributeName=SK,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST \
  --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES \
  --region us-east-1

# Wait for table to become ACTIVE
aws dynamodb wait table-exists --table-name content-table --region us-east-1

# 2. Add replica to create Global Table (v2019)
aws dynamodb update-table \
  --table-name content-table \
  --replica-updates '[
    {"Create": {"RegionName": "eu-west-1"}},
    {"Create": {"RegionName": "ap-southeast-1"}}
  ]' \
  --region us-east-1

# 3. Check replica status
aws dynamodb describe-table \
  --table-name content-table \
  --region us-east-1 \
  --query 'Table.Replicas[*].{Region:RegionName,Status:ReplicaStatus}'

# 4. Enable PITR on EACH replica individually (NOT synchronized)
for region in eu-west-1 ap-southeast-1; do
  aws dynamodb update-continuous-backups \
    --table-name content-table \
    --point-in-time-recovery-specification PointInTimeRecoveryEnabled=true \
    --region "$region"
  echo "PITR enabled in $region"
done

# 5. Monitor ReplicationLatency (MREC) between regions
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB \
  --metric-name ReplicationLatency \
  --dimensions \
      Name=TableName,Value=content-table \
      Name=ReceivingRegion,Value=eu-west-1 \
  --start-time "$(date -u -v-1H '+%Y-%m-%dT%H:%M:%SZ' 2>/dev/null || date -u -d '1 hour ago' '+%Y-%m-%dT%H:%M:%SZ')" \
  --end-time "$(date -u '+%Y-%m-%dT%H:%M:%SZ')" \
  --period 60 \
  --statistics Average,Maximum \
  --region us-east-1

# 6. Verify Global Table version (confirm it's v2019)
aws dynamodb describe-table \
  --table-name content-table \
  --region us-east-1 \
  --query 'Table.GlobalTableVersion'
# Expected result: "2019.11.21"
```

---

## Common pitfalls

**1. Silent last-writer-wins on concurrent writes**  
[FACT] The application is not notified when its write is overwritten by conflict resolution. The `PutItem` returns success, but the data may be discarded seconds later when replication arrives from another region with a more recent timestamp. Critical workloads that do not tolerate write loss should use MRSC or redirect writes to a single region.

**2. PITR is not synchronized — each replica needs to be configured**  
[FACT] Enabling PITR on the original table does not activate PITR on replicas. In an incident requiring restore, replicas without PITR cannot be restored to a previous point. Configure PITR on each replica individually and automate this via IaC.

**3. ACID transactions only within the origin region**  
[FACT] `TransactWriteItems` is atomic only in the region where it was invoked. Other replicas may see partial state transitionally during propagation. If you use Global Tables MREC and transactions, clients in other regions must be designed to tolerate this behavior or use `TransactGetItems` for atomic post-transaction reads.

**4. Streams in MREC: records may differ between replicas**  
[FACT] In MREC, the replication process may combine multiple changes into a single replicated write. Stream records from one replica may differ from another — both in content and in ordering between items. If you process Streams for CDC, add an `originRegion` attribute to the item and filter in Lambda to process only records originated in the desired region.

**5. DAX + Global Tables = stale cache from replication**  
[FACT] Writes replicated from other regions do not update the local DAX cache. An item updated in eu-west-1 may be read from DAX in us-east-1 with the old value until TTL expires. Adjust the item cache TTL to reflect staleness tolerance, or use direct DynamoDB reads for critical data.

**6. Cost multiplication with GSIs**  
[FACT] GSI writes in Global Tables are charged in WRU (not rWRU) **in each replica**. With 3 replicas and 1 GSI per table: a 1KB write generates 3 rWRU (base table) + 3 WRU (GSI) = 6 units total. With multiple GSIs the cost scales rapidly — design only the truly necessary GSIs.

---

## Reflection exercise

You are designing a user profile system for an app with users in Brazil (sa-east-1), USA (us-east-1) and Europe (eu-west-1). Each user edits their own profile; there are no cross-writes (a Brazilian user never edits a European user's profile).

Answer:

1. Given the described access pattern, what is the practical probability of last-writer-wins conflict? Does this make MREC safe for this specific use case? What type of workload would make MREC inadequate even with the same data model?

2. The team wants to ensure that after a profile write, the subsequent read in the same region returns updated data. How does the strongly consistent reads behavior in MREC affect this? Is there any situation where `ConsistentRead=True` would still return stale data?

3. Calculate the monthly rWRU cost for the following scenario: 500,000 active users, each updates their profile 2 times per day (2 KB item), 3 regions, no GSIs. Compare with the cost of a single-region table with the same write volume.

---

## Resources for further study

- [FACT] How DynamoDB global tables work (MREC vs MRSC, conflict resolution): https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/V2globaltables_HowItWorks.html
- [FACT] Understanding billing for global tables (rWRU/rWCU, GSI): https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/global-tables-billing.html
- [FACT] Best practices for global tables: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-global-table-design.html
- [FACT] Write modes with global tables: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-global-table-design.prescriptive-guidance.writemodes.html

---
