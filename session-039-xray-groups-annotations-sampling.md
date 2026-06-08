# Sessão 039 — X-Ray: groups, annotations, sampling rules e integração cross-service

**Duração estimada:** 60 minutos  
**Dependências:** session-038-cloudwatch-composite-alarms-anomaly

---

## Objetivo

Ao final desta sessão, você conseguirá criar X-Ray Groups para filtrar traces por anotação customizada, configurar sampling rules para aumentar amostragem em endpoints críticos sem inflar custo, navegar um trace distribuído que atravessa ALB → Lambda → DynamoDB identificando onde o tempo foi gasto, e distinguir annotations de metadata para uso em filter expressions.

---

## Contexto

[FATO] AWS X-Ray é um serviço de distributed tracing que coleta dados de segmentos enviados por aplicações instrumentadas e os agrupa em traces por `TraceId`. Cada request gera um trace que atravessa todos os serviços participantes, permitindo identificar onde a latência ocorre e onde erros se originam.

[FATO] X-Ray integra nativamente com ALB, API Gateway, Lambda, ECS, EC2, e AWS SDK clients (DynamoDB, SQS, SNS etc.). Para Lambda, o tracing é habilitado via configuração — sem modificar código. Para outros serviços, é necessário instrumentar o código com o X-Ray SDK.

---

## Conceitos principais

### 1. Anatomia de um trace: segmento, subsegmento, inferred segment

[FATO] A hierarquia de dados do X-Ray:

```
Trace (TraceId)
│  Agrupa todos os segmentos de uma mesma requisição
│  Retido por 30 dias
│
├── Segment (enviado por cada serviço instrumentado)
│     ├── name, trace_id, id (segment_id)
│     ├── start_time, end_time
│     ├── http request/response
│     ├── annotations (indexadas para filter expressions)
│     ├── metadata (não indexada, qualquer tipo)
│     ├── error / fault / throttle flags
│     └── subsegments
│           ├── AWS SDK calls (DynamoDB, S3, SQS...)
│           ├── HTTP downstream calls
│           └── custom subsegments (código arbitrário)
│
└── Inferred segment (gerado pelo X-Ray para serviços sem SDK)
      Criado a partir do subsegmento upstream que chamou o serviço
      Ex: DynamoDB não envia segmento → X-Ray infere pelo subsegmento
          do Lambda que fez a chamada ao DynamoDB
```

[FATO] `Segment documents` têm tamanho máximo de **64 KB**. Dados além desse limite são truncados.

**Tracing header — propagação cross-service:**  
[FATO] O primeiro serviço que recebe a requisição adiciona o header `X-Amzn-Trace-Id` com `Root` (TraceId), `Parent` (segmentId do serviço atual), e `Sampled` (0 ou 1). Todos os serviços downstream propagam esse header, mantendo o TraceId original.

```
X-Amzn-Trace-Id: Root=1-5759e988-bd862e3fe1be46a994272793;Parent=53995c3f42cd8ad8;Sampled=1
```

---

### 2. Sampling rules: taxa padrão e configuração por endpoint

[FATO] **Regra padrão (default rule)**: primeiro request por segundo + 5% dos requests adicionais. Esta regra é conservadora para controle de custos.

[FATO] Sampling rules são avaliadas em **ordem de prioridade** (menor número = maior prioridade, de 1 a 9999). A default rule tem prioridade 10000 e é sempre avaliada por último.

```
Estrutura de uma sampling rule:
─────────────────────────────────────────────────────────────
RuleName:        Nome identificador (único)
Priority:        1–9999 (menor = maior prioridade)
ReservoirSize:   Requests/segundo sempre amostrados (reservoir)
FixedRate:       % adicional amostrada além do reservoir (0.0–1.0)
ServiceName:     Filtro por nome do serviço (wildcard * permitido)
ServiceType:     Filtro por tipo: "AWS::Lambda::Function", "AWS::ECS::Container", etc.
Host:            Filtro por host
HTTPMethod:      GET, POST, PUT, DELETE, * (wildcard)
URLPath:         Filtro por path (wildcard * permitido)
ResourceARN:     Filtro por ARN do recurso

Custo de sampling:
  ReservoirSize = 10, FixedRate = 0.05 significa:
  - Primeiros 10 req/s: TODOS amostrados
  - Além disso: 5% amostrados
```

