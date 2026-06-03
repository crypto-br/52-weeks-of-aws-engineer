# Sessão 034 — DynamoDB: transações e operações condicionais

**Duração estimada:** 60 minutos  
**Dependências:** session-033-dax-arquitetura-casos-uso

---

## Objetivo

Ao final desta sessão, você conseguirá implementar uma `TransactWriteItems` que atualiza múltiplas entidades atomicamente, usar `ConditionExpression` para optimistic locking com version counter, entender os limites de transações (100 itens, 4 MB, 2× custo), e distinguir corretamente os níveis de isolamento de cada operação.

---

## Contexto

[FATO] DynamoDB oferece suporte nativo a transações ACID desde novembro de 2018. `TransactWriteItems` e `TransactGetItems` fornecem atomicidade e isolamento serializável para operações que precisam de consistência entre múltiplos itens — sem a necessidade de coordenação do lado do cliente.

[CONSENSO] O custo 2× das transações (cada item requer duas operações internas: prepare + commit) é frequentemente mal compreendido. Para transações com múltiplos itens, esse custo pode ser economicamente superior a tentar implementar consistência no lado da aplicação com retries e rollbacks manuais.

---

## Conceitos principais

### 1. TransactWriteItems: ações disponíveis e limites

[FATO] `TransactWriteItems` agrupa até **100 ações de escrita** em uma única operação all-or-nothing. As ações permitidas são:

```
Ação            Equivalente         Uso típico
─────────────────────────────────────────────────────────────
Put             PutItem             Criar ou substituir um item
Update          UpdateItem          Modificar atributos
Delete          DeleteItem          Remover um item
ConditionCheck  (sem equivalente)   Verificar condição sem escrita
                                    (ex: verificar se saldo suficiente
                                    antes de debitar em outro item)
```

[FATO] Limites hard:
- Máximo **100 itens distintos** por transação
- Tamanho agregado dos itens: máximo **4 MB**
- Apenas dentro da **mesma região** e **mesma conta AWS**
- **Não é possível** direcionar o mesmo item duas vezes na mesma transação (ex: ConditionCheck + Update no mesmo item → erro de validação)

[FATO] **Transações não funcionam via índices** — as ações devem sempre referenciar a tabela base via PK (+ SK se existir).

---

### 2. Custo 2× e como calcular

[FATO] Cada item em uma transação consome **2× o custo normal** de escrita ou leitura, porque o DynamoDB executa duas fases internas (prepare e commit):

```
Exemplo: TransactWriteItems com 3 itens de 500 bytes cada

Cálculo normal (sem transação):
  Item 1: ceil(500B / 1KB) = 1 WCU
  Item 2: 1 WCU
  Item 3: 1 WCU
  Total: 3 WCU

Com transação (2× por item):
  Item 1: 1 WCU × 2 = 2 WCU
  Item 2: 1 WCU × 2 = 2 WCU
  Item 3: 1 WCU × 2 = 2 WCU
  Total: 6 WCU

Com transação via DAX (extra RCU para popular cache):
  6 WCU (escrita) + 6 RCU (TransactGetItems pós-escrita)
  Total: 6 WCU + 6 RCU
```

[FATO] O custo 2× aparece no CloudWatch como `ConsumedWriteCapacityUnits` — as duas operações internas (prepare e commit) são visíveis separadamente nas métricas.

---

### 3. Isolamento: tabela de níveis por operação

[FATO] Os níveis de isolamento entre transações e outros tipos de operação são:

```
Operação concorrente                  Nível de isolamento
──────────────────────────────────────────────────────────────
PutItem / UpdateItem / DeleteItem     SERIALIZABLE
GetItem (individual)                  SERIALIZABLE
TransactWriteItems / TransactGetItems SERIALIZABLE (entre si)
BatchGetItem (unidade)                READ-COMMITTED *
BatchWriteItem (unidade)              NÃO serializável *
Query / Scan                          READ-COMMITTED

* individual items within batch/query = serializable;
  the batch/query as a unit = read-committed
```

