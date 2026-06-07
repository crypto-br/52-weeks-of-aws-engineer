# Sessão 038 — CloudWatch: Composite Alarms, anomaly detection e automated actions

**Duração estimada:** 60 minutos  
**Dependências:** session-037-cloudwatch-logs-insights-queries

---

## Objetivo

Ao final desta sessão, você conseguirá criar um Composite Alarm que só dispara quando erros **e** alta latência ocorrem simultaneamente (reduzindo falsos positivos), habilitar anomaly detection em uma métrica e ajustar a banda de tolerância, configurar alarm actions que invocam Lambda ou SSM Automation (além de SNS), e entender os estados de alarm e a semântica de `MISSING_DATA`.

---

## Contexto

[FATO] Um alarm CloudWatch padrão monitora **uma única métrica** com um threshold fixo. Dois recursos avançados estendem isso: **Composite Alarms** combinam múltiplos alarms via expressões lógicas (AND, OR, NOT), e **Anomaly Detection** substitui o threshold fixo por uma banda dinâmica calculada por ML com base no histórico da métrica.

[CONSENSO] A principal motivação para Composite Alarms em produção é reduzir **falsos positivos**: um spike de latência às 3h da manhã sem aumento de erros raramente exige acordar alguém. Composite Alarms permitem modelar esse raciocínio: `ALARM(HighLatency) AND ALARM(HighErrorRate)` — só dispara se ambos forem verdadeiros simultaneamente.

---

## Conceitos principais

### 1. Estados de alarm e semântica de avaliação

[FATO] Todo alarm CloudWatch (simples ou composto) pode estar em um de três estados:

```
Estado              Significado
────────────────────────────────────────────────────────────────
OK                  Métrica está dentro do threshold / banda
ALARM               Métrica violou o threshold / banda
INSUFFICIENT_DATA   Dados insuficientes para avaliar
                    (comum em alarmes recém-criados ou métricas
                    com low-cardinality de dados)
```

[FATO] O parâmetro **Datapoints to alarm (M out of N)** controla a sensibilidade:

```
M de N — exemplo: 3 de 5
  ┌─────────────────────────────────────────────────────┐
  │ Evaluation periods (N): 5                           │
  │ Datapoints to alarm (M): 3                          │
  │                                                     │
  │ Alarme só dispara se 3 dos últimos 5 pontos         │
  │ violarem o threshold — reduz falsos positivos       │
  │ em métricas ruidosas                                │
  └─────────────────────────────────────────────────────┘

M = N (ex: 3 de 3): N períodos consecutivos violando → alarme dispara
M < N (ex: 2 de 5): menos sensível a outliers isolados
```

[FATO] **Missing data treatment** — comportamento quando pontos de dados estão ausentes:

```
Valor                   Comportamento
────────────────────────────────────────────────────────────────
notBreaching            Ausência = OK (dados faltando são normais)
breaching               Ausência = ALARM (ex: heartbeat — ausência é falha)
ignore                  Mantém estado atual do alarm
missing                 Alarm vai para INSUFFICIENT_DATA
```

---

### 2. Composite Alarms: rule expression

[FATO] Um Composite Alarm avalia uma **AlarmRule** — expressão booleana sobre os estados de outros alarms. Suporta `ALARM()`, `OK()`, `INSUFFICIENT_DATA()` como funções de estado, e `AND`, `OR`, `NOT`, parênteses como operadores lógicos.

```
Exemplos de AlarmRule:

-- Dispara apenas se AMBAS as condições estiverem em ALARM
ALARM("payment-high-error-rate") AND ALARM("payment-high-latency")

-- Dispara se qualquer uma estiver em ALARM
ALARM("payment-high-error-rate") OR ALARM("payment-high-latency")

-- Dispara para erros de backend, a menos que manutenção esteja programada
ALARM("payment-high-error-rate") AND NOT ALARM("maintenance-window-active")

-- Expressão complexa com parênteses
(ALARM("payment-high-error-rate") OR ALARM("payment-high-latency"))
AND NOT OK("health-check-endpoint")

-- Composite pode referenciar outros Composite Alarms
ALARM("payment-composite") AND ALARM("database-composite")
```

