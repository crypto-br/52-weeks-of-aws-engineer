# Sessão session-014-ecs-services-discovery-alb — ECS: Services, service discovery e integração com ALB Target Groups

**Duração estimada:** 60 minutos
**Pré-requisitos:** session-013-ecs-task-definitions-logging

---

## Objetivo

Ao final, você conseguirá criar um ECS Service que registra tasks em um ALB Target Group, configurar health checks com grace period, usar AWS Cloud Map para service discovery service-to-service (sem passar pelo load balancer), e entender quando usar cada abordagem.

---

## Contexto

[FATO] Uma **Task Definition** descreve *o que* rodar. Um **ECS Service** descreve *como* rodar: quantas réplicas manter, como fazer rolling update, onde registrar as tasks para receber tráfego, e o que fazer quando uma task falha. O Service é o mecanismo de self-healing do ECS — ele monitora continuamente o número de tasks saudáveis e cria novas se o número cair abaixo do `desiredCount`.

[FATO] O ECS suporta dois tipos de integração para recebimento de tráfego: **ALB/NLB Target Groups** para tráfego externo (e interno passando pelo load balancer) e **AWS Cloud Map** para service discovery DNS service-to-service sem load balancer. Eles não são excludentes — um serviço pode ter ambos simultaneamente.

[CONSENSO] A escolha entre ALB e Cloud Map para comunicação interna é um trade-off entre operabilidade e eficiência. O ALB oferece terminação TLS, roteamento por path/header, circuit breaking, e observabilidade nativa via access logs. O Cloud Map é mais simples e eficiente para comunicação service-to-service de alta frequência onde o overhead do load balancer importa e onde a descoberta dinâmica de múltiplas instâncias é necessária sem a indireção de um VIP.

---

## Conceitos principais

### 1. Anatomia de um ECS Service

```
ECS Service
│
├── taskDefinition        → qual revisão deployar
├── desiredCount          → quantas tasks manter rodando
├── launchType            → FARGATE | EC2 | EXTERNAL
│
├── deploymentConfiguration
│   ├── minimumHealthyPercent  → % mínima de tasks saudáveis durante deploy
│   ├── maximumPercent         → % máxima de tasks durante deploy
│   └── deploymentCircuitBreaker
│       ├── enable: true/false
│       └── rollback: true/false
│
├── loadBalancers[]
│   └── { targetGroupArn, containerName, containerPort }
│
├── serviceRegistries[]
│   └── { registryArn }   → Cloud Map service ARN
│
├── healthCheckGracePeriodSeconds  → ignora health checks por N segundos após start
│
└── networkConfiguration
    └── awsvpcConfiguration
        ├── subnets[]
        ├── securityGroups[]
        └── assignPublicIp: ENABLED | DISABLED
```

[FATO] O Service scheduler é um loop contínuo. A cada ciclo, ele:
1. Conta as tasks no estado `RUNNING` que passaram no health check.
2. Se a contagem for menor que `desiredCount`, inicia novas tasks.
3. Se uma task falhar (exit code, OOM kill, health check falhou), para a task e inicia uma nova.
4. Durante um deploy (nova revisão da task definition), executa o rolling update respeitando `minimumHealthyPercent` e `maximumPercent`.

---

### 2. Rolling update: minimumHealthyPercent e maximumPercent

[FATO] Os dois parâmetros de deployment configuration definem o envelope de capacidade durante um rolling update:

```
desiredCount = 4 tasks

Valores padrão (REPLICA service):
  minimumHealthyPercent = 100%  → mínimo: ceil(4 × 1.0) = 4 tasks saudáveis
  maximumPercent        = 200%  → máximo: floor(4 × 2.0) = 8 tasks total

Comportamento com padrões:
  1. ECS inicia 4 novas tasks (nova revisão) → total: 8 tasks
  2. Aguarda as 4 novas passarem no health check
  3. Para as 4 antigas tasks
  4. Resultado: zero downtime, mas capacidade dobrada momentaneamente

Valores agressivos (deploy mais rápido, menor custo):
  minimumHealthyPercent = 50%   → mínimo: ceil(4 × 0.5) = 2 tasks saudáveis
  maximumPercent        = 150%  → máximo: floor(4 × 1.5) = 6 tasks

Comportamento agressivo:
  1. Para 2 tasks antigas → total: 2 tasks (50% de capacidade)
  2. Inicia 4 novas tasks → total: 6 tasks
  3. Aguarda novas passarem no health check
  4. Para as 2 tasks antigas restantes
  5. Resultado: redução temporária de 50% de capacidade durante o deploy
```

