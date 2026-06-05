# Sessão 036 — CloudWatch: Custom Metrics com EMF (Embedded Metrics Format)

**Duração estimada:** 60 minutos  
**Dependências:** session-018-ecs-observabilidade-firelens-xray, session-024-lambda-observabilidade-xray-insights

---

## Objetivo

Ao final desta sessão, você conseguirá emitir métricas customizadas de uma função Lambda usando o formato EMF (via `print()` para stdout em JSON estruturado, sem API call separada), criar um dashboard CloudWatch que plota essas métricas, explicar por que EMF é preferível a `put-metric-data` em termos de latência e custo, e usar o AWS Lambda Powertools para simplificar a emissão.

---

## Contexto

[FATO] O **Embedded Metrics Format (EMF)** é uma especificação JSON do CloudWatch que instrui o serviço CloudWatch Logs a extrair automaticamente métricas customizadas de log events estruturados. Em vez de fazer uma chamada de API separada ao CloudWatch, a função escreve um JSON especialmente formatado no stdout — o Lambda Logs Agent captura isso e encaminha ao CloudWatch Logs, que extrai as métricas de forma assíncrona.

[CONSENSO] EMF é a abordagem canônica para métricas customizadas em Lambda por três razões: (1) não bloqueia a execução da função (assíncrono via logs); (2) elimina o custo de `PutMetricData` por API call; (3) o JSON EMF também fica disponível como log event no CloudWatch Logs Insights para debugging.

---

## Conceitos principais

### 1. Estrutura do documento EMF

[FATO] Um documento EMF válido é um JSON com a chave `_aws` como metadado obrigatório, mais os valores das métricas e dimensões como campos no nível raiz:

```json
{
  "_aws": {
    "Timestamp": 1574109732004,
    "CloudWatchMetrics": [
      {
        "Namespace": "MyApp/Payments",
        "Dimensions": [["service", "environment"]],
        "Metrics": [
          { "Name": "ProcessingLatency", "Unit": "Milliseconds", "StorageResolution": 60 },
          { "Name": "SuccessfulPayments",  "Unit": "Count",        "StorageResolution": 60 }
        ]
      }
    ]
  },
  "service":             "payment-processor",
  "environment":         "production",
  "ProcessingLatency":   45.3,
  "SuccessfulPayments":  1,
  "orderId":             "ord-abc-123"
}
```

[FATO] Regras estruturais do EMF:
- `_aws.Timestamp`: epoch em **milissegundos** (obrigatório)
- `_aws.CloudWatchMetrics`: array de `MetricDirective` (obrigatório)
- Cada `MetricDirective` deve ter `Namespace`, `Dimensions` (array de arrays de strings), e `Metrics` (array de `MetricDefinition`)
- Máximo de **100 métricas** por documento EMF
- Máximo de **30 dimensões** por DimensionSet
- Valores de métrica: número ou array de números (máximo 100 elementos)
- Campos extras além das métricas/dimensões (como `orderId` acima) são **preservados no log** mas **não viram métricas** — servem como contexto para Logs Insights

[FATO] Unidades válidas: `Seconds`, `Microseconds`, `Milliseconds`, `Bytes`, `Kilobytes`, `Megabytes`, `Gigabytes`, `Terabytes`, `Bits`, `Kilobits`, `Megabits`, `Gigabits`, `Terabits`, `Percent`, `Count`, `Bytes/Second`, `Kilobytes/Second`, `Megabytes/Second`, `Gigabytes/Second`, `Terabytes/Second`, `Bits/Second`, `Count/Second`, `None`

---

### 2. EMF vs. PutMetricData: comparação de custo e latência

[FATO] Existem dois mecanismos de ingestão de métricas customizadas no CloudWatch:

```
                    EMF (via CloudWatch Logs)     PutMetricData API
────────────────────────────────────────────────────────────────────
Execução            Assíncrona — print() retorna  Síncrona — bloqueia
                    imediatamente                 até resposta da API
Latência na função  Nenhuma adicionada            Latência de rede
                                                  (tipicamente 10-50ms)
Custo de escrita    Custo de logs ingestion       $0.01 por 1.000 API calls
                    ($0.50/GB — us-east-1)        (independente do volume)
Custo de métrica    Igual: $0.30/métrica/mês      Igual: $0.30/métrica/mês
                    (primeiras 10.000)            (primeiras 10.000)
Permissão requerida logs:PutLogEvents             cloudwatch:PutMetricData
Dados de contexto   Log event completo disponível Apenas dados da métrica
                    no Logs Insights
Limite de batch     100 métricas por blob EMF     20 métricas por
                                                  PutMetricData call
```

