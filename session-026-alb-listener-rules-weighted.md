# Sessão 026 — ALB: listener rules complexas, weighted routing e fixed responses

**Duração estimada:** 60 minutos
**Pré-requisitos:** session-014-ecs-services-discovery-alb

---

## Objetivo

Ao final, você conseguirá criar listener rules com condições múltiplas (path + header +
query string), configurar weighted target groups para canary releases no nível do ALB
(sem CodeDeploy), e adicionar fixed response para health checks externos que não devem
chegar ao backend.

---

## Contexto

[FATO] O Application Load Balancer (ALB) opera na camada 7 (HTTP/HTTPS) e roteia
requisições para target groups com base em regras definidas no listener. Uma rule é
composta de: prioridade numérica, zero ou mais condições, e exatamente uma ação de
roteamento (forward, redirect ou fixed-response).

[CONSENSO] A expressividade das listener rules do ALB é suficiente para implementar
patterns sofisticados que em outras plataformas exigiriam um proxy reverso separado:
canary deployments por peso de tráfego, respostas sintéticas para probes externos,
roteamento por versão de API via header, e bloqueio de paths sem tocar no backend.
Entender esse mecanismo reduz a necessidade de camadas adicionais (Nginx, Envoy) para
casos comuns.

---

## Conceitos principais

### 1. Anatomia de uma listener rule

Toda regra no ALB tem três partes:

```
┌─────────────────────────────────────────────────────────────┐
│                     LISTENER (porta 443)                    │
├────────┬─────────────────────────────┬──────────────────────┤
│Priorid.│        Condições            │       Ação           │
├────────┼─────────────────────────────┼──────────────────────┤
│   10   │ path=/api/v2/*              │ → forward TG-v2      │
│        │ AND header[X-Beta]=true     │                      │
├────────┼─────────────────────────────┼──────────────────────┤
│   20   │ path=/health                │ fixed-response 200   │
├────────┼─────────────────────────────┼──────────────────────┤
│   30   │ path=/api/*                 │ → forward TG-v1 (90%)│
│        │                             │             TG-v2(10%)│
├────────┼─────────────────────────────┼──────────────────────┤
│ 65535  │ (default — sem condições)   │ → forward TG-v1      │
└────────┴─────────────────────────────┴──────────────────────┘
```

**Prioridade:** valor inteiro de 1 a 50.000. Regras avaliadas em ordem crescente. A
primeira regra cujas condições são satisfeitas executa sua ação e a avaliação para. A
regra default (prioridade 65535) não tem condições e é criada automaticamente ao criar
o listener — ela não pode ser removida, apenas ter sua ação alterada.

**Semântica AND/OR:** múltiplas condições dentro de uma regra são combinadas com AND
(todas devem ser verdadeiras). Dentro de uma única condição, múltiplos valores são
combinados com OR (qualquer valor basta).

```
Regra: path=/api/v2/*  AND  http-header[X-Env]=staging
                            ↑
                     Ambas devem ser TRUE

Condição: path-pattern Values=["/api/v1/*", "/api/v2/*"]
                                 ↑
                        Qualquer um basta (OR)
```

[FATO] Limites por regra (conforme documentação oficial):
- Máximo de 5 *match evaluations* (comparações) por regra
- Máximo de 3 strings de comparação por condição
- Máximo de 5 wildcards (`*` e `?`) por regra
- Exatamente zero ou um de: `host-header`, `http-request-method`, `path-pattern`, `source-ip`
- Zero ou mais de: `http-header`, `query-string`

---

### 2. Tipos de condição em detalhe

ALB suporta seis tipos de condição. A tabela abaixo resume o comportamento e
particularidades de cada um:

| Tipo              | Case-sensitive | Wildcard | Regex | Múltiplos por regra |
|-------------------|:--------------:|:--------:|:-----:|:-------------------:|
| `host-header`     | Não            | Sim      | Sim   | Não                 |
| `http-header`     | Não (valor)    | Sim      | Sim   | Sim                 |
| `http-request-method` | Sim        | Não      | Não   | Não                 |
| `path-pattern`    | **Sim**        | Sim      | Sim   | Não                 |
| `query-string`    | Não            | Sim      | Não   | Sim                 |
| `source-ip`       | N/A (CIDR)     | Não      | Não   | Não                 |

**path-pattern** é case-sensitive — `/API/v1/*` e `/api/v1/*` são padrões distintos.
Isso é uma armadilha frequente em migrações de APIs que antes viviam atrás de um Nginx
case-insensitive.

**http-header** usa o nome do header case-insensitivo, mas o valor case-insensitivo
também. Você pode criar múltiplas condições `http-header` na mesma regra para exigir
mais de um header simultaneamente (via AND).

**query-string** aceita pares chave/valor ou só valor. A chave `null` significa "qualquer
chave com este valor":

```json
// Condição: satisfeita se query tiver version=v2 OU qualquer key=beta
{
  "Field": "query-string",
  "QueryStringConfig": {
    "Values": [
      { "Key": "version", "Value": "v2" },
      { "Value": "beta" }
    ]
  }
}
```

**source-ip** avalia o IP da conexão TCP, não o `X-Forwarded-For`. Se o cliente estiver
atrás de um proxy, o IP avaliado é o do proxy. Para avaliar o IP original do cliente,
use uma condição `http-header` no campo `X-Forwarded-For`.

---

### 3. Weighted target groups para canary releases

O ALB permite que uma única ação `forward` distribua tráfego entre múltiplos target
groups com pesos independentes. Cada peso é um inteiro de 0 a 999. A proporção de
tráfego para cada TG é `peso_TG / soma_total_pesos`.

```
Exemplo: TG-v1 (peso=90) + TG-v2 (peso=10)
  → TG-v1 recebe 90/(90+10) = 90% das requisições
  → TG-v2 recebe 10/(90+10) = 10% das requisições

Canary progressivo (atualizando pesos via API):
  Fase 1: v1=95, v2=5    →  5% no canary
  Fase 2: v1=80, v2=20   → 20% no canary
  Fase 3: v1=50, v2=50   → 50% no canary
  Fase 4: v1=0,  v2=100  → rollout completo
  (TG-v1 com peso=0: regra existe mas TG não recebe tráfego)
```

[FATO] O balanceamento entre TGs é feito no nível da rule, não por sessão — cada
requisição é distribuída de forma independente baseada nos pesos. Para garantir que um
usuário específico seja sempre enviado ao mesmo TG durante uma sessão, habilite
**target group stickiness** na rule. O ALB então emite um cookie `AWSALBTG` que
codifica o TG selecionado; requisições subsequentes com esse cookie ignoram a
distribuição por peso e vão direto ao TG indicado.

[FATO] Comportamento com TG vazio ou unhealthy: se um TG configurado com peso > 0
não tiver targets saudáveis, o ALB **não** faz failover automático para o outro TG.
As requisições roteadas para o TG sem targets retornam 502 ou 503. Isso é diferente do
comportamento de um único TG, onde o ALB simplesmente não tem para onde rotear. Em
ambientes canary, monitore o TG de canary ativamente antes de aumentar o peso.

```
Diagrama de canary com sticky sessions:

  Request 1 (novo usuário)
       │
       ▼
  ALB Rule: TG-v1(80) + TG-v2(20)
       │
       ├── 80% → TG-v1 → [Set-Cookie: AWSALBTG=<hash-v1>]
       └── 20% → TG-v2 → [Set-Cookie: AWSALBTG=<hash-v2>]

  Request 2 (mesmo usuário, com cookie AWSALBTG=<hash-v1>)
       │
       ▼
  ALB Rule: detecta cookie → ignora pesos → vai direto para TG-v1
```

---

### 4. Fixed-response action

