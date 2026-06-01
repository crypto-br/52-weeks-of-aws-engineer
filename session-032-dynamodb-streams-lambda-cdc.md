# Sessão 032 — DynamoDB Streams: integração com Lambda para CDC e event-driven

**Duração estimada:** 60 minutos  
**Dependências:** session-031-dynamodb-gsi-lsi-hot-partitions, session-020-lambda-event-source-mappings

---

## Objetivo

Compreender a anatomia do DynamoDB Streams, suas garantias de entrega e ordenação, e dominar a configuração do event source mapping com Lambda para implementar padrões CDC (Change Data Capture) e pipelines event-driven confiáveis.

---

## Contexto

[FATO] DynamoDB Streams é um log de alterações ordenado e durável de uma tabela DynamoDB. Cada alteração em um item (INSERT, MODIFY, REMOVE) gera um registro no stream dentro de 24 horas de retenção. O stream captura as mudanças na ordem em que ocorreram, por item.

CDC (Change Data Capture) é o padrão de capturar e reagir a cada mutação de dados à medida que ela ocorre — em vez de fazer polling periódico. [CONSENSO] DynamoDB Streams é a forma canônica de implementar CDC em arquiteturas AWS serverless: auditoria, replicação cross-region, invalidação de cache, projeção em Elasticsearch/OpenSearch, e fan-out para múltiplos consumidores via SNS/SQS.

---

## Conceitos principais

### 1. Anatomia do Stream: view types e retenção

[FATO] Ao habilitar um stream, você escolhe um `StreamViewType` que determina o conteúdo de cada registro:

```
StreamViewType            O que cada record contém
─────────────────────────────────────────────────────────────────
KEYS_ONLY                 Apenas PK (+ SK se existir)
NEW_IMAGE                 Item inteiro após a mudança
OLD_IMAGE                 Item inteiro antes da mudança
NEW_AND_OLD_IMAGES        Ambos — imagem anterior e posterior
─────────────────────────────────────────────────────────────────
```

[FATO] `StreamViewType` **não pode ser alterado** após a criação do stream. Para mudar, é necessário desabilitar o stream atual e criar um novo.

[FATO] Retenção: 24 horas. Records mais antigos que 24h são deletados automaticamente.

**Caso especial — PutItem/UpdateItem sem mudança real:**  
[FATO] Se um `UpdateItem` não altera nenhum valor (SET attr = attr), **nenhum record é gravado no stream**. O stream só captura mudanças efetivas.

**Caso especial — TTL:**  
[FATO] Quando o DynamoDB remove um item por expiração de TTL, gera um record REMOVE com campo especial `userIdentity`:

```json
{
  "eventName": "REMOVE",
  "userIdentity": {
    "type": "Service",
    "principalId": "dynamodb.amazonaws.com"
  },
  "dynamodb": { ... }
}
```

Isso permite distinguir remoção por TTL de remoção por aplicação (que não tem `userIdentity`).

---

### 2. Shards: estrutura ephemeral e parent-child lineage

[FATO] O stream é dividido em **shards**. Cada shard contém um subconjunto dos records, ordenados por `SequenceNumber`.

```
Stream
├── Shard-A (parent) ────── fechado, 24h de records
│     └── Shard-C (child) ─ ativo
└── Shard-B (parent) ────── fechado
      └── Shard-D (child) ─ ativo

Regra: processar parent ANTES de child
       (SequenceNumber do parent é anterior ao child)
```

[FATO] Shards são **ephemeros**: criados e deletados automaticamente pelo DynamoDB em resposta a mudanças de throughput na tabela (split/merge de partições físicas). Você não controla o número de shards.

[FATO] **Limite crítico: máximo 2 leitores simultâneos por shard.** Acima disso: `ProvisionedThroughputExceededException` no stream. Se você usa Lambda + Kinesis Data Streams como fan-out, está perto do limite. [CONSENSO] Para múltiplos consumidores independentes, use o padrão: DynamoDB Streams → Lambda → SNS → múltiplas SQS queues (fan-out).

**Garantias formais:**  
[FATO] 1. **Exactly-once**: cada record aparece exatamente uma vez no stream.  
[FATO] 2. **In-order per item**: todas as mudanças em um mesmo item aparecem em ordem no mesmo shard. Mudanças em itens diferentes podem estar em shards diferentes, sem ordem relativa garantida entre elas.

---

### 3. Estrutura do evento Lambda

[FATO] Quando Lambda consome um stream, recebe um batch de records. Cada record segue esta estrutura:

```json
{
  "eventID": "1",
  "eventVersion": "1.1",
  "eventSource": "aws:dynamodb",
  "awsRegion": "us-east-1",
  "eventName": "INSERT",          // INSERT | MODIFY | REMOVE
  "dynamodb": {
    "StreamViewType": "NEW_AND_OLD_IMAGES",
    "SequenceNumber": "111",
    "ApproximateCreationDateTime": 1428537600,
    "Keys": {
      "PK": {"S": "USER#123"},
      "SK": {"S": "ORDER#2024-01-15T10:00:00Z"}
    },
    "NewImage": {
      "PK":     {"S": "USER#123"},
      "SK":     {"S": "ORDER#2024-01-15T10:00:00Z"},
      "status": {"S": "PENDING"},
      "total":  {"N": "299.90"}
    },
    "OldImage": {
      "PK":     {"S": "USER#123"},
      "SK":     {"S": "ORDER#2024-01-15T10:00:00Z"},
      "status": {"S": "PROCESSING"},
      "total":  {"N": "299.90"}
    }
  }
}
```

[FATO] Os valores usam o **DynamoDB type system** com discriminadores de tipo: `{"S": "string"}`, `{"N": "123"}`, `{"BOOL": true}`, `{"NULL": true}`, `{"L": [...]}`, `{"M": {...}}`.

---

### 4. Event Source Mapping: configuração e filtragem

[FATO] O event source mapping (ESM) é configurado no Lambda, não no stream. Parâmetros principais:

```
Parâmetro                     Valores / Notas
─────────────────────────────────────────────────────────────────
StartingPosition              TRIM_HORIZON (início do stream)
                              LATEST (apenas novos records)
                              AT_TIMESTAMP (ponto específico)
BatchSize                     1–10.000 (default: 100)
BisectBatchOnFunctionError    True: divide batch ao meio em erro
                              (binary search para isolar poison pill)
MaximumRetryAttempts          -1 (infinito) até 10.000
DestinationConfig             OnFailure → SQS ou SNS (DLQ)
FilterCriteria                Até 5 filtros (AND lógico entre campos
                              de um filtro; OR lógico entre filtros)
─────────────────────────────────────────────────────────────────
```

**Filtragem de eventos (event filtering):**  
[FATO] Filtros operam sobre a chave `dynamodb` do evento, não sobre o envelope externo (exceto `eventName` que é acessado diretamente).

```
Estrutura correta de filtro no ESM:
{
  "Filters": [
    {
      "Pattern": "{\"eventName\": [\"INSERT\"], \"dynamodb\": {\"NewImage\": {\"status\": {\"S\": [{\"prefix\": \"OPEN\"}]}}}}"
    }
  ]
}
```

[FATO] Regras de filtragem para DynamoDB Streams:
- Valores de string: `["exact_value"]` ou `[{"prefix": "val"}]`
- Valores numéricos: `[{"numeric": [">", 100]}]` — **somente** para atributos numéricos em `NewImage`/`OldImage`; operadores numéricos **não** são suportados em `Keys`
- Exists: `[{"exists": true}]` ou `[{"exists": false}]` — funciona apenas em **leaf nodes** (atributos escalares), não em objetos intermediários
- Máximo: 5 filtros por ESM; AND entre condições dentro do mesmo filtro; OR entre filtros diferentes

---

### 5. CDC Pattern: replicação e fan-out

[CONSENSO] O padrão CDC com DynamoDB Streams + Lambda para replicação segue esta topologia:

```
DynamoDB Table
      │
      ▼ (stream record)
DynamoDB Streams
      │
      ▼ (event source mapping)
Lambda (CDC processor)
      │
      ├──▶ OpenSearch (projeção para busca)
      ├──▶ SQS (fan-out assíncrono)
      └──▶ DynamoDB replica (cross-region replication)
```

[FATO] Para replicação cross-region, a AWS oferece **DynamoDB Global Tables** como solução gerenciada. [CONSENSO] O padrão manual com Streams + Lambda só é preferível quando você precisa de transformação dos dados antes da replicação ou quando o destino não é DynamoDB.

---

## Exemplo prático

### Cenário: CDC de pedidos → OpenSearch + auditoria em S3

**Tabela:** `orders-table` com Streams habilitado (NEW_AND_OLD_IMAGES)  
**Lambda 1:** CDC processor (indexa no OpenSearch e registra auditoria)

#### CDK Python — Stack completa

