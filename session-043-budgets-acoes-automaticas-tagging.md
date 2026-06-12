# Sessão 043 — AWS Budgets com Ações Automáticas e Tagging Strategy para Chargeback

**Duração estimada:** 60 min  
**Pré-requisito:** session-042 (Cost Explorer, Anomaly Detection, Compute Optimizer)

---

## Objetivos da sessão

- Configurar AWS Budgets com ações automáticas (IAM policy, SCP, instâncias EC2/RDS)
- Entender o modelo de execução AUTOMATIC vs MANUAL e quando usar cada um
- Projetar hierarquia de tags obrigatórias para chargeback por projeto/equipe
- Distinguir os três modelos de cost allocation (account-based, team-based, tag-based)
- Usar AWS Config Rules para detectar recursos não conformes com a tagging policy
- Usar Tag Policies para normalização de valores em Organizations

---

## 1. AWS Budgets — Tipos e Estrutura

[FATO] AWS Budgets suporta seis tipos de orçamento:

```
╔══════════════════════════════════╦═══════════════════════════════════════════╗
║ Tipo                             ║ O que monitora                            ║
╠══════════════════════════════════╬═══════════════════════════════════════════╣
║ COST                             ║ Custo monetário ($)                       ║
║ USAGE                            ║ Volume de uso (GB, horas, etc.)           ║
║ RI_UTILIZATION                   ║ % de uso das Reserved Instances           ║
║ RI_COVERAGE                      ║ % de horas On-Demand cobertas por RI      ║
║ SAVINGS_PLANS_UTILIZATION        ║ % de uso dos Savings Plans comprados      ║
║ SAVINGS_PLANS_COVERAGE           ║ % de horas On-Demand cobertas por SP      ║
╚══════════════════════════════════╩═══════════════════════════════════════════╝
```

[FATO] Alertas de Budget têm dois tipos de threshold:
- `ACTUAL`: custo/uso já incorrido (retroativo — não previne, avisa)
- `FORECASTED`: projeção do modelo de previsão para o período corrente

[FATO] Limite padrão: **20.000 budgets por conta**. Custo: os primeiros 2 budgets são gratuitos; a partir do 3º, $0.02/dia por budget (≈ $0.60/mês).

---

## 2. Budget Actions — Ações Automáticas

[FATO] Ao atingir um threshold, Budget Actions pode executar automaticamente (ou aguardar aprovação manual) uma das seguintes ações:

```
╔═══════════════════════════════════╦═════════════════════════════════════════════╗
║ Tipo de Ação                      ║ O que faz                                   ║
╠═══════════════════════════════════╬═════════════════════════════════════════════╣
║ APPLY_IAM_POLICY                  ║ Anexa uma IAM policy (deny) a user/role/    ║
║                                   ║ group — impede provisionar novos recursos    ║
╠═══════════════════════════════════╬═════════════════════════════════════════════╣
║ APPLY_SCP_POLICY                  ║ Aplica um SCP a uma conta-membro na         ║
║                                   ║ Organization — só management account pode    ║
╠═══════════════════════════════════╬═════════════════════════════════════════════╣
║ EC2_STOP_INSTANCES /              ║ Para ou encerra instâncias EC2              ║
║ EC2_TERMINATE_INSTANCES           ║ específicas na conta-alvo                   ║
╠═══════════════════════════════════╬═════════════════════════════════════════════╣
║ RDS_STOP_INSTANCES /              ║ Para instâncias RDS específicas             ║
║ RDS_TERMINATE_INSTANCES           ║ (não pode targetar outra conta)             ║
╚═══════════════════════════════════╩═════════════════════════════════════════════╝
```

### 2.1 AUTOMATIC vs MANUAL

[FATO] Cada ação configurada pode ser:
- **AUTOMATIC** (`ApprovalModel="AUTOMATIC"`): executa imediatamente ao atingir o threshold
- **MANUAL** (`ApprovalModel="MANUAL"`): coloca a ação em estado `STANDBY` — aguarda aprovação na console ou via `execute-budget-action`

[CONSENSO] Ações de **stop/terminate de instâncias** devem ser preferencialmente `MANUAL` em produção — o risco de encerrar instâncias produtivas acidentalmente é alto. Para ambientes de dev/sandbox, `AUTOMATIC` é adequado.

[FATO] A ação aplicada **não é revertida automaticamente** quando o custo cai abaixo do threshold. O operador deve manualmente remover o SCP ou desanexar a IAM policy.

