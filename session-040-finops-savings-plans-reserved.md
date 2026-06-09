# Sessão 040 — FinOps: Savings Plans vs Reserved Instances — diferenças e flexibilidade

**Duração estimada:** 60 minutos  
**Dependências:** nenhuma (tópico independente)

---

## Objetivo

Ao final desta sessão, você conseguirá comparar Compute Savings Plans, EC2 Instance Savings Plans e Reserved Instances por flexibilidade de instância, escopo regional vs zonal e desconto; calcular o ROI de cada opção para um perfil de workload específico; e entender o risco de over-commitment.

---

## Contexto

[FATO] On-Demand pricing é o modelo padrão: você paga por hora de uso sem compromisso. AWS oferece descontos substanciais em troca de **compromisso de uso por 1 ou 3 anos**. Os dois mecanismos principais são **Savings Plans** (compromisso em $/hora de uso) e **Reserved Instances** (compromisso em capacidade específica).

[FATO] AWS lançou Savings Plans em novembro de 2019 como evolução dos Reserved Instances, com maior flexibilidade. A própria AWS recomenda Savings Plans sobre Reserved Instances para a maioria dos casos de uso — mas Reserved Instances ainda oferecem vantagens específicas (capacity reservation, desconto zonal).

---

## Conceitos principais

### 1. Quatro tipos de Savings Plans

[FATO] AWS oferece quatro tipos de Savings Plans:

```
Tipo                    Desconto max   Serviços cobertos
────────────────────────────────────────────────────────────────────
Compute Savings Plans   até 66%        EC2 (qualquer família/tamanho/
                                       região/OS/tenancy), Fargate,
                                       Lambda

EC2 Instance            até 72%        EC2 dentro de uma família
Savings Plans                          específica em uma região
                                       específica (ex: m5 em us-east-1)
                                       — qualquer tamanho/OS/tenancy

Database Savings Plans  até 35%        Aurora, RDS, DynamoDB,
                                       ElastiCache, DocumentDB,
                                       Timestream, Neptune, Keyspaces,
                                       DMS, OpenSearch

SageMaker AI            até 64%        SageMaker AI — qualquer família/
Savings Plans                          tamanho/região/componente
────────────────────────────────────────────────────────────────────
```

[FATO] Savings Plans comprometem um valor em **$/hora** (não um número de instâncias). Se você usa mais do que o comprometido, o excedente é cobrado em On-Demand. Se usa menos, você paga pelo comprometido mesmo sem uso.

---

### 2. Tipos de Reserved Instances (RIs) para EC2

[FATO] Reserved Instances para EC2 têm dois tipos principais por escopo de flexibilidade:

```
Tipo                Desconto max   Flexibilidade
────────────────────────────────────────────────────────────────────
Standard RI         até 72%        Família, tamanho, OS e região
                                   fixos. Size flexibility em RIs
                                   regionais (dentro da família).
                                   Pode vender no Marketplace.

Convertible RI      até 66%        Pode fazer exchange para mudar
                                   família, tamanho, OS ou tenancy.
                                   Exchange é manual (não automático).
                                   NÃO pode vender no Marketplace.
────────────────────────────────────────────────────────────────────
```

[FATO] **Escopo de RI** — Regional vs. Zonal:

```
Escopo Regional (recomendado):
  - Desconto aplicado a qualquer instância na região
  - Instance size flexibility: desconto se aplica
    automaticamente a outros tamanhos da mesma família
    (ex: comprou m5.large — aplica em m5.xlarge, m5.2xlarge etc.)
  - SEM reserva de capacidade garantida

Escopo Zonal (específico por AZ):
  - Desconto aplicado apenas à AZ especificada
  - SEM size flexibility (instância exata)
  - COM reserva de capacidade garantida nessa AZ
  - Pode ser vendido no Marketplace (Standard RI)
```

---

### 3. Tabela comparativa completa

[FATO — AWS Docs]:

```
                        Compute   EC2 Instance   Convertible   Standard
                        SP        SP             RI            RI
─────────────────────────────────────────────────────────────────────────
Desconto máx.           66%       72%            66%           72%
Compromisso             $/hora    $/hora         # instâncias  # instâncias
Família flexível        ✓         —              manual        —
Tamanho flexível        ✓         ✓              manual        regional only
OS flexível             ✓         ✓              manual        —
Tenancy flexível        ✓         ✓              manual        —
Cross-region            ✓         —              —             —
Cobre Fargate           ✓         —              —             —
Cobre Lambda            ✓         —              —             —
Capacity reservation    —         —              zonal only    zonal only
Cancelamento            —         —              —             —
Exchange/modificação    N/A       N/A            ✓ (manual)    limitado
Venda no Marketplace    —         —              —             ✓ (zonal)
─────────────────────────────────────────────────────────────────────────
✓ = automático; — = não disponível; manual = requer ação explícita
```

---

### 4. Opções de pagamento e impacto no desconto

[FATO] Todos os tipos de Savings Plans e RIs oferecem três opções de pagamento:

```
Pagamento           Compromisso   Desconto
────────────────────────────────────────────────────────────────
All Upfront         100% adiantado  Máximo (66%, 72% etc.)
Partial Upfront     ~50% adiantado  Intermediário
No Upfront          Zero adiantado  Menor (ainda substancial)

Prazo de 3 anos > prazo de 1 ano em desconto para todas as opções.

Exemplo (m5.xlarge Linux us-east-1):
  On-Demand:              $0.192/hora  = $1.684/mês
  Standard RI 1y All Up:  $0.110/hora  = $965/mês  → 43% off
  Standard RI 3y All Up:  $0.069/hora  = $605/mês  → 64% off
  Compute SP 1y All Up:   $0.116/hora  = $1.018/mês → 40% off
  Compute SP 3y All Up:   $0.073/hora  = $640/mês  → 62% off
  EC2 Instance SP 1y:     $0.109/hora  = $957/mês  → 43% off
```

[INCERTO] Os valores exatos de desconto variam por região, família de instância e OS — sempre verifique o pricing calculator da AWS para o seu caso específico, pois os percentuais mudam periodicamente.

---

### 5. Como Savings Plans são aplicados

[FATO] A aplicação de Savings Plans segue uma hierarquia:

```
Ordem de aplicação:
1. Savings Plans cobrem primeiro as instâncias com MENOR preço
   On-Demand no seu portfólio (maximiza cobertura)
2. EC2 Instance SP (mais específico) é aplicado ANTES de
   Compute SP (mais genérico)
3. O que exceder o compromisso $/hora → cobrado On-Demand

Exemplo:
  Você tem Compute SP de $1.00/hora (3 anos, no upfront)
  Suas instâncias custam:
    - m5.large Linux us-east-1: $0.096/hora On-Demand
    - c5.xlarge Linux eu-west-1: $0.192/hora On-Demand
  
  Com Compute SP, você paga o preço SP em vez do On-Demand
  para as instâncias que cobrem o commitment:
    ~$1.00/hora → cobre essas instâncias com desconto de ~60%
  Acima disso → On-Demand
```

[FATO] Savings Plans **não fornecem capacity reservation**. Se você precisa garantir capacidade em uma AZ específica (ex: failover crítico), use **On-Demand Capacity Reservation (ODCR)** — Savings Plans se aplicam sobre o ODCR.

---

### 6. Análise de ROI e risco de over-commitment

[CONSENSO] O risco central de Savings Plans e RIs é o **over-commitment**: comprometer mais do que você usa. O comprometimento é irrevogável durante o prazo — você paga mesmo sem uso.

```
Framework de decisão:
─────────────────────────────────────────────────────────────
Baseline usage           → Comprometer (Savings Plans / RI)
Variável mas previsível  → Comprometer baseline, On-Demand para picos
Imprevisível             → On-Demand (ou Spot para tolerantes a interrupção)

Regra prática:
  Comprometa no máximo o uso do seu P10 (percentil 10)
  — o uso que você tem 90% do tempo como mínimo garantido.
  
  Nunca comprometa o uso médio se houver sazonalidade,
  pois nos períodos abaixo da média você paga sem receber.
```

**Calculando break-even (Savings Plans 1 ano vs. On-Demand):**

