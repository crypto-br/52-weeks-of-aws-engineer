# Session 052 — Aurora: Serverless, Global Database, and Differences vs Standard RDS

**Estimated duration:** 60 min  
**Prerequisite:** session-051 (RDS Multi-AZ, Read Replicas, RDS Proxy)

---

> **Naming note (April 2026)**: AWS renamed "Aurora Serverless v2" to simply **"Aurora serverless"** in April 2026. The change is name-only — no action needed on existing deployments. This guide uses the current naming.  
> Source: [How Aurora serverless works — Aurora User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.how-it-works.html)

---

## Session Objectives

- Understand the Aurora architecture (shared cluster volume, compute/storage separation) and how it differs from RDS
- Configure Aurora serverless with min/max ACU and understand the billing model, auto-pause, and scaling
- Create an Aurora Global Database with primary and secondary cross-region clusters
- Distinguish Switchover (RPO=0) from Managed Failover (RPO=seconds) and when to use each
- List the functional differences between Aurora PostgreSQL and RDS PostgreSQL that affect migration decisions

---

## 1. Aurora Architecture — Fundamental Differences from RDS

### 1.1 Shared cluster volume

[FACT] The most important architectural difference between Aurora and RDS is the **shared cluster volume**. In RDS, each instance has its own EBS volumes. In Aurora, the writer and all readers share a single distributed storage volume:

```
┌────────────────────────────────────────────────────────────────────────┐
│ RDS PostgreSQL (traditional)                                           │
│                                                                        │
│  Writer ──────── EBS gp3 (own volume, async replication)              │
│  Read Replica ── EBS gp3 (own volume, receives WAL stream)            │
│  Standby ─────── EBS gp3 (own volume, synchronous replication)        │
│                                                                        │
│  DATA replication between volumes → write latency                      │
└────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────┐
│ Aurora                                                                 │
│                                                                        │
│  Writer ──────────┐                                                    │
│  Reader 1 ────────┤──── AURORA CLUSTER VOLUME ────────────────────────│
│  Reader 2 ────────┘         (6 copies in 3 AZs)                      │
│                                                                        │
│  Replication via redo log in storage layer (not between instances)      │
│  Readers are promoted without copying data                             │
└────────────────────────────────────────────────────────────────────────┘
```

[FACT] The Aurora cluster volume maintains **6 copies of data across 3 AZs** (2 per AZ). Aurora tolerates:
- Loss of up to 1 copy without losing write availability
- Loss of up to 3 copies without losing read availability
- Automatic repair of corrupted copies

[FACT] Consequence of the shared volume: Aurora readers have **zero-lag** regarding data durability (the data is already in the shared volume when the writer confirms). There is a small visibility replica lag (the reader needs to see the invalidated cache), but data is never lost if the reader is promoted.

### 1.2 Aurora cluster components

[FACT] An Aurora DB cluster consists of:
- **Writer (primary) DB instance**: 1 per cluster, R/W, executes DDL and DML
- **Aurora Replicas (reader DB instances)**: up to 15 per cluster, R/O, share the same cluster volume
- **Cluster endpoint** (`cluster-xxx.us-east-1.rds.amazonaws.com`): always points to the current writer
- **Reader endpoint** (`cluster-ro-xxx.us-east-1.rds.amazonaws.com`): load-balances among available readers
- **Instance endpoints**: direct endpoint for each instance (diagnostics/troubleshooting)

[FACT] Failover within the cluster (intra-region): the reader with the lowest promotion tier (0 = highest priority, 15 = lowest priority) is automatically promoted. Typical time: **~30 seconds**. The cluster endpoint is automatically updated.

---

## 2. Aurora Serverless — ACUs, Scaling, and Billing

### 2.1 Unit of measure: ACU

[FACT] The capacity unit for Aurora serverless is the **ACU (Aurora Capacity Unit)**. Each ACU represents approximately **2 GiB of memory + corresponding CPU + network**. Capacity is not tied to instance classes (db.t3, db.r6g, etc.) from provisioned Aurora.

[FACT] ACU ranges by version (as of recent supported versions):

```
╔═══════════════════════════════════════════════════════════════╗
║ Version             ║ ACU Range     ║ Auto-pause (scale-to-0) ║
╠═════════════════════╬═══════════════╬═════════════════════════╣
║ Aurora MySQL 3.02+  ║ 0.5 – 128     ║ No                      ║
║ Aurora MySQL 3.06+  ║ 0.5 – 256     ║ No                      ║
║ Aurora MySQL 3.08+  ║ 0 – 256       ║ YES                     ║
║ Aurora PG 13.6+     ║ 0.5 – 128     ║ No                      ║
║ Aurora PG 13.13+    ║ 0.5 – 256     ║ No                      ║
║ Aurora PG 13.15+    ║ 0 – 256       ║ YES                     ║
║ Aurora PG 16.3+     ║ 0 – 256       ║ YES                     ║
╚═════════════════════╩═══════════════╩═════════════════════════╝
```

[FACT] Aurora serverless **platform versions** (automatically managed by AWS):
- v1: range 0–128 ACUs, baseline
- v2: range 0–256 ACUs, baseline
- v3: range 0–256 ACUs, +30% performance vs v2
- v4: range 0–256 ACUs, +30% performance vs v3 (available in selected regions including us-east-1, us-west-2, etc.)

[FACT] New clusters are always created on the latest available platform version. Existing clusters can be updated via stop/start or blue/green deployments.

### 2.2 Billing model

[FACT] Aurora serverless is billed in **ACU-hours**, measured **per second**. There is no "instance" charge — you are charged for the capacity actually used each second.

