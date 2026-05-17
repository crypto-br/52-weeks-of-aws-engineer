# Sessão — ECS: Deploy strategies — rolling update e blue/green com CodeDeploy

**Duração estimada:** 60 minutos
**Pré-requisitos:** session-016-ecs-capacity-providers-autoscaling, session-010-cdk-pipelines-stages-shellsteps

---

## Objetivo

Ao final, você conseguirá configurar um ECS Service com deployment type `CODE_DEPLOY`, escrever o AppSpec para blue/green deployment, implementar hooks de teste após o traffic shift, e configurar o rollback automático com base em CloudWatch Alarms.

---

## Contexto

[FATO] O ECS suporta três tipos de deployment: **rolling update** (padrão, gerenciado diretamente pelo ECS), **blue/green com CodeDeploy** (`CODE_DEPLOY`), e **external** (para integrações com ferramentas de terceiros). Rolling update é adequado para a maioria dos serviços; blue/green é necessário quando você precisa de verificação explícita antes de redirecionar tráfego de produção, de zero-downtime com possibilidade de rollback instantâneo, ou de testes em ambiente de produção antes do traffic shift completo.

[FATO] O modelo blue/green do CodeDeploy para ECS é fundamentalmente diferente do rolling update: em vez de substituir tasks gradualmente *dentro do mesmo target group*, o CodeDeploy cria um **replacement task set** completamente separado (green), faz testes nele, e então muda o ALB de apontar para o task set original (blue) para o green. Após o período de estabilização, o blue é destruído. Rollback significa mudar o ALB de volta para o blue — operação de segundos, sem precisar criar novas tasks.

[CONSENSO] Blue/green tem custo operacional maior que rolling update: requer mais componentes (dois target groups, listener de teste, deployment group CodeDeploy, IAM role adicional), duplica a capacidade de tasks durante o deployment, e adiciona complexidade de configuração. É justificado para serviços de produção críticos onde o custo de um deploy ruim (downtime, rollback lento) é alto. Para serviços internos ou de menor criticidade, rolling update com circuit breaker é geralmente suficiente.

---

## Conceitos principais

### 1. Arquitetura do blue/green no ECS

[FATO] O blue/green deployment para ECS requer os seguintes componentes adicionais em relação a um service rolling update:

```
┌─────────────────────────────────────────────────────────────────┐
│  ALB                                                            │
│                                                                 │
│  Listener :443 (produção)          Listener :8080 (teste)      │
│       │                                   │                     │
│       ▼                                   ▼                     │
│  Target Group BLUE (ativo)    Target Group GREEN (novo)        │
│  [tasks versão anterior]      [tasks nova versão]              │
└─────────────────────────────────────────────────────────────────┘

Antes do traffic shift:
  :443 → TG-BLUE (100% tráfego de produção)
  :8080 → TG-GREEN (tráfego de teste apenas)

Durante traffic shift (canary):
  :443 → TG-BLUE (90%) + TG-GREEN (10%)
  :8080 → TG-GREEN (100% do tráfego de teste)

Após traffic shift completo:
  :443 → TG-GREEN (100%)
  :8080 → N/A (pode ser destruído)

Após estabilização (termination wait):
  TG-BLUE e suas tasks são destruídos
```

[FATO] Os componentes necessários:

1. **Dois ALB Target Groups**: blue (ativo) e green (para o novo task set).
2. **Dois ALB Listeners**: porta de produção (ex: 443) e porta de teste (ex: 8080). O listener de teste é opcional mas recomendado para hooks de validação.
3. **CodeDeploy Application + Deployment Group**: com `computePlatform: ECS`.
4. **IAM Role para CodeDeploy**: `AWSCodeDeployRoleForECS` managed policy.
5. **ECS Service com `deploymentController: CODE_DEPLOY`**.

---

### 2. O arquivo AppSpec

[FATO] O AppSpec é um arquivo YAML ou JSON que descreve *o quê* deployar e *como* orquestrar o deployment. Para ECS, ele tem dois blocos principais:

```yaml
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        # ARN ou nome da task definition a deployar (nova versão)
        TaskDefinition: "arn:aws:ecs:us-east-1:123456789012:task-definition/api-service:42"
        LoadBalancerInfo:
          ContainerName: "api"        # container que recebe tráfego (portMappings)
          ContainerPort: 3000
        # Opcional: override de configurações de rede do service
        # PlatformVersion: "LATEST"
        # NetworkConfiguration:
        #   AwsvpcConfiguration:
        #     Subnets: ["subnet-abc", "subnet-def"]
        #     SecurityGroups: ["sg-xyz"]
        #     AssignPublicIp: "DISABLED"
        # Opcional: override de Capacity Provider strategy
        # CapacityProviderStrategy:
        #   - CapacityProvider: "FARGATE"
        #     Base: 2
        #     Weight: 1

Hooks:
  # Hook ANTES do ALB enviar qualquer tráfego de produção para o green
  - BeforeAllowTraffic:
      - location: "arn:aws:lambda:us-east-1:123456789012:function:PreTrafficHook"
        timeout: 300        # segundos — máx 3600

  # Hook APÓS o ALB ter direcionado tráfego de produção para o green
  - AfterAllowTraffic:
      - location: "arn:aws:lambda:us-east-1:123456789012:function:PostTrafficHook"
        timeout: 300
```

[FATO] Ciclo completo de hooks disponíveis para ECS blue/green (em ordem de execução):

```
1. BeforeInstall
   → antes de criar o replacement task set (green)
   → raramente usado para ECS

2. AfterInstall
   → após o task set green ser criado e tasks estarem RUNNING
   → antes de qualquer tráfego

3. AfterAllowTestTraffic
   → após o listener de TESTE (:8080) apontar para o green
   → antes do traffic shift de produção
   → LOCAL IDEAL para smoke tests automatizados

4. BeforeAllowTraffic
   → após os smoke tests, antes do traffic shift de produção
   → validação final

5. AfterAllowTraffic
   → após o traffic shift de produção estar completo
   → monitoramento de métricas, alertas de sucesso
```

[FATO] Um hook é uma Lambda que **deve** chamar `codedeploy:PutLifecycleEventHookExecutionStatus` com `status: Succeeded` ou `status: Failed`. Se a Lambda não chamar essa API dentro do `timeout`, o CodeDeploy considera o hook como falhou e inicia rollback.

```python
# Estrutura de uma Lambda de hook
import boto3

codedeploy = boto3.client('codedeploy')

def handler(event, context):
    deployment_id = event['DeploymentId']
    hook_execution_id = event['LifecycleEventHookExecutionId']
    
    try:
        # Executa os testes aqui
        run_smoke_tests()
        
        # Sinaliza sucesso para o CodeDeploy continuar
        codedeploy.put_lifecycle_event_hook_execution_status(
            deploymentId=deployment_id,
            lifecycleEventHookExecutionId=hook_execution_id,
            status='Succeeded'
        )
    except Exception as e:
        print(f"Hook falhou: {e}")
        codedeploy.put_lifecycle_event_hook_execution_status(
            deploymentId=deployment_id,
            lifecycleEventHookExecutionId=hook_execution_id,
            status='Failed'
        )
        raise
```

---

### 3. Estratégias de traffic shifting

[FATO] O CodeDeploy oferece configurações predefinidas e customizáveis de traffic shifting para ECS:

```
Configurações predefinidas (ECS):
  CodeDeployDefault.ECSAllAtOnce
    → 100% do tráfego imediatamente para o green
    → Sem período de validação com tráfego parcial
    → Rollback manual ou por alarm

  CodeDeployDefault.ECSCanary10Percent5Minutes
    → 10% para green por 5 minutos, depois 100%
    → Permite monitorar erros com tráfego real antes do shift completo

  CodeDeployDefault.ECSCanary10Percent15Minutes
    → 10% por 15 minutos, depois 100%

  CodeDeployDefault.ECSLinear10PercentEvery1Minutes
    → +10% a cada 1 minuto até 100% (10 incrementos)
    → Mais gradual, maior janela de detecção de problemas

  CodeDeployDefault.ECSLinear10PercentEvery3Minutes
    → +10% a cada 3 minutos até 100% (30 minutos total)
```

[FATO] Configuração customizada de traffic shifting (via CDK ou CLI):

