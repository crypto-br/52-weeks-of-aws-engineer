# Sessão — ECS Observabilidade: FireLens, Container Insights e X-Ray sidecar

**Duração estimada:** 60 minutos
**Pré-requisitos:** session-017-ecs-deploy-rolling-blue-green

---

## Objetivo

Ao final, você conseguirá configurar FireLens como router de logs Fluent Bit (enviando logs simultaneamente para CloudWatch e S3/Kinesis), habilitar Container Insights para métricas por task, e adicionar o X-Ray daemon como sidecar container com a task role correta.

---

## Contexto

[FATO] O `awslogs` driver (sessão 013) é simples e eficiente para enviar stdout/stderr direto ao CloudWatch, mas tem limitações: cada container vai para exatamente um destino, não é possível filtrar ou transformar logs antes de enviá-los, e não suporta roteamento para destinos não-CloudWatch (S3, Elasticsearch, Datadog, Splunk) sem código adicional. **FireLens** resolve essas limitações introduzindo um Fluent Bit sidecar como router de logs — o container principal envia para o FireLens, e o FireLens distribui para múltiplos destinos com transformações.

[FATO] **Container Insights** é o sistema de métricas detalhadas por container do CloudWatch. As métricas padrão do ECS (CPU e memory a nível de service) existem sem configuração adicional, mas Container Insights adiciona granularidade: métricas por task individual, por container dentro da task, disk I/O, e network I/O — publicadas no namespace `ECS/ContainerInsights`.

[CONSENSO] O X-Ray daemon como sidecar é o padrão recomendado pela AWS para tracing em ECS Fargate. O modelo é: a aplicação envia segmentos de trace via UDP para `localhost:2000`, o daemon coleta, batcha, e envia para a API do X-Ray de forma assíncrona. Isso desacopla o código da aplicação da rede pública — a aplicação não precisa de outbound HTTPS para X-Ray, apenas UDP loopback.

---

## Conceitos principais

### 1. FireLens: arquitetura e fluxo de dados

[FATO] FireLens é um mecanismo de logging do ECS que usa **Fluent Bit** (ou Fluentd) como container sidecar para receber e rotear logs. O fluxo é:

```
Container principal
  │
  │  logDriver: "awsfirelens"    ← não awslogs!
  │  options: { ... destino ... }
  │
  ▼
FireLens sidecar (Fluent Bit)   ← porta 24224 (Fluent protocol)
  │
  ├──▶ CloudWatch Logs           (plugin cloudwatch_logs)
  ├──▶ Amazon Kinesis Firehose   (plugin kinesis_firehose)
  ├──▶ Amazon Kinesis Streams    (plugin kinesis)
  └──▶ Qualquer destino Fluent Bit compatível
       (S3, Elasticsearch, Datadog, Splunk, etc.)
```

[FATO] A imagem oficial do FireLens mantida pela AWS já inclui os plugins para CloudWatch, Kinesis, e Firehose pré-instalados:

- **Fluent Bit**: `public.ecr.aws/aws-observability/aws-for-fluent-bit:stable`
- **Fluentd**: `public.ecr.aws/aws-observability/fluentd:latest`

O Fluent Bit é preferido na maioria dos casos por ser mais leve (C vs Ruby) e ter melhor performance em alta carga.

[FATO] A configuração do FireLens acontece em dois lugares na task definition:

1. **Container do FireLens** (o sidecar): tipo `firelens`, define qual imagem usar e configurações opcionais.
2. **Containers da aplicação**: usam `logDriver: awsfirelens` com opções que especificam o destino e eventuais filtros.

```
Task Definition
│
├── Container: firelens-router    ← tipo: firelens
│   ├── firelensConfiguration:
│   │   ├── type: "fluentbit"
│   │   └── options:
│   │       ├── enable-ecs-log-metadata: "true"  ← adiciona taskId, cluster, etc.
│   │       └── config-file-type: "file"          ← configuração customizada
│   │           config-file-value: "/fluent-bit/custom.conf"
│   └── essential: false   ← app continua se o firelens falhar
│
└── Container: api
    └── logConfiguration:
        ├── logDriver: "awsfirelens"    ← envia para o sidecar FireLens
        └── options:
            ├── Name: "cloudwatch_logs"
            ├── region: "us-east-1"
            ├── log_group_name: "/ecs/api"
            └── log_stream_prefix: "api/"
```