```python
from aws_cdk import (
    Stack, Duration, RemovalPolicy,
    aws_dynamodb as dynamodb,
    aws_lambda as lambda_,
    aws_lambda_event_sources as event_sources,
    aws_sqs as sqs,
    aws_iam as iam,
    aws_logs as logs,
)
from constructs import Construct


class OrdersCdcStack(Stack):
    def __init__(self, scope: Construct, construct_id: str, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        # ── Tabela com stream habilitado ──────────────────────────────
        orders_table = dynamodb.Table(
            self, "OrdersTable",
            table_name="orders-table",
            partition_key=dynamodb.Attribute(
                name="PK", type=dynamodb.AttributeType.STRING
            ),
            sort_key=dynamodb.Attribute(
                name="SK", type=dynamodb.AttributeType.STRING
            ),
            billing_mode=dynamodb.BillingMode.PAY_PER_REQUEST,
            stream=dynamodb.StreamViewType.NEW_AND_OLD_IMAGES,
            removal_policy=RemovalPolicy.DESTROY,
        )

        # ── DLQ para falhas não recuperáveis ──────────────────────────
        dlq = sqs.Queue(
            self, "CdcDlq",
            queue_name="orders-cdc-dlq",
            retention_period=Duration.days(14),
        )

        # ── Lambda CDC processor ──────────────────────────────────────
        cdc_fn = lambda_.Function(
            self, "CdcProcessor",
            function_name="orders-cdc-processor",
            runtime=lambda_.Runtime.PYTHON_3_12,
            handler="handler.lambda_handler",
            code=lambda_.Code.from_asset("lambda/cdc_processor"),
            timeout=Duration.seconds(30),
            memory_size=256,
            environment={
                "OPENSEARCH_ENDPOINT": "https://your-domain.us-east-1.es.amazonaws.com",
            },
            log_retention=logs.RetentionDays.ONE_WEEK,
        )

        # ── Event Source Mapping com filtragem e DLQ ─────────────────
        cdc_fn.add_event_source(
            event_sources.DynamoEventSource(
                table=orders_table,
                starting_position=lambda_.StartingPosition.TRIM_HORIZON,
                batch_size=100,
                bisect_batch_on_error=True,          # isola poison pills
                retry_attempts=3,
                on_failure=event_sources.SqsDlq(dlq),
                # Processa apenas INSERT e MODIFY de pedidos
                filters=[
                    lambda_.FilterCriteria.filter({
                        "eventName": lambda_.FilterRule.is_equal("INSERT"),
                        "dynamodb": {
                            "NewImage": {
                                "PK": {"S": lambda_.FilterRule.begins_with("ORDER#")}
                            }
                        }
                    }),
                    lambda_.FilterCriteria.filter({
                        "eventName": lambda_.FilterRule.is_equal("MODIFY"),
                        "dynamodb": {
                            "NewImage": {
                                "PK": {"S": lambda_.FilterRule.begins_with("ORDER#")}
                            }
                        }
                    }),
                ],
            )
        )

        # Permissões para ler do stream (já concedidas pelo DynamoEventSource)
        # Permissão adicional para OpenSearch
        cdc_fn.add_to_role_policy(
            iam.PolicyStatement(
                actions=["es:ESHttpPost", "es:ESHttpPut"],
                resources=["arn:aws:es:*:*:domain/orders-search/*"],
            )
        )
```

#### Lambda handler — CDC processor

