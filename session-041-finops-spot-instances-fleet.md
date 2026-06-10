# Sessão 041 — FinOps: Spot Instances, EC2 Fleet e Interruption Handling

**Duração estimada:** 60 min  
**Pré-requisito:** session-040 (Savings Plans e Reserved Instances)

---

## Objetivos da sessão

- Entender o modelo de preço e risco de Spot Instances
- Distinguir os APIs modernos (CreateFleet / Auto Scaling) dos legados (RequestSpotFleet — **não usar**)
- Dominar as estratégias de alocação: `price-capacity-optimized`, `capacity-optimized`, `diversified`
- Implementar interruption handling completo: rebalance recommendation + 2-min notice via IMDS e EventBridge
- Aplicar diversificação de instance types e attribute-based selection
- Usar Spot Placement Scores para escolha de região/AZ

---

## 1. Modelo Econômico e Risco

[FATO] Spot Instances oferecem desconto de **até 90%** em relação ao preço On-Demand, dando acesso à capacidade EC2 ociosa da AWS. A AWS pode interromper uma Spot Instance com **aviso de 2 minutos** quando precisa da capacidade de volta.

[FATO] Um **Spot capacity pool** é o conjunto de instâncias ociosas com o mesmo tipo de instância **e** a mesma Availability Zone. O preço Spot é definido por pool e varia com oferta e demanda.

```
Razões de interrupção (docs AWS):
┌─────────────────────────────────────────────────────────────────┐
│  CAPACITY  — AWS precisa da capacidade de volta (causa principal)│
│  PRICE     — Spot price subiu acima do seu maxPrice             │
│  CONSTRAINT— launch group / AZ group não pode mais ser satisfeito│
└─────────────────────────────────────────────────────────────────┘

Comportamento na interrupção (configurável):
  terminate  — padrão; instância encerra
  stop       — EBS preservado; pode reiniciar quando capacidade volta
  hibernate  — RAM salva em EBS; WITHOUT 2-min warning (hibernação imediata)
```

[FATO] Workloads adequados para Spot: **stateless, fault-tolerant, flexíveis** — big data, containerized, CI/CD, web servers sem estado, HPC, rendering. Workloads **não adequados**: inflexíveis, stateful, tightly-coupled entre nós, intolerantes a qualquer período de capacidade parcial.

[CONSENSO] Tentar fazer failover de Spot para On-Demand em resposta a interrupções pode inadvertidamente causar mais interrupções em outras Spot Instances. AWS desencoraja explicitamente esse padrão.

---

## 2. APIs: o que usar e o que evitar

[FATO] A documentação oficial (atualizada 2026) classifica explicitamente os APIs de Spot em:

```
╔══════════════════════════════════╦══════════════════════════════════════════╗
║ API                              ║ Recomendação                             ║
╠══════════════════════════════════╬══════════════════════════════════════════╣
║ CreateAutoScalingGroup           ║ ✅ SIM — lifecycle gerenciado, scaling    ║
║ CreateFleet (instant mode)       ║ ✅ SIM — sem auto scaling necessário      ║
║ RunInstances                     ║ ⚠️  LIMITED — apenas 1 tipo de instância  ║
║ RequestSpotFleet                 ║ ❌ NÃO — API legado, sem investimento     ║
║ RequestSpotInstances             ║ ❌ NÃO — API legado, sem investimento     ║
╚══════════════════════════════════╩══════════════════════════════════════════╝
```

[FATO] `RequestSpotFleet` e `RequestSpotInstances` são explicitamente marcados como legacy APIs com "no planned investment" na documentação AWS. Novos workloads devem usar **Auto Scaling Groups** ou **EC2 Fleet**.

---

## 3. Estratégias de Alocação

[FATO] Ao usar múltiplos pools de capacidade (EC2 Fleet ou Auto Scaling), a **allocation strategy** determina de quais pools as instâncias serão lançadas.

### 3.1 price-capacity-optimized (recomendada)

[FATO] Identificado como "best choice for most Spot workloads" pela documentação AWS. O fleet identifica os pools com **maior disponibilidade de capacidade** e, entre esses, escolhe os de **menor preço**. Resultado: menor taxa de interrupção + bom custo.

```
Seleção: pools com mais capacidade disponível → menor preço entre esses
Uso:     stateless containers, microservices, web apps, data/analytics, batch
CLI:     --spot-allocation-strategy price-capacity-optimized (EC2 Fleet)
         --spot-allocation-strategy priceCapacityOptimized  (Spot Fleet legado)
```