---

### 2. Roteamento para múltiplos destinos

[FATO] Para enviar logs para múltiplos destinos simultaneamente com FireLens, você usa uma **configuração customizada Fluent Bit**. A imagem padrão do AWS FireLens gera automaticamente uma configuração base (inputs, parsers, filtros de metadata do ECS), e você pode adicionar outputs extras via arquivo de configuração customizada.

Exemplo: enviar para CloudWatch **e** Kinesis Firehose simultaneamente:

**`/fluent-bit/custom.conf`** (empacotado na imagem ou em S3):

```ini
# Este arquivo é INCLUÍDO na configuração gerada automaticamente pelo ECS
# Não redefinir [INPUT] ou [FILTER] — o ECS já os gera

# Output 1: CloudWatch Logs
[OUTPUT]
    Name              cloudwatch_logs
    Match             api-*            # filtra apenas logs do container 'api'
    region            us-east-1
    log_group_name    /ecs/api-service
    log_stream_prefix api/
    auto_create_group true

# Output 2: Kinesis Firehose (para S3/analytics)
[OUTPUT]
    Name              kinesis_firehose
    Match             api-*
    region            us-east-1
    delivery_stream   api-logs-firehose

# Output 3: Descarta logs do próprio firelens (evita recursão)
[OUTPUT]
    Name              null
    Match             firelens*
```

[FATO] O `Match` no Fluent Bit usa o **tag** do log. O ECS FireLens gera automaticamente tags no formato `{container-name}-{task-id}`. Por isso, `Match api-*` captura todos os logs do container chamado `api`, independente do task ID.

[FATO] O campo `enable-ecs-log-metadata: "true"` nas opções do firelensConfiguration faz o FireLens adicionar automaticamente metadados do ECS a cada registro de log:

```json
{
  "log": "2026-06-10T10:30:00Z INFO request completed",
  "container_id": "abc123",
  "container_name": "api",
  "source": "stdout",
  "ecs_cluster": "prod-cluster",
  "ecs_task_arn": "arn:aws:ecs:...",
  "ecs_task_definition": "api-service:42"
}
```

Isso elimina a necessidade de correlacionar logs com tasks manualmente — cada linha de log já sabe de qual task veio.

---

### 3. Container Insights: métricas por container

[FATO] O Container Insights publica métricas no namespace `ECS/ContainerInsights` com dimensões que permitem drill-down até o nível de container individual. As métricas disponíveis com Container Insights **enhanced**:

| Métrica | Dimensão | Descrição |
|---|---|---|
| `CpuUtilized` | ClusterName, ServiceName, TaskId | CPU usada por task (unidades) |
| `CpuReserved` | ClusterName, ServiceName | CPU reservada total do service |
| `MemoryUtilized` | ClusterName, ServiceName, TaskId | Memória usada em MiB |
| `MemoryReserved` | ClusterName, ServiceName | Memória reservada total |
| `NetworkRxBytes` | ClusterName, TaskId | Bytes recebidos pela task |
| `NetworkTxBytes` | ClusterName, TaskId | Bytes transmitidos pela task |
| `StorageReadBytes` | ClusterName, TaskId | I/O de leitura |
| `StorageWriteBytes` | ClusterName, TaskId | I/O de escrita |
| `RunningTaskCount` | ClusterName, ServiceName | Tasks rodando |

[FATO] Há dois níveis de Container Insights:

- **`enabled`** (padrão ao ativar): métricas de cluster, service e task. Sem nível de container individual.
- **`enhanced`**: adiciona métricas por container dentro da task. Requer `enhanced` explicitamente.

```bash
# Ativar Container Insights enhanced em um cluster existente
aws ecs update-cluster-settings \
  --cluster prod-cluster \
  --settings name=containerInsights,value=enhanced

# Verificar configuração atual
aws ecs describe-clusters --clusters prod-cluster \
  --query 'clusters[0].settings'
```

[FATO] Container Insights tem custo adicional: as métricas são publicadas no CloudWatch e cobradas por métrica-por-mês (~$0.30/métrica/mês). Um cluster com 10 services pode gerar centenas de métricas — verifique o custo estimado antes de habilitar em escala.