### 2.2 IAM Role necessária

[FATO] AWS Budgets precisa de uma IAM Role com permissões específicas para executar ações. A AWS fornece managed policy `AWSBudgetsActionsRolePolicyForResourceAdministrationWithSSM`:

```json
// Trust policy necessária para Budget Actions
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "budgets.amazonaws.com" },
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": { "aws:SourceAccount": "ACCOUNT_ID" }
    }
  }]
}

// Permissões mínimas para SCP + IAM actions
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "iam:AttachGroupPolicy", "iam:AttachRolePolicy", "iam:AttachUserPolicy",
        "iam:DetachGroupPolicy", "iam:DetachRolePolicy", "iam:DetachUserPolicy"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": ["organizations:AttachPolicy", "organizations:DetachPolicy"],
      "Resource": "*"
    }
  ]
}
```

---

## 3. CDK Python — Budget com Ação Automática SCP

```python
from aws_cdk import (
    Stack, aws_budgets as budgets, aws_iam as iam,
    aws_sns as sns, aws_organizations as organizations,
)
from constructs import Construct

class BudgetActionsStack(Stack):
    def __init__(self, scope: Construct, construct_id: str,
                 target_account_id: str,
                 **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        # ──────────────────────────────────────────────────────────────
        # IAM Role para Budget Actions
        # ──────────────────────────────────────────────────────────────
        budget_actions_role = iam.Role(self, "BudgetActionsRole",
            assumed_by=iam.ServicePrincipal("budgets.amazonaws.com",
                conditions={
                    "StringEquals": {
                        "aws:SourceAccount": self.account
                    }
                }
            ),
            managed_policies=[
                iam.ManagedPolicy.from_aws_managed_policy_name(
                    "AWSBudgetsActionsRolePolicyForResourceAdministrationWithSSM"
                )
            ],
        )

        # ──────────────────────────────────────────────────────────────
        # SCP de deny — nega provisionamento de recursos EC2 novos
        # Será anexado à conta-alvo quando budget atingir 80%
        # ──────────────────────────────────────────────────────────────
        deny_ec2_scp = organizations.CfnPolicy(self, "DenyEC2ProvisionSCP",
            name="DenyNewEC2Provision",
            description="Applied automatically when budget reaches 80% — blocks new EC2 launches",
            type="SERVICE_CONTROL_POLICY",
            content={
                "Version": "2012-10-17",
                "Statement": [{
                    "Sid": "DenyEC2Launch",
                    "Effect": "Deny",
                    "Action": [
                        "ec2:RunInstances",
                        "ec2:StartInstances",
                    ],
                    "Resource": "*",
                    "Condition": {
                        "StringNotLike": {
                            "aws:PrincipalArn": [
                                # Exceções: conta de automação e SRE
                                f"arn:aws:iam::{target_account_id}:role/AutomationRole",
                                f"arn:aws:iam::{target_account_id}:role/SREBreakGlass",
                            ]
                        }
                    }
                }]
            },
            target_ids=[target_account_id],  # OU ID ou Account ID
        )

        # SNS para notificações
        alert_topic = sns.Topic(self, "BudgetAlertTopic")

        # ──────────────────────────────────────────────────────────────
        # Budget com múltiplos thresholds e ações encadeadas
        # ──────────────────────────────────────────────────────────────
        budget = budgets.CfnBudget(self, "ProjectBudget",
            budget=budgets.CfnBudget.BudgetDataProperty(
                budget_name="checkout-service-prod-monthly",
                budget_type="COST",
                time_unit="MONTHLY",
                budget_limit=budgets.CfnBudget.SpendProperty(
                    amount=10000, unit="USD"
                ),
                cost_filters={
                    "TagKeyValue": ["user:project$checkout-service"]
                },
                cost_types=budgets.CfnBudget.CostTypesProperty(
                    include_tax=True,
                    include_subscription=True,
                    use_amortized=True,   # AmortizedCost — inclui RI/SP amortizados
                    use_blended=False,
                ),
            ),
            notifications_with_subscribers=[
                # Threshold 1: 80% real → notificação
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
                # Threshold 2: 100% previsto → notificação
                budgets.CfnBudget.NotificationWithSubscribersProperty(
                    notification=budgets.CfnBudget.NotificationProperty(
                        comparison_operator="GREATER_THAN",
                        notification_type="FORECASTED",
                        threshold=100,
                        threshold_type="PERCENTAGE",
                    ),
                    subscribers=[
                        budgets.CfnBudget.SubscriberProperty(
                            address="finops-team@company.com",
                            subscription_type="EMAIL",
                        )
                    ],
                ),
            ],
        )

        # ──────────────────────────────────────────────────────────────
        # Budget Action: ao atingir 80% real → aplicar SCP (AUTOMATIC)
        # A ação é separada do CfnBudget — usa CfnBudgetsAction
        # ──────────────────────────────────────────────────────────────
        # Nota: CfnBudgetsAction não é suportado diretamente pelo CDK L1
        # como construct. Usar aws_cdk.aws_budgets.CfnBudgetsAction
        from aws_cdk import aws_budgets as budgets_l1
        
        budgets_l1.CfnBudgetsAction(self, "ApplySCPAt80Pct",
            budget_name=budget.ref,  # referência ao budget acima
            action_type="APPLY_SCP_POLICY",
            action_threshold=budgets_l1.CfnBudgetsAction.ActionThresholdProperty(
                type="PERCENTAGE",
                value=80,
            ),
            execution_role_arn=budget_actions_role.role_arn,
            notification_type="ACTUAL",
            approval_model="AUTOMATIC",  # executa sem aprovação manual
            definition=budgets_l1.CfnBudgetsAction.DefinitionProperty(
                scp_action_definition=budgets_l1.CfnBudgetsAction.ScpActionDefinitionProperty(
                    policy_id=deny_ec2_scp.ref,  # ID do SCP criado acima
                    target_ids=[target_account_id],
                )
            ),
            subscribers=[
                budgets_l1.CfnBudgetsAction.SubscriberProperty(
                    type="SNS",
                    address=alert_topic.topic_arn,
                )
            ],
        )

        # Budget Action: 90% real → parar instâncias de staging (MANUAL — requer aprovação)
        budgets_l1.CfnBudgetsAction(self, "StopStagingAt90Pct",
            budget_name=budget.ref,
            action_type="EC2_STOP_INSTANCES",
            action_threshold=budgets_l1.CfnBudgetsAction.ActionThresholdProperty(
                type="PERCENTAGE",
                value=90,
            ),
            execution_role_arn=budget_actions_role.role_arn,
            notification_type="ACTUAL",
            approval_model="MANUAL",    # coloca em STANDBY — aguarda aprovação
            definition=budgets_l1.CfnBudgetsAction.DefinitionProperty(
                ec2_action_definition=budgets_l1.CfnBudgetsAction.Ec2ActionDefinitionProperty(
                    instance_ids=["i-0staging123", "i-0staging456"],
                    region="us-east-1",
                )
            ),
            subscribers=[
                budgets_l1.CfnBudgetsAction.SubscriberProperty(
                    type="EMAIL",
                    address="oncall@company.com",
                )
            ],
        )
```

