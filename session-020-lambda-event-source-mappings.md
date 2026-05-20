# Sessão session-020-lambda-event-source-mappings — Lambda: event source mappings — SQS, Kinesis e DynamoDB Streams com filtering

**Duração estimada:** 60 minutos
**Pré-requisitos:** session-019-lambda-execution-cold-starts

---

## Objetivo

Ao final, você conseguirá configurar um event source mapping para SQS com batch size e bisect-on-error, adicionar um event filter para processar apenas eventos com propriedades específicas (sem invocar a Lambda desnecessariamente), e entender o comportamento de retry e DLQ para cada source.

---

## Contexto

[FATO] Um **Event Source Mapping (ESM)** é um recurso do Lambda que faz poll de uma fonte de eventos (SQS, Kinesis, DynamoDB Streams, Kafka, MQ) e invoca sua função com lotes de registros. É o Lambda que puxa os dados ativamente — diferente de triggers assíncronos (S3, SNS, EventBridge) onde a fonte empurra o evento para o Lambda. Essa distinção tem implicações profundas no comportamento de retry, ordenação, e tratamento de erros.

[FATO] Os três sources mais comuns têm modelos de dados e garantias muito diferentes:

| | SQS | Kinesis | DynamoDB Streams |
|---|---|---|---|
| Modelo | Fila (destruição após leitura) | Stream (retenção 24h-365 dias) | Stream de mudanças da tabela |
| Ordenação | Por message group (FIFO) ou sem garantia (Standard) | Garantida por shard | Garantida por partition key |
| Retry | Por visibility timeout | Por shard (bloqueia) | Por shard (bloqueia) |
| bisectOnError | ❌ Não disponível | ✅ Disponível | ✅ Disponível |
| DLQ | Na fila SQS (nativa) | On-failure destination (S3, SQS, SNS) | On-failure destination |

[CONSENSO] O erro mais crítico ao trabalhar com ESMs é não entender a semântica de falha de cada source. No SQS, uma falha retorna a mensagem à fila (pelo visibility timeout) e outro worker pode processá-la — o shard não é bloqueado. No Kinesis e DynamoDB Streams, uma falha **bloqueia todo o shard** até ser resolvida. Processar um item de Kinesis com erro indefinidamente pode paralisar o processamento de todos os registros subsequentes naquele shard — incluindo os que não têm nenhum problema.

---

## Conceitos principais

### 1. Event Source Mapping com SQS

[FATO] O Lambda ESM faz long polling na fila SQS (até 20 segundos por request) e invoca a função com um batch de mensagens. O ciclo de processamento é:

```
Lambda ESM                    SQS Queue
     │                             │
     │─── ReceiveMessage ─────────▶│
     │◀── Batch de N mensagens ────│
     │                             │  mensagens ficam invisíveis
     │                             │  durante visibility timeout
     │
     │─── Invoke(batch) ──────────▶ Lambda Function
     │                                    │
     │                              processa mensagens
     │                                    │
     │     ┌─────────── Sucesso ──────────┘
     │     │            Lambda retornou sem erro
     │     ▼
     │  DeleteMessage (todas as mensagens do batch)
     │
     │     ┌─────────── Falha total ──────┘
     │     │            Lambda lançou exceção
     │     ▼
     │  NÃO faz DeleteMessage
     │  Após visibility timeout: mensagens voltam à fila
     │  Se maxReceiveCount atingido → mensagens vão para DLQ

     │     ┌─────── Falha parcial ────────┘ (com ReportBatchItemFailures)
     │     │        Lambda retornou { batchItemFailures: [...] }
     │     ▼
     │  DeleteMessage apenas das mensagens com sucesso
     │  Mensagens com falha voltam à fila pelo visibility timeout
```

[FATO] Parâmetros chave do ESM para SQS:

| Parâmetro | Default | Range | Descrição |
|---|---|---|---|
| `batchSize` | 10 | 1–10.000 | Máximo de mensagens por invocação |
| `maximumBatchingWindowInSeconds` | 0 | 0–300 | Aguarda N segundos para acumular mensagens antes de invocar |
| `functionResponseTypes` | `[]` | `[ReportBatchItemFailures]` | Habilita partial batch response |