[FATO] **Serializable**: o resultado é equivalente a executar as operações uma após a outra, sem sobreposição. **Read-committed**: a leitura retorna apenas valores de transações já concluídas, mas pode ver versões diferentes de itens diferentes se uma transação concluiu entre as leituras.

[FATO] Após um `TransactWriteItems` bem-sucedido, leituras **eventually consistent** subsequentes ainda podem retornar o estado anterior por um curto período. Para garantir o valor mais recente imediatamente após a transação, use `ConsistentRead=True` via cliente DynamoDB (não DAX).

---

### 4. Idempotência com ClientRequestToken

[FATO] `TransactWriteItems` aceita um `ClientRequestToken` opcional para garantir idempotência em caso de retentativas por timeout ou problemas de conectividade:

```
Janela de idempotência: 10 minutos
  ├── Mesmo token, mesmos parâmetros → sucesso, sem repetição da escrita
  └── Mesmo token, parâmetros diferentes → IdempotentParameterMismatch

Após 10 minutos: mesmo token é tratado como nova requisição
```

[FATO] Se `ReturnConsumedCapacity` estiver configurado: a chamada original retorna WCUs consumidas; chamadas repetidas dentro da janela retornam RCUs (custo de leitura para verificar idempotência).

---

### 5. ConditionExpression: funções e operadores

[FATO] `ConditionExpression` pode ser usado em `PutItem`, `UpdateItem`, `DeleteItem`, e na ação `ConditionCheck` dentro de transações. Se a condição falhar, a operação retorna `ConditionalCheckFailedException`.

```
Funções disponíveis:
─────────────────────────────────────────────────────────────
attribute_exists(path)         Verdadeiro se o atributo existe
attribute_not_exists(path)     Verdadeiro se o atributo NÃO existe
attribute_type(path, type)     Verifica tipo: S, N, B, SS, NS, BS,
                               M, L, NULL, BOOL
begins_with(path, substr)      String começa com o valor dado
contains(path, operand)        Set contém elemento / string contém substr
size(path)                     Tamanho do atributo (número de bytes/chars)

Operadores de comparação: =  <>  <  <=  >  >=
Operadores lógicos: AND  OR  NOT
Operador de faixa: BETWEEN :lo AND :hi
Operador de lista: IN (:v1, :v2, ...)
```

[FATO] `attribute_not_exists(PK)` em um `PutItem` é o mecanismo canônico de **put-if-not-exists** — evita sobrescrever um item existente. Se a tabela tem PK + SK, a condição verifica se a **combinação** PK+SK não existe.

---

### 6. Optimistic locking com version counter

[FATO] O padrão de optimistic locking em DynamoDB usa um atributo `version` (número inteiro) que é verificado antes de cada escrita e incrementado junto com a atualização:

```
Fluxo do optimistic locking:

1. Client A lê item: {PK: "X", version: 5, data: "abc"}
2. Client B lê item: {PK: "X", version: 5, data: "abc"}
3. Client A faz UpdateItem:
   UpdateExpression:    "SET data = :new, version = version + :inc"
   ConditionExpression: "version = :expected"
   ExpressionAttributeValues: {":new": "xyz", ":inc": 1, ":expected": 5}
   → Sucesso: item agora tem version: 6

4. Client B tenta UpdateItem com version: 5
   ConditionExpression: "version = :expected" (esperava 5, encontrou 6)
   → ConditionalCheckFailedException: Client B deve reler e retentar
```

[CONSENSO] Optimistic locking é preferível a pessimistic locking (locks explícitos) em DynamoDB porque: (a) não há mecanismo nativo de lock; (b) conflitos reais são raros na maioria dos workloads; (c) o custo de retentativa é menor que o overhead de coordenação de locks.

---

## Exemplo prático

### Cenário: transferência bancária — débito e crédito atômicos com ConditionCheck de saldo

**Tabela:** `accounts-table` (single-table design)  
**Operação:** transferir valor entre duas contas com validação de saldo, versioning e auditoria atômica

#### CDK Python — Tabela e função Lambda

