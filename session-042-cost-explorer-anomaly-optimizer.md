# Sessão 042 — Cost Explorer, Cost Anomaly Detection e Compute Optimizer

**Duração estimada:** 60 min  
**Pré-requisito:** session-041 (Spot Instances e Fleet)

---

## Objetivos da sessão

- Dominar a API `GetCostAndUsage` com filtros por tag, dimensão e métricas corretas
- Entender as diferenças entre BlendedCost, UnblendedCost, AmortizedCost e NetAmortizedCost
- Configurar Cost Anomaly Detection com monitores gerenciados e por tag
- Interpretar o payload SNS de anomalia e seus campos (`anomalyScore`, `impact`, `rootCauses`)
- Usar Compute Optimizer para EC2 e Fargate com preferências de rightsizing
- Exportar recomendações do Compute Optimizer para análise em S3

---

## 1. Cost Explorer — Conceitos Fundamentais

### 1.1 Granularidade e janela de dados

[FATO] Cost Explorer disponibiliza dados com atraso de **até 24 horas**. A granularidade define o período mínimo de cada ponto:

```
╔════════════════╦════════════════════════════════════════════════╗
║ Granularidade  ║ Restrição                                      ║
╠════════════════╬════════════════════════════════════════════════╣
║ HOURLY         ║ Janela máxima: 14 dias atrás                   ║
║ DAILY          ║ Padrão; 13 meses gratuito, 38 meses (pago)     ║
║ MONTHLY        ║ Padrão; 13 meses gratuito, 38 meses (pago)     ║
╚════════════════╩════════════════════════════════════════════════╝
```

### 1.2 Métricas — qual usar para quê

[FATO] Cada métrica representa uma perspectiva diferente do custo:

```
UnblendedCost
  — Custo real cobrado na conta individual
  — Inclui o preço On-Demand sem qualquer spreading de RI/SP
  — Usar para: faturamento de conta individual, chargeback simples

BlendedCost
  — Distribui o custo de RIs e SPs proporcionalmente entre as contas vinculadas
  — Usado para: alocação de custo em Organizations (levela diferenças de preço)
  — Cuidado: pode não refletir o custo real de cada conta filha

AmortizedCost
  — Distribui o custo upfront de RI/SP pelo período de vigência
  — Ex.: RI 1 ano, $1000 upfront → $2,74/dia amortizado
  — Usar para: análise de custo real do período (ex.: dashboards FinOps)

NetAmortizedCost
  — AmortizedCost depois de descontos privados (Enterprise Discount Program, negociações)
  — Usar para: custo líquido real em contas com contratos customizados

NormalizedUsageAmount / UsageQuantity
  — Volume de uso, não custo; útil para análise de consumo por serviço
```

[CONSENSO] Para dashboards de custo por projeto/time, a métrica mais útil é **AmortizedCost** — reflete o custo econômico real, espalhando os upfront de RI/SP.

### 1.3 Cost Allocation Tags

[FATO] Para filtrar ou agrupar por tag no Cost Explorer, a tag precisa ser **ativada** como Cost Allocation Tag no console do Billing. Não é automático.

```
Fluxo de ativação:
  1. Billing console → Cost allocation tags
  2. Selecionar tag (ex.: "project") → Activate
  3. Aguardar 24h para aparecer nos dados do Cost Explorer
  4. Tags em recursos novos começam a aparecer imediatamente após ativação
  5. Tags em recursos históricos NÃO são retroativas

Limite: 500 Cost Allocation Tags ativadas por conta
```

---

## 2. API GetCostAndUsage — Estrutura e Exemplos

### 2.1 Estrutura de Expression (filtros)

[FATO] O tipo `Expression` na API do Cost Explorer suporta composição booleana com `And`, `Or`, `Not` (arrays), e folhas com `Dimensions`, `Tags`, `CostCategories`.

```python
# Estrutura base de Expression
expression = {
    "And": [
        # Folha 1: filtrar por tag
        {
            "Tags": {
                "Key": "project",
                "Values": ["checkout-service", "payments-api"],
                "MatchOptions": ["EQUALS"]
            }
        },
        # Folha 2: excluir tipo de uso (ex.: data transfer)
        {
            "Not": {
                "Dimensions": {
                    "Key": "USAGE_TYPE_GROUP",
                    "Values": ["EC2: Data Transfer - Internet (Out)"],
                    "MatchOptions": ["EQUALS"]
                }
            }
        }
    ]
}
```

**Dimensões disponíveis em `Dimensions.Key`:** `SERVICE`, `REGION`, `LINKED_ACCOUNT`, `INSTANCE_TYPE`, `USAGE_TYPE`, `USAGE_TYPE_GROUP`, `RECORD_TYPE` (On-Demand/Spot/SavingsPlan/etc.), `OPERATING_SYSTEM`, `TENANCY`, `PURCHASE_TYPE`, `AZ`.

**MatchOptions:** `EQUALS`, `STARTS_WITH`, `ENDS_WITH`, `CONTAINS`, `ABSENT` (recursos sem a tag), `CASE_SENSITIVE`, `CASE_INSENSITIVE`.