[FATO] `maximumBatchingWindowInSeconds` é crítico para throughput. Com `batchSize=100` e `batchingWindow=0`, o Lambda é invocado com qualquer quantidade de mensagens disponíveis (até 100), incluindo batches de 1 mensagem. Com `batchingWindow=5`, o Lambda acumula mensagens por até 5 segundos antes de invocar, resultando em batches maiores e menos invocações totais — reduz custo e aumenta eficiência.

---

### 2. Partial Batch Response (ReportBatchItemFailures)

[FATO] Por padrão, se *qualquer* mensagem do batch falhar, *todas* voltam à fila. Isso causa reprocessamento de mensagens que já foram processadas com sucesso. A solução é `ReportBatchItemFailures`: a função retorna explicitamente quais mensagens falharam, e o Lambda só devolve essas à fila.

```python
# Python: partial batch response para SQS
def handler(event, context):
    failures = []
    
    for record in event['Records']:
        try:
            process_message(record['body'])
        except Exception as e:
            print(f"Falha ao processar {record['messageId']}: {e}")
            failures.append({'itemIdentifier': record['messageId']})
    
    # Retorna apenas as mensagens que falharam
    # As mensagens não listadas aqui são deletadas com sucesso
    return {'batchItemFailures': failures}
```

```typescript
// TypeScript: partial batch response
import { SQSHandler, SQSBatchResponse } from 'aws-lambda';

export const handler: SQSHandler = async (event): Promise<SQSBatchResponse> => {
  const failures: SQSBatchResponse['batchItemFailures'] = [];

  await Promise.allSettled(
    event.Records.map(async (record) => {
      try {
        await processMessage(JSON.parse(record.body));
      } catch (err) {
        failures.push({ itemIdentifier: record.messageId });
      }
    })
  );

  return { batchItemFailures: failures };
};
```

[FATO] Para que `ReportBatchItemFailures` funcione, o ESM deve ser configurado com `functionResponseTypes: [ReportBatchItemFailures]`. Se o ESM não tiver essa configuração, o Lambda ignora o campo `batchItemFailures` no retorno e trata qualquer resposta como sucesso total.

---

### 3. Event Source Mapping com Kinesis e DynamoDB Streams

[FATO] Kinesis e DynamoDB Streams são diferentes do SQS em um aspecto fundamental: os registros são organizados em **shards**, e o processamento de cada shard é **sequencial e bloqueante**. Se um batch falha, o Lambda reprocessa o mesmo batch (ou um subconjunto, com bisect) antes de avançar para o próximo registro no shard.

```
Kinesis Shard
  ├── Record 1 (posição: 1000)  ← processado com sucesso
  ├── Record 2 (posição: 1001)  ← processado com sucesso
  ├── Record 3 (posição: 1002)  ← FALHA → retry do batch contendo 3+
  ├── Record 4 (posição: 1003)  ← BLOQUEADO aguardando resolução do 3
  ├── Record 5 (posição: 1004)  ← BLOQUEADO
  └── ...

Sem bisect: retenta o batch {3, 4, 5} inteiro até resolver
Com bisect:
  1ª tentativa: batch {3, 4, 5} falha
  2ª tentativa: bisect → {3} e {4, 5}
    - {3} falha → bisect → {3} (batch de 1, sem bisect possível)
    - {4, 5} sucesso
  3ª tentativa: retenta {3} até maxRetry ou expiração
```

[FATO] Parâmetros chave para Kinesis/DynamoDB Streams:

| Parâmetro | Default | Range | Descrição |
|---|---|---|---|
| `batchSize` | 100 | 1–10.000 | Registros por invocação |
| `bisectBatchOnFunctionError` | false | boolean | Divide batch ao meio em caso de erro |
| `maximumRetryAttempts` | -1 (infinito) | -1, 0–10.000 | Tentativas antes de descartar |
| `maximumRecordAgeInSeconds` | -1 (infinito) | -1, 60–604.800 | Descarta registros mais antigos que N segundos |
| `destinationConfig.onFailure` | nenhum | SQS/SNS/S3/Lambda ARN | Destino de registros descartados |
| `startingPosition` | obrigatório | `LATEST`, `TRIM_HORIZON`, `AT_TIMESTAMP` | Ponto de início de leitura |

[FATO] `maximumRetryAttempts: -1` (infinito) + `maximumRecordAgeInSeconds: -1` (infinito) significa que um registro problemático bloqueia o shard **para sempre** até ser manualmente resolvido. Em produção, sempre configure pelo menos um desses para evitar shards eternamente bloqueados.

[FATO] `startingPosition` para novos ESMs:
- `TRIM_HORIZON`: lê desde o registro mais antigo disponível. Use ao configurar um ESM pela primeira vez em um stream com dados existentes que devem ser processados.
- `LATEST`: ignora registros existentes e começa apenas com novos. Use quando os dados históricos não devem ser processados.
- `AT_TIMESTAMP`: começa a partir de um timestamp específico. Útil para replay de um período específico.

---

### 4. Event Filtering

[FATO] Event filters permitem que o ESM descarte registros *antes* de invocar a Lambda — sem custo de invocação. Para SQS, registros descartados pelo filtro são **deletados da fila automaticamente** (não voltam para reprocessamento). Para Kinesis e DynamoDB Streams, o iterator avança além dos registros filtrados (não bloqueia).

[FATO] A sintaxe de filtro é baseada em EventBridge pattern matching. Cada filtro é um objeto JSON que descreve as propriedades que o evento **deve ter** para ser processado. Múltiplos filtros são combinados com OR lógico.

**Para SQS** — filtra no campo `body` (que deve ser JSON válido):

```json
{
  "Filters": [
    {
      "Pattern": "{\"body\": {\"eventType\": [\"ORDER_PLACED\", \"ORDER_UPDATED\"]}}"
    }
  ]
}
```

Apenas mensagens SQS cujo `body` contém `eventType` igual a `ORDER_PLACED` ou `ORDER_UPDATED` chegam à Lambda.

**Para DynamoDB Streams** — filtra no campo `dynamodb`:

```json
{
  "Filters": [
    {
      "Pattern": "{\"dynamodb\": {\"NewImage\": {\"status\": {\"S\": [\"ACTIVE\"]}}}}"
    },
    {
      "Pattern": "{\"eventName\": [\"INSERT\"]}"
    }
  ]
}
```

Este filtro processa registros onde: `status = 'ACTIVE'` na nova imagem **OU** o tipo de evento é `INSERT`.

**Para Kinesis** — filtra no campo `data` (base64 decodificado, deve ser JSON):

```json
{
  "Filters": [
    {
      "Pattern": "{\"data\": {\"source\": [\"mobile-app\"], \"priority\": [{\"numeric\": [\">=\", 3]}]}}"
    }
  ]
}
```

[FATO] Operadores suportados nos filtros:

```
Comparação de valor:
  ["valor"]                   → igualdade exata
  [{"prefix": "ord-"}]        → começa com
  [{"suffix": "-v2"}]         → termina com (para strings)
  [{"anything-but": ["X"]}]   → qualquer valor exceto X

Numérico (SQS e Kinesis apenas, não DynamoDB Streams):
  [{"numeric": ["=", 100]}]
  [{"numeric": [">", 0, "<", 100]}]
  [{"numeric": [">=", 0]}]

Existência:
  [{"exists": true}]          → campo existe no evento
  [{"exists": false}]         → campo não existe
```

---

## Exemplo prático

**Cenário:** Um sistema de e-commerce com três pipelines de processamento:

1. **SQS**: fila de pedidos, processa apenas pedidos `status: "PLACED"` com partial batch response.
2. **DynamoDB Streams**: reage a mudanças na tabela de inventário, processa apenas quando `quantity` cai para zero (alerta de esgotamento).
3. **Kinesis**: stream de eventos de usuário, processa apenas eventos de clique de alta prioridade.

