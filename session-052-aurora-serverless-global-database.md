# Sessão 052 — Aurora: Serverless, Global Database e Diferenças vs RDS Padrão

**Duração estimada:** 60 min  
**Pré-requisito:** session-051 (RDS Multi-AZ, Read Replicas, RDS Proxy)

---

> **Nota de nomenclatura (abril 2026)**: A AWS renomeou o "Aurora Serverless v2" para simplesmente **"Aurora serverless"** em abril de 2026. A mudança é apenas de nome — sem ação necessária em deployments existentes. Este guia usa a nomenclatura atual.  
> Fonte: [How Aurora serverless works — Aurora User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.how-it-works.html)

---

## Objetivos da sessão

- Entender a arquitetura do Aurora (shared cluster volume, separação compute/storage) e como ela difere do RDS
- Configurar Aurora serverless com min/max ACU e entender o modelo de cobrança, auto-pause e scaling
- Criar Aurora Global Database com cluster primário e secundário cross-region
- Distinguir Switchover (RPO=0) de Managed Failover (RPO=segundos) e quando usar cada um
- Listar as diferenças funcionais entre Aurora PostgreSQL e RDS PostgreSQL que afetam decisões de migração

---

## 1. Arquitetura Aurora — diferenças fundamentais em relação ao RDS

### 1.1 Shared cluster volume

[FATO] A diferença arquitetural mais importante entre Aurora e RDS é o **cluster volume compartilhado**. No RDS, cada instância tem seus próprios volumes EBS. No Aurora, o writer e todos os readers compartilham um único volume de storage distribuído:

```
┌────────────────────────────────────────────────────────────────────────┐
│ RDS PostgreSQL (tradicional)                                           │
│                                                                        │
│  Writer ──────── EBS gp3 (volume próprio, replicação async)           │
│  Read Replica ── EBS gp3 (volume próprio, recebe WAL stream)          │
│  Standby ─────── EBS gp3 (volume próprio, replicação síncrona)        │
│                                                                        │
│  Replicação de DADOS entre volumes → latência de write                 │
└────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────┐
│ Aurora                                                                 │
│                                                                        │
│  Writer ──────────┐                                                    │
│  Reader 1 ────────┤──── AURORA CLUSTER VOLUME ────────────────────────│
│  Reader 2 ────────┘         (6 cópias em 3 AZs)                      │
│                                                                        │
│  Replicação via redo log no storage layer (não entre instâncias)       │
│  Readers são promovidos sem copiar dados                               │
└────────────────────────────────────────────────────────────────────────┘
```

[FATO] O cluster volume do Aurora mantém **6 cópias dos dados em 3 AZs** (2 por AZ). O Aurora tolera:
- Perda de até 1 cópia sem perder disponibilidade de escrita
- Perda de até 3 cópias sem perder disponibilidade de leitura
- Reparo automático de cópias corrompidas

[FATO] Consequência do shared volume: readers Aurora são **zero-lag** em relação à durabilidade dos dados (o dado já está no shared volume quando o writer confirma). Existe um pequeno replica lag de visibilidade (o reader precisa ver o cache invalidado), mas o dado nunca é perdido se o reader é promovido.

### 1.2 Componentes do Aurora cluster

[FATO] Um Aurora DB cluster consiste de:
- **Writer (primary) DB instance**: 1 por cluster, R/W, executa DDL e DML
- **Aurora Replicas (reader DB instances)**: até 15 por cluster, R/O, compartilham o mesmo cluster volume
- **Cluster endpoint** (`cluster-xxx.us-east-1.rds.amazonaws.com`): sempre aponta para o writer atual
- **Reader endpoint** (`cluster-ro-xxx.us-east-1.rds.amazonaws.com`): balanceia entre readers disponíveis
- **Instance endpoints**: endpoint direto para cada instância (diagnóstico/troubleshooting)

[FATO] Failover dentro do cluster (intra-region): o reader com menor promotion tier (0 = maior prioridade, 15 = menor prioridade) é promovido automaticamente. Tempo típico: **~30 segundos**. O cluster endpoint é atualizado automaticamente.

---

## 2. Aurora Serverless — ACUs, scaling e cobrança

### 2.1 Unidade de medida: ACU

[FATO] A unidade de capacidade do Aurora serverless é a **ACU (Aurora Capacity Unit)**. Cada ACU representa aproximadamente **2 GiB de memória + CPU correspondente + rede**. A capacidade não é vinculada às classes de instância (db.t3, db.r6g etc.) do Aurora provisionado.

[FATO] Ranges de ACU por versão (a partir de versões recentes suportadas):

