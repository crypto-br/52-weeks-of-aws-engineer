# Sessão 024 — Lambda Observabilidade: structured logging, X-Ray e Lambda Insights

**Duração estimada:** 60 minutos
**Pré-requisitos:** session-023-stepfunctions-parallel-map-error

---

## Objetivo

Ao final, você conseguirá emitir logs estruturados (JSON) de uma Lambda com campos de correlação (requestId, userId), habilitar X-Ray active tracing e adicionar subsegmentos customizados, e habilitar Lambda Insights para ver métricas de duração, erro e init time por função.

---

## Contexto

[FATO] Observabilidade em sistemas distribuídos se apoia nos três pilares clássicos: **logs** (registro de eventos discretos), **métricas** (séries temporais numéricas) e **traces** (rastreamento de requisições entre serviços). Em Lambda, cada invocação é efêmera e potencialmente distribuída em centenas de worker instances simultâneas — o que torna a correlação entre esses três pilares especialmente crítica.

[CONSENSO] O maior problema de observabilidade em Lambda não é falta de dados, mas falta de correlação. CloudWatch já captura métricas nativas e logs por padrão. O que diferencia um sistema observável de um sistema monitorado é a capacidade de, dado um ID de requisição ou um traceId, encontrar rapidamente o log da função, o trace X-Ray completo, as métricas de performance do worker específico e os erros ocorridos. Structured logging, X-Ray e Lambda Insights são as três ferramentas que permitem construir essa correlação de forma sistemática em Lambda.

[FATO] A partir de 2023, Lambda passou a suportar natively o formato JSON para logs de sistema (mensagens que o próprio serviço Lambda emite — como `START`, `END`, `REPORT`), além dos logs da aplicação. Isso simplifica a ingestão de logs em CloudWatch Logs Insights sem necessidade de parsers customizados.

---

## Conceitos principais

### 1. Structured Logging — logs como objetos, não strings

[FATO] O formato padrão de logs em Lambda é texto plano. Quando a aplicação usa `print()` ou `console.log()`, o CloudWatch recebe uma linha de texto que precisa ser parseada com regex ou glob expressions para extrair campos. Structured logging substitui strings por objetos JSON, tornando cada campo diretamente consultável.

```
Log não estruturado (difícil de consultar):
───────────────────────────────────────────────────────────────
[INFO] 2026-06-24T10:15:32Z - Pedido P001 processado para usuario U42 em 245ms

Log estruturado JSON (CloudWatch Insights auto-descobre campos):
───────────────────────────────────────────────────────────────
{
  "timestamp": "2026-06-24T10:15:32.410Z",
  "level": "INFO",
  "message": "Pedido processado",
  "requestId": "abc123-def456",
  "traceId": "1-66795-abc...",
  "pedido_id": "P001",
  "usuario_id": "U42",
  "duracao_ms": 245,
  "service": "pedidos",
  "version": "2.1.0"
}
```

[FATO] CloudWatch Logs Insights detecta automaticamente campos em linhas JSON sem necessidade de configuração. Uma vez que os logs estejam em JSON, queries como a abaixo funcionam diretamente:

```sql
-- Buscar todos os erros de um usuário específico na última hora
fields @timestamp, level, message, pedido_id, duracao_ms
| filter level = "ERROR" and usuario_id = "U42"
| sort @timestamp desc
| limit 50
```

#### Campos de correlação obrigatórios

[CONSENSO] A prática adotada pela maioria das equipes de produção é incluir pelo menos quatro campos de correlação em cada log:

| Campo | Origem | Uso |
|---|---|---|
| `requestId` | `context.aws_request_id` | Correlacionar todos os logs de uma invocação |
| `traceId` | `os.environ["_X_AMZN_TRACE_ID"]` | Correlacionar com trace X-Ray |
| `service` | Constante na função | Filtrar logs por serviço em log groups agregados |
| `cold_start` | Variável de inicialização | Identificar invocações com Init phase |

#### Habilitando JSON nativo no Lambda (log format da função)

[FATO] Desde 2023, é possível configurar a função Lambda para que as mensagens do **sistema** (START, END, REPORT) também sejam emitidas em JSON. Isso é separado do log format da aplicação:

```python
# CDK — log format e log level nativos da função
from aws_cdk import aws_lambda as lambda_

fn = lambda_.Function(
    self, "MinhaFuncao",
    # ...
    logging_format=lambda_.LoggingFormat.JSON,   # logs sistema em JSON
    system_log_level=lambda_.SystemLogLevel.INFO,
    application_log_level=lambda_.ApplicationLogLevel.INFO,
    log_retention=logs.RetentionDays.ONE_WEEK,
)
```

[FATO] Com `LoggingFormat.JSON`, o registro `REPORT` passa a ser:
```json
{
  "timestamp": "2026-06-24T10:15:32.660Z",
  "type": "platform.report",
  "record": {
    "requestId": "abc123",
    "metrics": {
      "durationMs": 245.12,
      "billedDurationMs": 246,
      "memorySizeMB": 256,
      "maxMemoryUsedMB": 89,
      "initDurationMs": 312.5
    }
  }
}
```

#### Structured logging com Python puro (sem Powertools)

```python
import json
import logging
import os

# Configura o logger raiz para emitir JSON
class JsonFormatter(logging.Formatter):
    def format(self, record):
        log_entry = {
            "timestamp": self.formatTime(record),
            "level": record.levelname,
            "message": record.getMessage(),
            "logger": record.name,
            "requestId": getattr(record, "requestId", None),
            "traceId": os.environ.get("_X_AMZN_TRACE_ID"),
            "service": "pedidos",
        }
        # Campos extras passados via extra={}
        for key in vars(record):
            if key not in logging.LogRecord.__dict__ and not key.startswith("_"):
                log_entry[key] = getattr(record, key)
        return json.dumps(log_entry)

logger = logging.getLogger()
logger.setLevel(logging.INFO)
if logger.handlers:
    logger.handlers[0].setFormatter(JsonFormatter())

# Variável para detectar cold start
COLD_START = True

def handler(event, context):
    global COLD_START
    cold = COLD_START
    COLD_START = False

    # Enriquece todos os logs desta invocação com requestId
    extra = {"requestId": context.aws_request_id, "cold_start": cold}

    logger.info("Invocação iniciada", extra={**extra, "evento_tipo": event.get("type")})

    try:
        resultado = processar(event, extra)
        logger.info("Invocação concluída", extra={**extra, "resultado": resultado["status"]})
        return resultado
    except Exception as e:
        logger.error("Erro na invocação", extra={**extra, "error_type": type(e).__name__, "error_msg": str(e)})
        raise
```

[CONSENSO] O uso de Lambda Powertools Logger (session-021) elimina a necessidade de implementar esse boilerplate manualmente. O decorator `@logger.inject_lambda_context` injeta automaticamente `requestId`, `cold_start`, `xray_trace_id` e outros campos em todos os logs da invocação.

---

### 2. X-Ray Active Tracing — rastreamento distribuído em Lambda

[FATO] AWS X-Ray é o serviço de distributed tracing da AWS. Em Lambda, o tracing funciona via um daemon X-Ray que roda dentro do execution environment e recebe dados via UDP (porta 2000 no loopback). O SDK X-Ray envia segmentos para esse daemon, que os encaminha para o serviço X-Ray.

#### Anatomia de um trace em Lambda

[FATO] Com active tracing habilitado, Lambda cria automaticamente dois segmentos por invocação:

```
Trace (X-Amzn-Trace-Id: Root=1-...;Sampled=1)
├── Segmento 1: "Lambda" (serviço)
│   └── Representa o Lambda service recebendo a invocação
│       Inclui: cold start time, queuing time
│
└── Segmento 2: "minhaFuncao" (função)
    ├── Subsegmento: Initialization (apenas em cold starts)
    │   └── Tempo do Init phase (carregamento do módulo)
    ├── Subsegmento: Invocation
    │   └── Tempo de execução do handler
    │       ├── [seus subsegmentos customizados aqui]
    └── Subsegmento: Overhead
        └── Tempo de checkpoint/extensões
```

[FATO] A variável de ambiente `_X_AMZN_TRACE_ID` contém o trace ID da invocação atual no formato `Root=1-<timestamp>-<hex>;Parent=<parentId>;Sampled=<0|1>`. Essa string deve ser propagada em chamadas downstream (headers HTTP, mensagens SQS, etc.) para manter a continuidade do trace.

#### Habilitando active tracing

```python
# CDK
fn = lambda_.Function(
    self, "MinhaFuncao",
    # ...
    tracing=lambda_.Tracing.ACTIVE,   # ou PASS_THROUGH para herdar do upstream
)
# CDK adiciona automaticamente xray:PutTraceSegments e xray:PutTelemetryRecords
# à execution role da função
```