[FATO] Composite Alarms têm custo: $0.50/alarm/mês (us-east-1, além do custo dos alarms filhos).

[FATO] Composite Alarms **não avaliam métricas diretamente** — eles apenas agregam estados de outros alarms. A expressão `ALARM("alarm-name")` verifica se o alarm nomeado está no estado ALARM.

[FATO] **Ciclos de dependência**: dois Composite Alarms dependendo um do outro param de ser avaliados. Para quebrar o ciclo, mude o `AlarmRule` de um deles para `FALSE`.

---

### 3. Anomaly Detection: banda dinâmica por ML

[FATO] Anomaly Detection analisa o histórico de uma métrica e cria um modelo de valores esperados, considerando padrões horárias, diários e semanais. O resultado é uma **banda** (lower/upper bound) ao redor dos valores esperados.

```
Métrica: RequestLatency (ms)
                                        ┌─ threshold fixo (rígido)
  ─── ─── ─── ─── ─── ─── ─── ─── ─── ┤
                                        │
     ╭─────────────────────────────╮   ← banda superior (normal)
     │    valores esperados        │
  ───┤        (ML model)           ├─── ← valores esperados
     │                             │
     ╰─────────────────────────────╯   ← banda inferior (normal)
  
  Fim de semana: banda mais larga (tráfego mais variável)
  Horário comercial: banda mais estreita (padrão mais consistente)
```

[FATO] O parâmetro **Anomaly Detection Threshold** (número positivo, não precisa ser inteiro) controla a largura da banda:
- Threshold = 1: banda estreita, mais sensível a desvios
- Threshold = 2: banda moderada (padrão comum)
- Threshold = 3+: banda larga, mais tolerante

[FATO] O modelo leva **até 2 semanas** para ser totalmente treinado. Nos primeiros 3 dias, a banda é uma aproximação. Excluir períodos anormais do treinamento (deployments, incidents, feriados) melhora a precisão do modelo.

[FATO] Você pode configurar o alarm para disparar quando a métrica estiver:
- **Above the band** (útil para detecção de latência anômala alta)
- **Below the band** (útil para detecção de queda anômala de tráfego)
- **Outside the band** (qualquer desvio, para cima ou para baixo)

[FATO] Anomaly Detection acumula custo por métrica monitorada: $0.30/mês pela criação do detector (além do custo normal da métrica) — us-east-1.

---

### 4. Alarm Actions além de SNS

[FATO] Alarm actions disponíveis por tipo de recurso:

```
Action Type         Trigger States      Uso
────────────────────────────────────────────────────────────────
SNS notification    ALARM, OK,          Fan-out para email, SMS,
                    INSUFFICIENT_DATA   Lambda, SQS, HTTP endpoint

Lambda invocation   ALARM, OK,          Remediação automatizada,
                    INSUFFICIENT_DATA   enriquecimento de alertas

EC2 action          ALARM, OK           Stop, Start, Reboot,
                    (apenas EC2 alarms) Terminate da instância

SSM OpsItem         ALARM only          Cria item no OpsCenter
                                        para gerenciamento de incidentes

SSM Incident Manager ALARM only         Cria incidente no AWS
                                        Incident Manager

Auto Scaling        ALARM, OK           Scale-out / scale-in
(via ScalingPolicy)                     (configurado na policy,
                                        não diretamente no alarm)
```

[FATO] Para invocar Lambda diretamente de um alarm (sem SNS como intermediário): o alarm precisa de permissão para chamar `lambda:InvokeFunction`. A permissão é concedida automaticamente pelo console; via CDK/CLI, é necessário criar um `lambda.Permission` explicitamente.

