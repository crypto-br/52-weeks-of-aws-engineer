# Sessão session-013-ecs-task-definitions-logging — ECS: Task Definitions — containers, volumes, logging e resource limits

**Duração estimada:** 60 minutos
**Pré-requisitos:** session-004-cdk-v2-setup-bootstrap

---

## Objetivo

Ao final, você conseguirá escrever uma task definition completa com múltiplos containers (sidecar pattern), configurar o awslogs driver para CloudWatch, usar volumes EFS montados em múltiplos containers, e entender os limites de CPU/memory por task vs por container.

---

## Contexto

[FATO] A **Task Definition** é o blueprint imutável de uma task no ECS — o equivalente a um pod spec no Kubernetes. Ela define quais containers rodam juntos, qual imagem cada um usa, quanto CPU e memória cada um recebe, como os logs são roteados, quais volumes são montados e qual IAM role a task assume. Cada vez que você modifica uma task definition, o ECS cria uma nova **revisão** (imutável) com número incrementado. Você faz deploy de uma revisão específica, nunca edita uma existente.

[FATO] Containers em uma mesma task definition compartilham o mesmo ciclo de vida de rede: na modalidade Fargate, cada task tem sua própria ENI com IP privado, e os containers da task se comunicam via `localhost`. Isso é o que torna o **sidecar pattern** eficiente no ECS — o sidecar não precisa atravessar a rede para falar com o container principal, usa loopback.

[CONSENSO] O modelo de task definition do ECS é deliberadamente mais simples que o pod spec do Kubernetes. Ele não tem conceitos como init containers de dependência complexa, readiness/liveness probes declarativas no nível de task (apenas health check de load balancer), ou resource requests distintos de resource limits. Essa simplicidade é tanto uma vantagem (curva de aprendizado menor) quanto uma limitação (menos controle fino em casos avançados).

---

## Conceitos principais

### 1. Estrutura de uma Task Definition

Uma task definition tem dois níveis de configuração: o **nível de task** (configurações que se aplicam à task como um todo) e o **nível de container** (configurações específicas de cada container dentro da task).

```
Task Definition (revisão imutável)
│
├── taskRoleArn        → IAM role que o código dentro dos containers usa
├── executionRoleArn   → IAM role que o ECS agent usa para pull de imagem e logs
├── networkMode        → awsvpc | bridge | host (Fargate: apenas awsvpc)
├── cpu                → CPU total da task (obrigatório no Fargate)
├── memory             → Memória total da task (obrigatório no Fargate)
├── volumes            → Definições de volumes EFS, bind mounts
│
└── containerDefinitions[]
    ├── [0] app container
    │   ├── image, cpu, memory, memoryReservation
    │   ├── portMappings
    │   ├── environment, secrets
    │   ├── logConfiguration (awslogs driver)
    │   ├── mountPoints (referência aos volumes)
    │   ├── healthCheck
    │   └── essential: true
    │
    └── [1] sidecar container
        ├── image, cpu, memory
        ├── logConfiguration
        ├── mountPoints
        └── essential: false   ← task continua se o sidecar falhar
```

[FATO] Dois campos IAM que confundem iniciantes:

- **`taskRoleArn`** (Task Role): identidade IAM dos *processos dentro dos containers*. Se sua app precisa chamar S3, DynamoDB, ou qualquer serviço AWS, é esta role que precisa ter as permissões.
- **`executionRoleArn`** (Execution Role): identidade IAM do *ECS agent*, que roda fora dos containers. É usada para fazer pull da imagem do ECR (`ecr:GetAuthorizationToken`, `ecr:BatchGetImage`) e para enviar logs ao CloudWatch (`logs:CreateLogStream`, `logs:PutLogEvents`). Sem essa role, o container não chega nem a iniciar.

---

### 2. CPU e Memory: task level vs container level

[FATO] No Fargate, CPU e memory **obrigatoriamente** são declarados no nível de task. O Fargate provisiona a capacidade baseado nesses valores. No nível de container, os valores são opcionais e têm semântica diferente:

```
Nível de task (obrigatório no Fargate):
  cpu: 1024    → 1 vCPU alocado para TODA a task
  memory: 2048 → 2 GB alocado para TODA a task

Nível de container (opcional):
  memory: 1536         → hard limit: o container é killed pelo OOM killer se
                         ultrapassar. Deve ser ≤ memory da task.
  memoryReservation: 768  → soft limit: garantia de reserva, mas pode usar mais
                            se houver memória disponível na task.
  cpu: 512             → peso relativo de CPU (CPU shares), não um hard limit.
```

[FATO] A tabela de combinações válidas de CPU × Memory no Fargate (em unidades de CPU e MiB):

| CPU (unidades) | vCPU | Memory (MiB) válida |
|---|---|---|
| 256 | 0.25 | 512, 1024, 2048 |
| 512 | 0.5 | 1024–4096 (incrementos de 1024) |
| 1024 | 1 | 2048–8192 (incrementos de 1024) |
| 2048 | 2 | 4096–16384 (incrementos de 1024) |
| 4096 | 4 | 8192–30720 (incrementos de 1024) |
| 8192 | 8 | 16384–61440 (incrementos de 4096) |
| 16384 | 16 | 32768–122880 (incrementos de 8192) |

[FATO] O `cpu` no nível de container no Fargate é tratado como **CPU shares** (peso relativo), não como alocação hard. Se um container está ocioso, outro pode usar a CPU não usada. Isso é diferente do EC2 launch type, onde o CPU do container é literalmente reservado no host.

[CONSENSO] A recomendação da AWS para Fargate é: defina `cpu` e `memory` no nível de task, e use `memoryReservation` (soft limit) no nível de container apenas se precisar de controle fino sobre como a memória é dividida entre containers. Usar `memory` (hard limit) no container sem entender o OOM killer é uma armadilha frequente — veremos isso nas armadilhas.

---

### 3. awslogs driver: roteamento de logs para CloudWatch

[FATO] O driver `awslogs` é o mecanismo nativo do ECS para enviar stdout/stderr dos containers para CloudWatch Logs. Ele é configurado no campo `logConfiguration` de cada container definition:

```json
{
  "logConfiguration": {
    "logDriver": "awslogs",
    "options": {
      "awslogs-group": "/ecs/my-app",
      "awslogs-region": "us-east-1",
      "awslogs-stream-prefix": "app",
      "awslogs-create-group": "true",
      "mode": "non-blocking",
      "max-buffer-size": "25m"
    }
  }
}
```

[FATO] Cada container dentro de uma task gera um **log stream** separado no CloudWatch Log Group, com o nome no formato:

```
{awslogs-stream-prefix}/{container-name}/{ecs-task-id}
```

Por exemplo: `app/my-app-container/a1b2c3d4-5678-90ab-cdef-1234567890ab`

[FATO] A opção `mode: non-blocking` merece atenção especial. Por padrão, o `awslogs` driver opera em modo **blocking**: se o CloudWatch Logs estiver indisponível ou lento, o container para de escrever em stdout/stderr até o driver conseguir enviar. Em aplicações de alta carga, isso pode causar paradas inexplicáveis. Com `mode: non-blocking` e `max-buffer-size`, o driver usa um buffer em memória e descarta logs se o buffer encher, mas nunca bloqueia o container.

```
Modo blocking (padrão):
  Container → awslogs driver → CloudWatch
             ↑ bloqueia aqui se CWL lento

Modo non-blocking:
  Container → buffer em memória → awslogs driver → CloudWatch
             ↑ nunca bloqueia       ↑ descarta se buffer cheio
```

[FATO] A Execution Role precisa de permissões específicas para o awslogs driver funcionar:

```json
{
  "Effect": "Allow",
  "Action": [
    "logs:CreateLogGroup",
    "logs:CreateLogStream",
    "logs:PutLogEvents",
    "logs:DescribeLogStreams"
  ],
  "Resource": "arn:aws:logs:*:*:*"
}
```

A permissão `logs:CreateLogGroup` só é necessária quando `awslogs-create-group: true`. Em produção, crie o Log Group com retention policy via IaC e remova essa permissão da Execution Role.

---

### 4. Sidecar pattern no ECS

[FATO] O sidecar pattern coloca um container auxiliar na mesma task para separar responsabilidades sem acoplamento de código. No ECS, o sidecar compartilha a mesma rede (loopback) e pode compartilhar volumes com o container principal.

Casos de uso comuns de sidecar no ECS:

```
┌─────────────────────────────────────────────────────────┐
│  Task (shared network namespace: localhost)             │
│                                                         │
│  ┌──────────────────┐    ┌──────────────────────────┐  │
│  │  App container   │    │  Sidecar container        │  │
│  │  (essential:true)│    │  (essential:false)        │  │
│  │                  │    │                           │  │
│  │  app:8080        │◀───│  nginx reverse proxy      │  │
│  │                  │    │  (TLS termination):443    │  │
│  └──────────────────┘    └──────────────────────────┘  │
│                                                         │
│  ┌──────────────────┐    ┌──────────────────────────┐  │
│  │  App container   │    │  CloudWatch Agent sidecar │  │
│  │  escrevendo      │    │  lendo métricas via       │  │
│  │  métricas EMF    │───▶│  UDP:25888                │  │
│  │  para UDP        │    │  e enviando para CWL      │  │
│  └──────────────────┘    └──────────────────────────┘  │
│                                                         │
│  ┌──────────────────┐    ┌──────────────────────────┐  │
│  │  App container   │    │  AWS X-Ray daemon         │  │
│  │  enviando traces │    │  coletando traces via     │  │
│  │  para UDP:2000   │───▶│  UDP:2000                 │  │
│  └────────┬─────────┘    └──────────────────────────┘  │
│           │                                             │
│      volume EFS                                         │
│      compartilhado                                      │
└─────────────────────────────────────────────────────────┘
```

[FATO] O campo `essential` controla o comportamento quando um container para:

- `essential: true` → se este container parar (exit code qualquer), a task inteira é parada e todos os outros containers são encerrados.
- `essential: false` → se este container parar, a task continua rodando. Útil para sidecars de coleta de métricas ou inicialização.

[FATO] O campo `dependsOn` permite definir ordem de inicialização entre containers:

```json
{
  "name": "app",
  "dependsOn": [
    {
      "containerName": "xray-daemon",
      "condition": "START"   // aguarda xray iniciar antes de subir app
    }
  ]
}
```

Condições disponíveis: `START` (container iniciou), `COMPLETE` (container terminou com sucesso — útil para init containers), `SUCCESS` (terminou com exit code 0), `HEALTHY` (passou no health check).

---

### 5. Volumes EFS em Task Definitions

[FATO] Volumes EFS em task definitions têm dois componentes: a **definição do volume** no nível de task (quais EFS file systems e access points usar) e os **mount points** no nível de container (onde montar cada volume dentro do container).

```json
{
  "volumes": [
    {
      "name": "shared-data",
      "efsVolumeConfiguration": {
        "fileSystemId": "fs-0abc1234",
        "rootDirectory": "/",
        "transitEncryption": "ENABLED",
        "authorizationConfig": {
          "accessPointId": "fsap-0abc1234",
          "iam": "ENABLED"
        }
      }
    }
  ],
  "containerDefinitions": [
    {
      "name": "writer",
      "mountPoints": [
        {
          "sourceVolume": "shared-data",
          "containerPath": "/data/output",
          "readOnly": false
        }
      ]
    },
    {
      "name": "reader",
      "mountPoints": [
        {
          "sourceVolume": "shared-data",
          "containerPath": "/data/input",
          "readOnly": true
        }
      ]
    }
  ]
}
```

[FATO] Para que o Fargate consiga montar um volume EFS, três requisitos devem ser atendidos:

1. **Platform version 1.4.0 ou superior** (no Fargate Linux). A versão 1.4.0 foi a que adicionou suporte a EFS.
2. **Security groups compatíveis**: o security group da task deve ter acesso de saída na porta 2049 (NFS) para o security group do EFS mount target.
3. **Task Role com permissão EFS**: se `iam: ENABLED` na `authorizationConfig`, a Task Role precisa de `elasticfilesystem:ClientMount` (e `ClientWrite` para escrita).

[FATO] Armazenamento efêmero no Fargate: por padrão, cada task Fargate tem 20 GB de armazenamento efêmero (para imagens de container + dados temporários). Você pode aumentar até 200 GB via `ephemeralStorage.sizeInGiB`. Esse storage é destruído quando a task para — não persiste.

---

## Exemplo prático