```python
# lambda/cdc_processor/handler.py
import json
import os
import boto3
import urllib3
from decimal import Decimal
from typing import Any

http = urllib3.PoolManager()
OPENSEARCH_ENDPOINT = os.environ["OPENSEARCH_ENDPOINT"]


def deserialize_dynamodb_item(item: dict) -> dict:
    """Converte DynamoDB type system → Python nativo."""
    from boto3.dynamodb.types import TypeDeserializer
    deserializer = TypeDeserializer()
    return {k: deserializer.deserialize(v) for k, v in item.items()}


def index_order(order_data: dict, doc_id: str) -> None:
    """Indexa ou atualiza pedido no OpenSearch."""
    url = f"{OPENSEARCH_ENDPOINT}/orders/_doc/{doc_id}"
    encoded = json.dumps(order_data, default=str).encode("utf-8")
    response = http.request(
        "PUT", url,
        body=encoded,
        headers={"Content-Type": "application/json"},
    )
    if response.status >= 400:
        raise RuntimeError(
            f"OpenSearch index failed: {response.status} {response.data}"
        )


def delete_order(doc_id: str) -> None:
    """Remove pedido do OpenSearch."""
    url = f"{OPENSEARCH_ENDPOINT}/orders/_doc/{doc_id}"
    response = http.request("DELETE", url)
    # 404 é aceitável (já removido)
    if response.status >= 400 and response.status != 404:
        raise RuntimeError(
            f"OpenSearch delete failed: {response.status}"
        )


def is_ttl_delete(record: dict) -> bool:
    """Distingue remoção por TTL de remoção por aplicação."""
    user_identity = record.get("userIdentity", {})
    return (
        user_identity.get("type") == "Service"
        and user_identity.get("principalId") == "dynamodb.amazonaws.com"
    )


def lambda_handler(event: dict, context: Any) -> None:
    records = event.get("Records", [])
    print(f"Processing {len(records)} records")

    for record in records:
        event_name = record["eventName"]          # INSERT | MODIFY | REMOVE
        dynamodb_data = record["dynamodb"]
        keys = deserialize_dynamodb_item(dynamodb_data["Keys"])

        # ID canônico no OpenSearch = PK + SK (sem separador ambíguo)
        doc_id = f"{keys['PK']}_{keys.get('SK', '')}"

        if event_name in ("INSERT", "MODIFY"):
            new_image = deserialize_dynamodb_item(dynamodb_data["NewImage"])
            old_image = (
                deserialize_dynamodb_item(dynamodb_data.get("OldImage", {}))
                if "OldImage" in dynamodb_data else {}
            )

            # Para MODIFY: registrar campos alterados para auditoria
            if event_name == "MODIFY" and old_image:
                changed_fields = [
                    k for k in new_image
                    if new_image.get(k) != old_image.get(k)
                ]
                print(f"MODIFY {doc_id}: fields changed = {changed_fields}")

            index_order(new_image, doc_id)

        elif event_name == "REMOVE":
            if is_ttl_delete(record):
                # TTL expiration: log e remove do índice
                print(f"TTL expiry for {doc_id} — removing from search index")
            else:
                # Remoção explícita pela aplicação
                print(f"Application delete for {doc_id}")

            delete_order(doc_id)

    print(f"Successfully processed {len(records)} records")
```

#### CLI — Habilitar stream e criar event source mapping com filtros

```bash
# 1. Habilitar stream em tabela existente
aws dynamodb update-table \
  --table-name orders-table \
  --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES

# 2. Obter ARN do stream
STREAM_ARN=$(aws dynamodb describe-table \
  --table-name orders-table \
  --query 'Table.LatestStreamArn' \
  --output text)
echo "Stream ARN: $STREAM_ARN"

# 3. Criar Event Source Mapping com filtragem
aws lambda create-event-source-mapping \
  --function-name orders-cdc-processor \
  --event-source-arn "$STREAM_ARN" \
  --starting-position TRIM_HORIZON \
  --batch-size 100 \
  --bisect-batch-on-function-error \
  --maximum-retry-attempts 3 \
  --destination-config '{"OnFailure":{"Destination":"arn:aws:sqs:us-east-1:123456789012:orders-cdc-dlq"}}' \
  --filter-criteria '{
    "Filters": [
      {
        "Pattern": "{\"eventName\":[\"INSERT\"],\"dynamodb\":{\"NewImage\":{\"PK\":{\"S\":[{\"prefix\":\"ORDER#\"}]}}}}"
      },
      {
        "Pattern": "{\"eventName\":[\"MODIFY\"],\"dynamodb\":{\"NewImage\":{\"PK\":{\"S\":[{\"prefix\":\"ORDER#\"}]}}}}"
      }
    ]
  }'

# 4. Verificar status do ESM
ESM_UUID=$(aws lambda list-event-source-mappings \
  --function-name orders-cdc-processor \
  --query 'EventSourceMappings[0].UUID' \
  --output text)

aws lambda get-event-source-mapping --uuid "$ESM_UUID" \
  --query '{State: State, BatchSize: BatchSize, LastProcessingResult: LastProcessingResult}'

# 5. Listar shards ativos do stream
aws dynamodbstreams describe-stream \
  --stream-arn "$STREAM_ARN" \
  --query 'StreamDescription.{Status:StreamStatus, Shards:Shards[*].{ShardId:ShardId,Parent:ParentShardId}}'

# 6. Monitorar erros no CloudWatch
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name Errors \
  --dimensions Name=FunctionName,Value=orders-cdc-processor \
  --start-time "$(date -u -v-1H '+%Y-%m-%dT%H:%M:%SZ')" \
  --end-time "$(date -u '+%Y-%m-%dT%H:%M:%SZ')" \
  --period 300 \
  --statistics Sum

# 7. Verificar DLQ (mensagens que falharam definitivamente)
aws sqs get-queue-attributes \
  --queue-url "https://sqs.us-east-1.amazonaws.com/123456789012/orders-cdc-dlq" \
  --attribute-names ApproximateNumberOfMessages
```