```python
from aws_cdk import (
    Stack, RemovalPolicy,
    aws_dynamodb as dynamodb,
    aws_lambda as lambda_,
    aws_iam as iam,
)
from constructs import Construct


class BankingStack(Stack):
    def __init__(self, scope: Construct, construct_id: str, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        accounts_table = dynamodb.Table(
            self, "AccountsTable",
            table_name="accounts-table",
            partition_key=dynamodb.Attribute(
                name="PK", type=dynamodb.AttributeType.STRING
            ),
            sort_key=dynamodb.Attribute(
                name="SK", type=dynamodb.AttributeType.STRING
            ),
            billing_mode=dynamodb.BillingMode.PAY_PER_REQUEST,
            removal_policy=RemovalPolicy.DESTROY,
        )

        transfer_fn = lambda_.Function(
            self, "TransferFn",
            function_name="bank-transfer",
            runtime=lambda_.Runtime.PYTHON_3_12,
            handler="handler.lambda_handler",
            code=lambda_.Code.from_asset("lambda/transfer"),
        )
        accounts_table.grant_read_write_data(transfer_fn)
```

#### Python — Implementação completa

```python
# lambda/transfer/handler.py
import uuid
from decimal import Decimal
from datetime import datetime, timezone
import boto3
from boto3.dynamodb.conditions import Attr
from botocore.exceptions import ClientError

dynamodb = boto3.resource("dynamodb", region_name="us-east-1")
table = dynamodb.Table("accounts-table")


# ── Operações de setup (criação de conta) ─────────────────────────────

def create_account(account_id: str, owner: str, initial_balance: Decimal) -> dict:
    """
    PutItem com attribute_not_exists — put-if-not-exists.
    Falha com ConditionalCheckFailedException se a conta já existe.
    """
    try:
        table.put_item(
            Item={
                "PK": f"ACCOUNT#{account_id}",
                "SK": "METADATA",
                "accountId": account_id,
                "owner": owner,
                "balance": initial_balance,
                "version": 0,
                "status": "ACTIVE",
                "createdAt": datetime.now(timezone.utc).isoformat(),
            },
            ConditionExpression="attribute_not_exists(PK)",
        )
        return {"success": True, "accountId": account_id}
    except ClientError as e:
        if e.response["Error"]["Code"] == "ConditionalCheckFailedException":
            raise ValueError(f"Account {account_id} already exists")
        raise


def deactivate_account_if_empty(account_id: str) -> None:
    """
    UpdateItem condicional: só desativa se saldo = 0.
    """
    try:
        table.update_item(
            Key={"PK": f"ACCOUNT#{account_id}", "SK": "METADATA"},
            UpdateExpression="SET #s = :inactive, updatedAt = :ts",
            ConditionExpression="balance = :zero AND #s = :active",
            ExpressionAttributeNames={"#s": "status"},  # "status" é palavra reservada
            ExpressionAttributeValues={
                ":inactive": "INACTIVE",
                ":active": "ACTIVE",
                ":zero": Decimal("0"),
                ":ts": datetime.now(timezone.utc).isoformat(),
            },
        )
    except ClientError as e:
        if e.response["Error"]["Code"] == "ConditionalCheckFailedException":
            raise ValueError("Cannot deactivate: account not active or balance not zero")
        raise


# ── Transferência atômica com TransactWriteItems ──────────────────────

def transfer(
    from_account_id: str,
    to_account_id: str,
    amount: Decimal,
    from_version: int,
    to_version: int,
    idempotency_key: str | None = None,
) -> dict:
    """
    Transferência atômica com:
    - ConditionCheck: valida que from_account tem saldo suficiente e está ATIVA
    - Update em from_account: decrementa saldo + incrementa version
    - Update em to_account: incrementa saldo + incrementa version
    - Put de registro de auditoria (transação imutável)

    from_version / to_version: versões lidas previamente (optimistic locking)
    idempotency_key: opcional, garante idempotência por 10 minutos
    """
    if amount <= 0:
        raise ValueError("Amount must be positive")

    transfer_id = str(uuid.uuid4())
    ts = datetime.now(timezone.utc).isoformat()

    try:
        kwargs = {
            "TransactItems": [
                # 1. ConditionCheck: saldo suficiente + conta ativa + versão correta
                {
                    "ConditionCheck": {
                        "TableName": "accounts-table",
                        "Key": {
                            "PK": {"S": f"ACCOUNT#{from_account_id}"},
                            "SK": {"S": "METADATA"},
                        },
                        "ConditionExpression": (
                            "balance >= :amount "
                            "AND #s = :active "
                            "AND version = :from_v"
                        ),
                        "ExpressionAttributeNames": {"#s": "status"},
                        "ExpressionAttributeValues": {
                            ":amount": {"N": str(amount)},
                            ":active": {"S": "ACTIVE"},
                            ":from_v": {"N": str(from_version)},
                        },
                    }
                },
                # 2. Update: débito na conta origem
                {
                    "Update": {
                        "TableName": "accounts-table",
                        "Key": {
                            "PK": {"S": f"ACCOUNT#{from_account_id}"},
                            "SK": {"S": "METADATA"},
                        },
                        "UpdateExpression": (
                            "SET balance = balance - :amount, "
                            "version = version + :inc, "
                            "updatedAt = :ts"
                        ),
                        "ExpressionAttributeValues": {
                            ":amount": {"N": str(amount)},
                            ":inc": {"N": "1"},
                            ":ts": {"S": ts},
                        },
                    }
                },
                # 3. Update: crédito na conta destino (com versioning)
                {
                    "Update": {
                        "TableName": "accounts-table",
                        "Key": {
                            "PK": {"S": f"ACCOUNT#{to_account_id}"},
                            "SK": {"S": "METADATA"},
                        },
                        "UpdateExpression": (
                            "SET balance = balance + :amount, "
                            "version = version + :inc, "
                            "updatedAt = :ts"
                        ),
                        "ConditionExpression": "version = :to_v",
                        "ExpressionAttributeValues": {
                            ":amount": {"N": str(amount)},
                            ":inc": {"N": "1"},
                            ":to_v": {"N": str(to_version)},
                            ":ts": {"S": ts},
                        },
                    }
                },
                # 4. Put: registro imutável de auditoria
                {
                    "Put": {
                        "TableName": "accounts-table",
                        "Key": {
                            "PK": {"S": f"TRANSFER#{transfer_id}"},
                            "SK": {"S": "RECORD"},
                        },
                        "Item": {
                            "PK": {"S": f"TRANSFER#{transfer_id}"},
                            "SK": {"S": "RECORD"},
                            "fromAccount": {"S": from_account_id},
                            "toAccount": {"S": to_account_id},
                            "amount": {"N": str(amount)},
                            "createdAt": {"S": ts},
                            "status": {"S": "COMPLETED"},
                        },
                        # Garante que transfer_id é único (UUID collision guard)
                        "ConditionExpression": "attribute_not_exists(PK)",
                    }
                },
            ]
        }

        if idempotency_key:
            kwargs["ClientRequestToken"] = idempotency_key

        # Nota: TransactWriteItems via baixo nível (client) não via resource
        client = boto3.client("dynamodb", region_name="us-east-1")
        client.transact_write_items(**kwargs)

        return {
            "success": True,
            "transferId": transfer_id,
            "amount": str(amount),
        }

    except ClientError as e:
        code = e.response["Error"]["Code"]

        if code == "TransactionCanceledException":
            # Extrai os motivos de cancelamento por item
            reasons = e.response.get("CancellationReasons", [])
            failed = [
                {"index": i, "code": r.get("Code"), "message": r.get("Message")}
                for i, r in enumerate(reasons)
                if r.get("Code") != "None"
            ]
            raise ValueError(
                f"Transaction cancelled. Reasons: {failed}"
            ) from e

        if code == "TransactionInProgressException":
            # Conflito com outra transação em andamento no mesmo item
            # SDK retenta automaticamente, mas se chegou aqui, esgotou retries
            raise RuntimeError("Transaction conflict — retry after backoff") from e

        raise


# ── Leitura consistente pós-transação ─────────────────────────────────

def get_account_balance(account_id: str) -> dict:
    """
    TransactGetItems: lê saldo de forma atomicamente consistente.
    Garante que não vemos estado parcial de uma transação em andamento.
    """
    client = boto3.client("dynamodb", region_name="us-east-1")
    response = client.transact_get_items(
        TransactItems=[
            {
                "Get": {
                    "TableName": "accounts-table",
                    "Key": {
                        "PK": {"S": f"ACCOUNT#{account_id}"},
                        "SK": {"S": "METADATA"},
                    },
                    "ProjectionExpression": "balance, version, #s",
                    "ExpressionAttributeNames": {"#s": "status"},
                }
            }
        ]
    )
    item = response["Responses"][0].get("Item")
    if not item:
        raise ValueError(f"Account {account_id} not found")
    return {
        "balance": item["balance"]["N"],
        "version": int(item["version"]["N"]),
        "status": item["status"]["S"],
    }
```