### 2.2 GroupBy

[FATO] `GroupBy` define como o resultado é segmentado. Máximo de 2 grupos por chamada. O tipo pode ser `DIMENSION` ou `TAG`.

```python
group_by = [
    {"Type": "TAG",       "Key": "project"},         # tag ativada
    {"Type": "DIMENSION", "Key": "SERVICE"},           # dimensão padrão
]
```

---

## 3. CDK Python — Stack de Monitoramento de Custo

```python
from aws_cdk import (
    Stack, aws_ce as ce, aws_sns as sns,
    aws_sns_subscriptions as subs,
    aws_budgets as budgets,
)
from constructs import Construct

class CostMonitoringStack(Stack):
    def __init__(self, scope: Construct, construct_id: str, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        # SNS topic para alertas de anomalia
        alert_topic = sns.Topic(self, "CostAlertTopic",
            display_name="AWS Cost Anomaly Alerts",
        )
        alert_topic.add_subscription(
            subs.EmailSubscription("finops-team@company.com")
        )

        # ──────────────────────────────────────────────────────────────
        # Cost Anomaly Detection — Monitor por serviço (AWS Managed)
        # Monitora todos os serviços automaticamente, top 5000 valores
        # ──────────────────────────────────────────────────────────────
        service_monitor = ce.CfnAnomalyMonitor(self, "ServiceMonitor",
            monitor_name="AllServicesMonitor",
            monitor_type="DIMENSIONAL",
            monitor_dimension="SERVICE",  # AWS Managed: monitora todos serviços
        )

        # Subscription com dois thresholds combinados (AND):
        # alerta se impacto >= $50 E percentual >= 20%
        ce.CfnAnomalySubscription(self, "ServiceSubscription",
            subscription_name="ServiceAnomalyAlerts",
            monitor_arn_list=[service_monitor.attr_monitor_arn],
            subscribers=[
                ce.CfnAnomalySubscription.SubscriberProperty(
                    address=alert_topic.topic_arn,
                    type="SNS",
                    status="CONFIRMED",
                )
            ],
            frequency="IMMEDIATE",  # alertas individuais via SNS (não diário/semanal)
            threshold_expression={
                "And": [
                    {
                        "Dimensions": {
                            "Key": "ANOMALY_TOTAL_IMPACT_ABSOLUTE",
                            "MatchOptions": ["GREATER_THAN_OR_EQUAL"],
                            "Values": ["50"]   # $50 de impacto absoluto
                        }
                    },
                    {
                        "Dimensions": {
                            "Key": "ANOMALY_TOTAL_IMPACT_PERCENTAGE",
                            "MatchOptions": ["GREATER_THAN_OR_EQUAL"],
                            "Values": ["20"]   # 20% de desvio percentual
                        }
                    }
                ]
            },
        )

        # ──────────────────────────────────────────────────────────────
        # Monitor por Cost Allocation Tag (project) — Customer Managed
        # Monitora projetos específicos com threshold personalizado
        # ──────────────────────────────────────────────────────────────
        tag_monitor = ce.CfnAnomalyMonitor(self, "ProjectTagMonitor",
            monitor_name="ProjectTagMonitor",
            monitor_type="CUSTOM",  # Customer managed
            monitor_specification=ce.CfnAnomalyMonitor.MonitorSpecificationProperty(
                expression={
                    "Tags": {
                        "Key": "project",
                        "Values": ["checkout-service", "payments-api", "fraud-detector"],
                        "MatchOptions": ["EQUALS"]
                    }
                }
            ),
        )

        ce.CfnAnomalySubscription(self, "ProjectSubscription",
            subscription_name="ProjectAnomalyAlerts",
            monitor_arn_list=[tag_monitor.attr_monitor_arn],
            subscribers=[
                ce.CfnAnomalySubscription.SubscriberProperty(
                    address=alert_topic.topic_arn,
                    type="SNS",
                    status="CONFIRMED",
                )
            ],
            frequency="IMMEDIATE",
            threshold_expression={
                "Dimensions": {
                    "Key": "ANOMALY_TOTAL_IMPACT_ABSOLUTE",
                    "MatchOptions": ["GREATER_THAN_OR_EQUAL"],
                    "Values": ["100"]  # threshold mais alto para projetos core
                }
            },
        )

        # ──────────────────────────────────────────────────────────────
        # Budget por tag de projeto — alerta na previsão de estourar
        # ──────────────────────────────────────────────────────────────
        budgets.CfnBudget(self, "ProjectBudget",
            budget=budgets.CfnBudget.BudgetDataProperty(
                budget_name="checkout-service-monthly",
                budget_type="COST",
                time_unit="MONTHLY",
                budget_limit=budgets.CfnBudget.SpendProperty(
                    amount=5000, unit="USD"
                ),
                cost_filters={
                    "TagKeyValue": ["user:project$checkout-service"]
                },
            ),
            notifications_with_subscribers=[
                budgets.CfnBudget.NotificationWithSubscribersProperty(
                    notification=budgets.CfnBudget.NotificationProperty(
                        comparison_operator="GREATER_THAN",
                        notification_type="ACTUAL",
                        threshold=80,
                        threshold_type="PERCENTAGE",
                    ),
                    subscribers=[
                        budgets.CfnBudget.SubscriberProperty(
                            address=alert_topic.topic_arn,
                            subscription_type="SNS",
                        )
                    ],
                ),
                budgets.CfnBudget.NotificationWithSubscribersProperty(
                    notification=budgets.CfnBudget.NotificationProperty(
                        comparison_operator="GREATER_THAN",
                        notification_type="FORECASTED",
                        threshold=100,
                        threshold_type="PERCENTAGE",
                    ),
                    subscribers=[
                        budgets.CfnBudget.SubscriberProperty(
                            address=alert_topic.topic_arn,
                            subscription_type="SNS",
                        )
                    ],
                ),
            ],
        )
```