```
╔═══════════════════════════════════════════════════════════════╗
║ Versão              ║ Range de ACUs ║ Auto-pause (scale-to-0) ║
╠═════════════════════╬═══════════════╬═════════════════════════╣
║ Aurora MySQL 3.02+  ║ 0.5 – 128     ║ Não                     ║
║ Aurora MySQL 3.06+  ║ 0.5 – 256     ║ Não                     ║
║ Aurora MySQL 3.08+  ║ 0 – 256       ║ SIM                     ║
║ Aurora PG 13.6+     ║ 0.5 – 128     ║ Não                     ║
║ Aurora PG 13.13+    ║ 0.5 – 256     ║ Não                     ║
║ Aurora PG 13.15+    ║ 0 – 256       ║ SIM                     ║
║ Aurora PG 16.3+     ║ 0 – 256       ║ SIM                     ║
╚═════════════════════╩═══════════════╩═════════════════════════╝
```

[FATO] **Platform versions** do Aurora serverless (gerenciadas automaticamente pela AWS):
- v1: range 0–128 ACUs, baseline
- v2: range 0–256 ACUs, baseline
- v3: range 0–256 ACUs, +30% performance vs v2
- v4: range 0–256 ACUs, +30% performance vs v3 (disponível em regiões selecionadas incluindo us-east-1, us-west-2, etc.)

[FATO] Novos clusters são sempre criados na platform version mais recente disponível. Clusters existentes podem ser atualizados via stop/start ou blue/green deployments.

### 2.2 Modelo de cobrança

[FATO] Aurora serverless é cobrado em **ACU-hours**, medidos **por segundo**. Não há cobrança por "instância" — cobra-se pela capacidade efetivamente usada a cada segundo.

[FATO] Cálculo de consumo do cluster:
- Consumo mínimo = `N_instâncias × ACU_mínimo`
- Consumo máximo = `N_instâncias × ACU_máximo`
- Onde N_instâncias = número total de writers + readers

[FATO] O storage do Aurora serverless é cobrado separadamente (mesmo modelo do Aurora provisionado): GB-mês de dados armazenados + I/O requests.

### 2.3 Scaling — como e quando acontece

[FATO] O Aurora serverless monitora continuamente CPU, memória e rede de cada writer/reader. O scaling acontece:
- **Scale-up**: quando a capacidade atual é insuficiente para a carga
- **Scale-down**: quando a capacidade atual é maior que o necessário
- Incrementos mínimos: **0.5 ACUs**
- O scaling é **não-disruptivo**: não aguarda um "quiet point", não fecha conexões abertas, não interrompe transações em andamento

[FATO] Métricas CloudWatch para monitorar scaling:
- `ServerlessDatabaseCapacity`: capacidade atual em ACUs (float)
- `ACUUtilization`: percentual de uso da capacidade atual

### 2.4 Auto-pause (scale to zero)

[FATO] Versões que suportam `minimum = 0 ACUs` permitem **auto-pause**: quando o cluster fica ocioso por um período configurável, ele pausa completamente. Ao receber conexão, resume automaticamente.

[FATO] Trade-off do auto-pause: a primeira conexão após uma pausa longa pode levar **10–30 segundos** para a instância retornar. Usar apenas em ambientes de desenvolvimento/teste onde latência de cold start é aceitável.

### 2.5 Promotion tiers e HA

[FATO] Readers em **promotion tiers 0 e 1**: a capacidade mínima é vinculada à capacidade atual do writer. Se o writer está em 8 ACUs, o reader em tier 0/1 terá mínimo de 8 ACUs também. Garante que o reader está sempre pronto para fazer failover sem precisar escalar primeiro.

[FATO] Readers em **promotion tiers 2–15**: escalam independentemente. Podem estar em 1 ACU enquanto o writer está em 32 ACUs. Adequados para workloads de relatórios ou análise que não precisam ser failover targets primários.

### 2.6 max_connections no Aurora serverless

[FATO] O `max_connections` no Aurora serverless é definido automaticamente pelo Aurora com base no **ACU máximo configurado**, não no ACU atual. Isso evita que conexões sejam dropadas quando a capacidade escala para baixo. Para Aurora PostgreSQL serverless:
```
max_connections = LEAST({max_acus × 1000}, 5000)
```
(A fórmula exata varia por engine; o princípio é baseado no max ACU.)

### 2.7 Mixed-configuration clusters

[FATO] Um cluster Aurora pode misturar instâncias serverless e provisionadas:
- Writer provisionado + readers serverless (ex: write workload estável, reads variáveis)
- Writer serverless + readers provisionados (ex: writers variáveis, reads de reports estáveis)
- Tudo serverless (configuração mais comum para novos projetos)

---

## 3. Aurora Global Database

### 3.1 Arquitetura