[FATO] Para serviços com `desiredCount = 1` (comum em desenvolvimento), os padrões `min=100%, max=200%` são os únicos que garantem zero downtime: o ECS inicia a nova task antes de parar a antiga. Com `min=0%, max=100%`, o deploy para a task antiga primeiro, causando downtime.

---

### 3. Deployment Circuit Breaker

[FATO] O circuit breaker detecta quando um deploy está falhando continuamente e, opcionalmente, faz rollback automático para a revisão anterior. Ele usa uma janela deslizante de tarefas com falha:

```
Threshold de falha = max(10, desiredCount × 2)
  → mínimo: 3 falhas   (documentação menciona mínimo de 3 neste contexto)
  → máximo: 200 falhas

Se a contagem de tasks falhando ultrapassar o threshold → circuit abre
  → se rollback=true: ECS reverte para a última revisão bem-sucedida
  → se rollback=false: deploy para, service fica em estado FAILED
```

[CONSENSO] Em produção, `deploymentCircuitBreaker: { enable: true, rollback: true }` é considerado uma boa prática. Sem circuit breaker, um deploy com imagem quebrada (que crasheia imediatamente) pode ficar em loop indefinidamente — o ECS continua tentando iniciar tasks que falham, pagando pelo custo de cada tentativa e impedindo que o serviço se recupere.

---

### 4. Integração com ALB Target Groups

[FATO] A integração entre ECS Service e ALB acontece via **Target Group** no modo IP. No Fargate com `networkMode: awsvpc`, cada task tem um IP privado próprio. O ECS registra e desregistra automaticamente esses IPs no Target Group conforme as tasks sobem e descem.

```
Internet / VPC
      │
      ▼
┌──────────────┐
│     ALB      │  Listener: HTTPS:443, HTTP:80
│              │
│  Listener    │──rule: path /* ──▶ Target Group (type: IP)
└──────────────┘                        │
                                        │ registra/desregistra automaticamente
                              ┌─────────┴──────────┐
                              │  Task 1: 10.0.1.5  │
                              │  Task 2: 10.0.1.8  │  ← port 3000
                              │  Task 3: 10.0.1.12 │
                              └────────────────────┘
```

[FATO] Ao remover uma task de circulação (durante rolling update ou scale-in), o ECS executa **connection draining** (também chamado de deregistration delay): envia a tarefa de desregistro ao Target Group, aguarda o ALB terminar as conexões ativas, e só então para o container. O tempo padrão de deregistration delay é 300 segundos — configurável no Target Group.

**Health Check Grace Period** é distinto do health check do Target Group:

```
healthCheckGracePeriodSeconds (no Service):
  → quanto tempo o ECS IGNORA falhas do ALB health check após a task iniciar
  → evita que tasks em bootstrap sejam mortas prematuramente
  → valor zero: ECS age imediatamente se o ALB marcar a task como unhealthy

Health Check no Target Group (no ALB):
  → intervalo, threshold, path — define quando o ALB considera uma task saudável
  → independente do grace period do ECS
```

[FATO] Um erro clássico: a aplicação demora 45 segundos para inicializar (baixando configs, aquecendo cache). O `healthCheckGracePeriodSeconds` padrão é zero. O ALB health check falha nos primeiros 45 segundos. O ECS mata a task. A nova task também demora 45 segundos. Loop infinito. Solução: `healthCheckGracePeriodSeconds` deve ser ligeiramente maior que o `startPeriod` do health check do container + o tempo de inicialização da aplicação.

---

### 5. AWS Cloud Map: service discovery DNS sem load balancer

[FATO] AWS Cloud Map é um serviço de registro e descoberta de recursos. No contexto do ECS, ele permite que serviços se descubram mutuamente via DNS sem passar por um load balancer — o cliente resolve um nome DNS e obtém diretamente os IPs das tasks rodando.

**Componentes do Cloud Map:**

```
Private DNS Namespace: internal.svc
│
├── Service: "api"
│   ├── DNS records: A records → IPs das tasks do serviço api
│   └── Health check: syncronizado com health check do ECS
│
└── Service: "worker"
    ├── DNS records: A records → IPs das tasks do serviço worker
    └── Health check: sincronizado com health check do ECS

Consulta DNS: api.internal.svc → [10.0.1.5, 10.0.1.8, 10.0.1.12]
```

[FATO] Quando um ECS Service é configurado com Cloud Map (`serviceRegistries`), o ECS registra automaticamente um **instance** no Cloud Map para cada task que passa no health check. Quando a task para ou falha, o ECS remove o registro. O TTL do DNS determina quanto tempo os clientes cachearão os IPs antigos após uma task ser removida.