---

## 4. Python — Cost Explorer GetCostAndUsage

```python
import boto3
from datetime import datetime, date, timedelta
from dataclasses import dataclass, field
from typing import Optional
import calendar

ce = boto3.client("ce", region_name="us-east-1")


@dataclass
class ProjectCostReport:
    project: str
    service: str
    monthly_cost: float
    previous_month_cost: float
    mom_change_pct: float  # month-over-month %


def get_cost_by_project_and_service(
    tag_key: str = "project",
    tag_values: Optional[list[str]] = None,
    lookback_months: int = 2,
) -> list[ProjectCostReport]:
    """
    Retorna custo mensal segmentado por tag:project × SERVICE,
    usando AmortizedCost (correto para FinOps — amortiza RI/SP upfront).
    """
    today = date.today()
    # Primeiro dia do mês atual
    start_current = today.replace(day=1)
    # Primeiro dia de lookback_months atrás
    start_prev = (start_current - timedelta(days=lookback_months * 31)).replace(day=1)
    # Fim = amanhã (para incluir hoje)
    end = today + timedelta(days=1)

    filter_expr: dict = {}
    if tag_values:
        filter_expr = {
            "Tags": {
                "Key": tag_key,
                "Values": tag_values,
                "MatchOptions": ["EQUALS"]
            }
        }
    # Sem filtro de tag = todos os projetos

    params = dict(
        TimePeriod={
            "Start": start_prev.strftime("%Y-%m-%d"),
            "End": end.strftime("%Y-%m-%d"),
        },
        Granularity="MONTHLY",
        Metrics=["AmortizedCost"],
        GroupBy=[
            {"Type": "TAG",       "Key": tag_key},
            {"Type": "DIMENSION", "Key": "SERVICE"},
        ],
    )
    if filter_expr:
        params["Filter"] = filter_expr

    # Paginar resultados
    all_results = []
    next_token = None
    while True:
        if next_token:
            params["NextPageToken"] = next_token
        response = ce.get_cost_and_usage(**params)
        all_results.extend(response.get("ResultsByTime", []))
        next_token = response.get("NextPageToken")
        if not next_token:
            break

    # Organizar por (project, service) → {month_key: cost}
    from collections import defaultdict
    cost_map: dict[tuple, dict[str, float]] = defaultdict(dict)
    for result in all_results:
        period_start = result["TimePeriod"]["Start"][:7]  # "2025-11"
        for group in result.get("Groups", []):
            keys = group["Keys"]  # ["project$checkout", "Amazon EC2"]
            project_val = keys[0].replace(f"{tag_key}$", "")
            service_val = keys[1]
            cost = float(group["Metrics"]["AmortizedCost"]["Amount"])
            cost_map[(project_val, service_val)][period_start] = cost

    # Montar relatório com variação MoM
    months = sorted({m for costs in cost_map.values() for m in costs})
    if len(months) < 2:
        return []
    prev_month, curr_month = months[-2], months[-1]

    reports = []
    for (project, service), monthly_data in cost_map.items():
        curr = monthly_data.get(curr_month, 0.0)
        prev = monthly_data.get(prev_month, 0.0)
        mom = ((curr - prev) / prev * 100) if prev > 0 else 0.0
        if curr > 0.01 or prev > 0.01:  # filtrar zeros
            reports.append(ProjectCostReport(
                project=project,
                service=service,
                monthly_cost=curr,
                previous_month_cost=prev,
                mom_change_pct=mom,
            ))

    return sorted(reports, key=lambda r: r.monthly_cost, reverse=True)


def get_cost_forecast(
    tag_key: str = "project",
    tag_value: str = "checkout-service",
    months_ahead: int = 1,
) -> dict:
    """Previsão de custo para o próximo mês usando GetCostForecast."""
    today = date.today()
    # Forecast começa amanhã
    start = (today + timedelta(days=1)).strftime("%Y-%m-%d")
    # Último dia do mês seguinte
    target_month = today.month + months_ahead
    target_year = today.year + (target_month - 1) // 12
    target_month = ((target_month - 1) % 12) + 1
    last_day = calendar.monthrange(target_year, target_month)[1]
    end = f"{target_year}-{target_month:02d}-{last_day:02d}"

    response = ce.get_cost_forecast(
        TimePeriod={"Start": start, "End": end},
        Metric="AMORTIZED_COST",
        Granularity="MONTHLY",
        Filter={
            "Tags": {
                "Key": tag_key,
                "Values": [tag_value],
                "MatchOptions": ["EQUALS"]
            }
        },
    )
    total = response["Total"]
    return {
        "forecast_amount": float(total["Amount"]),
        "unit": total["Unit"],
        "prediction_interval_lower": float(
            response["ForecastResultsByTime"][0]["PredictionIntervalLowerBound"]
        ),
        "prediction_interval_upper": float(
            response["ForecastResultsByTime"][0]["PredictionIntervalUpperBound"]
        ),
    }


# Exemplo de uso
if __name__ == "__main__":
    reports = get_cost_by_project_and_service(
        tag_key="project",
        tag_values=["checkout-service", "payments-api"],
    )
    print(f"\n{'Projeto':25} {'Serviço':35} {'Mês atual':12} {'Mês ant.':12} {'MoM':8}")
    print("-" * 95)
    for r in reports[:20]:
        print(
            f"{r.project:25} {r.service:35} "
            f"${r.monthly_cost:>9.2f}  ${r.previous_month_cost:>9.2f}  "
            f"{r.mom_change_pct:>+6.1f}%"
        )

    fc = get_cost_forecast("project", "checkout-service")
    print(f"\nPrevisão checkout-service: ${fc['forecast_amount']:.2f} "
          f"(range: ${fc['prediction_interval_lower']:.2f}–${fc['prediction_interval_upper']:.2f})")
```

