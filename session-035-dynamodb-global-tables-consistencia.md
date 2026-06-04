# Sessão 035 — DynamoDB Global Tables: consistência eventual e conflict resolution

**Duração estimada:** 60 minutos  
**Dependências:** session-034-dynamodb-transacoes-condicionais

---

## Objetivo

Ao final desta sessão, você conseguirá habilitar Global Tables v2019 em múltiplas regiões, explicar o mecanismo de conflict resolution (last-writer-wins baseado em timestamp interno), identificar padrões onde Global Tables agrega valor real (latência de leitura por região, DR ativo-ativo) vs onde é overkill, e calcular o custo adicional de replicação com rWRU/rWCU.

---

## Contexto

[FATO] DynamoDB Global Tables é uma feature de replicação multi-região e multi-ativa gerenciada: cada réplica aceita leituras **e** escritas. Quando uma aplicação escreve em uma réplica, o DynamoDB replica automaticamente a mudança para todas as outras réplicas. Não há réplica "primária" — todas são ativas.

[FATO] Existem duas versões: **2019.11.21 (Current)** e 2017.11.29 (Legacy). Sempre use a versão 2019, que é mais eficiente em custo, suporta mais features (tabelas com dados existentes, on-demand mode) e tem modelo de faturamento simplificado.

---

## Conceitos principais

### 1. Modos de consistência: MREC vs. MRSC

[FATO] Global Tables suporta dois modos de consistência, definidos na criação e **imutáveis** depois:

```
                    MREC                         MRSC
                    (Multi-Region Eventual        (Multi-Region Strong
                    Consistency) — default        Consistency)
────────────────────────────────────────────────────────────────────────
Replicação          Assíncrona                   Síncrona (≥ 1 região
                                                 antes de confirmar)
Latência de escrita Baixa                        Mais alta
Strongly consistent Retorna dado local           Sempre retorna dado
read                (pode ser stale se escrito   mais recente de qualquer
                    em outra região)             região
RPO                 > 0 (segundos tipicamente)   0
TransactWriteItems  Suportado (só na região      NÃO suportado
                    de origem — não ACID cross)
TTL                 Suportado                    NÃO suportado
LSI                 Suportado                    NÃO suportado
Regiões             Qualquer região disponível   Apenas conjuntos fixos:
                                                 US (N.Virginia+Ohio+Oregon),
                                                 EU (Ireland+London+Paris+Frankfurt),
                                                 AP (Tokyo+Seoul+Osaka)
Topologia           Qualquer número de réplicas  Exatamente 3 regiões
                                                 (2 réplicas + 1 witness,
                                                 ou 3 réplicas)
```

[FATO] O modo **não pode ser alterado** após a criação. Para mudar de MREC para MRSC seria necessário recriar a tabela.

---

### 2. Conflict resolution: last-writer-wins

[FATO] No modo MREC, se o mesmo item é modificado simultaneamente em múltiplas regiões, o DynamoDB resolve o conflito com **last-writer-wins baseado em timestamp interno**:

```
Região us-east-1                    Região eu-west-1
──────────────────────              ──────────────────────
t=100ms: Write item X               t=101ms: Write item X
         {status: "A"}                       {status: "B"}
         (timestamp interno: T1)             (timestamp interno: T2)

Replicação cruzada:
  us-east-1 recebe escrita de eu-west-1 (T2)
  eu-west-1 recebe escrita de us-east-1 (T1)

Resolução:
  T2 > T1 → versão "B" vence em AMBAS as regiões
  Resultado final: item X = {status: "B"} nas duas réplicas

IMPORTANTE: a aplicação em us-east-1 NÃO é notificada que sua
escrita foi sobrescrita. Ela recebeu "success" no PutItem.
```

[FATO] O timestamp usado é **interno ao DynamoDB** — não é um atributo visível na aplicação nem é o `createdAt`/`updatedAt` que você define. Não é possível controlar ou influenciar o resultado do conflict resolution.

