# Sessão — ECS: Capacity Providers e Application Auto Scaling

**Duração estimada:** 60 minutos
**Pré-requisitos:** session-015-ecs-fargate-networking-iam

---

## Objetivo

Ao final, você conseguirá configurar um Capacity Provider para Fargate e Fargate Spot com pesos, escalar um ECS Service baseado em métricas customizadas (não apenas CPU/memory), e calcular a economia de custo de Fargate Spot vs Fargate padrão em um cenário de workload com tolerância a interrupção.

---

## Contexto

[FATO] Capacity Providers são a camada de abstração do ECS que separa a definição de *onde* rodar tasks (qual pool de capacidade) da definição de *o que* rodar (task definition). Antes de Capacity Providers, você especificava `launchType: FARGATE` ou `launchType: EC2` diretamente no service — uma escolha binária e estática. Com Capacity Providers, você define uma **estratégia** com múltiplos providers e pesos, e o ECS distribui tasks entre eles automaticamente.

[FATO] Application Auto Scaling é o serviço AWS que gerencia o scaling de ECS Services (entre outros recursos como DynamoDB, Aurora, etc.). Ele é separado do EC2 Auto Scaling: enquanto o EC2 Auto Scaling gerencia instâncias, o Application Auto Scaling gerencia o `desiredCount` de um ECS Service. Os dois podem coexistir em uma arquitetura EC2-based (Application Auto Scaling aumenta tasks, EC2 Auto Scaling via Capacity Provider aumenta instâncias para acomodá-las).

[CONSENSO] O par Fargate + Fargate Spot é a combinação mais usada para serviços web que toleram alguma interrupção. A estratégia típica de produção mantém um base de tasks Fargate padrão (garantia de disponibilidade) e escala predominantemente em Fargate Spot (economia de custo), usando pesos para controlar a proporção.

---

## Conceitos principais

### 1. Capacity Providers: base e weight

[FATO] A estratégia de Capacity Provider de um service tem dois parâmetros por provider:

- **`base`**: número mínimo absoluto de tasks que devem rodar neste provider. Apenas um provider por estratégia pode ter base > 0. É satisfeito antes de qualquer cálculo de weight.
- **`weight`**: peso relativo. Após satisfazer o `base`, tasks adicionais são distribuídas proporcionalmente aos pesos.

```
Estratégia exemplo:
  FARGATE:       base=2, weight=1
  FARGATE_SPOT:  base=0, weight=4

desiredCount=2:
  → base de FARGATE é satisfeito com 2 tasks
  → nenhuma task restante para distribuir por weight
  Resultado: 2 tasks em FARGATE, 0 em FARGATE_SPOT

desiredCount=7:
  → base: 2 tasks em FARGATE (base=2)
  → restam 5 tasks para distribuir por weight 1:4
    → 1 task em FARGATE  (1/5 × 5 = 1)
    → 4 tasks em FARGATE_SPOT  (4/5 × 5 = 4)
  Resultado: 3 tasks em FARGATE, 4 tasks em FARGATE_SPOT

desiredCount=10:
  → base: 2 tasks em FARGATE
  → restam 8 tasks por weight 1:4
    → 1.6 → arredonda para 2 em FARGATE
    → 6.4 → arredonda para 6 em FARGATE_SPOT
  Resultado: ~4 tasks em FARGATE, ~6 tasks em FARGATE_SPOT
```

[FATO] O `base` garante que mesmo quando o desiredCount é baixo (ex: 1 task à meia-noite), você sempre tem ao menos N tasks em capacidade estável. Isso é crítico para serviços que não toleram zero disponibilidade durante um evento de interrupção Fargate Spot.

---

### 2. Fargate Spot: preço, interrupção e graceful shutdown

[FATO] Fargate Spot executa tasks em capacidade Fargate sobressalente a um preço reduzido em relação ao Fargate padrão. O desconto histórico observado é de aproximadamente **70% em relação ao preço on-demand** do Fargate (o valor exato varia por região e flutuação de oferta/demanda — a AWS não publica o percentual como faz com EC2 Spot).

[FATO] Quando a AWS precisa recuperar capacidade Fargate Spot, ela emite um aviso de interrupção de **2 minutos** antes de encerrar a task. O aviso chega por dois canais simultâneos:

1. **EventBridge**: um evento `ECS Task State Change` com `stopCode: SpotInterruption` é publicado.
2. **SIGTERM** no container: o processo principal recebe SIGTERM, igual a um shutdown normal.