---

### 4. X-Ray daemon como sidecar

[FATO] O X-Ray daemon é um processo que recebe segmentos de trace (UDP datagramas) na porta 2000, os bufferiza, e os envia em batch para a API do X-Ray. Rodar como sidecar tem duas vantagens sobre embutir o SDK de envio direto na aplicação:

1. **Desacoplamento de rede**: a aplicação envia para `localhost:2000` (UDP, sem latência de rede). O daemon gerencia a conexão com a API X-Ray, retries, e batching.
2. **Independência de linguagem**: o mesmo sidecar serve para Java, Python, Node.js, Go — cada SDK da aplicação só precisa saber enviar para `localhost:2000`.

[FATO] Configuração do container X-Ray daemon na task definition:

```json
{
  "name": "xray-daemon",
  "image": "public.ecr.aws/xray/aws-xray-daemon:3.x",
  "essential": false,
  "cpu": 32,
  "memoryReservation": 256,
  "portMappings": [
    {
      "containerPort": 2000,
      "protocol": "udp"
    }
  ],
  "environment": [
    { "name": "AWS_REGION", "value": "us-east-1" }
  ],
  "logConfiguration": {
    "logDriver": "awslogs",
    "options": {
      "awslogs-group": "/ecs/xray-daemon",
      "awslogs-region": "us-east-1",
      "awslogs-stream-prefix": "xray",
      "mode": "non-blocking"
    }
  }
}
```

[FATO] A **Task Role** precisa da permissão para enviar traces ao X-Ray:

```json
{
  "Effect": "Allow",
  "Action": [
    "xray:PutTraceSegments",
    "xray:PutTelemetryRecords",
    "xray:GetSamplingRules",
    "xray:GetSamplingTargets",
    "xray:GetSamplingStatisticSummaries"
  ],
  "Resource": "*"
}
```

Ou use a managed policy `AWSXRayDaemonWriteAccess`.

[FATO] No container da aplicação, configure o SDK X-Ray para enviar para o daemon:

```bash
# Variável de ambiente para todos os SDKs X-Ray
AWS_XRAY_DAEMON_ADDRESS=localhost:2000
```

No CDK/task definition, como os containers da task compartilham a rede (loopback), `localhost:2000` alcança o sidecar X-Ray sem configuração adicional de rede.

---

## Exemplo prático

**Cenário completo:** Um `api-service` com observabilidade full-stack:
- Logs via FireLens: CloudWatch para operação + Kinesis Firehose para analytics (S3).
- Container Insights enhanced para métricas por task.
- X-Ray daemon sidecar para distributed tracing.

### Task definition completa em JSON

```json
{
  "family": "api-service-observavel",
  "taskRoleArn": "arn:aws:iam::123456789012:role/api-task-role",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecs-execution-role",
  "networkMode": "awsvpc",
  "cpu": "1024",
  "memory": "2048",
  "requiresCompatibilities": ["FARGATE"],
  "containerDefinitions": [
    {
      "name": "log-router",
      "image": "public.ecr.aws/aws-observability/aws-for-fluent-bit:stable",
      "essential": false,
      "firelensConfiguration": {
        "type": "fluentbit",
        "options": {
          "enable-ecs-log-metadata": "true"
        }
      },
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/firelens",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "firelens",
          "mode": "non-blocking"
        }
      }
    },
    {
      "name": "xray-daemon",
      "image": "public.ecr.aws/xray/aws-xray-daemon:3.x",
      "essential": false,
      "cpu": 32,
      "memoryReservation": 256,
      "portMappings": [
        { "containerPort": 2000, "protocol": "udp" }
      ],
      "logConfiguration": {
        "logDriver": "awsfirelens",
        "options": {
          "Name": "cloudwatch_logs",
          "region": "us-east-1",
          "log_group_name": "/ecs/api-service",
          "log_stream_prefix": "xray/"
        }
      }
    },
    {
      "name": "api",
      "image": "my-org/api:latest",
      "essential": true,
      "cpu": 512,
      "memoryReservation": 1024,
      "portMappings": [
        { "containerPort": 3000, "protocol": "tcp" }
      ],
      "environment": [
        { "name": "AWS_XRAY_DAEMON_ADDRESS", "value": "localhost:2000" },
        { "name": "AWS_REGION", "value": "us-east-1" }
      ],
      "logConfiguration": {
        "logDriver": "awsfirelens",
        "options": {
          "Name": "cloudwatch_logs",
          "region": "us-east-1",
          "log_group_name": "/ecs/api-service",
          "log_stream_prefix": "api/",
          "auto_create_group": "true"
        }
      },
      "dependsOn": [
        { "containerName": "log-router", "condition": "START" },
        { "containerName": "xray-daemon", "condition": "START" }
      ]
    }
  ]
}
```