[CONSENSO] Para workloads onde conflitos simultâneos são possíveis e a perda silenciosa de escritas é inaceitável (ex: transações financeiras, inventário), Global Tables MREC **não é adequado**. Use MRSC ou projete a aplicação para escrever em uma única região.

---

### 3. Modelo de faturamento: rWRU e rWCU

[FATO] Quando uma tabela se torna parte de um Global Table, as unidades de escrita mudam de WRU/WCU para **rWRU/rWCU (replicated)**, cobradas em **todas as regiões que contêm uma réplica**:

```
Cenário: on-demand, 2 regiões (us-east-1 + eu-west-1)
         Item de 1 KB escrito em us-east-1

─────────────────────────────────────────────────────────────
Tabela single-region:   1 WRU (em us-east-1)
─────────────────────────────────────────────────────────────
Tabela global (2 regiões):
  us-east-1: 1 rWRU  (onde foi escrito)
  eu-west-1: 1 rWRU  (réplica de destino)
  Total: 2 rWRU
─────────────────────────────────────────────────────────────
Tabela global (3 regiões):
  us-east-1: 1 rWRU
  eu-west-1: 1 rWRU
  ap-southeast-1: 1 rWRU
  Total: 3 rWRU
─────────────────────────────────────────────────────────────

Preço: rWRU ≈ mesmo preço que WRU (us-east-1: $1.25 / milhão)
       → com 2 regiões: custo de escrita ~2×
       → com 3 regiões: custo de escrita ~3×

GSI updates: cobrados em WRU (não rWRU) em todas as regiões
             → um write com 1 GSI em 2 regiões:
             2 rWRU (base table) + 2 WRU (GSI) = 4 unidades no total

Leituras: cobradas normalmente em RRU/RCU por réplica
          (sem multiplicador — você paga apenas pelas leituras feitas)

Cross-region data transfer: cobrança adicional por GB transferido
                            entre regiões
```

[FATO] Modo MRSC com **witness**: replicação para o witness **não** gera custo de rWRU, storage, ou data transfer. O witness é transparente em termos de faturamento.

---

### 4. Settings sincronizados vs. não sincronizados

[FATO] Nem todas as configurações são sincronizadas entre réplicas:

```
SEMPRE sincronizados (mudar em uma réplica → muda em todas):
  - Capacity mode (provisioned ↔ on-demand)
  - Write capacity (provisioned) e write auto scaling
  - Schema de chave e definições de GSI
  - SSE type (KMS key type)
  - TTL (configuração, não os valores dos itens)
  - Streams definition (MREC)

SINCRONIZADOS mas sobrescritíveis por réplica:
  - Read capacity e read auto scaling (override por réplica)
  - Table class
  - On-demand max read throughput

NUNCA sincronizados (gerenciar manualmente por réplica):
  - Deletion protection
  - Point-in-time recovery (PITR)
  - Tags
  - CloudWatch Contributor Insights
  - Kinesis Data Streams
  - Resource Policies
```

[CONSENSO] PITR não sincronizado é uma armadilha comum: habilitar PITR na tabela original não protege as réplicas. Você deve habilitar PITR **em cada réplica individualmente**.

---

### 5. DAX com Global Tables: comportamento crítico

[FATO] Escritas replicadas de outras regiões chegam **diretamente ao DynamoDB**, **bypassando o DAX**. O DAX na região de destino não é atualizado quando chega uma replicação — apenas quando o TTL do cache expira ou quando a aplicação local escreve pelo DAX.

```
Região eu-west-1:
  Aplicação escreve via DAX → DynamoDB → replicado para us-east-1

Região us-east-1:
  Replicação chega → DynamoDB atualizado
  DAX em us-east-1: STALE até TTL expirar
  Aplicação lê via DAX em us-east-1: pode retornar dado antigo
  (mesmo após a replicação ter chegado ao DynamoDB)
```