```bash
# CLI
aws lambda update-function-configuration \
  --function-name MinhaFuncao \
  --tracing-config Mode=Active
```

#### Subsegmentos customizados

[FATO] O SDK X-Ray permite criar subsegmentos para qualquer operação dentro do handler — chamadas a banco de dados, APIs externas, processamento pesado:

```python
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.core import patch_all

# Patcha automaticamente boto3, requests, httplib, pymongo, etc.
patch_all()

def handler(event, context):
    pedido_id = event["pedido_id"]

    # Subsegmento manual com context manager
    with xray_recorder.in_subsegment("validar-pedido") as subseg:
        subseg.put_annotation("pedido_id", pedido_id)   # indexado — filtrável
        subseg.put_annotation("valor", event["valor"])
        subseg.put_metadata("evento_completo", event)   # não indexado — apenas armazenado
        resultado = validar_pedido(pedido_id)

    # Decorator em funções internas
    resultado_bd = salvar_no_banco(pedido_id, resultado)

    return {"status": "ok"}

@xray_recorder.capture("salvar-no-banco")
def salvar_no_banco(pedido_id, dados):
    # boto3 já está patchado — as chamadas DynamoDB aparecem como
    # subsegmentos automáticos dentro de "salvar-no-banco"
    tabela.put_item(Item={"id": pedido_id, **dados})
    return True
```

#### Annotations vs Metadata

[FATO] A distinção entre `put_annotation` e `put_metadata` é crítica para o uso do X-Ray:

```
Annotations                          Metadata
────────────────────────────────     ────────────────────────────────
Tipos: string, número, booleano      Tipos: qualquer JSON serializável
Indexados pelo X-Ray                 NÃO indexados
Aparecem em filter expressions       Apenas visíveis no detalhe do trace
Limite: 50 anotações por trace       Limite: 64KB por segmento
Uso: agrupamento, filtros, alertas   Uso: debug, dados de contexto
```

[FATO] Filter expressions no console X-Ray usam anotações:
```
# Encontrar todos os traces com erro para um pedido específico
annotation.pedido_id = "P001" AND error = true

# Traces lentos (>2s) de um serviço específico
annotation.service = "pedidos" AND responsetime > 2
```

#### Sampling rules

[FATO] Por padrão, X-Ray amostra 5% das requisições (ou 1 req/s, o que for maior). Em produção com alto volume, isso é essencial para controlar custo. Regras customizadas podem ser configuradas:

```bash
aws xray create-sampling-rule --cli-input-json '{
  "SamplingRule": {
    "RuleName": "PedidosAltoValor",
    "Priority": 1,
    "FixedRate": 1.0,
    "ReservoirSize": 5,
    "ServiceName": "pedidos",
    "ServiceType": "AWS::Lambda::Function",
    "Host": "*",
    "HTTPMethod": "*",
    "URLPath": "*",
    "ResourceARN": "*",
    "Attributes": { "pedido_valor": "alto" }
  }
}'
```

---

### 3. Lambda Insights — métricas de sistema por invocação

[FATO] Lambda Insights é implementado como uma **Lambda Extension** (interna), distribuída como uma Lambda Layer gerenciada pela AWS. Quando habilitada, a extensão coleta métricas de sistema de cada invocação e as envia para CloudWatch Logs no grupo `/aws/lambda/insights` usando o formato EMF (Embedded Metric Format), que CloudWatch interpreta para criar métricas de série temporal.

#### Métricas coletadas

[FATO] Lambda Insights coleta as seguintes métricas por invocação:

```
Métricas de performance:
┌─────────────────────────┬──────────────────────────────────────────────────┐
│ Métrica                 │ Descrição                                        │
├─────────────────────────┼──────────────────────────────────────────────────┤
│ duration                │ Duração da invocação em ms                       │
│ billed_duration         │ Duração cobrada (arredondada para 1ms)           │
│ init_duration           │ Tempo do Init phase (cold start apenas)          │
│ memory_utilization      │ % de memória configurada utilizada               │
│ used_memory_max         │ Pico de uso de memória em MB                     │
│ cpu_total_time          │ Tempo total de CPU em ms                         │
├─────────────────────────┼──────────────────────────────────────────────────┤
│ Métricas de I/O:        │                                                  │
│ rx_bytes                │ Bytes recebidos via rede                         │
│ tx_bytes                │ Bytes enviados via rede                          │
│ disk_used               │ Uso de /tmp em MB                                │
│ disk_total              │ Espaço total em /tmp em MB                       │
├─────────────────────────┼──────────────────────────────────────────────────┤
│ Diagnóstico:            │                                                  │
│ cold_start              │ 1 se foi cold start, 0 caso contrário            │
│ out_of_memory           │ 1 se a função excedeu memória                    │
│ timeout                 │ 1 se a função atingiu timeout                    │
│ errors                  │ 1 se houve erro não tratado                      │
└─────────────────────────┴──────────────────────────────────────────────────┘
```

