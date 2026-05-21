# Sessão — Lambda: Extensions, Layers e Power Tools

**Duração estimada:** 60 minutos
**Pré-requisitos:** session-020-lambda-event-source-mappings

---

## Objetivo

Ao final, você conseguirá criar uma Lambda Layer com dependências compartilhadas, configurar uma extension externa para rodar no lifecycle da Lambda, e instrumentar uma função com Lambda Power Tools (structured logging, tracing e metrics com uma linha de código).

---

## Contexto

[FATO] Três mecanismos complementares resolvem problemas distintos em funções Lambda em escala: **Layers** resolvem duplicação de dependências entre funções, **Extensions** resolvem a necessidade de código auxiliar (agentes de telemetria, scanners de segurança, gerenciadores de secrets) que roda ao lado do handler sem modificar o código da aplicação, e **Powertools** resolve a implementação de padrões de observabilidade (logging estruturado, tracing, métricas) sem boilerplate repetitivo.

[CONSENSO] Esses três mecanismos são frequentemente combinados: Powertools é distribuído como uma Layer AWS-gerenciada, muitos agentes de observabilidade de terceiros (Datadog, Dynatrace, New Relic) são distribuídos como Extensions dentro de Layers, e qualquer dependência compartilhada entre funções deve ser extraída para uma Layer para evitar que cada deploy de função inclua centenas de MB de bibliotecas idênticas.

---

## Conceitos principais

### 1. Lambda Layers: anatomia e funcionamento

[FATO] Uma Layer é um arquivo ZIP contendo código ou dependências que o Lambda extrai para `/opt` antes de executar sua função. O runtime adiciona subdiretórios de `/opt` ao `PATH` e ao caminho de importação da linguagem, tornando as bibliotecas da layer diretamente importáveis sem configuração adicional.

```
Estrutura de diretórios extraída em /opt:

Python:
  /opt/python/                     ← adicionado ao PYTHONPATH
  /opt/python/lib/python3.12/site-packages/
    requests/
    boto3/
    aws_lambda_powertools/

Node.js:
  /opt/nodejs/node_modules/        ← adicionado ao NODE_PATH
    @aws-lambda-powertools/
    axios/

Binários (qualquer runtime):
  /opt/bin/                        ← adicionado ao PATH
    datadog-agent

Compartilhado:
  /opt/lib/                        ← libraries .so
```

[FATO] Uma função pode ter até **5 layers** simultaneamente. O tamanho total descomprimido da função + todas as layers não pode exceder **250 MB**. Cada layer tem versões imutáveis numeradas — você sempre referencia uma versão específica.

```
ARN de uma layer:
arn:aws:lambda:us-east-1:123456789012:layer:my-deps:7
                          ↑ account   ↑ nome   ↑ versão
```

[FATO] Layers podem ser compartilhadas entre contas via resource-based policy. A AWS e parceiros publicam layers públicas — por exemplo, a layer de Powertools for Python:

```
arn:aws:lambda:us-east-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312:8
               ↑ conta AWS oficial      ↑ nome com runtime embutido
```

---

### 2. Criando e publicando uma Layer

**Estrutura de pacote para Python:**

```bash
# 1. Instala dependências no diretório correto
mkdir -p layer/python
pip install requests psycopg2-binary --target layer/python

# 2. Cria o ZIP com a estrutura correta
cd layer
zip -r ../my-deps-layer.zip python/

# 3. Publica a layer
aws lambda publish-layer-version \
  --layer-name my-dependencies \
  --zip-file fileb://../my-deps-layer.zip \
  --compatible-runtimes python3.12 python3.13 \
  --description "Dependências compartilhadas: requests, psycopg2"

# Saída: LayerVersionArn com número de versão
```

**CDK:**