```
On-Demand:       $0.192/hora × 8.760h/ano = $1.682/ano
Compute SP 1y:   $0.116/hora × 8.760h/ano = $1.016/ano  (66% utilização)

Break-even: você precisa usar a instância pelo menos X% do tempo
para o SP ser mais barato que On-Demand:

Break-even % = SP_price / OD_price = 0.116 / 0.192 = 60.4%

→ Se a instância ficar ativa > 60.4% do tempo: SP é mais barato
→ Se ficar ativa < 60.4% do tempo: On-Demand seria mais barato
```

---

## Exemplo prático

### Cenário: avaliando a estratégia FinOps para uma plataforma de e-commerce

**Workload:**
- 10 instâncias `m5.xlarge` Linux em us-east-1 rodando 24/7 (web tier estável)
- 5 instâncias `c5.2xlarge` Linux em us-east-1 rodando 70% do tempo (processamento batch)
- Lambda functions (1.000.000 invocações/mês, 512MB, 500ms avg duration)
- Fargate tasks (3.000 vCPU-hora/mês, 6.000 GB-hora/mês)

#### CDK Python — Cost Anomaly Detection + Budget alert

```python
from aws_cdk import (
    Stack,
    aws_budgets as budgets,
    aws_ce as ce,  # Cost Explorer
)
from constructs import Construct


class FinOpsCostControlStack(Stack):
    def __init__(self, scope: Construct, construct_id: str, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        # ── Budget: alerta quando custo mensal exceder $10.000 ────────
        monthly_budget = budgets.CfnBudget(
            self, "MonthlyComputeBudget",
            budget=budgets.CfnBudget.BudgetDataProperty(
                budget_name="monthly-compute-budget",
                budget_type="COST",
                time_unit="MONTHLY",
                budget_limit=budgets.CfnBudget.SpendProperty(
                    amount=10000,
                    unit="USD",
                ),
                cost_filters={
                    "Service": ["Amazon Elastic Compute Cloud - Compute",
                                "AWS Lambda",
                                "AWS Fargate"],
                },
                cost_types=budgets.CfnBudget.CostTypesProperty(
                    include_tax=True,
                    include_subscription=True,
                    use_blended=False,
                ),
            ),
            notifications_with_subscribers=[
                budgets.CfnBudget.NotificationWithSubscribersProperty(
                    notification=budgets.CfnBudget.NotificationProperty(
                        comparison_operator="GREATER_THAN",
                        notification_type="ACTUAL",   # custo real (não forecast)
                        threshold=80,                 # 80% do budget
                        threshold_type="PERCENTAGE",
                    ),
                    subscribers=[
                        budgets.CfnBudget.SubscriberProperty(
                            address="finops@company.com",
                            subscription_type="EMAIL",
                        ),
                    ],
                ),
                budgets.CfnBudget.NotificationWithSubscribersProperty(
                    notification=budgets.CfnBudget.NotificationProperty(
                        comparison_operator="GREATER_THAN",
                        notification_type="FORECASTED",  # forecast excede budget
                        threshold=100,
                        threshold_type="PERCENTAGE",
                    ),
                    subscribers=[
                        budgets.CfnBudget.SubscriberProperty(
                            address="finops@company.com",
                            subscription_type="EMAIL",
                        ),
                    ],
                ),
            ],
        )

        # ── Cost Anomaly Detection ────────────────────────────────────
        # Detecta anomalias de custo automaticamente (ML-based)
        anomaly_monitor = ce.CfnAnomalyMonitor(
            self, "ComputeCostMonitor",
            monitor_name="compute-cost-monitor",
            monitor_type="DIMENSIONAL",
            monitor_dimension="SERVICE",
        )

        anomaly_subscription = ce.CfnAnomalySubscription(
            self, "ComputeAnomalyAlert",
            subscription_name="compute-anomaly-alert",
            monitor_arn_list=[anomaly_monitor.attr_monitor_arn],
            subscribers=[
                ce.CfnAnomalySubscription.SubscriberProperty(
                    address="finops@company.com",
                    type="EMAIL",
                    status="CONFIRMED",
                ),
            ],
            # Alerta quando impacto total exceder $100
            threshold_expression=ce.CfnAnomalySubscription.ExpressionProperty(
                dimensions=ce.CfnAnomalySubscription.DimensionValuesProperty(
                    key="ANOMALY_TOTAL_IMPACT_ABSOLUTE",
                    values=["100"],
                    match_options=["GREATER_THAN_OR_EQUAL"],
                )
            ),
            frequency="DAILY",
        )
```