**Cenário:** Uma API Node.js que precisa de:
- Um sidecar AWS X-Ray daemon para distributed tracing.
- Logs estruturados enviados para CloudWatch via `awslogs` em modo non-blocking.
- Um volume EFS montado para armazenar uploads de usuário de forma persistente.

### Task Definition em JSON (formato ECS RegisterTaskDefinition)

```json
{
  "family": "api-service",
  "taskRoleArn": "arn:aws:iam::123456789012:role/api-task-role",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecs-execution-role",
  "networkMode": "awsvpc",
  "cpu": "512",
  "memory": "1024",
  "requiresCompatibilities": ["FARGATE"],
  "volumes": [
    {
      "name": "user-uploads",
      "efsVolumeConfiguration": {
        "fileSystemId": "fs-0abc1234",
        "transitEncryption": "ENABLED",
        "authorizationConfig": {
          "accessPointId": "fsap-0abc1234",
          "iam": "ENABLED"
        }
      }
    }
  ],
  "containerDefinitions": [
    {
      "name": "api",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/api:v1.2.3",
      "essential": true,
      "cpu": 384,
      "memoryReservation": 768,
      "portMappings": [
        { "containerPort": 3000, "protocol": "tcp" }
      ],
      "environment": [
        { "name": "NODE_ENV", "value": "production" },
        { "name": "AWS_XRAY_DAEMON_ADDRESS", "value": "localhost:2000" }
      ],
      "secrets": [
        {
          "name": "DATABASE_URL",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:prod/db-url"
        }
      ],
      "mountPoints": [
        {
          "sourceVolume": "user-uploads",
          "containerPath": "/app/uploads",
          "readOnly": false
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/api-service",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "api",
          "mode": "non-blocking",
          "max-buffer-size": "25m"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:3000/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      },
      "dependsOn": [
        { "containerName": "xray-daemon", "condition": "START" }
      ]
    },
    {
      "name": "xray-daemon",
      "image": "public.ecr.aws/xray/aws-xray-daemon:latest",
      "essential": false,
      "cpu": 64,
      "memoryReservation": 128,
      "portMappings": [
        { "containerPort": 2000, "protocol": "udp" }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/api-service",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "xray",
          "mode": "non-blocking",
          "max-buffer-size": "5m"
        }
      }
    }
  ]
}
```

### Equivalente CDK (TypeScript)

