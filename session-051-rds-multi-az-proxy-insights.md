# Sessão 051 — RDS: Multi-AZ, Read Replicas, RDS Proxy e Performance Insights

**Duração estimada:** 60 min  
**Pré-requisito:** session-044 (Secrets Manager, rotação de credenciais RDS)

---

> ⚠️ **Nota de atualização crítica**: A AWS anunciou o fim do suporte ao console do **Performance Insights em 31 de julho de 2026** (em ~47 dias a partir da data de criação deste guia). O Performance Insights está sendo substituído pelo **CloudWatch Database Insights**. A API do Performance Insights é preservada sem alterações. Este guia cobre a transição.  
> Fonte: [AWS RDS User Guide — Performance Insights](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_PerfInsights.html)

---

## Objetivos da sessão

- Distinguir Multi-AZ DB Instance (1 standby, sincrônico, não legível) de Multi-AZ DB Cluster (2 readers semi-síncronos, legíveis) e de Read Replicas (assíncrono, legível, promoção manual)
- Entender o comportamento de failover de cada modelo e as implicações de lag de replicação
- Configurar RDS Proxy com connection pooling, multiplexing, e autenticação via IAM + Secrets Manager
- Identificar quando o RDS Proxy causa **pinning** e como evitá-lo
- Usar Performance Insights (e seu substituto Database Insights) para identificar a query responsável por um spike de DB Load

---

## 1. Modelos de alta disponibilidade e leitura no RDS

### 1.1 Comparação estrutural