---

## Exemplo prático

### Cenário: sistema de pagamentos — alerta composto + remediação automatizada

**Arquitetura de observabilidade:**
- Alarm 1: `HighErrorRate` — erros > 5% em 2 de 3 períodos de 1 min
- Alarm 2: `HighLatency` (Anomaly Detection) — latência fora da banda esperada
- Alarm 3: `MaintenanceWindow` — alarm manual para suprimir notificações durante manutenção
- Composite Alarm: dispara quando `HighErrorRate AND HighLatency AND NOT MaintenanceWindow`
- Action: Lambda que enriquece o alerta e cria OpsItem no SSM

#### CDK Python — Stack completa

```python
from aws_cdk import (
    Stack, Duration, RemovalPolicy,
    aws_cloudwatch as cw,
    aws_cloudwatch_actions as cwa,
    aws_lambda as lambda_,
    aws_sns as sns,
    aws_sns_subscriptions as sns_subs,
    aws_iam as iam,
)
from constructs import Construct


class PaymentAlarmsStack(Stack):
    def __init__(self, scope: Construct, construct_id: str, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        # ── Métricas base (emitidas via EMF da sessão 036) ────────────
        error_metric = cw.Metric(
            namespace="MyApp/Payments",
            metric_name="FailedPayments",
            dimensions_map={"service": "payment-processor", "environment": "production"},
            statistic="Sum",
            period=Duration.minutes(1),
        )
        total_metric = cw.Metric(
            namespace="MyApp/Payments",
            metric_name="SuccessfulPayments",
            dimensions_map={"service": "payment-processor", "environment": "production"},
            statistic="Sum",
            period=Duration.minutes(1),
        )
        latency_metric = cw.Metric(
            namespace="MyApp/Payments",
            metric_name="ProcessingLatency",
            dimensions_map={"service": "payment-processor", "environment": "production"},
            statistic="p95",
            period=Duration.minutes(1),
        )

        # ── Metric Math: taxa de erro (%) ─────────────────────────────
        error_rate_metric = cw.MathExpression(
            expression="100 * errors / (errors + successes)",
            using_metrics={
                "errors":    error_metric,
                "successes": total_metric,
            },
            period=Duration.minutes(1),
            label="Error Rate (%)",
        )

        # ── Alarm 1: Alta taxa de erros (threshold fixo) ──────────────
        high_error_alarm = cw.Alarm(
            self, "HighErrorRateAlarm",
            alarm_name="payment-high-error-rate",
            alarm_description=(
                "Payment error rate > 5% for 2 of 3 consecutive minutes.\n"
                "**Runbook:** https://wiki.internal/runbooks/payment-errors"
            ),
            metric=error_rate_metric,
            threshold=5.0,
            evaluation_periods=3,
            datapoints_to_alarm=2,          # 2 de 3: tolerante a outliers
            comparison_operator=cw.ComparisonOperator.GREATER_THAN_THRESHOLD,
            treat_missing_data=cw.TreatMissingData.NOT_BREACHING,
        )

        # ── Alarm 2: Latência anômala (Anomaly Detection) ─────────────
        # CfnAnomalyDetector configura o modelo ML
        anomaly_detector = cw.CfnAnomalyDetector(
            self, "LatencyAnomalyDetector",
            namespace="MyApp/Payments",
            metric_name="ProcessingLatency",
            stat="p95",
            dimensions=[
                cw.CfnAnomalyDetector.DimensionProperty(
                    name="service",  value="payment-processor"
                ),
                cw.CfnAnomalyDetector.DimensionProperty(
                    name="environment", value="production"
                ),
            ],
            # Exclui janelas de manutenção do treinamento do modelo
            # configuration=cw.CfnAnomalyDetector.ConfigurationProperty(
            #     excluded_time_ranges=[
            #         cw.CfnAnomalyDetector.RangeProperty(
            #             start_time="2024-01-01T02:00:00",
            #             end_time="2024-01-01T04:00:00",
            #         )
            #     ]
            # ),
        )

        # Alarm baseado na banda do anomaly detector
        high_latency_alarm = cw.CfnAlarm(
            self, "HighLatencyAnomalyAlarm",
            alarm_name="payment-high-latency-anomaly",
            alarm_description=(
                "Payment p95 latency is outside the expected band "
                "(anomaly detection threshold=2)."
            ),
            metrics=[
                cw.CfnAlarm.MetricDataQueryProperty(
                    id="m1",
                    metric_stat=cw.CfnAlarm.MetricStatProperty(
                        metric=cw.CfnAlarm.MetricProperty(
                            namespace="MyApp/Payments",
                            metric_name="ProcessingLatency",
                            dimensions=[
                                cw.CfnAlarm.DimensionProperty(
                                    name="service", value="payment-processor"
                                ),
                                cw.CfnAlarm.DimensionProperty(
                                    name="environment", value="production"
                                ),
                            ],
                        ),
                        stat="p95",
                        period=60,
                    ),
                    return_data=True,
                ),
                cw.CfnAlarm.MetricDataQueryProperty(
                    id="ad1",
                    expression="ANOMALY_DETECTION_BAND(m1, 2)",  # threshold=2
                    label="Anomaly Band",
                    return_data=True,
                ),
            ],
            comparison_operator="GreaterThanUpperThreshold",
            threshold_metric_id="ad1",
            evaluation_periods=3,
            datapoints_to_alarm=2,
            treat_missing_data="notBreaching",
        )

        # ── Alarm 3: Janela de manutenção (controle manual) ───────────
        # Este alarm é colocado em ALARM manualmente durante manutenção
        # para suprimir notificações do Composite Alarm
        maintenance_alarm = cw.Alarm(
            self, "MaintenanceWindowAlarm",
            alarm_name="payment-maintenance-window",
            alarm_description=(
                "Set this alarm to ALARM manually during planned maintenance "
                "to suppress composite alarm notifications."
            ),
            metric=cw.Metric(
                namespace="MyApp/Payments",
                metric_name="MaintenanceWindowActive",
                dimensions_map={"service": "payment-processor"},
                statistic="Sum",
                period=Duration.minutes(1),
            ),
            threshold=0,
            evaluation_periods=1,
            comparison_operator=cw.ComparisonOperator.GREATER_THAN_THRESHOLD,
            treat_missing_data=cw.TreatMissingData.NOT_BREACHING,
        )

        # ── Lambda de remediação ──────────────────────────────────────
        remediation_fn = lambda_.Function(
            self, "AlarmRemediationFn",
            function_name="payment-alarm-remediation",
            runtime=lambda_.Runtime.PYTHON_3_12,
            handler="handler.lambda_handler",
            code=lambda_.Code.from_asset("lambda/alarm_remediation"),
            timeout=Duration.seconds(30),
        )

        # Permissão para o CloudWatch invocar a Lambda
        remediation_fn.add_permission(
            "CloudWatchInvoke",
            principal=iam.ServicePrincipal("lambda.alarms.cloudwatch.amazonaws.com"),
            source_arn=f"arn:aws:cloudwatch:{self.region}:{self.account}:alarm:payment-critical-composite",
        )

        # ── SNS Topic para notificações ───────────────────────────────
        ops_topic = sns.Topic(
            self, "OpsTopic",
            topic_name="payment-ops-alerts",
        )
        ops_topic.add_subscription(
            sns_subs.EmailSubscription("ops-team@company.com")
        )

        # ── Composite Alarm ───────────────────────────────────────────
        composite_alarm = cw.CfnCompositeAlarm(
            self, "PaymentCriticalComposite",
            alarm_name="payment-critical-composite",
            alarm_description=(
                "Critical: payment errors AND latency anomaly detected simultaneously.\n"
                "Suppressed during maintenance window.\n"
                "**Runbook:** https://wiki.internal/runbooks/payment-critical"
            ),
            alarm_rule=(
                f'ALARM("{high_error_alarm.alarm_name}") '
                f'AND ALARM("{high_latency_alarm.alarm_name}") '
                f'AND NOT ALARM("{maintenance_alarm.alarm_name}")'
            ),
            alarm_actions=[
                ops_topic.topic_arn,
                remediation_fn.function_arn,
            ],
            ok_actions=[ops_topic.topic_arn],
            # actions_suppressor: outro alarm que suprime ações (feature avançada)
            # actions_suppressor="payment-maintenance-window",
        )
```