[CONSENSO] Para funções Lambda com alta taxa de invocações, EMF é financeiramente superior porque você não paga por API call — paga apenas pela ingestão de logs (que já ocorreria de qualquer forma). A break-even point é baixa: qualquer função com mais de ~1.000 invocações/dia normalmente economiza com EMF.

[OPINIÃO — AWS Well-Architected Serverless Lens] EMF é a abordagem recomendada para métricas customizadas em workloads serverless por eliminar chamadas síncronas que aumentam a duração faturável da função.

---

### 3. High-resolution metrics

[FATO] O campo `StorageResolution` no MetricDefinition define a granularidade de armazenamento:

```
StorageResolution = 60  →  Standard resolution: CloudWatch armazena em
                            granularidade de 1 minuto
                            (default, menor custo)

StorageResolution = 1   →  High resolution: CloudWatch armazena em
                            granularidade de 1 segundo
                            (útil para anomaly detection em tempo real,
                            alertas sub-minuto)
```

[FATO] High-resolution metrics são cobradas da mesma forma que standard em termos de custo de métrica ($0.30/métrica/mês), mas consomem mais armazenamento interno. Alarms em high-resolution metrics podem ter período mínimo de 10 ou 30 segundos.

---

### 4. Unique metric = nome + namespace + dimensões

[FATO] O CloudWatch define uma **métrica única** pela combinação de: `metric_name` + `namespace` + `{dimension_key: dimension_value, ...}`. Esta distinção é crítica para entender o custo:

```
Métrica A:  Namespace="MyApp", Name="Latency", {service="payment"}
Métrica B:  Namespace="MyApp", Name="Latency", {service="checkout"}
→ São 2 métricas distintas, cobradas separadamente.

Armadilha de alta cardinalidade:
Namespace="MyApp", Name="Latency", {requestId="abc-123"}
Namespace="MyApp", Name="Latency", {requestId="def-456"}
→ Cada requestId único cria uma nova métrica!
   1M de requisições/dia = potencialmente 1M de métricas novas/dia
   = custo explosivo ($0.30 × 1.000.000 = $300.000/mês)
```

[FATO] Dimensões de alta cardinalidade (`requestId`, `userId`, `sessionId`) **nunca devem ser usadas como dimensões**. Use-as como **campos de metadata** no EMF (ficam no log, pesquisáveis via Logs Insights, sem custo de métrica).

---

### 5. AWS Lambda Powertools — Metrics utility

[FATO] O módulo `aws_lambda_powertools.metrics` é uma abstração sobre EMF que: valida o schema, serializa o JSON, faz flush no decorator `@log_metrics`, e impede criação acidental de métricas com dimensões de alta cardinalidade.

```
Comportamento do Metrics (singleton compartilhado entre módulos):
  - Acumula métricas em memória durante a invocação
  - Flush automático no fim do handler (via @log_metrics)
  - Flush automático ao atingir 100 métricas (limite EMF)
  - Valida unidades, namespace, dimensões na emissão

EphemeralMetrics (não-singleton):
  - Instância isolada — não compartilha estado
  - Útil para multi-tenant ou métricas com dimensões completamente distintas
```

---

## Exemplo prático

### Cenário: API de pagamentos — métricas de negócio e operacionais

#### CDK Python — Lambda + Layer Powertools + Dashboard