[FATO] O RDS oferece três mecanismos distintos — com características ortogonais — para HA e escalabilidade de leitura:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                     VISÃO GERAL DOS 3 MODELOS                                   │
│                                                                                 │
│  ┌──────────────────────┐   ┌──────────────────────┐   ┌──────────────────────┐│
│  │ Multi-AZ DB Instance │   │ Multi-AZ DB Cluster  │   │   Read Replica       ││
│  │                      │   │                      │   │                      ││
│  │   Primary (R/W)      │   │   Writer (R/W)       │   │   Primary (R/W)      ││
│  │        │             │   │   /    \             │   │        │ (async)      ││
│  │   [sync]             │   │[semi-  [semi-        │   │   Replica (R/O)       ││
│  │        ↓             │   │ sync]   sync]        │   │   (legível)           ││
│  │   Standby (não       │   │   ↓         ↓        │   │   Pode ser promovido  ││
│  │   legível)           │   │ Reader1  Reader2     │   │   manualmente         ││
│  │                      │   │ (legíveis)           │   │                      ││
│  │ Propósito: HA        │   │ Propósito: HA + Read │   │ Propósito: Read Scale││
│  │ Failover: automático │   │ Failover: automático │   │ Failover: manual      ││
│  │ RPO: 0 (sincrônico)  │   │ RPO: quase 0         │   │ RPO: lag variável    ││
│  │ RTO: ~35s            │   │ RTO: menor que inst  │   │ RTO: alto (promoção) ││
│  └──────────────────────┘   └──────────────────────┘   └──────────────────────┘│
└─────────────────────────────────────────────────────────────────────────────────┘
```

[FATO] Tabela comparativa detalhada:

```
╔═══════════════════════════╦══════════════════╦═══════════════════╦═════════════════╗
║ Dimensão                  ║ Multi-AZ Instance║ Multi-AZ Cluster  ║ Read Replica    ║
╠═══════════════════════════╬══════════════════╬═══════════════════╬═════════════════╣
║ Topologia                 ║ 1 primary +      ║ 1 writer +        ║ 1+ replicas     ║
║                           ║ 1 standby        ║ 2 readers         ║                 ║
╠═══════════════════════════╬══════════════════╬═══════════════════╬═════════════════╣
║ Tipo de replicação        ║ Sincrônica       ║ Semi-sincrônica   ║ Assíncrona      ║
║                           ║ (antes do ACK)   ║ (1 reader confirma║ (eventual)      ║
╠═══════════════════════════╬══════════════════╬═══════════════════╬═════════════════╣
║ Standby/reader legível?   ║ NÃO              ║ SIM (readers)     ║ SIM             ║
╠═══════════════════════════╬══════════════════╬═══════════════════╬═════════════════╣
║ Failover automático?      ║ SIM (~35s)       ║ SIM (mais rápido) ║ NÃO (manual)    ║
╠═══════════════════════════╬══════════════════╬═══════════════════╬═════════════════╣
║ RPO (perda de dados)      ║ Zero             ║ Quase zero        ║ Lag variável    ║
║                           ║ (sincrônico)     ║ (1+ ack)          ║ (s a min)       ║
╠═══════════════════════════╬══════════════════╬═══════════════════╬═════════════════╣
║ Escala de leitura         ║ NÃO              ║ SIM (2 readers)   ║ SIM (até 5+)    ║
╠═══════════════════════════╬══════════════════╬═══════════════════╬═════════════════╣
║ Cross-region              ║ NÃO              ║ NÃO               ║ SIM             ║
╠═══════════════════════════╬══════════════════╬═══════════════════╬═════════════════╣
║ Promoção                  ║ Automática       ║ Automática        ║ Manual          ║
╠═══════════════════════════╬══════════════════╬═══════════════════╬═════════════════╣
║ Motores suportados        ║ Todos RDS        ║ MySQL, PostgreSQL ║ Todos RDS       ║
╠═══════════════════════════╬══════════════════╬═══════════════════╬═════════════════╣
║ Custo adicional           ║ 2× instância     ║ 3× instância      ║ 1× por réplica  ║
╠═══════════════════════════╬══════════════════╬═══════════════════╬═════════════════╣
║ Latência de escrita       ║ Maior que        ║ Maior que         ║ Sem impacto     ║
║                           ║ Single-AZ        ║ Single-AZ         ║ no primary      ║
╚═══════════════════════════╩══════════════════╩═══════════════════╩═════════════════╝
```

---

## 2. Multi-AZ DB Instance Deployment

### 2.1 Como funciona

[FATO] O RDS provisiona automaticamente uma réplica **standby sincrônica** em uma AZ diferente. Cada transação confirmada no primary deve ser replicada e confirmada no standby **antes** do ACK ser enviado ao cliente — isso garante RPO zero.

[FATO] O standby **não serve leitura** em nenhum momento antes da promoção. Ele existe exclusivamente para failover.

[FATO] Gatilhos de failover automático:
- Falha na instância primária (OS crash, hardware failure)
- Falha de rede no primary AZ
- Manutenção planejada (DB engine upgrade, patching)
- Comando manual `reboot-db-instance --force-failover`

[FATO] Tempo de failover típico: **menos de 35 segundos**. O endpoint DNS do cluster é atualizado, mas há propagação de DNS (~30s adicionais que o RDS Proxy elimina — ver seção 4).

[FATO] Tecnologia de replicação por engine:
- MariaDB, MySQL, Oracle, PostgreSQL, RDS Custom for SQL Server → **Amazon failover technology** (replicação proprietária da AWS)
- Microsoft SQL Server → **SQL Server Database Mirroring (DBM)** ou **Always On Availability Groups (AGs)**

[FATO] Multi-AZ aumenta a latência de writes em comparação ao Single-AZ devido à replicação sincrônica. A AWS recomenda **Provisioned IOPS** para workloads de produção Multi-AZ para minimizar essa latência.

### 2.2 Multi-AZ DB Cluster (1 writer + 2 readers)

[FATO] O Multi-AZ DB Cluster é uma variante diferente: 1 writer + 2 reader instances em 3 AZs distintas. Os readers **podem** servir leitura. Usa replicação semi-sincrônica (pelo menos 1 reader deve confirmar antes do ACK).

[FATO] No failover do Multi-AZ DB Cluster, o novo writer deve aguardar os readers aplicarem transactions não aplicadas (replica lag). O failover é mais rápido que o da instância simples.

[FATO] Suporte: MySQL e PostgreSQL apenas (não inclui SQL Server, Oracle, MariaDB).

---

## 3. Read Replicas

### 3.1 Modelo de replicação

[FATO] O RDS usa o mecanismo nativo de replicação do engine (MySQL binlog, PostgreSQL WAL streaming, etc.) para replicar **assincronamente** para as réplicas de leitura.

[FATO] Criação: o RDS tira um snapshot da instância source e cria a réplica a partir dele; a partir daí, a replicação aplica mudanças continuamente. A réplica fica em modo read-only.

[FATO] Restrições por engine:
- **Oracle (mounted mode)** e **Db2 (standby mode)**: não aceitam conexões de leitura — usadas apenas para DR cross-region
- **SQL Server, Oracle, Db2**: não suportam cascade (réplica de réplica)
- **MariaDB, MySQL, PostgreSQL (algumas versões)**: suportam cascade

[FATO] O RDS **não** suporta autoscaling de read replicas (ao contrário do Aurora). Criação e remoção são sempre manuais.

[FATO] Preço: mesma taxa da classe da instância. **Transferência de dados de replicação dentro da mesma região não é cobrada.** Cross-region: cobrança por dados transferidos.

### 3.2 Replica Lag

[FATO] A métrica CloudWatch `ReplicaLag` (em segundos) mede o atraso entre a última transação no writer e a última transação aplicada na réplica.

[FATO] Causas de lag elevado:
- Writes de alta taxa no primary (réplica não consegue acompanhar)
- Queries longas na réplica bloqueando a aplicação de replication
- Instância de réplica subdimensionada (CPU/IO)
- Eventos de manutenção (backup, parameter group changes)
- Rede congestionada (especialmente cross-region)

```
Monitorar lag via CloudWatch:
Namespace: AWS/RDS
Métrica:   ReplicaLag
Dimensão:  DBInstanceIdentifier = <replica-id>
Alarme:    > 300 segundos (5 min) → ação imediata
```

### 3.3 Promoção de Read Replica

[FATO] Promover uma read replica para instância standalone é uma operação **irreversível e manual**. Após a promoção:
- A réplica para de receber replicação
- Ela vira uma instância standalone com R/W
- Pode levar vários minutos (replica lag deve ser zerado primeiro)
- O binário de replicação (MySQL) ou o recovery WAL (PostgreSQL) continua disponível, mas a réplica não replicará mais do antigo primary

[CONSENSO] Promoção de read replica é usada como estratégia de Disaster Recovery (DR) quando o primary falha em outra região e não há mecanismo de failover automático. O RPO depende do lag no momento da falha.

---

## 4. RDS Proxy

### 4.1 Arquitetura e motivação

[FATO] O RDS Proxy é um **proxy gerenciado, multi-AZ, serverless** que fica entre a aplicação e o banco. Ele resolve dois problemas principais:

1. **Connection exhaustion**: lambdas, containers e microservices criam conexões efêmeras em alta velocidade. O banco tem um limite fixo de conexões simultâneas baseado em memória (ex: PostgreSQL padrão tem `max_connections = LEAST({DBInstanceClassMemory/9531392}, 5000)`). O Proxy mantém um pool menor de conexões com o banco e multiplexa conexões de múltiplos clientes.

2. **Failover sem reconexão**: durante um failover Multi-AZ, conexões existentes morrem e a aplicação precisa reconectar. O RDS Proxy absorve o failover internamente e redireciona as conexões para o novo primary sem que os clientes percam as conexões idle.

```
┌──────────────────────────────────────────────────────────────────────┐
│ FLUXO DE CONEXÃO COM RDS PROXY                                       │
│                                                                      │
│  App (Lambda, ECS, K8s)                                              │
│  ├─ 1000 conexões simultâneas → RDS Proxy endpoint                   │
│                                     │                                │
│                              ┌──────┴──────┐                         │
│                              │  POOL (50   │                         │
│                              │  conexões)  │                         │
│                              └──────┬──────┘                         │
│                                     │  multiplexing                  │
│                                     ↓                                │
│                          RDS DB Instance (max 50 conex)              │
│                                                                      │
│  Resultado: banco vê 50 conexões, não 1000                           │
└──────────────────────────────────────────────────────────────────────┘
```

### 4.2 Multiplexing e Pinning

[FATO] **Multiplexing** (connection reuse): por padrão, o Proxy reutiliza conexões ao nível de transação. Uma conexão do pool serve um cliente durante uma transação; ao final da transação (COMMIT/ROLLBACK), a conexão retorna ao pool para outro cliente. Com `autocommit=ON`, cada statement pode liberar a conexão.

[FATO] **Pinning**: quando o Proxy detecta que não é seguro compartilhar a conexão, ele a "pina" ao cliente pela duração da sessão inteira. Pinning é transparente para o cliente mas reduz o benefício do pool.

[FATO] Causas de pinning (PostgreSQL e MySQL):
- `SET` statements que alteram o estado da sessão (ex: `SET statement_timeout = 5000`)
- Transações abertas com `BEGIN`
- Uso de prepared statements (MySQL com `mysql_stmt_*`)
- Statements DDL (`CREATE TABLE`, `ALTER TABLE`)
- Funções que retornam estado de sessão (`LAST_INSERT_ID()`, `ROW_COUNT()`, `lastval()`)
- Statements com texto maior que **16 KB**
- `GET DIAGNOSTICS` (MariaDB/MySQL)

[FATO] Para detectar pinning: métrica CloudWatch `DatabaseConnectionsCurrentlySessionPinned` — valor alto indica que o pool está sendo subutilizado.

[CONSENSO] Para reduzir pinning: evitar `SET` locais desnecessários (usar **proxy initialization query** para configurações de sessão fixas), preferir ORM que use transações curtas, usar `INSERT...RETURNING` em vez de `lastval()` no PostgreSQL.

### 4.3 Limites do RDS Proxy

[FATO] Limites importantes:

```
╔══════════════════════════════════════════════════╦══════════════════╗
║ Limite                                           ║ Valor            ║
╠══════════════════════════════════════════════════╬══════════════════╣
║ Proxies por conta AWS                            ║ 20 (soft limit)  ║
║ Secrets Manager secrets por proxy                ║ 200              ║
║ (= máx. de usuários DB quando usando secrets)    ║                  ║
║ Endpoints adicionais por proxy                   ║ 20               ║
║ AZs do endpoint default                          ║ 2 (das subnets)  ║
║ Statement size para pinning                      ║ > 16 KB          ║
║ DB instances por proxy                           ║ 1                ║
║ Proxies por DB instance                          ║ Múltiplos (N)    ║
╚══════════════════════════════════════════════════╩══════════════════╝
```

[FATO] Restrições de rede:
- O Proxy deve estar no **mesmo VPC** que o banco
- O Proxy **não pode ser publicly accessible**
- Não funciona com VPC de tenancy `dedicated`
- Não funciona com VPC com `Enforce Mode` de encryption controls

[FATO] Em clusters Multi-AZ, o Proxy associa-se apenas ao **writer** (não às read replicas). Para leitura via Proxy em clusters, usar endpoints de leitura adicionais do Proxy.

### 4.4 Autenticação no RDS Proxy

[FATO] Três modos de autenticação:

```
MODO 1 — Database credentials apenas (sem IAM enforcement):
  App → (username/password) → Proxy → (SM secret) → DB
  Proxy recupera creds do Secrets Manager, não força IAM para o cliente

MODO 2 — Standard IAM Authentication (recomendado para aplicações):
  App → (token IAM via rds-db:connect) → Proxy → (SM secret) → DB
  Aplica IAM para cliente; Proxy faz auth no DB via Secrets Manager
  Funciona mesmo se o DB usa password nativa

MODO 3 — End-to-end IAM Authentication:
  App → (token IAM) → Proxy → (IAM token) → DB
  DB deve ter IAM database authentication habilitado
  Elimina necessidade de SM secrets para credenciais do banco
  NÃO suportado para SQL Server
```

[FATO] Para IAM auth (modos 2 e 3), o cliente deve gerar um token via:
```python
import boto3
token = boto3.client('rds').generate_db_auth_token(
    DBHostname='proxy-endpoint.proxy-xxx.us-east-1.rds.amazonaws.com',
    Port=5432,
    DBUsername='app_user',
    Region='us-east-1'
)
# Token tem validade de 15 minutos
```

[FATO] Ao usar Secrets Manager (modo 1 e 2): cria-se **um secret por usuário do banco**. Cada secret deve ter o formato:
```json
{"username": "app_user", "password": "supersecret"}
```

[FATO] O Proxy cria automaticamente o usuário `rdsproxyadmin` no banco. Não deletar nem modificar as permissões desse usuário — pode tornar o Proxy inoperante.

---

## 5. Performance Insights e Database Insights

### 5.1 Performance Insights — status atual (EOL jul/2026)

> ⚠️ **[FATO] O console do Performance Insights será encerrado em 31 de julho de 2026**. O console redirecionará para o CloudWatch Database Insights. A **API do Performance Insights permanece sem alterações**. Parâmetros de CloudFormation e Terraform continuam funcionando.

[FATO] Ao não agir, instâncias com Performance Insights serão migradas automaticamente para o **Standard mode** do Database Insights com o mesmo período de retenção.

[FATO] Modos do Database Insights:
- **Standard mode**: mesma experiência do Performance Insights hoje, mesmos preços de retenção flexível (1–24 meses)
- **Advanced mode**: monitoramento em nível de fleet, lock diagnostics, execution plan capture (mais caro)

### 5.2 Conceitos do Performance Insights / Database Insights

[FATO] O Performance Insights monitora **DB Load** = número médio de sessões ativas (Average Active Sessions / AAS) usando o motor de banco.

[FATO] DB Load é decomposto por:
- **Wait events**: o que as sessões estão aguardando (ex: `io/datafileread`, `Lock:relation`, `CPU`)
- **SQL statements**: quais queries contribuem para o load
- **Hosts**: de qual host vêm as conexões
- **Users**: qual usuário de banco está contribuindo

[FATO] A linha de referência visual é `max_vCPUs` — quando o DB Load ultrapassa o número de vCPUs, o banco está saturado.

[FATO] Métricas de Counter no Performance Insights (exemplos para PostgreSQL):
- `db.transactions.xact_commit` — commits por segundo
- `db.rows.tup_fetched` — rows retornadas por segundo
- `db.io.blks_hit` — cache hits (buffer pool)
- `db.io.blks_read` — disk reads (I/O físico)

[FATO] Período de retenção:
- Free tier: 7 dias
- Pago: 1–24 meses (preservados no Standard mode do Database Insights)

[FATO] API para leitura programática de métricas:
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

## 6. CDK Python — Stack completa com Multi-AZ, Read Replica, Proxy e Monitoring

```python
"""
CDK Stack: RDS PostgreSQL com Multi-AZ, Read Replica,
RDS Proxy (IAM auth), Performance Insights e alarmes.
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

        # Proxy aceita conexões da aplicação
        proxy_sg.add_ingress_rule(app_sg, ec2.Port.tcp(5432))
        # Banco aceita conexões apenas do proxy (não da aplicação diretamente)
        db_sg.add_ingress_rule(proxy_sg, ec2.Port.tcp(5432))

        # ──────────────────────────────────────────────────────────────
        # Secret (credenciais do banco)
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
            # Multi-AZ: standby sincrônico em AZ diferente
            multi_az=True,
            # Storage
            allocated_storage=100,
            storage_type=rds.StorageType.GP3,
            iops=3000,
            storage_encrypted=True,
            # Backups
            backup_retention=Duration.days(7),
            preferred_backup_window="03:00-04:00",
            # Manutenção
            preferred_maintenance_window="Sun:04:00-Sun:05:00",
            # Performance Insights (Standard mode / Database Insights)
            enable_performance_insights=True,
            performance_insight_retention=rds.PerformanceInsightRetention.DEFAULT,  # 7 dias
            # Monitoring
            monitoring_interval=Duration.seconds(60),
            # IAM auth: permite autenticação por token IAM além de password
            iam_database_authentication_enabled=True,
            # Deletion protection em prod
            deletion_protection=True,
            removal_policy=RemovalPolicy.RETAIN,
            # Parameter group customizado
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
        # Read Replica (mesma região, AZ diferente)
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
        # IAM Role para RDS Proxy
        # ──────────────────────────────────────────────────────────────
        proxy_role = iam.Role(self, "RdsProxyRole",
            assumed_by=iam.ServicePrincipal("rds.amazonaws.com"),
        )
        # Permite que o Proxy leia o secret do SM
        db_secret.grant_read(proxy_role)

        # ──────────────────────────────────────────────────────────────
        # RDS Proxy — autenticação Standard IAM (app → IAM → Proxy → SM → DB)
        # ──────────────────────────────────────────────────────────────
        proxy = rds.DatabaseProxy(self, "PgProxy",
            proxy_target=rds.ProxyTarget.from_instance(db_instance),
            secrets=[db_secret],
            vpc=vpc,
            vpc_subnets=ec2.SubnetSelection(
                subnet_type=ec2.SubnetType.PRIVATE_WITH_EGRESS
            ),
            security_groups=[proxy_sg],
            # Exige IAM auth dos clientes
            iam_auth=True,
            # Requer TLS
            require_tls=True,
            # Idle connection timeout (5-1800s)
            idle_client_timeout=Duration.minutes(30),
            # Inicialização de sessão (evita SET ad-hoc que causaria pinning)
            init_query="SET application_name='rds-proxy'",
            # Logs de debug (cuidado: verboso em prod)
            debug_logging=False,
        )

        # IAM policy para a aplicação conectar ao proxy via token
        app_connect_policy = iam.PolicyStatement(
            effect=iam.Effect.ALLOW,
            actions=["rds-db:connect"],
            resources=[
                # ARN do proxy user: arn:aws:rds-db:REGION:ACCOUNT:dbuser:PROXY-ID/USERNAME
                f"arn:aws:rds-db:{self.region}:{self.account}:dbuser:{proxy.db_proxy_arn.split(':')[-1]}/pgadmin",
            ],
        )

        # ──────────────────────────────────────────────────────────────
        # CloudWatch Alarms
        # ──────────────────────────────────────────────────────────────
        alert_topic = sns.Topic(self, "DbAlerts")

        # Alarme: ReplicaLag > 5 minutos
        cw.Alarm(self, "ReplicaLagAlarm",
            metric=cw.Metric(
                namespace="AWS/RDS",
                metric_name="ReplicaLag",
                dimensions_map={"DBInstanceIdentifier": read_replica.instance_identifier},
                period=Duration.minutes(1),
                statistic="Average",
            ),
            threshold=300,           # 5 minutos em segundos
            evaluation_periods=3,
            comparison_operator=cw.ComparisonOperator.GREATER_THAN_THRESHOLD,
            alarm_description="Replica lag > 5 minutos",
            treat_missing_data=cw.TreatMissingData.MISSING,
        ).add_alarm_action(cwa.SnsAction(alert_topic))

        # Alarme: DatabaseConnections (Proxy pinning indireta — muitas conexões ativas)
        cw.Alarm(self, "ProxyPinningAlarm",
            metric=cw.Metric(
                namespace="AWS/RDS",
                metric_name="DatabaseConnectionsCurrentlySessionPinned",
                dimensions_map={"ProxyName": proxy.db_proxy_name},
                period=Duration.minutes(5),
                statistic="Average",
            ),
            threshold=50,    # ajustar conforme baseline
            evaluation_periods=2,
            comparison_operator=cw.ComparisonOperator.GREATER_THAN_THRESHOLD,
            alarm_description="RDS Proxy pinning excessivo — revisar session SET statements",
        ).add_alarm_action(cwa.SnsAction(alert_topic))

        # Outputs
        CfnOutput(self, "ProxyEndpoint", value=proxy.endpoint)
        CfnOutput(self, "ReplicaEndpoint", value=read_replica.db_instance_endpoint_address)
        CfnOutput(self, "SecretArn", value=db_secret.secret_arn)
```

---

## 7. Python — Aplicação conectando via RDS Proxy com IAM token

```python
"""
Módulo de conexão ao RDS via Proxy usando IAM token.
Demonstra: geração de token, pool de conexões, e detecção de lag.
"""
import boto3
import psycopg2
from psycopg2 import pool
import logging
import time
from functools import lru_cache

logger = logging.getLogger(__name__)


# ──────────────────────────────────────────────────────────────────────
# Configuração
# ──────────────────────────────────────────────────────────────────────

PROXY_HOST   = "my-proxy.proxy-abc123.us-east-1.rds.amazonaws.com"
REPLICA_HOST = "postgres-read-replica.abc123.us-east-1.rds.amazonaws.com"
DB_PORT      = 5432
DB_NAME      = "appdb"
DB_USER      = "pgadmin"
AWS_REGION   = "us-east-1"


# ──────────────────────────────────────────────────────────────────────
# Geração de token IAM (válido 15 min — cachear por sessão)
# ──────────────────────────────────────────────────────────────────────

class IAMTokenManager:
    """Gerencia tokens IAM para autenticação no RDS Proxy."""

    def __init__(self, host: str, port: int, user: str, region: str):
        self.host = host
        self.port = port
        self.user = user
        self.region = region
        self._token: str | None = None
        self._token_expiry: float = 0

    def get_token(self) -> str:
        """Retorna token IAM, gerando novo se necessário (cache de 14 min)."""
        now = time.time()
        # Token dura 15 min; renovar 1 min antes para evitar expiração mid-connection
        if not self._token or now >= self._token_expiry - 60:
            rds_client = boto3.client("rds", region_name=self.region)
            self._token = rds_client.generate_db_auth_token(
                DBHostname=self.host,
                Port=self.port,
                DBUsername=self.user,
                Region=self.region,
            )
            self._token_expiry = now + 15 * 60
            logger.debug("IAM token renovado para %s", self.host)
        return self._token


# ──────────────────────────────────────────────────────────────────────
# Pool de conexões (Proxy já faz pooling real, mas o pool local
# evita overhead de TLS/TCP a cada request do Lambda/container)
# ──────────────────────────────────────────────────────────────────────

class RDSConnectionManager:
    """
    Gerencia conexão ao banco via RDS Proxy com IAM auth.
    Uso típico: instanciar uma vez no módulo (fora do handler Lambda)
    para reutilização entre invocações quentes (warm Lambda).
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
            password=token,   # token IAM como senha
            sslmode="require",
            connect_timeout=5,
        )
        conn.autocommit = True   # facilita multiplexing no Proxy
        return conn

    def get_connection(self) -> psycopg2.extensions.connection:
        """Retorna conexão ativa; reconecta se necessário."""
        try:
            if self._conn is None or self._conn.closed:
                self._conn = self._connect()
            else:
                # Ping leve para verificar que a conexão ainda está viva
                self._conn.cursor().execute("SELECT 1")
        except Exception:
            logger.warning("Reconectando ao banco...")
            self._conn = self._connect()
        return self._conn

    def execute(self, query: str, params: tuple = ()) -> list[dict]:
        """Executa uma query e retorna resultados como lista de dicts."""
        conn = self.get_connection()
        with conn.cursor() as cur:
            cur.execute(query, params)
            if cur.description:
                cols = [desc[0] for desc in cur.description]
                return [dict(zip(cols, row)) for row in cur.fetchall()]
        return []


