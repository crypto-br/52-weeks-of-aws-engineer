# Sessão 31 — DynamoDB: GSIs e LSIs, hot partitions e write amplification

**Duração estimada:** 60 minutos
**Pré-requisitos:** session-030-dynamodb-single-table-adjacency

---

## Objetivo

Ao final, você conseguirá projetar um GSI que resolve um access pattern secundário sem criar
uma hot partition, calcular o custo de write amplification ao escrever em uma tabela com
múltiplos GSIs, e decidir quando uma LSI é preferível a um GSI (trade-off de flexibilidade
vs custo por item).

---

## Contexto

[FATO] Os índices secundários são a principal ferramenta para suportar access patterns que não
podem ser atendidos pela chave primária da tabela. Sem eles, a única alternativa seria o `Scan`
— uma varredura completa da tabela que consome throughput proporcional ao tamanho total dos dados,
independente de quantos itens são relevantes para a query. Um GSI bem projetado transforma uma
query O(n) em O(log n) com custo proporcional apenas ao resultado.

[FATO] O custo dos índices secundários é real e mensurável: cada GSI replica dados (storage) e
consome WCUs adicionais em cada write da tabela base (write amplification). Entender esses custos
é essencial para projetar schemas que escalam sem surpresas de fatura ou throttling.

---

## Conceitos principais

### 1. GSI — anatomy e modelo de consistência

[FATO] Um **Global Secondary Index (GSI)** é um índice com PK e SK que podem ser completamente
diferentes dos da tabela base. É "global" porque pode indexar itens de qualquer partição da
tabela base — um query no GSI atravessa todas as partições.

```
Tabela base: Orders
  PK = order_id (String)
  SK = customer_id (String)

GSI: OrdersByStatusDate
  PK = status (String)   ← diferente da tabela base
  SK = order_date (String)
```

**Propriedades fundamentais do GSI:**

```
1. PK e SK independentes da tabela base (podem ser qualquer atributo top-level String/Number/Binary)
2. Consistência EVENTUAL por padrão (GSI é atualizado assincronamente após write na tabela base)
3. Sem GetItem — somente Query e Scan são suportados em GSIs
4. Limite padrão de 20 GSIs por tabela (aumentável via quota request)
5. Throughput provisionado SEPARADO da tabela base (em modo provisioned)
6. GSI herda o table class da tabela base
7. Chave do GSI não precisa ser única — múltiplos itens podem ter o mesmo PK+SK no GSI
8. Se um item não tem o atributo do GSI key, ele NÃO é projetado no índice (sparse index)
```

[FATO] A consistência eventual do GSI tem implicação prática importante: logo após um write na
tabela base, uma query no GSI pode não retornar o item atualizado. Em cenários onde a leitura
imediata após write é crítica, o código deve ler da tabela base (que suporta `ConsistentRead=True`),
não do GSI.

**Sparse index — quando a ausência de atributo é uma feature:**

[FATO] Se o atributo do GSI key não existir no item, o item **não é projetado no GSI**. Isso
permite criar índices que cobrem apenas um subconjunto da tabela:

```
Tabela: Tasks
  PK = task_id
  SK = task_id
  status (atributo opcional — só existe quando status = "OPEN")

GSI: OpenTasksIndex
  PK = status

Itens com status="OPEN" → projetados no GSI
Itens concluídos (sem atributo status) → NÃO projetados no GSI

Resultado: o GSI contém APENAS tasks abertas, sem custo de storage para as concluídas.
Query "listar tasks abertas" → Query GSI, pequeno, eficiente.
```

Este padrão é chamado de **sparse index** e é um dos mais valiosos em DynamoDB: o GSI só armazena
itens com um determinado estado, reduzindo storage e throughput de leitura.

### 2. Write amplification — calculando o custo real de múltiplos GSIs

[FATO] Cada write na tabela base (PutItem, UpdateItem, DeleteItem) pode disparar writes adicionais
em cada GSI. O número exato de writes por GSI depende do que mudou:

```
Cenário                                          Writes adicionais no GSI
─────────────────────────────────────────────────────────────────────────
Item novo que define o atributo GSI key         +1 write (insere no GSI)
Update que muda o VALOR de um atributo GSI key  +2 writes (deleta antigo + insere novo)
Update que deleta um atributo GSI key           +1 write (deleta do GSI)
Item não tinha o atributo e ainda não tem       +0 writes
Update de atributo projetado (não GSI key)      +1 write (atualiza projeção no GSI)
```