### 3.2 capacity-optimized

[FATO] Foca exclusivamente em **máxima disponibilidade de capacidade**, sem considerar preço. Útil quando o custo de reprocessamento de uma interrupção é muito alto (CI longa, renderização, HPC com horas de computação). Aceita variante `capacity-optimized-prioritized` para ordenar tipos de instância.

### 3.3 diversified

[FATO] Distribui instâncias entre **todos os pools** configurados igualmente. Se configurados 10 pools e target=100, lança 10 instâncias em cada. Protege contra interrupção em massa de um único pool (somente 10% afetado).

```
Quando usar: fleets grandes ou que rodam por muito tempo
Limitação:   não lança em pools onde Spot price ≥ On-Demand price
```

### 3.4 lowest-price (NÃO recomendada)

[FATO] **Maior risco de interrupção** — só considera preço, ignora disponibilidade de capacidade. Pools com maior demanda (mais baratos) tendem a ter maior taxa de interrupção. AWS documenta explicitamente: "We don't recommend the lowest-price allocation strategy."

---

## 4. Diversificação de Instance Types

[CONSENSO] A regra de ouro da AWS é ser flexível em pelo menos **10 tipos de instância** por workload, além de habilitar **todas as AZs** disponíveis na VPC.

```
Estratégia de diversificação para workload de processamento:
  Família c (compute): c5.xlarge, c5a.xlarge, c5n.xlarge, c4.xlarge
  Família m (general): m5.xlarge, m5a.xlarge, m5n.xlarge, m4.xlarge
  Família r (memory):  r5.large, r5a.large  (se workload aceitar)

Princípio: se flexível verticalmente → incluir instâncias maiores (mais vCPUs)
           se apenas escala horizontal → incluir gerações antigas (menor demanda OD)
```

### 4.1 Attribute-based Instance Type Selection (ABITS)

[FATO] Em vez de listar tipos específicos, especifica **atributos** (vCPUs mínimo/máximo, memória, arquitetura, etc.) e a AWS seleciona automaticamente todos os tipos compatíveis. Garante uso de novos tipos de instância conforme forem lançados.

```python
# CDK Python — attribute-based via CfnAutoScalingGroup
# (ver seção 7 para exemplo completo)
instance_requirements = autoscaling.CfnAutoScalingGroup.InstanceRequirementsProperty(
    v_cpu_count=autoscaling.CfnAutoScalingGroup.VCpuCountRequestProperty(min=2, max=8),
    memory_mi_b=autoscaling.CfnAutoScalingGroup.MemoryMiBRequestProperty(min=4096, max=16384),
    cpu_manufacturers=["intel", "amd"],
    instance_generations=["current"],
)
```

---

## 5. Spot Placement Scores

[FATO] O **Spot Placement Score** (1–10) indica a probabilidade de provisionar a capacidade Spot solicitada em uma região ou AZ específica. Score 10 = altamente provável de sucesso. É uma recomendação **point-in-time** — não garante capacidade nem prediz taxa de interrupção futura.

```bash
# Obter placement score para 100 vCPUs em múltiplas regiões
aws ec2 get-spot-placement-scores \
  --target-capacity 100 \
  --target-capacity-unit-type vcpu \
  --single-availability-zone-flag false \
  --instance-requirements-with-metadata '{
    "ArchitectureTypes": ["x86_64"],
    "VirtualizationTypes": ["hvm"],
    "InstanceRequirements": {
      "VCpuCount": {"Min": 2, "Max": 4},
      "MemoryMiB": {"Min": 4096}
    }
  }'
# Retorna: RegionName, Score (1-10), AvailabilityZoneId (se solicitado)
```

---

## 6. Interruption Handling — Dois Sinais

### 6.1 Arquitetura temporal de interrupção

```
Linha do tempo de uma interrupção:

 T-?min          T-2min                    T
  │               │                         │
  ▼               ▼                         ▼
 [Rebalance      [Interruption Notice    [Instância
  Recommendation] aparece no IMDS e       interrompida]
  EventBridge]    EventBridge emite]
  
  ← best-effort → ← garantido* 2 min →

* exceto hibernation: começa imediatamente, sem 2 min de aviso
```

[FATO] O **Rebalance Recommendation** pode chegar **antes** do aviso de 2 minutos, dando mais tempo para ação proativa. Porém, é emitido em **best-effort** — pode chegar junto com o aviso de 2 minutos, ou não chegar.