[FATO] Aurora Global Database é uma feature do Aurora para replicação **cross-region**:
- **1 cluster primário** (R/W) em uma região
- **Até 5 clusters secundários** (R/O) em regiões diferentes
- Replicação: storage-based, assíncrona, **lag típico < 1 segundo**
- Até **16 réplicas por região secundária**

```
┌────────────────────────────────────────────────────────────────────┐
│                    AURORA GLOBAL DATABASE                          │
│                                                                    │
│  us-east-1 (PRIMARY)                us-west-2 (SECONDARY)         │
│  ┌──────────────────────┐           ┌──────────────────────┐       │
│  │ Writer (R/W)         │           │ Reader(s) — R/O       │       │
│  │ Reader 1             │──────────▶│ (réplicas da primary) │       │
│  │ Reader 2             │ storage   └──────────────────────┘       │
│  └──────────────────────┘   repl     lag < 1s tipicamente         │
│                                                                    │
│  ┌──────────────────────┐                                          │
│  │ sa-east-1 (SECONDARY)│ ← até 5 regiões secundárias            │
│  │ Reader(s) — R/O      │                                          │
│  └──────────────────────┘                                          │
│                                                                    │
│  ENDPOINTS:                                                        │
│  Global writer endpoint → cluster-xxx.global.rds.amazonaws.com    │
│  Secondary reader     → cluster-ro-xxx.us-west-2.rds.amazonaws.com│
└────────────────────────────────────────────────────────────────────┘
```

[FATO] O **global writer endpoint** é um endpoint DNS único que sempre aponta para o writer do cluster primário atual. Em um switchover/failover, o global writer endpoint é atualizado automaticamente. Usar esse endpoint nas aplicações elimina a necessidade de alteração de string de conexão após failover.

[FATO] Métricas CloudWatch para monitorar Global Database:
- `AuroraGlobalDBRPOLag` (ms): lag de replicação medido como RPO — disponível para Aurora PostgreSQL e Aurora MySQL ≥ 3.04.0 (metric *por cluster secundário*)
- `AuroraGlobalDBReplicationLag` (ms): para versões MySQL mais antigas

### 3.2 Switchover (RPO = 0) — failover planejado

[FATO] **Switchover** (anteriormente chamado "managed planned failover") é usado em cenários controlados:
- Manutenção operacional planejada
- Rotação regional (ex: regulação financeira que exige exercitar DR)
- "Follow-the-sun": mover o writer para a região com usuários ativos
- Fail-back após um failover não planejado

[FATO] O switchover **aguarda o cluster secundário target estar completamente sincronizado** com o primário antes de trocar os papéis. Por isso, **RPO = 0 (sem perda de dados)**. O banco fica indisponível por um curto período durante a troca de papéis.

[FATO] Requisito: clusters primário e secundário devem ter a mesma versão major e minor (patch levels podem diferir dependendo da versão).

[FATO] CLI para switchover:
```bash
aws rds --region <primary-region> \
  switchover-global-cluster \
  --global-cluster-identifier <global-db-id> \
  --target-db-cluster-identifier <arn-of-secondary-to-promote>
```

### 3.3 Managed Failover (RPO = segundos) — failover não planejado

[FATO] **Managed Failover** é usado para recuperação de desastre (outage regional não planejado):
- **Não aguarda** sincronização com o primário
- **RPO ≠ 0**: dados replicados parcialmente podem ser perdidos
- A quantidade de perda depende do `AuroraGlobalDBRPOLag` no momento da falha

[FATO] O managed failover é "gerenciado" porque:
1. O secondary promovido assume o papel de primary
2. Quando a região antiga se recupera, Aurora **automaticamente a re-adiciona como secondary**
3. Aurora tenta tirar um snapshot do storage antigo: `rds:unplanned-global-failover-<nome>-<timestamp>` (disponível no console para recuperação de dados perdidos)
4. "Write fencing": Aurora tenta bloquear writes na região antiga para evitar split-brain (best-effort)

[FATO] CLI para managed failover (unplanned):
```bash
aws rds --region <secondary-region> \
  failover-global-cluster \
  --global-cluster-identifier <global-db-id> \
  --target-db-cluster-identifier <arn-of-secondary-to-promote> \
  --allow-data-loss   # flag obrigatória — reconhece que pode haver perda de dados
```

### 3.4 RPO management (Aurora PostgreSQL)

[FATO] Para Aurora PostgreSQL, o parâmetro `rds.global_db_rpo` permite definir um RPO máximo em segundos. Quando ativo, se **todos** os clusters secundários excederem o lag configurado, o primário **bloqueia commits** até que pelo menos um secondary volte ao target. Faixa válida: 20s a 2.147.483.647s.

[FATO] `rds.global_db_rpo` não é recomendado em setups com apenas 2 regiões: uma falha da região secundária causaria bloqueio de todos os writes na primária.