---

## 4. Modelos de Cost Allocation

[FATO] O whitepaper oficial "Best Practices for Tagging AWS Resources" descreve três modelos:

### 4.1 Account-based (menor esforço)

```
Organização AWS Organizations:
  Management Account
    ├── OU: Workloads
    │     ├── Account: checkout-service (prod)
    │     ├── Account: checkout-service (staging)
    │     └── Account: payments-api (prod)
    └── OU: Shared Services
          └── Account: observability (compartilhado)

Vantagem: custo já isolado por conta — sem dependência de tags
Limitação: tags de OU/Organization NÃO são usáveis em Cost Explorer
```

[FATO] Tags de Organizational Units (OUs) no AWS Organizations **não são propagadas** para cost allocation. Para agrupar contas em categorias de custo, use **AWS Cost Categories**.

### 4.2 Business Unit / Team-based (esforço moderado)

[FATO] Combine contas por time/aplicação e use **AWS Cost Categories** para agregar:

```
Cost Category: "Checkout Platform"
  Rule 1: LinkedAccount IN [account-checkout-prod, account-checkout-staging]
  Rule 2: Tag project = "checkout-service"  (para recursos em conta compartilhada)
```

### 4.3 Tag-based (maior esforço, mais granular)

[FATO] Quando múltiplos times compartilham a mesma conta AWS, tags são o único mecanismo de cost allocation por projeto. Requer rigor na aplicação e **ativação no console do Billing** da conta de gerenciamento.

---