```python
from aws_cdk import (
    Stack, Duration, RemovalPolicy,
    aws_lambda as lambda_,
    aws_logs as logs,
    aws_cloudwatch as cw,
    aws_cloudwatch_actions as cw_actions,
    aws_sns as sns,
)
from constructs import Construct


class PaymentMetricsStack(Stack):
    def __init__(self, scope: Construct, construct_id: str, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        # ── Lambda Powertools Layer (ARN oficial por região/versão) ───
        # Consulte: https://docs.aws.amazon.com/powertools/python/latest/
        powertools_layer = lambda_.LayerVersion.from_layer_version_arn(
            self, "PowertoolsLayer",
            layer_version_arn=(
                f"arn:aws:lambda:{self.region}:017000801446:"
                "layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:31"
            ),
        )

        # ── Função Lambda ─────────────────────────────────────────────
        payment_fn = lambda_.Function(
            self, "PaymentProcessor",
            function_name="payment-processor",
            runtime=lambda_.Runtime.PYTHON_3_12,
            handler="handler.lambda_handler",
            code=lambda_.Code.from_asset("lambda/payment"),
            timeout=Duration.seconds(30),
            memory_size=256,
            layers=[powertools_layer],
            environment={
                "POWERTOOLS_SERVICE_NAME":      "payment-processor",
                "POWERTOOLS_METRICS_NAMESPACE": "MyApp/Payments",
                "ENVIRONMENT":                  "production",
                "LOG_LEVEL":                    "INFO",
            },
            log_retention=logs.RetentionDays.ONE_WEEK,
        )

        # ── Dashboard CloudWatch ──────────────────────────────────────
        dashboard = cw.Dashboard(
            self, "PaymentDashboard",
            dashboard_name="payment-metrics",
        )

        # Widget 1: Taxa de sucesso vs falha
        dashboard.add_widgets(
            cw.GraphWidget(
                title="Payment Success vs Failure Rate",
                width=12,
                left=[
                    cw.Metric(
                        namespace="MyApp/Payments",
                        metric_name="SuccessfulPayments",
                        dimensions_map={"service": "payment-processor", "environment": "production"},
                        statistic="Sum",
                        period=Duration.minutes(1),
                        color="#2ca02c",
                    ),
                    cw.Metric(
                        namespace="MyApp/Payments",
                        metric_name="FailedPayments",
                        dimensions_map={"service": "payment-processor", "environment": "production"},
                        statistic="Sum",
                        period=Duration.minutes(1),
                        color="#d62728",
                    ),
                ],
            ),
            # Widget 2: Latência de processamento (p50, p95, p99)
            cw.GraphWidget(
                title="Processing Latency (ms)",
                width=12,
                left=[
                    cw.Metric(
                        namespace="MyApp/Payments",
                        metric_name="ProcessingLatency",
                        dimensions_map={"service": "payment-processor", "environment": "production"},
                        statistic="p50",
                        period=Duration.minutes(1),
                        label="p50",
                        color="#1f77b4",
                    ),
                    cw.Metric(
                        namespace="MyApp/Payments",
                        metric_name="ProcessingLatency",
                        dimensions_map={"service": "payment-processor", "environment": "production"},
                        statistic="p95",
                        period=Duration.minutes(1),
                        label="p95",
                        color="#ff7f0e",
                    ),
                    cw.Metric(
                        namespace="MyApp/Payments",
                        metric_name="ProcessingLatency",
                        dimensions_map={"service": "payment-processor", "environment": "production"},
                        statistic="p99",
                        period=Duration.minutes(1),
                        label="p99",
                        color="#d62728",
                    ),
                ],
            ),
        )

        # Widget 3: Valor total processado
        dashboard.add_widgets(
            cw.GraphWidget(
                title="Payment Value Processed (USD)",
                width=12,
                left=[
                    cw.Metric(
                        namespace="MyApp/Payments",
                        metric_name="PaymentValueUSD",
                        dimensions_map={"service": "payment-processor", "environment": "production"},
                        statistic="Sum",
                        period=Duration.minutes(5),
                    ),
                ],
            ),
            # Widget 4: Cold starts
            cw.GraphWidget(
                title="Cold Starts",
                width=12,
                left=[
                    cw.Metric(
                        namespace="MyApp/Payments",
                        metric_name="ColdStart",
                        dimensions_map={
                            "function_name": "payment-processor",
                            "service": "payment-processor",
                        },
                        statistic="Sum",
                        period=Duration.minutes(5),
                    ),
                ],
            ),
        )

        # ── Alarm em falhas ───────────────────────────────────────────
        alarm_topic = sns.Topic(self, "PaymentAlarmTopic")

        cw.Alarm(
            self, "HighFailureRateAlarm",
            alarm_name="payment-high-failure-rate",
            metric=cw.Metric(
                namespace="MyApp/Payments",
                metric_name="FailedPayments",
                dimensions_map={"service": "payment-processor", "environment": "production"},
                statistic="Sum",
                period=Duration.minutes(5),
            ),
            threshold=10,
            evaluation_periods=2,
            comparison_operator=cw.ComparisonOperator.GREATER_THAN_OR_EQUAL_TO_THRESHOLD,
            treat_missing_data=cw.TreatMissingData.NOT_BREACHING,
        ).add_alarm_action(cw_actions.SnsAction(alarm_topic))
```