# Singletons — instanciar fora do handler em Lambda (warm reuse)
writer_conn = RDSConnectionManager(PROXY_HOST,   is_writer=True)
reader_conn = RDSConnectionManager(REPLICA_HOST, is_writer=False)


# ──────────────────────────────────────────────────────────────────────
# Consulta de Replica Lag via CloudWatch
# ──────────────────────────────────────────────────────────────────────

def get_replica_lag_seconds(replica_identifier: str) -> float | None:
    """
    Retorna o lag atual da réplica em segundos.
    Retorna None se a métrica não estiver disponível.
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
        logger.info("Replica lag: %.1f segundos", lag)
        return lag
    return None


def route_query(query: str, params: tuple = (), max_lag_seconds: float = 30) -> list[dict]:
    """
    Roteia queries de leitura para a réplica se o lag for aceitável.
    Para writes ou se o lag for alto, usa o writer via proxy.
    """
    # Detectar se é uma query de leitura (heurística simples)
    is_read = query.strip().upper().startswith("SELECT")

    if is_read:
        lag = get_replica_lag_seconds("my-postgres-replica")
        if lag is not None and lag <= max_lag_seconds:
            logger.info("Roteando para réplica (lag=%.1fs)", lag)
            return reader_conn.execute(query, params)
        else:
            logger.warning("Lag alto (%.1fs) — roteando para writer", lag or 9999)

    return writer_conn.execute(query, params)


# ──────────────────────────────────────────────────────────────────────
# Consulta de DB Load via Performance Insights API
# ──────────────────────────────────────────────────────────────────────

def get_top_sql_by_db_load(
    db_resource_id: str,
    lookback_minutes: int = 60,
    top_n: int = 5,
) -> list[dict]:
    """
    Retorna as top N queries por DB Load médio no período.
    db_resource_id: obtido via describe-db-instances (.DbiResourceId)
    
    NOTA: a API do Performance Insights é preservada mesmo após
    a migração para Database Insights (EOL jul/2026).
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
        sql_text = key_dims.get("db.sql.statement", "N/A")[:200]   # truncar

        # Média do DB Load para essa query no período
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
    # Exemplo: listar top 5 queries por DB Load
    top_queries = get_top_sql_by_db_load(
        db_resource_id="db-ABCDEF1234567890",
        lookback_minutes=60,
    )
    for i, q in enumerate(top_queries, 1):
        print(f"\n#{i} — DB Load: {q['avg_db_load_aas']} AAS")
        print(f"  SQL: {q['sql'][:100]}...")