---

## Armadilhas comuns

**1. StreamViewType imutável causa perda de dados históricos**  
[FATO] Mudar `StreamViewType` exige desabilitar o stream atual (perdendo records não processados) e criar um novo. Planeje o tipo necessário antes do deploy em produção.

**2. Limit de 2 leitores por shard**  
[FATO] Lambda consome 1 leitor por shard. Se você adicionar um segundo consumidor (ex: Kinesis Data Streams consumer, ou segundo Lambda ESM), você usa 2/2 leitores. Um terceiro causará throttling. Use fan-out via SNS/SQS se precisar de mais consumidores.

**3. FilterExpression no ESM filtra antes do Lambda, mas você ainda paga pelo Lambda invocado**  
[CONSENSO] Correto: a filtragem evita *invocações* do Lambda para records que não passam no filtro. Mas você **ainda paga** pela capacidade de leitura do stream (GetRecords). A economia é nas invocações Lambda, não nas leituras do stream.

**4. Exists não funciona em nodes intermediários**  
[FATO] O filtro `{"exists": true}` só funciona em leaf nodes. Exemplo correto:
```json
{"dynamodb": {"NewImage": {"discount": {"N": [{"exists": true}]}}}}
```
Exemplo **incorreto** (não funciona):
```json
{"dynamodb": {"NewImage": {"discount": {"exists": true}}}}
```

**5. Reprocessamento ao trocar StartingPosition para TRIM_HORIZON**  
[CONSENSO] `TRIM_HORIZON` processa todos os records das últimas 24h. Em tabelas com alta taxa de escrita, isso pode significar milhões de records. Use `LATEST` para consumidores novos que não precisam de histórico, ou `AT_TIMESTAMP` para um ponto específico.

**6. Poison pill sem BisectBatchOnFunctionError**  
[CONSENSO] Um único record malformado pode bloquear indefinidamente o processamento do shard (Lambda reprocessa o batch até `MaximumRetryAttempts`). Sempre habilite `BisectBatchOnFunctionError=True` + DLQ para isolar o record problemático e continuar o processamento.

**7. TTL deletes chegam com até 48h de atraso**  
[FATO] O DynamoDB não remove itens TTL-expired instantaneamente. A remoção pode ocorrer até 48h após a expiração. O record no stream terá `ApproximateCreationDateTime` refletindo *quando a remoção foi processada*, não quando o TTL expirou. Não use timestamps de stream records de TTL como referência temporal confiável.

---

## Exercício de reflexão

Você tem uma tabela `users-table` com DynamoDB Streams habilitado (NEW_AND_OLD_IMAGES). O CDC processor indexa usuários no OpenSearch. Um desenvolvedor reporta que alguns usuários aparecem no OpenSearch com dados desatualizados — o status foi atualizado para `PREMIUM` na tabela, mas o OpenSearch ainda mostra `FREE`.

Analise o cenário e responda:

1. O código CDC recebe corretamente `NEW_IMAGE` no MODIFY. Qual seria a causa mais provável do dado desatualizado no OpenSearch?

2. O mesmo desenvolvedor sugere adicionar um segundo Lambda ESM apontando para o mesmo stream, para manter um cache Redis atualizado. Qual risco isso introduz e como você arquitetaria a solução corretamente?

3. Para reduzir custos, a equipe quer processar apenas mudanças de `status` para `PREMIUM`. Escreva o `FilterCriteria` JSON correto para esse filtro, sabendo que o atributo é: `{"status": {"S": "PREMIUM"}}` em `NewImage`, e o evento é MODIFY.

---

## Recursos para aprofundar

- [FATO] Documentação oficial — DynamoDB Streams: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Streams.html
- [FATO] Lambda event filtering para DynamoDB: https://docs.aws.amazon.com/lambda/latest/dg/with-ddb-filtering.html
- [FATO] Lambda — DynamoDB event source mapping: https://docs.aws.amazon.com/lambda/latest/dg/with-ddb.html
- [OPINIÃO] Blog AWS — CDC patterns com DynamoDB Streams: https://aws.amazon.com/blogs/database/dynamodb-streams-use-cases-and-design-patterns/

---