#### CLI — Exemplos de operações condicionais e transações

```bash
# 1. PutItem com attribute_not_exists (put-if-not-exists)
aws dynamodb put-item \
  --table-name accounts-table \
  --item '{
    "PK": {"S": "ACCOUNT#acc-001"},
    "SK": {"S": "METADATA"},
    "balance": {"N": "1000"},
    "version": {"N": "0"},
    "status": {"S": "ACTIVE"}
  }' \
  --condition-expression "attribute_not_exists(PK)"

# 2. UpdateItem com optimistic locking (version counter)
aws dynamodb update-item \
  --table-name accounts-table \
  --key '{"PK": {"S": "ACCOUNT#acc-001"}, "SK": {"S": "METADATA"}}' \
  --update-expression "SET balance = balance - :amount, version = version + :inc" \
  --condition-expression "version = :expected AND balance >= :amount AND #s = :active" \
  --expression-attribute-names '{"#s": "status"}' \
  --expression-attribute-values '{
    ":amount": {"N": "200"},
    ":inc": {"N": "1"},
    ":expected": {"N": "0"},
    ":active": {"S": "ACTIVE"}
  }' \
  --return-values ALL_NEW

# 3. TransactWriteItems via CLI (transferência simplificada)
aws dynamodb transact-write-items \
  --transact-items '[
    {
      "Update": {
        "TableName": "accounts-table",
        "Key": {"PK": {"S": "ACCOUNT#acc-001"}, "SK": {"S": "METADATA"}},
        "UpdateExpression": "SET balance = balance - :amt, version = version + :inc",
        "ConditionExpression": "balance >= :amt AND version = :v",
        "ExpressionAttributeValues": {
          ":amt": {"N": "100"},
          ":inc": {"N": "1"},
          ":v": {"N": "1"}
        }
      }
    },
    {
      "Update": {
        "TableName": "accounts-table",
        "Key": {"PK": {"S": "ACCOUNT#acc-002"}, "SK": {"S": "METADATA"}},
        "UpdateExpression": "SET balance = balance + :amt, version = version + :inc",
        "ExpressionAttributeValues": {
          ":amt": {"N": "100"},
          ":inc": {"N": "1"}
        }
      }
    }
  ]' \
  --client-request-token "transfer-$(uuidgen)" \
  --return-consumed-capacity TOTAL

# 4. TransactGetItems — snapshot atômico de múltiplas contas
aws dynamodb transact-get-items \
  --transact-items '[
    {
      "Get": {
        "TableName": "accounts-table",
        "Key": {"PK": {"S": "ACCOUNT#acc-001"}, "SK": {"S": "METADATA"}},
        "ProjectionExpression": "balance, version"
      }
    },
    {
      "Get": {
        "TableName": "accounts-table",
        "Key": {"PK": {"S": "ACCOUNT#acc-002"}, "SK": {"S": "METADATA"}},
        "ProjectionExpression": "balance, version"
      }
    }
  ]'

# 5. Verificar custo de transações no CloudWatch
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB \
  --metric-name ConsumedWriteCapacityUnits \
  --dimensions Name=TableName,Value=accounts-table \
  --start-time "$(date -u -d '1 hour ago' '+%Y-%m-%dT%H:%M:%SZ' 2>/dev/null || date -u -v-1H '+%Y-%m-%dT%H:%M:%SZ')" \
  --end-time "$(date -u '+%Y-%m-%dT%H:%M:%SZ')" \
  --period 300 \
  --statistics Sum

# 6. Verificar conflitos de transação
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB \
  --metric-name TransactionConflict \
  --dimensions Name=TableName,Value=accounts-table \
  --start-time "$(date -u -d '1 hour ago' '+%Y-%m-%dT%H:%M:%SZ' 2>/dev/null || date -u -v-1H '+%Y-%m-%dT%H:%M:%SZ')" \
  --end-time "$(date -u '+%Y-%m-%dT%H:%M:%SZ')" \
  --period 300 \
  --statistics Sum
```

