# Session 051 — RDS: Multi-AZ, Read Replicas, RDS Proxy, and Performance Insights

**Estimated duration:** 60 min  
**Prerequisite:** session-044 (Secrets Manager, RDS credential rotation)

---

> ⚠️ **Critical update note**: AWS announced the end of support for the **Performance Insights console on July 31, 2026** (~47 days from this guide's creation date). Performance Insights is being replaced by **CloudWatch Database Insights**. The Performance Insights API is preserved without changes. This guide covers the transition.  
> Source: [AWS RDS User Guide — Performance Insights](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_PerfInsights.html)

---

## Session Objectives

- Distinguish Multi-AZ DB Instance (1 standby, synchronous, not readable) from Multi-AZ DB Cluster (2 semi-synchronous readers, readable) and from Read Replicas (asynchronous, readable, manual promotion)
- Understand the failover behavior of each model and the implications of replication lag
- Configure RDS Proxy with connection pooling, multiplexing, and authentication via IAM + Secrets Manager
- Identify when RDS Proxy causes **pinning** and how to avoid it
- Use Performance Insights (and its replacement Database Insights) to identify the query responsible for a DB Load spike

---

## 1. High Availability and Read Models in RDS

### 1.1 Structural Comparison

[FACT] RDS offers three distinct mechanisms — with orthogonal characteristics — for HA and read scalability:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                     OVERVIEW OF THE 3 MODELS                                    │
│                                                                                 │
│  ┌──────────────────────┐   ┌──────────────────────┐   ┌──────────────────────┐│
│  │ Multi-AZ DB Instance │   │ Multi-AZ DB Cluster  │   │   Read Replica       ││
│  │                      │   │                      │   │                      ││
│  │   Primary (R/W)      │   │   Writer (R/W)       │   │   Primary (R/W)      ││
│  │        │             │   │   /    \             │   │        │ (async)      ││
│  │   [sync]             │   │[semi-  [semi-        │   │   Replica (R/O)       ││
│  │        ↓             │   │ sync]   sync]        │   │   (readable)          ││
│  │   Standby (not       │   │   ↓         ↓        │   │   Can be promoted     ││
│  │   readable)          │   │ Reader1  Reader2     │   │   manually            ││
│  │                      │   │ (readable)           │   │                      ││
│  │ Purpose: HA          │   │ Purpose: HA + Read   │   │ Purpose: Read Scale  ││
│  │ Failover: automatic  │   │ Failover: automatic  │   │ Failover: manual     ││
│  │ RPO: 0 (synchronous) │   │ RPO: near zero       │   │ RPO: variable lag    ││
│  │ RTO: ~35s            │   │ RTO: less than inst  │   │ RTO: high (promotion)││
│  └──────────────────────┘   └──────────────────────┘   └──────────────────────┘│
└─────────────────────────────────────────────────────────────────────────────────┘
```

[FACT] Detailed comparison table:

```
╔═══════════════════════════╦══════════════════╦═══════════════════╦═════════════════╗
║ Dimension                 ║ Multi-AZ Instance║ Multi-AZ Cluster  ║ Read Replica    ║
╠═══════════════════════════╬══════════════════╬═══════════════════╬═════════════════╣
║ Topology                  ║ 1 primary +      ║ 1 writer +        ║ 1+ replicas     ║
║                           ║ 1 standby        ║ 2 readers         ║                 ║
╠═══════════════════════════╬══════════════════╬═══════════════════╬═════════════════╣
║ Replication type          ║ Synchronous      ║ Semi-synchronous  ║ Asynchronous    ║
║                           ║ (before ACK)     ║ (1 reader confirms║ (eventual)      ║
╠═══════════════════════════╬══════════════════╬═══════════════════╬═════════════════╣
║ Standby/reader readable?  ║ NO               ║ YES (readers)     ║ YES             ║
╠═══════════════════════════╬══════════════════╬═══════════════════╬═════════════════╣
║ Automatic failover?       ║ YES (~35s)       ║ YES (faster)      ║ NO (manual)     ║
╠═══════════════════════════╬══════════════════╬═══════════════════╬═════════════════╣
║ RPO (data loss)           ║ Zero             ║ Near zero         ║ Variable lag    ║
║                           ║ (synchronous)    ║ (1+ ack)          ║ (s to min)      ║
╠═══════════════════════════╬══════════════════╬═══════════════════╬═════════════════╣
║ Read scaling              ║ NO               ║ YES (2 readers)   ║ YES (up to 5+)  ║
╠═══════════════════════════╬══════════════════╬═══════════════════╬═════════════════╣
║ Cross-region              ║ NO               ║ NO                ║ YES             ║
╠═══════════════════════════╬══════════════════╬═══════════════════╬═════════════════╣
║ Promotion                 ║ Automatic        ║ Automatic         ║ Manual          ║
╠═══════════════════════════╬══════════════════╬═══════════════════╬═════════════════╣
║ Supported engines         ║ All RDS          ║ MySQL, PostgreSQL ║ All RDS         ║
╠═══════════════════════════╬══════════════════╬═══════════════════╬═════════════════╣
║ Additional cost           ║ 2× instance      ║ 3× instance       ║ 1× per replica  ║
╠═══════════════════════════╬══════════════════╬═══════════════════╬═════════════════╣
║ Write latency             ║ Higher than      ║ Higher than       ║ No impact       ║
║                           ║ Single-AZ        ║ Single-AZ         ║ on primary      ║
╚═══════════════════════════╩══════════════════╩═══════════════════╩═════════════════╝
```

---

## 2. Multi-AZ DB Instance Deployment

### 2.1 How it works

[FACT] RDS automatically provisions a **synchronous standby** replica in a different AZ. Each transaction committed on the primary must be replicated and confirmed on the standby **before** the ACK is sent to the client — this guarantees zero RPO.

[FACT] The standby **does not serve reads** at any point before promotion. It exists exclusively for failover.

[FACT] Automatic failover triggers:
- Primary instance failure (OS crash, hardware failure)
- Network failure in the primary AZ
- Planned maintenance (DB engine upgrade, patching)
- Manual command `reboot-db-instance --force-failover`

[FACT] Typical failover time: **less than 35 seconds**. The cluster DNS endpoint is updated, but there is DNS propagation (~30s additional that RDS Proxy eliminates — see section 4).

[FACT] Replication technology by engine:
- MariaDB, MySQL, Oracle, PostgreSQL, RDS Custom for SQL Server → **Amazon failover technology** (AWS proprietary replication)
- Microsoft SQL Server → **SQL Server Database Mirroring (DBM)** or **Always On Availability Groups (AGs)**

[FACT] Multi-AZ increases write latency compared to Single-AZ due to synchronous replication. AWS recommends **Provisioned IOPS** for production Multi-AZ workloads to minimize this latency.

### 2.2 Multi-AZ DB Cluster (1 writer + 2 readers)

[FACT] The Multi-AZ DB Cluster is a different variant: 1 writer + 2 reader instances in 3 distinct AZs. The readers **can** serve reads. Uses semi-synchronous replication (at least 1 reader must confirm before the ACK).

[FACT] In Multi-AZ DB Cluster failover, the new writer must wait for readers to apply unapplied transactions (replica lag). The failover is faster than the single instance model.

[FACT] Support: MySQL and PostgreSQL only (does not include SQL Server, Oracle, MariaDB).

---

## 3. Read Replicas

### 3.1 Replication Model

[FACT] RDS uses the engine's native replication mechanism (MySQL binlog, PostgreSQL WAL streaming, etc.) to replicate **asynchronously** to read replicas.

[FACT] Creation: RDS takes a snapshot of the source instance and creates the replica from it; from that point on, replication continuously applies changes. The replica remains in read-only mode.

[FACT] Restrictions by engine:
- **Oracle (mounted mode)** and **Db2 (standby mode)**: do not accept read connections — used only for cross-region DR
- **SQL Server, Oracle, Db2**: do not support cascade (replica of replica)
- **MariaDB, MySQL, PostgreSQL (some versions)**: support cascade

[FACT] RDS **does not** support autoscaling of read replicas (unlike Aurora). Creation and removal are always manual.

[FACT] Pricing: same rate as the instance class. **Replication data transfer within the same region is not charged.** Cross-region: data transfer charges apply.

### 3.2 Replica Lag

[FACT] The CloudWatch metric `ReplicaLag` (in seconds) measures the delay between the last transaction on the writer and the last transaction applied on the replica.

[FACT] Causes of high lag:
- High write rate on the primary (replica cannot keep up)
- Long-running queries on the replica blocking replication application
- Undersized replica instance (CPU/IO)
- Maintenance events (backup, parameter group changes)
- Network congestion (especially cross-region)

```
Monitor lag via CloudWatch:
Namespace: AWS/RDS
Metric:    ReplicaLag
Dimension: DBInstanceIdentifier = <replica-id>
Alarm:     > 300 seconds (5 min) → immediate action
```

### 3.3 Read Replica Promotion

[FACT] Promoting a read replica to a standalone instance is an **irreversible and manual** operation. After promotion:
- The replica stops receiving replication
- It becomes a standalone R/W instance
- It may take several minutes (replica lag must be zeroed first)
- The replication binary (MySQL) or recovery WAL (PostgreSQL) remains available, but the replica will no longer replicate from the former primary

[CONSENSUS] Read replica promotion is used as a Disaster Recovery (DR) strategy when the primary fails in another region and there is no automatic failover mechanism. The RPO depends on the lag at the time of failure.

---

## 4. RDS Proxy

### 4.1 Architecture and Motivation

[FACT] RDS Proxy is a **managed, multi-AZ, serverless proxy** that sits between the application and the database. It solves two main problems:

1. **Connection exhaustion**: lambdas, containers, and microservices create ephemeral connections at high speed. The database has a fixed limit of simultaneous connections based on memory (e.g.: PostgreSQL default has `max_connections = LEAST({DBInstanceClassMemory/9531392}, 5000)`). The Proxy maintains a smaller connection pool with the database and multiplexes connections from multiple clients.

2. **Failover without reconnection**: during a Multi-AZ failover, existing connections die and the application needs to reconnect. RDS Proxy absorbs the failover internally and redirects connections to the new primary without clients losing their idle connections.

```
┌──────────────────────────────────────────────────────────────────────┐
│ CONNECTION FLOW WITH RDS PROXY                                       │
│                                                                      │
│  App (Lambda, ECS, K8s)                                              │
│  ├─ 1000 simultaneous connections → RDS Proxy endpoint               │
│                                     │                                │
│                              ┌──────┴──────┐                         │
│                              │  POOL (50   │                         │
│                              │  connections)│                         │
│                              └──────┬──────┘                         │
│                                     │  multiplexing                  │
│                                     ↓                                │
│                          RDS DB Instance (max 50 conns)              │
│                                                                      │
│  Result: database sees 50 connections, not 1000                      │
└──────────────────────────────────────────────────────────────────────┘
```

### 4.2 Multiplexing and Pinning

[FACT] **Multiplexing** (connection reuse): by default, the Proxy reuses connections at the transaction level. A pool connection serves a client during a transaction; at the end of the transaction (COMMIT/ROLLBACK), the connection returns to the pool for another client. With `autocommit=ON`, each statement can release the connection.

[FACT] **Pinning**: when the Proxy detects that it is not safe to share the connection, it "pins" it to the client for the entire session duration. Pinning is transparent to the client but reduces the pool's benefit.

[FACT] Causes of pinning (PostgreSQL and MySQL):
- `SET` statements that alter session state (e.g.: `SET statement_timeout = 5000`)
- Open transactions with `BEGIN`
- Use of prepared statements (MySQL with `mysql_stmt_*`)
- DDL statements (`CREATE TABLE`, `ALTER TABLE`)
- Functions that return session state (`LAST_INSERT_ID()`, `ROW_COUNT()`, `lastval()`)
- Statements with text larger than **16 KB**
- `GET DIAGNOSTICS` (MariaDB/MySQL)

[FACT] To detect pinning: CloudWatch metric `DatabaseConnectionsCurrentlySessionPinned` — a high value indicates that the pool is being underutilized.

[CONSENSUS] To reduce pinning: avoid unnecessary local `SET` statements (use **proxy initialization query** for fixed session settings), prefer ORMs that use short transactions, use `INSERT...RETURNING` instead of `lastval()` in PostgreSQL.

### 4.3 RDS Proxy Limits

[FACT] Important limits:

```
╔══════════════════════════════════════════════════╦══════════════════╗
║ Limit                                            ║ Value            ║
╠══════════════════════════════════════════════════╬══════════════════╣
║ Proxies per AWS account                          ║ 20 (soft limit)  ║
║ Secrets Manager secrets per proxy                ║ 200              ║
║ (= max DB users when using secrets)              ║                  ║
║ Additional endpoints per proxy                   ║ 20               ║
║ AZs of default endpoint                          ║ 2 (from subnets) ║
║ Statement size for pinning                       ║ > 16 KB          ║
║ DB instances per proxy                           ║ 1                ║
║ Proxies per DB instance                          ║ Multiple (N)     ║
╚══════════════════════════════════════════════════╩══════════════════╝
```

[FACT] Network restrictions:
- The Proxy must be in the **same VPC** as the database
- The Proxy **cannot be publicly accessible**
- Does not work with `dedicated` tenancy VPC
- Does not work with VPC with `Enforce Mode` of encryption controls

[FACT] In Multi-AZ clusters, the Proxy associates only with the **writer** (not with read replicas). For read routing via Proxy in clusters, use additional Proxy read endpoints.

### 4.4 Authentication in RDS Proxy

[FACT] Three authentication modes:

```
MODE 1 — Database credentials only (no IAM enforcement):
  App → (username/password) → Proxy → (SM secret) → DB
  Proxy retrieves creds from Secrets Manager, does not enforce IAM for the client

MODE 2 — Standard IAM Authentication (recommended for applications):
  App → (IAM token via rds-db:connect) → Proxy → (SM secret) → DB
  Applies IAM for client; Proxy authenticates to DB via Secrets Manager
  Works even if DB uses native password

MODE 3 — End-to-end IAM Authentication:
  App → (IAM token) → Proxy → (IAM token) → DB
  DB must have IAM database authentication enabled
  Eliminates need for SM secrets for database credentials
  NOT supported for SQL Server
```

[FACT] For IAM auth (modes 2 and 3), the client must generate a token via:
```python
import boto3
token = boto3.client('rds').generate_db_auth_token(
    DBHostname='proxy-endpoint.proxy-xxx.us-east-1.rds.amazonaws.com',
    Port=5432,
    DBUsername='app_user',
    Region='us-east-1'
)
# Token is valid for 15 minutes
```

[FACT] When using Secrets Manager (modes 1 and 2): create **one secret per database user**. Each secret must have the format:
```json
{"username": "app_user", "password": "supersecret"}
```

[FACT] The Proxy automatically creates the `rdsproxyadmin` user in the database. Do not delete or modify this user's permissions — it can render the Proxy inoperative.

---

## 5. Performance Insights and Database Insights

### 5.1 Performance Insights — current status (EOL Jul/2026)

> ⚠️ **[FACT] The Performance Insights console will be discontinued on July 31, 2026**. The console will redirect to CloudWatch Database Insights. The **Performance Insights API remains unchanged**. CloudFormation and Terraform parameters continue to work.

[FACT] If no action is taken, instances with Performance Insights will be automatically migrated to **Standard mode** of Database Insights with the same retention period.

[FACT] Database Insights modes:
- **Standard mode**: same experience as Performance Insights today, same flexible retention pricing (1–24 months)
- **Advanced mode**: fleet-level monitoring, lock diagnostics, execution plan capture (more expensive)

### 5.2 Performance Insights / Database Insights Concepts

[FACT] Performance Insights monitors **DB Load** = average number of active sessions (Average Active Sessions / AAS) using the database engine.

[FACT] DB Load is decomposed by:
- **Wait events**: what sessions are waiting for (e.g.: `io/datafileread`, `Lock:relation`, `CPU`)
- **SQL statements**: which queries contribute to the load
- **Hosts**: which host the connections come from
- **Users**: which database user is contributing

[FACT] The visual reference line is `max_vCPUs` — when DB Load exceeds the number of vCPUs, the database is saturated.

[FACT] Counter metrics in Performance Insights (examples for PostgreSQL):
- `db.transactions.xact_commit` — commits per second
- `db.rows.tup_fetched` — rows returned per second
- `db.io.blks_hit` — cache hits (buffer pool)
- `db.io.blks_read` — disk reads (physical I/O)

[FACT] Retention period:
- Free tier: 7 days
- Paid: 1–24 months (preserved in Database Insights Standard mode)

[FACT] API for programmatic metric reading:
```
aws pi get-resource-metrics \
  --service-type RDS \
  --identifier db:<db-resource-id> \
  --metric-queries '[{"Metric":"db.load.avg","GroupBy":{"Group":"db.sql","Limit":5}}]' \
  --start-time 2026-06-14T10:00:00Z \
  --end-time   2026-06-14T11:00:00Z \
  --period-in-seconds 60
```

---

## 6. CDK Python — Complete Stack with Multi-AZ, Read Replica, Proxy, and Monitoring

```python
"""
CDK Stack: RDS PostgreSQL with Multi-AZ, Read Replica,
RDS Proxy (IAM auth), Performance Insights, and alarms.
"""
from aws_cdk import (
    Stack, Duration, RemovalPolicy, CfnOutput,
    aws_rds as rds,
    aws_ec2 as ec2,
    aws_iam as iam,
    aws_secretsmanager as sm,
    aws_cloudwatch as cw,
    aws_cloudwatch_actions as cwa,
    aws_sns as sns,
)
from constructs import Construct


class RDSStack(Stack):
    def __init__(self, scope: Construct, construct_id: str, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        # ──────────────────────────────────────────────────────────────
        # VPC
        # ──────────────────────────────────────────────────────────────
        vpc = ec2.Vpc(self, "AppVpc",
            max_azs=3,
            nat_gateways=1,
        )

        # Security Groups
        db_sg = ec2.SecurityGroup(self, "DbSg", vpc=vpc,
            description="RDS PostgreSQL access",
        )
        proxy_sg = ec2.SecurityGroup(self, "ProxySg", vpc=vpc,
            description="RDS Proxy",
        )
        app_sg = ec2.SecurityGroup(self, "AppSg", vpc=vpc,
            description="Application layer",
        )

        # Proxy accepts connections from the application
        proxy_sg.add_ingress_rule(app_sg, ec2.Port.tcp(5432))
        # Database accepts connections only from the proxy (not directly from the application)
        db_sg.add_ingress_rule(proxy_sg, ec2.Port.tcp(5432))

        # ──────────────────────────────────────────────────────────────
        # Secret (database credentials)
        # ──────────────────────────────────────────────────────────────
        db_secret = sm.Secret(self, "DbSecret",
            description="RDS PostgreSQL master credentials",
            generate_secret_string=sm.SecretStringGenerator(
                secret_string_template='{"username": "pgadmin"}',
                generate_string_key="password",
                exclude_punctuation=True,
            ),
        )

        # ──────────────────────────────────────────────────────────────
        # RDS — Multi-AZ DB Instance (PostgreSQL)
        # ──────────────────────────────────────────────────────────────
        db_instance = rds.DatabaseInstance(self, "Postgres",
            engine=rds.DatabaseInstanceEngine.postgres(
                version=rds.PostgresEngineVersion.VER_16_3,
            ),
            instance_type=ec2.InstanceType.of(
                ec2.InstanceClass.BURSTABLE3, ec2.InstanceSize.LARGE
            ),
            vpc=vpc,
            vpc_subnets=ec2.SubnetSelection(
                subnet_type=ec2.SubnetType.PRIVATE_WITH_EGRESS
            ),
            security_groups=[db_sg],
            credentials=rds.Credentials.from_secret(db_secret),
            # Multi-AZ: synchronous standby in a different AZ
            multi_az=True,
            # Storage
            allocated_storage=100,
            storage_type=rds.StorageType.GP3,
            iops=3000,
            storage_encrypted=True,
            # Backups
            backup_retention=Duration.days(7),
            preferred_backup_window="03:00-04:00",
            # Maintenance
            preferred_maintenance_window="Sun:04:00-Sun:05:00",
            # Performance Insights (Standard mode / Database Insights)
            enable_performance_insights=True,
            performance_insight_retention=rds.PerformanceInsightRetention.DEFAULT,  # 7 days
            # Monitoring
            monitoring_interval=Duration.seconds(60),
            # IAM auth: allows token-based IAM authentication in addition to password
            iam_database_authentication_enabled=True,
            # Deletion protection in prod
            deletion_protection=True,
            removal_policy=RemovalPolicy.RETAIN,
            # Custom parameter group
            parameter_group=rds.ParameterGroup(self, "PgParams",
                engine=rds.DatabaseInstanceEngine.postgres(
                    version=rds.PostgresEngineVersion.VER_16_3
                ),
                parameters={
                    "log_min_duration_statement": "1000",   # log queries > 1s
                    "log_lock_waits": "on",
                    "idle_in_transaction_session_timeout": "30000",  # 30s
                    "work_mem": "16384",    # 16 MB
                },
            ),
        )

        # ──────────────────────────────────────────────────────────────
        # Read Replica (same region, different AZ)
        # ──────────────────────────────────────────────────────────────
        read_replica = rds.DatabaseInstanceReadReplica(self, "PgReadReplica",
            source_database_instance=db_instance,
            instance_type=ec2.InstanceType.of(
                ec2.InstanceClass.BURSTABLE3, ec2.InstanceSize.LARGE
            ),
            vpc=vpc,
            vpc_subnets=ec2.SubnetSelection(
                subnet_type=ec2.SubnetType.PRIVATE_WITH_EGRESS
            ),
            security_groups=[db_sg],
            enable_performance_insights=True,
            storage_encrypted=True,
            deletion_protection=False,
            removal_policy=RemovalPolicy.DESTROY,
        )

        # ──────────────────────────────────────────────────────────────
        # IAM Role for RDS Proxy
        # ──────────────────────────────────────────────────────────────
        proxy_role = iam.Role(self, "RdsProxyRole",
            assumed_by=iam.ServicePrincipal("rds.amazonaws.com"),
        )
        # Allow Proxy to read the SM secret
        db_secret.grant_read(proxy_role)

        # ──────────────────────────────────────────────────────────────
        # RDS Proxy — Standard IAM auth (app → IAM → Proxy → SM → DB)
        # ──────────────────────────────────────────────────────────────
        proxy = rds.DatabaseProxy(self, "PgProxy",
            proxy_target=rds.ProxyTarget.from_instance(db_instance),
            secrets=[db_secret],
            vpc=vpc,
            vpc_subnets=ec2.SubnetSelection(
                subnet_type=ec2.SubnetType.PRIVATE_WITH_EGRESS
            ),
            security_groups=[proxy_sg],
            # Require IAM auth from clients
            iam_auth=True,
            # Require TLS
            require_tls=True,
            # Idle connection timeout (5-1800s)
            idle_client_timeout=Duration.minutes(30),
            # Session initialization (avoids ad-hoc SET that would cause pinning)
            init_query="SET application_name='rds-proxy'",
            # Debug logs (caution: verbose in prod)
            debug_logging=False,
        )

        # IAM policy for the application to connect to the proxy via token
        app_connect_policy = iam.PolicyStatement(
            effect=iam.Effect.ALLOW,
            actions=["rds-db:connect"],
            resources=[
                # Proxy user ARN: arn:aws:rds-db:REGION:ACCOUNT:dbuser:PROXY-ID/USERNAME
                f"arn:aws:rds-db:{self.region}:{self.account}:dbuser:{proxy.db_proxy_arn.split(':')[-1]}/pgadmin",
            ],
        )

        # ──────────────────────────────────────────────────────────────
        # CloudWatch Alarms
        # ──────────────────────────────────────────────────────────────
        alert_topic = sns.Topic(self, "DbAlerts")

        # Alarm: ReplicaLag > 5 minutes
        cw.Alarm(self, "ReplicaLagAlarm",
            metric=cw.Metric(
                namespace="AWS/RDS",
                metric_name="ReplicaLag",
                dimensions_map={"DBInstanceIdentifier": read_replica.instance_identifier},
                period=Duration.minutes(1),
                statistic="Average",
            ),
            threshold=300,           # 5 minutes in seconds
            evaluation_periods=3,
            comparison_operator=cw.ComparisonOperator.GREATER_THAN_THRESHOLD,
            alarm_description="Replica lag > 5 minutes",
            treat_missing_data=cw.TreatMissingData.MISSING,
        ).add_alarm_action(cwa.SnsAction(alert_topic))

        # Alarm: DatabaseConnections (indirect Proxy pinning — many active connections)
        cw.Alarm(self, "ProxyPinningAlarm",
            metric=cw.Metric(
                namespace="AWS/RDS",
                metric_name="DatabaseConnectionsCurrentlySessionPinned",
                dimensions_map={"ProxyName": proxy.db_proxy_name},
                period=Duration.minutes(5),
                statistic="Average",
            ),
            threshold=50,    # adjust according to baseline
            evaluation_periods=2,
            comparison_operator=cw.ComparisonOperator.GREATER_THAN_THRESHOLD,
            alarm_description="Excessive RDS Proxy pinning — review session SET statements",
        ).add_alarm_action(cwa.SnsAction(alert_topic))

        # Outputs
        CfnOutput(self, "ProxyEndpoint", value=proxy.endpoint)
        CfnOutput(self, "ReplicaEndpoint", value=read_replica.db_instance_endpoint_address)
        CfnOutput(self, "SecretArn", value=db_secret.secret_arn)
```

---

## 7. Python — Application Connecting via RDS Proxy with IAM Token

```python
"""
Module for connecting to RDS via Proxy using IAM token.
Demonstrates: token generation, connection pool, and lag detection.
"""
import boto3
import psycopg2
from psycopg2 import pool
import logging
import time
from functools import lru_cache

logger = logging.getLogger(__name__)


# ──────────────────────────────────────────────────────────────────────
# Configuration
# ──────────────────────────────────────────────────────────────────────

PROXY_HOST   = "my-proxy.proxy-abc123.us-east-1.rds.amazonaws.com"
REPLICA_HOST = "postgres-read-replica.abc123.us-east-1.rds.amazonaws.com"
DB_PORT      = 5432
DB_NAME      = "appdb"
DB_USER      = "pgadmin"
AWS_REGION   = "us-east-1"


# ──────────────────────────────────────────────────────────────────────
# IAM Token Generation (valid 15 min — cache per session)
# ──────────────────────────────────────────────────────────────────────

class IAMTokenManager:
    """Manages IAM tokens for RDS Proxy authentication."""

    def __init__(self, host: str, port: int, user: str, region: str):
        self.host = host
        self.port = port
        self.user = user
        self.region = region
        self._token: str | None = None
        self._token_expiry: float = 0

    def get_token(self) -> str:
        """Returns IAM token, generating a new one if needed (14 min cache)."""
        now = time.time()
        # Token lasts 15 min; renew 1 min early to avoid mid-connection expiration
        if not self._token or now >= self._token_expiry - 60:
            rds_client = boto3.client("rds", region_name=self.region)
            self._token = rds_client.generate_db_auth_token(
                DBHostname=self.host,
                Port=self.port,
                DBUsername=self.user,
                Region=self.region,
            )
            self._token_expiry = now + 15 * 60
            logger.debug("IAM token renewed for %s", self.host)
        return self._token


# ──────────────────────────────────────────────────────────────────────
# Connection Pool (Proxy already does real pooling, but a local pool
# avoids TLS/TCP overhead on each Lambda/container request)
# ──────────────────────────────────────────────────────────────────────

class RDSConnectionManager:
    """
    Manages database connection via RDS Proxy with IAM auth.
    Typical usage: instantiate once in the module (outside the Lambda handler)
    for reuse between warm invocations (warm Lambda).
    """

    def __init__(self, host: str, is_writer: bool = True):
        self.host = host
        self.token_mgr = IAMTokenManager(host, DB_PORT, DB_USER, AWS_REGION)
        self._conn: psycopg2.extensions.connection | None = None

    def _connect(self) -> psycopg2.extensions.connection:
        token = self.token_mgr.get_token()
        conn = psycopg2.connect(
            host=self.host,
            port=DB_PORT,
            dbname=DB_NAME,
            user=DB_USER,
            password=token,   # IAM token as password
            sslmode="require",
            connect_timeout=5,
        )
        conn.autocommit = True   # facilitates multiplexing in the Proxy
        return conn

    def get_connection(self) -> psycopg2.extensions.connection:
        """Returns active connection; reconnects if needed."""
        try:
            if self._conn is None or self._conn.closed:
                self._conn = self._connect()
            else:
                # Lightweight ping to verify connection is still alive
                self._conn.cursor().execute("SELECT 1")
        except Exception:
            logger.warning("Reconnecting to database...")
            self._conn = self._connect()
        return self._conn

    def execute(self, query: str, params: tuple = ()) -> list[dict]:
        """Executes a query and returns results as a list of dicts."""
        conn = self.get_connection()
        with conn.cursor() as cur:
            cur.execute(query, params)
            if cur.description:
                cols = [desc[0] for desc in cur.description]
                return [dict(zip(cols, row)) for row in cur.fetchall()]
        return []


# Singletons — instantiate outside the handler in Lambda (warm reuse)
writer_conn = RDSConnectionManager(PROXY_HOST,   is_writer=True)
reader_conn = RDSConnectionManager(REPLICA_HOST, is_writer=False)


# ──────────────────────────────────────────────────────────────────────
# Replica Lag Query via CloudWatch
# ──────────────────────────────────────────────────────────────────────

def get_replica_lag_seconds(replica_identifier: str) -> float | None:
    """
    Returns the current replica lag in seconds.
    Returns None if the metric is not available.
    """
    cw = boto3.client("cloudwatch", region_name=AWS_REGION)
    from datetime import datetime, timezone, timedelta

    response = cw.get_metric_statistics(
        Namespace="AWS/RDS",
        MetricName="ReplicaLag",
        Dimensions=[{"Name": "DBInstanceIdentifier", "Value": replica_identifier}],
        StartTime=datetime.now(timezone.utc) - timedelta(minutes=5),
        EndTime=datetime.now(timezone.utc),
        Period=60,
        Statistics=["Average"],
    )

    datapoints = sorted(response["Datapoints"], key=lambda x: x["Timestamp"])
    if datapoints:
        lag = datapoints[-1]["Average"]
        logger.info("Replica lag: %.1f seconds", lag)
        return lag
    return None


def route_query(query: str, params: tuple = (), max_lag_seconds: float = 30) -> list[dict]:
    """
    Routes read queries to the replica if lag is acceptable.
    For writes or if lag is high, uses the writer via proxy.
    """
    # Detect if it's a read query (simple heuristic)
    is_read = query.strip().upper().startswith("SELECT")

    if is_read:
        lag = get_replica_lag_seconds("my-postgres-replica")
        if lag is not None and lag <= max_lag_seconds:
            logger.info("Routing to replica (lag=%.1fs)", lag)
            return reader_conn.execute(query, params)
        else:
            logger.warning("High lag (%.1fs) — routing to writer", lag or 9999)

    return writer_conn.execute(query, params)


# ──────────────────────────────────────────────────────────────────────
# DB Load Query via Performance Insights API
# ──────────────────────────────────────────────────────────────────────

def get_top_sql_by_db_load(
    db_resource_id: str,
    lookback_minutes: int = 60,
    top_n: int = 5,
) -> list[dict]:
    """
    Returns the top N queries by average DB Load in the period.
    db_resource_id: obtained via describe-db-instances (.DbiResourceId)
    
    NOTE: the Performance Insights API is preserved even after
    the migration to Database Insights (EOL Jul/2026).
    """
    from datetime import datetime, timezone, timedelta

    pi = boto3.client("pi", region_name=AWS_REGION)
    end_time   = datetime.now(timezone.utc)
    start_time = end_time - timedelta(minutes=lookback_minutes)

    response = pi.get_resource_metrics(
        ServiceType="RDS",
        Identifier=f"db:{db_resource_id}",
        MetricQueries=[{
            "Metric": "db.load.avg",
            "GroupBy": {
                "Group": "db.sql",
                "Dimensions": ["db.sql.statement"],
                "Limit": top_n,
            },
        }],
        StartTime=start_time,
        EndTime=end_time,
        PeriodInSeconds=60,
    )

    results = []
    for metric_list in response.get("MetricList", []):
        key_dims = metric_list.get("Key", {}).get("Dimensions", {})
        sql_text = key_dims.get("db.sql.statement", "N/A")[:200]   # truncate

        # Average DB Load for this query in the period
        datapoints = metric_list.get("DataPoints", [])
        if datapoints:
            avg_load = sum(dp.get("Value", 0) for dp in datapoints) / len(datapoints)
            results.append({
                "sql": sql_text,
                "avg_db_load_aas": round(avg_load, 3),
                "datapoints": len(datapoints),
            })

    return sorted(results, key=lambda x: x["avg_db_load_aas"], reverse=True)


if __name__ == "__main__":
    # Example: list top 5 queries by DB Load
    top_queries = get_top_sql_by_db_load(
        db_resource_id="db-ABCDEF1234567890",
        lookback_minutes=60,
    )
    for i, q in enumerate(top_queries, 1):
        print(f"\n#{i} — DB Load: {q['avg_db_load_aas']} AAS")
        print(f"  SQL: {q['sql'][:100]}...")
```

---

## 8. CLI — Complete Operations

```bash
# ═══════════════════════════════════════════════════════════════
# Multi-AZ — Creation and failover
# ═══════════════════════════════════════════════════════════════

export DB_ID="postgres-prod"
export REGION="us-east-1"

# Create Multi-AZ PostgreSQL instance
aws rds create-db-instance \
  --db-instance-identifier "$DB_ID" \
  --engine postgres \
  --engine-version "16.3" \
  --db-instance-class db.t3.large \
  --allocated-storage 100 \
  --storage-type gp3 \
  --iops 3000 \
  --storage-encrypted \
  --multi-az \
  --master-username pgadmin \
  --master-user-password "$(aws secretsmanager get-random-password \
    --password-length 16 --no-include-space --output text)" \
  --backup-retention-period 7 \
  --preferred-backup-window "03:00-04:00" \
  --enable-performance-insights \
  --iam-database-authentication-enabled \
  --region "$REGION"

# Check secondary AZ and status
aws rds describe-db-instances \
  --db-instance-identifier "$DB_ID" \
  --query 'DBInstances[0].{Status:DBInstanceStatus,AZ:AvailabilityZone,SecondaryAZ:SecondaryAvailabilityZone,MultiAZ:MultiAZ}' \
  --output table

# Force manual failover (test)
aws rds reboot-db-instance \
  --db-instance-identifier "$DB_ID" \
  --force-failover

# Wait for availability after failover
aws rds wait db-instance-available \
  --db-instance-identifier "$DB_ID"

# Confirm that AZ changed (failover occurred)
aws rds describe-db-instances \
  --db-instance-identifier "$DB_ID" \
  --query 'DBInstances[0].{AZ:AvailabilityZone,SecondaryAZ:SecondaryAvailabilityZone}'

# ═══════════════════════════════════════════════════════════════
# Read Replica
# ═══════════════════════════════════════════════════════════════

# Create read replica (same region)
aws rds create-db-instance-read-replica \
  --db-instance-identifier "${DB_ID}-replica" \
  --source-db-instance-identifier "$DB_ID" \
  --db-instance-class db.t3.large \
  --enable-performance-insights

# Monitor ReplicaLag in real time (30s per point)
watch -n 30 "aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name ReplicaLag \
  --dimensions Name=DBInstanceIdentifier,Value=${DB_ID}-replica \
  --start-time \$(date -u -v-10M +%Y-%m-%dT%H:%M:%SZ) \
  --end-time \$(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 60 \
  --statistics Average \
  --query 'sort_by(Datapoints,&Timestamp)[-1].Average'"

# Promote read replica to standalone instance (DR)
aws rds promote-read-replica \
  --db-instance-identifier "${DB_ID}-replica" \
  --backup-retention-period 7 \
  --preferred-backup-window "03:00-04:00"

# Wait for promotion
aws rds wait db-instance-available \
  --db-instance-identifier "${DB_ID}-replica"

echo "Replica promoted — now a standalone instance"

# ═══════════════════════════════════════════════════════════════
# RDS Proxy
# ═══════════════════════════════════════════════════════════════

export SECRET_ARN=$(aws secretsmanager describe-secret \
  --secret-id "rds/postgres-prod/pgadmin" \
  --query SecretArn --output text)

export PROXY_ROLE_ARN="arn:aws:iam::123456789012:role/rds-proxy-role"
export SUBNET_IDS="subnet-aaa,subnet-bbb,subnet-ccc"
export SG_ID="sg-proxy123"

# Create proxy with Standard IAM auth
aws rds create-db-proxy \
  --db-proxy-name "${DB_ID}-proxy" \
  --engine-family POSTGRESQL \
  --auth "[{
    \"AuthScheme\": \"SECRETS\",
    \"SecretArn\": \"$SECRET_ARN\",
    \"IAMAuth\": \"REQUIRED\"
  }]" \
  --role-arn "$PROXY_ROLE_ARN" \
  --vpc-subnet-ids $SUBNET_IDS \
  --vpc-security-group-ids "$SG_ID" \
  --require-tls \
  --idle-client-timeout 1800 \
  --debug-logging false

# Register the DB as a proxy target
aws rds register-db-proxy-targets \
  --db-proxy-name "${DB_ID}-proxy" \
  --db-instance-identifiers "$DB_ID"

# Wait for proxy to become available
aws rds wait db-proxy-available \
  --db-proxy-name "${DB_ID}-proxy"

# Check proxy endpoint
aws rds describe-db-proxies \
  --db-proxy-name "${DB_ID}-proxy" \
  --query 'DBProxies[0].{Endpoint:Endpoint,Status:Status}'

# Check target health
aws rds describe-db-proxy-targets \
  --db-proxy-name "${DB_ID}-proxy" \
  --query 'Targets[*].{TargetArn:TargetArn,RDSResourceId:RDSResourceId,State:TargetHealth.State,Reason:TargetHealth.Reason}'

# Test connection via proxy with IAM token
PROXY_HOST="my-proxy.proxy-abc.us-east-1.rds.amazonaws.com"
TOKEN=$(aws rds generate-db-auth-token \
  --hostname "$PROXY_HOST" \
  --port 5432 \
  --username pgadmin \
  --region "$REGION")

PGPASSWORD="$TOKEN" psql \
  --host="$PROXY_HOST" \
  --port=5432 \
  --username=pgadmin \
  --dbname=postgres \
  --no-password \
  -c "SELECT current_user, inet_server_addr(), version();"

# ═══════════════════════════════════════════════════════════════
# Performance Insights → Database Insights
# ═══════════════════════════════════════════════════════════════

# Get DbiResourceId (required for the PI API)
DB_RESOURCE_ID=$(aws rds describe-db-instances \
  --db-instance-identifier "$DB_ID" \
  --query 'DBInstances[0].DbiResourceId' \
  --output text)

echo "Resource ID: $DB_RESOURCE_ID"

# Top 5 queries by DB Load — last hour
aws pi get-resource-metrics \
  --service-type RDS \
  --identifier "db:${DB_RESOURCE_ID}" \
  --metric-queries '[{
    "Metric": "db.load.avg",
    "GroupBy": {
      "Group": "db.sql",
      "Dimensions": ["db.sql.statement", "db.sql.db_id"],
      "Limit": 5
    }
  }]' \
  --start-time "$(date -u -v-1H +%Y-%m-%dT%H:%M:%SZ)" \
  --end-time   "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --period-in-seconds 300 \
  --query 'MetricList[].{SQL:Key.Dimensions."db.sql.statement",Load:DataPoints[-1].Value}' \
  --output table