#### Cálculo de ROI — Python (análise offline)

```python
"""
Script de análise FinOps: calcula ROI de Savings Plans vs On-Demand
para o perfil de workload descrito no cenário.

Preços aproximados us-east-1 (verificar sempre no pricing calculator):
https://aws.amazon.com/ec2/pricing/reserved-instances/pricing/
"""

from dataclasses import dataclass
from typing import Literal


@dataclass
class WorkloadComponent:
    name: str
    on_demand_hourly: float
    utilization_pct: float      # 0.0 – 1.0
    count: int
    hours_per_year: float = 8760.0


def annual_on_demand_cost(w: WorkloadComponent) -> float:
    return w.on_demand_hourly * w.utilization_pct * w.hours_per_year * w.count


def annual_savings_plan_cost(
    w: WorkloadComponent,
    sp_hourly_rate: float,
) -> float:
    """
    Savings Plan é sempre cobrado pelo compromisso (sp_hourly_rate * count),
    independente da utilização — por isso over-commitment é perigoso.
    """
    committed = sp_hourly_rate * w.hours_per_year * w.count
    return committed


def break_even_utilization(
    on_demand_rate: float,
    sp_rate: float,
) -> float:
    """Utilização mínima para SP ser mais barato que On-Demand."""
    return sp_rate / on_demand_rate


def analyze_workload():
    # ── Workload components ───────────────────────────────────────────
    web_tier = WorkloadComponent(
        name="Web tier (m5.xlarge, 10x)",
        on_demand_hourly=0.192,     # m5.xlarge Linux us-east-1
        utilization_pct=1.0,        # 100% — sempre ligado
        count=10,
    )
    batch_tier = WorkloadComponent(
        name="Batch tier (c5.2xlarge, 5x)",
        on_demand_hourly=0.340,     # c5.2xlarge Linux us-east-1
        utilization_pct=0.70,       # 70% do tempo
        count=5,
    )

    # ── Preços aproximados Savings Plans 1y No Upfront ────────────────
    # NOTA: sempre verifique o pricing atual em aws.amazon.com
    WEB_SP_RATE   = 0.116   # Compute SP ~40% off On-Demand (m5.xlarge)
    BATCH_SP_RATE = 0.204   # Compute SP ~40% off On-Demand (c5.2xlarge)

    print("=" * 65)
    print("ANÁLISE FINOPS — Savings Plans vs On-Demand")
    print("=" * 65)

    for workload, sp_rate in [(web_tier, WEB_SP_RATE), (batch_tier, BATCH_SP_RATE)]:
        od_annual = annual_on_demand_cost(workload)
        sp_annual = annual_savings_plan_cost(workload, sp_rate)
        savings   = od_annual - sp_annual
        be_util   = break_even_utilization(workload.on_demand_hourly, sp_rate)

        print(f"\n{workload.name}")
        print(f"  On-Demand anual:        ${od_annual:,.0f}")
        print(f"  Savings Plan anual:     ${sp_annual:,.0f}")
        print(f"  Economia:               ${savings:,.0f}  ({savings/od_annual*100:.1f}%)")
        print(f"  Break-even utilização:  {be_util*100:.1f}%")
        print(f"  Utilização atual:       {workload.utilization_pct*100:.0f}%")
        print(f"  → {'✓ SP RECOMENDADO' if workload.utilization_pct > be_util else '✗ On-Demand mais barato'}")

    # ── Risco de over-commitment no batch tier ────────────────────────
    print("\n--- ANÁLISE DE RISCO (Batch tier) ---")
    print("Cenário: utilização cai para 40% (metade do esperado)")
    batch_worst = WorkloadComponent(
        name="Batch tier — worst case",
        on_demand_hourly=0.340,
        utilization_pct=0.40,   # pior caso
        count=5,
    )
    od_worst  = annual_on_demand_cost(batch_worst)
    sp_annual = annual_savings_plan_cost(batch_tier, BATCH_SP_RATE)
    waste     = sp_annual - od_worst
    print(f"  On-Demand (40% util):   ${od_worst:,.0f}")
    print(f"  Savings Plan (fixo):    ${sp_annual:,.0f}")
    print(f"  Desperdício:            ${waste:,.0f}  (pagando sem usar)")
    print(f"  → RECOMENDAÇÃO: comprometa apenas o baseline seguro (P10)")


if __name__ == "__main__":
    analyze_workload()
```