```typescript
import { Stack, StackProps, Duration, RemovalPolicy } from 'aws-cdk-lib';
import * as ecs from 'aws-cdk-lib/aws-ecs';
import * as iam from 'aws-cdk-lib/aws-iam';
import * as logs from 'aws-cdk-lib/aws-logs';
import * as secretsmanager from 'aws-cdk-lib/aws-secretsmanager';
import * as efs from 'aws-cdk-lib/aws-efs';
import { Construct } from 'constructs';

export class ApiTaskStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    // Log Group com retention explícita (evita acumular logs indefinidamente)
    const logGroup = new logs.LogGroup(this, 'LogGroup', {
      logGroupName: '/ecs/api-service',
      retention: logs.RetentionDays.ONE_MONTH,
      removalPolicy: RemovalPolicy.DESTROY,
    });

    // Task Role: permissões para o código da aplicação
    const taskRole = new iam.Role(this, 'TaskRole', {
      assumedBy: new iam.ServicePrincipal('ecs-tasks.amazonaws.com'),
    });
    taskRole.addManagedPolicy(
      iam.ManagedPolicy.fromAwsManagedPolicyName('AWSXRayDaemonWriteAccess')
    );
    // taskRole.addToPolicy(new iam.PolicyStatement({ ... s3, dynamodb, etc. }));

    // Task Definition
    const taskDef = new ecs.FargateTaskDefinition(this, 'TaskDef', {
      cpu: 512,
      memoryLimitMiB: 1024,
      taskRole,
      // executionRole é criada automaticamente pelo CDK com as permissões mínimas
    });

    // Volume EFS
    const fileSystem = efs.FileSystem.fromFileSystemAttributes(this, 'EFS', {
      fileSystemId: 'fs-0abc1234',
      securityGroup: /* sg do EFS */ undefined as any,
    });
    const accessPoint = new efs.AccessPoint(this, 'AccessPoint', {
      fileSystem,
      path: '/uploads',
      posixUser: { uid: '1000', gid: '1000' },
      createAcl: { ownerUid: '1000', ownerGid: '1000', permissions: '755' },
    });

    // Adicionar volume à task definition
    taskDef.addVolume({
      name: 'user-uploads',
      efsVolumeConfiguration: {
        fileSystemId: fileSystem.fileSystemId,
        transitEncryption: 'ENABLED',
        authorizationConfig: {
          accessPointId: accessPoint.accessPointId,
          iam: 'ENABLED',
        },
      },
    });

    // Sidecar X-Ray
    const xrayContainer = taskDef.addContainer('xray-daemon', {
      image: ecs.ContainerImage.fromRegistry('public.ecr.aws/xray/aws-xray-daemon:latest'),
      cpu: 64,
      memoryReservationMiB: 128,
      essential: false,
      logging: ecs.LogDrivers.awsLogs({
        streamPrefix: 'xray',
        logGroup,
        mode: ecs.AwsLogDriverMode.NON_BLOCKING,
        maxBufferSize: cdk.Size.mebibytes(5),
      }),
    });
    xrayContainer.addPortMappings({ containerPort: 2000, protocol: ecs.Protocol.UDP });

    // Container principal
    const appContainer = taskDef.addContainer('api', {
      image: ecs.ContainerImage.fromEcrRepository(/* repo */ undefined as any, 'v1.2.3'),
      cpu: 384,
      memoryReservationMiB: 768,
      essential: true,
      environment: {
        NODE_ENV: 'production',
        AWS_XRAY_DAEMON_ADDRESS: 'localhost:2000',
      },
      secrets: {
        DATABASE_URL: ecs.Secret.fromSecretsManager(
          secretsmanager.Secret.fromSecretNameV2(this, 'DbSecret', 'prod/db-url')
        ),
      },
      logging: ecs.LogDrivers.awsLogs({
        streamPrefix: 'api',
        logGroup,
        mode: ecs.AwsLogDriverMode.NON_BLOCKING,
        maxBufferSize: cdk.Size.mebibytes(25),
      }),
      healthCheck: {
        command: ['CMD-SHELL', 'curl -f http://localhost:3000/health || exit 1'],
        interval: Duration.seconds(30),
        timeout: Duration.seconds(5),
        retries: 3,
        startPeriod: Duration.seconds(60),
      },
    });
    appContainer.addPortMappings({ containerPort: 3000 });
    appContainer.addMountPoints({
      sourceVolume: 'user-uploads',
      containerPath: '/app/uploads',
      readOnly: false,
    });
    appContainer.addContainerDependencies({
      container: xrayContainer,
      condition: ecs.ContainerDependencyCondition.START,
    });

    // Permite que a task role acesse o EFS
    fileSystem.grantRootAccess(taskRole);
  }
}
```

---

## Armadilhas comuns

### Armadilha 1: Confundir Task Role com Execution Role

**O erro:** A aplicação precisa ler um segredo do Secrets Manager. Você adiciona a permissão `secretsmanager:GetSecretValue` na **Execution Role** em vez da **Task Role**. O container inicia normalmente (a Execution Role tem o que precisa para pull de imagem e logs), mas quando o código tenta chamar `GetSecretValue`, recebe `AccessDenied`.

**Por que acontece:** A Execution Role é usada pelo agente ECS **antes** do container iniciar (para puxar a imagem e configurar logs). A Task Role é injetada como credenciais dentro do container via o endpoint de metadata (169.254.170.2). São dois contextos de execução completamente diferentes.

**Como reconhecer:** Erros de `AccessDenied` em chamadas de SDK AWS dentro do código da aplicação, enquanto o container inicia sem problemas. O log de inicialização do ECS não mostra erros.

**Como evitar:** Regra mnemônica: "Execution Role = o ECS precisa para fazer o container nascer. Task Role = o código dentro do container precisa para fazer seu trabalho."

---

### Armadilha 2: OOM kill silencioso por hard limit no container

**O erro:** Você define `memory: 512` (hard limit) para o container principal em uma task com `memory: 1024`. Em carga alta, o processo Node.js ultrapassa 512 MiB. O OOM killer do Linux mata o processo. O ECS registra o container como terminado com `OOMKilled: true` no evento, mas a task definition marca o container como `essential: true`, então a task inteira é parada e imediatamente reiniciada. O resultado é um loop de crashes silenciosos que parece para o usuário como "serviço instável".