```typescript
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as path from 'path';

// Layer criada a partir de um diretório local
const depsLayer = new lambda.LayerVersion(this, 'DepsLayer', {
  code: lambda.Code.fromAsset(path.join(__dirname, '../layers/dependencies'), {
    bundling: {
      // Bundling no Docker para garantir binários Linux corretos
      image: lambda.Runtime.PYTHON_3_12.bundlingImage,
      command: [
        'bash', '-c',
        'pip install -r requirements.txt -t /asset-output/python && ' +
        'cp -r /asset-input/python /asset-output/',
      ],
    },
  }),
  compatibleRuntimes: [lambda.Runtime.PYTHON_3_12],
  description: 'Dependências Python compartilhadas',
});

// Usando a layer em uma função
const fn = new lambda.Function(this, 'MyFunction', {
  runtime: lambda.Runtime.PYTHON_3_12,
  handler: 'index.handler',
  code: lambda.Code.fromAsset('lambda'),
  layers: [depsLayer],
});
```

[FATO] **Compatibilidade de runtime é crítica**: uma layer construída com Python 3.12 pode ter binários compilados (extensões C como `psycopg2`, `numpy`) incompatíveis com Python 3.11 ou 3.13. Sempre construa a layer no mesmo ambiente que a função — use o Docker bundling do CDK ou o Amazon Linux 2023 para garantir compatibilidade com o execution environment do Lambda.

---

### 3. Lambda Extensions: internal vs external

[FATO] Extensions são processos que rodam **no mesmo execution environment** que a função Lambda, mas com acesso direto ao ciclo de vida do environment via a **Extensions API**. Elas se registram para receber notificações de eventos (`INVOKE`, `SHUTDOWN`) e podem executar código antes, durante, e depois de cada invocação.

**Dois tipos:**

```
Internal Extensions (internas):
  → Rodam no MESMO PROCESSO do runtime
  → Implementadas via wrapper scripts (PYTHON_EXEC_WRAPPER, NODE_OPTIONS, JAVA_TOOL_OPTIONS)
  → Acesso ao processo da função
  → Exemplo: interceptor de chamadas de SDK para segurança

External Extensions (externas — mais comuns):
  → Rodam como PROCESSOS SEPARADOS no execution environment
  → Registram-se na Extensions API via HTTP
  → Continuam rodando após a função retornar
  → Têm acesso ao filesystem /tmp
  → Exemplo: agente Datadog, scanner de secrets, agente de telemetria
```

[FATO] O ciclo de vida de uma external extension:

```
INIT phase:
  1. Extension bootstrap (executável /opt/extensions/my-extension)
  2. POST /register → registra para eventos INVOKE e/ou SHUTDOWN
  3. GET /event/next → bloqueia aguardando o próximo evento
     (todas as extensions devem chegar aqui antes da função executar)

INVOKE phase:
  4. Lambda recebe invocação
  5. Extension recebe evento INVOKE via /event/next
  6. Extension e função executam CONCORRENTEMENTE
  7. Função retorna resposta ao caller
  8. Extension conclui seu trabalho pós-invocação
  9. Extension faz GET /event/next → aguarda próxima invocação

SHUTDOWN phase:
  10. Lambda decide encerrar o environment
  11. Extension recebe evento SHUTDOWN via /event/next
  12. Extension tem 2 segundos para finalizar (flush de buffers, etc.)
  13. Environment é destruído
```

[FATO] Extensions **aumentam o tempo de cold start**: o Lambda aguarda todas as extensions registradas chegarem ao estado "ready" (primeiro GET /event/next) antes de executar o handler. Uma extension pesada pode adicionar 100-500ms ao cold start. Parceiros como Datadog e Dynatrace otimizaram suas extensions para minimizar esse impacto, mas é algo a medir em produção.

[FATO] Extensions ficam no diretório `/opt/extensions/` quando distribuídas via Layer. O arquivo deve ser executável (`chmod +x`). O Lambda executa automaticamente todos os binários nesse diretório.

---

### 4. Lambda Powertools: observabilidade sem boilerplate

[FATO] Lambda Powertools é uma biblioteca open-source mantida pela AWS que implementa os três pilares de observabilidade para Lambda com mínimo de código: **Logger** (structured logging), **Tracer** (X-Ray tracing), e **Metrics** (CloudWatch EMF). Disponível para Python e TypeScript (Node.js).