### Pipeline 1: SQS com filtro e partial batch response (CDK)

```typescript
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as lambdaEventSources from 'aws-cdk-lib/aws-lambda-event-sources';
import * as sqs from 'aws-cdk-lib/aws-sqs';
import { Duration } from 'aws-cdk-lib';

const orderQueue = new sqs.Queue(this, 'OrderQueue', {
  visibilityTimeout: Duration.seconds(180),  // >= 6x timeout da Lambda
  deadLetterQueue: {
    queue: new sqs.Queue(this, 'OrderDLQ'),
    maxReceiveCount: 5,   // após 5 falhas → DLQ
  },
});

const orderProcessor = new lambda.Function(this, 'OrderProcessor', {
  runtime: lambda.Runtime.NODEJS_22_X,
  handler: 'index.handler',
  code: lambda.Code.fromAsset('lambda/order-processor'),
  timeout: Duration.seconds(30),
});

orderProcessor.addEventSource(
  new lambdaEventSources.SqsEventSource(orderQueue, {
    batchSize: 50,
    maxBatchingWindow: Duration.seconds(10),
    // Habilita partial batch response
    reportBatchItemFailures: true,
    // Filtro: processa apenas pedidos com status PLACED
    filters: [
      lambda.FilterCriteria.filter({
        body: {
          status: lambda.FilterRule.isEqual('PLACED'),
        },
      }),
    ],
  })
);
```

### Pipeline 2: DynamoDB Streams com filtro de esgotamento

```typescript
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';

const inventoryTable = new dynamodb.Table(this, 'InventoryTable', {
  partitionKey: { name: 'productId', type: dynamodb.AttributeType.STRING },
  stream: dynamodb.StreamViewType.NEW_AND_OLD_IMAGES,  // precisa de ambas as imagens
  billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
});

const stockAlertFn = new lambda.Function(this, 'StockAlert', {
  runtime: lambda.Runtime.PYTHON_3_12,
  handler: 'index.handler',
  code: lambda.Code.fromAsset('lambda/stock-alert'),
  timeout: Duration.seconds(30),
});

stockAlertFn.addEventSource(
  new lambdaEventSources.DynamoEventSource(inventoryTable, {
    startingPosition: lambda.StartingPosition.LATEST,
    batchSize: 100,
    bisectBatchOnError: true,            // divide batch problemático
    retryAttempts: 3,                    // máximo de 3 tentativas
    maxRecordAge: Duration.hours(1),     // descarta registros com > 1h
    // On-failure: salva registros descartados no SQS para análise manual
    onFailure: new lambdaEventSources.SqsDlq(
      new sqs.Queue(this, 'StreamDLQ')
    ),
    // Filtro: apenas MODIFY onde quantity passou para 0
    filters: [
      lambda.FilterCriteria.filter({
        eventName: lambda.FilterRule.isEqual('MODIFY'),
        dynamodb: {
          NewImage: {
            quantity: {
              N: lambda.FilterRule.isEqual('0'),
            },
          },
        },
      }),
    ],
  })
);
```

### Pipeline 3: Kinesis com filtro de prioridade

```typescript
import * as kinesis from 'aws-cdk-lib/aws-kinesis';

const eventStream = new kinesis.Stream(this, 'UserEventStream', {
  shardCount: 4,
  retentionPeriod: Duration.days(7),
});

const highPriorityProcessor = new lambda.Function(this, 'HighPriorityProcessor', {
  runtime: lambda.Runtime.NODEJS_22_X,
  handler: 'index.handler',
  code: lambda.Code.fromAsset('lambda/high-priority'),
  timeout: Duration.seconds(60),
});

highPriorityProcessor.addEventSource(
  new lambdaEventSources.KinesisEventSource(eventStream, {
    startingPosition: lambda.StartingPosition.LATEST,
    batchSize: 200,
    maxBatchingWindow: Duration.seconds(5),
    bisectBatchOnError: true,
    retryAttempts: 2,
    maxRecordAge: Duration.minutes(30),
    reportBatchItemFailures: true,
    // On-failure: salva no S3 para análise posterior
    onFailure: new lambdaEventSources.S3OnFailureDestination(
      new s3.Bucket(this, 'FailedRecords')
    ),
    // Filtro: apenas eventos de clique com prioridade >= 3
    filters: [
      lambda.FilterCriteria.filter({
        data: {
          eventType: lambda.FilterRule.isEqual('CLICK'),
          priority: lambda.FilterRule.between(3, 10),
        },
      }),
    ],
  })
);
```

