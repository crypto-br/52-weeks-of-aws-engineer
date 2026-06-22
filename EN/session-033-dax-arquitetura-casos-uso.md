# Session 033 — DAX: architecture, use cases and when NOT to use

**Estimated duration:** 60 minutes  
**Dependencies:** session-032-dynamodb-streams-lambda-cdc

---

## Objective

By the end of this session, you will be able to identify read patterns where DAX delivers real gains (repeated reads of the same item, read-heavy workloads), the cases where DAX does not help or hurts (writes, strongly consistent reads, complex queries, cold data), calculate the cost-benefit of a DAX cluster versus the cost of reads it replaces, and provision a DAX cluster via CDK Python.

---

## Context

[FACT] Amazon DynamoDB Accelerator (DAX) is a fully managed in-memory cache, compatible with the DynamoDB API, designed to reduce read latency from single-digit milliseconds to **microseconds** — without changing application code beyond swapping the client.

[CONSENSUS] DAX is not a generic cache: it understands the DynamoDB data model, distinguishes item cache from query cache, and implements write-through to maintain consistency between cache and table. This specialization is its advantage and its limit — it does not replace Redis/ElastiCache for use cases requiring advanced data structures.

---

## Main concepts

### 1. DAX cluster architecture

[FACT] A DAX cluster runs **inside a VPC**. It consists of a primary node (read + write) and up to 10 replica nodes (read only). The application uses the **DAX Client** — a drop-in replacement for the standard DynamoDB client — that points to the **cluster endpoint**.

```
Application (EC2 / Lambda in VPC)
        │
        ▼ DAX Client (endpoint: mycluster.abc123.dax-clusters.amazonaws.com)
┌───────────────────────────────────┐
│           DAX Cluster (VPC)       │
│  ┌─────────────┐  ┌────────────┐  │
│  │  Primary    │  │  Replica 1 │  │
│  │ (reads+writes)│ │ (read only)│  │
│  └──────┬──────┘  └────────────┘  │
│         │  async replication      │
│  ┌──────▼──────┐  ┌────────────┐  │
│  │  Replica 2  │  │  Replica 3 │  │
│  └─────────────┘  └────────────┘  │
└───────────────────┬───────────────┘
                    │ cache miss / write-through
                    ▼
             DynamoDB Table
```

[FACT] The DAX Client does **intelligent load balancing** between nodes — directs reads to replicas and writes to the primary. The application does not need to know which node is being used.

[FACT] DAX only supports **data** operations — not table management. `CreateTable`, `UpdateTable`, `DeleteTable`, `ListTables` etc. must go directly to the standard DynamoDB client.

---

### 2. Item cache vs. Query cache

[FACT] DAX maintains two completely independent caches:

```
┌─────────────────────────────────────────────────────────────┐
│                        DAX                                  │
│                                                             │
│  ┌──────────────────────┐   ┌───────────────────────────┐  │
│  │      Item Cache      │   │       Query Cache         │  │
│  │                      │   │                           │  │
│  │ GetItem / BatchGetItem│  │ Query / Scan              │  │
│  │                      │   │                           │  │
│  │ Key: PK (+ SK)       │   │ Key: operation parameters │  │
│  │ TTL default: 5 min   │   │ (KeyCondition,            │  │
│  │ Eviction: LRU        │   │ FilterExpr, Limit etc.)   │  │
│  │                      │   │ TTL default: 5 min        │  │
│  │                      │   │ Eviction: LRU             │  │
│  └──────────────────────┘   └───────────────────────────┘  │
│                                                             │
│  Write to item cache does NOT invalidate the query cache    │
└─────────────────────────────────────────────────────────────┘
```

[FACT] **Item cache TTL default: 5 minutes** (configurable at cluster creation — cannot be changed later without recreating the cluster).  
[FACT] **Query cache TTL default: 5 minutes** (equally configurable, independent of item cache TTL).  
[FACT] TTL = 0 for item cache: items are only removed by LRU eviction or by write-through. TTL = 0 for query cache: no Query/Scan results are stored.

**Critical implication — inconsistency between caches:**  
[FACT] If an item is updated (write-through updates the item cache), the **query cache that contains that item in a result set is NOT invalidated**. The next Query may return the old result set from the query cache until it expires by TTL.

---

### 3. Write-through: what happens on each operation

[FACT] For write operations (`PutItem`, `UpdateItem`, `DeleteItem`, `BatchWriteItem`), the flow is:

```
Application
    │
    ▼ PutItem(item)
DAX Client
    │
    ├──▶ 1. Writes to DynamoDB (synchronously)
    │         │
    │         ├── Success → 2. Updates item in item cache
    │         │                    │
    │         │                    └──▶ Returns success to application
    │         │
    │         └── Failure (throttle, etc.) → Does NOT write to cache
    │                                         Returns error to application
    │
    └── (query cache is NOT invalidated)
```

[FACT] If DynamoDB fails (including throttling), the item **is not written to the cache**. This prevents cache poisoning with data that was not persisted.

---

### 4. Behavior with strongly consistent reads and transactions

[FACT] DAX **does not serve** strongly consistent reads from cache:

```
Operation                       DAX Behavior
─────────────────────────────────────────────────────────────
GetItem (eventually consistent) Cache hit → returns from cache
GetItem (strongly consistent)   Passes directly to DynamoDB,
                                does NOT store result in cache
Query (eventually consistent)   Cache hit → returns from cache
Query (strongly consistent)     Passes directly to DynamoDB,
                                does NOT store result in cache
TransactGetItems                Passes directly to DynamoDB,
                                does NOT store result in cache
TransactWriteItems              Passes directly to DynamoDB,
                                does NOT update item cache
```

[FACT] `TransactWriteItems` does not update the DAX item cache. After a successful transaction, the item cache may contain the previous version of items until the TTL expires or a subsequent write-through occurs.

---

### 5. When to use and when NOT to use DAX

[FACT — AWS Docs] Suitable scenarios for DAX:

```
✅ Use DAX when:
─────────────────────────────────────────────────────────────
Read-heavy (10:1 or more)   High reads/writes ratio → high hit rate
Hot data                    Same items read repeatedly
Microsecond latency         SLA below 1ms for reads
Traffic bursts              DAX absorbs spikes; DynamoDB scales behind
> 3,000 RCU/item/partition  DAX breaks the per-item partition limit
Eventually consistent OK    Tolerance for slightly stale data
```

[FACT — AWS Docs] **Unsuitable** scenarios for DAX:

```
❌ Do NOT use DAX when:
─────────────────────────────────────────────────────────────
Write-heavy                 Writes use more DAX resources than reads;
                            cache benefit is minimal
Cold data / long tail       Low hit rate → cost without benefit
Strongly consistent reads   DAX passes to DynamoDB; resources wasted
TransactGetItems/WriteItems Same reason: DAX bypasses cache
Bulk scans (ETL)            Full-table scan should go directly to DynamoDB
Compliance (e.g., SOC)     DAX does not have all DynamoDB accreditations
Highly variable data        Practically all accesses are cache misses
```

[OPINION — AWS Docs] For cases that need cache but with advanced structures (lists, sorted sets, pub/sub, Lua scripting), AWS itself recommends **Amazon ElastiCache (Redis OSS)** as an alternative to DAX.

---

### 6. Node types and sizing

[FACT] DAX offers instance families dedicated to DynamoDB acceleration:

```
Family      vCPU  Memory    Typical for
──────────────────────────────────────────────────────
dax.r4.large   2    15 GB   Dev/test, light workloads
dax.r4.xlarge  4    30 GB   Small/medium production
dax.r4.4xlarge 16  122 GB   High scale
dax.r5.large   2    16 GB   Current generation — replaces r4
dax.r5.xlarge  4    32 GB   Standard production
dax.r5.4xlarge 16  128 GB   High scale current generation
dax.r5.24xlarge 96 768 GB   Extreme workloads
```

[FACT] Node memory determines how many items fit in cache. Correct sizing requires estimating: average size of hot items × number of unique items that fit in the working set.

[CONSENSUS] For high availability in production: **minimum 3 nodes** (1 primary + 2 replicas), distributed across 3 different AZs. With 1 node, any failure causes DAX downtime (the application falls back to DynamoDB, but with higher latency).

---

## Practical example

### Scenario: e-commerce product catalog — repeated reads on popular items

**Table:** `products-table` (pay-per-request)  
**Pattern:** 95% of reads on ~5% of products (popular products)  
**SLA:** p99 < 1ms for GetItem

#### CDK Python — DAX cluster + table