[FATO] Cloud Map suporta dois tipos de registros DNS para ECS:

| Tipo de record | Quando usar | Retorna |
|---|---|---|
| **A record** | `networkMode: awsvpc` (Fargate ou EC2) | IP privado da task |
| **SRV record** | `networkMode: bridge` ou `host` (EC2) | IP + porta do container |

No Fargate (sempre `awsvpc`), use A records. O SRV record é necessário apenas em EC2 com bridge mode, onde o IP do host é o mesmo para várias tasks e a porta é dinâmica.

---

### 6. ALB vs Cloud Map: quando usar cada abordagem

```
                    ALB Target Group          AWS Cloud Map
                    ────────────────          ─────────────
Latência            +20-40ms (hop extra)      Zero overhead (DNS)
Custo               ~$18/mês + LCUs           ~$1/mês + queries
TLS termination     Sim (ACM integrado)       Não (app gerencia)
Roteamento avançado Sim (path, header, host)  Não
Métricas nativas    Request count, latency    Não (precisa de X-Ray)
Circuit breaking    Sim (5xx % via listener)  Não
Múltiplos serviços  Sim (rules por path)      Um nome DNS por serviço
Descoberta de porta Não (port fixo no TG)     SRV records (bridge mode)
```

[CONSENSO] A regra prática adotada pela maioria dos times:

- **Use ALB** para: entrada de tráfego externo, APIs públicas, qualquer serviço que precise de TLS terminado, roteamento por path, ou quando você quer métricas de request no nível de application.
- **Use Cloud Map** para: comunicação service-to-service de alta frequência dentro da VPC onde a latência do ALB importa, serviços de background que não recebem tráfego externo, ou quando você precisa que o cliente conheça os IPs de todas as réplicas (ex: para client-side load balancing em gRPC).
- **Use ambos** para: serviços que recebem tráfego externo via ALB E são chamados internamente por outros serviços com baixa latência via Cloud Map.

---

## Exemplo prático

**Cenário:** Um sistema com dois serviços:
- `api-service`: recebe tráfego HTTPS externo via ALB, expõe `/api/*`.
- `worker-service`: processamento background, chamado apenas pelo `api-service` internamente via Cloud Map.

### Arquitetura

```
Internet
    │ HTTPS:443
    ▼
┌───────────────────────────────────┐
│  ALB: api.example.com             │
│  Listener rule: /* → Target Group │
└───────────────┬───────────────────┘
                │ registra IPs
                ▼
     ┌─────────────────────┐
     │  api-service (ECS)  │──── Cloud Map DNS lookup
     │  3 tasks Fargate    │     worker.internal.svc
     │  port 3000          │────────────────▶ ┌─────────────────────┐
     └─────────────────────┘                  │ worker-service (ECS)│
                                              │ 2 tasks Fargate     │
                                              │ port 8080           │
                                              └─────────────────────┘
```

### CDK (TypeScript) — stack completa