**Cálculo de write amplification com N GSIs:**

```
Tabela com 3 GSIs, cada item tem os 3 atributos de GSI key definidos:

PutItem → 1 write (tabela base) + 3 writes (1 por GSI) = 4 writes totais
UpdateItem mudando os 3 atributos GSI key → 1 write + 3×2 writes = 7 writes totais
```

[FATO] O custo em WCUs de cada write no GSI é calculado pelo **tamanho do item projetado no GSI**
(arredondado para cima para o próximo KB), não pelo tamanho do item na tabela base. Isso significa:
projeção `KEYS_ONLY` minimiza o WCU por write de GSI; projeção `ALL` maximiza.

**Exemplo numérico:**

```
Tabela com on-demand pricing, item de 3 KB, 3 GSIs:
  - GSI1: projeção KEYS_ONLY (item projetado = 200 bytes → 1 WCU)
  - GSI2: projeção INCLUDE (item projetado = 1.5 KB → 2 WCUs)
  - GSI3: projeção ALL (item projetado = 3 KB → 3 WCUs)

PutItem (novo item):
  Tabela base: 3 KB → 3 WCUs
  GSI1: 200 bytes → 1 WCU
  GSI2: 1.5 KB → 2 WCUs
  GSI3: 3 KB → 3 WCUs
  ─────────────────────────
  Total: 9 WCUs por PutItem  (3x o custo sem GSIs)
```

[FATO] No modo **on-demand**, não há WCU provisionado explicitamente — o DynamoDB escala
automaticamente. Mas o custo por write request unit ainda multiplica pelo número de GSIs afetados.
Verificar o custo em `aws.amazon.com/dynamodb/pricing` para a região e tabela class.

### 3. GSI throttling e back-pressure — o efeito cascata mais perigoso

[FATO] O mecanismo de back-pressure é o aspecto mais contraintuitivo do GSI: **se um GSI não
tem capacidade suficiente para processar updates, o DynamoDB throttle os WRITES NA TABELA BASE**,
não apenas queries no GSI. Isso significa que um GSI subdimensionado pode derrubar toda a
capacidade de escrita da sua tabela.

```
Tabela base: 1.000 WCU provisionados (suficiente para o workload)
GSI1: 100 WCU provisionados (subdimensionado)

Workload: 500 writes/s na tabela, todos afetam GSI1

Resultado:
  Tabela base: OK (500 WCU < 1.000)
  GSI1: THROTTLED (500 WCU > 100)
  → Back-pressure: writes na tabela base também começam a throttle
  → ProvisionedThroughputExceededException nos writes da tabela base
     (ResourceArn aponta para o GSI1, não para a tabela — confusão garantida)
```

[FATO] Há quatro tipos distintos de throttling em GSIs:

```
1. IndexWriteProvisionedThroughputExceeded
   Causa: WCU provisionado do GSI insuficiente (modo provisioned)
   Fix: aumentar WCU do GSI

2. IndexWriteMaxOnDemandThroughputExceeded
   Causa: limite máximo de throughput configurado no GSI on-demand excedido
   Fix: aumentar o max throughput configurado ou remover o limite

3. IndexWriteKeyRangeThroughputExceeded
   Causa: HOT PARTITION no GSI — uma única partição do GSI excede o limite
          físico (~3.000 WCU/s por partição), mesmo com WCU total suficiente
   Fix: redesenhar a PK do GSI para melhor distribuição (write sharding)

4. IndexWriteAccountLimitExceeded
   Causa: tabela ultrapassou o limite regional de throughput da conta
   Fix: solicitar aumento de quota via Service Quotas
```

### 4. Hot partition no GSI — diagnóstico e solução com write sharding

[FATO] Uma **hot partition** em GSI acontece quando a PK do GSI tem baixa cardinalidade (poucos
valores distintos), concentrando todos os writes em poucas partições físicas do índice. O caso
clássico é usar um atributo de status como PK do GSI:

```
GSI: OrdersByStatus
  PK = status   → valores: "OPEN", "PROCESSING", "CLOSED"

Se 80% dos pedidos têm status="PROCESSING":
  80% dos writes no GSI vão para a partição "PROCESSING"
  Essa partição pode receber 4.000 writes/s → excede o limite de ~3.000 WCU/s por partição
  → ThrottlingException com IndexWriteKeyRangeThroughputExceeded
```

**Solução: write sharding no GSI key**

A técnica padrão é adicionar um sufixo aleatório (shard) ao valor do PK do GSI:

```
Sem sharding:         GSI PK = "PROCESSING"
Com 10 shards:        GSI PK = "PROCESSING#" + str(random.randint(0, 9))
                      → valores: "PROCESSING#0" a "PROCESSING#9"
```

Ao escrever, distribui-se aleatoriamente entre os shards. Ao ler, faz-se N queries em paralelo
(uma por shard) e consolida os resultados no cliente:

```python
import asyncio
import boto3
from boto3.dynamodb.conditions import Key

NUM_SHARDS = 10
table = boto3.resource("dynamodb").Table("orders")

# Write: escolhe shard aleatório
import random
shard = random.randint(0, NUM_SHARDS - 1)
table.put_item(Item={
    "order_id": "ord123",
    "status_shard": f"PROCESSING#{shard}",  # PK do GSI com shard
    "order_date": "2024-04-01",
    # ... outros atributos
})

# Read: query em todos os shards em paralelo
def query_shard(shard_num: int) -> list:
    response = table.query(
        IndexName="OrdersByStatusShard",
        KeyConditionExpression=(
            Key("status_shard").eq(f"PROCESSING#{shard_num}") &
            Key("order_date").begins_with("2024-04")
        ),
    )
    return response.get("Items", [])

# Paralelismo com ThreadPoolExecutor (boto3 não é async-native)
from concurrent.futures import ThreadPoolExecutor
with ThreadPoolExecutor(max_workers=NUM_SHARDS) as executor:
    futures = [executor.submit(query_shard, i) for i in range(NUM_SHARDS)]
    all_items = []
    for future in futures:
        all_items.extend(future.result())
```

[FATO] O número de shards deve ser calibrado: poucos shards não resolvem a hot partition; muitos
shards aumentam a latência da query paralela (mais roundtrips em paralelo). Uma regra prática:
`num_shards = ceil(peak_writes_per_second / 800)` — reservando margem em relação ao limite de
~1.000 WCU/s por partição que é considerado seguro para evitar throttling esporádico.

### 5. LSI — Local Secondary Index: alternativa para sort key alternativo

[FATO] Um **Local Secondary Index (LSI)** mantém a mesma PK da tabela base, mas com um SK
diferente. É "local" porque cada partição do LSI é co-localizada com a partição da tabela base
com o mesmo valor de PK. Isso garante que um query no LSI sempre é **fortemente consistente**
quando `ConsistentRead=True`.

```
Tabela base: Thread
  PK = ForumName (String)
  SK = Subject (String)

LSI: LastPostIndex
  PK = ForumName (String)  ← MESMO que a tabela base — obrigatório
  SK = LastPostDateTime (String)  ← diferente da tabela base
```

**Restrições do LSI:**

```
1. SOMENTE criado no momento da criação da tabela (CreateTable)
   Não pode ser adicionado posteriormente — ao contrário do GSI
2. PK deve ser idêntica à PK da tabela base
3. SK deve ser um único atributo escalar (String, Number, Binary)
4. Máximo de 5 LSIs por tabela
5. Item collection limit: 10 GB por valor de PK (tabela + todos os LSIs)
   → Se uma partition key acumular > 10 GB entre tabela e LSIs: ItemCollectionSizeLimitExceededException
```

[FATO] O limite de 10 GB por item collection é a restrição mais crítica do LSI e a principal razão
para preferir GSIs em entidades que crescem indefinidamente. Em contrapartida, o LSI compartilha
o throughput provisionado da tabela base (sem custo adicional de WCU por índice) e suporta leitura
fortemente consistente — vantagens que o GSI não oferece.

**Fetch de atributos não projetados no LSI:**

[FATO] Uma propriedade única do LSI (não disponível no GSI): se uma query no LSI precisar de
atributos não projetados no índice, o DynamoDB faz **fetch automático** na tabela base. Isso é
transparente para o código, mas tem custo: cada item que requer fetch consome RCUs adicionais —
calculados pelo tamanho do item na tabela base (arredondado para 4 KB), não pelo tamanho do
item no LSI. Em GSIs, isso não é possível — o GSI não pode acessar a tabela base.