```python
from aws_cdk import (
    Stack, Duration, RemovalPolicy,
    aws_dynamodb as dynamodb,
    aws_dax as dax,
    aws_ec2 as ec2,
    aws_iam as iam,
)
from constructs import Construct


class DaxProductsCatalogStack(Stack):
    def __init__(self, scope: Construct, construct_id: str, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        # ── VPC (use default or create a new one) ─────────────────────
        vpc = ec2.Vpc.from_lookup(self, "VPC", is_default=True)

        # ── DynamoDB Table ────────────────────────────────────────────
        products_table = dynamodb.Table(
            self, "ProductsTable",
            table_name="products-table",
            partition_key=dynamodb.Attribute(
                name="PK", type=dynamodb.AttributeType.STRING
            ),
            sort_key=dynamodb.Attribute(
                name="SK", type=dynamodb.AttributeType.STRING
            ),
            billing_mode=dynamodb.BillingMode.PAY_PER_REQUEST,
            removal_policy=RemovalPolicy.DESTROY,
        )

        # ── IAM Role for the DAX cluster ──────────────────────────────
        dax_role = iam.Role(
            self, "DaxRole",
            assumed_by=iam.ServicePrincipal("dax.amazonaws.com"),
        )
        products_table.grant_read_write_data(dax_role)

        # ── Security Group for the DAX cluster ────────────────────────
        dax_sg = ec2.SecurityGroup(
            self, "DaxSG",
            vpc=vpc,
            description="DAX cluster SG",
            allow_all_outbound=True,
        )
        # Allow access on DAX port (8111 without TLS, 9111 with TLS)
        # from the subnet where the application runs
        dax_sg.add_ingress_rule(
            peer=ec2.Peer.ipv4(vpc.vpc_cidr_block),
            connection=ec2.Port.tcp(8111),
            description="DAX unencrypted access from VPC",
        )

        # ── Subnet group for the cluster ──────────────────────────────
        private_subnet_ids = [
            subnet.subnet_id
            for subnet in vpc.private_subnets
        ]

        dax_subnet_group = dax.CfnSubnetGroup(
            self, "DaxSubnetGroup",
            subnet_group_name="dax-products-subnet-group",
            subnet_ids=private_subnet_ids,
            description="Subnets for DAX cluster",
        )

        # ── Parameter group (custom TTLs) ─────────────────────────────
        # item cache TTL: 10 min (products change infrequently)
        # query cache TTL: 2 min (lists change more)
        dax_param_group = dax.CfnParameterGroup(
            self, "DaxParamGroup",
            parameter_group_name="dax-products-params",
            parameter_name_values={
                "record-ttl-millis":    "600000",  # 10 min
                "query-ttl-millis":     "120000",  # 2 min
            },
        )

        # ── DAX Cluster (3 nodes — multi-AZ HA) ──────────────────────
        dax_cluster = dax.CfnCluster(
            self, "DaxCluster",
            cluster_name="products-dax-cluster",
            node_type="dax.r5.large",
            replication_factor=3,          # 1 primary + 2 replicas
            iam_role_arn=dax_role.role_arn,
            subnet_group_name=dax_subnet_group.subnet_group_name,
            security_group_ids=[dax_sg.security_group_id],
            parameter_group_name=dax_param_group.parameter_group_name,
            # SSE at rest
            sse_specification=dax.CfnCluster.SSESpecificationProperty(
                sse_enabled=True,
            ),
            # Encryption in transit (TLS)
            cluster_endpoint_encryption_type="TLS",
            # Maintenance window
            preferred_maintenance_window="sun:05:00-sun:06:00",
        )
        dax_cluster.add_dependency(dax_subnet_group)
        dax_cluster.add_dependency(dax_param_group)
```

#### Python — Using the DAX Client (boto3 + amazon-dax-client)