A ação `fixed-response` faz o ALB retornar uma resposta HTTP sintética sem encaminhar
a requisição para nenhum target. O payload é configurado diretamente na rule:

```json
{
  "Type": "fixed-response",
  "FixedResponseConfig": {
    "StatusCode": "200",
    "ContentType": "text/plain",
    "MessageBody": "OK"
  }
}
```

[FATO] `StatusCode` pode ser qualquer código 2XX, 4XX ou 5XX. `ContentType` aceita:
`text/plain`, `text/css`, `text/html`, `application/javascript`, `application/json`.
`MessageBody` é opcional; máximo de 1024 bytes.

Casos de uso principais:

1. **Health check de uptime tools**: serviços como Pingdom, Datadog Synthetics ou o
   próprio Route 53 Health Checks podem verificar `/health` sem que essa requisição
   chegue ao backend. Reduz carga e elimina a necessidade de um endpoint de health check
   nos servidores de aplicação.

2. **Bloqueio de paths**: retornar 403 para rotas legacy que não devem mais existir, sem
   manter código ou lógica no backend.

3. **Manutenção planejada**: durante um deploy disruptivo, uma rule temporária de alta
   prioridade captura todo o tráfego e retorna 503 com uma mensagem customizada.

4. **CORS preflight em camadas sem backend**: retornar 204 para `OPTIONS` em paths
   estáticos sem encaminhar para containers.

[FATO] Quando uma ação `fixed-response` é executada, ela é registrada nos access logs
do ALB e contabilizada na métrica `HTTP_Fixed_Response_Count` no CloudWatch.

---

## Exemplo prático

**Cenário:** um serviço de e-commerce com API versionada precisa:
1. Rotear `/api/v2/*` apenas quando header `X-Api-Version: 2` estiver presente → TG-v2
2. Implementar canary de 10% para `/api/*` geral, direcionando 10% para TG-v2
3. Responder `200 OK` para `/health` sem chegar ao backend
4. Enviar tudo o mais para TG-v1 (default)

### CDK Python

```python
from aws_cdk import Stack, aws_elasticloadbalancingv2 as elbv2
import aws_cdk.aws_elasticloadbalancingv2 as elbv2
from constructs import Construct


class AlbRulesStack(Stack):
    def __init__(self, scope: Construct, id: str, **kwargs):
        super().__init__(scope, id, **kwargs)

        # ALB, VPC, TGs criados previamente (referenciados por ARN ou import)
        alb = elbv2.ApplicationLoadBalancer.from_lookup(
            self, "ALB",
            load_balancer_arn="arn:aws:elasticloadbalancing:...",
        )

        tg_v1 = elbv2.ApplicationTargetGroup.from_target_group_attributes(
            self, "TG-v1",
            target_group_arn="arn:aws:elasticloadbalancing:...:targetgroup/api-v1/...",
        )
        tg_v2 = elbv2.ApplicationTargetGroup.from_target_group_attributes(
            self, "TG-v2",
            target_group_arn="arn:aws:elasticloadbalancing:...:targetgroup/api-v2/...",
        )

        listener = elbv2.ApplicationListener.from_lookup(
            self, "Listener",
            load_balancer_arn=alb.load_balancer_arn,
            listener_port=443,
        )

        # Regra 1 (prioridade 10): /api/v2/* + header X-Api-Version: 2 → TG-v2 (100%)
        listener.add_action(
            "RouteV2WithHeader",
            priority=10,
            conditions=[
                elbv2.ListenerCondition.path_patterns(["/api/v2/*"]),
                elbv2.ListenerCondition.http_header(
                    "X-Api-Version", ["2"]
                ),
            ],
            action=elbv2.ListenerAction.forward([tg_v2]),
        )

        # Regra 2 (prioridade 20): /health → fixed-response 200
        listener.add_action(
            "HealthCheck",
            priority=20,
            conditions=[
                elbv2.ListenerCondition.path_patterns(["/health"]),
            ],
            action=elbv2.ListenerAction.fixed_response(
                status_code=200,
                content_type=elbv2.ContentType.TEXT_PLAIN,
                message_body="OK",
            ),
        )

        # Regra 3 (prioridade 30): /api/* → canary 10% para TG-v2, 90% para TG-v1
        # Com target group stickiness de 1 hora
        listener.add_action(
            "CanaryApi",
            priority=30,
            conditions=[
                elbv2.ListenerCondition.path_patterns(["/api/*"]),
            ],
            action=elbv2.ListenerAction.weighted_forward(
                target_groups=[
                    elbv2.WeightedTargetGroup(target_group=tg_v1, weight=90),
                    elbv2.WeightedTargetGroup(target_group=tg_v2, weight=10),
                ],
                stickiness_duration=Duration.hours(1),
            ),
        )

        # Regra default (criada ao criar o listener): → TG-v1
        # Adicionada via listener default_action no momento da criação:
        #   elbv2.ApplicationListener(..., default_action=ListenerAction.forward([tg_v1]))
```