```python
# Query no LSI com atributo não projetado
response = table.query(
    IndexName="LastPostIndex",
    ConsistentRead=True,   # suportado em LSI, não em GSI
    KeyConditionExpression=(
        Key("ForumName").eq("EC2") &
        Key("LastPostDateTime").between("2024-01-01", "2024-12-31")
    ),
    ProjectionExpression="Subject, Replies, LastPostDateTime, Tags",
    # Tags não está projetado no índice → DynamoDB faz fetch na tabela base
    # Custo: RCUs do LSI + RCUs do fetch (por item, arredondado para 4 KB)
)
```

### 6. GSI vs LSI — tabela de decisão

```
Critério                         GSI                      LSI
─────────────────────────────────────────────────────────────────────────────
PK do índice                     Qualquer atributo        Mesma da tabela base
SK do índice                     Qualquer atributo        Qualquer atributo
Quando pode ser criado           A qualquer momento       Somente na criação da tabela
Quando pode ser deletado         A qualquer momento       Somente ao deletar a tabela
Consistência de leitura          Eventual (apenas)        Eventual OU Forte
Throughput                       Separado da tabela       Compartilhado com a tabela
Limite de tamanho por PK         Sem limite               10 GB (tabela + todos LSIs)
Fetch de atributos não projetados Não suportado           Suportado (com custo extra)
Sparse index                     Suportado               Suportado
Máximo por tabela                20 (aumentável)         5 (fixo)
Custo de storage                 WCU + storage próprios  Storage compartilhado
```

**Quando preferir LSI:**
- Acesso alternativo ao sort key com **mesma partição** (ex: ordenar posts por data ao invés de título, dentro de um mesmo fórum)
- Necessidade de **leitura fortemente consistente** no índice
- Item collection comprovadamente pequeno (< 5 GB) e sem previsão de crescimento para 10 GB
- Aplicação já existe com tabela criada e LSI planejado desde o início

**Quando preferir GSI:**
- Acesso por atributo de entidade diferente (ex: buscar pedidos pelo email do cliente)
- Item collection que pode crescer indefinidamente
- Necessidade de adicionar o índice após a tabela já estar em produção

---

## Exemplo prático

**Cenário:** plataforma de e-commerce com tabela `Orders`. Access patterns:
1. Buscar pedido por ID (PK da tabela)
2. Listar pedidos de um cliente por data (GSI: customer_id + order_date)
3. Listar pedidos abertos por data (GSI sparse: status + order_date, somente itens OPEN)
4. Listar pedidos por representante comercial no último mês (GSI: rep_id + order_date)

### CDK Python — tabela com 3 GSIs

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

        # GSI 1: pedidos por cliente (AP2)
        # PK = customer_id, SK = order_date → boa distribuição (muitos clientes)
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

        # GSI 2: pedidos abertos (AP3) — sparse index
        # status só existe quando = "OPEN"; pedidos fechados não são projetados
        # ATENÇÃO: PK = status tem baixa cardinalidade → usar sharding em alto volume
        table.add_global_secondary_index(
            index_name="OpenOrdersByDate",
            partition_key=dynamodb.Attribute(
                name="open_status_shard",  # "OPEN#0" a "OPEN#9"
                type=dynamodb.AttributeType.STRING,
            ),
            sort_key=dynamodb.Attribute(
                name="order_date",
                type=dynamodb.AttributeType.STRING,
            ),
            projection_type=dynamodb.ProjectionType.INCLUDE,
            non_key_attributes=["customer_id", "total_amount", "rep_id"],
        )

        # GSI 3: pedidos por rep comercial (AP4)
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

### boto3 — write com sharding no GSI sparse e cálculo de amplification