## 5. Hierarquia de Tags Obrigatórias

[CONSENSO] O whitepaper de tagging recomenda categorizar tags em 4 grupos, cada um com finalidade distinta:

```
╔════════════════╦═════════════════════╦══════════════════════════════════════╗
║ Categoria      ║ Tag Key             ║ Propósito / Exemplos                 ║
╠════════════════╬═════════════════════╬══════════════════════════════════════╣
║ TÉCNICA        ║ Application         ║ Nome da app (ex: checkout-service)   ║
║                ║ Environment         ║ prod | staging | dev | sandbox       ║
║                ║ Version             ║ v2.3.1 (para rastreio de releases)   ║
║                ║ Component           ║ api | worker | database | cache      ║
╠════════════════╬═════════════════════╬══════════════════════════════════════╣
║ NEGÓCIO        ║ CostCenter          ║ Código do centro de custo contábil   ║
║                ║ Owner               ║ Email do responsável técnico         ║
║                ║ Project             ║ Iniciativa de negócio (ex: pix-v2)   ║
║                ║ Team                ║ Nome do time de engenharia           ║
╠════════════════╬═════════════════════╬══════════════════════════════════════╣
║ SEGURANÇA      ║ Confidentiality     ║ public | internal | confidential     ║
║                ║ Compliance          ║ pci | hipaa | lgpd | none            ║
╠════════════════╬═════════════════════╬══════════════════════════════════════╣
║ AUTOMAÇÃO      ║ Schedule            ║ "Mon-Fri:08:00-20:00" (auto stop)    ║
║                ║ BackupPolicy        ║ daily | weekly | none                ║
║                ║ PatchGroup          ║ grupo de patch SSM                   ║
╚════════════════╩═════════════════════╩══════════════════════════════════════╝
```

[OPINIÃO] Manter tags de negócio e técnicas separadas facilita delegação: times técnicos gerenciam `Application`/`Version`/`Component`; FinOps gerencia `CostCenter`/`Project`; segurança gerencia `Confidentiality`/`Compliance`.

### 5.1 Tags Mínimas para Chargeback

[CONSENSO] O conjunto mínimo funcional para chargeback efetivo (baseado no whitepaper AWS):

```
OBRIGATÓRIAS (sem essas, recurso é não-conforme):
  • project      — alocação de custo principal
  • environment  — separar prod de non-prod
  • owner        — email do responsável

RECOMENDADAS (para granularidade):
  • cost-center  — código contábil para fatura interna
  • team         — agrupamento de times

NÃO USAR como Cost Allocation Tags:
  • Version / Component — alta cardinalidade → custo de indexação excessivo
    (cada combinação única de tags gera uma linha no billing report)
```

---

## 6. Enforcing Tags — Mecanismos de Compliance

[FATO] AWS oferece quatro mecanismos complementares para enforçar tagging:

### 6.1 AWS Config Rules — detectar não-conformidade

[FATO] A managed rule `required-tags` verifica se recursos têm as tags obrigatórias. Suporta até **6 pares key:value** por regra.

```python
# CDK Python: AWS Config Rule para tags obrigatórias
from aws_cdk import aws_config as config

required_tags_rule = config.ManagedRule(self, "RequiredTagsRule",
    identifier="REQUIRED_TAGS",
    input_parameters={
        "tag1Key": "project",
        "tag2Key": "environment",
        "tag2Value": "prod,staging,dev,sandbox",  # valores permitidos
        "tag3Key": "owner",
    },
    description="Verifica se recursos EC2, RDS e Lambda têm tags obrigatórias",
)
```

### 6.2 Tag Policies (AWS Organizations) — normalizar valores

[FATO] Tag Policies (não confundir com IAM policies) impõem formatação consistente de valores de tag. Exemplos: `Environment` sempre em lowercase, valores restritos a lista permitida.

```json
// Tag Policy — normaliza a tag "environment"
{
  "tags": {
    "environment": {
      "tag_key": {
        "@@assign": "environment"
      },
      "tag_value": {
        "@@assign": ["prod", "staging", "dev", "sandbox"]
      },
      "enforced_for": {
        "@@assign": [
          "ec2:instance",
          "rds:db",
          "lambda:function",
          "s3:bucket"
        ]
      }
    }
  }
}
```

[FATO] Tag Policies **não bloqueiam** a criação de recursos sem a tag — elas apenas reportam inconformidade. Para bloquear, use SCP combinado com `aws:RequestedRegion` e condição `aws:TagKeys`.