---

## Armadilhas comuns

**1. Mesmo item duas vezes na mesma transação → erro de validação**  
[FATO] `TransactWriteItems` não permite duas ações no mesmo item dentro da mesma chamada — nem `ConditionCheck` + `Update`, nem dois `Update`s. Se precisar verificar e atualizar o mesmo item, combine a condição no `Update` diretamente com `ConditionExpression`.

**2. Transações em Global Tables: ACID apenas na região de escrita**  
[FATO] `TransactWriteItems` garante atomicidade apenas na região onde foi invocado. A replicação para outras regiões ocorre de forma assíncrona e pode mostrar estado parcial transitoriamente. Não assuma consistência cross-region com transações.

**3. TransactionCanceledException: extrair os motivos por item**  
[FATO] Quando uma transação é cancelada, a exceção contém `CancellationReasons` — um array com um entry por item, na mesma ordem do `TransactItems`. Apenas os itens que falharam têm `Code` diferente de `"None"`. Sem extrair esses motivos, é impossível saber qual condição falhou.

**4. `BatchWriteItem` não é transacional — confusão comum**  
[FATO] `BatchWriteItem` é uma operação de conveniência (reduz round-trips), não transacional. Alguns itens podem ser escritos com sucesso enquanto outros falham. Para atomicidade all-or-nothing, use `TransactWriteItems`. A diferença de custo: `BatchWriteItem` = 1× WCU/item; `TransactWriteItems` = 2× WCU/item.