[FATO] Sampling rules são **gerenciadas centralmente** no X-Ray (não por instância ou função). O X-Ray SDK consulta as regras periodicamente (a cada 10 segundos) e as aplica localmente, sem latência adicional por decisão de sampling.

---

### 3. Annotations vs. Metadata

[FATO] Dois tipos de dados customizados podem ser adicionados a segmentos e subsegmentos:

```
                Annotations                 Metadata
────────────────────────────────────────────────────────────────
Indexação       SIM — indexado para         NÃO — não pesquisável
                filter expressions           via filter expressions
Tipos           Boolean, Number, String     Qualquer tipo JSON
                                            (objetos, arrays, etc.)
Uso típico      UserId, orderId, endpoint,  Payload completo,
                versão do app, feature flag configurações, dados
                                            de debugging
Limite          50 annotations por trace    Não documentado limite rígido
                                            (limitado pelo 64KB do segment)
Visibilidade    Console X-Ray, GetTrace     Console X-Ray,
                Summaries, filter expr.     GetTraceSummaries (sem filtro)
SDK             put_annotation(key, value)  put_metadata(key, value,
                                            namespace="default")
```

---

### 4. Filter expressions: sintaxe completa

[FATO] Filter expressions são usadas tanto no console (busca de traces) quanto na definição de Groups. Sintaxe: `keyword operator value`, combinados com `AND` / `OR`.

```
Tipo de keyword     Exemplos
────────────────────────────────────────────────────────────────
Boolean             ok, error, fault, throttle, partial
                    (usados sem operador: "fault" = trace com 5XX)
                    ou com: "ok = false"

Number              responsetime > 2
                    duration >= 5 AND duration <= 8
                    http.status != 200
                    http.status = 429

String              http.url CONTAINS "/api/payments"
                    http.method = "POST"
                    http.url BEGINSWITH "https://api."
                    user CONTAINS ""      (field exists check)
                    name = "payment-service"

Annotation          annotation[userId] = "u-abc123"
                    annotation[orderId] CONTAINS "ORD-"
                    annotation[isPremium] = true
                    annotation[retryCount] > 2
                    !annotation[userId]   (annotation não presente)

Complex — service   service("payment-lambda") { fault }
                    service() { fault }          (qualquer serviço)
                    service(id(name: "payment", type: "AWS::Lambda::Function")) { error }

Complex — edge      edge("alb", "payment-lambda") { error }

Group               group.name = "payment-errors" AND user = "alice"
```

---

### 5. X-Ray Groups: service graph e métricas por contexto

[FATO] Um Group é uma coleção de traces definida por uma filter expression. Quando criado, o X-Ray:
1. Compara traces entrantes contra a filter expression ao armazená-los
2. Gera um **service graph separado** para os traces que correspondem ao grupo
3. Publica **métricas CloudWatch** (namespace `AWS/X-Ray`) para o grupo a cada minuto: `ApproximateTraceCount`, `Throttle`, `Fault`, `Error`

[FATO] Groups são **cobrados por trace recuperado** que corresponde à filter expression — não pela criação do grupo.

[FATO] Atualizar a filter expression de um Group não afeta traces já armazenados — apenas traces futuros. Para evitar dados mesclados de expressões antigas e novas, delete o grupo e recrie.

---

## Exemplo prático

### Cenário: API de pagamentos — trace ALB → Lambda → DynamoDB

**Fluxo:** ALB → Lambda `payment-processor` → DynamoDB `orders-table`

#### CDK Python — Habilitar X-Ray + Sampling Rule + Group