### 6.3 SCP com condição de tag — bloquear criação sem tag

```json
// SCP: bloqueia RunInstances sem tag "project"
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "RequireProjectTagOnEC2",
    "Effect": "Deny",
    "Action": ["ec2:RunInstances"],
    "Resource": "arn:aws:ec2:*:*:instance/*",
    "Condition": {
      "Null": { "aws:RequestTag/project": "true" }
    }
  }]
}
```

[FATO] SCPs com condição `aws:RequestTag` bloqueiam na criação — não corrigem recursos existentes. Para corrigir retroativamente, use Resource Groups Tag Editor ou AWS CLI.

---

## 7. Python — Tag Compliance Scan com Resource Groups Tagging API

```python
import boto3
from collections import defaultdict
from dataclasses import dataclass, field

tagging = boto3.client("resourcegroupstaggingapi", region_name="us-east-1")
ec2 = boto3.client("ec2", region_name="us-east-1")

MANDATORY_TAGS = {"project", "environment", "owner"}
ALLOWED_ENVIRONMENTS = {"prod", "staging", "dev", "sandbox"}

@dataclass
class NonCompliantResource:
    arn: str
    resource_type: str
    missing_tags: set[str]
    invalid_tags: dict[str, str]  # tag_key → valor_inválido
    existing_tags: dict[str, str]


def scan_tag_compliance(
    resource_type_filters: list[str] | None = None,
    tag_filters: list[dict] | None = None,
) -> list[NonCompliantResource]:
    """
    Varre todos os recursos e retorna os não-conformes com a tag policy.
    resource_type_filters: ex. ["ec2:instance", "rds:db", "lambda:function"]
    """
    params: dict = {}
    if resource_type_filters:
        params["ResourceTypeFilters"] = resource_type_filters
    if tag_filters:
        params["TagFilters"] = tag_filters

    non_compliant = []
    paginator = tagging.get_paginator("get_resources")

    for page in paginator.paginate(**params):
        for resource in page.get("ResourceTagMappingList", []):
            arn = resource["ResourceARN"]
            tags = {t["Key"]: t["Value"] for t in resource.get("Tags", [])}
            tag_keys_lower = {k.lower() for k in tags}

            missing = set()
            invalid = {}

            # Verificar tags obrigatórias
            for required in MANDATORY_TAGS:
                if required not in tags:
                    missing.add(required)

            # Verificar valores permitidos para "environment"
            if "environment" in tags:
                env_val = tags["environment"].lower()
                if env_val not in ALLOWED_ENVIRONMENTS:
                    invalid["environment"] = tags["environment"]

            # Extrair tipo de recurso do ARN (ex.: arn:aws:ec2:...:instance/...)
            parts = arn.split(":")
            resource_type = f"{parts[2]}:{parts[5].split('/')[0]}" if len(parts) > 5 else arn

            if missing or invalid:
                non_compliant.append(NonCompliantResource(
                    arn=arn,
                    resource_type=resource_type,
                    missing_tags=missing,
                    invalid_tags=invalid,
                    existing_tags=tags,
                ))

    return sorted(non_compliant, key=lambda r: len(r.missing_tags), reverse=True)


def print_compliance_summary(resources: list[NonCompliantResource]):
    """Imprime relatório de conformidade agrupado por tipo de recurso."""
    by_type = defaultdict(list)
    for r in resources:
        by_type[r.resource_type].append(r)

    total_missing_by_tag: dict[str, int] = defaultdict(int)
    for r in resources:
        for t in r.missing_tags:
            total_missing_by_tag[t] += 1

    print(f"\n{'='*60}")
    print(f"RELATÓRIO DE CONFORMIDADE DE TAGS")
    print(f"Total não conformes: {len(resources)}")
    print(f"\nTags mais ausentes:")
    for tag, count in sorted(total_missing_by_tag.items(), key=lambda x: -x[1]):
        print(f"  {tag:20} ausente em {count:>4} recursos")

    print(f"\nPor tipo de recurso:")
    for resource_type, res_list in sorted(by_type.items()):
        print(f"\n  {resource_type} ({len(res_list)} não conformes):")
        for r in res_list[:5]:  # mostrar até 5 por tipo
            missing_str = ", ".join(sorted(r.missing_tags)) if r.missing_tags else "-"
            invalid_str = str(r.invalid_tags) if r.invalid_tags else "-"
            print(f"    {r.arn[-50:]:50} | faltando: {missing_str} | inválido: {invalid_str}")
        if len(res_list) > 5:
            print(f"    ... e mais {len(res_list) - 5} recursos")


def apply_default_tags_for_project(
    resource_arns: list[str],
    project: str,
    environment: str,
    owner_email: str,
) -> dict:
    """
    Aplica tags padrão a uma lista de recursos.
    Usar com cautela — sobrescreve tags existentes se as chaves coincidirem.
    """
    response = tagging.tag_resources(
        ResourceARNList=resource_arns,
        Tags={
            "project": project,
            "environment": environment,
            "owner": owner_email,
        },
    )
    failed = response.get("FailedResourcesMap", {})
    succeeded = len(resource_arns) - len(failed)
    return {
        "succeeded": succeeded,
        "failed": len(failed),
        "failed_details": failed,
    }


# Exemplo de uso
if __name__ == "__main__":
    print("Iniciando scan de conformidade...")
    non_compliant = scan_tag_compliance(
        resource_type_filters=["ec2:instance", "rds:db", "lambda:function", "s3:bucket"]
    )
    print_compliance_summary(non_compliant)

    # Exportar ARNs sem tag "project" para remediação
    no_project_arns = [
        r.arn for r in non_compliant if "project" in r.missing_tags
    ]
    print(f"\nARNs sem tag 'project' para remediação: {len(no_project_arns)}")
```