```typescript
// CDK — deployment config customizado
import * as codedeploy from 'aws-cdk-lib/aws-codedeploy';

const deploymentConfig = new codedeploy.EcsDeploymentConfig(this, 'Config', {
  trafficRouting: codedeploy.TrafficRouting.timeBasedCanary({
    interval: Duration.minutes(5),   // aguarda 5 min entre incrementos
    percentage: 20,                   // 20% no primeiro incremento
    // → 20% por 5 min, depois 100%
  }),
  // Alternativa: linear
  // trafficRouting: codedeploy.TrafficRouting.timeBasedLinear({
  //   interval: Duration.minutes(2),
  //   percentage: 10,  // +10% a cada 2 minutos
  // }),
});
```

---

### 4. Rollback automático por CloudWatch Alarms

[FATO] O CodeDeploy pode monitorar CloudWatch Alarms durante o deployment e disparar rollback automático se algum alarme entrar no estado `ALARM`. Os alarmes são associados ao deployment group, não ao AppSpec.

```
Deployment group
│
├── autoRollback:
│   ├── onDeploymentFailure: true     (hook retornou Failed)
│   ├── onAlarmThreshold: true        (CW Alarm disparou)
│   └── alarms:
│       ├── "HighErrorRate"           (ex: 5xx > 5% por 2 min)
│       └── "HighLatency"             (ex: P99 > 2s por 2 min)
│
└── terminationWait: 60 minutos       (tempo antes de destruir o blue)
```

[FATO] O `terminationWait` é o período após o traffic shift completo durante o qual o CodeDeploy aguarda antes de destruir o task set blue. Durante esse período, o blue ainda existe e pode ser usado para rollback instantâneo. O padrão é 0 minutos (destrói imediatamente). Em produção, 30-60 minutos é recomendado para ter janela de rollback pós-deploy.

---

## Exemplo prático

**Cenário completo:** Um `api-service` crítico com:
- Blue/green via CodeDeploy com canary 10% por 5 minutos.
- Hook `AfterAllowTestTraffic` que valida o health check do green via listener de teste.
- Rollback automático se `HighErrorRate` alarm disparar.
- Rollback automático se o hook falhar.

### CDK completo

```typescript
import { Stack, StackProps, Duration } from 'aws-cdk-lib';
import * as ecs from 'aws-cdk-lib/aws-ecs';
import * as codedeploy from 'aws-cdk-lib/aws-codedeploy';
import * as elbv2 from 'aws-cdk-lib/aws-elasticloadbalancingv2';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as cloudwatch from 'aws-cdk-lib/aws-cloudwatch';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as iam from 'aws-cdk-lib/aws-iam';
import { Construct } from 'constructs';

export class BlueGreenStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    const vpc = ec2.Vpc.fromLookup(this, 'Vpc', { vpcName: 'prod-vpc' });
    const cluster = ecs.Cluster.fromClusterAttributes(this, 'Cluster', {
      clusterName: 'prod-cluster', vpc, securityGroups: [],
    });

    // ─── ALB com dois target groups e dois listeners ───────────────────────

    const alb = new elbv2.ApplicationLoadBalancer(this, 'ALB', {
      vpc, internetFacing: true,
    });

    // Target Group BLUE (ativo, começa com as tasks da versão atual)
    const blueTG = new elbv2.ApplicationTargetGroup(this, 'BlueTG', {
      vpc,
      protocol: elbv2.ApplicationProtocol.HTTP,
      port: 3000,
      targetType: elbv2.TargetType.IP,
      healthCheck: {
        path: '/health',
        interval: Duration.seconds(15),
        healthyThresholdCount: 2,
      },
      deregistrationDelay: Duration.seconds(30),
    });

    // Target Group GREEN (vazio, será preenchido pelo CodeDeploy no deploy)
    const greenTG = new elbv2.ApplicationTargetGroup(this, 'GreenTG', {
      vpc,
      protocol: elbv2.ApplicationProtocol.HTTP,
      port: 3000,
      targetType: elbv2.TargetType.IP,
      healthCheck: {
        path: '/health',
        interval: Duration.seconds(15),
        healthyThresholdCount: 2,
      },
      deregistrationDelay: Duration.seconds(30),
    });

    // Listener de produção (:443) → começa apontando para BLUE
    const prodListener = alb.addListener('ProdListener', {
      port: 443,
      defaultTargetGroups: [blueTG],
      open: true,
    });

    // Listener de teste (:8080) → usado pelos hooks para validação
    const testListener = alb.addListener('TestListener', {
      port: 8080,
      defaultTargetGroups: [greenTG],
      open: false,  // restrito (acesso interno apenas)
    });

    // ─── ECS Service com deploymentController: CODE_DEPLOY ────────────────

    const taskDef = new ecs.FargateTaskDefinition(this, 'TaskDef', {
      cpu: 512,
      memoryLimitMiB: 1024,
    });
    taskDef.addContainer('api', {
      image: ecs.ContainerImage.fromRegistry('my-org/api:v1'),
      portMappings: [{ containerPort: 3000 }],
      logging: ecs.LogDrivers.awsLogs({ streamPrefix: 'api' }),
    });

    const service = new ecs.FargateService(this, 'Service', {
      cluster,
      taskDefinition: taskDef,
      desiredCount: 3,
      deploymentController: {
        type: ecs.DeploymentControllerType.CODE_DEPLOY,  // chave para blue/green
      },
      // Registra no target group BLUE (green é gerenciado pelo CodeDeploy)
      loadBalancers: [{
        targetGroup: blueTG,
        containerName: 'api',
        containerPort: 3000,
      }],
      vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS },
    });

    // ─── Lambda de hook (AfterAllowTestTraffic) ────────────────────────────

    const hookFn = new lambda.Function(this, 'TestHook', {
      runtime: lambda.Runtime.PYTHON_3_12,
      handler: 'index.handler',
      timeout: Duration.seconds(300),
      code: lambda.Code.fromInline(`