#### Lambda handler — remediação automatizada

```python
# lambda/alarm_remediation/handler.py
import json
import boto3
from typing import Any

ssm = boto3.client("ssm")
sns = boto3.client("sns")


def lambda_handler(event: dict, context: Any) -> None:
    """
    Invocado diretamente pelo CloudWatch Alarm quando muda de estado.
    O evento contém: AlarmName, NewStateValue, NewStateReason, StateChangeTime,
                     OldStateValue, Trigger (threshold/band info)
    """
    print(json.dumps(event))

    alarm_name  = event.get("alarmData", {}).get("alarmName", event.get("AlarmName", "unknown"))
    new_state   = event.get("alarmData", {}).get("state", {}).get("value", event.get("NewStateValue", "unknown"))
    reason      = event.get("alarmData", {}).get("state", {}).get("reason", event.get("NewStateReason", ""))
    change_time = event.get("alarmData", {}).get("state", {}).get("timestamp", event.get("StateChangeTime", ""))

    if new_state != "ALARM":
        # Ação de remediação só faz sentido ao entrar em ALARM
        print(f"Alarm {alarm_name} changed to {new_state} — no action needed")
        return

    print(f"ALARM triggered: {alarm_name} at {change_time}")
    print(f"Reason: {reason}")

    # 1. Criar OpsItem no SSM OpsCenter para rastreamento
    try:
        ssm.create_ops_item(
            Title=f"Payment Critical Alarm: {alarm_name}",
            Description=(
                f"CloudWatch Composite Alarm triggered at {change_time}.\n\n"
                f"Reason: {reason}\n\n"
                "Runbook: https://wiki.internal/runbooks/payment-critical"
            ),
            Source="cloudwatch",
            OperationalData={
                "/aws/resources": {
                    "Value": json.dumps([{
                        "arn": f"arn:aws:cloudwatch::alarm:{alarm_name}"
                    }]),
                    "Type": "SearchableString",
                }
            },
            Severity="2",   # 1=Critical, 2=High, 3=Medium, 4=Low
            Category="Availability",
            Tags=[
                {"Key": "Service",     "Value": "payment-processor"},
                {"Key": "AlarmName",   "Value": alarm_name},
                {"Key": "Environment", "Value": "production"},
            ],
        )
        print("SSM OpsItem created successfully")
    except Exception as e:
        print(f"Failed to create SSM OpsItem: {e}")

    # 2. Enriquecimento adicional: ex. verificar se há deploy recente em andamento
    # (chamada ao CodeDeploy, Deployment, etc. omitida por brevidade)
```