#### Lambda handler — EMF via Powertools e EMF manual

```python
# lambda/payment/handler.py
import json
import os
import time
from typing import Any

from aws_lambda_powertools import Logger, Metrics
from aws_lambda_powertools.metrics import MetricUnit, MetricResolution
from aws_lambda_powertools.utilities.typing import LambdaContext

# Inicializado globalmente (singleton compartilhado)
# Namespace e service vêm de env vars:
#   POWERTOOLS_METRICS_NAMESPACE="MyApp/Payments"
#   POWERTOOLS_SERVICE_NAME="payment-processor"
logger = Logger()
metrics = Metrics()

# Dimensão adicional compartilhada por todas as métricas
ENVIRONMENT = os.environ.get("ENVIRONMENT", "development")
metrics.set_default_dimensions(environment=ENVIRONMENT)


@metrics.log_metrics(capture_cold_start_metric=True)
@logger.inject_lambda_context
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    """
    @metrics.log_metrics:
      - Serializa e faz flush do EMF blob no stdout ao final do handler
      - Captura cold start em blob EMF separado (dimensão function_name)
      - Se ocorrer exceção, o flush ainda é executado
    """
    start_time = time.perf_counter()

    order_id   = event.get("orderId", "unknown")
    amount_usd = float(event.get("amountUSD", 0))
    currency   = event.get("currency", "USD")

    try:
        # Simula processamento
        result = _process_payment(order_id, amount_usd, currency)

        # ── Métricas de negócio ──────────────────────────────────────
        metrics.add_metric(
            name="SuccessfulPayments",
            unit=MetricUnit.Count,
            value=1,
        )
        metrics.add_metric(
            name="PaymentValueUSD",
            unit=MetricUnit.Count,      # CloudWatch não tem "Currency"
            value=amount_usd,
        )

        # ── Metadata de alta cardinalidade (NO log, NOT a dimension) ──
        # orderId vai para o log event mas NÃO cria nova métrica por invocação
        metrics.add_metadata(key="orderId",   value=order_id)
        metrics.add_metadata(key="currency",  value=currency)
        metrics.add_metadata(key="processor", value=result.get("processor"))

        return {"statusCode": 200, "body": json.dumps(result)}

    except ValueError as e:
        # Erro de validação do pedido
        metrics.add_metric(name="FailedPayments", unit=MetricUnit.Count, value=1)
        metrics.add_metadata(key="errorType",    value="ValidationError")
        metrics.add_metadata(key="errorMessage", value=str(e))
        metrics.add_metadata(key="orderId",      value=order_id)
        logger.error("Payment validation failed", extra={"orderId": order_id, "error": str(e)})
        return {"statusCode": 400, "body": json.dumps({"error": str(e)})}

    except Exception as e:
        metrics.add_metric(name="FailedPayments", unit=MetricUnit.Count, value=1)
        metrics.add_metadata(key="errorType",    value=type(e).__name__)
        metrics.add_metadata(key="orderId",      value=order_id)
        logger.exception("Unexpected payment error", extra={"orderId": order_id})
        return {"statusCode": 500, "body": json.dumps({"error": "Internal error"})}

    finally:
        # Latência sempre emitida, independente de sucesso/falha
        elapsed_ms = (time.perf_counter() - start_time) * 1000
        metrics.add_metric(
            name="ProcessingLatency",
            unit=MetricUnit.Milliseconds,
            value=elapsed_ms,
            resolution=MetricResolution.High,  # StorageResolution=1: sub-minuto
        )


def _process_payment(order_id: str, amount: float, currency: str) -> dict:
    if amount <= 0:
        raise ValueError(f"Invalid amount: {amount}")
    if currency not in ("USD", "EUR", "BRL"):
        raise ValueError(f"Unsupported currency: {currency}")
    time.sleep(0.02)  # simula chamada externa
    return {"orderId": order_id, "status": "approved", "processor": "stripe"}
```