**5. Palavras reservadas em ConditionExpression**  
[FATO] `status`, `name`, `size`, `value`, `type`, `timestamp`, entre outros, são palavras reservadas no DynamoDB. Usá-las diretamente em expressões causa erro de sintaxe. Use sempre `ExpressionAttributeNames` com o prefixo `#` para substituir: `"#s": "status"`.

**6. Custo dobrado ao usar transações com DAX**  
[FATO] `TransactWriteItems` via DAX: além das 2× WCUs normais, o DAX executa internamente um `TransactGetItems` para popular o cache após cada write transacional — adicionando 2× RCUs por item. Um item de 500 bytes em `TransactWriteItems` via DAX consome 2 WCUs + 2 RCUs.

---

## Exercício de reflexão

Você está modelando um sistema de reserva de ingressos. Quando um cliente compra um ingresso, você precisa:
1. Verificar que o evento ainda tem capacidade disponível (`availableSeats > 0`)
2. Decrementar `availableSeats` no item do evento
3. Criar um item de ingresso para o cliente
4. Registrar a venda em um item de auditoria

Responda:

1. Qual é o problema de implementar esse fluxo com 3 `PutItem`/`UpdateItem` sequenciais separados? Descreva um cenário de race condition concreto que pode ocorrer.

2. Você decide usar `TransactWriteItems`. Quais 3 ou 4 ações coloca na transação e qual `ConditionExpression` usa para garantir que a operação é segura? Escreva o esqueleto da chamada.

3. O sistema tem 100 req/s de compras simultâneas para o mesmo evento. Quantas `TransactionConflict` você esperaria? O que pode ser feito para reduzir conflitos mantendo a consistência?

---

## Recursos para aprofundar

- [FATO] DynamoDB Transactions — How it works: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/transaction-apis.html
- [FATO] ConditionExpression — CLI examples: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Expressions.ConditionExpressions.html
- [FATO] Optimistic locking with version number: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/BestPractices_OptimisticLocking.html
- [FATO] Best practices — implementing version control: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/BestPractices_ImplementingVersionControl.html

---