#### CLI — Criar e gerenciar alarms

```bash
# 1. Criar Composite Alarm via CLI
aws cloudwatch put-composite-alarm \
  --alarm-name "payment-critical-composite" \
  --alarm-description "Dispara quando erros E latência anômala ocorrem simultaneamente" \
  --alarm-rule 'ALARM("payment-high-error-rate") AND ALARM("payment-high-latency-anomaly") AND NOT ALARM("payment-maintenance-window")' \
  --alarm-actions \
      "arn:aws:sns:us-east-1:123456789012:payment-ops-alerts" \
      "arn:aws:lambda:us-east-1:123456789012:function:payment-alarm-remediation" \
  --ok-actions "arn:aws:sns:us-east-1:123456789012:payment-ops-alerts"

# 2. Criar anomaly detector para latência
aws cloudwatch put-anomaly-detector \
  --namespace "MyApp/Payments" \
  --metric-name "ProcessingLatency" \
  --stat "p95" \
  --dimensions Name=service,Value=payment-processor Name=environment,Value=production

# 3. Criar alarm baseado no anomaly detector
aws cloudwatch put-metric-alarm \
  --alarm-name "payment-high-latency-anomaly" \
  --metrics '[
    {
      "Id": "m1",
      "MetricStat": {
        "Metric": {
          "Namespace": "MyApp/Payments",
          "MetricName": "ProcessingLatency",
          "Dimensions": [
            {"Name": "service", "Value": "payment-processor"},
            {"Name": "environment", "Value": "production"}
          ]
        },
        "Stat": "p95",
        "Period": 60
      },
      "ReturnData": true
    },
    {
      "Id": "ad1",
      "Expression": "ANOMALY_DETECTION_BAND(m1, 2)",
      "Label": "ProcessingLatency (expected)",
      "ReturnData": true
    }
  ]' \
  --comparison-operator GreaterThanUpperThreshold \
  --threshold-metric-id ad1 \
  --evaluation-periods 3 \
  --datapoints-to-alarm 2 \
  --treat-missing-data notBreaching

# 4. Simular manutenção: forçar alarm para ALARM state manualmente
# (coloca maintenance alarm em ALARM para suprimir composite)
aws cloudwatch set-alarm-state \
  --alarm-name "payment-maintenance-window" \
  --state-value ALARM \
  --state-reason "Planned maintenance window 02:00-04:00 UTC"

# 5. Restaurar após manutenção
aws cloudwatch set-alarm-state \
  --alarm-name "payment-maintenance-window" \
  --state-value OK \
  --state-reason "Maintenance window completed"

# 6. Listar estados de todos os alarms de pagamento
aws cloudwatch describe-alarms \
  --alarm-name-prefix "payment-" \
  --query 'MetricAlarms[*].{Name:AlarmName,State:StateValue,Reason:StateReason}'

aws cloudwatch describe-alarms \
  --alarm-types CompositeAlarm \
  --alarm-name-prefix "payment-" \
  --query 'CompositeAlarms[*].{Name:AlarmName,State:StateValue,Rule:AlarmRule}'

# 7. Excluir período anômalo do modelo de anomaly detection
# (ex: excluir janela de incident de ontem do treinamento)
YESTERDAY_START=$(date -u -d 'yesterday 02:00' '+%Y-%m-%dT%H:%M:%S' 2>/dev/null || date -u -v-1d '+%Y-%m-%dT02:00:00')
YESTERDAY_END=$(date -u -d 'yesterday 06:00' '+%Y-%m-%dT%H:%M:%S' 2>/dev/null || date -u -v-1d '+%Y-%m-%dT06:00:00')

aws cloudwatch put-anomaly-detector \
  --namespace "MyApp/Payments" \
  --metric-name "ProcessingLatency" \
  --stat "p95" \
  --dimensions Name=service,Value=payment-processor Name=environment,Value=production \
  --configuration "{
    \"ExcludedTimeRanges\": [
      {\"StartTime\": \"${YESTERDAY_START}\", \"EndTime\": \"${YESTERDAY_END}\"}
    ]
  }"
```

