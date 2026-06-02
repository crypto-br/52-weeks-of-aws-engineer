# Sessão 033 — DAX: arquitetura, casos de uso e quando NÃO usar

**Duração estimada:** 60 minutos  
**Dependências:** session-032-dynamodb-streams-lambda-cdc

---

## Objetivo

Ao final desta sessão, você conseguirá identificar os padrões de leitura onde DAX entrega ganho real (reads repetidas do mesmo item, workloads read-heavy), os casos onde DAX não ajuda ou prejudica (writes, strongly consistent reads, queries complexas, dados frios), calcular o custo-benefício de um cluster DAX frente ao custo das leituras que ele substitui, e provisionar um cluster DAX via CDK Python.

---

## Contexto

[FATO] Amazon DynamoDB Accelerator (DAX) é um cache in-memory totalmente gerenciado, compatível com a API DynamoDB, projetado para reduzir a latência de leituras de single-digit milliseconds para **microssegundos** — sem alterar o código da aplicação além da troca do cliente.

[CONSENSO] DAX não é um cache genérico: ele entende o modelo de dados do DynamoDB, distingue item cache de query cache, e implementa write-through para manter consistência entre cache e tabela. Essa especialização é sua vantagem e seu limite — ele não substitui Redis/ElastiCache para casos de uso que exigem estruturas de dados avançadas.

---

## Conceitos principais

### 1. Arquitetura do cluster DAX

[FATO] Um cluster DAX roda **dentro de uma VPC**. Ele consiste em um nó primário (leitura + escrita) e até 10 nós de réplica (somente leitura). A aplicação usa o **DAX Client** — um drop-in replacement do cliente DynamoDB padrão — que aponta para o **cluster endpoint**.

```
Aplicação (EC2 / Lambda na VPC)
        │
        ▼ DAX Client (endpoint: mycluster.abc123.dax-clusters.amazonaws.com)
┌───────────────────────────────────┐
│           DAX Cluster (VPC)       │
│  ┌─────────────┐  ┌────────────┐  │
│  │  Primário   │  │  Réplica 1 │  │
│  │ (lê + escreve)│ │ (lê só)   │  │
│  └──────┬──────┘  └────────────┘  │
│         │  replicação async       │
│  ┌──────▼──────┐  ┌────────────┐  │
│  │  Réplica 2  │  │  Réplica 3 │  │
│  └─────────────┘  └────────────┘  │
└───────────────────┬───────────────┘
                    │ cache miss / write-through
                    ▼
             DynamoDB Table
```

[FATO] O DAX Client faz **load balancing inteligente** entre os nós — direciona leituras para réplicas e escritas para o primário. A aplicação não precisa saber qual nó está sendo usado.

[FATO] DAX só suporta operações de **dados** — não de gerenciamento de tabela. `CreateTable`, `UpdateTable`, `DeleteTable`, `ListTables` etc. devem ir diretamente ao cliente DynamoDB padrão.

---

### 2. Item cache vs. Query cache

[FATO] DAX mantém dois caches completamente independentes:

```
┌─────────────────────────────────────────────────────────────┐
│                        DAX                                  │
│                                                             │
│  ┌──────────────────────┐   ┌───────────────────────────┐  │
│  │      Item Cache      │   │       Query Cache         │  │
│  │                      │   │                           │  │
│  │ GetItem / BatchGetItem│  │ Query / Scan              │  │
│  │                      │   │                           │  │
│  │ Chave: PK (+ SK)     │   │ Chave: parâmetros da      │  │
│  │ TTL default: 5 min   │   │ operação (KeyCondition,   │  │
│  │ Evicção: LRU         │   │ FilterExpr, Limit etc.)   │  │
│  │                      │   │ TTL default: 5 min        │  │
│  │                      │   │ Evicção: LRU              │  │
│  └──────────────────────┘   └───────────────────────────┘  │
│                                                             │
│  Escrita no item cache NÃO invalida o query cache           │
└─────────────────────────────────────────────────────────────┘
```