```typescript
import { Stack, StackProps, Duration } from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as ecs from 'aws-cdk-lib/aws-ecs';
import * as elbv2 from 'aws-cdk-lib/aws-elasticloadbalancingv2';
import * as servicediscovery from 'aws-cdk-lib/aws-servicediscovery';
import { Construct } from 'constructs';

export class AppStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    const vpc = ec2.Vpc.fromLookup(this, 'Vpc', { vpcName: 'prod-vpc' });

    // Cluster ECS
    const cluster = new ecs.Cluster(this, 'Cluster', {
      vpc,
      containerInsights: true,   // habilita métricas detalhadas por container
    });

    // Cloud Map namespace DNS privado
    const namespace = new servicediscovery.PrivateDnsNamespace(this, 'Namespace', {
      name: 'internal.svc',
      vpc,
    });

    // ─── Worker Service ───────────────────────────────────────────────────

    const workerTaskDef = new ecs.FargateTaskDefinition(this, 'WorkerTaskDef', {
      cpu: 256,
      memoryLimitMiB: 512,
    });
    workerTaskDef.addContainer('worker', {
      image: ecs.ContainerImage.fromRegistry('my-org/worker:latest'),
      portMappings: [{ containerPort: 8080 }],
      logging: ecs.LogDrivers.awsLogs({ streamPrefix: 'worker' }),
    });

    const workerSg = new ec2.SecurityGroup(this, 'WorkerSg', { vpc });

    const workerService = new ecs.FargateService(this, 'WorkerService', {
      cluster,
      taskDefinition: workerTaskDef,
      desiredCount: 2,
      securityGroups: [workerSg],
      // Sem load balancer — descoberta apenas via Cloud Map
      cloudMapOptions: {
        name: 'worker',                     // DNS: worker.internal.svc
        cloudMapNamespace: namespace,
        dnsRecordType: servicediscovery.DnsRecordType.A,
        dnsTtl: Duration.seconds(10),       // TTL baixo: falhas detectadas rápido
      },
      deploymentCircuitBreaker: { rollback: true },
      minHealthyPercent: 50,
      maxHealthyPercent: 200,
    });

    // ─── API Service + ALB ────────────────────────────────────────────────

    const apiTaskDef = new ecs.FargateTaskDefinition(this, 'ApiTaskDef', {
      cpu: 512,
      memoryLimitMiB: 1024,
    });
    apiTaskDef.addContainer('api', {
      image: ecs.ContainerImage.fromRegistry('my-org/api:latest'),
      portMappings: [{ containerPort: 3000 }],
      logging: ecs.LogDrivers.awsLogs({ streamPrefix: 'api' }),
      environment: {
        // O serviço api descobre o worker via Cloud Map
        WORKER_URL: 'http://worker.internal.svc:8080',
      },
    });

    // ALB
    const alb = new elbv2.ApplicationLoadBalancer(this, 'ALB', {
      vpc,
      internetFacing: true,
    });

    const listener = alb.addListener('HttpsListener', {
      port: 443,
      // certificates: [cert],  // ACM certificate em produção real
      open: true,
    });

    const apiSg = new ec2.SecurityGroup(this, 'ApiSg', { vpc });

    const apiService = new ecs.FargateService(this, 'ApiService', {
      cluster,
      taskDefinition: apiTaskDef,
      desiredCount: 3,
      securityGroups: [apiSg],
      deploymentCircuitBreaker: { rollback: true },
      minHealthyPercent: 100,
      maxHealthyPercent: 200,
      // Grace period: app demora ~30s para inicializar
      healthCheckGracePeriod: Duration.seconds(60),
    });

    // Registra o serviço no Target Group do ALB
    apiService.registerLoadBalancerTargets({
      containerName: 'api',
      containerPort: 3000,
      newTargetGroupId: 'ApiTG',
      listener: ecs.ListenerConfig.applicationListener(listener, {
        protocol: elbv2.ApplicationProtocol.HTTP,
        healthCheck: {
          path: '/health',
          interval: Duration.seconds(15),
          healthyThresholdCount: 2,
          unhealthyThresholdCount: 3,
          timeout: Duration.seconds(5),
        },
        deregistrationDelay: Duration.seconds(30),  // reduz de 300s padrão
      }),
    });

    // API precisa falar com worker via porta 8080
    workerSg.addIngressRule(apiSg, ec2.Port.tcp(8080), 'API para Worker');
  }
}
```

---

## Armadilhas comuns

### Armadilha 1: healthCheckGracePeriodSeconds zero com aplicação lenta para inicializar

**O erro:** O serviço tem `healthCheckGracePeriodSeconds` no valor padrão (zero). A aplicação demora 45 segundos para subir (carrega configurações, estabelece conexões com banco). O ALB health check começa imediatamente com intervalo de 30 segundos. Na primeira checagem (30s após início), a app ainda não está pronta → retorna 503. O ECS marca a task como unhealthy e a substitui. A nova task também demora 45 segundos. O serviço nunca consegue subir.

**Por que acontece:** O `healthCheckGracePeriodSeconds` foi projetado exatamente para esse cenário. Sem ele, o ECS trata falhas iniciais de health check da mesma forma que falhas em uma task já estabilizada.

**Como reconhecer:** No console ECS, a aba Events do serviço mostra `service X: task Y is unhealthy in target-group Z` repetidamente, com as tasks sendo substituídas antes de chegarem a 60 segundos de vida. No ALB, os access logs mostram apenas requests de health check retornando 5xx.

**Como evitar:** Meça o tempo de inicialização da sua aplicação em condições reais (não localhost). Configure `healthCheckGracePeriodSeconds` = tempo de inicialização + 20s de margem. Para aplicações Java com JVM warmup ou Python com carregamento de modelos ML, isso pode ser 120-180 segundos.

---

### Armadilha 2: TTL de DNS alto no Cloud Map causando chamadas para tasks mortas

**O erro:** Você configura `dnsTtl: Duration.seconds(60)` no Cloud Map (ou usa o padrão, que é 60 segundos). Durante um rolling update do `worker-service`, uma task é removida do Cloud Map. O `api-service` ainda tem o IP da task morta em cache DNS por até 60 segundos e continua tentando conectar a ela. As chamadas falham com `connection refused` durante esse período.