# Top wait events — last hour
aws pi get-resource-metrics \
  --service-type RDS \
  --identifier "db:${DB_RESOURCE_ID}" \
  --metric-queries '[{
    "Metric": "db.load.avg",
    "GroupBy": {
      "Group": "db.wait_event",
      "Dimensions": ["db.wait_event.name", "db.wait_event.type"],
      "Limit": 10
    }
  }]' \
  --start-time "$(date -u -v-1H +%Y-%m-%dT%H:%M:%SZ)" \
  --end-time   "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --period-in-seconds 300 \
  --output json | python3 -c "
import json, sys
data = json.load(sys.stdin)
for m in data['MetricList']:
  dims = m['Key'].get('Dimensions', {})
  event = dims.get('db.wait_event.name', 'N/A')
  wtype = dims.get('db.wait_event.type', 'N/A')
  dps = m.get('DataPoints', [])
  avg = sum(d['Value'] for d in dps) / len(dps) if dps else 0
  print(f'{avg:.3f} AAS | {wtype} | {event}')
" | sort -rn

# Counter metrics (I/O)
aws pi get-resource-metrics \
  --service-type RDS \
  --identifier "db:${DB_RESOURCE_ID}" \
  --metric-queries '[
    {"Metric": "db.io.blks_read.avg"},
    {"Metric": "db.io.blks_hit.avg"},
    {"Metric": "db.transactions.xact_commit.avg"}
  ]' \
  --start-time "$(date -u -v-30M +%Y-%m-%dT%H:%M:%SZ)" \
  --end-time   "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --period-in-seconds 60 \
  --output json | python3 -c "