[FATO] **Item cache TTL default: 5 minutos** (configurável na criação do cluster — não pode ser alterado depois sem recriar o cluster).  
[FATO] **Query cache TTL default: 5 minutos** (igualmente configurável, independente do item cache TTL).  
[FATO] TTL = 0 para item cache: itens só são removidos por LRU eviction ou por write-through. TTL = 0 para query cache: nenhum resultado de Query/Scan é armazenado.

**Implicação crítica — inconsistência entre caches:**  
[FATO] Se um item é atualizado (write-through atualiza o item cache), o **query cache que contém esse item em um result set NÃO é invalidado**. A próxima Query pode retornar o result set antigo do query cache até ele expirar pelo TTL.

---

### 3. Write-through: o que acontece em cada operação

[FATO] Para operações de escrita (`PutItem`, `UpdateItem`, `DeleteItem`, `BatchWriteItem`), o fluxo é:

```
Aplicação
    │
    ▼ PutItem(item)
DAX Client
    │
    ├──▶ 1. Escreve no DynamoDB (sincronamente)
    │         │
    │         ├── Sucesso → 2. Atualiza item no item cache
    │         │                    │
    │         │                    └──▶ Retorna sucesso à aplicação
    │         │
    │         └── Falha (throttle, etc.) → NÃO escreve no cache
    │                                       Retorna erro à aplicação
    │
    └── (query cache NÃO é invalidado)
```

[FATO] Se o DynamoDB falhar (incluindo throttling), o item **não é escrito no cache**. Isso evita cache poisoning com dados que não foram persistidos.

---

### 4. Comportamento com strongly consistent reads e transações

[FATO] DAX **não serve** strongly consistent reads a partir do cache:

```
Operação                        Comportamento DAX
─────────────────────────────────────────────────────────────
GetItem (eventually consistent) Cache hit → retorna do cache
GetItem (strongly consistent)   Passa direto ao DynamoDB,
                                NÃO armazena resultado no cache
Query (eventually consistent)   Cache hit → retorna do cache
Query (strongly consistent)     Passa direto ao DynamoDB,
                                NÃO armazena resultado no cache
TransactGetItems                Passa direto ao DynamoDB,
                                NÃO armazena resultado no cache
TransactWriteItems              Passa direto ao DynamoDB,
                                NÃO atualiza o item cache
```

[FATO] `TransactWriteItems` não atualiza o item cache do DAX. Após uma transação bem-sucedida, o item cache pode conter a versão anterior dos itens até o TTL expirar ou um write-through posterior ocorrer.

---

### 5. Quando usar e quando NÃO usar DAX

[FATO — AWS Docs] Cenários adequados para DAX:

```
✅ Use DAX quando:
─────────────────────────────────────────────────────────────
Read-heavy (10:1 ou mais)   Alta razão reads/writes → alto hit rate
Dados quentes               Mesmos itens lidos repetidamente
Latência em microssegundos  SLA abaixo de 1ms para leituras
Bursts de tráfego           DAX absorve picos; DynamoDB escala atrás
> 3.000 RCU/item/partição   DAX rompe o limite por-item da partição
Eventually consistent OK    Tolerância a dados levemente desatualizados
```

[FATO — AWS Docs] Cenários **inadequados** para DAX:

```
❌ NÃO use DAX quando:
─────────────────────────────────────────────────────────────
Write-heavy                 Writes usam mais recursos DAX que reads;
                            benefício de cache é mínimo
Dados frios / long tail     Baixo hit rate → custo sem benefício
Strongly consistent reads   DAX passa ao DynamoDB; recursos desperdiçados
TransactGetItems/WriteItems Mesma razão: DAX bypassa o cache
Bulk scans (ETL)            Scan full-table deve ir direto ao DynamoDB
Compliance (ex: SOC)        DAX não tem todas as acreditações do DynamoDB
Dados de alta variabilidade Praticamente todos os acessos são cache miss
```

[OPINIÃO — AWS Docs] Para casos que precisam de cache mas com estruturas avançadas (listas, sorted sets, pub/sub, Lua scripting), a própria AWS recomenda **Amazon ElastiCache (Redis OSS)** como alternativa ao DAX.