import boto3
import urllib.request
import json
import os

codedeploy = boto3.client('codedeploy')
ALB_TEST_URL = os.environ.get('ALB_TEST_URL', '')

def handler(event, context):
    deployment_id = event['DeploymentId']
    hook_id = event['LifecycleEventHookExecutionId']
    
    try:
        # Chama o health endpoint via listener de teste (:8080)
        req = urllib.request.urlopen(f'{ALB_TEST_URL}/health', timeout=10)
        body = json.loads(req.read())
        
        if body.get('status') != 'ok':
            raise ValueError(f"Health check falhou: {body}")
        
        print("Smoke test passou. Continuando deployment.")
        codedeploy.put_lifecycle_event_hook_execution_status(
            deploymentId=deployment_id,
            lifecycleEventHookExecutionId=hook_id,
            status='Succeeded'
        )
    except Exception as e:
        print(f"Smoke test FALHOU: {e}")
        codedeploy.put_lifecycle_event_hook_execution_status(
            deploymentId=deployment_id,
            lifecycleEventHookExecutionId=hook_id,
            status='Failed'
        )
`),
      environment: {
        ALB_TEST_URL: `http://${alb.loadBalancerDnsName}:8080`,
      },
    });

    // A Lambda precisa chamar codedeploy:PutLifecycleEventHookExecutionStatus
    hookFn.addToRolePolicy(new iam.PolicyStatement({
      actions: ['codedeploy:PutLifecycleEventHookExecutionStatus'],
      resources: ['*'],
    }));

    // ─── CloudWatch Alarms para rollback automático ────────────────────────

    const errorRateAlarm = new cloudwatch.Alarm(this, 'HighErrorRate', {
      metric: new cloudwatch.MathExpression({
        expression: '(errors / requests) * 100',
        usingMetrics: {
          errors: new cloudwatch.Metric({
            namespace: 'AWS/ApplicationELB',
            metricName: 'HTTPCode_Target_5XX_Count',
            dimensionsMap: {
              LoadBalancer: alb.loadBalancerFullName,
              TargetGroup: blueTG.targetGroupFullName,
            },
            statistic: 'Sum',
          }),
          requests: new cloudwatch.Metric({
            namespace: 'AWS/ApplicationELB',
            metricName: 'RequestCount',
            dimensionsMap: {
              LoadBalancer: alb.loadBalancerFullName,
              TargetGroup: blueTG.targetGroupFullName,
            },
            statistic: 'Sum',
          }),
        },
        period: Duration.minutes(2),
      }),
      threshold: 5,           // rollback se 5xx > 5%
      evaluationPeriods: 2,
      comparisonOperator: cloudwatch.ComparisonOperator.GREATER_THAN_THRESHOLD,
      alarmName: 'api-HighErrorRate',
    });

    // ─── CodeDeploy Application e Deployment Group ─────────────────────────

    const app = new codedeploy.EcsApplication(this, 'App', {
      applicationName: 'api-service',
    });

    const deploymentGroup = new codedeploy.EcsDeploymentGroup(this, 'DG', {
      application: app,
      deploymentGroupName: 'api-service-dg',
      service,
      blueGreenDeploymentConfig: {
        // Listeners e target groups para o blue/green
        listener: prodListener,
        testListener,
        blueTargetGroup: blueTG,
        greenTargetGroup: greenTG,
        // Aguarda 60 min antes de destruir o task set blue
        terminationWaitTime: Duration.minutes(60),
      },
      deploymentConfig: codedeploy.EcsDeploymentConfig.CANARY_10PERCENT_5MINUTES,
      autoRollback: {
        failedDeployment: true,       // rollback se hook falhar
        deploymentInAlarm: true,      // rollback se alarm disparar
        stoppedDeployment: false,
      },
      alarms: [errorRateAlarm],
    });
  }
}
```

### AppSpec gerado (para referência no pipeline)

```yaml
# appspec.yaml — gerado pelo pipeline, substituindo <TASK_DEF_ARN>
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: "<TASK_DEF_ARN>"
        LoadBalancerInfo:
          ContainerName: "api"
          ContainerPort: 3000