[FACT] Cluster consumption calculation:
- Minimum consumption = `N_instances × min_ACU`
- Maximum consumption = `N_instances × max_ACU`
- Where N_instances = total number of writers + readers

[FACT] Aurora serverless storage is billed separately (same model as provisioned Aurora): GB-month of stored data + I/O requests.

### 2.3 Scaling — how and when it happens

[FACT] Aurora serverless continuously monitors CPU, memory, and network for each writer/reader. Scaling happens:
- **Scale-up**: when current capacity is insufficient for the load
- **Scale-down**: when current capacity is greater than needed
- Minimum increments: **0.5 ACUs**
- Scaling is **non-disruptive**: does not wait for a "quiet point", does not close open connections, does not interrupt in-progress transactions

[FACT] CloudWatch metrics for monitoring scaling:
- `ServerlessDatabaseCapacity`: current capacity in ACUs (float)
- `ACUUtilization`: percentage of current capacity usage

### 2.4 Auto-pause (scale to zero)

[FACT] Versions that support `minimum = 0 ACUs` allow **auto-pause**: when the cluster is idle for a configurable period, it pauses completely. Upon receiving a connection, it resumes automatically.

[FACT] Auto-pause trade-off: the first connection after a long pause can take **10–30 seconds** for the instance to return. Use only in development/test environments where cold start latency is acceptable.

### 2.5 Promotion tiers and HA

[FACT] Readers in **promotion tiers 0 and 1**: the minimum capacity is tied to the writer's current capacity. If the writer is at 8 ACUs, the reader in tier 0/1 will also have a minimum of 8 ACUs. Ensures the reader is always ready to failover without needing to scale first.

[FACT] Readers in **promotion tiers 2–15**: scale independently. They can be at 1 ACU while the writer is at 32 ACUs. Suitable for reporting or analytics workloads that don't need to be primary failover targets.

### 2.6 max_connections in Aurora serverless

[FACT] `max_connections` in Aurora serverless is automatically set by Aurora based on the **configured max ACU**, not the current ACU. This prevents connections from being dropped when capacity scales down. For Aurora PostgreSQL serverless:
```
max_connections = LEAST({max_acus × 1000}, 5000)
```
(The exact formula varies by engine; the principle is based on max ACU.)

### 2.7 Mixed-configuration clusters

[FACT] An Aurora cluster can mix serverless and provisioned instances:
- Provisioned writer + serverless readers (e.g.: stable write workload, variable reads)
- Serverless writer + provisioned readers (e.g.: variable writes, stable report reads)
- All serverless (most common configuration for new projects)

---

## 3. Aurora Global Database

### 3.1 Architecture

[FACT] Aurora Global Database is an Aurora feature for **cross-region** replication:
- **1 primary cluster** (R/W) in one region
- **Up to 5 secondary clusters** (R/O) in different regions
- Replication: storage-based, asynchronous, **typical lag < 1 second**
- Up to **16 replicas per secondary region**

```
┌────────────────────────────────────────────────────────────────────┐
│                    AURORA GLOBAL DATABASE                          │
│                                                                    │
│  us-east-1 (PRIMARY)                us-west-2 (SECONDARY)         │
│  ┌──────────────────────┐           ┌──────────────────────┐       │
│  │ Writer (R/W)         │           │ Reader(s) — R/O       │       │
│  │ Reader 1             │──────────▶│ (replicas of primary) │       │
│  │ Reader 2             │ storage   └──────────────────────┘       │
│  └──────────────────────┘   repl     lag < 1s typically           │
│                                                                    │
│  ┌──────────────────────┐                                          │
│  │ sa-east-1 (SECONDARY)│ ← up to 5 secondary regions            │
│  │ Reader(s) — R/O      │                                          │
│  └──────────────────────┘                                          │
│                                                                    │
│  ENDPOINTS:                                                        │
│  Global writer endpoint → cluster-xxx.global.rds.amazonaws.com    │
│  Secondary reader     → cluster-ro-xxx.us-west-2.rds.amazonaws.com│
└────────────────────────────────────────────────────────────────────┘
```

[FACT] The **global writer endpoint** is a single DNS endpoint that always points to the current primary cluster's writer. During a switchover/failover, the global writer endpoint is automatically updated. Using this endpoint in applications eliminates the need to change connection strings after failover.

[FACT] CloudWatch metrics for monitoring Global Database:
- `AuroraGlobalDBRPOLag` (ms): replication lag measured as RPO — available for Aurora PostgreSQL and Aurora MySQL ≥ 3.04.0 (metric *per secondary cluster*)
- `AuroraGlobalDBReplicationLag` (ms): for older MySQL versions

### 3.2 Switchover (RPO = 0) — planned failover

[FACT] **Switchover** (formerly called "managed planned failover") is used in controlled scenarios:
- Planned operational maintenance
- Regional rotation (e.g.: financial regulation requiring DR exercise)
- "Follow-the-sun": move the writer to the region with active users
- Fail-back after an unplanned failover

[FACT] The switchover **waits for the target secondary cluster to be fully synchronized** with the primary before switching roles. Therefore, **RPO = 0 (no data loss)**. The database is unavailable for a short period during the role swap.

[FACT] Requirement: primary and secondary clusters must have the same major and minor version (patch levels may differ depending on the version).

[FACT] CLI for switchover:
```bash
aws rds --region <primary-region> \
  switchover-global-cluster \
  --global-cluster-identifier <global-db-id> \
  --target-db-cluster-identifier <arn-of-secondary-to-promote>
```

### 3.3 Managed Failover (RPO = seconds) — unplanned failover