#### Habilitando Lambda Insights via CDK

```python
# CDK
fn = lambda_.Function(
    self, "MinhaFuncao",
    # ...
    tracing=lambda_.Tracing.ACTIVE,
    insights_version=lambda_.LambdaInsightsVersion.VERSION_1_0_229_0,
    # CDK adiciona automaticamente:
    # - A layer gerenciada arn:aws:lambda:<region>:580247275435:layer:LambdaInsightsExtension:...
    # - A policy CloudWatchLambdaInsightsExecutionRolePolicy à execution role
)
```

[FATO] A ARN da layer muda por região. `LambdaInsightsVersion.VERSION_1_0_229_0` é a versão mais recente até maio de 2026 — verifique [docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Lambda-Insights.html](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Lambda-Insights.html) para versões disponíveis na sua região.

#### Dashboard e Log Insights

[FATO] O console CloudWatch cria automaticamente um dashboard em `/LambdaInsights` quando Lambda Insights está habilitado. As métricas ficam disponíveis no namespace `LambdaInsights`.

```sql
-- CloudWatch Logs Insights: invocações mais lentas com cold start
-- Log group: /aws/lambda/insights
fields @timestamp, function_name, duration, init_duration, memory_utilization, cold_start
| filter cold_start = 1
| sort duration desc
| limit 20

-- Correlacionar log de aplicação com métricas Insights
-- Log group: /aws/lambda/MinhaFuncao
fields @timestamp, @message, @requestId
| filter level = "ERROR"
| join insights on requestId = @requestId   -- correlação via requestId
```

---

### 4. Correlação entre os três pilares

[FATO] O campo que une logs, traces e métricas em Lambda é o `requestId` (também chamado `aws_request_id` no context object do Python). O fluxo de correlação é:

```
Invocação recebida
       │
       ▼
Lambda Service gera requestId ─────────────────────────────────────────┐
       │                                                                │
       ▼                                                                ▼
┌──────────────────────┐    ┌────────────────────────┐    ┌────────────────────────┐
│    LOGS              │    │      X-RAY             │    │  LAMBDA INSIGHTS       │
│                      │    │                        │    │                        │
│ Log estruturado com  │    │ Trace ID gerado pelo   │    │ Métricas EMF emitidas  │
│ "requestId": "abc"   │    │ X-Ray daemon           │    │ com requestId e        │
│ "traceId": "1-..."   │    │                        │    │ function_name          │
│ "cold_start": true   │    │ Segmento da função tem │    │                        │
│ "usuario_id": "U42"  │    │ anotação requestId     │    │ init_duration: 312ms   │
│                      │    │                        │    │ memory_utilization: 35%│
└──────────┬───────────┘    └───────────┬────────────┘    └────────────┬───────────┘
           │                            │                              │
           └────────────────────────────┴──────────────────────────────┘
                         requestId como chave de correlação

Console CloudWatch → ServiceLens: une logs + traces em uma view única
```

[FATO] **CloudWatch ServiceLens** (aba no console CloudWatch) consome automaticamente a correlação entre logs e X-Ray traces quando:
1. A função tem active tracing habilitado.
2. Os logs incluem o campo `@xrayTraceId` (Lambda Powertools injeta isso automaticamente; sem Powertools, use `os.environ["_X_AMZN_TRACE_ID"]`).

---

## Exemplo prático

**Cenário:** Função de processamento de pedidos com structured logging completo, X-Ray com subsegmentos customizados e Lambda Insights.

### Handler Python com todos os três pilares