#### CLI — Verificar recomendações e comprar Savings Plan

```bash
# 1. Obter recomendações de Savings Plans do Cost Explorer (AWS CLI)
aws ce get-savings-plans-purchase-recommendation \
  --savings-plans-type COMPUTE_SP \
  --term-in-years ONE_YEAR \
  --payment-option NO_UPFRONT \
  --lookback-period-in-days THIRTY_DAYS \
  --query 'SavingsPlansPurchaseRecommendation.{
    Summary:SavingsPlansPurchaseRecommendationSummary,
    Details:SavingsPlansPurchaseRecommendationDetails[0:3]
  }'

# 2. Ver utilização atual de Savings Plans existentes
aws ce get-savings-plans-utilization \
  --time-period '{"Start":"2026-06-01","End":"2026-06-30"}' \
  --query 'Total.{
    Utilized:Utilization.UtilizationPercentage,
    UnusedHours:Savings.NetSavings,
    TotalCommitment:Utilization.TotalCommitment
  }'

# 3. Ver cobertura de Savings Plans (% do On-Demand coberto)
aws ce get-savings-plans-coverage \
  --time-period '{"Start":"2026-06-01","End":"2026-06-30"}' \
  --granularity MONTHLY \
  --query 'SavingsPlansCoverages[*].Coverage.{
    CoverageHoursPercentage:CoverageHoursPercentage,
    SpendCoveredBySavingsPlans:SpendCoveredBySavingsPlans,
    OnDemandCost:OnDemandCost
  }'

# 4. Comprar Savings Plan (CUIDADO — compromisso irrevogável)
# Sempre verificar recomendação do Cost Explorer antes
aws savingsplans create-savings-plan \
  --savings-plan-offering-id "offering-id-from-describe" \
  --commitment 1.00 \
  --purchase-time "2026-07-01T00:00:00Z" \
  --client-token "unique-idempotency-key-$(uuidgen)"

# 5. Listar Savings Plans ativos
aws savingsplans describe-savings-plans \
  --states ACTIVE \
  --query 'savingsPlans[*].{
    Id:savingsPlanId,
    Type:savingsPlanType,
    Commitment:commitment,
    Start:start,
    End:end,
    State:state,
    PaymentOption:paymentOption,
    Term:termDurationInSeconds
  }'

# 6. Ver recomendações de Reserved Instances
aws ce get-reservation-purchase-recommendation \
  --service "Amazon EC2" \
  --lookback-period-in-days THIRTY_DAYS \
  --term-in-years ONE_YEAR \
  --payment-option NO_UPFRONT \
  --query 'Recommendations[0:3].{
    Summary:RecommendationSummary,
    Details:RecommendationDetails[0:2]
  }'

# 7. Ver utilização de Reserved Instances
aws ce get-reservation-utilization \
  --time-period '{"Start":"2026-06-01","End":"2026-06-30"}' \
  --query 'Total.UtilizationsByTime[*].{
    UtilizationPct:Utilization.UtilizationPercentage,
    UnusedHours:Utilization.UnusedHours
  }'
```

---

## Armadilhas comuns

**1. Over-commitment: comprometer a média, não o baseline**  
[CONSENSO] A armadilha mais comum em FinOps é comprar Savings Plans ou RIs baseando-se no uso médio dos últimos 30 dias, sem considerar sazonalidade. Um workload com pico de 100 instâncias no Black Friday e mínimo de 20 instâncias em julho deve comprometer apenas as 20 instâncias de baseline — não a média. Over-commitment desperdiça capital em capacidade ociosa.