---

## 4. Aurora PostgreSQL vs RDS PostgreSQL — Diferenças para decisões de migração

[FATO] Comparação técnica:

```
╔══════════════════════════════════╦═════════════════════════╦═════════════════════════╗
║ Aspecto                          ║ RDS PostgreSQL          ║ Aurora PostgreSQL        ║
╠══════════════════════════════════╬═════════════════════════╬═════════════════════════╣
║ Storage                          ║ EBS por instância       ║ Cluster volume          ║
║                                  ║ (sync replication)      ║ (6 cópias, 3 AZs)       ║
╠══════════════════════════════════╬═════════════════════════╬═════════════════════════╣
║ Max readers                      ║ Até 5 read replicas     ║ Até 15 Aurora Replicas  ║
╠══════════════════════════════════╬═════════════════════════╬═════════════════════════╣
║ Failover intra-region            ║ ~35s (DNS propagation)  ║ ~30s                    ║
╠══════════════════════════════════╬═════════════════════════╬═════════════════════════╣
║ Replica lag (readers)            ║ Async — segundos/minutos║ Quase zero (shared vol) ║
╠══════════════════════════════════╬═════════════════════════╬═════════════════════════╣
║ Cross-region DR                  ║ Cross-region read       ║ Global Database         ║
║                                  ║ replica (manual promo.) ║ (switchover/failover)   ║
╠══════════════════════════════════╬═════════════════════════╬═════════════════════════╣
║ Serverless                       ║ NÃO                     ║ SIM (ACU-based)         ║
╠══════════════════════════════════╬═════════════════════════╬═════════════════════════╣
║ Logical replication slots        ║ Suportado plenamente    ║ Suportado, mas slots    ║
║                                  ║ (include standby)       ║ precisam ser recriados  ║
║                                  ║                         ║ após switchover/failover║
╠══════════════════════════════════╬═════════════════════════╬═════════════════════════╣
║ file_fdw                         ║ Suportado               ║ NÃO suportado           ║
║                                  ║                         ║ (sem acesso direto ao FS)║
╠══════════════════════════════════╬═════════════════════════╬═════════════════════════╣
║ Parameter groups                 ║ Apenas instance level   ║ Cluster level +         ║
║                                  ║                         ║ instance level          ║
╠══════════════════════════════════╬═════════════════════════╬═════════════════════════╣
║ max_connections default          ║ LEAST(mem/9531392, 5000)║ Baseado no max ACU      ║
║                                  ║                         ║ (serverless) ou class   ║
╠══════════════════════════════════╬═════════════════════════╬═════════════════════════╣
║ Aurora-specific functions        ║ Não                     ║ aurora_version(),       ║
║                                  ║                         ║ aurora_replica_status() ║
╠══════════════════════════════════╬═════════════════════════╬═════════════════════════╣
║ Custo storage                    ║ GB-mês (EBS gp3/io2)    ║ GB-mês (Aurora I/O     ║
║                                  ║                         ║ Standard ou Optimized)  ║
╠══════════════════════════════════╬═════════════════════════╬═════════════════════════╣
║ Cloning                          ║ Não (usar snapshot)     ║ SIM — Aurora Fast Clone ║
║                                  ║                         ║ (copy-on-write, instant)║
╚══════════════════════════════════╩═════════════════════════╩═════════════════════════╝
```

[FATO] **Replication slots e failover**: em Aurora PostgreSQL, os logical replication slots existem apenas no writer. Após um switchover ou failover, os slots precisam ser recriados no novo writer. Se houver consumers (ex: Debezium/CDC tools) usando logical replication slots, eles devem ser pausados durante o failover e reconectados ao novo writer com slots recriados.

[FATO] **file_fdw**: o `file_fdw` lê arquivos diretamente do filesystem do servidor. Como o Aurora não expõe filesystem diretamente, esse FDW não funciona. `postgres_fdw` (conecta em outro PostgreSQL) funciona normalmente.

[FATO] **Aurora Storage I/O**: Aurora oferece dois modos de storage:
- **Aurora I/O Standard**: cobra por I/O requests separadamente
- **Aurora I/O Optimized**: preço maior por GB-mês, mas sem cobranças separadas por I/O — melhor para workloads I/O intensivos (>25% do custo de storage em I/O)

---

## 5. CDK Python — Aurora Serverless + Global Database