[FACT] **Managed Failover** is used for disaster recovery (unplanned regional outage):
- **Does not wait** for synchronization with the primary
- **RPO ≠ 0**: partially replicated data may be lost
- The amount of loss depends on `AuroraGlobalDBRPOLag` at the time of failure

[FACT] The managed failover is "managed" because:
1. The promoted secondary assumes the primary role
2. When the old region recovers, Aurora **automatically re-adds it as secondary**
3. Aurora attempts to take a snapshot of the old storage: `rds:unplanned-global-failover-<name>-<timestamp>` (available in the console for recovery of lost data)
4. "Write fencing": Aurora attempts to block writes in the old region to avoid split-brain (best-effort)

[FACT] CLI for managed failover (unplanned):
```bash
aws rds --region <secondary-region> \
  failover-global-cluster \
  --global-cluster-identifier <global-db-id> \
  --target-db-cluster-identifier <arn-of-secondary-to-promote> \
  --allow-data-loss   # mandatory flag — acknowledges potential data loss
```

### 3.4 RPO management (Aurora PostgreSQL)

[FACT] For Aurora PostgreSQL, the parameter `rds.global_db_rpo` allows defining a maximum RPO in seconds. When active, if **all** secondary clusters exceed the configured lag, the primary **blocks commits** until at least one secondary returns to the target. Valid range: 20s to 2,147,483,647s.

[FACT] `rds.global_db_rpo` is not recommended in setups with only 2 regions: a failure of the secondary region would cause blocking of all writes in the primary.

---

## 4. Aurora PostgreSQL vs RDS PostgreSQL — Differences for Migration Decisions

[FACT] Technical comparison:

```
╔══════════════════════════════════╦═════════════════════════╦═════════════════════════╗
║ Aspect                           ║ RDS PostgreSQL          ║ Aurora PostgreSQL        ║
╠══════════════════════════════════╬═════════════════════════╬═════════════════════════╣
║ Storage                          ║ EBS per instance        ║ Cluster volume          ║
║                                  ║ (sync replication)      ║ (6 copies, 3 AZs)       ║
╠══════════════════════════════════╬═════════════════════════╬═════════════════════════╣
║ Max readers                      ║ Up to 5 read replicas   ║ Up to 15 Aurora Replicas║
╠══════════════════════════════════╬═════════════════════════╬═════════════════════════╣
║ Intra-region failover            ║ ~35s (DNS propagation)  ║ ~30s                    ║
╠══════════════════════════════════╬═════════════════════════╬═════════════════════════╣
║ Replica lag (readers)            ║ Async — seconds/minutes ║ Near zero (shared vol)  ║
╠══════════════════════════════════╬═════════════════════════╬═════════════════════════╣
║ Cross-region DR                  ║ Cross-region read       ║ Global Database         ║
║                                  ║ replica (manual promo.) ║ (switchover/failover)   ║
╠══════════════════════════════════╬═════════════════════════╬═════════════════════════╣
║ Serverless                       ║ NO                      ║ YES (ACU-based)         ║
╠══════════════════════════════════╬═════════════════════════╬═════════════════════════╣
║ Logical replication slots        ║ Fully supported         ║ Supported, but slots    ║
║                                  ║ (include standby)       ║ need to be recreated    ║
║                                  ║                         ║ after switchover/failover║
╠══════════════════════════════════╬═════════════════════════╬═════════════════════════╣
║ file_fdw                         ║ Supported               ║ NOT supported           ║
║                                  ║                         ║ (no direct FS access)   ║
╠══════════════════════════════════╬═════════════════════════╬═════════════════════════╣
║ Parameter groups                 ║ Instance level only     ║ Cluster level +         ║
║                                  ║                         ║ instance level          ║
╠══════════════════════════════════╬═════════════════════════╬═════════════════════════╣
║ max_connections default          ║ LEAST(mem/9531392, 5000)║ Based on max ACU        ║
║                                  ║                         ║ (serverless) or class   ║
╠══════════════════════════════════╬═════════════════════════╬═════════════════════════╣
║ Aurora-specific functions        ║ No                      ║ aurora_version(),       ║
║                                  ║                         ║ aurora_replica_status() ║
╠══════════════════════════════════╬═════════════════════════╬═════════════════════════╣
║ Storage cost                     ║ GB-month (EBS gp3/io2)  ║ GB-month (Aurora I/O    ║
║                                  ║                         ║ Standard or Optimized)  ║
╠══════════════════════════════════╬═════════════════════════╬═════════════════════════╣
║ Cloning                          ║ No (use snapshot)       ║ YES — Aurora Fast Clone ║
║                                  ║                         ║ (copy-on-write, instant)║
╚══════════════════════════════════╩═════════════════════════╩═════════════════════════╝
```

[FACT] **Replication slots and failover**: in Aurora PostgreSQL, logical replication slots exist only on the writer. After any switchover or failover, slots need to be recreated on the new writer. If there are consumers (e.g.: Debezium/CDC tools) using logical replication slots, they must be paused during failover and reconnected to the new writer with recreated slots.

[FACT] **file_fdw**: `file_fdw` reads files directly from the server filesystem. Since Aurora does not expose the filesystem directly, this FDW does not work. `postgres_fdw` (connects to another PostgreSQL) works normally.

[FACT] **Aurora Storage I/O**: Aurora offers two storage modes:
- **Aurora I/O Standard**: charges for I/O requests separately
- **Aurora I/O Optimized**: higher price per GB-month, but no separate I/O charges — better for I/O intensive workloads (>25% of storage cost in I/O)

---

## 5. CDK Python — Aurora Serverless + Global Database