---

## 8. CLI — Exemplos Essenciais

```bash
# ── BUDGETS ────────────────────────────────────────────────────────────────

# 1. Criar budget simples via arquivo JSON (para budgets complexos)
cat > /tmp/budget.json << 'EOF'
{
  "BudgetName": "checkout-service-monthly",
  "BudgetLimit": {"Amount": "10000", "Unit": "USD"},
  "CostFilters": {"TagKeyValue": ["user:project$checkout-service"]},
  "TimeUnit": "MONTHLY",
  "BudgetType": "COST",
  "CostTypes": {
    "IncludeTax": true,
    "IncludeSubscription": true,
    "UseAmortized": true,
    "UseBlended": false
  }
}
EOF

aws budgets create-budget \
  --account-id $(aws sts get-caller-identity --query Account --output text) \
  --budget file:///tmp/budget.json

# 2. Listar budgets da conta
aws budgets describe-budgets \
  --account-id $(aws sts get-caller-identity --query Account --output text) \
  --query 'Budgets[*].{Name:BudgetName,Type:BudgetType,Limit:BudgetLimit.Amount,Actual:CalculatedSpend.ActualSpend.Amount,Forecast:CalculatedSpend.ForecastedSpend.Amount}' \
  --output table

# 3. Listar ações de um budget
aws budgets describe-budget-actions-for-budget \
  --account-id $(aws sts get-caller-identity --query Account --output text) \
  --budget-name "checkout-service-monthly" \
  --query 'Actions[*].{ID:ActionId,Type:ActionType,Status:Status,Approval:ApprovalModel}'

# 4. Listar todas as ações da conta (todos os budgets)
aws budgets describe-budget-actions-for-account \
  --account-id $(aws sts get-caller-identity --query Account --output text) \
  --query 'Actions[?Status==`STANDBY`].[BudgetName,ActionId,ActionType]' \
  --output table

# 5. Executar ação manualmente que está em STANDBY
aws budgets execute-budget-action \
  --account-id $(aws sts get-caller-identity --query Account --output text) \
  --budget-name "checkout-service-monthly" \
  --action-id "abc12345-def6-7890-ghij-klmnopqrstuv" \
  --execution-type APPROVE_BUDGET_ACTION

# 6. Reverter ação executada (remover SCP ou IAM policy)
aws budgets execute-budget-action \
  --account-id $(aws sts get-caller-identity --query Account --output text) \
  --budget-name "checkout-service-monthly" \
  --action-id "abc12345-def6-7890-ghij-klmnopqrstuv" \
  --execution-type REVERSE_BUDGET_ACTION


# ── TAGGING E CONFORMIDADE ─────────────────────────────────────────────────

# 7. Listar recursos EC2 SEM a tag "project" (instâncias não-tagadas)
aws ec2 describe-instances \
  --filters "Name=tag-key,Values=project" \
  --query 'Reservations[*].Instances[*].InstanceId' \
  --output text > /tmp/with-project-tag.txt

aws ec2 describe-instances \
  --query 'Reservations[*].Instances[*].InstanceId' \
  --output text > /tmp/all-instances.txt

# Diferença = instâncias sem tag
comm -23 <(sort /tmp/all-instances.txt) <(sort /tmp/with-project-tag.txt)

# 8. Resource Groups Tagging API — listar recursos sem tag obrigatória
aws resourcegroupstaggingapi get-resources \
  --resource-type-filters "ec2:instance" "lambda:function" "rds:db" \
  --tags-per-page 100 \
  --query 'ResourceTagMappingList[?!contains(Tags[*].Key, `project`)].[ResourceARN]' \
  --output text

# 9. Aplicar tags em batch via Tag Editor API
aws resourcegroupstaggingapi tag-resources \
  --resource-arn-list \
    "arn:aws:ec2:us-east-1:123456789012:instance/i-0abc123" \
    "arn:aws:lambda:us-east-1:123456789012:function:my-function" \
  --tags project=checkout-service,environment=prod,owner=team@company.com

# 10. Relatório de conformidade AWS Config — recursos sem tags
aws configservice get-compliance-details-by-config-rule \
  --config-rule-name "required-tags" \
  --compliance-types "NON_COMPLIANT" \
  --query 'EvaluationResults[*].EvaluationResultIdentifier.EvaluationResultQualifier.ResourceId' \
  --output text

# 11. Tag Policies — listar policies de tag na Organization
aws organizations list-policies \
  --filter TAG_POLICY \
  --query 'Policies[*].{Name:Name,Id:Id,Arn:Arn}'

# 12. Verificar tag policy efetiva em uma conta
aws organizations describe-effective-policy \
  --policy-type TAG_POLICY \
  --target-id 123456789012   # account ID

# 13. Cost Categories — criar categoria para agrupar contas por BU
aws ce create-cost-category-definition \
  --name "BusinessUnit" \
  --rule-version "CostCategoryExpression.v1" \
  --rules '[
    {
      "Value": "Checkout Platform",
      "Rule": {
        "Or": [
          {"Dimensions": {"Key": "LINKED_ACCOUNT", "Values": ["111111111111", "222222222222"]}},
          {"Tags": {"Key": "project", "Values": ["checkout-service", "payments-api"]}}
        ]
      }
    },
    {
      "Value": "Shared Services",
      "Rule": {
        "Dimensions": {"Key": "LINKED_ACCOUNT", "Values": ["333333333333"]}
      }
    }
  ]' \
  --default-value "Unallocated"
```