```
t=0:   AWS decide recuperar capacidade
t=0:   EventBridge publica SpotInterruption event
t=0:   Container recebe SIGTERM
t=120: Container recebe SIGKILL (se ainda estiver rodando)
       Task é encerrada
```

[FATO] O parâmetro `stopTimeout` na container definition controla por quanto tempo o ECS aguarda entre o SIGTERM e o SIGKILL. O padrão é 30 segundos. Para Fargate Spot, você pode configurar até 120 segundos (o limite do aviso de 2 minutos). Configurar `stopTimeout: 120` dá ao container o máximo de tempo para:
- Terminar processamento em andamento.
- Fazer checkpoint de estado.
- Devolver mensagens SQS para a fila (nack/visibility timeout).
- Desregistrar do service registry.

**Workloads ideais para Fargate Spot:**

```
✅ Adequados para Spot:
  - Processamento de filas SQS (mensagens retornam à fila no SIGTERM)
  - Batch jobs com checkpoint (recomeça do checkpoint)
  - Rendering, transcodificação (job é reprocessado)
  - Ambientes de desenvolvimento/staging
  - Workers stateless com idempotência garantida

❌ Inadequados para Spot (use Fargate padrão):
  - Bases de dados stateful
  - Processamento transacional sem idempotência
  - WebSockets com estado de conexão crítico
  - Tasks que não tratam SIGTERM
```

---

### 3. Application Auto Scaling para ECS

[FATO] O Application Auto Scaling gerencia o `desiredCount` de um ECS Service. Ele requer três componentes:

1. **Scalable Target**: registra o service como alvo de scaling.
2. **Scaling Policy**: define *como* escalar (target tracking ou step scaling).
3. **CloudWatch Alarm**: (automático para target tracking, manual para step scaling).

**Target Tracking vs Step Scaling:**

```
Target Tracking:
  → Você define um TARGET VALUE para uma métrica
  → O AS calcula quantas tasks são necessárias para manter esse valor
  → Escala out quando métrica > target, in quando métrica < target
  → Mais simples, recomendado para a maioria dos casos
  → Cooldown de scale-in padrão: 300 segundos

Step Scaling:
  → Você define LIMIARES e quantas tasks adicionar/remover por limiar
  → Mais controle, mais complexo de configurar corretamente
  → Recomendado quando você precisa de reação rápida a spikes
  → Pode empilhar múltiplos steps (ex: +2 tasks quando CPU > 60%,
    +5 tasks quando CPU > 80%)
```

[FATO] Métricas predefinidas disponíveis para target tracking em ECS:

| Métrica | Descrição | Valor típico de target |
|---|---|---|
| `ECSServiceAverageCPUUtilization` | CPU média das tasks | 50-70% |
| `ECSServiceAverageMemoryUtilization` | Memória média | 60-80% |
| `ALBRequestCountPerTarget` | Requests/segundo por task via ALB | Depende da capacidade da app |

---

### 4. Scaling por métricas customizadas: o caso SQS

[FATO] O padrão mais robusto para serviços de processamento de fila é escalar baseado em **backlog por task** — não no número absoluto de mensagens na fila, mas na relação entre mensagens pendentes e tasks processando. Isso evita o problema de under-scaling quando o desiredCount é alto e over-scaling quando é baixo.

```
Fórmula do backlog por task:
  backlog_per_task = ApproximateNumberOfMessages / desiredCount

Exemplo:
  ApproximateNumberOfMessages = 1000 mensagens na fila
  desiredCount = 5 tasks
  backlog_per_task = 200 mensagens/task

  Se o target for 100 mensagens/task:
  → Para processar 1000 mensagens com 100/task cada
  → Precisamos de 1000/100 = 10 tasks
  → O AS escala de 5 para 10
```

[FATO] Para implementar essa fórmula no Application Auto Scaling, você usa **Metric Math** em Target Tracking:

```json
{
  "CustomizedMetricSpecification": {
    "Metrics": [
      {
        "Id": "messages",
        "MetricStat": {
          "Metric": {
            "Namespace": "AWS/SQS",
            "MetricName": "ApproximateNumberOfMessagesVisible",
            "Dimensions": [{ "Name": "QueueName", "Value": "my-queue" }]
          },
          "Stat": "Sum"
        }
      },
      {
        "Id": "tasks",
        "MetricStat": {
          "Metric": {
            "Namespace": "ECS/ContainerInsights",
            "MetricName": "RunningTaskCount",
            "Dimensions": [
              { "Name": "ClusterName", "Value": "my-cluster" },
              { "Name": "ServiceName", "Value": "my-worker" }
            ]
          },
          "Stat": "Average"
        }
      },
      {
        "Id": "backlog",
        "Expression": "messages / tasks",
        "ReturnData": true
      }
    ]
  },
  "TargetValue": 100.0
}
```

---

## Exemplo prático

**Cenário:** Um serviço de processamento de imagens (`image-processor`) que:
- Consome de uma fila SQS.
- Tolera interrupções (jobs idempotentes com SQS visibility timeout como mecanismo de retry).
- Deve manter pelo menos 2 tasks on-demand para garantir processamento mínimo.
- Deve escalar predominantemente em Fargate Spot para economia de custo.
- Deve escalar baseado em backlog por task (alvo: 50 mensagens/task).

### CDK completo (TypeScript)

```typescript
import { Stack, StackProps, Duration } from 'aws-cdk-lib';
import * as ecs from 'aws-cdk-lib/aws-ecs';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as sqs from 'aws-cdk-lib/aws-sqs';
import * as appscaling from 'aws-cdk-lib/aws-applicationautoscaling';
import * as cloudwatch from 'aws-cdk-lib/aws-cloudwatch';
import { Construct } from 'constructs';

export class ImageProcessorStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    const vpc = ec2.Vpc.fromLookup(this, 'Vpc', { vpcName: 'prod-vpc' });

    // Fila SQS de entrada
    const queue = new sqs.Queue(this, 'ImageQueue', {
      queueName: 'image-processing-queue',
      visibilityTimeout: Duration.seconds(300),  // 5 min para processar cada job
    });

    // Cluster com Capacity Providers Fargate e Fargate Spot
    const cluster = new ecs.Cluster(this, 'Cluster', {
      vpc,
      enableFargateCapacityProviders: true,  // habilita FARGATE e FARGATE_SPOT
    });

    // Task Definition
    const taskDef = new ecs.FargateTaskDefinition(this, 'TaskDef', {
      cpu: 1024,
      memoryLimitMiB: 2048,
    });

    taskDef.addContainer('processor', {
      image: ecs.ContainerImage.fromRegistry('my-org/image-processor:latest'),
      environment: { QUEUE_URL: queue.queueUrl },
      logging: ecs.LogDrivers.awsLogs({ streamPrefix: 'processor' }),
      // Máximo de tempo para graceful shutdown em Spot
      stopTimeout: Duration.seconds(120),
    });

    // Permissão para consumir da fila
    queue.grantConsumeMessages(taskDef.taskRole);

    // ECS Service com estratégia de Capacity Provider
    const service = new ecs.FargateService(this, 'Service', {
      cluster,
      taskDefinition: taskDef,
      desiredCount: 2,
      // Estratégia: 2 tasks fixas em Fargate, escala em Fargate Spot
      capacityProviderStrategies: [
        {
          capacityProvider: 'FARGATE',
          base: 2,       // sempre 2 tasks on-demand
          weight: 1,     // 1 de 5 tasks adicionais em on-demand
        },
        {
          capacityProvider: 'FARGATE_SPOT',
          base: 0,
          weight: 4,     // 4 de 5 tasks adicionais em Spot (80%)
        },
      ],
      vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS },
    });

    // ─── Application Auto Scaling ──────────────────────────────────────────

    const scaling = service.autoScaleTaskCount({
      minCapacity: 2,
      maxCapacity: 20,
    });

    // Métrica de mensagens na fila
    const messagesVisible = new cloudwatch.Metric({
      namespace: 'AWS/SQS',
      metricName: 'ApproximateNumberOfMessagesVisible',
      dimensionsMap: { QueueName: queue.queueName },
      statistic: 'Sum',
      period: Duration.minutes(1),
    });

    // Escalar baseado em ALB Request Count (alternativa simples para APIs)
    // Para SQS, usamos step scaling com a métrica de backlog
    scaling.scaleOnMetric('ScaleOnQueueDepth', {
      metric: messagesVisible,
      scalingSteps: [
        { upper: 0, change: -1 },     // fila vazia: remove 1 task
        { lower: 100, change: +1 },   // > 100 mensagens: adiciona 1 task
        { lower: 500, change: +3 },   // > 500 mensagens: adiciona 3 tasks
        { lower: 1000, change: +5 },  // > 1000 mensagens: adiciona 5 tasks
      ],
      adjustmentType: appscaling.AdjustmentType.CHANGE_IN_CAPACITY,
      cooldown: Duration.seconds(120),
    });

    // Scale in conservador: espera 5 minutos antes de remover tasks
    scaling.scaleOnCpuUtilization('ScaleOnCpu', {
      targetUtilizationPercent: 60,
      scaleInCooldown: Duration.seconds(300),
      scaleOutCooldown: Duration.seconds(60),
    });
  }
}
```