### CLI — criando e modificando pesos em runtime

```bash
# Criar rule de canary (10% v2)
aws elbv2 create-rule \
  --listener-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:listener/app/my-alb/... \
  --priority 30 \
  --conditions '[{"Field":"path-pattern","PathPatternConfig":{"Values":["/api/*"]}}]' \
  --actions '[{
    "Type": "forward",
    "ForwardConfig": {
      "TargetGroups": [
        {"TargetGroupArn": "arn:...targetgroup/api-v1/...", "Weight": 90},
        {"TargetGroupArn": "arn:...targetgroup/api-v2/...", "Weight": 10}
      ],
      "TargetGroupStickinessConfig": {"Enabled": true, "DurationSeconds": 3600}
    }
  }]'

# Escalar canary para 50%: modificar pesos sem recriar a regra
aws elbv2 modify-rule \
  --rule-arn arn:aws:elasticloadbalancing:...:rule/... \
  --actions '[{
    "Type": "forward",
    "ForwardConfig": {
      "TargetGroups": [
        {"TargetGroupArn": "arn:...targetgroup/api-v1/...", "Weight": 50},
        {"TargetGroupArn": "arn:...targetgroup/api-v2/...", "Weight": 50}
      ],
      "TargetGroupStickinessConfig": {"Enabled": true, "DurationSeconds": 3600}
    }
  }]'

# Rollout completo: peso 0 em v1
aws elbv2 modify-rule \
  --rule-arn arn:aws:elasticloadbalancing:...:rule/... \
  --actions '[{
    "Type": "forward",
    "ForwardConfig": {
      "TargetGroups": [
        {"TargetGroupArn": "arn:...targetgroup/api-v1/...", "Weight": 0},
        {"TargetGroupArn": "arn:...targetgroup/api-v2/...", "Weight": 100}
      ]
    }
  }]'

# Criar fixed-response para /health
aws elbv2 create-rule \
  --listener-arn arn:aws:elasticloadbalancing:...:listener/... \
  --priority 20 \
  --conditions '[{"Field":"path-pattern","PathPatternConfig":{"Values":["/health"]}}]' \
  --actions '[{
    "Type": "fixed-response",
    "FixedResponseConfig": {
      "StatusCode": "200",
      "ContentType": "text/plain",
      "MessageBody": "OK"
    }
  }]'

# Listar todas as rules do listener
aws elbv2 describe-rules \
  --listener-arn arn:aws:elasticloadbalancing:...:listener/... \
  --query 'Rules[*].{Priority:Priority,Conditions:Conditions[*].Field,Action:Actions[0].Type}' \
  --output table
```

---

## Armadilhas comuns