[FATO] O **Interruption Notice** (aviso de 2 minutos) é garantido para as ações `terminate` e `stop`. Para `hibernate`, a hibernação começa imediatamente sem aviso prévio de 2 minutos.

### 6.2 Monitoramento via IMDS (Instance Metadata Service)

[FATO] Ambos os sinais são disponibilizados como itens no IMDS v2. AWS recomenda polling a cada **5 segundos**.

```
Endpoint do Rebalance Recommendation:
  GET http://169.254.169.254/latest/meta-data/events/recommendations/rebalance
  Resposta quando presente: {"noticeTime": "2024-01-15T14:22:00Z"}
  Resposta quando ausente:  HTTP 404

Endpoint do Interruption Notice:
  GET http://169.254.169.254/latest/meta-data/spot/instance-action
  Resposta quando presente: {"action": "terminate", "time": "2024-01-15T14:25:00Z"}
                            {"action": "stop",      "time": "2024-01-15T14:25:00Z"}
  Resposta quando ausente:  HTTP 404
```

### 6.3 Monitoramento via EventBridge

[FATO] Ambos os sinais são emitidos como eventos no EventBridge na conta/região da instância. `detail-type` permite filtrar:

```json
// Evento: Rebalance Recommendation
{
  "detail-type": "EC2 Instance Rebalance Recommendation",
  "source": "aws.ec2",
  "detail": { "instance-id": "i-0abcdef1234567890" }
}

// Evento: Interruption Warning (2 min)
{
  "detail-type": "EC2 Spot Instance Interruption Warning",
  "source": "aws.ec2",
  "detail": {
    "instance-id": "i-0abcdef1234567890",
    "instance-action": "terminate"   // ou "stop" ou "hibernate"
  }
}
```

### 6.4 Capacity Rebalancing (Auto Scaling / EC2 Fleet)

[FATO] Auto Scaling Groups e EC2 Fleet têm feature nativa de **Capacity Rebalancing**: quando recebem o sinal de Rebalance Recommendation, automaticamente lançam instâncias de substituição **antes** de encerrar as instâncias em risco. Isso mantém a capacidade alvo durante transições.

---

## 7. CDK Python — Auto Scaling Group com Spot (Mixed Instances)