### Handler com partial batch response para Kinesis

```typescript
// lambda/high-priority/index.ts
import { KinesisStreamHandler, KinesisStreamBatchResponse } from 'aws-lambda';

export const handler: KinesisStreamHandler = async (
  event
): Promise<KinesisStreamBatchResponse> => {
  const failures: KinesisStreamBatchResponse['batchItemFailures'] = [];

  for (const record of event.Records) {
    try {
      const data = JSON.parse(
        Buffer.from(record.kinesis.data, 'base64').toString('utf-8')
      );
      await processHighPriorityClick(data);
    } catch (err) {
      console.error(`Falha no record ${record.kinesis.sequenceNumber}:`, err);
      // Para Kinesis, o identificador é sequenceNumber
      failures.push({ itemIdentifier: record.kinesis.sequenceNumber });
    }
  }

  return { batchItemFailures: failures };
};
```

---

## Armadilhas comuns

### Armadilha 1: visibilityTimeout menor que o timeout da Lambda (SQS)

**O erro:** A fila SQS tem `visibilityTimeout: 30s` e a Lambda tem `timeout: 30s`. A Lambda processa um batch que leva exatamente 30 segundos. Nesse ponto, o visibility timeout expira, e a fila torna as mensagens visíveis novamente — o ESM ou outro consumer as pega e começa a reprocessar. A Lambda ainda está em execução (os 30s são o timeout, não o tempo real). Resultado: mensagens processadas em duplicata.

**Por que acontece:** O visibility timeout deve ser maior que o tempo máximo que a Lambda pode levar para processar um batch. A AWS recomenda `visibilityTimeout >= 6 × functionTimeout` para dar margem a retentativas.

**Como evitar:** `visibilityTimeout = max(6 × functionTimeout, 30s)`. Para uma Lambda com timeout de 5 minutos processando batches grandes, o visibility timeout deve ser pelo menos 30 minutos.

---

### Armadilha 2: Shard Kinesis/DynamoDB Streams bloqueado por registro irrecuperável

**O erro:** Um registro corrompido (JSON inválido, schema inesperado) entra no stream. A Lambda falha ao processar. Com `maximumRetryAttempts: -1` (infinito) e `maximumRecordAgeInSeconds: -1` (infinito), o ESM retenta o mesmo registro para sempre. Todos os registros posteriores naquele shard ficam bloqueados. O lag do Kinesis cresce indefinidamente. O sistema para de processar eventos daquele shard.

**Por que acontece:** Registros em streams são processados em ordem por shard. Uma falha permanente sem configuração de descarte ou limite de retry trava o shard indefinidamente.

**Como reconhecer:** A métrica `IteratorAgeMilliseconds` do Kinesis cresce continuamente sem se estabilizar. O CloudWatch Alarm de `IteratorAge` dispara. No console Lambda, o ESM mostra `IteratorAge` muito alto para alguns shards e zero para outros.

**Como evitar:** Sempre configure:
- `maximumRetryAttempts: 3` (ou outro valor finito)
- `maximumRecordAgeInSeconds: 3600` (ou valor adequado ao SLA)
- `destinationConfig.onFailure`: para não perder os registros descartados
- `bisectBatchOnError: true`: para isolar registros problemáticos rapidamente

---

### Armadilha 3: Filtro SQS deletando mensagens não intencionalmente