---

## Armadilhas comuns

**1. Ciclos de dependência em Composite Alarms**  
[FATO] Dois Composite Alarms que referenciam um ao outro criam um ciclo que paralisa a avaliação de ambos. Não é possível deletar alarms em ciclo sem antes quebrar a dependência — defina `AlarmRule = FALSE` em um deles via `set-alarm-state` ou `put-composite-alarm` e então delete.

**2. Anomaly Detection leva até 2 semanas para precisão máxima**  
[FATO] Nas primeiras horas após criação do detector, a banda é uma estimativa grosseira. Criar alarms baseados em anomaly detection imediatamente após o deploy pode gerar alarmes falsos frequentes. Aguarde pelo menos 3 dias para a banda estabilizar; 2 semanas para precisão ideal.

**3. `set-alarm-state` é temporário — CloudWatch sobrescreve no próximo ciclo**  
[FATO] `set-alarm-state` força o estado do alarm mas esse estado é **sobrescrito na próxima avaliação de métrica** (no próximo período de evaluation). É útil para testes e para manutenções manuais temporárias, mas não persiste. Para supressão durante manutenção, use o padrão de Maintenance Window Alarm descrito acima.

**4. Lambda action requer permissão explícita — console cuida, CDK não**  
[FATO] O console CloudWatch adiciona automaticamente a permissão `lambda:InvokeFunction` para o principal `lambda.alarms.cloudwatch.amazonaws.com`. Via CDK ou CLI, você precisa criar um `lambda.Permission` explicitamente com `source_arn` apontando para o ARN do alarm. Sem isso, o alarm cria a action mas a invocação falha silenciosamente (sem erro visível no alarm — o erro aparece apenas nos CloudWatch Logs da Lambda).