```python
# pip install amazon-dax-client
import amazondax
import boto3
from botocore.exceptions import ClientError

# ── Initialization: DAX Client for normal reads/writes ────────────────
DAX_ENDPOINT = "daxs://products-dax-cluster.abc123.dax-clusters.amazonaws.com"

dax_client = amazondax.AmazonDaxClient.resource(
    endpoint_url=DAX_ENDPOINT,
    region_name="us-east-1",
)
dax_table = dax_client.Table("products-table")

# ── Direct DynamoDB Client (for strongly consistent + transactions) ───
dynamodb = boto3.resource("dynamodb", region_name="us-east-1")
ddb_table = dynamodb.Table("products-table")


def get_product(product_id: str) -> dict | None:
    """
    Eventual read — goes to DAX (item cache).
    p99 in microseconds after cache warm-up.
    """
    response = dax_table.get_item(
        Key={"PK": f"PRODUCT#{product_id}", "SK": "METADATA"}
    )
    return response.get("Item")


def get_product_consistent(product_id: str) -> dict | None:
    """
    Strongly consistent read — BYPASSES DAX, goes to DynamoDB.
    Use only when you need the guaranteed most recent value.
    NOTE: NOT stored in DAX cache after the read.
    """
    response = ddb_table.get_item(
        Key={"PK": f"PRODUCT#{product_id}", "SK": "METADATA"},
        ConsistentRead=True,
    )
    return response.get("Item")


def update_product_price(product_id: str, new_price: float) -> None:
    """
    Write-through: DAX writes to DynamoDB first,
    then updates the item cache.
    Query cache with this product in result sets is NOT invalidated.
    """
    dax_table.update_item(
        Key={"PK": f"PRODUCT#{product_id}", "SK": "METADATA"},
        UpdateExpression="SET price = :p, updatedAt = :ts",
        ExpressionAttributeValues={
            ":p": str(new_price),           # DynamoDB uses Decimal
            ":ts": "2024-01-15T10:00:00Z",
        },
    )


def list_products_by_category(category: str, limit: int = 20) -> list:
    """
    Query — goes to DAX query cache.
    High hit rate for popular categories.
    WARNING: result may have up to query-ttl-millis of staleness.
    """
    response = dax_table.query(
        IndexName="CategoryIndex",
        KeyConditionExpression="category = :cat",
        ExpressionAttributeValues={":cat": category},
        Limit=limit,
    )
    return response.get("Items", [])


def bulk_scan_for_etl() -> list:
    """
    Full scan for ETL — uses DynamoDB DIRECTLY.
    Does not go through DAX: avoids polluting cache with cold data
    and does not waste cluster resources.
    """
    items = []
    last_key = None
    while True:
        kwargs = {"Limit": 1000}
        if last_key:
            kwargs["ExclusiveStartKey"] = last_key
        response = ddb_table.scan(**kwargs)
        items.extend(response.get("Items", []))
        last_key = response.get("LastEvaluatedKey")
        if not last_key:
            break
    return items
```

#### CLI — Provision cluster and check metrics

```bash
# 1. Create subnet group
aws dax create-subnet-group \
  --subnet-group-name dax-products-subnet-group \
  --subnet-ids subnet-abc123 subnet-def456 subnet-ghi789

# 2. Create parameter group with custom TTLs
aws dax create-parameter-group \
  --parameter-group-name dax-products-params

aws dax update-parameter-group \
  --parameter-group-name dax-products-params \
  --parameter-name-values \
      "ParameterName=record-ttl-millis,ParameterValue=600000" \
      "ParameterName=query-ttl-millis,ParameterValue=120000"

# 3. Create cluster (3 nodes, TLS, SSE)
aws dax create-cluster \
  --cluster-name products-dax-cluster \
  --node-type dax.r5.large \
  --replication-factor 3 \
  --iam-role-arn arn:aws:iam::123456789012:role/DaxRole \
  --subnet-group-name dax-products-subnet-group \
  --security-group-ids sg-abc123 \
  --parameter-group-name dax-products-params \
  --sse-specification Enabled=true \
  --cluster-endpoint-encryption-type TLS

# 4. Check cluster status
aws dax describe-clusters \
  --cluster-names products-dax-cluster \
  --query 'Clusters[0].{Status:Status, Endpoint:ClusterDiscoveryEndpoint, Nodes:Nodes[*].{Id:NodeId,Status:NodeStatus,AZ:AvailabilityZone}}'

# 5. Monitor cache hit ratio (key DAX health metric)
aws cloudwatch get-metric-statistics \
  --namespace AWS/DAX \
  --metric-name ItemCacheHits \
  --dimensions Name=ClusterName,Value=products-dax-cluster \
  --start-time "$(date -u -v-1H '+%Y-%m-%dT%H:%M:%SZ' 2>/dev/null || date -u -d '1 hour ago' '+%Y-%m-%dT%H:%M:%SZ')" \
  --end-time "$(date -u '+%Y-%m-%dT%H:%M:%SZ')" \
  --period 300 \
  --statistics Sum

aws cloudwatch get-metric-statistics \
  --namespace AWS/DAX \
  --metric-name ItemCacheMisses \
  --dimensions Name=ClusterName,Value=products-dax-cluster \
  --start-time "$(date -u -v-1H '+%Y-%m-%dT%H:%M:%SZ' 2>/dev/null || date -u -d '1 hour ago' '+%Y-%m-%dT%H:%M:%SZ')" \
  --end-time "$(date -u '+%Y-%m-%dT%H:%M:%SZ')" \
  --period 300 \
  --statistics Sum

# 6. Calculate hit rate manually:
#    HitRate = ItemCacheHits / (ItemCacheHits + ItemCacheMisses)
#    Healthy target: > 80% for read-heavy workloads

# 7. Check throttling on the cluster
aws cloudwatch get-metric-statistics \
  --namespace AWS/DAX \
  --metric-name ThrottledRequestCount \
  --dimensions Name=ClusterName,Value=products-dax-cluster \
  --start-time "$(date -u -v-1H '+%Y-%m-%dT%H:%M:%SZ' 2>/dev/null || date -u -d '1 hour ago' '+%Y-%m-%dT%H:%M:%SZ')" \
  --end-time "$(date -u '+%Y-%m-%dT%H:%M:%SZ')" \
  --period 300 \
  --statistics Sum
```