**Por que acontece:** O hard limit de memória no container (`memory`) é aplicado pelo kernel do Linux via cgroups. Quando o processo ultrapassa o limite, é killed sem aviso. O ECS reinicia a task automaticamente, criando o loop.

**Como reconhecer:** Eventos de `StoppedReason: Essential container in task exited` no console ECS com frequência. CloudWatch Metrics mostrando picos de MemoryUtilization seguidos de quedas abruptas para zero. `aws ecs describe-tasks` mostrando `exitCode: 137` (sinal 9 = SIGKILL do OOM killer).

**Como evitar:** Prefira `memoryReservation` (soft limit) em vez de `memory` (hard limit) no nível de container. Use métricas de CloudWatch Container Insights para descobrir o P99 de uso de memória e dimensionar o hard limit com margem de segurança de 20-30%.

---

### Armadilha 3: awslogs bloqueando o container em modo padrão (blocking)

**O erro:** Seu serviço ECS sofre uma latência de CloudWatch Logs (região sobrecarregada, brief outage). O `awslogs` driver em modo blocking para todos os containers que tentam escrever em stdout. A aplicação trava em `console.log()` ou equivalente. O ALB health check falha. O ECS substitui a task. O log do problema some porque o driver não conseguiu enviá-lo.

**Por que acontece:** O `awslogs` driver no modo padrão (blocking) faz backpressure no processo quando não consegue enviar para CloudWatch. É um comportamento correto de throughput, mas catastrófico para disponibilidade.

**Como evitar:** Sempre configure `mode: non-blocking` e `max-buffer-size: 25m` (ou valor adequado à sua workload) em todos os containers de produção. Aceite que em caso de indisponibilidade do CloudWatch você pode perder alguns logs — isso é preferível a perder disponibilidade.

---

## Exercício de reflexão

Você está projetando uma task definition para um serviço de processamento de imagens: um container principal que recebe imagens, as processa com FFmpeg (intensivo em CPU), e grava os resultados em um volume EFS. Um segundo container lê os resultados processados e faz upload para S3, agindo como sidecar de exportação. A task roda no Fargate.

Como você dividiria os recursos (CPU e memory) entre os dois containers e no nível de task? O sidecar de exportação seria `essential: true` ou `false`? Você usaria `dependsOn` e com qual condição? Como configuraria os logs de cada container (mesmo Log Group ou grupos separados)? E se o processo FFmpeg for instável e abortar com exit code 1 frequentemente — que campo de health check ou dependência você usaria para detectar isso antes que o ECS marque a task como unhealthy?

---

## Recursos para aprofundar

### 1. Amazon ECS task definition parameters (Fargate)
**URL:** https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definition_parameters.html
**O que encontrar:** Referência completa de todos os campos de uma task definition para Fargate: container definitions, volumes, network mode, resource limits. É o documento que você consulta quando não tem certeza de um campo específico.
**Por que é a fonte certa:** É a referência canônica da AWS — mais precisa que qualquer tutorial.

### 2. Using the awslogs log driver
**URL:** https://docs.aws.amazon.com/AmazonECS/latest/developerguide/using_awslogs.html
**O que encontrar:** Configuração completa do `awslogs` driver, incluindo opções de modo non-blocking, multi-line log patterns para capturar stack traces em uma única entrada, e as permissões IAM necessárias na Execution Role.
**Por que é a fonte certa:** Documenta todas as opções do driver, incluindo as menos óbvias como `awslogs-multiline-pattern` que é essencial para stack traces de Java/Python.

### 3. ECS Best Practices Guide
**URL:** https://docs.aws.amazon.com/AmazonECS/latest/bestpracticesguide/intro.html
**O que encontrar:** Capítulos separados sobre sizing de tasks, padrões de logging, sidecar patterns, e uso de volumes persistentes. É mais prescritivo que o developer guide — diz *o que fazer*, não apenas *como funciona*.
**Por que é a fonte certa:** É o guia de boas práticas oficial da equipe ECS, consolidando lições de usuários em produção, e cobre trade-offs que o developer guide não discute explicitamente.

---