#### EMF manual sem Powertools (para entender o mecanismo)

```python
import json
import time

def emit_emf_manually(metric_name: str, value: float, unit: str,
                      namespace: str, dimensions: dict) -> None:
    """
    Emite uma métrica EMF via print() — sem nenhuma dependência externa.
    O Lambda Logs Agent captura o stdout e envia ao CloudWatch Logs.
    CloudWatch Logs extrai a métrica automaticamente.
    """
    emf_document = {
        "_aws": {
            "Timestamp": int(time.time() * 1000),   # epoch em milissegundos
            "CloudWatchMetrics": [
                {
                    "Namespace": namespace,
                    "Dimensions": [list(dimensions.keys())],  # array de arrays
                    "Metrics": [
                        {"Name": metric_name, "Unit": unit, "StorageResolution": 60}
                    ],
                }
            ],
        },
        metric_name: value,
        **dimensions,   # dimensões no nível raiz
    }
    # Uma linha por documento EMF (sem quebra de linha interna)
    print(json.dumps(emf_document))
```

#### CLI — Criar métricas manualmente e criar dashboard

```bash
# 1. Emitir uma métrica via PutMetricData (para comparação com EMF)
aws cloudwatch put-metric-data \
  --namespace "MyApp/Payments" \
  --metric-name "ManualTestMetric" \
  --value 42 \
  --unit Count \
  --dimensions service=payment-processor,environment=production

# 2. Verificar se as métricas EMF foram criadas
aws cloudwatch list-metrics \
  --namespace "MyApp/Payments" \
  --query 'Metrics[*].{Name:MetricName,Dims:Dimensions}'

# 3. Obter estatísticas de latência (últimos 15 minutos)
aws cloudwatch get-metric-statistics \
  --namespace "MyApp/Payments" \
  --metric-name "ProcessingLatency" \
  --dimensions Name=service,Value=payment-processor \
               Name=environment,Value=production \
  --start-time "$(date -u -d '15 minutes ago' '+%Y-%m-%dT%H:%M:%SZ' 2>/dev/null || date -u -v-15M '+%Y-%m-%dT%H:%M:%SZ')" \
  --end-time "$(date -u '+%Y-%m-%dT%H:%M:%SZ')" \
  --period 60 \
  --statistics Average Minimum Maximum p95 p99 \
  --extended-statistics p95 p99

# 4. Criar alarm em taxa de erros alta
aws cloudwatch put-metric-alarm \
  --alarm-name "payment-high-failure-rate" \
  --alarm-description "More than 10 payment failures in 5 minutes" \
  --namespace "MyApp/Payments" \
  --metric-name "FailedPayments" \
  --dimensions Name=service,Value=payment-processor Name=environment,Value=production \
  --statistic Sum \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 10 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --treat-missing-data notBreaching

# 5. Verificar JSON EMF gerado pelo Lambda nos logs
LOG_GROUP="/aws/lambda/payment-processor"
LATEST_STREAM=$(aws logs describe-log-streams \
  --log-group-name "$LOG_GROUP" \
  --order-by LastEventTime \
  --descending \
  --limit 1 \
  --query 'logStreams[0].logStreamName' \
  --output text)

aws logs get-log-events \
  --log-group-name "$LOG_GROUP" \
  --log-stream-name "$LATEST_STREAM" \
  --limit 20 \
  --query 'events[*].message' \
  --output text | grep -A5 '"_aws"' | head -50

# 6. Consultar métricas e orderId via Logs Insights
# (os campos de metadata ficam pesquisáveis mesmo não sendo dimensões)
aws logs start-query \
  --log-group-name "/aws/lambda/payment-processor" \
  --start-time "$(date -u -d '1 hour ago' +%s 2>/dev/null || date -u -v-1H +%s)" \
  --end-time "$(date -u +%s)" \
  --query-string '
    fields @timestamp, orderId, ProcessingLatency, errorType
    | filter ispresent(FailedPayments)
    | sort @timestamp desc
    | limit 20
  '
```