---

## Common pitfalls

**1. Query cache is not invalidated by writes**  
[FACT] Write-through updates the **item cache** of the changed item. But if a previous Query cached a result set containing that item, that result set remains in the **query cache** until it expires by TTL. Applications requiring immediate consistency between writes and Query results should go directly to DynamoDB or use very low TTL on the query cache.

**2. TransactWriteItems does not update item cache**  
[FACT] Transactional operations pass directly to DynamoDB and do not interact with the DAX cache. After a successful `TransactWriteItems`, DAX may still serve the previous version of affected items until TTL expires. If this is unacceptable, read with strongly consistent after the transaction (directly via DynamoDB client).

**3. Using DAX Client for ETL scans**  
[CONSENSUS] Full table scans via DAX waste cluster resources (pollute the query cache with data that won't be reused) and increase latency for other requests. Always use the standard DynamoDB client for batch/ETL operations.

**4. Cluster in single AZ without replicas**  
[CONSENSUS] A cluster with `replication-factor=1` (only the primary node) offers zero HA. Any maintenance or failure makes the cluster unavailable. The DAX Client falls back to DynamoDB, but there is a window of elevated latency during recovery. In production: minimum 3 nodes across 3 AZs.

**5. Item cache TTL too high for frequently changing data**  
[CONSENSUS] The default of 5 minutes is reasonable for many cases, but data with high update rate (e.g., real-time inventory, flash sale prices) requires lower TTL or direct DynamoDB use. The staleness cost must be evaluated against the latency benefit.

**6. DAX Client does not support all DynamoDB operations**  
[FACT] Table control operations (`CreateTable`, `DescribeTable`, `UpdateTable`, etc.) are not supported by the DAX Client. The application needs to maintain two clients: DAX Client for data operations and direct DynamoDB Client for control plane operations.

---

## Reflection exercise

You are evaluating whether to add DAX to an e-commerce application with the following characteristics:

- 500,000 GetItem/s at peak (popular products: top 1,000 SKUs account for 80% of reads)  
- 10,000 UpdateItem/s (real-time inventory updates)  
- SLA: p99 < 5ms for catalog reads, p99 < 50ms for inventory reads  
- Inventory must reflect changes within 1 second (to avoid oversell)

Answer:

1. For which of the two operations (catalog vs inventory) is DAX more suitable and why? What determines this trade-off?

2. The inventory SLA requires that reads reflect changes within 1 second. Could you use DAX for these reads? If yes, how would you configure the TTL? If not, what would be the alternative?

3. Considering the DAX cluster (3x `dax.r5.xlarge`) costs approximately $0.65/hour per node (total ~$1.95/hour or ~$1,400/month), and the table without DAX would consume approximately 500,000 RCU/s (~$5,400/month in pay-per-request), what would be the minimum break-even cache hit rate for DAX to be financially justifiable?

---

## Resources for further study

- [FACT] DAX: How it works (item cache, query cache, write-through): https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DAX.concepts.html
- [FACT] Evaluating DAX suitability: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/evaluate-dax-suitability.html
- [FACT] DAX and DynamoDB consistency models: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DAX.consistency.html
- [FACT] DAX cluster sizing guide: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DAX.sizing-guide.html
- [FACT] DAX metrics and dimensions (CloudWatch): https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/dax-metrics-dimensions-dax.html

---