```python
"""
CDK Stacks para Aurora serverless (PostgreSQL) com Global Database.
Stack 1 (primary): cria o cluster serverless na região primária
Stack 2 (secondary): adiciona cluster secundário em outra região
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
    """Cluster Aurora serverless PostgreSQL na região primária."""

    def __init__(self, scope: Construct, construct_id: str, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        # ──────────────────────────────────────────────────────────────
        # VPC e security group
        # ──────────────────────────────────────────────────────────────
        vpc = ec2.Vpc(self, "Vpc", max_azs=3, nat_gateways=1)
        db_sg = ec2.SecurityGroup(self, "DbSg", vpc=vpc)
        db_sg.add_ingress_rule(ec2.Peer.ipv4(vpc.vpc_cidr_block), ec2.Port.tcp(5432))

        # ──────────────────────────────────────────────────────────────
        # Credenciais
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
            min_capacity=0,    # 0 = auto-pause habilitado (Aurora PG 16.3+)
            max_capacity=16,   # 16 ACUs ≈ 32 GiB RAM max
        )

        # Cluster PostgreSQL 16
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
            # Writer serverless (tier 0: failover target prioritário)
            writer=rds.ClusterInstance.serverless_v2("Writer",
                scale_with_writer=True,   # reader segue capacidade do writer
            ),
            # Reader serverless em tier 0 (min capacity = writer capacity)
            readers=[
                rds.ClusterInstance.serverless_v2("Reader1",
                    scale_with_writer=True,    # tier 0/1: escala com writer
                    promotion_tier=1,
                ),
                # Reader para relatórios: escala independentemente (tier 2)
                rds.ClusterInstance.serverless_v2("ReaderReports",
                    scale_with_writer=False,   # tier 2-15: independente
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
        # Global Database — associa este cluster como primary
        # ──────────────────────────────────────────────────────────────
        global_cluster = rds.CfnGlobalCluster(self, "GlobalCluster",
            global_cluster_identifier="checkout-global",
            source_db_cluster_identifier=self.cluster.cluster_arn,
            deletion_protection=False,
        )

        # ──────────────────────────────────────────────────────────────
        # Alarmes CloudWatch
        # ──────────────────────────────────────────────────────────────
        alert_topic = sns.Topic(self, "AuroraAlerts")

        # ACU Utilization > 80% → considerar aumentar max ACU
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
            alarm_description="Aurora serverless ACU utilization > 80% — considerar aumentar max ACU",
        ).add_alarm_action(cwa.SnsAction(alert_topic))

        # Global DB replication lag > 5 segundos
        cw.Alarm(self, "GlobalDbLagAlarm",
            metric=cw.Metric(
                namespace="AWS/RDS",
                metric_name="AuroraGlobalDBRPOLag",
                # Monitorar no cluster secundário, não no primário
                dimensions_map={"DBClusterIdentifier": self.cluster.cluster_identifier},
                period=Duration.minutes(1),
                statistic="Maximum",
            ),
            threshold=5000,   # 5 segundos em milissegundos
            evaluation_periods=3,
            alarm_description="Global DB replication lag > 5s — verificar conectividade cross-region",
        ).add_alarm_action(cwa.SnsAction(alert_topic))

        CfnOutput(self, "ClusterEndpoint", value=self.cluster.cluster_endpoint.hostname)
        CfnOutput(self, "ReaderEndpoint",  value=self.cluster.cluster_read_endpoint.hostname)
        CfnOutput(self, "ClusterArn",      value=self.cluster.cluster_arn)
        CfnOutput(self, "SecretArn",       value=db_secret.secret_arn)


class AuroraGlobalSecondaryStack(Stack):
    """Cluster Aurora serverless na região secundária (joined to Global Database)."""

    def __init__(self, scope: Construct, construct_id: str,
                 global_cluster_id: str, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        vpc = ec2.Vpc(self, "Vpc", max_azs=3, nat_gateways=1)
        db_sg = ec2.SecurityGroup(self, "DbSg", vpc=vpc)
        db_sg.add_ingress_rule(ec2.Peer.ipv4(vpc.vpc_cidr_block), ec2.Port.tcp(5432))

        # Cluster secundário: replica do global database
        # Não precisa de credenciais próprias — herda do primário
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
            # Serverless scaling para a região secundária
            serverless_v2_scaling_configuration=rds.CfnDBCluster.ServerlessV2ScalingConfigurationProperty(
                min_capacity=0.5,   # não usar auto-pause no secondary (latência de resume)
                max_capacity=16,
            ),
        )

        # Adicionar 1 reader serverless ao cluster secundário
        rds.CfnDBInstance(self, "SecondaryReader",
            db_instance_class="db.serverless",
            db_cluster_identifier=secondary.ref,
            engine="aurora-postgresql",
            db_instance_identifier="checkout-secondary-reader-1",
        )

        CfnOutput(self, "SecondaryClusterArn", value=secondary.attr_db_cluster_arn)
```

---

## 6. Python — Monitoramento de Global Database e detecção de capacidade