[CONSENSO] Se você usa DAX com Global Tables e os dados são escritos em múltiplas regiões, ajuste o TTL do item cache para refletir a tolerância ao staleness aceitável — ou evite DAX para esses padrões de acesso.

---

## Exemplo prático

### Cenário: catálogo de conteúdo global — baixa latência de leitura por região

**Caso de uso:** plataforma de streaming com usuários na América do Norte, Europa e Ásia Pacífico. Conteúdo é publicado centralmente (us-east-1), lido em todas as regiões. Writes de usuário (favoritos, histórico) ocorrem localmente.

#### CDK Python — Global Table MREC com 3 regiões

```python
# IMPORTANTE: Global Tables requerem que as réplicas existam em stacks separadas
# ou usando CfnGlobalTable (L1 construct). O construct L2 Table não suporta
# Global Tables diretamente — é necessário usar CfnGlobalTable ou
# table.add_global_secondary_index após a criação via CfnReplicationGroup.
#
# Padrão recomendado com CDK: usar CfnGlobalTable (L1).

from aws_cdk import (
    Stack, RemovalPolicy,
    aws_dynamodb as dynamodb,
)
from constructs import Construct


class GlobalContentTableStack(Stack):
    """
    Stack deployada em us-east-1.
    O CfnGlobalTable cria réplicas nas regiões especificadas automaticamente.
    """

    def __init__(self, scope: Construct, construct_id: str, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        # CfnGlobalTable cria a tabela global com réplicas em múltiplas regiões
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
            # Streams habilitado (obrigatório para MREC — gerenciado pelo DynamoDB)
            stream_specification=dynamodb.CfnGlobalTable.StreamSpecificationProperty(
                stream_view_type="NEW_AND_OLD_IMAGES"
            ),
            # SSE com chave gerenciada pela AWS (sincronizado entre réplicas)
            sse_specification=dynamodb.CfnGlobalTable.SSESpecificationProperty(
                sse_enabled=True,
            ),
            # TTL no atributo "expiresAt"
            time_to_live_specification=dynamodb.CfnGlobalTable.TimeToLiveSpecificationProperty(
                attribute_name="expiresAt",
                enabled=True,
            ),
            # Réplicas: uma por região
            replicas=[
                # Região principal (us-east-1) — onde a stack é deployada
                dynamodb.CfnGlobalTable.ReplicaSpecificationProperty(
                    region="us-east-1",
                    point_in_time_recovery_specification=dynamodb.CfnGlobalTable.PointInTimeRecoverySpecificationProperty(
                        point_in_time_recovery_enabled=True
                    ),
                    tags=[{"key": "Environment", "value": "production"}],
                ),
                # Réplica Europa
                dynamodb.CfnGlobalTable.ReplicaSpecificationProperty(
                    region="eu-west-1",
                    point_in_time_recovery_specification=dynamodb.CfnGlobalTable.PointInTimeRecoverySpecificationProperty(
                        point_in_time_recovery_enabled=True
                    ),
                    tags=[{"key": "Environment", "value": "production"}],
                ),
                # Réplica Ásia Pacífico
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

#### Python — Escrita regional e leitura com fallback

```python
import boto3
from botocore.exceptions import ClientError
from datetime import datetime, timezone

# Cada região tem seu próprio cliente
REGIONS = ["us-east-1", "eu-west-1", "ap-southeast-1"]
HOME_REGION = "us-east-1"  # região onde conteúdo é publicado

clients = {
    region: boto3.resource("dynamodb", region_name=region)
    for region in REGIONS
}

def get_local_table(region: str):
    return clients[region].Table("content-table")