---

## 5. Compute Optimizer — Rightsizing

### 5.1 Recursos suportados e pré-requisitos

[FATO] Compute Optimizer suporta: EC2 instances, EC2 Auto Scaling groups, EBS volumes, Lambda functions, ECS services on Fargate, Aurora/RDS databases, commercial software licenses.

[FATO] Compute Optimizer **não está ativo por padrão** — é necessário opt-in explícito na conta ou na conta de gerenciamento da Organization.

[FATO] Por padrão, analisa **14 dias** de métricas CloudWatch. Com Enhanced Infrastructure Metrics (pago), estende para **93 dias**.

[FATO] Para recomendações que consideram memória em EC2, é necessário instalar o **CloudWatch agent** na instância (ou configurar ingestão de métricas externas via Datadog/Dynatrace).

### 5.2 Findings (resultados possíveis)

```
╔══════════════════════╦══════════════════════════════════════════════════════╗
║ Finding              ║ Significado                                          ║
╠══════════════════════╬══════════════════════════════════════════════════════╣
║ OVER_PROVISIONED     ║ Instância maior que necessário; há economia possível ║
║ UNDER_PROVISIONED    ║ Instância menor que necessário; risco de performance ║
║ OPTIMIZED            ║ Configuração adequada ao uso atual                   ║
║ NOT_OPTIMIZED        ║ Dados insuficientes ou configuração especial         ║
╚══════════════════════╩══════════════════════════════════════════════════════╝
```

### 5.3 Preferências de Rightsizing — Presets

[FATO] Compute Optimizer oferece 4 presets configuráveis, com impacto direto no conservadorismo das recomendações:

```
╔══════════════════════╦══════════════╦══════════════╦════════════════╗
║ Preset               ║ CPU Threshold║ CPU Headroom ║ Memory Headroom║
╠══════════════════════╬══════════════╬══════════════╬════════════════╣
║ Maximum savings      ║ P90          ║ 0%           ║ 10%            ║
║ Balanced             ║ P95          ║ 30%          ║ 30%            ║
║ Default              ║ P99.5        ║ 20%          ║ 20%            ║
║ Maximum performance  ║ P99.5        ║ 30%          ║ 30%            ║
╚══════════════════════╩══════════════╩══════════════╩════════════════╝

CPU Threshold = percentil acima do qual dados são ignorados (ex.: P90 ignora top 10% picos)
CPU Headroom  = margem adicionada acima do uso atual para buffer
Memory Headroom = margem adicionada acima do uso de memória
```

[CONSENSO] Para produção conservadora, `Default` (P99.5 + 20% headroom) é adequado. Para ambientes de dev/staging onde picos não são críticos, `Balanced` ou `Maximum savings` geram mais economia.

---

## 6. Python — Compute Optimizer: análise e exportação de recomendações