```python
import boto3
import random
from boto3.dynamodb.conditions import Key
from concurrent.futures import ThreadPoolExecutor
from datetime import datetime, timezone

dynamodb = boto3.resource("dynamodb", region_name="us-east-1")
table = dynamodb.Table("orders")

NUM_SHARDS = 10  # calibrar conforme peak writes/s


def create_order(order_id: str, customer_id: str, rep_id: str, total: float) -> None:
    """
    Write amplification para este PutItem:
      - Tabela base: 1 write
      - GSI OrdersByCustomerDate: +1 write (customer_id e order_date definidos)
      - GSI OpenOrdersByDate: +1 write (open_status_shard definido → pedido aberto)
      - GSI OrdersByRepDate: +1 write (rep_id e order_date definidos)
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
        "open_status_shard": f"OPEN#{shard}",  # atributo do GSI sparse com sharding
        "total_amount": str(total),
        "items_count": 0,
    })


def close_order(order_id: str) -> None:
    """
    Update amplification para fechar um pedido:
      - Tabela base: 1 write
      - GSI OpenOrdersByDate:
          +1 write para DELETAR open_status_shard do GSI (item sai do sparse index)
          → REMOVE open_status_shard do item na tabela base
      - GSIs OrdersByCustomerDate e OrdersByRepDate: 0 writes adicionais (chave não muda)
    Total: 2 writes

    Nota: ao remover o atributo open_status_shard, o item deixa de existir no GSI sparse.
    """
    table.update_item(
        Key={"order_id": order_id},
        UpdateExpression="SET #s = :closed REMOVE open_status_shard",
        ExpressionAttributeNames={"#s": "status"},
        ExpressionAttributeValues={":closed": "CLOSED"},
    )


# AP3: listar pedidos abertos em paralelo nos shards
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

    # Consolidar e ordenar por data (cada shard retorna ordenado, merge manual)
    all_items = [item for shard_items in results for item in shard_items]
    return sorted(all_items, key=lambda x: x["order_date"], reverse=True)
```

### CLI — verificar throttling e tamanho de item collection

```bash
# Verificar throughput consumido por GSI (CloudWatch)
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB \
  --metric-name ConsumedWriteCapacityUnits \
  --dimensions Name=TableName,Value=orders Name=GlobalSecondaryIndexName,Value=OpenOrdersByDate \
  --start-time 2024-04-01T00:00:00Z \
  --end-time 2024-04-01T01:00:00Z \
  --period 60 \
  --statistics Sum

# Verificar throttling por GSI
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB \
  --metric-name WriteThrottleEvents \
  --dimensions Name=TableName,Value=orders Name=GlobalSecondaryIndexName,Value=OpenOrdersByDate \
  --start-time 2024-04-01T00:00:00Z \
  --end-time 2024-04-01T01:00:00Z \
  --period 60 \
  --statistics Sum

# Monitorar tamanho da item collection (para tabelas com LSI)
# Usar ReturnItemCollectionMetrics=SIZE no PutItem/UpdateItem
aws dynamodb put-item \
  --table-name forum-threads \
  --item '{"ForumName":{"S":"EC2"}, "Subject":{"S":"test"}, ...}' \
  --return-item-collection-metrics SIZE

# Descrever GSIs de uma tabela
aws dynamodb describe-table \
  --table-name orders \
  --query 'Table.GlobalSecondaryIndexes[*].{Name:IndexName, Status:IndexStatus, WCU:ProvisionedThroughput.WriteCapacityUnits}'
```

---

## Armadilhas comuns

### Armadilha 1: GSI subdimensionado throttle a tabela base (back-pressure silencioso)

A armadilha mais insidiosa: você provisiona 1.000 WCU na tabela base e apenas 50 WCU no GSI.
O workload aumenta para 200 writes/s, cada um afetando o GSI. O GSI fica throttled (200 > 50),
e o DynamoDB começa a throttle os writes da **tabela base** para proteger a consistência do GSI.
O `ResourceArn` na exceção aponta para o GSI, mas o código que falha está escrevendo na tabela
base — o desenvolvedor fica confuso. A regra documentada pela AWS: o WCU provisionado do GSI deve
ser **igual ou maior** que o WCU da tabela base (porque cada write na tabela base pode gerar um
write no GSI). Em modo on-demand, o back-pressure ainda existe — o GSI tem seu próprio limite
de throughput máximo configurável.

### Armadilha 2: projeção ALL em GSI de tabela com itens grandes

Projeção `ALL` no GSI significa que cada item da tabela base é completamente replicado no índice.
Para uma tabela com itens de 10 KB e 10 milhões de itens, um GSI com `ALL` adiciona ~100 GB de
storage e multiplica o custo de cada write pelo tamanho do item (10 KB → 10 WCUs de write por
GSI, por write na tabela base). A alternativa é `INCLUDE` com apenas os atributos necessários
para a query. Se o código faz `ProjectionExpression` no GSI para buscar apenas 3 atributos, esses
3 atributos devem estar projetados — não faz sentido usar `ALL` para depois selecionar 3 campos.