import json, sys
data = json.load(sys.stdin)
for m in data['MetricList']:
  metric = m['Key']['Metric']
  dps = m.get('DataPoints', [])
  if dps:
    latest = sorted(dps, key=lambda x: x['Timestamp'])[-1]
    print(f'{metric}: {latest[\"Value\"]:.2f}')
"

# Migration to Database Insights (Standard mode — no changes needed)
# DB instances with PI enabled will be automatically migrated on Jul 31 2026
# To activate Advanced mode (lock diagnostics, execution plans):
aws rds modify-db-instance \
  --db-instance-identifier "$DB_ID" \
  --database-insights-mode advanced \
  --apply-immediately
```

---

## 9. Pitfalls

[FACT] **Confusing Multi-AZ standby with read replica**: the Multi-AZ DB Instance standby **does not accept reads**. Attempting a direct connection to the standby will fail — there is no separate endpoint. For reads with HA, use Multi-AZ DB Cluster or create separate read replicas.

[FACT] **Promoting read replica before zeroing lag**: when promoting a replica during DR, recent queries on the primary that have not yet been replicated will be lost. Verify `ReplicaLag = 0` before promotion if RPO is critical.

[FACT] **RDS Proxy does not support read replicas as target**: the Proxy associates only with the writer. For read routing via Proxy in clusters, use additional Proxy endpoints pointing to the cluster readers (Multi-AZ Cluster or Aurora).

[FACT] **Undeclared pinning silently degrades the pool**: `SET` statements in application code (timezone, search_path, etc.) cause pinning without a visible error — the pool works, but with less reuse. Monitor `DatabaseConnectionsCurrentlySessionPinned`. Move fixed configurations to the Proxy's `init_query`.

[FACT] **IAM token with 15-minute validity**: do not generate a new token per request — this negates the pool's benefit. The token should be cached and renewed ~1 minute before expiration (see IAMTokenManager above).

[FACT] **PostgreSQL `max_connections` based on memory**: on small instances, the default can be surprisingly low. `db.t3.medium` has ~1 GB of memory → `max_connections ≈ 107`. RDS Proxy is especially critical for Lambda/ECS that create many ephemeral connections.

[FACT] **Performance Insights EOL Jul/2026**: dashboards and automations that use the Performance Insights console directly need to be migrated to the CloudWatch Database Insights console before 07/31/2026. The **`pi:*` API does not change** — existing scripts and code continue to work.

[FACT] **Multi-AZ does not protect against logical errors**: a `DROP TABLE` on the primary is synchronously replicated to the standby. Multi-AZ protects against hardware/AZ failures, not application errors. For that: backups + Point-in-Time Recovery (PITR).

---

## Reflection Exercise

An e-commerce system has:
- `orders` DB (PostgreSQL 16, Multi-AZ DB Instance, db.r6g.xlarge)
- A read replica in the same region for reports
- 500 Lambdas that process payments (each invocation opens and closes a connection to the database)
- A reporting service that runs heavy analytical queries (JOIN of 3 tables, ~30s execution)

**Answer:**

1. The 500 Lambdas are causing `FATAL: remaining connection slots are reserved for non-replication superuser connections`. What is the root cause and how does RDS Proxy solve it? What would be the estimated `max_connections` of a `db.r6g.xlarge` (32 GB of RAM) with the formula `LEAST({DBInstanceClassMemory/9531392}, 5000)`?

2. The reporting service uses `SET work_mem = '256MB'` at the beginning of each session to improve JOIN performance. Why would this cause pinning in RDS Proxy? How would you configure the Proxy to apply this setting without causing pinning for **all** connections?

3. During an incident: the primary AZ `us-east-1a` suffered a network failure at 14:00. At 14:01, the standby in `us-east-1b` was promoted. At 14:02, the team noticed that application connections were still going to the old address. What is the cause (DNS propagation delay) and how would RDS Proxy have avoided the interruption?

4. Performance Insights showed that the query `SELECT * FROM orders JOIN order_items ON ... WHERE created_at > $1` contributes 3.2 AAS of DB Load when the system has 4 vCPUs. What does this number mean and what would be the first analysis to do in the Performance Insights dashboard to diagnose the bottleneck?

5. After July 31, 2026, does a monitoring script that uses `aws pi get-resource-metrics` to collect DB Load continue to work? And the Performance Insights console dashboard?

---

## References

- [FACT] [Multi-AZ DB instance deployments — Amazon RDS User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZSingleStandby.html)
- [FACT] [Multi-AZ DB cluster deployments — Amazon RDS User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/multi-az-db-clusters-concepts.html)
- [FACT] [Working with DB instance read replicas — Amazon RDS User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReadRepl.html)
- [FACT] [Amazon RDS Proxy — Amazon RDS User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy.html)
- [FACT] [RDS Proxy concepts and terminology — Amazon RDS User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy.howitworks.html)
- [FACT] [Performance Insights EOL announcement + Database Insights — Amazon RDS User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_PerfInsights.html)