```python
import boto3
import json
from dataclasses import dataclass
from typing import Optional

co = boto3.client("compute-optimizer", region_name="us-east-1")
ec2_client = boto3.client("ec2", region_name="us-east-1")


@dataclass
class EC2Recommendation:
    instance_id: str
    instance_type: str
    finding: str  # OVER_PROVISIONED / UNDER_PROVISIONED / OPTIMIZED
    recommended_instance_type: Optional[str]
    estimated_monthly_savings: float
    cpu_max_30d: float
    memory_max_30d: Optional[float]
    reason: str


def get_ec2_rightsizing_recommendations(
    account_ids: Optional[list[str]] = None,
    finding_filter: Optional[list[str]] = None,  # ex.: ["OVER_PROVISIONED"]
) -> list[EC2Recommendation]:
    """
    Busca recomendações de rightsizing para instâncias EC2.
    finding_filter: lista de findings para filtrar (None = todos).
    """
    params: dict = {}
    if account_ids:
        params["accountIds"] = account_ids
    if finding_filter:
        params["filters"] = [
            {"name": "Finding", "values": finding_filter}
        ]

    recommendations = []
    next_token = None
    while True:
        if next_token:
            params["nextToken"] = next_token
        response = co.get_ec2_instance_recommendations(**params)

        for rec in response.get("instanceRecommendations", []):
            # Pegar a top recomendação (primeira da lista)
            top_options = rec.get("recommendationOptions", [])
            if not top_options:
                continue
            top = top_options[0]

            # Extrair métricas de utilização
            cpu_max = next(
                (float(m["value"]) for m in rec.get("utilizationMetrics", [])
                 if m["name"] == "CPU" and m["statistic"] == "MAXIMUM"),
                0.0
            )
            mem_max = next(
                (float(m["value"]) for m in rec.get("utilizationMetrics", [])
                 if m["name"] == "MEMORY" and m["statistic"] == "MAXIMUM"),
                None
            )

            # Estimativa de economia mensal
            savings = top.get("estimatedMonthlySavings", {})
            monthly_savings = float(savings.get("value", 0.0))

            # Motivos da recomendação
            reasons = [r.get("name", "") for r in top.get("migrationEffort", [])]
            reason_str = ", ".join(reasons) if reasons else "CPU over-provisioned"

            recommendations.append(EC2Recommendation(
                instance_id=rec["instanceArn"].split("/")[-1],
                instance_type=rec["currentInstanceType"],
                finding=rec["finding"],
                recommended_instance_type=top.get("instanceType"),
                estimated_monthly_savings=monthly_savings,
                cpu_max_30d=cpu_max,
                memory_max_30d=mem_max,
                reason=reason_str,
            ))

        next_token = response.get("nextToken")
        if not next_token:
            break

    return sorted(recommendations, key=lambda r: r.estimated_monthly_savings, reverse=True)


def export_ec2_recommendations_to_s3(
    s3_bucket: str,
    s3_prefix: str = "compute-optimizer/",
    file_format: str = "Csv",  # "Csv" ou "Json"
    include_member_accounts: bool = True,
) -> str:
    """
    Exporta recomendações EC2 para S3 (processamento assíncrono).
    Retorna o JobId para acompanhamento.
    """
    response = co.export_ec2_instance_recommendations(
        s3DestinationConfig={
            "bucket": s3_bucket,
            "keyPrefix": s3_prefix,
        },
        fileFormat=file_format,
        includeMemberAccounts=include_member_accounts,
        # Campos opcionais para filtrar
        # accountIds=["123456789012"],
        # filters=[{"name": "Finding", "values": ["OVER_PROVISIONED"]}],
    )
    return response["jobId"]


def get_ecs_fargate_recommendations() -> list[dict]:
    """
    Recomendações para ECS services on Fargate:
    Compute Optimizer recomenda task CPU, task memory, container CPU/memory.
    """
    response = co.get_ecs_service_recommendations(
        filters=[
            {"name": "Finding", "values": ["OVER_PROVISIONED", "UNDER_PROVISIONED"]}
        ]
    )

    results = []
    for rec in response.get("ecsServiceRecommendations", []):
        current_config = rec.get("currentServiceConfiguration", {})
        top_option = (rec.get("recommendationOptions") or [{}])[0]
        recommended_config = top_option.get("containerRecommendations", [])
        savings = top_option.get("estimatedMonthlySavings", {})

        results.append({
            "service_arn": rec["serviceArn"],
            "finding": rec["finding"],
            "current_cpu": current_config.get("cpu"),
            "current_memory": current_config.get("memory"),
            "recommended_containers": [
                {
                    "name": c.get("containerName"),
                    "recommended_cpu": c.get("memorySizeConfiguration", {}).get("cpu"),
                    "recommended_memory": c.get("memorySizeConfiguration", {}).get("memory"),
                }
                for c in recommended_config
            ],
            "estimated_monthly_savings_usd": float(savings.get("value", 0.0)),
        })

    return sorted(results, key=lambda r: r["estimated_monthly_savings_usd"], reverse=True)


# Exemplo de uso
if __name__ == "__main__":
    # EC2 — lista top recomendações de over-provisioned
    recs = get_ec2_rightsizing_recommendations(
        finding_filter=["OVER_PROVISIONED"]
    )
    total_savings = sum(r.estimated_monthly_savings for r in recs)
    print(f"\nOportunidade total de rightsizing EC2: ${total_savings:.2f}/mês")
    print(f"\n{'ID':20} {'Atual':15} {'Recomendado':15} {'Economia/mês':13} {'CPU max%':9}")
    print("-" * 75)
    for r in recs[:10]:
        print(
            f"{r.instance_id:20} {r.instance_type:15} "
            f"{str(r.recommended_instance_type):15} "
            f"${r.estimated_monthly_savings:>10.2f}   {r.cpu_max_30d:>6.1f}%"
        )

    # Exportar para S3 (assíncrono)
    job_id = export_ec2_recommendations_to_s3(
        s3_bucket="my-finops-bucket",
        s3_prefix="compute-optimizer/ec2/",
    )
    print(f"\nExportação iniciada. Job ID: {job_id}")

    # ECS Fargate
    ecs_recs = get_ecs_fargate_recommendations()
    if ecs_recs:
        print(f"\nRecomendações ECS Fargate: {len(ecs_recs)} serviços")
        for r in ecs_recs[:5]:
            print(f"  {r['service_arn'].split('/')[-1]}: "
                  f"{r['finding']} | "
                  f"Economia: ${r['estimated_monthly_savings_usd']:.2f}/mês")
```