```python
"""
CDK Stacks for Aurora serverless (PostgreSQL) with Global Database.
Stack 1 (primary): creates the serverless cluster in the primary region
Stack 2 (secondary): adds a secondary cluster in another region
"""
from aws_cdk import (
    Stack, Duration, RemovalPolicy, CfnOutput,
    aws_rds as rds,
    aws_ec2 as ec2,
    aws_secretsmanager as sm,
    aws_cloudwatch as cw,
    aws_cloudwatch_actions as cwa,
    aws_sns as sns,
)
from constructs import Construct


class AuroraServerlessPrimaryStack(Stack):
    """Aurora serverless PostgreSQL cluster in the primary region."""

    def __init__(self, scope: Construct, construct_id: str, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        # ──────────────────────────────────────────────────────────────
        # VPC and security group
        # ──────────────────────────────────────────────────────────────
        vpc = ec2.Vpc(self, "Vpc", max_azs=3, nat_gateways=1)
        db_sg = ec2.SecurityGroup(self, "DbSg", vpc=vpc)
        db_sg.add_ingress_rule(ec2.Peer.ipv4(vpc.vpc_cidr_block), ec2.Port.tcp(5432))

        # ──────────────────────────────────────────────────────────────
        # Credentials
        # ──────────────────────────────────────────────────────────────
        db_secret = sm.Secret(self, "AuroraSecret",
            generate_secret_string=sm.SecretStringGenerator(
                secret_string_template='{"username": "pgadmin"}',
                generate_string_key="password",
                exclude_punctuation=True,
            ),
        )

        # ──────────────────────────────────────────────────────────────
        # Aurora Serverless PostgreSQL cluster
        # ──────────────────────────────────────────────────────────────
        # Serverless scaling config: min 0 ACUs (auto-pause) → max 16 ACUs
        scaling = rds.ServerlessV2ScalingConfiguration(
            min_capacity=0,    # 0 = auto-pause enabled (Aurora PG 16.3+)
            max_capacity=16,   # 16 ACUs ≈ 32 GiB RAM max
        )

        # PostgreSQL 16 cluster
        self.cluster = rds.DatabaseCluster(self, "AuroraCluster",
            engine=rds.DatabaseClusterEngine.aurora_postgres(
                version=rds.AuroraPostgresEngineVersion.VER_16_3,
            ),
            vpc=vpc,
            vpc_subnets=ec2.SubnetSelection(
                subnet_type=ec2.SubnetType.PRIVATE_WITH_EGRESS
            ),
            security_groups=[db_sg],
            credentials=rds.Credentials.from_secret(db_secret),
            serverless_v2_min_capacity=scaling.min_capacity,
            serverless_v2_max_capacity=scaling.max_capacity,
            # Serverless writer (tier 0: priority failover target)
            writer=rds.ClusterInstance.serverless_v2("Writer",
                scale_with_writer=True,   # reader follows writer capacity
            ),
            # Serverless reader in tier 0 (min capacity = writer capacity)
            readers=[
                rds.ClusterInstance.serverless_v2("Reader1",
                    scale_with_writer=True,    # tier 0/1: scales with writer
                    promotion_tier=1,
                ),
                # Reader for reports: scales independently (tier 2)
                rds.ClusterInstance.serverless_v2("ReaderReports",
                    scale_with_writer=False,   # tier 2-15: independent
                    promotion_tier=2,
                ),
            ],
            # Cluster parameter group
            parameter_group=rds.ParameterGroup(self, "ClusterParams",
                engine=rds.DatabaseClusterEngine.aurora_postgres(
                    version=rds.AuroraPostgresEngineVersion.VER_16_3
                ),
                parameters={
                    "log_min_duration_statement": "1000",
                    "log_lock_waits": "on",
                    "shared_preload_libraries": "pg_stat_statements",
                },
            ),
            # Storage
            storage_encrypted=True,
            storage_type=rds.DBClusterStorageType.AURORA_IOPT1,  # I/O Optimized
            # Backups
            backup=rds.BackupProps(retention=Duration.days(7)),
            # Performance Insights
            enable_performance_insights=True,
            # Deletion
            deletion_protection=True,
            removal_policy=RemovalPolicy.RETAIN,
        )

        # ──────────────────────────────────────────────────────────────
        # Global Database — associates this cluster as primary
        # ──────────────────────────────────────────────────────────────
        global_cluster = rds.CfnGlobalCluster(self, "GlobalCluster",
            global_cluster_identifier="checkout-global",
            source_db_cluster_identifier=self.cluster.cluster_arn,
            deletion_protection=False,
        )

        # ──────────────────────────────────────────────────────────────
        # CloudWatch Alarms
        # ──────────────────────────────────────────────────────────────
        alert_topic = sns.Topic(self, "AuroraAlerts")

        # ACU Utilization > 80% → consider increasing max ACU
        cw.Alarm(self, "HighACUUtilization",
            metric=cw.Metric(
                namespace="AWS/RDS",
                metric_name="ACUUtilization",
                dimensions_map={"DBClusterIdentifier": self.cluster.cluster_identifier},
                period=Duration.minutes(5),
                statistic="Average",
            ),
            threshold=80,
            evaluation_periods=3,
            alarm_description="Aurora serverless ACU utilization > 80% — consider increasing max ACU",
        ).add_alarm_action(cwa.SnsAction(alert_topic))

        # Global DB replication lag > 5 seconds
        cw.Alarm(self, "GlobalDbLagAlarm",
            metric=cw.Metric(
                namespace="AWS/RDS",
                metric_name="AuroraGlobalDBRPOLag",
                # Monitor on secondary cluster, not primary
                dimensions_map={"DBClusterIdentifier": self.cluster.cluster_identifier},
                period=Duration.minutes(1),
                statistic="Maximum",
            ),
            threshold=5000,   # 5 seconds in milliseconds
            evaluation_periods=3,
            alarm_description="Global DB replication lag > 5s — check cross-region connectivity",
        ).add_alarm_action(cwa.SnsAction(alert_topic))

        CfnOutput(self, "ClusterEndpoint", value=self.cluster.cluster_endpoint.hostname)
        CfnOutput(self, "ReaderEndpoint",  value=self.cluster.cluster_read_endpoint.hostname)
        CfnOutput(self, "ClusterArn",      value=self.cluster.cluster_arn)
        CfnOutput(self, "SecretArn",       value=db_secret.secret_arn)


class AuroraGlobalSecondaryStack(Stack):
    """Aurora serverless cluster in the secondary region (joined to Global Database)."""

    def __init__(self, scope: Construct, construct_id: str,
                 global_cluster_id: str, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        vpc = ec2.Vpc(self, "Vpc", max_azs=3, nat_gateways=1)
        db_sg = ec2.SecurityGroup(self, "DbSg", vpc=vpc)
        db_sg.add_ingress_rule(ec2.Peer.ipv4(vpc.vpc_cidr_block), ec2.Port.tcp(5432))

        # Secondary cluster: replica of the global database
        # Does not need its own credentials — inherits from primary
        secondary = rds.CfnDBCluster(self, "SecondaryCluster",
            engine="aurora-postgresql",
            engine_version="16.3",
            db_cluster_identifier="checkout-secondary",
            global_cluster_identifier=global_cluster_id,
            db_subnet_group_name=rds.SubnetGroup(self, "SubnetGroup",
                vpc=vpc,
                vpc_subnets=ec2.SubnetSelection(
                    subnet_type=ec2.SubnetType.PRIVATE_WITH_EGRESS
                ),
                description="Aurora secondary subnet group",
            ).subnet_group_name,
            vpc_security_group_ids=[db_sg.security_group_id],
            storage_encrypted=True,
            # Serverless scaling for the secondary region
            serverless_v2_scaling_configuration=rds.CfnDBCluster.ServerlessV2ScalingConfigurationProperty(
                min_capacity=0.5,   # don't use auto-pause on secondary (resume latency)
                max_capacity=16,
            ),
        )

        # Add 1 serverless reader to the secondary cluster
        rds.CfnDBInstance(self, "SecondaryReader",
            db_instance_class="db.serverless",
            db_cluster_identifier=secondary.ref,
            engine="aurora-postgresql",
            db_instance_identifier="checkout-secondary-reader-1",
        )

        CfnOutput(self, "SecondaryClusterArn", value=secondary.attr_db_cluster_arn)
```