### Calculando economia de custo: Fargate vs Fargate Spot

```
Cenário: serviço com desiredCount médio de 10 tasks (1 vCPU, 2 GB cada)

Preço Fargate (us-east-1, referência aproximada em 2025):
  CPU:    $0.04048 por vCPU-hora
  Memory: $0.004445 por GB-hora

Custo por task/hora (Fargate padrão):
  = (1 vCPU × $0.04048) + (2 GB × $0.004445)
  = $0.04048 + $0.00889
  = $0.04937/hora por task

Custo por task/hora (Fargate Spot, ~70% desconto):
  = $0.04937 × 0.30
  = ~$0.01481/hora por task

Custo mensal com estratégia mista (base=2 Fargate, 8 Spot):
  Fargate padrão:  2 tasks × $0.04937 × 720h = $71.09
  Fargate Spot:    8 tasks × $0.01481 × 720h = $85.34
  Total:           $156.43/mês

Custo mensal com 100% Fargate padrão:
  10 tasks × $0.04937 × 720h = $355.46/mês

Economia com a estratégia mista: ~56% (~$199/mês)
```

[INCERTO] Os preços exatos do Fargate Spot variam por região e flutuam conforme oferta/demanda. O desconto de 70% é baseado em observações históricas comuns, mas pode ser diferente no momento que você estiver lendo. Verifique o preço atual em `aws.amazon.com/fargate/pricing`.

---

## Armadilhas comuns

### Armadilha 1: base=0 em ambos os providers — tasks sempre Spot durante scaling reduzido

**O erro:** Você configura a estratégia `FARGATE: base=0, weight=1` e `FARGATE_SPOT: base=0, weight=4`. Com `desiredCount=1` (ex: serviço em baixa carga à noite), o único task vai para Fargate Spot (maior peso). Uma interrupção Spot à noite derruba o único task do serviço. O ALB começa a retornar 503. O ECS inicia nova task em Spot, mas enquanto a nova task sobe (~30-60s), o serviço está indisponível.

**Por que acontece:** Com `base=0`, o ECS distribui *todas* as tasks por weight, incluindo a primeira. Sem nenhuma task garantida em Fargate padrão, uma interrupção Spot resulta em downtime durante o tempo de replacement.

**Como evitar:** Sempre defina `base >= 1` (ou `base >= 2` para zero-downtime durante replacement) no provider mais estável (FARGATE padrão). O `base` é o seguro de disponibilidade mínima.

---

### Armadilha 2: cooldown de scale-in curto causando thrashing

**O erro:** Você configura target tracking com `scaleInCooldown: Duration.seconds(60)`. Uma fila SQS tem processamento em burst: enche rapidamente, é processada em 2 minutos, esvazia. O AS escala out para 10 tasks, a fila esvazia, o AS escala in para 2 tasks em 60 segundos. Trinta segundos depois, nova mensagem entra, o AS escala out novamente. Esse ciclo cria **thrashing** — tasks sendo criadas e destruídas a cada poucos minutos, o que:
- Aumenta o custo (cobrança mínima de 1 minuto por task Fargate).
- Degrada o desempenho (tempo de cold start das tasks).
- Cria ruído excessivo nos logs e métricas.

**Por que acontece:** Scale-in rápido sem entender o padrão de chegada de mensagens. Para filas com bursts periódicos, o cooldown deve ser maior que o período entre bursts.

**Como evitar:** Para filas SQS, use `scaleInCooldown: Duration.seconds(300)` (5 minutos) ou maior. Scale out pode ser rápido (30-60s), mas scale in deve ser conservador. Analise o padrão histórico de tráfego antes de configurar cooldowns.

---