```python
from aws_cdk import (
    Stack, Duration, aws_ec2 as ec2,
    aws_autoscaling as autoscaling,
    aws_iam as iam, aws_sns as sns,
    aws_events as events, aws_events_targets as targets,
    aws_lambda as _lambda,
)
from constructs import Construct

class SpotFleetStack(Stack):
    def __init__(self, scope: Construct, construct_id: str, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        vpc = ec2.Vpc(self, "VPC",
            max_azs=3,  # todos os AZs para diversificação
            nat_gateways=1,
        )

        # Security Group para o processador
        sg = ec2.SecurityGroup(self, "WorkerSG", vpc=vpc, allow_all_outbound=True)

        # IAM Role para as instâncias
        role = iam.Role(self, "WorkerRole",
            assumed_by=iam.ServicePrincipal("ec2.amazonaws.com"),
            managed_policies=[
                iam.ManagedPolicy.from_aws_managed_policy_name(
                    "AmazonSSMManagedInstanceCore"
                ),
            ],
        )

        # Launch Template com user data para graceful shutdown
        user_data = ec2.UserData.for_linux()
        user_data.add_commands(
            "#!/bin/bash",
            "yum install -y aws-cli jq",
            # Instala e inicia o daemon de interruption handling
            "cat > /usr/local/bin/spot-handler.sh << 'EOF'",
            "#!/bin/bash",
            "while true; do",
            "  TOKEN=$(curl -s -X PUT http://169.254.169.254/latest/api/token \\",
            "    -H 'X-aws-ec2-metadata-token-ttl-seconds: 21600')",
            "  ACTION=$(curl -s -H \"X-aws-ec2-metadata-token: $TOKEN\" \\",
            "    http://169.254.169.254/latest/meta-data/spot/instance-action 2>/dev/null)",
            "  if echo \"$ACTION\" | grep -q 'terminate\\|stop'; then",
            "    echo 'INTERRUPTION NOTICE: '$ACTION",
            "    systemctl stop worker.service",
            "    aws sqs send-message --queue-url $DRAIN_QUEUE_URL \\",
            "      --message-body \"{\\\"instance_id\\\":\\\"$(curl -s -H \\\"X-aws-ec2-metadata-token: $TOKEN\\\" \\",
            "      http://169.254.169.254/latest/meta-data/instance-id)\\\"}\"",
            "    break",
            "  fi",
            "  REBALANCE=$(curl -s -H \"X-aws-ec2-metadata-token: $TOKEN\" \\",
            "    http://169.254.169.254/latest/meta-data/events/recommendations/rebalance 2>/dev/null)",
            "  if echo \"$REBALANCE\" | grep -q 'noticeTime'; then",
            "    echo 'REBALANCE RECOMMENDATION: '$REBALANCE",
            "    # Para novos jobs mas não encerra os existentes",
            "    touch /var/run/spot-draining",
            "  fi",
            "  sleep 5",
            "done",
            "EOF",
            "chmod +x /usr/local/bin/spot-handler.sh",
            "nohup /usr/local/bin/spot-handler.sh &",
        )

        launch_template = ec2.LaunchTemplate(self, "WorkerLT",
            machine_image=ec2.MachineImage.latest_amazon_linux2(),
            role=role,
            security_group=sg,
            user_data=user_data,
            # NÃO definir instance_type aqui quando usar mixed instances policy
        )

        # Auto Scaling Group com política de instâncias mistas
        # Usa CfnAutoScalingGroup (L1) para acesso à MixedInstancesPolicy completa
        asg = autoscaling.CfnAutoScalingGroup(self, "WorkerASG",
            min_size="2",
            max_size="20",
            desired_capacity="6",
            vpc_zone_identifier=vpc.select_subnets(
                subnet_type=ec2.SubnetType.PRIVATE_WITH_EGRESS
            ).subnet_ids,
            # Capacity Rebalancing — lança substituta antes de encerrar instância em risco
            capacity_rebalance=True,
            mixed_instances_policy=autoscaling.CfnAutoScalingGroup.MixedInstancesPolicyProperty(
                launch_template=autoscaling.CfnAutoScalingGroup.LaunchTemplateProperty(
                    launch_template_specification=autoscaling.CfnAutoScalingGroup.LaunchTemplateSpecificationProperty(
                        launch_template_id=launch_template.launch_template_id,
                        version=launch_template.latest_version_number,
                    ),
                    overrides=[
                        # Diversificação: ≥ 10 tipos de instância
                        autoscaling.CfnAutoScalingGroup.LaunchTemplateOverridesProperty(
                            instance_type="c5.xlarge"),
                        autoscaling.CfnAutoScalingGroup.LaunchTemplateOverridesProperty(
                            instance_type="c5a.xlarge"),
                        autoscaling.CfnAutoScalingGroup.LaunchTemplateOverridesProperty(
                            instance_type="c5n.xlarge"),
                        autoscaling.CfnAutoScalingGroup.LaunchTemplateOverridesProperty(
                            instance_type="c4.xlarge"),
                        autoscaling.CfnAutoScalingGroup.LaunchTemplateOverridesProperty(
                            instance_type="m5.xlarge"),
                        autoscaling.CfnAutoScalingGroup.LaunchTemplateOverridesProperty(
                            instance_type="m5a.xlarge"),
                        autoscaling.CfnAutoScalingGroup.LaunchTemplateOverridesProperty(
                            instance_type="m5n.xlarge"),
                        autoscaling.CfnAutoScalingGroup.LaunchTemplateOverridesProperty(
                            instance_type="m4.xlarge"),
                        autoscaling.CfnAutoScalingGroup.LaunchTemplateOverridesProperty(
                            instance_type="r5.large"),
                        autoscaling.CfnAutoScalingGroup.LaunchTemplateOverridesProperty(
                            instance_type="r5a.large"),
                    ],
                ),
                instances_distribution=autoscaling.CfnAutoScalingGroup.InstancesDistributionProperty(
                    # 0% On-Demand base, 100% Spot
                    on_demand_base_capacity=0,
                    on_demand_percentage_above_base_capacity=0,
                    spot_allocation_strategy="price-capacity-optimized",  # RECOMENDADA
                ),
            ),
        )

        # --- EventBridge: Lambda para Interruption Warning ---
        interrupt_fn = _lambda.Function(self, "SpotInterruptHandler",
            runtime=_lambda.Runtime.PYTHON_3_12,
            handler="index.handler",
            code=_lambda.Code.from_inline("""
import boto3, json, os

ec2 = boto3.client('ec2')
asg = boto3.client('autoscaling')

def handler(event, context):
    instance_id = event['detail']['instance-id']
    action = event['detail']['instance-action']
    print(f"INTERRUPTION WARNING: {instance_id} will be {action}")
    
    # Desanexar do ASG graciosamente (permite que o ASG repõe a instância)
    try:
        response = asg.describe_auto_scaling_instances(InstanceIds=[instance_id])
        if response['AutoScalingInstances']:
            asg_name = response['AutoScalingInstances'][0]['AutoScalingGroupName']
            asg.detach_instances(
                InstanceIds=[instance_id],
                AutoScalingGroupName=asg_name,
                ShouldDecrementDesiredCapacity=False,  # ASG provisiona substituta
            )
            print(f"Detached {instance_id} from ASG {asg_name}")
    except Exception as e:
        print(f"ASG detach failed (may be OK): {e}")
    
    return {"statusCode": 200, "instanceId": instance_id, "action": action}
"""),
        )
        interrupt_fn.add_to_role_policy(iam.PolicyStatement(
            actions=["autoscaling:DescribeAutoScalingInstances",
                     "autoscaling:DetachInstances",
                     "ec2:DescribeInstances"],
            resources=["*"],
        ))

        # EventBridge rule: Interruption Warning → Lambda
        events.Rule(self, "SpotInterruptRule",
            event_pattern=events.EventPattern(
                source=["aws.ec2"],
                detail_type=["EC2 Spot Instance Interruption Warning"],
            ),
            targets=[targets.LambdaFunction(interrupt_fn)],
        )

        # EventBridge rule: Rebalance Recommendation → Lambda (mesma função ou outra)
        events.Rule(self, "SpotRebalanceRule",
            event_pattern=events.EventPattern(
                source=["aws.ec2"],
                detail_type=["EC2 Instance Rebalance Recommendation"],
            ),
            targets=[targets.LambdaFunction(interrupt_fn)],
        )
```