---

## 6. Python — Global Database Monitoring and Capacity Detection

```python
"""
Utilities for Aurora serverless and Global Database monitoring.
"""
import boto3
import psycopg2
from datetime import datetime, timezone, timedelta
import time


# ──────────────────────────────────────────────────────────────────────
# 1. ACU Monitor — detects if the cluster is near its limit
# ──────────────────────────────────────────────────────────────────────

def get_acu_stats(cluster_id: str, lookback_minutes: int = 60,
                  region: str = "us-east-1") -> dict:
    """
    Returns ACU statistics for the serverless cluster.
    Useful for deciding if max_capacity needs to be increased.
    """
    cw = boto3.client("cloudwatch", region_name=region)
    end_time   = datetime.now(timezone.utc)
    start_time = end_time - timedelta(minutes=lookback_minutes)

    def get_metric(metric_name: str, stat: str) -> float | None:
        resp = cw.get_metric_statistics(
            Namespace="AWS/RDS",
            MetricName=metric_name,
            Dimensions=[{"Name": "DBClusterIdentifier", "Value": cluster_id}],
            StartTime=start_time,
            EndTime=end_time,
            Period=300,
            Statistics=[stat],
        )
        dps = sorted(resp["Datapoints"], key=lambda x: x["Timestamp"])
        return dps[-1][stat] if dps else None

    acu_capacity   = get_metric("ServerlessDatabaseCapacity", "Average")
    acu_util_avg   = get_metric("ACUUtilization", "Average")
    acu_util_max   = get_metric("ACUUtilization", "Maximum")
    db_connections = get_metric("DatabaseConnections", "Average")

    result = {
        "cluster_id": cluster_id,
        "period_minutes": lookback_minutes,
        "avg_capacity_acu": round(acu_capacity, 2) if acu_capacity else None,
        "avg_utilization_pct": round(acu_util_avg, 1) if acu_util_avg else None,
        "max_utilization_pct": round(acu_util_max, 1) if acu_util_max else None,
        "avg_connections": round(db_connections, 0) if db_connections else None,
    }

    # Simple diagnostics
    if acu_util_max and acu_util_max > 90:
        result["recommendation"] = (
            f"WARNING: max ACU utilization = {acu_util_max:.1f}%. "
            "Consider increasing max_capacity to avoid throttling."
        )
    elif acu_util_avg and acu_util_avg < 20:
        result["recommendation"] = (
            f"Low average utilization ({acu_util_avg:.1f}%). "
            "Consider reducing max_capacity to save costs."
        )
    else:
        result["recommendation"] = "Capacity within adequate ranges."

    return result


# ──────────────────────────────────────────────────────────────────────
# 2. Global Database Monitor — lag and replication status
# ──────────────────────────────────────────────────────────────────────

def get_global_db_replication_status(
    global_cluster_id: str,
    primary_region: str = "us-east-1",
    secondary_regions: list[str] | None = None,
) -> dict:
    """
    Checks replication lag of all secondary regions.
    Uses the AuroraGlobalDBRPOLag metric (in ms).
    """
    secondary_regions = secondary_regions or ["us-west-2", "sa-east-1"]

    rds_client = boto3.client("rds", region_name=primary_region)

    # Get global cluster information
    response = rds_client.describe_global_clusters(
        GlobalClusterIdentifier=global_cluster_id
    )
    global_cluster = response["GlobalClusters"][0]

    result = {
        "global_cluster_id": global_cluster_id,
        "status": global_cluster["Status"],
        "engine": global_cluster["Engine"],
        "engine_version": global_cluster["EngineVersion"],
        "members": [],
    }

    for member in global_cluster.get("GlobalClusterMembers", []):
        is_writer = member.get("IsWriter", False)
        cluster_arn = member["DBClusterArn"]
        region = cluster_arn.split(":")[3]

        member_info = {
            "region": region,
            "cluster_arn": cluster_arn,
            "is_primary": is_writer,
            "rpo_lag_ms": None,
        }

        # Collect lag only for secondaries
        if not is_writer:
            cw = boto3.client("cloudwatch", region_name=region)
            cluster_id = cluster_arn.split(":")[-1]
            end_time = datetime.now(timezone.utc)
            resp = cw.get_metric_statistics(
                Namespace="AWS/RDS",
                MetricName="AuroraGlobalDBRPOLag",
                Dimensions=[{"Name": "DBClusterIdentifier", "Value": cluster_id}],
                StartTime=end_time - timedelta(minutes=5),
                EndTime=end_time,
                Period=60,
                Statistics=["Maximum"],
            )
            dps = sorted(resp["Datapoints"], key=lambda x: x["Timestamp"])
            if dps:
                member_info["rpo_lag_ms"] = round(dps[-1]["Maximum"], 0)
                member_info["lag_status"] = (
                    "OK" if dps[-1]["Maximum"] < 1000 else
                    "WARNING" if dps[-1]["Maximum"] < 5000 else
                    "CRITICAL"
                )

        result["members"].append(member_info)

    return result


# ──────────────────────────────────────────────────────────────────────
# 3. Aurora vs RDS verification via SQL
# ──────────────────────────────────────────────────────────────────────

def check_aurora_specifics(conn_string: str) -> dict:
    """
    Connects to Aurora PostgreSQL and checks specific functions/views.
    Useful for post-migration validation from RDS to Aurora.
    """
    conn = psycopg2.connect(conn_string)
    conn.autocommit = True
    cur = conn.cursor()
    results = {}

    # aurora_version() — returns Aurora engine version
    try:
        cur.execute("SELECT aurora_version()")
        results["aurora_version"] = cur.fetchone()[0]
    except Exception as e:
        results["aurora_version"] = f"ERROR: {e}"

    # aurora_replica_status() — reader lag (execute on writer)
    try:
        cur.execute("""
            SELECT server_id, session_id, durable_lsn,
                   highest_lsn_rcvd, current_read_lsn,
                   feedback_epoch, feedback_xmin, replica_lag_in_msec
            FROM aurora_replica_status()
            WHERE session_id != 'MASTER_SESSION_ID'
            ORDER BY replica_lag_in_msec DESC
            LIMIT 5
        """)
        cols = [desc[0] for desc in cur.description]
        results["replica_status"] = [dict(zip(cols, row)) for row in cur.fetchall()]
    except Exception as e:
        results["replica_status"] = f"ERROR: {e}"

    # Check configured max_connections
    cur.execute("SHOW max_connections")
    results["max_connections"] = int(cur.fetchone()[0])

    # Check logical replication slots (attention post-switchover)
    cur.execute("""
        SELECT slot_name, plugin, active, confirmed_flush_lsn
        FROM pg_replication_slots
        WHERE slot_type = 'logical'
    """)
    cols = [desc[0] for desc in cur.description]
    results["logical_slots"] = [dict(zip(cols, row)) for row in cur.fetchall()]

    cur.close()
    conn.close()
    return results


if __name__ == "__main__":
    import json

    # Check ACU stats
    stats = get_acu_stats("checkout-aurora", lookback_minutes=60)
    print("ACU Stats:", json.dumps(stats, indent=2, default=str))

    # Check global replication
    global_status = get_global_db_replication_status(
        global_cluster_id="checkout-global",
        primary_region="us-east-1",
    )
    print("Global DB Status:", json.dumps(global_status, indent=2, default=str))
```