### Armadilha 3: criar LSI depois de verificar que a tabela já existe em produção

O LSI só pode ser criado no `CreateTable`. Se você perceber, 6 meses depois do lançamento, que
precisa de um sort key alternativo com leitura fortemente consistente na mesma partição, não é
possível adicionar um LSI — a única opção é criar uma nova tabela, migrar os dados, e atualizar
o código. A alternativa disponível é um GSI (que pode ser adicionado a qualquer momento), mas
GSI não suporta leitura fortemente consistente. Para evitar essa situação, o correto é planejar
todos os LSIs necessários **antes** de criar a tabela em produção, mesmo que os access patterns
correspondentes ainda não sejam críticos. O custo de um LSI não usado é apenas storage — muito
menor do que uma migração de tabela.

---

## Exercício de reflexão

Você está trabalhando com uma tabela `Events` para uma plataforma de eventos ao vivo. A tabela
tem:
- PK = `event_id`
- Atributos: `venue_id`, `start_time`, `status` (`UPCOMING`/`LIVE`/`ENDED`), `category`, `ticket_price`

Os access patterns são:
- AP1: buscar evento por ID → GetItem na tabela base
- AP2: listar todos os eventos de um venue por data → GSI
- AP3: listar todos os eventos `LIVE` agora (pode ter 5-500 eventos simultâneos, pico de 2.000 writes/s na tabela) → GSI sparse com sharding?
- AP4: listar eventos de uma categoria por preço crescente → GSI
- AP5: listar eventos de um venue ordenados por preço → alternativa ao sort key padrão do AP2

Para o AP3, calcule: com pico de 2.000 writes/s, cada write afetando o GSI de status, quantos
shards são necessários para evitar hot partition (assumindo limite de 800 WCU/s por partição como
margem segura)? Para o AP5, discuta se uma LSI ou um segundo GSI seria mais apropriado, e quais
são as implicações de escolher a LSI dado que a plataforma pode ter venues com milhares de eventos
ao longo dos anos.

---

## Recursos para aprofundar

**1. Using Global Secondary Indexes in DynamoDB**
URL: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GSI.html
O que você vai encontrar: modelo completo do GSI (projeções, consistência eventual, sincronização
assíncrona), cálculo exato de WCU por operação no GSI (com tabela de cenários), considerações
de storage (overhead de 100 bytes por item no índice).
Por que é a fonte certa: documentação primária com a tabela de cenários de write cost por tipo
de operação — exatamente o que é necessário para calcular write amplification.

**2. Understanding Global Secondary Index (GSI) write throttling and back pressure in DynamoDB**
URL: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/gsi-throttling.html
O que você vai encontrar: os 4 tipos de throttling de GSI com os códigos de erro específicos
(`IndexWriteProvisionedThroughputExceeded`, `IndexWriteKeyRangeThroughputExceeded`, etc.) e o
mecanismo de back-pressure explicado com o exemplo de `status` como PK de GSI.
Por que é a fonte certa: é a única página da documentação oficial que explica o back-pressure de
forma explícita — o conceito mais contraintuitivo dos GSIs.

**3. Local secondary indexes**
URL: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/LSI.html
O que você vai encontrar: limitação de criação somente no CreateTable, limite de 10 GB por item
collection, comportamento de fetch de atributos não projetados (com custo calculado pelo item
completo da tabela base), `ReturnItemCollectionMetrics` para monitorar o tamanho da coleção.
Por que é a fonte certa: documentação primária com o cálculo exato de custo do fetch de atributos
não projetados — uma das nuances mais negligenciadas do LSI.

**4. General guidelines for secondary indexes in DynamoDB**
URL: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-indexes-general.html
O que você vai encontrar: guidelines práticos para projeção (quando usar KEYS_ONLY vs INCLUDE vs
ALL), recomendações de quando criar índices vs quando usar FilterExpression, e quando sparse
indexes são a solução correta.
Por que é a fonte certa: é a síntese das melhores práticas de indexação — mais prático do que as
páginas de referência individuais de GSI/LSI.

---