**Instalação como Layer AWS-gerenciada (sem incluir no pacote):**

```python
# Python — usar a layer pública da AWS:
# arn:aws:lambda:us-east-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312:8
# (versão mais recente em docs.powertools.aws.dev/lambda/python)

# TypeScript — usar a layer pública:
# arn:aws:lambda:us-east-1:094274105915:layer:AWSLambdaPowertoolsTypeScriptV2:27
```

**Logger — structured logging com contexto Lambda automático:**

```python
from aws_lambda_powertools import Logger

logger = Logger(service="order-service")  # inicialização FORA do handler

@logger.inject_lambda_context(log_event=True)  # decorator: injeta requestId, etc.
def handler(event, context):
    logger.info("Processando pedido", order_id=event["orderId"], amount=event["amount"])
    
    try:
        result = process_order(event)
        logger.info("Pedido processado com sucesso", result=result)
        return result
    except Exception as e:
        logger.exception("Falha ao processar pedido")  # inclui stack trace
        raise
```

Saída JSON estruturada no CloudWatch:
```json
{
  "level": "INFO",
  "location": "handler:12",
  "message": "Processando pedido",
  "order_id": "ord-123",
  "amount": 99.90,
  "service": "order-service",
  "timestamp": "2026-06-17T10:30:00.123Z",
  "xray_trace_id": "1-abc123",
  "cold_start": true,
  "function_request_id": "req-456",
  "function_arn": "arn:aws:lambda:..."
}
```

**Tracer — X-Ray tracing com captura automática:**

```python
from aws_lambda_powertools import Tracer

tracer = Tracer(service="order-service")

@tracer.capture_lambda_handler    # cria subsegment para o handler inteiro
def handler(event, context):
    return process_order(event)

@tracer.capture_method            # cria subsegment para qualquer método
def process_order(event):
    # X-Ray automaticamente captura erros e duração deste método
    charge_card(event["paymentInfo"])
    update_inventory(event["items"])
    return {"status": "ok"}
```

**Metrics — CloudWatch EMF sem chamadas de API síncronas:**

```python
from aws_lambda_powertools import Metrics
from aws_lambda_powertools.metrics import MetricUnit

metrics = Metrics(namespace="OrderService", service="order-service")

@metrics.log_metrics(capture_cold_start_metric=True)  # flushed automaticamente
def handler(event, context):
    metrics.add_metric(name="OrdersProcessed", unit=MetricUnit.Count, value=1)
    metrics.add_metric(name="OrderAmount", unit=MetricUnit.Dollars, value=event["amount"])
    metrics.add_dimension(name="PaymentMethod", value=event["paymentMethod"])
    
    result = process_order(event)
    return result
```

[FATO] O Powertools Metrics usa **CloudWatch Embedded Metric Format (EMF)**: métricas são embutidas nos logs como JSON estruturado e extraídas pelo CloudWatch automaticamente. Isso evita chamadas síncronas à API do CloudWatch durante a invocação, que adicionariam latência e custo extra.

**TypeScript (Middy middleware — abordagem funcional):**

```typescript
import { Logger } from '@aws-lambda-powertools/logger';
import { Tracer } from '@aws-lambda-powertools/tracer';
import { Metrics, MetricUnit } from '@aws-lambda-powertools/metrics';
import { injectLambdaContext } from '@aws-lambda-powertools/logger/middleware';
import { captureLambdaHandler } from '@aws-lambda-powertools/tracer/middleware';
import { logMetrics } from '@aws-lambda-powertools/metrics/middleware';
import middy from '@middy/core';

const logger = new Logger({ serviceName: 'order-service' });
const tracer = new Tracer({ serviceName: 'order-service' });
const metrics = new Metrics({ namespace: 'OrderService', serviceName: 'order-service' });

const lambdaHandler = async (event: APIGatewayEvent) => {
  logger.info('Processando pedido', { orderId: event.pathParameters?.id });
  
  metrics.addMetric('OrdersProcessed', MetricUnit.Count, 1);
  
  const result = await processOrder(event);
  return { statusCode: 200, body: JSON.stringify(result) };
};

// Aplica todos os middlewares em uma linha
export const handler = middy(lambdaHandler)
  .use(injectLambdaContext(logger, { logEvent: true }))
  .use(captureLambdaHandler(tracer))
  .use(logMetrics(metrics, { captureColdStartMetric: true }));
```