```python
import json
import logging
import os
import time
import boto3
from aws_xray_sdk.core import xray_recorder, patch_all

# Patcha clientes boto3 automaticamente para X-Ray
patch_all()

# ── Structured Logger ──────────────────────────────────────────────────────────
class StructuredLogger:
    def __init__(self, service_name: str, level: str = "INFO"):
        self.service = service_name
        self.level = getattr(logging, level)
        self._base_fields: dict = {}

    def set_invocation_context(self, request_id: str, cold_start: bool):
        self._base_fields = {
            "requestId": request_id,
            "cold_start": cold_start,
            "traceId": os.environ.get("_X_AMZN_TRACE_ID", ""),
            "service": self.service,
        }

    def _emit(self, level: str, message: str, **kwargs):
        entry = {
            "timestamp": time.strftime("%Y-%m-%dT%H:%M:%S.000Z", time.gmtime()),
            "level": level,
            "message": message,
            **self._base_fields,
            **kwargs,
        }
        # Lambda captura stdout; usar print garante flush imediato
        print(json.dumps(entry))

    def info(self, msg: str, **kwargs):  self._emit("INFO", msg, **kwargs)
    def warn(self, msg: str, **kwargs):  self._emit("WARN", msg, **kwargs)
    def error(self, msg: str, **kwargs): self._emit("ERROR", msg, **kwargs)
    def debug(self, msg: str, **kwargs):
        if self.level <= logging.DEBUG:
            self._emit("DEBUG", msg, **kwargs)

logger = StructuredLogger("pedidos")

# ── Clientes (inicializados fora do handler = reutilizados em warm starts) ──────
dynamodb = boto3.resource("dynamodb")
tabela = dynamodb.Table(os.environ["TABELA_PEDIDOS"])

# Detecta cold start
_COLD_START = True

# ── Handler ───────────────────────────────────────────────────────────────────
def handler(event, context):
    global _COLD_START
    cold = _COLD_START
    _COLD_START = False

    logger.set_invocation_context(context.aws_request_id, cold)
    logger.info("Invocação iniciada", pedido_id=event.get("pedido_id"))

    inicio = time.time()

    try:
        resultado = processar_pedido(event)
        duracao_ms = int((time.time() - inicio) * 1000)
        logger.info(
            "Pedido processado",
            pedido_id=event["pedido_id"],
            status=resultado["status"],
            duracao_ms=duracao_ms,
        )
        return resultado

    except ValueError as e:
        logger.error(
            "Erro de validação",
            pedido_id=event.get("pedido_id"),
            error_type="ValueError",
            error_msg=str(e),
        )
        raise
    except Exception as e:
        logger.error(
            "Erro inesperado",
            pedido_id=event.get("pedido_id"),
            error_type=type(e).__name__,
            error_msg=str(e),
        )
        raise


def processar_pedido(event: dict) -> dict:
    pedido_id = event["pedido_id"]

    # ── Subsegmento: validação ─────────────────────────────────────────────────
    with xray_recorder.in_subsegment("validar-pedido") as seg:
        seg.put_annotation("pedido_id", pedido_id)
        seg.put_annotation("valor", event.get("valor", 0))
        seg.put_metadata("evento_completo", event, namespace="pedidos")

        if not pedido_id or not isinstance(event.get("valor"), (int, float)):
            raise ValueError(f"Pedido inválido: campos obrigatórios ausentes")

        if event["valor"] <= 0:
            raise ValueError(f"Valor do pedido deve ser positivo: {event['valor']}")

    # ── Subsegmento: persistência ──────────────────────────────────────────────
    with xray_recorder.in_subsegment("persistir-pedido") as seg:
        seg.put_annotation("pedido_id", pedido_id)
        # boto3 patchado → a chamada DynamoDB aparece como sub-subsegmento
        tabela.put_item(Item={
            "pedido_id": pedido_id,
            "valor": str(event["valor"]),
            "status": "PROCESSADO",
            "request_id": xray_recorder.current_segment().id,
        })

    return {"status": "PROCESSADO", "pedido_id": pedido_id}
```

### CDK — função com todos os três pilares habilitados