---

## 7. CLI — Aurora Serverless and Global Database Operations

```bash
# ═══════════════════════════════════════════════════════════════
# Aurora Serverless — creation and configuration
# ═══════════════════════════════════════════════════════════════

export CLUSTER_ID="checkout-aurora"
export GLOBAL_ID="checkout-global"
export REGION_PRIMARY="us-east-1"
export REGION_SECONDARY="us-west-2"

# Create Aurora serverless PostgreSQL 16 cluster
aws rds create-db-cluster \
  --db-cluster-identifier "$CLUSTER_ID" \
  --engine aurora-postgresql \
  --engine-version "16.3" \
  --master-username pgadmin \
  --manage-master-user-password \
  --serverless-v2-scaling-configuration MinCapacity=0,MaxCapacity=16 \
  --enable-performance-insights \
  --storage-encrypted \
  --storage-type aurora-iopt1 \
  --backup-retention-period 7 \
  --region "$REGION_PRIMARY"

# Add serverless writer (required — cluster without instance doesn't accept connections)
aws rds create-db-instance \
  --db-instance-identifier "${CLUSTER_ID}-writer" \
  --db-cluster-identifier "$CLUSTER_ID" \
  --db-instance-class db.serverless \
  --engine aurora-postgresql \
  --region "$REGION_PRIMARY"

# Add serverless reader (tier 1 = priority failover)
aws rds create-db-instance \
  --db-instance-identifier "${CLUSTER_ID}-reader-1" \
  --db-cluster-identifier "$CLUSTER_ID" \
  --db-instance-class db.serverless \
  --engine aurora-postgresql \
  --promotion-tier 1 \
  --region "$REGION_PRIMARY"

# Wait for cluster to be available
aws rds wait db-cluster-available \
  --db-cluster-identifier "$CLUSTER_ID" \
  --region "$REGION_PRIMARY"

# Check current capacity and serverless configuration
aws rds describe-db-clusters \
  --db-cluster-identifier "$CLUSTER_ID" \
  --query 'DBClusters[0].ServerlessV2ScalingConfiguration' \
  --region "$REGION_PRIMARY"

# Modify capacity at runtime (no downtime)
aws rds modify-db-cluster \
  --db-cluster-identifier "$CLUSTER_ID" \
  --serverless-v2-scaling-configuration MinCapacity=1,MaxCapacity=32 \
  --apply-immediately \
  --region "$REGION_PRIMARY"

# Monitor ACU in real time
watch -n 30 "aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name ServerlessDatabaseCapacity \
  --dimensions Name=DBClusterIdentifier,Value=${CLUSTER_ID} \
  --start-time \$(date -u -v-5M +%Y-%m-%dT%H:%M:%SZ) \
  --end-time \$(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 60 \
  --statistics Average Maximum \
  --region $REGION_PRIMARY \
  --query 'sort_by(Datapoints,&Timestamp)[-1]'"

# ═══════════════════════════════════════════════════════════════
# Global Database — creation and configuration
# ═══════════════════════════════════════════════════════════════

# Create global cluster from existing cluster
aws rds create-global-cluster \
  --global-cluster-identifier "$GLOBAL_ID" \
  --source-db-cluster-identifier \
    "arn:aws:rds:${REGION_PRIMARY}:$(aws sts get-caller-identity --query Account --output text):cluster:${CLUSTER_ID}" \
  --region "$REGION_PRIMARY"

# Add secondary region to Global Database
# (must create EMPTY cluster in the secondary region — no master-user)
SUBNET_GROUP_ID_SECONDARY="aurora-secondary-subnet-group"

aws rds create-db-cluster \
  --db-cluster-identifier "${CLUSTER_ID}-secondary" \
  --global-cluster-identifier "$GLOBAL_ID" \
  --engine aurora-postgresql \
  --engine-version "16.3" \
  --db-subnet-group-name "$SUBNET_GROUP_ID_SECONDARY" \
  --serverless-v2-scaling-configuration MinCapacity=0.5,MaxCapacity=16 \
  --storage-encrypted \
  --region "$REGION_SECONDARY"

# Add reader to secondary cluster
aws rds create-db-instance \
  --db-instance-identifier "${CLUSTER_ID}-secondary-reader-1" \
  --db-cluster-identifier "${CLUSTER_ID}-secondary" \
  --db-instance-class db.serverless \
  --engine aurora-postgresql \
  --region "$REGION_SECONDARY"

# Check global database status
aws rds describe-global-clusters \
  --global-cluster-identifier "$GLOBAL_ID" \
  --query 'GlobalClusters[0].{Status:Status,Members:GlobalClusterMembers[*].{Region:DBClusterArn,Writer:IsWriter}}' \
  --output json

# Check replication lag (metric on secondary cluster)
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name AuroraGlobalDBRPOLag \
  --dimensions "Name=DBClusterIdentifier,Value=${CLUSTER_ID}-secondary" \
  --start-time "$(date -u -v-10M +%Y-%m-%dT%H:%M:%SZ)" \
  --end-time "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --period 60 \
  --statistics Average Maximum \
  --region "$REGION_SECONDARY"

# ═══════════════════════════════════════════════════════════════
# Switchover (RPO=0, planned)
# ═══════════════════════════════════════════════════════════════

# Check lag before switchover (should be near zero)
SECONDARY_ARN=$(aws rds describe-global-clusters \
  --global-cluster-identifier "$GLOBAL_ID" \
  --query 'GlobalClusters[0].GlobalClusterMembers[?IsWriter==`false`].DBClusterArn' \
  --output text \
  --region "$REGION_PRIMARY")

echo "Secondary ARN: $SECONDARY_ARN"

# Execute switchover (promotes secondary to primary, no data loss)
aws rds switchover-global-cluster \
  --global-cluster-identifier "$GLOBAL_ID" \
  --target-db-cluster-identifier "$SECONDARY_ARN" \
  --region "$REGION_PRIMARY"

# Wait for completion (may take a few minutes)
aws rds wait db-cluster-available \
  --db-cluster-identifier "${CLUSTER_ID}-secondary" \
  --region "$REGION_SECONDARY"

# Confirm new primary
aws rds describe-global-clusters \
  --global-cluster-identifier "$GLOBAL_ID" \
  --query 'GlobalClusters[0].GlobalClusterMembers[*].{ARN:DBClusterArn,IsPrimary:IsWriter}'

# ═══════════════════════════════════════════════════════════════
# Managed Failover (unplanned DR)
# ═══════════════════════════════════════════════════════════════

# WARNING: use only in actual disaster or DR tests
# --allow-data-loss flag is mandatory
aws rds failover-global-cluster \
  --global-cluster-identifier "$GLOBAL_ID" \
  --target-db-cluster-identifier "$SECONDARY_ARN" \
  --allow-data-loss \
  --region "$REGION_SECONDARY"   # --region = secondary target region

# ═══════════════════════════════════════════════════════════════
# rds.global_db_rpo (Aurora PostgreSQL) — RPO control
# ═══════════════════════════════════════════════════════════════

# Create custom parameter group for the primary
aws rds create-db-cluster-parameter-group \
  --db-cluster-parameter-group-name "checkout-global-params" \
  --db-cluster-parameter-family "aurora-postgresql16" \
  --description "Global DB params with RPO control"

# Set maximum RPO of 30 seconds
aws rds modify-db-cluster-parameter-group \
  --db-cluster-parameter-group-name "checkout-global-params" \
  --parameters "ParameterName=rds.global_db_rpo,ParameterValue=30,ApplyMethod=immediate"

# Check configured RPO via psql
# psql -h <endpoint> -c "SHOW rds.global_db_rpo;"
# Query global status function:
# psql -h <endpoint> -c "SELECT * FROM aurora_global_db_status();"

# ═══════════════════════════════════════════════════════════════
# Aurora Fast Clone — instant clone for staging/dev
# ═══════════════════════════════════════════════════════════════

aws rds restore-db-cluster-to-point-in-time \
  --source-db-cluster-identifier "$CLUSTER_ID" \
  --db-cluster-identifier "${CLUSTER_ID}-staging" \
  --restore-type copy-on-write \
  --use-latest-restorable-time \
  --serverless-v2-scaling-configuration MinCapacity=0,MaxCapacity=4 \
  --region "$REGION_PRIMARY"
```