---

## Exemplo prático

**Cenário completo:** Uma Lambda de processamento de pedidos com:
- Layer para dependências Python compartilhadas (`requests`, `psycopg2`).
- Layer Powertools AWS-gerenciada.
- Observabilidade completa: Logger + Tracer + Metrics.
- Extension Datadog via Layer (simulado com a estrutura de extensão).

### CDK completo

```typescript
import { Stack, StackProps, Duration } from 'aws-cdk-lib';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as logs from 'aws-cdk-lib/aws-logs';
import * as iam from 'aws-cdk-lib/aws-iam';
import { Construct } from 'constructs';

export class InstrumentedLambdaStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    // Layer 1: Dependências Python customizadas
    const depsLayer = new lambda.LayerVersion(this, 'DepsLayer', {
      layerVersionName: 'order-service-deps',
      code: lambda.Code.fromAsset('layers/deps', {
        bundling: {
          image: lambda.Runtime.PYTHON_3_12.bundlingImage,
          command: [
            'bash', '-c',
            'pip install requests==2.31.0 psycopg2-binary==2.9.9 ' +
            '-t /asset-output/python --no-cache-dir',
          ],
        },
      }),
      compatibleRuntimes: [lambda.Runtime.PYTHON_3_12],
      description: 'requests + psycopg2 — versões fixadas',
    });

    // Layer 2: Powertools (layer AWS-gerenciada pública)
    const powertoolsLayer = lambda.LayerVersion.fromLayerVersionArn(
      this,
      'PowertoolsLayer',
      // ARN da layer pública Powertools Python v3 para Python 3.12
      `arn:aws:lambda:${this.region}:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312:8`
    );

    // Log Group com retention
    const logGroup = new logs.LogGroup(this, 'FnLogs', {
      logGroupName: '/aws/lambda/order-processor',
      retention: logs.RetentionDays.ONE_MONTH,
    });

    // Função com as duas layers
    const fn = new lambda.Function(this, 'OrderProcessor', {
      functionName: 'order-processor',
      runtime: lambda.Runtime.PYTHON_3_12,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambda/order-processor'),
      layers: [depsLayer, powertoolsLayer],
      timeout: Duration.seconds(30),
      memorySize: 512,
      environment: {
        // Powertools lê essas variáveis automaticamente
        POWERTOOLS_SERVICE_NAME: 'order-service',
        POWERTOOLS_METRICS_NAMESPACE: 'OrderService',
        LOG_LEVEL: 'INFO',
        // X-Ray tracing ativo (necessário com tracer.capture_lambda_handler)
        POWERTOOLS_TRACE_ENABLED: 'true',
      },
      tracing: lambda.Tracing.ACTIVE,  // habilita X-Ray
      logGroup,
    });

    // Permissão para X-Ray
    fn.addToRolePolicy(new iam.PolicyStatement({
      actions: ['xray:PutTraceSegments', 'xray:PutTelemetryRecords'],
      resources: ['*'],
    }));
  }
}
```

### Código da função (`lambda/order-processor/index.py`)