```python
from aws_cdk import (
    Stack, Duration, RemovalPolicy,
    aws_lambda as lambda_,
    aws_logs as logs,
    aws_iam as iam,
)

class PedidosObservabilidadeStack(Stack):

    def __init__(self, scope, construct_id, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        # Layer com aws-xray-sdk (construída via Docker para compatibilidade Linux)
        xray_layer = lambda_.LayerVersion(
            self, "XRayLayer",
            code=lambda_.Code.from_asset(
                "layers/xray",
                bundling={
                    "image": lambda_.Runtime.PYTHON_3_12.bundling_image,
                    "command": [
                        "bash", "-c",
                        "pip install aws-xray-sdk -t /asset-output/python"
                    ],
                }
            ),
            compatible_runtimes=[lambda_.Runtime.PYTHON_3_12],
            description="aws-xray-sdk para instrumentação customizada",
        )

        fn = lambda_.Function(
            self, "ProcessarPedido",
            runtime=lambda_.Runtime.PYTHON_3_12,
            handler="handler.handler",
            code=lambda_.Code.from_asset("src/pedidos"),
            memory_size=256,
            timeout=Duration.seconds(30),
            layers=[xray_layer],
            environment={
                "TABELA_PEDIDOS": "pedidos",
                "POWERTOOLS_SERVICE_NAME": "pedidos",
            },
            # Pilar 1: Structured logging nativo
            logging_format=lambda_.LoggingFormat.JSON,
            system_log_level=lambda_.SystemLogLevel.INFO,
            application_log_level=lambda_.ApplicationLogLevel.INFO,
            log_retention=logs.RetentionDays.ONE_WEEK,
            # Pilar 2: X-Ray active tracing
            tracing=lambda_.Tracing.ACTIVE,
            # Pilar 3: Lambda Insights
            insights_version=lambda_.LambdaInsightsVersion.VERSION_1_0_229_0,
        )

        # Permissão adicional para DynamoDB (X-Ray já é adicionado pelo CDK)
        fn.add_to_role_policy(iam.PolicyStatement(
            actions=["dynamodb:PutItem", "dynamodb:GetItem"],
            resources=["arn:aws:dynamodb:*:*:table/pedidos"],
        ))
```

### Queries CloudWatch Logs Insights para diagnóstico

```sql
-- 1. Erros das últimas 3 horas agrupados por tipo
fields @timestamp, message, error_type, pedido_id
| filter level = "ERROR"
| stats count(*) as total by error_type
| sort total desc

-- 2. Latência p95 e p99 por hora (usando campo duracao_ms do log)
fields @timestamp, duracao_ms
| filter ispresent(duracao_ms)
| stats
    pct(duracao_ms, 95) as p95,
    pct(duracao_ms, 99) as p99,
    avg(duracao_ms) as media
  by bin(1h)

-- 3. Cold starts e seus requestIds (para cruzar com X-Ray)
fields @timestamp, requestId, cold_start, message
| filter cold_start = true
| sort @timestamp desc
| limit 100

-- 4. No log group /aws/lambda/insights — funções com memory > 80%
fields @timestamp, function_name, memory_utilization, duration, cold_start
| filter memory_utilization > 80
| sort memory_utilization desc
| limit 50
```

---

## Armadilhas comuns

### Armadilha 1 — `print()` com objeto JSON não é o mesmo que log estruturado verdadeiro

**O erro:** O desenvolvedor faz `print(json.dumps({"level": "INFO", "message": "ok"}))` e assume que CloudWatch Logs Insights vai parsear como JSON. Funciona — mas o timestamp gerado pelo Lambda para a linha de log não está dentro do JSON, dificultando a ordenação. Além disso, se o objeto JSON contiver quebras de linha, o CloudWatch pode interpretá-lo como múltiplos eventos de log.

**Por que acontece:** CloudWatch Logs captura cada linha (`\n`) como um evento separado. Se o `json.dumps` não tiver `separators=(',', ':')` e produzir JSON multi-linha, o evento é fragmentado.

**Como evitar:**
- Use sempre `json.dumps(obj, separators=(',', ':'))` (sem espaços) para garantir que o JSON seja uma linha única.
- Ou use `json.dumps(obj)` sem `indent` (que é o default — sem indent produz uma linha).
- Para o timestamp, confie no campo `@timestamp` que o CloudWatch adiciona automaticamente — não é necessário incluir timestamp no JSON (mas incluir não prejudica e facilita correlação).

---

### Armadilha 2 — `patch_all()` do X-Ray SDK fora do handler causa erros em ambiente de teste

**O erro:** O desenvolvedor chama `patch_all()` no escopo global do módulo. Em testes unitários sem o daemon X-Ray rodando, o SDK tenta registrar o trace e falha com `SegmentNotFoundException: cannot find the current segment/subsegment`.