```python
from aws_cdk import (
    Stack, Duration,
    aws_lambda as lambda_,
    aws_xray as xray,
    aws_iam as iam,
)
from constructs import Construct


class XRayObservabilityStack(Stack):
    def __init__(self, scope: Construct, construct_id: str, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        # ── Lambda com X-Ray Active Tracing ───────────────────────────
        payment_fn = lambda_.Function(
            self, "PaymentProcessor",
            function_name="payment-processor",
            runtime=lambda_.Runtime.PYTHON_3_12,
            handler="handler.lambda_handler",
            code=lambda_.Code.from_asset("lambda/payment"),
            timeout=Duration.seconds(30),
            memory_size=256,
            # Active = X-Ray registra TODOS os invocações amostradas
            # PassThrough = X-Ray propaga tracing header mas não registra
            tracing=lambda_.Tracing.ACTIVE,
            environment={
                "POWERTOOLS_SERVICE_NAME": "payment-processor",
            },
        )

        # Permissão para o Lambda enviar traces ao X-Ray
        payment_fn.add_to_role_policy(
            iam.PolicyStatement(
                actions=[
                    "xray:PutTraceSegments",
                    "xray:PutTelemetryRecords",
                    "xray:GetSamplingRules",
                    "xray:GetSamplingTargets",
                ],
                resources=["*"],
            )
        )

        # ── Sampling Rule: mais amostragem em /payments (endpoint crítico) ─
        payment_sampling_rule = xray.CfnSamplingRule(
            self, "PaymentSamplingRule",
            sampling_rule=xray.CfnSamplingRule.SamplingRuleProperty(
                rule_name="payment-high-value-endpoints",
                priority=100,               # Alta prioridade (menor número)
                reservoir_size=50,          # 50 req/s sempre amostrados
                fixed_rate=0.10,            # + 10% dos req adicionais
                service_name="payment-processor",
                service_type="AWS::Lambda::Function",
                http_method="POST",
                url_path="/api/v*/payments*",
                host="*",
                resource_arn="*",
                version=1,
            ),
        )

        # Regra de baixa amostragem para health checks (ruído desnecessário)
        health_check_rule = xray.CfnSamplingRule(
            self, "HealthCheckSamplingRule",
            sampling_rule=xray.CfnSamplingRule.SamplingRuleProperty(
                rule_name="health-check-low-sample",
                priority=50,               # Maior prioridade que payment rule
                reservoir_size=1,          # 1 req/s — só para confirmar que funciona
                fixed_rate=0.0,            # 0% adicional
                service_name="*",
                service_type="*",
                http_method="GET",
                url_path="/health*",
                host="*",
                resource_arn="*",
                version=1,
            ),
        )

        # ── X-Ray Group: traces com falhas em pagamentos ──────────────
        payment_errors_group = xray.CfnGroup(
            self, "PaymentErrorsGroup",
            group_name="payment-errors",
            filter_expression=(
                'service("payment-processor") { fault OR error } '
                'AND annotation[endpoint] BEGINSWITH "/api/v"'
            ),
            insights_configuration=xray.CfnGroup.InsightsConfigurationProperty(
                insights_enabled=True,
                notifications_enabled=True,  # CloudWatch Events para insights
            ),
        )

        # Group para traces lentos (latência > 2s)
        slow_payments_group = xray.CfnGroup(
            self, "SlowPaymentsGroup",
            group_name="slow-payments",
            filter_expression=(
                'service("payment-processor") { responsetime > 2 } '
                'AND annotation[isPremiumUser] = true'
            ),
            insights_configuration=xray.CfnGroup.InsightsConfigurationProperty(
                insights_enabled=True,
            ),
        )
```

#### Python — Lambda handler com X-Ray SDK (Powertools Tracer)