---

## 7. CLI — Exemplos Essenciais

```bash
# ── COST EXPLORER ──────────────────────────────────────────────────────────

# 1. Custo mensal por serviço nos últimos 3 meses
aws ce get-cost-and-usage \
  --time-period Start=$(date -d '3 months ago' +%Y-%m-01),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY \
  --metrics AmortizedCost \
  --group-by Type=DIMENSION,Key=SERVICE \
  --query 'ResultsByTime[].{Month:TimePeriod.Start,Groups:Groups[*].{Service:Keys[0],Cost:Metrics.AmortizedCost.Amount}}' \
  --output json

# 2. Custo por tag de projeto no mês atual
aws ce get-cost-and-usage \
  --time-period Start=$(date +%Y-%m-01),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY \
  --metrics AmortizedCost \
  --group-by Type=TAG,Key=project \
  --filter '{
    "Not": {
      "Dimensions": {
        "Key": "RECORD_TYPE",
        "Values": ["Credit", "Refund"],
        "MatchOptions": ["EQUALS"]
      }
    }
  }' \
  --query 'ResultsByTime[0].Groups | sort_by(@, &Metrics.AmortizedCost.Amount) | reverse(@) | [:10].[Keys[0], Metrics.AmortizedCost.Amount]' \
  --output table

# 3. Previsão de custo para o próximo mês
aws ce get-cost-forecast \
  --time-period Start=$(date -d 'tomorrow' +%Y-%m-%d),End=$(date -d 'last day of next month' +%Y-%m-%d) \
  --metric AMORTIZED_COST \
  --granularity MONTHLY \
  --filter '{"Tags": {"Key": "project","Values": ["checkout-service"],"MatchOptions": ["EQUALS"]}}' \
  --query '{Forecast:Total.Amount,Unit:Total.Unit}'

# 4. Listar Cost Allocation Tags ativadas
aws ce list-cost-allocation-tags \
  --status Active \
  --query 'CostAllocationTags[*].{Key:TagKey,Type:Type}'


# ── COST ANOMALY DETECTION ──────────────────────────────────────────────────

# 5. Listar monitores de anomalia
aws ce get-anomaly-monitors \
  --query 'AnomalyMonitors[*].{Name:MonitorName,Type:MonitorType,Arn:MonitorArn}'

# 6. Listar anomalias detectadas nos últimos 30 dias
aws ce get-anomalies \
  --date-interval Start=$(date -d '30 days ago' +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --query 'Anomalies[*].{
      ID:AnomalyId,
      Start:AnomalyStartDate,
      Impact:Impact.TotalImpact,
      ImpactPct:Impact.TotalImpactPercentage,
      Service:RootCauses[0].Service
  }' \
  --output table

# 7. Ver detalhes de uma anomalia específica
aws ce get-anomalies \
  --anomaly-id "12345678-abcd-ef12-3456-987654321a12" \
  --date-interval Start=2024-01-01,End=2024-12-31 \
  --query 'Anomalies[0]'


# ── COMPUTE OPTIMIZER ──────────────────────────────────────────────────────

# 8. Opt-in (necessário antes de qualquer uso)
aws compute-optimizer update-enrollment-status \
  --status Active \
  --include-member-accounts   # para Organizations

# 9. Verificar status de enrollment
aws compute-optimizer get-enrollment-status \
  --query '{Status:Status,NumberOfMemberAccountsOptedIn:NumberOfMemberAccountsOptedIn}'

# 10. Recomendações EC2 — instâncias over-provisioned com maior economia
aws compute-optimizer get-ec2-instance-recommendations \
  --filters name=Finding,values=OVER_PROVISIONED \
  --query 'instanceRecommendations | sort_by(@, &recommendationOptions[0].estimatedMonthlySavings.value) | reverse(@) | [:10] | [*].{
    ID:instanceArn,
    Current:currentInstanceType,
    Recommended:recommendationOptions[0].instanceType,
    Savings:recommendationOptions[0].estimatedMonthlySavings.value,
    SavingsUnit:recommendationOptions[0].estimatedMonthlySavings.currency
  }' \
  --output table

# 11. Recomendações ECS Fargate
aws compute-optimizer get-ecs-service-recommendations \
  --filters name=Finding,values=OVER_PROVISIONED \
  --query 'ecsServiceRecommendations[*].{
    Service:serviceArn,
    Finding:finding,
    CurrentCPU:currentServiceConfiguration.cpu,
    CurrentMem:currentServiceConfiguration.memory,
    Savings:recommendationOptions[0].estimatedMonthlySavings.value
  }' \
  --output table

# 12. Exportar todas as recomendações EC2 para S3 (assíncrono)
aws compute-optimizer export-ec2-instance-recommendations \
  --s3-destination-config bucket=my-finops-bucket,keyPrefix=compute-optimizer/ \
  --file-format Csv \
  --include-member-accounts \
  --query 'jobId'

# 13. Verificar status da exportação
aws compute-optimizer describe-recommendation-export-jobs \
  --job-ids "job-0abc123def456789" \
  --query 'recommendationExportJobs[0].{Status:status,S3:s3Destination}'

# 14. Preferências de rightsizing — aplicar preset "Balanced" para conta
aws compute-optimizer put-recommendation-preferences \
  --resource-type Ec2Instance \
  --scope name=AccountId,value=123456789012 \
  --cpu-vendor-architectures CURRENT_OR_FUTURE_GENERATION \
  --utilization-preferences '[
    {"metricName": "CpuUtilization", "metricParameters": {"threshold": "P95", "headroom": "PERCENT_30"}},
    {"metricName": "MemoryUtilization", "metricParameters": {"headroom": "PERCENT_30"}}
  ]'
```