```python
import os
import json
import requests  # da DepsLayer
from aws_lambda_powertools import Logger, Tracer, Metrics
from aws_lambda_powertools.metrics import MetricUnit
from aws_lambda_powertools.utilities.typing import LambdaContext

# Inicialização fora do handler (reutilizada em warm invocations)
logger = Logger()       # lê POWERTOOLS_SERVICE_NAME do env
tracer = Tracer()       # lê POWERTOOLS_TRACE_ENABLED do env
metrics = Metrics()     # lê POWERTOOLS_METRICS_NAMESPACE do env

@logger.inject_lambda_context(log_event=True)
@tracer.capture_lambda_handler
@metrics.log_metrics(capture_cold_start_metric=True)
def handler(event: dict, context: LambdaContext) -> dict:
    order_id = event.get("orderId")
    amount = event.get("amount", 0)

    logger.info("Iniciando processamento de pedido", order_id=order_id)
    
    metrics.add_metric("OrdersReceived", MetricUnit.Count, 1)
    metrics.add_dimension("Environment", os.environ.get("ENV", "prod"))

    try:
        result = _process_order(order_id, amount)
        metrics.add_metric("OrdersSucceeded", MetricUnit.Count, 1)
        metrics.add_metric("OrderAmount", MetricUnit.NoUnit, amount)
        logger.info("Pedido processado", order_id=order_id, result=result)
        return {"statusCode": 200, "body": json.dumps(result)}
    except Exception as e:
        metrics.add_metric("OrdersFailed", MetricUnit.Count, 1)
        logger.exception("Falha no processamento", order_id=order_id)
        raise

@tracer.capture_method   # cria subsegment "## _process_order" no X-Ray
def _process_order(order_id: str, amount: float) -> dict:
    # Simula chamada a serviço externo
    response = requests.post(
        "https://payment-api.internal/charge",
        json={"orderId": order_id, "amount": amount},
        timeout=5,
    )
    response.raise_for_status()
    return response.json()
```

### Validando a instrumentação

```bash
# 1. Verificar logs estruturados no CloudWatch
aws logs tail /aws/lambda/order-processor --follow \
  | python3 -m json.tool

# 2. Verificar métricas EMF no namespace OrderService
aws cloudwatch list-metrics --namespace OrderService

# 3. Verificar trace no X-Ray
aws xray get-trace-summaries \
  --start-time $(date -d '10 minutes ago' +%s) \
  --end-time $(date +%s) \
  --query 'TraceSummaries[?HasError==`false`].[Id,Duration]'

# 4. Verificar layers configuradas na função
aws lambda get-function-configuration \
  --function-name order-processor \
  --query 'Layers[*].Arn'
```

---

## Armadilhas comuns

### Armadilha 1: Layer construída no macOS com binários incompatíveis com Linux

**O erro:** Você instala dependências Python com `pip install -t ./python` no macOS e empacota como layer. A layer funciona localmente (tests com moto/localstack) mas falha em produção com `ImportError: /opt/python/psycopg2/_psycopg.cpython-312-darwin.so: cannot execute binary file`.

**Por que acontece:** Pacotes com extensões C (`psycopg2`, `cryptography`, `numpy`, `Pillow`) compilam binários específicos para o sistema operacional. Binários macOS (darwin) são incompatíveis com Linux (o execution environment do Lambda).

**Como reconhecer:** `ImportError` com `.so` ou `.dylib` no caminho, ou mensagem `cannot execute binary file`. O problema só aparece em produção — testes locais com o mesmo macOS passam.

**Como evitar:** Sempre construa layers com extensões C usando Docker com a imagem correta do runtime Lambda (`public.ecr.aws/lambda/python:3.12`), ou use o bundling Docker do CDK (`lambda.Runtime.PYTHON_3_12.bundlingImage`). Para psycopg2 especificamente, existe o pacote `psycopg2-binary` que já inclui as bibliotecas necessárias.

---

### Armadilha 2: Extension aumentando cold start acima do aceitável

**O erro:** Você adiciona uma extension de segurança (scanner de vulnerabilidades) de um vendor que inicia um processo Go pesado no cold start. O cold start de uma função Node.js que era de 200ms passa para 1.8 segundos após adicionar a extension via Layer. As SLAs de latência do serviço são violadas.

**Por que acontece:** Extensions externas bloqueiam a fase INIT: o Lambda aguarda todas as extensions chegarem ao estado "ready" (GET /event/next) antes de executar o handler. Uma extension lenta bloqueia toda a fase INIT.

**Como reconhecer:** O `Init Duration` nos logs do Lambda aumenta significativamente após adicionar a extension. O painel CloudWatch mostra spike no P99 de duração após o deploy.