**2. Savings Plans não fornecem capacity reservation**  
[FATO] Ao contrário de Reserved Instances zonais, Savings Plans não garantem que a capacidade estará disponível. Em cenários de escassez regional (ex: durante eventos de grande escala ou outages), On-Demand e Savings Plans concorrem pela capacidade disponível. Para cargas críticas com SLA de disponibilidade, combine Savings Plans com On-Demand Capacity Reservation (ODCR).

**3. Savings Plans não cobrem Spot usage**  
[FATO] Savings Plans e RIs são aplicados apenas ao uso On-Demand e ODCR. Horas Spot não recebem desconto adicional de Savings Plans — elas já têm seu próprio preço com desconto. Planejar cobertura de Savings Plans sobre workloads que você pretende mover para Spot resulta em compromisso desnecessário.

**4. Convertible RIs exigem exchange manual — não são automáticas**  
[FATO] Ao contrário de EC2 Instance Savings Plans (que se aplicam automaticamente a qualquer tamanho na família), Convertible RIs requerem que você **execute manualmente o exchange** via console ou API quando quer mudar família, tamanho ou OS. Esquecer de fazer o exchange significa que a RI antiga continua sem ser utilizada enquanto você paga On-Demand pelas novas instâncias.

**5. Standard RI zonal: size flexibility ausente**  
[FATO] Standard RIs com escopo **Zonal** não têm size flexibility — o desconto só se aplica exatamente ao tamanho, OS e tenancy especificados na compra. RIs com escopo **Regional** têm size flexibility dentro da família. Para obter flexibilidade de tamanho com RI, sempre use escopo Regional.

**6. Savings Plans são aplicados sequencialmente, não em paralelo**  
[FATO] Se você tem múltiplos Savings Plans (ex: EC2 Instance SP e Compute SP), o X-Ray aplica primeiro o EC2 Instance SP (mais específico), depois o Compute SP para o restante. Comprar ambos sem analisar a sobreposição pode resultar em cobertura redundante pagando pelo Compute SP sem aproveitá-lo.

---

## Exercício de reflexão

Sua empresa tem o seguinte perfil de uso EC2 em us-east-1 (dados dos últimos 90 dias):

- **Instâncias estáveis:** 20 x `m5.2xlarge` Linux rodando 100% do tempo (aplicação web)
- **Instâncias de processamento:** 10 x `c5.4xlarge` Linux rodando em média 60% do tempo, com mínimo de 40% e máximo de 90% (jobs diurnos)
- **Instâncias de ML:** 3 x `p3.2xlarge` Linux rodando em média 30% do tempo (treinamento semanal)

Responda:

1. Para as instâncias estáveis, qual a diferença financeira anual entre Compute Savings Plans 1 ano All Upfront (desconto ~60%) e Standard RI 1 ano All Upfront (desconto ~64%), considerando `m5.2xlarge` On-Demand a $0.384/hora? Quando a flexibilidade do Compute SP justifica a diferença de desconto?

2. Para as instâncias de processamento, qual seria o commitment seguro de Savings Plans considerando o risco de over-commitment? Justifique usando o conceito de P10 (percentil 10 de utilização = 40%).

3. Para as instâncias de ML (`p3.2xlarge`, 30% de utilização média): o break-even de Compute SP (desconto ~60%) justifica o compromisso? Qual seria a alternativa mais adequada para esse tipo de workload?

---

## Recursos para aprofundar

- [FATO] Savings Plans types (Compute, EC2 Instance, Database, SageMaker): https://docs.aws.amazon.com/savingsplans/latest/userguide/plan-types.html
- [FATO] Compute Savings Plans vs Reserved Instances (tabela comparativa): https://docs.aws.amazon.com/savingsplans/latest/userguide/sp-ris.html
- [FATO] How Savings Plans apply to usage: https://docs.aws.amazon.com/savingsplans/latest/userguide/sp-applying.html
- [FATO] Reserved Instances for EC2 (escopo regional vs zonal, Standard vs Convertible): https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-reserved-instances.html
- [INCERTO] AWS Pricing Calculator (preços atuais — verificar antes de comprometer): https://calculator.aws/pricing/2/home

---