---

## 8. Payload SNS — Interpretando Anomalias

[FATO] O payload enviado ao SNS quando uma anomalia é detectada tem a seguinte estrutura:

```json
{
  "accountId": "123456789012",
  "anomalyDetailsLink": "https://console.aws.amazon.com/...",
  "anomalyEndDate": null,
  "anomalyId": "12345678-abcd-ef12-3456-987654321a12",
  "anomalyScore": {
    "currentScore": 0.87,   // quão anômalo é agora (0–1, não documentado como %
                             // mas quanto maior, mais anômalo)
    "maxScore":    0.87
  },
  "anomalyStartDate": "2024-03-15T00:00:00Z",
  "dimensionKey": {"type": "DIMENSION", "key": "SERVICE"},
  "dimensionalValue": "Amazon EC2",
  "impact": {
    "maxImpact":            1203.45,   // maior impacto diário
    "totalActualSpend":     5412.78,   // gasto real no período da anomalia
    "totalExpectedSpend":   1800.00,   // gasto esperado pelo modelo ML
    "totalImpact":          3612.78,   // actual - expected
    "totalImpactPercentage": 200.71    // (totalImpact / totalExpectedSpend) * 100
  },
  "rootCauses": [
    {
      "linkedAccount": "987654321098",
      "linkedAccountName": "prod-account",
      "region": "us-east-1",
      "service": "Amazon EC2",
      "usageType": "BoxUsage:c5.4xlarge",
      "impact": {"contribution": 2800.00}
    }
  ],
  "subscriptionId": "...",
  "subscriptionName": "ServiceAnomalyAlerts"
}
```

**Campos-chave para triage automático:**
- `impact.totalImpactPercentage` → severidade relativa
- `impact.totalImpact` → valor absoluto do desvio
- `rootCauses[0]` → account + region + usageType para diagnóstico rápido
- `anomalyEndDate == null` → anomalia ainda em andamento

---

## 9. Diagrama: Pipeline Completo de Governança de Custo

```
                     PIPELINE DE GOVERNANÇA DE CUSTO

  Recursos AWS          Coleta/Análise            Alertas/Ação
  ─────────────         ──────────────            ────────────
  EC2, ECS,             Cost Explorer             SNS → Email/Slack
  Lambda,          ───► GetCostAndUsage     ────► Lambda Handler:
  RDS, etc.             (AmortizedCost)            • abrir ticket Jira
      │                      │                     • notify #finops
      │             ┌────────┴──────────┐
      │             │ Cost Anomaly Det. │──────► SNS → Lambda:
      │             │ ML baseline 90d   │         • parse rootCauses
      │             │ 3× checks/day     │         • tag offending resource
      │             │ up to 24h delay   │
      │             └───────────────────┘
      │
      │             Compute Optimizer               Relatório
      └──────────── (opt-in, 14d default)  ───────► S3 CSV:
                    EC2, ECS, Lambda, EBS,          rightsizing_YYYY-MM.csv
                    RDS, ASG                        • instance_id
                    Finding:                        • current_type
                      OVER_PROVISIONED              • recommended_type
                      UNDER_PROVISIONED             • monthly_savings
                      OPTIMIZED
```

---

## 10. Armadilhas (Pitfalls)

[FATO] **Cost Allocation Tags não são retroativas**: ao ativar uma tag, só dados **futuros** são indexados. Custos anteriores à ativação não aparecem filtrados por essa tag.