---

## 8. Python — Análise de Spot Price History e Savings

```python
import boto3
from datetime import datetime, timedelta, timezone
from statistics import mean, stdev
from dataclasses import dataclass
from typing import Optional

ec2 = boto3.client("ec2", region_name="us-east-1")

@dataclass
class SpotPoolAnalysis:
    instance_type: str
    az: str
    current_price: float
    avg_price_7d: float
    price_stdev: float
    on_demand_price: float
    discount_pct: float
    volatility_coefficient: float  # stdev / mean — quanto varia relativamente


def analyze_spot_pools(
    instance_types: list[str],
    on_demand_prices: dict[str, float],  # {instance_type: price}
    lookback_days: int = 7,
) -> list[SpotPoolAnalysis]:
    """
    Analisa histórico de preço Spot para encontrar pools mais estáveis e baratos.
    Nota: preço baixo + baixa volatilidade = pool com boa disponibilidade consistente.
    """
    end = datetime.now(timezone.utc)
    start = end - timedelta(days=lookback_days)

    results = []
    for instance_type in instance_types:
        paginator = ec2.get_paginator("describe_spot_price_history")
        pages = paginator.paginate(
            InstanceTypes=[instance_type],
            ProductDescriptions=["Linux/UNIX"],
            StartTime=start,
            EndTime=end,
        )

        # Agrupa por AZ
        prices_by_az: dict[str, list[float]] = {}
        for page in pages:
            for entry in page["SpotPriceHistory"]:
                az = entry["AvailabilityZone"]
                price = float(entry["SpotPrice"])
                prices_by_az.setdefault(az, []).append(price)

        od_price = on_demand_prices.get(instance_type, 0.0)

        for az, prices in prices_by_az.items():
            if len(prices) < 2:
                continue
            avg = mean(prices)
            sd = stdev(prices)
            cv = sd / avg if avg > 0 else 0  # coefficient of variation

            results.append(SpotPoolAnalysis(
                instance_type=instance_type,
                az=az,
                current_price=prices[0],  # mais recente
                avg_price_7d=avg,
                price_stdev=sd,
                on_demand_price=od_price,
                discount_pct=((od_price - avg) / od_price * 100) if od_price > 0 else 0,
                volatility_coefficient=cv,
            ))

    # Ordena: menor volatilidade primeiro, depois menor preço
    return sorted(results, key=lambda r: (r.volatility_coefficient, r.avg_price_7d))


def print_pool_ranking(pools: list[SpotPoolAnalysis], top_n: int = 5):
    print(f"\n{'Tipo':15} {'AZ':20} {'Atual':8} {'Média7d':8} {'Desconto':9} {'Volatilidade':13}")
    print("-" * 75)
    for p in pools[:top_n]:
        print(
            f"{p.instance_type:15} {p.az:20} "
            f"${p.current_price:.4f} ${p.avg_price_7d:.4f} "
            f"{p.discount_pct:6.1f}%    CV={p.volatility_coefficient:.3f}"
        )


# Exemplo de uso
if __name__ == "__main__":
    instance_types = [
        "c5.xlarge", "c5a.xlarge", "c5n.xlarge",
        "m5.xlarge", "m5a.xlarge",
    ]
    # Preços On-Demand us-east-1 (referência — verificar em ec2.aws.amazon.com/pricing)
    od_prices = {
        "c5.xlarge":  0.170,
        "c5a.xlarge": 0.154,
        "c5n.xlarge": 0.216,
        "m5.xlarge":  0.192,
        "m5a.xlarge": 0.172,
    }

    pools = analyze_spot_pools(instance_types, od_prices, lookback_days=7)
    print_pool_ranking(pools, top_n=10)

    # Identifica pools com desconto > 50% e volatilidade baixa (CV < 0.10)
    prime_pools = [p for p in pools if p.discount_pct > 50 and p.volatility_coefficient < 0.10]
    print(f"\nPools 'prime' (>50% desconto, CV<0.10): {len(prime_pools)}")
    for p in prime_pools:
        print(f"  → {p.instance_type} @ {p.az}: ${p.avg_price_7d:.4f}/h ({p.discount_pct:.1f}% off)")
```