---

### 6. Tipos de nó e sizing

[FATO] DAX oferece famílias de instâncias dedicadas à aceleração de DynamoDB:

```
Família     vCPU  Memória   Típico para
──────────────────────────────────────────────────────
dax.r4.large   2    15 GB   Dev/test, workloads leves
dax.r4.xlarge  4    30 GB   Produção pequena/média
dax.r4.4xlarge 16  122 GB   Alta escala
dax.r5.large   2    16 GB   Geração atual — substitui r4
dax.r5.xlarge  4    32 GB   Produção padrão
dax.r5.4xlarge 16  128 GB   Alta escala geração atual
dax.r5.24xlarge 96 768 GB   Workloads extremas
```

[FATO] A memória do nó determina quantos itens cabem no cache. O sizing correto exige estimar: tamanho médio dos itens quentes × número de itens únicos que cabem no working set.

[CONSENSO] Para alta disponibilidade em produção: **mínimo 3 nós** (1 primário + 2 réplicas), distribuídos em 3 AZs diferentes. Com 1 nó, qualquer falha causa downtime do DAX (a aplicação faz fallback ao DynamoDB, mas com latência maior).

---

## Exemplo prático

### Cenário: catálogo de produtos e-commerce — reads repetidas em itens populares

**Tabela:** `products-table` (pay-per-request)  
**Padrão:** 95% de leituras em ~5% dos produtos (produtos populares)  
**SLA:** p99 < 1ms para GetItem

#### CDK Python — Cluster DAX + tabela

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

        # ── VPC (usa a default ou cria uma nova) ──────────────────────
        vpc = ec2.Vpc.from_lookup(self, "VPC", is_default=True)

        # ── Tabela DynamoDB ───────────────────────────────────────────
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

        # ── IAM Role para o cluster DAX ───────────────────────────────
        dax_role = iam.Role(
            self, "DaxRole",
            assumed_by=iam.ServicePrincipal("dax.amazonaws.com"),
        )
        products_table.grant_read_write_data(dax_role)

        # ── Security Group para o cluster DAX ─────────────────────────
        dax_sg = ec2.SecurityGroup(
            self, "DaxSG",
            vpc=vpc,
            description="DAX cluster SG",
            allow_all_outbound=True,
        )
        # Permite acesso na porta DAX (8111 sem TLS, 9111 com TLS)
        # da subnet onde roda a aplicação
        dax_sg.add_ingress_rule(
            peer=ec2.Peer.ipv4(vpc.vpc_cidr_block),
            connection=ec2.Port.tcp(8111),
            description="DAX unencrypted access from VPC",
        )

        # ── Subnet group para o cluster ───────────────────────────────
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

        # ── Parameter group (TTLs customizados) ──────────────────────
        # item cache TTL: 10 min (produtos mudam pouco)
        # query cache TTL: 2 min (listas mudam mais)
        dax_param_group = dax.CfnParameterGroup(
            self, "DaxParamGroup",
            parameter_group_name="dax-products-params",
            parameter_name_values={
                "record-ttl-millis":    "600000",  # 10 min
                "query-ttl-millis":     "120000",  # 2 min
            },
        )

        # ── Cluster DAX (3 nós — HA multi-AZ) ────────────────────────
        dax_cluster = dax.CfnCluster(
            self, "DaxCluster",
            cluster_name="products-dax-cluster",
            node_type="dax.r5.large",
            replication_factor=3,          # 1 primário + 2 réplicas
            iam_role_arn=dax_role.role_arn,
            subnet_group_name=dax_subnet_group.subnet_group_name,
            security_group_ids=[dax_sg.security_group_id],
            parameter_group_name=dax_param_group.parameter_group_name,
            # SSE em repouso
            sse_specification=dax.CfnCluster.SSESpecificationProperty(
                sse_enabled=True,
            ),
            # Encryption in transit (TLS)
            cluster_endpoint_encryption_type="TLS",
            # Janela de manutenção
            preferred_maintenance_window="sun:05:00-sun:06:00",
        )
        dax_cluster.add_dependency(dax_subnet_group)
        dax_cluster.add_dependency(dax_param_group)