**Como evitar:** Meça o impacto de cada extension no cold start com e sem ela antes de ir para produção. Verifique se o vendor tem opções de inicialização lazy ou assíncrona. Considere usar Provisioned Concurrency para absorver o cold start se a extension for obrigatória mas o impacto for inaceitável.

---

### Armadilha 3: Powertools Metrics sem decorator `@log_metrics` — métricas nunca enviadas

**O erro:** O desenvolvedor usa o `Metrics` do Powertools mas esquece o decorator `@metrics.log_metrics` no handler:

```python
# ❌ Métricas nunca são enviadas
metrics = Metrics()

def handler(event, context):
    metrics.add_metric("OrdersProcessed", MetricUnit.Count, 1)
    return {"statusCode": 200}
    # → métricas adicionadas mas nunca flushed para o CloudWatch
```

**Por que acontece:** O Powertools Metrics usa EMF — as métricas são emitidas escrevendo um JSON especial no stdout ao final da invocação. Sem o decorator (ou chamada manual a `metrics.flush_metrics()`), o JSON nunca é escrito, e as métricas são silenciosamente descartadas.

**Como reconhecer:** Nenhuma métrica aparece no CloudWatch namespace configurado, mesmo com o código de `add_metric` visível nos logs. Não há erro — as métricas simplesmente somem.

**Como evitar:** Sempre use `@metrics.log_metrics` no handler principal. Para casos onde o decorator não é viável (handlers assíncronos complexos, frameworks web), chame `metrics.flush_metrics()` explicitamente em um bloco `finally` para garantir que as métricas são enviadas mesmo em caso de erro.

---

## Exercício de reflexão

Você tem 15 funções Lambda em um microserviço de pagamentos. Todas usam: `requests==2.31.0`, `boto3==1.34.0`, `cryptography==42.0.0`, `aws-lambda-powertools==3.x`. O deployment package de cada função inclui essas dependências — cada ZIP tem ~45 MB.

Como você estruturaria as layers para reduzir o tamanho dos deployments e simplificar as atualizações de dependências? Você criaria uma única layer com tudo ou múltiplas layers separadas por categoria — e quais são os trade-offs de cada abordagem? Quando uma vulnerabilidade for descoberta em `cryptography` e você precisar atualizar para `42.0.1`, qual é o processo de atualização da layer e como garantir que todas as 15 funções usem a versão atualizada sem downtime? E se duas funções precisarem de versões diferentes de `boto3` por incompatibilidade com outras dependências — layers podem resolver esse problema, ou há uma limitação fundamental que torna isso impossível?

---

## Recursos para aprofundar

### 1. Managing Lambda dependencies with layers
**URL:** https://docs.aws.amazon.com/lambda/latest/dg/chapter-layers.html
**O que encontrar:** Guia completo de criação e gerenciamento de layers: estrutura de diretórios por runtime, como publicar, como compartilhar entre contas, limites (250MB, 5 layers), e o comportamento de layers deletadas em funções existentes.
**Por que é a fonte certa:** É a referência primária para layers — cobre todos os detalhes que afetam compatibilidade e manutenção.

### 2. Augment Lambda functions using Lambda extensions
**URL:** https://docs.aws.amazon.com/lambda/latest/dg/lambda-extensions.html
**O que encontrar:** Tipos de extension (internal vs external), como o Extensions API funciona, o ciclo de vida completo com diagrama, como extensions são distribuídas via layers, e a lista de extensions parceiras (Datadog, Dynatrace, New Relic, etc.).
**Por que é a fonte certa:** É o ponto de entrada oficial para o tema — inclui o diagrama de sequência do lifecycle que clarifica o impacto no cold start.

### 3. Lambda Powertools for Python — documentação oficial
**URL:** https://docs.powertools.aws.dev/lambda/python/latest/
**O que encontrar:** Documentação completa de Logger, Tracer, Metrics com exemplos avançados: sampling de logs, annotation de X-Ray, high-resolution metrics, middleware pattern, e utilitários adicionais (idempotency, batch processing, feature flags).
**Por que é a fonte certa:** É a fonte primária — mais atualizada que a documentação AWS e inclui exemplos de padrões avançados que não aparecem no guia básico.

---