### CDK equivalente

```typescript
import { Stack, StackProps, RemovalPolicy } from 'aws-cdk-lib';
import * as ecs from 'aws-cdk-lib/aws-ecs';
import * as iam from 'aws-cdk-lib/aws-iam';
import * as logs from 'aws-cdk-lib/aws-logs';
import * as firehose from 'aws-cdk-lib/aws-kinesisfirehose';
import { Construct } from 'constructs';

export class ObservableServiceStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    // ─── Cluster com Container Insights enhanced ───────────────────────────
    const cluster = new ecs.Cluster(this, 'Cluster', {
      vpc: /* vpc */,
      containerInsights: true,   // habilita Container Insights (enhanced via CLI ou console)
    });

    // ─── Task Role com permissões para X-Ray ──────────────────────────────
    const taskRole = new iam.Role(this, 'TaskRole', {
      assumedBy: new iam.ServicePrincipal('ecs-tasks.amazonaws.com'),
    });
    taskRole.addManagedPolicy(
      iam.ManagedPolicy.fromAwsManagedPolicyName('AWSXRayDaemonWriteAccess')
    );

    // Permissão para FireLens enviar ao CloudWatch e Firehose
    const execRole = new iam.Role(this, 'ExecRole', {
      assumedBy: new iam.ServicePrincipal('ecs-tasks.amazonaws.com'),
      managedPolicies: [
        iam.ManagedPolicy.fromAwsManagedPolicyName(
          'service-role/AmazonECSTaskExecutionRolePolicy'
        ),
      ],
    });
    execRole.addToPolicy(new iam.PolicyStatement({
      actions: [
        'logs:CreateLogGroup',
        'logs:CreateLogStream',
        'logs:PutLogEvents',
        'firehose:PutRecordBatch',
      ],
      resources: ['*'],
    }));

    // ─── Task Definition ──────────────────────────────────────────────────
    const taskDef = new ecs.FargateTaskDefinition(this, 'TaskDef', {
      cpu: 1024,
      memoryLimitMiB: 2048,
      taskRole,
      executionRole: execRole,
    });

    // Log Group para operação
    const logGroup = new logs.LogGroup(this, 'AppLogs', {
      logGroupName: '/ecs/api-service',
      retention: logs.RetentionDays.ONE_MONTH,
      removalPolicy: RemovalPolicy.DESTROY,
    });

    // 1. FireLens sidecar
    const fireLensContainer = taskDef.addFirelensLogRouter('log-router', {
      image: ecs.ContainerImage.fromRegistry(
        'public.ecr.aws/aws-observability/aws-for-fluent-bit:stable'
      ),
      firelensConfig: {
        type: ecs.FirelensLogRouterType.FLUENTBIT,
        options: {
          enableECSLogMetadata: true,
        },
      },
      essential: false,
      logging: ecs.LogDrivers.awsLogs({
        streamPrefix: 'firelens',
        logGroup,
        mode: ecs.AwsLogDriverMode.NON_BLOCKING,
      }),
    });

    // 2. X-Ray sidecar
    const xrayContainer = taskDef.addContainer('xray-daemon', {
      image: ecs.ContainerImage.fromRegistry('public.ecr.aws/xray/aws-xray-daemon:3.x'),
      essential: false,
      cpu: 32,
      memoryReservationMiB: 256,
      logging: ecs.LogDrivers.firelens({
        options: {
          Name: 'cloudwatch_logs',
          region: 'us-east-1',
          log_group_name: logGroup.logGroupName,
          log_stream_prefix: 'xray/',
        },
      }),
    });
    xrayContainer.addPortMappings({ containerPort: 2000, protocol: ecs.Protocol.UDP });

    // 3. Container principal com FireLens + X-Ray
    const appContainer = taskDef.addContainer('api', {
      image: ecs.ContainerImage.fromRegistry('my-org/api:latest'),
      essential: true,
      cpu: 512,
      memoryReservationMiB: 1024,
      environment: {
        AWS_XRAY_DAEMON_ADDRESS: 'localhost:2000',
        AWS_REGION: 'us-east-1',
      },
      // Logs via FireLens → CloudWatch E Firehose simultaneamente
      logging: ecs.LogDrivers.firelens({
        options: {
          // Output primário: CloudWatch
          Name: 'cloudwatch_logs',
          region: 'us-east-1',
          log_group_name: logGroup.logGroupName,
          log_stream_prefix: 'api/',
          // Para múltiplos outputs, use config customizada no sidecar FireLens
        },
      }),
    });
    appContainer.addPortMappings({ containerPort: 3000 });
    appContainer.addContainerDependencies(
      { container: fireLensContainer, condition: ecs.ContainerDependencyCondition.START },
      { container: xrayContainer, condition: ecs.ContainerDependencyCondition.START }
    );
  }
}
```