```

---

## 8. CLI — Operações completas

```bash
# ═══════════════════════════════════════════════════════════════
# Multi-AZ — Criação e failover
# ═══════════════════════════════════════════════════════════════

export DB_ID="postgres-prod"
export REGION="us-east-1"

# Criar instância Multi-AZ PostgreSQL
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

# Verificar secondary AZ e status
aws rds describe-db-instances \
  --db-instance-identifier "$DB_ID" \
  --query 'DBInstances[0].{Status:DBInstanceStatus,AZ:AvailabilityZone,SecondaryAZ:SecondaryAvailabilityZone,MultiAZ:MultiAZ}' \
  --output table

# Forçar failover manual (teste)
aws rds reboot-db-instance \
  --db-instance-identifier "$DB_ID" \
  --force-failover

# Aguardar disponibilidade após failover
aws rds wait db-instance-available \
  --db-instance-identifier "$DB_ID"

# Confirmar que AZ mudou (failover ocorreu)
aws rds describe-db-instances \
  --db-instance-identifier "$DB_ID" \
  --query 'DBInstances[0].{AZ:AvailabilityZone,SecondaryAZ:SecondaryAvailabilityZone}'

# ═══════════════════════════════════════════════════════════════
# Read Replica
# ═══════════════════════════════════════════════════════════════