---

## 9. Showback vs Chargeback

[FATO] Conforme o whitepaper AWS:

```
SHOWBACK
  Finalidade: visibilidade — "você gerou este custo"
  Resultado:  conscientização, mudança de comportamento
  Impacto:    sem transferência financeira real
  Ferramenta: Cost Explorer + dashboards

CHARGEBACK
  Finalidade: recuperação — "você paga por este custo"
  Resultado:  responsabilidade financeira direta
  Impacto:    transferência de custo para centro de custo do time
  Ferramenta: CUR (Cost and Usage Report) + sistema financeiro
```

[CONSENSO] AWS recomenda começar com Showback para criar consciência de custo antes de implementar Chargeback — a mudança cultural é necessária para o Chargeback funcionar adequadamente.

### 9.1 Custo Não-Alocável (Unallocatable Spend)

[FATO] Certos custos AWS **não podem ser alocados por tag**:
- Taxas de compromisso de RI/SP (subscription fees) não têm tag na hora da compra
- Data Transfer fees nem sempre têm tag
- Support Plan é cobrado no nível da conta (não por recurso)

[FATO] Para RI/SP, é possível rastrear *como os descontos foram aplicados* a cada recurso/tag no Cost Explorer (usando AmortizedCost por tag), mas a **taxa de compra** em si não é tagável.

---

## 10. Diagrama: Pipeline de Governança de Tags