---

## Armadilhas comuns

**1. Dimensões de alta cardinalidade = explosão de custo**  
[FATO] Cada combinação única de (namespace + metric_name + dimensions) é uma métrica distinta cobrada a $0.30/mês (para as primeiras 10.000). Usar `requestId`, `userId` ou `sessionId` como dimensão com 1M de valores únicos/mês gera 1M de novas métricas = $300.000/mês. Use `add_metadata()` para dados de alta cardinalidade — ficam no log, não criam métricas.

**2. EMF em múltiplas linhas não é processado**  
[FATO] O documento EMF deve ser um único objeto JSON em **uma única linha** no stdout. Se o JSON estiver pretty-printed (com quebras de linha), o CloudWatch Logs não reconhece o formato e descarta a extração de métricas silenciosamente. O Powertools garante isso; no modo manual, use `json.dumps(doc)` sem `indent`.

**3. Métricas ou dimensões definidas fora do handler (escopo global)**  
[CONSENSO] Dimensões ou métricas adicionadas no escopo global da função (`import` time) só são aplicadas durante o cold start. Nas invocações seguintes, o estado global persiste entre chamadas na mesma instância, mas não é reinicializado. O Powertools avisa sobre isso. Dimensões permanentes devem ser configuradas via `set_default_dimensions()`.

**4. Falta de flush em exceção sem o decorator**  
[FATO] Se você usa `metrics.flush_metrics()` manualmente (sem o decorator `@log_metrics`), uma exceção não capturada em um bloco intermediário pode deixar métricas não emitidas. Sempre use `try/finally` ou use o decorator, que garante flush mesmo em caso de exceção.

**5. `StorageResolution=1` (high-resolution) sem necessidade real**  
[CONSENSO] High-resolution metrics permitem alarms com período de 10/30 segundos, mas não custam mais em métricas — o custo extra é marginal em armazenamento. O problema é que alarms em high-resolution metrics consomem mais leituras de avaliação, aumentando levemente o custo de alarm. Use high-resolution apenas quando a granularidade sub-minuto for realmente necessária para o SLA.

**6. Namespace com alta granularidade cria silos de visibilidade**  
[CONSENSO] Usar namespaces muito específicos (ex: `MyApp/Payments/OrderType/Subscription`) fragmenta as métricas em silos difíceis de correlacionar no dashboard. Use um namespace por serviço/aplicação e use dimensões para segmentar — as dimensões são filtráveis no console e nas queries.

---

## Exercício de reflexão

Você tem uma função Lambda que processa eventos de uma fila SQS. Para cada mensagem, você quer monitorar:
- `MessageProcessed` (Count) — por tipo de mensagem (`messageType`: order, refund, notification)
- `ProcessingDuration` (Milliseconds) — latência do processamento
- O `messageId` original para correlação com logs

1. Como você estruturaria as dimensões para `MessageProcessed` considerando que há 3 tipos de mensagem (`order`, `refund`, `notification`)? Quantas métricas distintas isso cria no CloudWatch? Qual seria o problema se você usasse `messageId` como dimensão?

2. Escreva o trecho de código Python com Powertools que emite `MessageProcessed` e `ProcessingDuration` com a dimensão `messageType`, mantendo `messageId` como metadata (não dimensão). Use `@metrics.log_metrics`.

3. O EMF blob gerado pelo Powertools é também um log event em CloudWatch Logs. Escreva uma query Logs Insights que lista os últimos 10 `messageId` de mensagens do tipo `refund` com `ProcessingDuration > 500ms`.

---

## Recursos para aprofundar

- [FATO] EMF Specification (estrutura JSON, limites, schema): https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Embedded_Metric_Format_Specification.html
- [FATO] Embedding metrics within logs (overview): https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Embedded_Metric_Format.html
- [FATO] Powertools for AWS Lambda (Python) — Metrics: https://docs.aws.amazon.com/powertools/python/latest/core/metrics/
- [FATO] CloudWatch Pricing (custom metrics, ingestion): https://aws.amazon.com/cloudwatch/pricing/

---