**O erro:** Você configura um filtro no ESM do SQS para processar apenas mensagens com `eventType: "ORDER_PLACED"`. Outras mensagens (ex: `eventType: "INVENTORY_UPDATE"`) eram antes processadas por outra Lambda via um ESM separado, mas você esqueceu de remover o filtro da nova Lambda. Quando a nova Lambda é implantada com o filtro, mensagens do tipo `INVENTORY_UPDATE` são automaticamente **deletadas da fila** pelo ESM, sem serem processadas por ninguém.

**Por que acontece:** [FATO] Para SQS, mensagens filtradas pelo ESM são deletadas da fila automaticamente — não voltam para processamento. Esse comportamento é diferente do Kinesis/DynamoDB Streams, onde o iterator simplesmente avança.

**Como reconhecer:** Mensagens sumindo da fila sem aparecer nos logs da Lambda (nenhuma invocação foi feita). A métrica `NumberOfMessagesSent` da fila é maior que `NumberOfMessagesDeleted` via Lambda + DLQ — há um delta de mensagens que desapareceram sem processamento.

**Como evitar:** Entenda que filtros em SQS ESM são destrutivos. Use filtros SQS apenas quando você tem certeza de que mensagens filtradas devem ser descartadas permanentemente, não apenas ignoradas. Para roteamento (uma mensagem deve ir para Lambda A ou Lambda B), use SNS fan-out com filtros de subscription ou EventBridge rules em vez de filtros no ESM do SQS.

---

## Exercício de reflexão

Você está projetando o pipeline de processamento de pedidos de um e-commerce. A fila SQS recebe três tipos de mensagens: `ORDER_PLACED`, `ORDER_CANCELLED`, e `PAYMENT_CONFIRMED`. Cada tipo deve ser processado por uma Lambda diferente.

Se você criar um único ESM para cada Lambda com filtros para o respectivo tipo de mensagem, qual problema fundamental ocorre com as mensagens dos tipos não filtrados? Como você resolveria isso mantendo o baixo acoplamento? Considere também o seguinte: o processamento de `PAYMENT_CONFIRMED` é crítico e não pode ter duplicatas; o processamento de `ORDER_PLACED` pode ter leve duplicação se necessário. Como você configuraria o visibility timeout, o maxReceiveCount do DLQ, e o uso de `ReportBatchItemFailures` de forma diferente para cada Lambda, dado esses requisitos distintos?

---

## Recursos para aprofundar

### 1. Using Lambda with Amazon SQS
**URL:** https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html
**O que encontrar:** Guia completo do ESM para SQS: configuração de batch size e batching window, partial batch response com exemplos de código, comportamento de retry, e configuração de DLQ. Inclui a recomendação de `visibilityTimeout >= 6 × functionTimeout`.
**Por que é a fonte certa:** É a referência primária para SQS + Lambda — cobre todos os detalhes de configuração que afetam confiabilidade.

### 2. Control which events Lambda sends to your function (event filtering)
**URL:** https://docs.aws.amazon.com/lambda/latest/dg/invocation-eventfiltering.html
**O que encontrar:** Referência completa da sintaxe de filtros, operadores disponíveis, exemplos para cada source (SQS, Kinesis, DynamoDB Streams), e o comportamento de descarte por source (SQS deleta, streams avançam o iterator).
**Por que é a fonte certa:** É a documentação canônica de filtros — essencial para entender as diferenças de comportamento entre sources.

### 3. How Lambda processes records from stream and queue-based event sources
**URL:** https://docs.aws.amazon.com/lambda/latest/dg/invocation-eventsourcemapping.html
**O que encontrar:** Visão unificada de como o ESM funciona para todos os sources: polling, batch window, concorrência por shard, e todos os parâmetros de error handling (`bisectBatchOnFunctionError`, `maximumRetryAttempts`, `maximumRecordAgeInSeconds`, `destinationConfig`).
**Por que é a fonte certa:** É o documento de referência mais completo sobre o comportamento do ESM — o lugar certo para entender a mecânica antes de debugar problemas de produção.

---