**Por que acontece:** `patch_all()` monkeypatches os clientes boto3 globalmente. Em testes, não há contexto X-Ray ativo — o daemon não está rodando e não há segment aberto.

**Como evitar:**
- Configure o SDK para ignorar erros quando não há contexto: `xray_recorder.configure(context_missing='LOG_ERROR')` (padrão em Lambda) ou `'IGNORE_ERROR'`.
- Em testes, configure via variável de ambiente: `AWS_XRAY_CONTEXT_MISSING=LOG_ERROR`.
- O CDK/Lambda já configura isso automaticamente quando tracing está habilitado, mas ambientes de teste local podem não ter essa variável.

```python
from aws_xray_sdk.core import xray_recorder, patch_all
xray_recorder.configure(context_missing='IGNORE_ERROR')
patch_all()
```

---

### Armadilha 3 — Lambda Insights sem permissão `CloudWatchLambdaInsightsExecutionRolePolicy` faz a extensão falhar silenciosamente

**O erro:** Lambda Insights é habilitado (layer adicionada), mas as métricas não aparecem em `/aws/lambda/insights`. A função roda normalmente, mas nenhum dado chega.

**Por que acontece:** A extensão Lambda Insights precisa de permissão para escrever logs no log group `/aws/lambda/insights` com permissões específicas: `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents`. Sem essas permissões, a extensão falha ao inicializar e é ignorada — não lança erro na invocação principal.

**Como evitar:**
- Com CDK: `insights_version=lambda_.LambdaInsightsVersion.*` adiciona a managed policy automaticamente.
- Manualmente: adicione `CloudWatchLambdaInsightsExecutionRolePolicy` (AWS managed) à execution role da função.
- Para verificar: cheque os logs da extensão no log group `/aws/lambda/insights` ou ative `LAMBDA_INSIGHTS_LOG_LEVEL=info` nas variáveis de ambiente.

```python
# CDK faz isso automaticamente, mas se precisar fazer manualmente:
fn.role.add_managed_policy(
    iam.ManagedPolicy.from_aws_managed_policy_name(
        "CloudWatchLambdaInsightsExecutionRolePolicy"
    )
)
```

---

## Exercício de reflexão

Você tem uma função Lambda que processa pagamentos e está recebendo reclamações de usuários de que "alguns pagamentos não processam". O sistema não tem observabilidade configurada além dos logs padrão do Lambda (texto plano, sem correlação). Você precisa propor uma solução de observabilidade que permita, dado o ID de uma transação reportada pelo usuário, encontrar em menos de 5 minutos: (a) o log completo da invocação que processou aquela transação, (b) se houve retry ou cold start, (c) quais chamadas externas (banco, API de pagamento) foram feitas e qual delas foi mais lenta, e (d) se o problema é sistêmico (afeta X% das transações) ou pontual.

**Questão:** Quais campos você incluiria nos logs estruturados? Como configuraria X-Ray para rastrear a chamada à API de pagamento (que é HTTP externo, não AWS)? Que query de CloudWatch Logs Insights usaria para identificar se o problema é sistêmico? Onde Lambda Insights ajudaria (ou não ajudaria) nesse diagnóstico?

---

## Recursos para aprofundar

1. **Monitor function performance with Amazon CloudWatch Lambda Insights**
   URL: https://docs.aws.amazon.com/lambda/latest/dg/monitoring-insights.html
   Guia oficial de habilitação do Lambda Insights com ARNs das layers por região, passo a passo via console/CDK/CLI, e lista completa de métricas coletadas. Inclui como interpretar o dashboard `/LambdaInsights`.

2. **Visualize Lambda function invocations using AWS X-Ray**
   URL: https://docs.aws.amazon.com/lambda/latest/dg/services-xray.html
   Explica a integração nativa do X-Ray com Lambda: como os segmentos são criados automaticamente, como habilitar active tracing, e como usar o SDK X-Ray Python dentro de funções Lambda. Inclui exemplos de subsegmentos e configuração de sampling.

3. **Configuring JSON and plain text log formats**
   URL: https://docs.aws.amazon.com/lambda/latest/dg/monitoring-cloudwatchlogs-logformat.html
   Documentação do novo formato JSON nativo para logs de sistema (START, END, REPORT). Descreve os campos emitidos em cada tipo de evento, como configurar via console/CLI/CDK, e como usar com CloudWatch Logs Insights.

---