# Criar read replica (mesma região)
aws rds create-db-instance-read-replica \
  --db-instance-identifier "${DB_ID}-replica" \
  --source-db-instance-identifier "$DB_ID" \
  --db-instance-class db.t3.large \
  --enable-performance-insights

# Monitorar ReplicaLag em tempo real (30s por ponto)
watch -n 30 "aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name ReplicaLag \
  --dimensions Name=DBInstanceIdentifier,Value=${DB_ID}-replica \
  --start-time \$(date -u -v-10M +%Y-%m-%dT%H:%M:%SZ) \
  --end-time \$(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 60 \
  --statistics Average \
  --query 'sort_by(Datapoints,&Timestamp)[-1].Average'"

# Promover read replica a instância standalone (DR)
aws rds promote-read-replica \
  --db-instance-identifier "${DB_ID}-replica" \
  --backup-retention-period 7 \
  --preferred-backup-window "03:00-04:00"

# Aguardar promoção
aws rds wait db-instance-available \
  --db-instance-identifier "${DB_ID}-replica"

echo "Réplica promovida — agora é uma instância standalone"

# ═══════════════════════════════════════════════════════════════
# RDS Proxy
# ═══════════════════════════════════════════════════════════════

export SECRET_ARN=$(aws secretsmanager describe-secret \
  --secret-id "rds/postgres-prod/pgadmin" \
  --query SecretArn --output text)