### Armadilha 3: Não tratar SIGTERM em containers Fargate Spot

**O erro:** O container do worker em Fargate Spot usa um processo Python sem tratamento de sinal. Quando a interrupção Spot ocorre, o processo recebe SIGTERM mas não o trata — continua rodando. Após o `stopTimeout` (padrão: 30 segundos), o ECS envia SIGKILL. A mensagem SQS que estava sendo processada não volta para a fila (o worker não fez `nack` ou reiniciou o visibility timeout). A mensagem vai para o DLQ após exceder o `maxReceiveCount`, ou fica "perdida" se o processamento estava a meio caminho.

**Por que acontece:** A maioria dos runtimes ignora SIGTERM por padrão. Python, Node.js e Java precisam de handlers explícitos para SIGTERM.

**Como evitar:** Implemente handler de SIGTERM que:
1. Para de aceitar novas mensagens da fila.
2. Aguarda o processamento atual terminar (ou cancela e faz nack).
3. Sai com `exit(0)`.

```python
# Python — exemplo de handler SIGTERM para worker SQS
import signal
import sys

shutdown_requested = False

def handle_sigterm(signum, frame):
    global shutdown_requested
    print("SIGTERM recebido — iniciando graceful shutdown")
    shutdown_requested = True

signal.signal(signal.SIGTERM, handle_sigterm)

while not shutdown_requested:
    messages = sqs.receive_message(QueueUrl=QUEUE_URL, MaxNumberOfMessages=1)
    if 'Messages' in messages:
        msg = messages['Messages'][0]
        try:
            process(msg)
            sqs.delete_message(QueueUrl=QUEUE_URL, ReceiptHandle=msg['ReceiptHandle'])
        except Exception as e:
            # Não deleta — mensagem volta à fila automaticamente
            print(f"Erro: {e}")

print("Shutdown completo")
sys.exit(0)
```

---

## Exercício de reflexão

Você está dimensionando um serviço de transcodificação de vídeo no ECS Fargate que processa jobs de uma fila SQS. Cada job leva em média 8 minutos para completar. O volume de jobs tem padrão diário claro: pico de 500 jobs/hora das 14h às 18h e baixa de 10 jobs/hora das 0h às 6h.

Como você configuraria a estratégia de Capacity Provider (base e weight entre Fargate e Fargate Spot)? Qual seria o `stopTimeout` adequado para o container? Qual seria a métrica de scaling mais adequada — CPU, contagem de mensagens na fila, ou backlog por task — e qual seria o valor de target? Quais são os riscos de usar Fargate Spot para esse tipo de workload e como o design do worker (tratamento de SIGTERM, idempotência, checkpoint) mitiga esses riscos? Por fim, como você calcularia o custo mensal estimado com a estratégia mista versus 100% Fargate padrão?

---

## Recursos para aprofundar

### 1. Amazon ECS clusters for Fargate (Capacity Providers)
**URL:** https://docs.aws.amazon.com/AmazonECS/latest/developerguide/fargate-capacity-providers.html
**O que encontrar:** Documentação completa dos Capacity Providers FARGATE e FARGATE_SPOT: como configurar a estratégia, os valores de base e weight, o mecanismo de interrupção Spot e como tratar o SIGTERM.
**Por que é a fonte certa:** É a referência primária para Fargate Capacity Providers — inclui os detalhes do protocolo de interrupção que qualquer operador de Spot precisa entender.

### 2. Automatically scale your Amazon ECS service
**URL:** https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-auto-scaling.html
**O que encontrar:** Guia completo de Application Auto Scaling para ECS: target tracking, step scaling, métricas predefinidas e customizadas, e configuração de cooldowns. Inclui exemplos com AWS CLI.
**Por que é a fonte certa:** É o guia oficial que integra ECS com Application Auto Scaling, cobrindo tanto os conceitos quanto os exemplos práticos de configuração.

### 3. Optimizing Amazon ECS service auto scaling
**URL:** https://docs.aws.amazon.com/AmazonECS/latest/developerguide/capacity-autoscaling-best-practice.html
**O que encontrar:** Guia de boas práticas específico para ECS scaling: como escolher a métrica certa por tipo de workload, armadilhas de cooldown, e o padrão de backlog-per-task para filas SQS.
**Por que é a fonte certa:** É o guia prescritivo — diz *o que fazer* além de *como fazer*, com justificativas para cada recomendação.

---