```python
# lambda/payment/handler.py
import json
import time
import boto3
from typing import Any

from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.utilities.typing import LambdaContext

# Tracer: wraps X-Ray SDK — service vem de POWERTOOLS_SERVICE_NAME env var
tracer = Tracer()
logger = Logger()

dynamodb = boto3.resource("dynamodb", region_name="us-east-1")
orders_table = dynamodb.Table("orders-table")


@tracer.capture_lambda_handler   # cria segmento raiz + flush automático
@logger.inject_lambda_context
def lambda_handler(event: dict, context: LambdaContext) -> dict:
    order_id   = event.get("orderId", "unknown")
    user_id    = event.get("userId", "anonymous")
    amount     = float(event.get("amount", 0))
    endpoint   = event.get("requestContext", {}).get("path", "/unknown")
    is_premium = event.get("isPremium", False)

    # ── Annotations: indexadas para filter expressions e Groups ───────
    # Máximo 50 por trace
    tracer.put_annotation(key="orderId",       value=order_id)
    tracer.put_annotation(key="userId",        value=user_id)
    tracer.put_annotation(key="endpoint",      value=endpoint)
    tracer.put_annotation(key="isPremiumUser", value=is_premium)

    # ── Metadata: não indexada, para debugging detalhado ──────────────
    # Útil para dados grandes ou estruturados
    tracer.put_metadata(
        key="requestPayload",
        value={"orderId": order_id, "amount": amount, "isPremium": is_premium},
        namespace="payment",   # namespace organiza metadata por domínio
    )

    try:
        result = _process_payment(order_id, user_id, amount)

        # Annotation de sucesso para filter expressions
        tracer.put_annotation(key="paymentStatus", value="approved")
        tracer.put_annotation(key="processorUsed", value=result.get("processor"))

        return {"statusCode": 200, "body": json.dumps(result)}

    except Exception as e:
        tracer.put_annotation(key="paymentStatus", value="failed")
        tracer.put_annotation(key="errorType",     value=type(e).__name__)
        raise


@tracer.capture_method   # cria subsegmento automático para este método
def _process_payment(order_id: str, user_id: str, amount: float) -> dict:
    """
    @tracer.capture_method cria um subsegmento com o nome do método.
    Aparece no trace timeline como "## _process_payment".
    """
    # Simulando validação com subsegmento customizado
    with tracer.provider.in_subsegment("## validate_payment") as subsegment:
        subsegment.put_annotation("validationStep", "amount_check")
        if amount <= 0:
            raise ValueError(f"Invalid amount: {amount}")
        subsegment.put_annotation("validationResult", "passed")

    # Chamada ao DynamoDB — X-Ray SDK instrumenta automaticamente boto3
    # O AWS SDK call aparece como subsegmento "DynamoDB" no trace
    _record_order(order_id, user_id, amount)

    return {"orderId": order_id, "status": "approved", "processor": "stripe"}


@tracer.capture_method
def _record_order(order_id: str, user_id: str, amount: float) -> None:
    """
    Escrita no DynamoDB — o boto3 já instrumentado pelo X-Ray SDK
    aparece como subsegmento "DynamoDB PutItem" no trace.
    """
    orders_table.put_item(
        Item={
            "PK": f"ORDER#{order_id}",
            "SK": "METADATA",
            "userId": user_id,
            "amount": str(amount),
            "status": "approved",
        }
    )
```

#### CLI — Configurar sampling, criar groups, consultar traces