```python
"""
Utilitários para monitoramento de Aurora serverless e Global Database.
"""
import boto3
import psycopg2
from datetime import datetime, timezone, timedelta
import time


# ──────────────────────────────────────────────────────────────────────
# 1. Monitor de ACU — detecta se o cluster está próximo do limite
# ──────────────────────────────────────────────────────────────────────

def get_acu_stats(cluster_id: str, lookback_minutes: int = 60,
                  region: str = "us-east-1") -> dict:
    """
    Retorna estatísticas de ACU do cluster serverless.
    Útil para decidir se o max_capacity precisa ser aumentado.
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

    # Diagnóstico simples
    if acu_util_max and acu_util_max > 90:
        result["recommendation"] = (
            f"ATENÇÃO: utilização máxima de ACU = {acu_util_max:.1f}%. "
            "Considere aumentar max_capacity para evitar throttling."
        )
    elif acu_util_avg and acu_util_avg < 20:
        result["recommendation"] = (
            f"Utilização média baixa ({acu_util_avg:.1f}%). "
            "Considere reduzir max_capacity para economizar."
        )
    else:
        result["recommendation"] = "Capacidade dentro de faixas adequadas."

    return result


# ──────────────────────────────────────────────────────────────────────
# 2. Monitor de Global Database — lag e status de replicação
# ──────────────────────────────────────────────────────────────────────

def get_global_db_replication_status(
    global_cluster_id: str,
    primary_region: str = "us-east-1",
    secondary_regions: list[str] | None = None,
) -> dict:
    """
    Verifica lag de replicação de todas as regiões secundárias.
    Usa a métrica AuroraGlobalDBRPOLag (em ms).
    """
    secondary_regions = secondary_regions or ["us-west-2", "sa-east-1"]

    rds_client = boto3.client("rds", region_name=primary_region)

    # Obter informação do global cluster
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

        # Coletar lag apenas para secundários
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
# 3. Verificação de diferenças Aurora vs RDS via SQL
# ──────────────────────────────────────────────────────────────────────

def check_aurora_specifics(conn_string: str) -> dict:
    """
    Conecta ao Aurora PostgreSQL e verifica funções/views específicas.
    Útil para validar pós-migração de RDS para Aurora.
    """
    conn = psycopg2.connect(conn_string)
    conn.autocommit = True
    cur = conn.cursor()
    results = {}

    # aurora_version() — retorna versão do Aurora engine
    try:
        cur.execute("SELECT aurora_version()")
        results["aurora_version"] = cur.fetchone()[0]
    except Exception as e:
        results["aurora_version"] = f"ERROR: {e}"

    # aurora_replica_status() — lag dos readers (executar no writer)
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

    # Verificar max_connections configurado
    cur.execute("SHOW max_connections")
    results["max_connections"] = int(cur.fetchone()[0])

    # Verificar logical replication slots (atenção pós-switchover)
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

    # Verificar ACU stats
    stats = get_acu_stats("checkout-aurora", lookback_minutes=60)
    print("ACU Stats:", json.dumps(stats, indent=2, default=str))

    # Verificar replicação global
    global_status = get_global_db_replication_status(
        global_cluster_id="checkout-global",
        primary_region="us-east-1",
    )
    print("Global DB Status:", json.dumps(global_status, indent=2, default=str))
```

---

## 7. CLI — Operações Aurora Serverless e Global Database