### Validando a observabilidade após o deploy

```bash
# 1. Verificar logs chegando ao CloudWatch
aws logs tail /ecs/api-service --follow

# 2. Verificar Container Insights metrics
aws cloudwatch get-metric-statistics \
  --namespace ECS/ContainerInsights \
  --metric-name CpuUtilized \
  --dimensions Name=ClusterName,Value=prod-cluster \
               Name=ServiceName,Value=api-service \
  --start-time 2026-06-10T10:00:00Z \
  --end-time 2026-06-10T11:00:00Z \
  --period 60 \
  --statistics Average

# 3. Verificar traces X-Ray
aws xray get-trace-summaries \
  --start-time 2026-06-10T10:00:00 \
  --end-time 2026-06-10T11:00:00 \
  --query 'TraceSummaries[*].{Id:Id,Duration:Duration,HasError:HasError}'
```

---

## Armadilhas comuns

### Armadilha 1: FireLens com `essential: true` derrubando a task se o router falhar

**O erro:** O container FireLens está configurado com `essential: true`. Em produção, o Fluent Bit sidecar fica sem memória (processa um pico de logs maior que o configurado) e é killed pelo OOM killer. Como é essential, a task inteira é encerrada — incluindo o container da aplicação que estava saudável. O serviço começa a reiniciar tasks em loop.

**Por que acontece:** O design padrão do FireLens é `essential: false` exatamente para esse cenário: se o router de logs falhar, é preferível continuar servindo tráfego (com logs perdidos temporariamente) do que derrubar o serviço.

**Como reconhecer:** Tasks parando com `StoppedReason: Essential container in task exited` e o container que saiu é o `log-router` ou similar. Nos logs do FireLens (via CloudWatch — irônico), você pode ver mensagens de OOM ou crash antes do encerramento.

**Como evitar:** Sempre configure o FireLens sidecar com `essential: false`. Dimensione adequadamente a memória do container FireLens (`memoryReservation` de 256 MiB é o mínimo; para alta volumetria de logs, use 512 MiB ou mais). Configure `mem_buf_limit` no Fluent Bit para limitar o buffer em memória e evitar OOM.

---

### Armadilha 2: X-Ray sem permissões na Task Role, mas com `essential: false` mascarando o erro

**O erro:** O X-Ray daemon está configurado com `essential: false` e sem a permissão `xray:PutTraceSegments` na Task Role. O daemon inicia, tenta enviar traces, recebe `AccessDenied`, e exibe erros nos seus próprios logs — mas como é `essential: false`, a task não cai. O desenvolvedor não percebe que o tracing está quebrado por dias ou semanas, até alguém precisar dos dados de trace para debugar um problema de produção.

**Por que acontece:** `essential: false` desacopla a saúde do daemon da saúde da task. Erros silenciosos no sidecar não se propagam para os health checks do serviço.