```bash
# 1. Criar sampling rule para endpoint de pagamentos
aws xray create-sampling-rule \
  --sampling-rule '{
    "RuleName": "payment-high-value-endpoints",
    "Priority": 100,
    "FixedRate": 0.10,
    "ReservoirSize": 50,
    "ServiceName": "payment-processor",
    "ServiceType": "AWS::Lambda::Function",
    "Host": "*",
    "HTTPMethod": "POST",
    "URLPath": "/api/v*/payments*",
    "ResourceARN": "*",
    "Version": 1
  }'

# 2. Criar sampling rule de baixa prioridade para health checks
aws xray create-sampling-rule \
  --sampling-rule '{
    "RuleName": "health-check-low-sample",
    "Priority": 50,
    "FixedRate": 0.0,
    "ReservoirSize": 1,
    "ServiceName": "*",
    "ServiceType": "*",
    "Host": "*",
    "HTTPMethod": "GET",
    "URLPath": "/health*",
    "ResourceARN": "*",
    "Version": 1
  }'

# 3. Listar sampling rules ativas (verificar prioridades)
aws xray get-sampling-rules \
  --query 'SamplingRuleRecords[*].SamplingRule.{Name:RuleName,Priority:Priority,Reservoir:ReservoirSize,Rate:FixedRate,Method:HTTPMethod,Path:URLPath}'

# 4. Criar Group para traces com falha em pagamentos
aws xray create-group \
  --group-name "payment-errors" \
  --filter-expression 'service("payment-processor") { fault OR error } AND annotation[endpoint] BEGINSWITH "/api/v"' \
  --insights-configuration 'InsightsEnabled=true,NotificationsEnabled=true'

# 5. Criar Group para traces lentos de usuários premium
aws xray create-group \
  --group-name "slow-payments" \
  --filter-expression 'service("payment-processor") { responsetime > 2 } AND annotation[isPremiumUser] = true'

# 6. Listar groups existentes
aws xray get-groups \
  --query 'Groups[*].{Name:GroupName,Filter:FilterExpression,ARN:GroupARN}'

# 7. Consultar traces por filter expression (últimos 30 min)
START=$(date -u -d '30 minutes ago' +%s 2>/dev/null || date -u -v-30M +%s)
END=$(date -u +%s)

aws xray get-trace-summaries \
  --start-time $START \
  --end-time $END \
  --filter-expression 'annotation[paymentStatus] = "failed" AND fault' \
  --query 'TraceSummaries[*].{Id:Id,Duration:Duration,Fault:HasFault,Error:HasError}'

# 8. Obter trace completo por ID (substituir <trace-id>)
aws xray batch-get-traces \
  --trace-ids "1-5759e988-bd862e3fe1be46a994272793" \
  --query 'Traces[0].Segments[*].{Id:Id,Document:Document}'

# 9. Ver service graph do Group payment-errors
aws xray get-service-graph \
  --start-time $START \
  --end-time $END \
  --group-name "payment-errors" \
  --query 'Services[*].{Name:Name,Type:Type,Edges:Edges[*].ReferenceId}'

# 10. Métricas CloudWatch geradas automaticamente pelo Group
aws cloudwatch get-metric-statistics \
  --namespace "AWS/X-Ray" \
  --metric-name "ErrorRate" \
  --dimensions Name=GroupName,Value=payment-errors \
  --start-time "$(date -u -d '1 hour ago' '+%Y-%m-%dT%H:%M:%SZ' 2>/dev/null || date -u -v-1H '+%Y-%m-%dT%H:%M:%SZ')" \
  --end-time "$(date -u '+%Y-%m-%dT%H:%M:%SZ')" \
  --period 300 \
  --statistics Average

# 11. Atualizar filter expression de um Group existente
aws xray update-group \
  --group-name "payment-errors" \
  --filter-expression 'service("payment-processor") { fault OR error } AND annotation[endpoint] BEGINSWITH "/api/v" AND duration > 1'
```

#### Navegando um trace distribuído: ALB → Lambda → DynamoDB

```
Trace ID: 1-5759e988-bd862e3fe1be46a994272793

Timeline (total: 1.85s):
├── [ALB]  0ms →  12ms  : ALB segment — recebe request, rota para Lambda
│
├── [Lambda service]  12ms → 48ms : Lambda service segment (cold start)
│   └── @initDuration: 450ms (cold start separado — paralelo ao invocation)
│
├── [Lambda function]  48ms → 1850ms : payment-processor segment
│   ├── ## lambda_handler (subsegment)  48ms → 1850ms
│   │   ├── ## _process_payment         52ms → 1820ms
│   │   │   ├── ## validate_payment    52ms →   55ms  : 3ms (validação local)
│   │   │   └── ## _record_order       55ms → 1820ms : 1765ms  ← GARGALO!
│   │   │       └── [DynamoDB] PutItem  56ms → 1819ms : 1763ms
│   │   │             ← Inferred segment (DynamoDB não envia segmento)
│   │   │             ← 1.76s para um PutItem de 500B → throttling?
│   │
│   Annotations:  orderId=ORD-123, userId=u-abc, endpoint=/api/v1/payments
│                 paymentStatus=approved, processorUsed=stripe
│   Metadata:     requestPayload={orderId: ..., amount: 299.90}
│
Diagnóstico: 95% do tempo gasto em DynamoDB PutItem
Ação: verificar ConsumedWriteCapacityUnits e WriteThrottleEvents
      via CloudWatch para a tabela orders-table
```

---

## Armadilhas comuns