def publish_content(content_id: str, title: str, content_type: str, body: str) -> dict:
    """
    Publica conteúdo na região home (us-east-1).
    DynamoDB replica automaticamente para eu-west-1 e ap-southeast-1.

    ATENÇÃO: leituras em outras regiões após este write podem ser
    eventually consistent — podem retornar dado stale por alguns milissegundos
    a segundos dependendo da ReplicationLatency.
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
            # Região de origem — útil para filtrar Stream records por região
            "originRegion": HOME_REGION,
        },
        # put-if-not-exists
        ConditionExpression="attribute_not_exists(PK)",
    )
    return {"contentId": content_id, "publishedAt": now}


def get_content(content_id: str, preferred_region: str) -> dict | None:
    """
    Lê conteúdo da réplica local ao usuário.
    Se a réplica local estiver indisponível, tenta home region.

    Para conteúdo publicado recentemente, pode retornar dado stale
    (ReplicationLatency tipicamente < 1 segundo para regiões próximas).
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
                # Tenta próxima região
                continue
            raise

    return None


def record_user_favorite(
    user_id: str,
    content_id: str,
    user_region: str,
) -> None:
    """
    Favorito do usuário é escrito na réplica local ao usuário.
    Não há conflito esperado (um usuário opera em uma região por vez).

    Em casos raros (mesmo usuário em múltiplas sessões simultâneas
    em regiões diferentes), last-writer-wins resolve.
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

#### CLI — Criar Global Table e monitorar replicação

```bash
# 1. Criar tabela single-region primeiro (base para Global Table)
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

# Aguardar tabela ficar ACTIVE
aws dynamodb wait table-exists --table-name content-table --region us-east-1

# 2. Adicionar réplica para criar Global Table (v2019)
aws dynamodb update-table \
  --table-name content-table \
  --replica-updates '[
    {"Create": {"RegionName": "eu-west-1"}},
    {"Create": {"RegionName": "ap-southeast-1"}}
  ]' \
  --region us-east-1

# 3. Verificar status das réplicas
aws dynamodb describe-table \
  --table-name content-table \
  --region us-east-1 \
  --query 'Table.Replicas[*].{Region:RegionName,Status:ReplicaStatus}'

# 4. Habilitar PITR em CADA réplica individualmente (NÃO é sincronizado)
for region in eu-west-1 ap-southeast-1; do
  aws dynamodb update-continuous-backups \
    --table-name content-table \
    --point-in-time-recovery-specification PointInTimeRecoveryEnabled=true \
    --region "$region"
  echo "PITR enabled in $region"
done

# 5. Monitorar ReplicationLatency (MREC) entre regiões
# Métrica publicada por par de regiões (source→destination)
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB \
  --metric-name ReplicationLatency \
  --dimensions \
      Name=TableName,Value=content-table \
      Name=ReceivingRegion,Value=eu-west-1 \
  --start-time "$(date -u -d '1 hour ago' '+%Y-%m-%dT%H:%M:%SZ' 2>/dev/null || date -u -v-1H '+%Y-%m-%dT%H:%M:%SZ')" \
  --end-time "$(date -u '+%Y-%m-%dT%H:%M:%SZ')" \
  --period 60 \
  --statistics Average,Maximum \
  --region us-east-1

# 6. Verificar versão da Global Table (confirmar que é v2019)
aws dynamodb describe-table \
  --table-name content-table \
  --region us-east-1 \
  --query 'Table.GlobalTableVersion'
# Resultado esperado: "2019.11.21"

# 7. Calcular custo aproximado de escritas (exemplo: 1M writes/dia, item 1KB, 3 regiões)
echo "Writes: 1.000.000/dia × 3 regiões × \$1.25/milhão rWRU"
echo "= \$3.75/dia apenas em rWRU (sem contar cross-region transfer)"

# 8. Remover réplica quando não mais necessária
aws dynamodb update-table \
  --table-name content-table \
  --replica-updates '[{"Delete": {"RegionName": "ap-southeast-1"}}]' \
  --region us-east-1
```

---

## Armadilhas comuns

**1. Last-writer-wins silencioso em escritas concorrentes**  
[FATO] A aplicação não é notificada quando sua escrita é sobrescrita por conflict resolution. O `PutItem` retorna sucesso, mas o dado pode ser descartado segundos depois ao chegar a replicação de outra região com timestamp mais recente. Workloads críticos que não toleram perda de escritas devem usar MRSC ou redirecionar escritas para uma única região.

**2. PITR não é sincronizado — cada réplica precisa ser configurada**  
[FATO] Habilitar PITR na tabela original não ativa PITR nas réplicas. Em um incidente que exige restore, as réplicas sem PITR não podem ser restauradas para um ponto anterior. Configure PITR em cada réplica individualmente e automatize isso via IaC.

**3. Transações ACID apenas dentro da região de origem**  
[FATO] `TransactWriteItems` é atômico somente na região onde foi invocado. As outras réplicas podem ver estado parcial transitoriamente durante a propagação. Se você usa Global Tables MREC e transações, os clientes em outras regiões devem ser projetados para tolerar esse comportamento ou usar `TransactGetItems` para leitura atômica pós-transação.

**4. Streams em MREC: records podem diferir entre réplicas**  
[FATO] Em MREC, o processo de replicação pode combinar múltiplas mudanças em uma única escrita replicada. Os registros de Stream de uma réplica podem diferir dos de outra réplica — tanto em conteúdo quanto em ordering entre itens. Se você processa Streams para CDC, adicione um atributo `originRegion` ao item e filtre no Lambda para processar apenas records originados na região desejada.

**5. DAX + Global Tables = cache stale por replicação**  
[FATO] Escritas replicadas de outras regiões não atualizam o cache do DAX local. Um item atualizado em eu-west-1 pode ser lido do DAX em us-east-1 com o valor antigo até o TTL expirar. Ajuste o TTL do item cache para refletir a tolerância ao staleness, ou use leituras diretas ao DynamoDB para dados críticos.

**6. Multiplicação de custo com GSIs**  
[FATO] GSI writes em Global Tables são cobrados em WRU (não rWRU) **em cada réplica**. Com 3 réplicas e 1 GSI por tabela: um write de 1KB gera 3 rWRU (base table) + 3 WRU (GSI) = 6 unidades no total. Com múltiplos GSIs o custo escala rapidamente — projete apenas os GSIs realmente necessários.

---

## Exercício de reflexão

Você está projetando um sistema de perfis de usuários para um app com usuários no Brasil (sa-east-1), EUA (us-east-1) e Europa (eu-west-1). Cada usuário edita seu próprio perfil; não há escritas cruzadas (usuário brasileiro nunca edita perfil de usuário europeu).

Responda:

1. Dado o padrão de acesso descrito, qual é a probabilidade prática de conflito de last-writer-wins? Isso torna MREC seguro para este caso de uso específico? Que tipo de workload tornaria MREC inadequado mesmo com o mesmo modelo de dados?

2. A equipe quer garantir que após uma escrita de perfil, a leitura subsequente na mesma região retorne o dado atualizado. Como o comportamento de strongly consistent reads em MREC afeta isso? Existe alguma situação em que `ConsistentRead=True` ainda retornaria dado stale?

3. Calcule o custo mensal de rWRU para o seguinte cenário: 500.000 usuários ativos, cada um atualiza o perfil 2 vezes por dia (item de 2 KB), 3 regiões, sem GSIs. Compare com o custo de uma tabela single-region com o mesmo volume de escritas.

---

## Recursos para aprofundar

- [FATO] How DynamoDB global tables work (MREC vs MRSC, conflict resolution): https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/V2globaltables_HowItWorks.html
- [FATO] Understanding billing for global tables (rWRU/rWCU, GSI): https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/global-tables-billing.html
- [FATO] Best practices for global tables: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-global-table-design.html
- [FATO] Write modes with global tables: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-global-table-design.prescriptive-guidance.writemodes.html

---