### 1. Múltiplas condições como OR em vez de AND
**O erro:** um desenvolvedor quer rotear requisições que TÊM `/api/v2/*` E header
`X-Beta: true`. Ele cria duas rules separadas, uma por condição, ambas apontando para
TG-v2. Resultado: qualquer requisição para `/api/v2/*` (sem o header) vai para TG-v2,
e qualquer requisição com `X-Beta: true` (qualquer path) também vai.

**Por que acontece:** o desenvolvedor pensa em cada condição como um filtro adicional.
Na realidade, rules separadas são avaliadas independentemente — se qualquer uma delas
fizer match, ela executa.

**Como evitar:** condições que devem ser combinadas com AND precisam estar na mesma
rule. Uma rule com `path-pattern=/api/v2/*` E `http-header[X-Beta]=true` só faz match
se ambas forem verdadeiras simultaneamente.

---

### 2. TG com peso 0 vs remover o TG da rule
**O erro:** para "pausar" o canary, o time muda o peso do TG-v2 para 0. Depois de
semanas, sobe novamente para 10% sem testar o estado do TG.

**Por que acontece:** peso 0 não significa que o TG é removido do balanceamento — ele
ainda existe na rule e o ALB monitorará suas targets. Se o TG-v2 tiver apenas targets
deregistrados e a stickiness estiver ativa, usuários que já receberam o cookie
`AWSALBTG` para v2 continuarão sendo roteados para ele, resultando em 502.

**Como evitar:** ao fazer rollback completo, remova o TG da rule e use `forward`
simples para um único TG. Use peso 0 apenas como etapa transitória breve. Sempre
verifique saúde do TG antes de aumentar peso novamente.

---

### 3. path-pattern é case-sensitive
**O erro:** a aplicação aceita `/API/users` e `/api/users` indiscriminadamente (padrão
em alguns frameworks), mas a rule usa `/api/*`. Requisições para `/API/*` vão parar na
regra default, frequentemente resultando em comportamento inesperado.

**Por que acontece:** path-pattern no ALB é case-sensitive, conforme documentação.
Muitos desenvolvedores assumem comportamento de HTTP (onde o path pode ou não ser
case-sensitive dependendo do servidor).

**Como evitar:** normalize URLs na aplicação ou no CDN antes de chegarem ao ALB.
Se necessário, crie rules duplicadas para variantes de case. Alternativamente, use
regex matching (`RegexValues`) com flag de case-insensitive: `^(?i)/api/.*$`.

---

## Exercício de reflexão

Você é responsável por uma API REST com dois endpoints críticos: `/api/payments/*` e
`/api/catalog/*`. O time decidiu migrar pagamentos para uma nova versão do serviço
(v2) de forma gradual, mas quer que a migração seja controlável por cliente: clientes
enterprise (identificados pelo header `X-Client-Tier: enterprise`) devem ir direto para
v2 logo de início, enquanto o restante passa pelo canary normal de 5% para v2.

Além disso, o monitoramento externo da Datadog verifica `/health/ping` a cada 30
segundos — você quer que essa requisição retorne 200 sem chegar ao backend.

Descreva a estrutura completa das listener rules necessárias: quais prioridades atribuir,
quais condições combinar em cada rule, quais ações configurar, e que comportamentos
emergiriam se a order de prioridades fosse trocada entre algumas regras. Considere
também o que acontece se o TG-v2 de payments ficar unhealthy durante o canary.

---

## Recursos para aprofundar

**[Listener rules for your Application Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/listener-rules.html)**
Documentação principal: estrutura de rules, como criar/modificar/reordenar. Seção
de interesse: "Rule evaluation order" e "Default rules".

**[Condition types for listener rules](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/rule-condition-types.html)**
Referência completa de cada tipo de condição com exemplos JSON de CLI. Inclui
exemplos de regex matching para `host-header` e `path-pattern`.

**[Action types for listener rules](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/rule-action-types.html)**
Cobre forward (simples e weighted), fixed-response e redirect com todos os campos.
Seção "Sticky sessions and weighted target groups" explica o cookie `AWSALBTG`.

---