**1. Annotations não aparecem em filter expressions se adicionadas no escopo errado**  
[FATO] Annotations adicionadas em um **subsegmento** são visíveis no detalhe do subsegmento, mas `annotation[key]` em filter expressions busca no **segmento raiz**. Use `tracer.put_annotation()` no nível do handler (segmento raiz) ou explicitamente no segmento correto para que apareçam nos resultados de filter expressions e GetTraceSummaries.

**2. Atualizar filter expression de um Group não retroage**  
[FATO] Traces já armazenados não são re-avaliados quando a filter expression de um Group é atualizada. O service graph do Group pode mostrar dados de ambas as expressões por até 30 dias. Para dados limpos, delete o Group e recrie.

**3. Sampling rules aplicadas pelo SDK localmente — lag de 10 segundos**  
[FATO] O X-Ray SDK busca sampling rules do serviço a cada 10 segundos. Mudanças nas rules não são aplicadas instantaneamente — há uma janela de até 10s de lag. Em funções Lambda com cold starts frequentes, o SDK pode precisar buscar as rules em toda nova instância.

**4. `Sampled=0` no header propagado → trace ignorado em todos os downstream**  
[FATO] Quando o serviço upstream decide não amostrar (`Sampled=0`), esse header é propagado para todos os downstream. Mesmo que uma sampling rule downstream tivesse alta taxa, o trace não é registrado porque a decisão é tomada uma vez e propagada. Isso é intencional — garante que o trace seja completo ou não exista.

**5. Lambda cold start: `@initDuration` aparece em segment separado**  
[FATO] O tempo de cold start do Lambda (`@initDuration`) é registrado em um segmento separado do segmento de invocação. No trace timeline, aparece como um nó paralelo. Confundir o cold start com latência de invocação pode levar a diagnósticos incorretos — separe `@duration` (invocação) de `@initDuration` (inicialização).

**6. DynamoDB não envia segmento — apenas inferred segments**  
[FATO] O DynamoDB não instrumenta X-Ray nativamente. O X-Ray cria um "inferred segment" a partir do subsegmento do AWS SDK no cliente (Lambda). Esse inferred segment mostra a latência **do ponto de vista do cliente** — inclui latência de rede. Não é possível distinguir se a latência ocorreu na rede ou dentro do DynamoDB através do X-Ray.

---

## Exercício de reflexão

Você tem uma aplicação com o seguinte fluxo: API Gateway → Lambda `checkout` → (SQS + DynamoDB). O Lambda `checkout` coloca uma mensagem no SQS e grava metadados no DynamoDB. Outro Lambda `order-processor` consome o SQS.

1. Como o X-Ray propaga o TraceId entre o Lambda `checkout` e o Lambda `order-processor`? O SQS preserva automaticamente o `X-Amzn-Trace-Id` header? O que acontece se o `order-processor` não estiver instrumentado?

2. Você quer criar um Group chamado `checkout-failures` que capture traces onde: o endpoint seja `/api/checkout`, haja falha (fault), e o usuário seja do tipo premium (`annotation[isPremiumUser] = true`). Escreva a filter expression correta.

3. O `checkout` Lambda tem 2.000 req/s. A regra padrão (1 req/s + 5%) amostrava ~101 traces/s. Você quer amostrar pelo menos 200 req/s do `/api/checkout` sem mais do que 10% adicional. Quais valores de `ReservoirSize` e `FixedRate` você usaria? Qual a taxa máxima de traces/s nessa configuração?

---

## Recursos para aprofundar

- [FATO] AWS X-Ray concepts (segments, subsegments, sampling, annotations): https://docs.aws.amazon.com/xray/latest/devguide/xray-concepts.html
- [FATO] Filter expressions — sintaxe completa: https://docs.aws.amazon.com/xray/latest/devguide/xray-console-filters.html
- [FATO] Configuring groups: https://docs.aws.amazon.com/xray/latest/devguide/xray-console-groups.html
- [FATO] Configuring sampling rules: https://docs.aws.amazon.com/xray/latest/devguide/xray-console-sampling.html
- [FATO] Powertools for AWS Lambda — Tracer: https://docs.aws.amazon.com/powertools/python/latest/core/tracer/

---