[FATO] **Cost Explorer tem delay de até 24h**: anomalias detectadas podem ter até 24h de atraso. Para alertas em tempo real, use CloudWatch Alarms em vez de Cost Anomaly Detection.

[FATO] **Compute Optimizer requer opt-in**: não coleta dados até ser explicitamente habilitado. Se ativado hoje, as primeiras recomendações só aparecem após 14 dias de coleta de métricas.

[FATO] **Memória EC2 sem CloudWatch agent = recomendações apenas por CPU**: Compute Optimizer não consegue recomendar baseado em memória sem o CloudWatch agent ou ingestão de métricas externas. Recomendações sem memória podem superestimar over-provisioning.

[FATO] **BlendedCost ≠ UnblendedCost em Organizations**: em contas membros, BlendedCost distribui descontos de RI/SP da conta de gerenciamento proporcionalmente, podendo aparecer **menor** que o custo real cobrado à conta. Use UnblendedCost para chargeback real por conta.

[CONSENSO] **Customer Managed Monitor com threshold único**: ao usar um único monitor customer-managed para múltiplos projetos/tags com volumes de custo muito diferentes (ex.: $50 e $50.000/mês), o mesmo threshold absoluto gera muitos falsos positivos no menor ou silence no maior. Prefira monitores separados por grupo de custo similar.

[FATO] **Compute Optimizer não considera comprometimento de SP/RI**: por padrão, recomenda instâncias sem considerar seus Savings Plans ou Reserved Instances existentes. Use `put-recommendation-preferences` com `preferredResources` para restringir às famílias cobertas.

---

## 11. Quando usar cada ferramenta

```
┌─────────────────────────────────┬────────────────────────────────────┐
│ Pergunta                        │ Ferramenta                         │
├─────────────────────────────────┼────────────────────────────────────┤
│ Quanto custou o projeto X       │ Cost Explorer — GetCostAndUsage    │
│ este mês?                       │ com filter por tag + AmortizedCost │
├─────────────────────────────────┼────────────────────────────────────┤
│ Gasto subiu inesperadamente?    │ Cost Anomaly Detection             │
│ Qual serviço causou?            │ (ML detects + rootCauses)          │
├─────────────────────────────────┼────────────────────────────────────┤
│ Vou estourar o budget?          │ Cost Explorer — GetCostForecast    │
│                                 │ + Budgets com threshold FORECASTED │
├─────────────────────────────────┼────────────────────────────────────┤
│ Qual instância está             │ Compute Optimizer — EC2/ECS recs   │
│ superdimensionada?              │ finding=OVER_PROVISIONED           │
├─────────────────────────────────┼────────────────────────────────────┤
│ Comparar períodos de custo      │ Cost Explorer — GetCostComparisons │
├─────────────────────────────────┼────────────────────────────────────┤
│ Rightsizing em escala (500+     │ Compute Optimizer export → S3 →   │
│ instâncias)?                    │ Athena + QuickSight                │
└─────────────────────────────────┴────────────────────────────────────┘
```

---

## Exercício de Reflexão

Um time de engenharia quer implementar um sistema automático de governança de custo que:

1. **Detecte** quando qualquer serviço de um projeto específico gasta mais que 150% do esperado
2. **Identifique** a causa raiz (conta, região, tipo de uso)
3. **Atualize** uma tag de instância com o status `cost-review=pending`
4. **Abra** um ticket automático no sistema de tickets da empresa

**Desenhe a arquitetura completa**, respondendo:

1. Qual tipo de monitor do Cost Anomaly Detection usar — AWS Managed ou Customer Managed? Por quê?
2. Como configurar o threshold para detectar exatamente "150% do esperado"? Qual campo usar: absoluto ou percentual?
3. Qual é o caminho completo do dado: da ocorrência do gasto até o ticket ser aberto? Quais são os delays?
4. Por que não usar Cost Explorer diretamente para detectar anomalias em tempo real?
5. O que o Compute Optimizer acrescentaria nesse fluxo? Em que momento do pipeline seria mais útil?

---

## Referências

- [FATO] [What is AWS Cost Explorer](https://docs.aws.amazon.com/cost-management/latest/userguide/ce-what-is.html) — docs.aws.amazon.com
- [FATO] [Getting started with AWS Cost Anomaly Detection](https://docs.aws.amazon.com/cost-management/latest/userguide/getting-started-ad.html) — docs.aws.amazon.com
- [FATO] [What is AWS Compute Optimizer](https://docs.aws.amazon.com/compute-optimizer/latest/ug/what-is-compute-optimizer.html) — docs.aws.amazon.com
- [FATO] [Rightsizing recommendation preferences](https://docs.aws.amazon.com/compute-optimizer/latest/ug/rightsizing-preferences.html) — docs.aws.amazon.com
- [FATO] [Cost allocation tags](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/cost-alloc-tags.html) — docs.aws.amazon.com
- [FATO] [GetCostAndUsage API Reference](https://docs.aws.amazon.com/aws-cost-management/latest/APIReference/API_GetCostAndUsage.html) — docs.aws.amazon.com