---

## 9. CLI — Exemplos Essenciais

```bash
# 1. Histórico de preço Spot — últimas 24h para tipos específicos
aws ec2 describe-spot-price-history \
  --instance-types c5.xlarge m5.xlarge \
  --product-descriptions "Linux/UNIX" \
  --start-time $(date -u -d '24 hours ago' +%Y-%m-%dT%H:%M:%SZ) \
  --query 'SpotPriceHistory[*].{Type:InstanceType,AZ:AvailabilityZone,Price:SpotPrice,Time:Timestamp}' \
  --output table

# 2. Spot Placement Score — identifica melhor região para 100 vCPUs
aws ec2 get-spot-placement-scores \
  --target-capacity 100 \
  --target-capacity-unit-type vcpu \
  --single-availability-zone-flag false \
  --instance-requirements-with-metadata '{
    "ArchitectureTypes": ["x86_64"],
    "VirtualizationTypes": ["hvm"],
    "InstanceRequirements": {
      "VCpuCount": {"Min": 2, "Max": 8},
      "MemoryMiB": {"Min": 4096, "Max": 32768},
      "InstanceGenerations": ["current"]
    }
  }' \
  --query 'SpotPlacementScores | sort_by(@, &Score) | reverse(@)' \
  --output table

# 3. Criar EC2 Fleet (instant mode) com price-capacity-optimized
aws ec2 create-fleet \
  --type instant \
  --target-capacity-specification TotalTargetCapacity=10,DefaultTargetCapacityType=spot \
  --spot-options AllocationStrategy=price-capacity-optimized,MaxTotalPrice=1.50 \
  --launch-template-configs '[
    {
      "LaunchTemplateSpecification": {
        "LaunchTemplateId": "lt-0abcdef1234567890",
        "Version": "$Latest"
      },
      "Overrides": [
        {"InstanceType": "c5.xlarge",  "SubnetId": "subnet-aaa"},
        {"InstanceType": "c5a.xlarge", "SubnetId": "subnet-aaa"},
        {"InstanceType": "m5.xlarge",  "SubnetId": "subnet-bbb"},
        {"InstanceType": "m5a.xlarge", "SubnetId": "subnet-bbb"},
        {"InstanceType": "c5.xlarge",  "SubnetId": "subnet-ccc"},
        {"InstanceType": "m5.xlarge",  "SubnetId": "subnet-ccc"}
      ]
    }
  ]' \
  --query 'FleetId'

# 4. Descrever frota e suas instâncias
aws ec2 describe-fleets \
  --fleet-ids fleet-0abc1234def56789 \
  --query 'Fleets[0].{State:FleetState,Capacity:TargetCapacitySpecification}'

aws ec2 describe-fleet-instances \
  --fleet-id fleet-0abc1234def56789

# 5. ASG — verificar instâncias Spot vs On-Demand na frota mista
aws autoscaling describe-auto-scaling-instances \
  --query 'AutoScalingInstances[?AutoScalingGroupName==`WorkerASG`].[InstanceId,InstanceType,LifecycleState,HealthStatus]' \
  --output table

# 6. Cancelar EC2 Fleet e suas instâncias
aws ec2 delete-fleets \
  --fleet-ids fleet-0abc1234def56789 \
  --terminate-instances

# 7. Simular interrupção com AWS FIS (Fault Injection Service)
#    — útil para testar graceful shutdown em ambientes de staging
aws fis create-experiment-template \
  --description "Test Spot interruption handling" \
  --actions '{"InterruptSpot": {
    "actionId": "aws:ec2:send-spot-instance-interruptions",
    "parameters": {"durationBeforeInterruption": "PT2M"},
    "targets": {"SpotInstances": "targetInstances"}
  }}' \
  --targets '{"targetInstances": {
    "resourceType": "aws:ec2:spot-instance",
    "resourceArns": ["arn:aws:ec2:us-east-1:123456789012:instance/i-0abc123"],
    "selectionMode": "ALL"
  }}' \
  --role-arn arn:aws:iam::123456789012:role/FISRole \
  --stop-conditions '[{"source":"none"}]'
```