Hooks:
  - AfterAllowTestTraffic:
      - location: "arn:aws:lambda:us-east-1:123456789012:function:BlueGreenStack-TestHook"
        timeout: 300
```

### Timeline de um deployment bem-sucedido

```
t=0m    CodeDeploy cria task set green com nova task definition
t=2m    Tasks green passam no health check do TG verde
t=2m    Listener :8080 aponta para TG verde (tráfego de teste disponível)
t=2m    Hook AfterAllowTestTraffic é invocado
t=3m    Hook valida /health via :8080 → Succeeded
t=3m    BeforeAllowTraffic (nenhum hook configurado, prossegue)
t=3m    Traffic shift: :443 passa a 10% green / 90% blue
t=8m    Nenhum alarm disparou no período de observação (5 min)
t=8m    Traffic shift: :443 passa a 100% green / 0% blue
t=8m    Hook AfterAllowTraffic invocado (se configurado)
t=8m    Deployment marcado como Succeeded
t=68m   terminationWaitTime esgotado → task set blue destruído
```

---

## Armadilhas comuns

### Armadilha 1: Hook Lambda sem chamar PutLifecycleEventHookExecutionStatus

**O erro:** Você cria uma Lambda de hook que executa os testes mas retorna normalmente (sem exceção). A Lambda termina com sucesso (exit 200), mas o CodeDeploy não recebe a confirmação via `PutLifecycleEventHookExecutionStatus`. Após o `timeout` configurado (ex: 300 segundos), o CodeDeploy considera o hook como `Failed` e inicia rollback. O deployment é revertido mesmo com todos os testes passando.

**Por que acontece:** O CodeDeploy não usa o exit code da Lambda para determinar sucesso/falha do hook. Ele *exclusivamente* aguarda a chamada explícita à API `PutLifecycleEventHookExecutionStatus`. Retornar da Lambda sem chamar essa API é indistinguível de um timeout para o CodeDeploy.

**Como reconhecer:** No console CodeDeploy, o deployment mostra `Lifecycle event hook timed out` ou `Lifecycle event hook failed`. Os logs da Lambda mostram execução normal sem erro. A Lambda foi invocada, executou, e retornou — mas o deployment reverteu.

**Como evitar:** Use um bloco `try/finally` para garantir que `PutLifecycleEventHookExecutionStatus` seja sempre chamado, mesmo em caso de exceção. Nunca retorne da Lambda sem ter chamado essa API.

---

### Armadilha 2: terminationWait zero — sem janela de rollback pós-deploy

**O erro:** O `terminationWaitTime` está configurado como 0 (padrão). Um deploy bem-sucedido destrói o task set blue imediatamente após o traffic shift. Trinta minutos depois, um problema latente aparece na nova versão (ex: um job batch que roda a cada hora começa a falhar). Para reverter, você precisa criar um novo deployment com a versão anterior — o que leva tempo e pode ter downtime.

**Por que acontece:** Com `terminationWaitTime = 0`, o CodeDeploy destrói o task set blue assim que o traffic shift é concluído. Não há blue para voltar instantaneamente.

**Como evitar:** Configure `terminationWaitTime` de 30-60 minutos para serviços críticos. Durante esse período, um rollback é literalmente uma operação de redirecionamento de ALB — segundos, sem criar novas tasks. O custo é manter o dobro de tasks por esse período, o que é aceitável para serviços importantes. Se quiser economizar, 30 minutos é suficiente para a maioria dos problemas pós-deploy se tornarem visíveis.

---

### Armadilha 3: Alarm configurado no target group errado causa falsos positivos

**O erro:** Você configura o alarm de rollback monitorando métricas do `blueTG` (target group original). Durante o deployment, o CodeDeploy está enviando 10% do tráfego para o `greenTG` e 90% para o `blueTG`. Se o `blueTG` estava com alta taxa de erros *antes* do deployment (o que motivou o deploy da nova versão), o alarm já está ativo e o CodeDeploy imediatamente reverte o deployment recém-iniciado.

**Por que acontece:** O alarm monitora o target group errado. Durante o traffic shift canary, o que você quer monitorar são os erros no target group *novo* (green), não no antigo (blue).

**Como reconhecer:** Deployments que revertêm imediatamente ao iniciar o traffic shift, mesmo quando a nova versão está funcionando corretamente. Os logs do deployment group mostram `Alarm entered ALARM state` como motivo do rollback.

**Como evitar:** Configure os alarms de rollback para monitorar o `greenTG` (ou o ALB como um todo, que inclui ambos). Alternativamente, monitore métricas de negócio (ex: taxa de conversão, latência P99) que refletem a saúde do serviço independente de qual target group está ativo.

---

## Exercício de reflexão

Você está migrando um serviço de pagamentos de rolling update para blue/green. O serviço tem uma característica: a nova versão inclui uma migração de esquema de banco de dados que é compatível com ambas as versões do código (a antiga e a nova rodam contra o esquema novo). O deployment leva tipicamente 3 minutos para completar o traffic shift.

Como você estruturaria os hooks? Qual lifecycle event (`BeforeAllowTraffic`, `AfterAllowTestTraffic`, etc.) seria mais adequado para cada tipo de validação — verificação de health check, teste de fluxo de pagamento via listener de teste, e monitoramento de taxa de erro pós-shift? Qual `terminationWaitTime` você escolheria, considerando que a rollback do esquema de banco não é trivial? E se um alarm de alta latência disparasse 45 minutos após o deployment bem-sucedido (após o blue já ter sido destruído) — qual seria o procedimento de rollback nesse caso?

---

## Recursos para aprofundar

### 1. CodeDeploy blue/green deployments for Amazon ECS
**URL:** https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-type-bluegreen.html
**O que encontrar:** Visão geral completa do deployment type `CODE_DEPLOY` para ECS: componentes necessários, fluxo de deployment, diferenças em relação ao rolling update, e limitações (ex: não suporta Capacity Provider com base > 0 para o task set green em todas as configurações).
**Por que é a fonte certa:** É o ponto de entrada oficial para entender o modelo completo.

### 2. AppSpec 'hooks' section for ECS
**URL:** https://docs.aws.amazon.com/codedeploy/latest/userguide/reference-appspec-file-structure-hooks.html
**O que encontrar:** Referência completa de todos os lifecycle hooks disponíveis para ECS, a ordem de execução, como a Lambda de hook é invocada (payload), e o formato da chamada `PutLifecycleEventHookExecutionStatus`.
**Por que é a fonte certa:** É a referência técnica que define exatamente o protocolo entre CodeDeploy e a Lambda de hook.

### 3. Working with deployment configurations in CodeDeploy
**URL:** https://docs.aws.amazon.com/codedeploy/latest/userguide/deployment-configurations.html
**O que encontrar:** Lista completa de configurações predefinidas para ECS (Canary, Linear, AllAtOnce), como criar configurações customizadas, e como o traffic shifting funciona em cada modo.
**Por que é a fonte certa:** É a referência para escolher ou criar a estratégia de traffic shifting correta para cada contexto de risco.

---