```bash
# ═══════════════════════════════════════════════════════════════
# Aurora Serverless — criação e configuração
# ═══════════════════════════════════════════════════════════════

export CLUSTER_ID="checkout-aurora"
export GLOBAL_ID="checkout-global"
export REGION_PRIMARY="us-east-1"
export REGION_SECONDARY="us-west-2"

# Criar cluster Aurora serverless PostgreSQL 16
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

# Adicionar writer serverless (obrigatório — cluster sem instância não aceita conexões)
aws rds create-db-instance \
  --db-instance-identifier "${CLUSTER_ID}-writer" \
  --db-cluster-identifier "$CLUSTER_ID" \
  --db-instance-class db.serverless \
  --engine aurora-postgresql \
  --region "$REGION_PRIMARY"

# Adicionar reader serverless (tier 1 = failover prioritário)
aws rds create-db-instance \
  --db-instance-identifier "${CLUSTER_ID}-reader-1" \
  --db-cluster-identifier "$CLUSTER_ID" \
  --db-instance-class db.serverless \
  --engine aurora-postgresql \
  --promotion-tier 1 \
  --region "$REGION_PRIMARY"

# Aguardar cluster disponível
aws rds wait db-cluster-available \
  --db-cluster-identifier "$CLUSTER_ID" \
  --region "$REGION_PRIMARY"

# Verificar capacidade atual e configuração serverless
aws rds describe-db-clusters \
  --db-cluster-identifier "$CLUSTER_ID" \
  --query 'DBClusters[0].ServerlessV2ScalingConfiguration' \
  --region "$REGION_PRIMARY"

# Modificar capacidade em runtime (sem downtime)
aws rds modify-db-cluster \
  --db-cluster-identifier "$CLUSTER_ID" \
  --serverless-v2-scaling-configuration MinCapacity=1,MaxCapacity=32 \
  --apply-immediately \
  --region "$REGION_PRIMARY"

# Monitorar ACU em tempo real
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
# Global Database — criação e configuração
# ═══════════════════════════════════════════════════════════════

# Criar global cluster a partir do cluster existente
aws rds create-global-cluster \
  --global-cluster-identifier "$GLOBAL_ID" \
  --source-db-cluster-identifier \
    "arn:aws:rds:${REGION_PRIMARY}:$(aws sts get-caller-identity --query Account --output text):cluster:${CLUSTER_ID}" \
  --region "$REGION_PRIMARY"

# Adicionar região secundária ao Global Database
# (deve criar cluster VAZIO na região secondary — sem master-user)
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

# Adicionar reader ao cluster secundário
aws rds create-db-instance \
  --db-instance-identifier "${CLUSTER_ID}-secondary-reader-1" \
  --db-cluster-identifier "${CLUSTER_ID}-secondary" \
  --db-instance-class db.serverless \
  --engine aurora-postgresql \
  --region "$REGION_SECONDARY"

# Verificar status do global database
aws rds describe-global-clusters \
  --global-cluster-identifier "$GLOBAL_ID" \
  --query 'GlobalClusters[0].{Status:Status,Members:GlobalClusterMembers[*].{Region:DBClusterArn,Writer:IsWriter}}' \
  --output json

# Verificar lag de replicação (métrica no cluster secundário)
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
# Switchover (RPO=0, planejado)
# ═══════════════════════════════════════════════════════════════

# Verificar lag antes do switchover (deve ser próximo a zero)
SECONDARY_ARN=$(aws rds describe-global-clusters \
  --global-cluster-identifier "$GLOBAL_ID" \
  --query 'GlobalClusters[0].GlobalClusterMembers[?IsWriter==`false`].DBClusterArn' \
  --output text \
  --region "$REGION_PRIMARY")

echo "Secondary ARN: $SECONDARY_ARN"

# Executar switchover (promove secondary para primary, sem perda de dados)
aws rds switchover-global-cluster \
  --global-cluster-identifier "$GLOBAL_ID" \
  --target-db-cluster-identifier "$SECONDARY_ARN" \
  --region "$REGION_PRIMARY"

# Aguardar conclusão (pode levar alguns minutos)
aws rds wait db-cluster-available \
  --db-cluster-identifier "${CLUSTER_ID}-secondary" \
  --region "$REGION_SECONDARY"

# Confirmar novo primary
aws rds describe-global-clusters \
  --global-cluster-identifier "$GLOBAL_ID" \
  --query 'GlobalClusters[0].GlobalClusterMembers[*].{ARN:DBClusterArn,IsPrimary:IsWriter}'

# ═══════════════════════════════════════════════════════════════
# Managed Failover (unplanned DR)
# ═══════════════════════════════════════════════════════════════

# ATENÇÃO: só usar em desastre real ou testes de DR
# --allow-data-loss flag é obrigatória
aws rds failover-global-cluster \
  --global-cluster-identifier "$GLOBAL_ID" \
  --target-db-cluster-identifier "$SECONDARY_ARN" \
  --allow-data-loss \
  --region "$REGION_SECONDARY"   # --region = região do secondary target

# ═══════════════════════════════════════════════════════════════
# rds.global_db_rpo (Aurora PostgreSQL) — controle de RPO
# ═══════════════════════════════════════════════════════════════

# Criar parameter group customizado para o primary
aws rds create-db-cluster-parameter-group \
  --db-cluster-parameter-group-name "checkout-global-params" \
  --db-cluster-parameter-family "aurora-postgresql16" \
  --description "Global DB params com RPO control"

# Definir RPO máximo de 30 segundos
aws rds modify-db-cluster-parameter-group \
  --db-cluster-parameter-group-name "checkout-global-params" \
  --parameters "ParameterName=rds.global_db_rpo,ParameterValue=30,ApplyMethod=immediate"

# Verificar RPO configurado via psql
# psql -h <endpoint> -c "SHOW rds.global_db_rpo;"
# Consultar função de status global:
# psql -h <endpoint> -c "SELECT * FROM aurora_global_db_status();"

# ═══════════════════════════════════════════════════════════════
# Aurora Fast Clone — clone instantâneo para staging/dev
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

## 8. Armadilhas (Pitfalls)

[FATO] **Auto-pause e cold start**: configurar `min_capacity=0` em ambientes de produção onde cold start de ~10–30s é inaceitável. Usar `min_capacity=0.5` ou maior para produção; `min_capacity=0` apenas para ambientes de desenvolvimento.

[FATO] **Logical replication slots após switchover/failover**: slots de replicação lógica no Aurora PostgreSQL existem apenas no writer atual. Após qualquer switchover ou failover, todos os slots precisam ser recriados no novo writer — consumers como Debezium perderão conexão e precisarão ser reinicializados. Planejar para isso na estratégia de CDC.

[FATO] **Global writer endpoint vs cluster endpoint**: usar o cluster endpoint do primary diretamente (não o global writer endpoint) significa que após um switchover, a aplicação continuará apontando para o antigo primary (agora secondary, read-only). Usar sempre o global writer endpoint para writes em cenários de Global Database.

[FATO] **Readers em promotion tier 0/1 consomem mais ACUs**: um reader em tier 0 sempre tem no mínimo a capacidade do writer atual. Com writer em 16 ACUs e 1 reader em tier 0, o cluster consome mínimo de 32 ACUs mesmo em idle. Para ambientes de custo reduzido, usar leitores em tiers 2+.

[FATO] **max_capacity muito alto infla o max_connections**: o `max_connections` no Aurora serverless é baseado no `max_capacity`. Definir `max_capacity=256` sem necessidade gera um `max_connections` irreal e pode interferir no gerenciamento de conexões (ex: pool sizing). Calibrar `max_capacity` com base em workloads reais.

[FATO] **Aurora I/O Optimized é mais caro em GB-mês**: I/O Optimized é vantajoso quando I/O representa >25% do custo total. Para clusters com baixo I/O (dev/staging), Aurora I/O Standard com cobrança por I/O request é mais econômico.

[FATO] **Mixed-configuration clusters e parameter groups**: em clusters mistos (serverless + provisionado), parâmetros definidos no cluster parameter group afetam todos os tipos. Parâmetros capacity-related são ignorados para instâncias provisionadas. Verificar sempre o efeito de mudanças de parameter group em cada tipo de instância do cluster.

---

## Exercício de Reflexão

Um serviço de e-commerce global tem as seguintes características:
- Tráfego muito irregular: picos de 100× o baseline durante eventos de Black Friday (1–2h) e inatividade quase completa de madrugada
- Regulação europeia exige que dados sejam disponibilizados localmente (GDPR); usuários na EU precisam ler de `eu-west-1`, usuários na América do Sul leem de `sa-east-1`
- RTO máximo tolerado: 5 minutos; RPO máximo: 30 segundos
- Atualmente usa RDS PostgreSQL Multi-AZ com 3 read replicas

**Responda:**

1. O perfil de tráfego 100× de pico justifica Aurora serverless ou provisioned? Explique considerando o trade-off entre custo de overprovisioning, velocidade de scaling e o fato de que o Aurora serverless não aguarda "quiet points" para escalar.

2. Para atender ao requisito de leitura local em EU e SA com Aurora Global Database, qual topologia você usaria? Descreva: (a) onde ficaria o writer, (b) quantos clusters secundários, (c) como o roteamento de leitura funcionaria para cada região.

3. Para o RPO de 30 segundos, o `rds.global_db_rpo=30` garantiria esse RPO? Explique a mecânica: o que acontece com writes no primary quando todos os secondaries ultrapassam 30s de lag?

4. A equipe de dados usa Debezium (CDC via logical replication) para alimentar um data warehouse. Qual é o impacto de um **switchover planejado** mensal para exercitar DR? O que precisa ser feito no Debezium antes e depois do switchover?

5. Compare o custo de operação em "idle" entre: (a) RDS PostgreSQL Multi-AZ + 3 read replicas com db.r6g.large; (b) Aurora serverless com 1 writer + 2 readers (tiers 0 e 2) com max_capacity=32, min_capacity=0.5. Quais são as variáveis que determinam qual é mais econômico?

---

## Referências

- [FATO] [How Aurora serverless works — Amazon Aurora User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.how-it-works.html)
- [FATO] [Aurora serverless capacity (ACU ranges, platform versions) — Amazon Aurora User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.setting-capacity.html)
- [FATO] [Scaling to 0 ACUs (auto-pause) — Amazon Aurora User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2-auto-pause.html)
- [FATO] [Using Amazon Aurora Global Database — Amazon Aurora User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html)
- [FATO] [Switchover and failover in Aurora Global Database — Amazon Aurora User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-disaster-recovery.html)
- [FATO] [Amazon Aurora DB clusters overview — Amazon Aurora User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Overview.html)