export PROXY_ROLE_ARN="arn:aws:iam::123456789012:role/rds-proxy-role"
export SUBNET_IDS="subnet-aaa,subnet-bbb,subnet-ccc"
export SG_ID="sg-proxy123"

# Criar proxy com Standard IAM auth
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

# Registrar o DB como target do proxy
aws rds register-db-proxy-targets \
  --db-proxy-name "${DB_ID}-proxy" \
  --db-instance-identifiers "$DB_ID"

# Aguardar proxy ficar disponível
aws rds wait db-proxy-available \
  --db-proxy-name "${DB_ID}-proxy"

# Verificar endpoint do proxy
aws rds describe-db-proxies \
  --db-proxy-name "${DB_ID}-proxy" \
  --query 'DBProxies[0].{Endpoint:Endpoint,Status:Status}'

# Verificar saúde dos targets
aws rds describe-db-proxy-targets \
  --db-proxy-name "${DB_ID}-proxy" \
  --query 'Targets[*].{TargetArn:TargetArn,RDSResourceId:RDSResourceId,State:TargetHealth.State,Reason:TargetHealth.Reason}'

# Testar conexão via proxy com IAM token
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

# Obter DbiResourceId (necessário para a PI API)
DB_RESOURCE_ID=$(aws rds describe-db-instances \
  --db-instance-identifier "$DB_ID" \
  --query 'DBInstances[0].DbiResourceId' \
  --output text)