---

## 10. Billing de Instâncias Interrompidas

[FATO] Conforme documentação oficial AWS:

```
╔═══════════════════════════════════════════════════════════════════╗
║ Quem interrompeu       │ Cobrança da hora parcial                 ║
╠═══════════════════════════════════════════════════════════════════╣
║ AWS (interrupção Spot) │ Primeira hora: NÃO cobrada               ║
║                        │ Horas subsequentes parciais: COBRADAS    ║
╠═══════════════════════════════════════════════════════════════════╣
║ Você (terminate manual)│ Hora parcial: COBRADA normalmente        ║
╚═══════════════════════════════════════════════════════════════════╝

Exemplo: instância roda 2h 23min, AWS termina → paga 2h (inteiras)
         instância roda 0h 47min, AWS termina → paga R$0 (primeira hora)
         instância roda 1h 23min, você termina → paga 1h 23min (proporcional)
```

[FATO] Para instâncias Spot com comportamento `stop` e `hibernate`: a instância para de acumular cobrança de computação quando parada, mas EBS e Elastic IPs continuam sendo cobrados.

---

## 11. Diagrama: Pipeline de Interrupção Handling

```
Fluxo completo de interrupção em produção:

 AWS decide         IMDS/EventBridge        Aplicação
 interromper
     │
     ├──[early]──► EC2 Instance        ──► ASG lança instância
     │             Rebalance Rec.           substituta proativamente
     │             (best-effort)
     │
     ├──[T-2min]─► EC2 Spot Instance   ──► Lambda EventBridge Handler:
     │             Interruption Warning      • Detach do ASG
     │             (garantido*)              • Notifica SQS/SNS
     │                                       • Log de contexto
     │
     ├──[T-2min]─► IMDS polls (5s)     ──► spot-handler.sh:
     │             /spot/instance-action     • systemctl stop worker
     │                                       • flush pending work
     │                                       • upload logs to S3
     │
     └──[T=0]────► Instância termina   ──► ASG mantém desired capacity
                   (ou para/hiberna)        com nova instância saudável

* Exceto hibernate: sem aviso de 2min (hiberna imediatamente)
```

---

## 12. Armadilhas (Pitfalls)

[FATO] **RequestSpotFleet é legado**: projetos novos devem usar CreateFleet ou Auto Scaling Groups. A documentação AWS marca RequestSpotFleet como "no planned investment" — não há garantia de suporte futuro.

[FATO] **lowest-price tem maior risco de interrupção**: a estratégia foca exclusivamente em preço, sem considerar disponibilidade. Pools baratos tendem a ter alta demanda On-Demand → maiores interrupções. Use `price-capacity-optimized`.

[FATO] **maxPrice mal configurado aumenta interrupções**: especificar `maxPrice` causa mais interrupções do que não especificá-lo. Se o Spot price exceder seu limite, a instância é interrompida. A maioria dos workloads não deve definir maxPrice.

[FATO] **Hibernate não tem 2 min de aviso**: a hibernação começa imediatamente ao receber o sinal — não há janela de 2 minutos para graceful shutdown. Use `terminate` ou `stop` se precisar de tempo para cleanup.

[CONSENSO] **Spot Instances não são adequadas para banco de dados primário**: stateful, tightly coupled, e intolerante a interrupção. Use On-Demand ou Reserved Instances para databases.

[FATO] **Capacidade Rebalancing pode causar scale-out temporário**: ao lançar substituta antes de encerrar a instância em risco, o ASG fica temporariamente acima da desired capacity. Isso é esperado e não é bug.