**5. Actions Suppressor vs. NOT em AlarmRule**  
[CONSENSO] Há dois mecanismos para suprimir ações de Composite Alarms: (1) incluir `NOT ALARM("maintenance-alarm")` na AlarmRule — o alarm muda de estado mas não executa actions; (2) usar `ActionsSuppressor` no `put-composite-alarm` — o alarm muda de estado E suprime actions com um mecanismo de "wait" configurável. O segundo é mais robusto para janelas de manutenção longas pois permite configurar `ActionsSuppressorWaitPeriod` e `ActionsSuppressorExtensionPeriod`.

**6. Custo de anomaly detection não é trivial em alta escala**  
[FATO] Cada detector custa $0.30/mês (além do custo da métrica). 100 métricas com anomaly detection = $30/mês apenas nos detectores. Para workloads com centenas de microserviços e múltiplas métricas por serviço, o custo escala rapidamente. Priorize anomaly detection para métricas de SLO críticas.

---

## Exercício de reflexão

Você tem três microserviços — `checkout`, `inventory`, e `notification` — cada um com alarms individuais de `HighErrorRate` e `HighLatency`. Você quer criar uma arquitetura de observabilidade que:

1. Alerte de forma distinta quando apenas um serviço está degradado vs. quando múltiplos serviços estão degradados simultaneamente (o que indica um problema de infraestrutura compartilhada, não de serviço individual).

2. Modele a AlarmRule correta para um "CriticalSystemDegradation" Composite Alarm que dispara quando **dois ou mais** dos três serviços tiverem `HighErrorRate` em ALARM simultaneamente. Escreva a expressão lógica — dica: `A AND B OR A AND C OR B AND C` é equivalente a "2 ou mais de 3".

3. O serviço `notification` tem tráfego muito baixo às 3h-6h UTC (menos de 10 requests/minuto). Durante esse período, `HighErrorRate` frequentemente vai para `INSUFFICIENT_DATA`. Como você configuraria `treat_missing_data` para esse alarm, e que impacto isso teria no Composite Alarm? Há algum risco nessa escolha?

---

## Recursos para aprofundar

- [FATO] Create a Composite Alarm: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Create_Composite_Alarm.html
- [FATO] Create an alarm based on anomaly detection: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Create_Anomaly_Detection_Alarm.html
- [FATO] Using CloudWatch Anomaly Detection: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Anomaly_Detection.html
- [FATO] Alarm evaluation and missing data: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/alarms-and-missing-data.html
- [FATO] PutCompositeAlarm API (AlarmRule syntax): https://docs.aws.amazon.com/AmazonCloudWatch/latest/APIReference/API_PutCompositeAlarm.html

---