echo "Resource ID: $DB_RESOURCE_ID"

# Top 5 queries por DB Load — última hora
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

# Top wait events — última hora
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

# Métricas de contador (I/O)
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

# Migração para Database Insights (Standard mode — sem alterações necessárias)
# DB instances com PI ativo serão migradas automaticamente em 31 jul 2026
# Para ativar Advanced mode (lock diagnostics, execution plans):
aws rds modify-db-instance \
  --db-instance-identifier "$DB_ID" \
  --database-insights-mode advanced \
  --apply-immediately
```

---

## 9. Armadilhas (Pitfalls)

[FATO] **Confundir Multi-AZ standby com read replica**: o standby do Multi-AZ DB Instance **não aceita leitura**. Tentativa de conexão direta ao standby falhará — não há endpoint separado. Para leitura com HA, usar Multi-AZ DB Cluster ou criar read replicas separadas.

[FATO] **Promover read replica antes de zerar o lag**: ao promover uma réplica durante DR, queries recentes no primary que ainda não foram replicadas serão perdidas. Verificar `ReplicaLag = 0` antes da promoção se o RPO for crítico.

[FATO] **RDS Proxy não suporta read replicas como target**: o Proxy associa-se somente ao writer. Para roteamento de leitura via Proxy, usar endpoints adicionais do Proxy apontando para os readers do cluster (Multi-AZ Cluster ou Aurora).

[FATO] **Pinning não declarado silenciosamente degrada o pool**: `SET` statements no código da aplicação (timezone, search_path, etc.) causam pinning sem erro visível — o pool funciona, mas com menos reutilização. Monitorar `DatabaseConnectionsCurrentlySessionPinned`. Mover configurações fixas para o `init_query` do Proxy.

[FATO] **Token IAM com validade de 15 minutos**: não gerar um novo token por request — isso anula o benefício do pool. O token deve ser cacheado e renovado ~1 minuto antes da expiração (ver IAMTokenManager acima).

[FATO] **`max_connections` do PostgreSQL baseado em memória**: em instâncias pequenas, o default pode ser surpreendentemente baixo. `db.t3.medium` tem ~1 GB de memória → `max_connections ≈ 107`. O RDS Proxy é especialmente crítico para Lambda/ECS que criam muitas conexões efêmeras.

[FATO] **Performance Insights EOL jul/2026**: dashboards e automações que usam o console do Performance Insights diretamente precisam ser migrados para o console do CloudWatch Database Insights antes de 31/07/2026. A **API `pi:*` não muda** — scripts e código existente continuam funcionando.

[FATO] **Multi-AZ não protege contra erros lógicos**: um `DROP TABLE` no primary é replicado sincronicamente para o standby. Multi-AZ protege contra falhas de hardware/AZ, não contra erros de aplicação. Para isso: backups + Point-in-Time Recovery (PITR).

---

## Exercício de Reflexão

Um sistema de e-commerce tem:
- `orders` DB (PostgreSQL 16, Multi-AZ DB Instance, db.r6g.xlarge)
- Uma read replica na mesma região para relatórios
- 500 Lambdas que processam pagamentos (cada invocação abre e fecha conexão com o banco)
- Um serviço de relatórios que roda queries analíticas pesadas (JOIN de 3 tabelas, ~30s de execução)

**Responda:**

1. Os 500 Lambdas estão causando `FATAL: remaining connection slots are reserved for non-replication superuser connections`. Qual é a causa raiz e como o RDS Proxy resolve? Qual seria o `max_connections` estimado de um `db.r6g.xlarge` (32 GB de RAM) com a fórmula `LEAST({DBInstanceClassMemory/9531392}, 5000)`?

2. O serviço de relatórios usa `SET work_mem = '256MB'` no início de cada sessão para melhorar o desempenho dos JOINs. Por que isso causaria pinning no RDS Proxy? Como você configuraria o Proxy para aplicar essa configuração sem causar pinning para **todas** as conexões?

3. Durante um incidente: o primary AZ `us-east-1a` sofreu uma falha de rede às 14:00. Às 14:01, o standby em `us-east-1b` foi promovido. Às 14:02, a equipe notou que as conexões da aplicação ainda estavam indo para o antigo endereço. Qual é a causa (DNS propagation delay) e como o RDS Proxy teria evitado a interrupção?

4. O Performance Insights mostrou que a query `SELECT * FROM orders JOIN order_items ON ... WHERE created_at > $1` contribui com 3.2 AAS de DB Load quando o sistema tem 4 vCPUs. O que esse número significa e qual seria a primeira análise a fazer no dashboard do Performance Insights para diagnosticar o gargalo?

5. Após 31 de julho de 2026, um script de monitoramento que usa `aws pi get-resource-metrics` para coletar DB Load continua funcionando? E o dashboard do console do Performance Insights?

---

## Referências

- [FATO] [Multi-AZ DB instance deployments — Amazon RDS User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZSingleStandby.html)
- [FATO] [Multi-AZ DB cluster deployments — Amazon RDS User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/multi-az-db-clusters-concepts.html)
- [FATO] [Working with DB instance read replicas — Amazon RDS User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReadRepl.html)
- [FATO] [Amazon RDS Proxy — Amazon RDS User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy.html)
- [FATO] [RDS Proxy concepts and terminology — Amazon RDS User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy.howitworks.html)
- [FATO] [Performance Insights EOL announcement + Database Insights — Amazon RDS User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_PerfInsights.html)