```

#### Python — Usando o DAX Client (boto3 + amazon-dax-client)

```python
# pip install amazon-dax-client
import amazondax
import boto3
from botocore.exceptions import ClientError

# ── Inicialização: DAX Client para reads/writes normais ──────────────
DAX_ENDPOINT = "daxs://products-dax-cluster.abc123.dax-clusters.amazonaws.com"

dax_client = amazondax.AmazonDaxClient.resource(
    endpoint_url=DAX_ENDPOINT,
    region_name="us-east-1",
)
dax_table = dax_client.Table("products-table")

# ── DynamoDB Client direto (para strongly consistent + transações) ───
dynamodb = boto3.resource("dynamodb", region_name="us-east-1")
ddb_table = dynamodb.Table("products-table")


def get_product(product_id: str) -> dict | None:
    """
    Leitura eventual — vai para o DAX (item cache).
    p99 em microssegundos após o warm-up do cache.
    """
    response = dax_table.get_item(
        Key={"PK": f"PRODUCT#{product_id}", "SK": "METADATA"}
    )
    return response.get("Item")


def get_product_consistent(product_id: str) -> dict | None:
    """
    Leitura strongly consistent — BYPASSA o DAX, vai ao DynamoDB.
    Use apenas quando precisar do valor mais recente garantido.
    NOTA: NÃO armazenado no cache do DAX após a leitura.
    """
    response = ddb_table.get_item(
        Key={"PK": f"PRODUCT#{product_id}", "SK": "METADATA"},
        ConsistentRead=True,
    )
    return response.get("Item")


def update_product_price(product_id: str, new_price: float) -> None:
    """
    Write-through: DAX escreve no DynamoDB primeiro,
    depois atualiza o item cache.
    Query cache com esse produto em result sets NÃO é invalidado.
    """
    dax_table.update_item(
        Key={"PK": f"PRODUCT#{product_id}", "SK": "METADATA"},
        UpdateExpression="SET price = :p, updatedAt = :ts",
        ExpressionAttributeValues={
            ":p": str(new_price),           # DynamoDB usa Decimal
            ":ts": "2024-01-15T10:00:00Z",
        },
    )