**Por que acontece:** DNS é um cache distribuído. O TTL define por quanto tempo os resolvers (e a JVM, e o Node.js) mantêm o valor. Se o TTL é 60s, um cliente que acabou de resolver o nome pode continuar usando o IP por até 60s após ele ser removido do Cloud Map.

**Como reconhecer:** Erros de `ECONNREFUSED` ou `connection reset` no `api-service` com duração de 30-60 segundos durante deployments do `worker-service`, mesmo com health check configurado.

**Como evitar:** Use `dnsTtl: Duration.seconds(10)` para serviços que fazem rolling updates frequentes. Valores menores reduzem o tempo de propagação mas aumentam o volume de queries DNS (custo mínimo no Cloud Map, geralmente aceitável). Adicionalmente, implemente retry com backoff no cliente para absorver o período de transição.

---

### Armadilha 3: Deregistration delay alto causando slowdowns no deploy

**O erro:** O Target Group tem o deregistration delay padrão de 300 segundos. Durante um rolling update de um serviço com `desiredCount=4`, o ECS precisa parar 4 tasks antigas. Para cada task, o ECS aguarda 300 segundos de connection draining antes de pará-la. Um rolling update completo demora: 4 tasks × 300 segundos = 20 minutos, mesmo que a nova versão esteja 100% saudável há 19 minutos.

**Por que acontece:** 300 segundos foi o padrão escolhido pela AWS para workloads com conexões de longa duração (WebSockets, streaming). Para APIs REST com conexões curtas (< 1 segundo), é um valor excessivo.

**Como reconhecer:** Deploys que demoram muito mais do que o esperado. O console ECS mostra tasks no estado `DEREGISTERING` por longos períodos. O painel do Target Group mostra targets em estado `draining`.

**Como evitar:** Configure `deregistrationDelay` no Target Group de acordo com o tipo de conexão. Para APIs REST típicas, 15-30 segundos é suficiente. Para WebSockets, mantenha em 300s ou mais. Você pode configurar isso via CDK no `HealthCheck` ao registrar o Target Group.

---

## Exercício de reflexão

Você está projetando um sistema de microserviços com 5 serviços no ECS Fargate:
- `gateway`: recebe tráfego externo (HTTPS via ALB)
- `auth`: verifica tokens JWT, chamado por `gateway` para cada request
- `orders`: lógica de negócio, chamado por `gateway`
- `inventory`: consultado por `orders` para verificar estoque
- `notifications`: envia emails/SMS, acionado por `orders` de forma assíncrona via SQS (não é chamado diretamente)

Para quais serviços você usaria ALB? Para quais usaria Cloud Map? O `notifications` precisa de algum dos dois mecanismos de descoberta? Qual seria o impacto de um TTL de DNS de 60 segundos no serviço `auth`, dado que ele é chamado em cada request do gateway? Como você configuraria os `healthCheckGracePeriodSeconds` para cada serviço, sabendo que `auth` usa uma imagem leve de Node.js (inicialização em 2 segundos) e `orders` usa uma app Java Spring Boot (inicialização em 40 segundos)?

---

## Recursos para aprofundar

### 1. Amazon ECS service definition parameters
**URL:** https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service_definition_parameters.html
**O que encontrar:** Referência completa de todos os parâmetros de um ECS Service: deployment configuration, load balancer integration, service registries, network configuration. É o documento-base quando você precisa entender o significado exato de um campo.
**Por que é a fonte certa:** É a referência oficial — mais precisa que o console ou tutoriais.

### 2. Deployment circuit breaker
**URL:** https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-circuit-breaker.html
**O que encontrar:** Como o circuit breaker detecta falhas de deploy, os thresholds de ativação, o comportamento de rollback, e as limitações (só funciona com rolling update, não com blue/green).
**Por que é a fonte certa:** Documenta o algoritmo de detecção — essencial para entender quando o circuit breaker vai e quando não vai proteger você.

### 3. Use service discovery to connect Amazon ECS services with DNS names
**URL:** https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-discovery.html
**O que encontrar:** Como configurar Cloud Map com ECS, diferença entre A e SRV records, sincronização de health checks entre ECS e Cloud Map, e limitações (máximo de 1000 instâncias por serviço Cloud Map).
**Por que é a fonte certa:** É o guia oficial de integração ECS + Cloud Map, com exemplos de configuração via console, CLI e CloudFormation.

---