**Como reconhecer:** No console X-Ray, zero traces para o serviço (ou gaps nos traces). Nos logs do X-Ray daemon (`/ecs/xray-daemon`), mensagens como `Send failed with StatusCode: 403` ou `AccessDeniedException`.

**Como evitar:** Adicione um alarme CloudWatch monitorando `xray:PutTraceSegments` via CloudTrail ou monitore a métrica `TracesReceivedCount` no X-Ray. Teste explicitamente o tracing após cada deploy: `aws xray get-trace-summaries` deve retornar traces recentes.

---

### Armadilha 3: `enable-ecs-log-metadata: false` perdendo contexto de correlação

**O erro:** A configuração do FireLens não inclui `enable-ecs-log-metadata: true` (ou está explicitamente como `false`). Os logs chegam ao CloudWatch, mas sem os campos `ecs_task_arn`, `ecs_cluster`, `container_name`. Quando um erro ocorre, você tem o log com a mensagem de erro, mas não sabe de qual task específica veio — dificultando a correlação com métricas, traces e eventos do ECS.

**Por que acontece:** `enable-ecs-log-metadata` não é habilitado por padrão em todas as configurações. Quando omitido, o FireLens não injeta os metadados de contexto do ECS nos logs.

**Como evitar:** Sempre inclua `"enable-ecs-log-metadata": "true"` nas opções do `firelensConfiguration`. Isso é especialmente crítico em ambientes multi-task onde múltiplas réplicas geram logs no mesmo log group — sem o `ecs_task_arn`, é impossível filtrar logs de uma task específica para correlacionar com um incidente.

---

## Exercício de reflexão

Você está projetando a estratégia de observabilidade para um sistema de e-commerce com 8 microserviços no ECS Fargate. Cada serviço gera em média 50 MB de logs por hora em operação normal, chegando a 500 MB/hora nos picos. Os requisitos são:

1. Logs devem ser retidos por 30 dias para operação (CloudWatch) e por 1 ano para compliance (S3).
2. Traces distribuídos devem correlacionar requests entre os 8 serviços.
3. Alertas devem disparar se a taxa de erro de qualquer serviço ultrapassar 1% por 5 minutos.
4. O custo de observabilidade deve ser justificável.

Como você estruturaria o FireLens para atender os requisitos 1 e 4 simultaneamente (CloudWatch para 30 dias + S3 para 1 ano via Firehose, sem duplicar o custo de ingestão)? Para o requisito 2, o X-Ray daemon sozinho é suficiente ou você precisaria de algo adicional para correlacionar traces entre serviços que se chamam via Cloud Map? E para o requisito 3, você usaria métricas do Container Insights, métricas do ALB, ou logs do FireLens como fonte de dados para os alarmes?

---

## Recursos para aprofundar

### 1. Send Amazon ECS logs to an AWS service or AWS Partner (FireLens)
**URL:** https://docs.aws.amazon.com/AmazonECS/latest/developerguide/using_firelens.html
**O que encontrar:** Visão geral do FireLens, como configurar o container sidecar na task definition, as opções do `firelensConfiguration`, e links para exemplos de configuração para cada destino suportado.
**Por que é a fonte certa:** É o ponto de entrada oficial — inclui o diagrama de arquitetura e a lista de imagens oficiais mantidas pela AWS.

### 2. Amazon ECS Container Insights metrics
**URL:** https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Container-Insights-metrics-ECS.html
**O que encontrar:** Tabela completa de todas as métricas publicadas pelo Container Insights para ECS, as dimensões disponíveis (ClusterName, ServiceName, TaskId, ContainerName), e a diferença entre métricas standard e enhanced.
**Por que é a fonte certa:** É a referência de quais métricas existem e com quais dimensões — essencial para criar alarmes e dashboards.

### 3. Running the X-Ray daemon on Amazon ECS
**URL:** https://docs.aws.amazon.com/xray/latest/devguide/xray-daemon-ecs.html
**O que encontrar:** Exemplos completos de task definition com o X-Ray daemon como sidecar, as permissões IAM necessárias na Task Role, variáveis de ambiente para configurar o SDK da aplicação, e como verificar que os traces estão chegando.
**Por que é a fonte certa:** É a documentação canônica do X-Ray para ECS — inclui exemplos de task definition que você pode usar diretamente.

---