def list_products_by_category(category: str, limit: int = 20) -> list:
    """
    Query — vai para o query cache do DAX.
    Hit rate alto para categorias populares.
    ATENÇÃO: resultado pode ter até query-ttl-millis de staleness.
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
    Scan completo para ETL — usa DynamoDB DIRETAMENTE.
    Não passa pelo DAX: evita poluir o cache com dados frios
    e não desperdiça recursos do cluster.
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

#### CLI — Provisionar cluster e verificar métricas

```bash
# 1. Criar subnet group
aws dax create-subnet-group \
  --subnet-group-name dax-products-subnet-group \
  --subnet-ids subnet-abc123 subnet-def456 subnet-ghi789

# 2. Criar parameter group com TTLs customizados
aws dax create-parameter-group \
  --parameter-group-name dax-products-params

aws dax update-parameter-group \
  --parameter-group-name dax-products-params \
  --parameter-name-values \
      "ParameterName=record-ttl-millis,ParameterValue=600000" \
      "ParameterName=query-ttl-millis,ParameterValue=120000"

# 3. Criar cluster (3 nós, TLS, SSE)
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

# 4. Verificar status do cluster
aws dax describe-clusters \
  --cluster-names products-dax-cluster \
  --query 'Clusters[0].{Status:Status, Endpoint:ClusterDiscoveryEndpoint, Nodes:Nodes[*].{Id:NodeId,Status:NodeStatus,AZ:AvailabilityZone}}'

# 5. Monitorar cache hit ratio (métrica chave de saúde do DAX)
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

# 6. Calcular hit rate manualmente:
#    HitRate = ItemCacheHits / (ItemCacheHits + ItemCacheMisses)
#    Alvo saudável: > 80% para workloads read-heavy

# 7. Verificar throttling no cluster
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

## Armadilhas comuns

**1. Query cache não é invalidado por writes**  
[FATO] Write-through atualiza o **item cache** do item alterado. Mas se uma Query anterior cacheou um result set que continha esse item, esse result set permanece no **query cache** até expirar pelo TTL. Aplicações que requerem consistência imediata entre escrita e resultado de Query devem ir direto ao DynamoDB ou usar TTL baixíssimo no query cache.

**2. TransactWriteItems não atualiza o item cache**  
[FATO] Operações transacionais passam direto ao DynamoDB e não interagem com o cache do DAX. Após um `TransactWriteItems` bem-sucedido, o DAX ainda pode servir a versão anterior dos itens afetados até o TTL expirar. Se isso for inaceitável, leia com strongly consistent após a transação (diretamente pelo DynamoDB client).

**3. Usar DAX Client para scans de ETL**  
[CONSENSO] Scans completos de tabela via DAX desperdiçam recursos do cluster (poluem o query cache com dados que não serão reutilizados) e aumentam latência para outras requisições. Use sempre o DynamoDB client padrão para operações batch/ETL.

**4. Cluster em single AZ sem réplicas**  
[CONSENSO] Um cluster com `replication-factor=1` (apenas o nó primário) oferece zero de HA. Qualquer manutenção ou falha torna o cluster indisponível. O DAX Client faz fallback ao DynamoDB, mas há uma janela de latência elevada durante a recuperação. Em produção: mínimo 3 nós em 3 AZs.

**5. TTL do item cache muito alto para dados que mudam com frequência**  
[CONSENSO] O default de 5 minutos é razoável para muitos casos, mas dados com alta taxa de update (ex: inventário em tempo real, preços de flash sale) exigem TTL menor ou uso direto do DynamoDB. O custo de staleness deve ser avaliado contra o benefício de latência.

**6. DAX Client não suporta todas as operações DynamoDB**  
[FATO] Operações de controle de tabela (`CreateTable`, `DescribeTable`, `UpdateTable`, etc.) não são suportadas pelo DAX Client. A aplicação precisa manter dois clientes: DAX Client para operações de dados e DynamoDB Client direto para operações de plano de controle.

---

## Exercício de reflexão

Você está avaliando se adicionar DAX a uma aplicação de e-commerce que tem as seguintes características:

- 500.000 GetItem/s em horário de pico (produtos populares: top 1.000 SKUs respondem por 80% das leituras)  
- 10.000 UpdateItem/s (atualizações de inventário em tempo real)  
- SLA: p99 < 5ms para leituras de catálogo, p99 < 50ms para leituras de inventário  
- Inventário deve refletir mudanças em até 1 segundo (para evitar oversell)

Responda:

1. Para qual das duas operações (catálogo vs inventário) DAX é mais adequado e por quê? O que determina esse trade-off?

2. O SLA de inventário exige que reads reflitam mudanças em até 1 segundo. Você poderia usar DAX para essas leituras? Se sim, como configuraria o TTL? Se não, qual seria a alternativa?

3. Considerando que o cluster DAX (3x `dax.r5.xlarge`) custa aproximadamente $0.65/hora por nó (total ~$1.95/hora ou ~$1.400/mês), e que a tabela sem DAX consumiria aproximadamente 500.000 RCU/s (~$5.400/mês em pay-per-request), qual seria o ponto de break-even mínimo de cache hit rate para o DAX ser financeiramente justificável?

---

## Recursos para aprofundar

- [FATO] DAX: How it works (item cache, query cache, write-through): https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DAX.concepts.html
- [FATO] Evaluating DAX suitability: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/evaluate-dax-suitability.html
- [FATO] DAX and DynamoDB consistency models: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DAX.consistency.html
- [FATO] DAX cluster sizing guide: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DAX.sizing-guide.html
- [FATO] DAX metrics and dimensions (CloudWatch): https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/dax-metrics-dimensions-dax.html

---