---

## 8. Pitfalls

[FACT] **Auto-pause and cold start**: configuring `min_capacity=0` in production environments where cold start of ~10–30s is unacceptable. Use `min_capacity=0.5` or higher for production; `min_capacity=0` only for development environments.

[FACT] **Logical replication slots after switchover/failover**: logical replication slots in Aurora PostgreSQL exist only on the current writer. After any switchover or failover, all slots need to be recreated on the new writer — consumers like Debezium will lose connection and need to be reinitialized. Plan for this in the CDC strategy.

[FACT] **Global writer endpoint vs cluster endpoint**: using the primary's cluster endpoint directly (not the global writer endpoint) means that after a switchover, the application will continue pointing to the old primary (now secondary, read-only). Always use the global writer endpoint for writes in Global Database scenarios.

[FACT] **Readers in promotion tier 0/1 consume more ACUs**: a reader in tier 0 always has at minimum the writer's current capacity. With writer at 16 ACUs and 1 reader in tier 0, the cluster consumes a minimum of 32 ACUs even when idle. For cost-reduced environments, use readers in tiers 2+.

[FACT] **Very high max_capacity inflates max_connections**: `max_connections` in Aurora serverless is based on `max_capacity`. Setting `max_capacity=256` without need generates an unrealistic `max_connections` and can interfere with connection management (e.g.: pool sizing). Calibrate `max_capacity` based on real workloads.