[CONSENSO] **Spot + Savings Plans não se acumulam**: Savings Plans cobrem uso On-Demand e Spot de Fargate/Lambda, mas **não cobrem Spot EC2**. Para Spot EC2, o desconto já está embutido no preço Spot.

[INCERTO] **Taxa de interrupção por pool**: o Spot Instance Advisor (console) mostra frequências históricas de interrupção categorizadas (<5%, 5-10%, 10-15%, >20%). Essas categorias são indicativas mas não garantem comportamento futuro.

---

## 13. Quando NÃO usar Spot Instances

[FATO] A documentação AWS é explícita — Spot Instances **não são adequadas** para:

```
NÃO usar Spot para:
  ✗ Workloads inflexíveis (exigem tipo exato de instância)
  ✗ Workloads stateful sem checkpointing
  ✗ Aplicações tightly-coupled entre nós (HPC MPI sem checkpoint)
  ✗ Banco de dados primário (stateful, intolerante)
  ✗ Workloads que não toleram qualquer período sem capacidade total
  ✗ Aplicações que dependem de failover para On-Demand*

USAR Spot para:
  ✓ Big data / ETL batch
  ✓ Containers stateless (ECS, EKS)
  ✓ CI/CD runners
  ✓ Web servers stateless (com ALB)
  ✓ HPC com checkpointing
  ✓ Rendering / encoding
  ✓ ML training (com checkpointing em S3)

* Failover Spot→OD pode causar mais interrupções em outras Spot
```

---

## 14. Resumo Visual

```
                    SPOT INSTANCES — MAPA CONCEITUAL

  Preço                                   Interrupção
  ─────                                   ───────────
  Até 90% desconto                        AWS avisa ~2 min antes
  Varia por pool (tipo + AZ)              Rebalance Rec: aviso antecipado
  Histórico: describe-spot-price-history  Polling IMDS a cada 5s
  Placement Score: 1-10 por região/AZ     EventBridge: 2 event types
                  │                                     │
                  ▼                                     ▼
         Diversificação                       Handling Actions
         ─────────────                       ────────────────
         ≥ 10 instance types                 Graceful shutdown
         Todos os AZs habilitados            Checkpoint em S3/DynamoDB
         ABITS: atributos, não tipos fixos   Detach do ASG
                  │                          Capacity Rebalancing
                  ▼
         Allocation Strategy
         ───────────────────
         price-capacity-optimized ← RECOMENDADA
         capacity-optimized       ← alto custo de interrupção
         diversified              ← fleets grandes/longas
         lowest-price             ← NÃO USAR
                  │
                  ▼
         APIs Modernos
         ─────────────
         CreateAutoScalingGroup   ← com lifecycle
         CreateFleet (instant)    ← sem auto scaling
         RequestSpotFleet         ← LEGADO, não usar
         RequestSpotInstances     ← LEGADO, não usar
```

---

## Exercício de Reflexão

Um time de ML usa um Auto Scaling Group com política `lowest-price` e 3 tipos de instância (`p3.2xlarge` apenas). O treinamento leva 6 horas e não tem checkpointing. A taxa de interrupção está alta (>20% dos jobs são interrompidos antes de completar), causando custo alto de reprocessamento.

**Identifique todos os problemas de design e proponha uma solução arquitetural completa**, considerando:

1. Qual estratégia de alocação deve ser usada e por quê?
2. Como a diversificação de instance types deve ser feita para GPUs?
3. O que deve mudar no design do job de treinamento para tolerar interrupções?
4. Como o Capacity Rebalancing ajuda (ou não) nesse cenário específico?
5. Existe alguma situação em que Spot seria inadequado mesmo com todas as melhorias acima?

---

## Referências

- [FATO] [Best practices for Amazon EC2 Spot](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-best-practices.html) — docs.aws.amazon.com (atualizado 2026)
- [FATO] [Spot Instance interruption notices](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-instance-termination-notices.html) — docs.aws.amazon.com
- [FATO] [EC2 instance rebalance recommendations](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/rebalance-recommendations.html) — docs.aws.amazon.com
- [FATO] [Allocation strategies for EC2 Fleet or Spot Fleet](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-fleet-allocation-strategy.html) — docs.aws.amazon.com
- [FATO] [Prepare for Spot Instance interruptions](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/prepare-for-interruptions.html) — docs.aws.amazon.com
- [FATO] [Spot placement score](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-placement-score.html) — docs.aws.amazon.com