```
                PIPELINE DE CONFORMIDADE DE TAGGING

  Criação de Recurso          Detecção                    Remediação
  ──────────────────          ─────────                   ──────────
  IaC (CDK/Terraform)
    → tags no template    AWS Config Rule               Tag Editor API
                          required-tags         ──────► tag-resources em batch
  Console/API direto    ──► avalia a cada
    → SCP bloqueia se         mudança
      sem tag "project"       ↓
      (Deny + Null cond)  NON_COMPLIANT            Notificação ao Owner
                          → SNS → Lambda   ──────► email "recurso sem tag"
                              ↓                    slack #finops-alerts
  Tag Policy (Org)        EventBridge Rule
    → enforce lowercase   aws.config →
    → enforce valores      Config change
      permitidos            (via SNS)
                              ↓
                          CloudWatch            Relatório Semanal
                          Dashboard      ──────► % recursos conformes
                                                 top 10 não-conformes
                                                 custo não-alocado $
```

---

## 11. Armadilhas (Pitfalls)

[FATO] **Budget Actions não revertem automaticamente**: ao aplicar um SCP ou IAM policy, a ação permanece ativa mesmo após o custo cair. É necessário reverter manualmente via `execute-budget-action` com `REVERSE_BUDGET_ACTION`.

[FATO] **SCPs só podem ser aplicados pela management account**: se o budget está em uma conta-membro, a Budget Action de tipo `APPLY_SCP_POLICY` não funcionará. Só a management account pode criar/anexar SCPs.

[FATO] **Tag Policies não bloqueiam, apenas reportam**: Tag Policies (Organizations) geram relatório de conformidade mas **não impedem** a criação de recursos com valores incorretos. Para bloquear, combine com SCP.

[FATO] **Tags de OU/Account no Organizations não funcionam no Cost Explorer**: tags aplicadas a OUs ou contas do Organizations não são usáveis para filtragem no Cost Explorer. Use Cost Categories para agregar contas em grupos de custo.

[CONSENSO] **Alta cardinalidade de tags aumenta custo de billing**: cada combinação única de tags ativas gera uma linha no Cost and Usage Report (CUR). Tags como `Version` (muda a cada deploy) ou `InstanceId` (muito específico) podem causar explosão de linhas no CUR. Reserve Cost Allocation Tags para dimensões estáveis (project, team, environment).

[FATO] **Budget FORECASTED pode não disparar**: a previsão só é calculada após dados suficientes do mês corrente. No início do mês, o modelo tem pouca base histórica e pode subestimar a previsão.

[INCERTO] **Limite de 500 Cost Allocation Tags ativadas**: há um limite documentado de 500 tags ativadas por conta de gerenciamento na Organization (a verificar na sua organização — pode ter sido atualizado).

---

## Exercício de Reflexão

Uma empresa tem 200 contas AWS em uma Organization com um total de $500K/mês de gasto. O CFO quer implementar chargeback para 8 business units diferentes, mas há três problemas: (1) apenas 60% dos recursos têm tags `project` e `team`; (2) RI e Savings Plans foram comprados na management account sem separação por BU; (3) recursos de infraestrutura compartilhada (VPC, Route53, IAM) não têm como ser tagados por BU.

**Proponha uma estratégia completa de cost allocation**, respondendo:

1. Qual dos três modelos (account-based, team-based, tag-based) — ou combinação — você adotaria? Por quê?
2. Como lidar com os 40% de recursos sem tags? Qual sequência de ações para remediação sem interromper produção?
3. Como alocar o custo de RI/SP que foi comprado centralmente entre as 8 BUs?
4. Como tratar o custo de infraestrutura compartilhada (VPC, DNS, NAT Gateway) — o que o whitepaper sugere?
5. Como usar Budget Actions para criar guardrails automáticos **sem risco** de desligar produção?

---

## Referências

- [FATO] [Configuring budget actions](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-controls.html) — docs.aws.amazon.com
- [FATO] [Configuring a budget action](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-action-configure.html) — docs.aws.amazon.com
- [FATO] [Building a cost allocation strategy](https://docs.aws.amazon.com/whitepapers/latest/tagging-best-practices/building-a-cost-allocation-strategy.html) — docs.aws.amazon.com (whitepaper)
- [FATO] [Implementing and enforcing tagging](https://docs.aws.amazon.com/whitepapers/latest/tagging-best-practices/implementing-and-enforcing-tagging.html) — docs.aws.amazon.com (whitepaper)
- [FATO] [Cost allocation tags](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/cost-alloc-tags.html) — docs.aws.amazon.com
- [FATO] [Tags for cost allocation and financial management](https://docs.aws.amazon.com/whitepapers/latest/tagging-best-practices/tags-for-cost-allocation-and-financial-management.html) — docs.aws.amazon.com (whitepaper)