[FACT] **Aurora I/O Optimized is more expensive per GB-month**: I/O Optimized is advantageous when I/O represents >25% of total cost. For clusters with low I/O (dev/staging), Aurora I/O Standard with per-I/O-request billing is more economical.

[FACT] **Mixed-configuration clusters and parameter groups**: in mixed clusters (serverless + provisioned), parameters set in the cluster parameter group affect all types. Capacity-related parameters are ignored for provisioned instances. Always verify the effect of parameter group changes on each instance type in the cluster.

---

## Reflection Exercise

A global e-commerce service has the following characteristics:
- Very irregular traffic: peaks of 100× the baseline during Black Friday events (1–2h) and nearly complete inactivity at dawn
- European regulation requires data to be available locally (GDPR); EU users need to read from `eu-west-1`, South American users read from `sa-east-1`
- Maximum tolerated RTO: 5 minutes; Maximum RPO: 30 seconds
- Currently uses RDS PostgreSQL Multi-AZ with 3 read replicas

**Answer:**

1. Does the 100× peak traffic profile justify Aurora serverless or provisioned? Explain considering the trade-off between overprovisioning cost, scaling speed, and the fact that Aurora serverless does not wait for "quiet points" to scale.

2. To meet the local read requirement in EU and SA with Aurora Global Database, what topology would you use? Describe: (a) where the writer would be located, (b) how many secondary clusters, (c) how read routing would work for each region.

3. For the 30-second RPO, would `rds.global_db_rpo=30` guarantee this RPO? Explain the mechanics: what happens with writes on the primary when all secondaries exceed 30s of lag?

4. The data team uses Debezium (CDC via logical replication) to feed a data warehouse. What is the impact of a monthly **planned switchover** to exercise DR? What needs to be done in Debezium before and after the switchover?

5. Compare the idle operation cost between: (a) RDS PostgreSQL Multi-AZ + 3 read replicas with db.r6g.large; (b) Aurora serverless with 1 writer + 2 readers (tiers 0 and 2) with max_capacity=32, min_capacity=0.5. What are the variables that determine which is more economical?

---

## References

- [FACT] [How Aurora serverless works — Amazon Aurora User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.how-it-works.html)
- [FACT] [Aurora serverless capacity (ACU ranges, platform versions) — Amazon Aurora User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.setting-capacity.html)
- [FACT] [Scaling to 0 ACUs (auto-pause) — Amazon Aurora User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2-auto-pause.html)
- [FACT] [Using Amazon Aurora Global Database — Amazon Aurora User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html)
- [FACT] [Switchover and failover in Aurora Global Database — Amazon Aurora User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-disaster-recovery.html)
- [FACT] [Amazon Aurora DB clusters overview — Amazon Aurora User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Overview.html)
