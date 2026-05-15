# Sessão session-015-ecs-fargate-networking-iam — ECS Fargate: networking, security groups e IAM Roles for Tasks

**Duração estimada:** 60 minutos
**Pré-requisitos:** session-014-ecs-services-discovery-alb

---

## Objetivo

Ao final, você conseguirá explicar o modelo de networking `awsvpc` do Fargate (cada task tem uma ENI própria), configurar security groups no nível da task, criar uma task IAM role com permissões mínimas e verificar via `aws sts get-caller-identity` dentro do container.

---

## Contexto

[FATO] Antes do `awsvpc` network mode (introduzido em 2017), containers ECS em EC2 compartilhavam a interface de rede do host. Isso significava que security groups só podiam ser aplicados no nível da instância EC2, não por container — um modelo grosseiro que tornava a segmentação de rede por serviço impossível sem instâncias dedicadas. O Fargate, lançado no mesmo ano, adotou `awsvpc` como o único modo disponível e tornou o isolamento de rede granular o padrão.

[FATO] O `awsvpc` dá a cada task ECS a mesma posição de rede de uma instância EC2: IP privado próprio, security groups próprios, ENI gerenciada pela AWS, e presença na tabela de roteamento da VPC. Do ponto de vista da rede, uma task Fargate é indistinguível de uma instância EC2 pequena.

[CONSENSO] O modelo de credenciais do Fargate é frequentemente mal compreendido por equipes que vêm de ambientes EC2 tradicionais. No EC2, credenciais de instância chegam via IMDS (`169.254.169.254`). No Fargate e em ECS com `awsvpc`, credenciais de task chegam via um endpoint diferente (`169.254.170.2`), gerenciado pelo ECS agent. Entender essa distinção é essencial para depurar `AccessDenied` em containers e para configurar IAM corretamente.

---

## Conceitos principais

### 1. awsvpc: uma ENI por task

[FATO] Quando o ECS inicia uma task Fargate, ele provisiona automaticamente uma **Elastic Network Interface (ENI)** na subnet especificada. Essa ENI recebe:

- Um IP privado primário (da faixa CIDR da subnet).
- Os security groups especificados na configuração de rede do service ou task.
- Opcionalmente, um IP público (se `assignPublicIp: ENABLED` e a subnet for pública).

```
VPC (10.0.0.0/16)
│
├── Subnet privada A (10.0.1.0/24)
│   ├── ENI: 10.0.1.5  → Task 1 do api-service  [sg-api]
│   ├── ENI: 10.0.1.8  → Task 2 do api-service  [sg-api]
│   └── ENI: 10.0.1.12 → Task 1 do db-migrator  [sg-migrator]
│
└── Subnet privada B (10.0.2.0/24)
    ├── ENI: 10.0.2.3  → Task 3 do api-service  [sg-api]
    └── ENI: 10.0.2.7  → Task 1 do worker       [sg-worker]
```

[FATO] Implicações operacionais importantes do modelo uma-ENI-por-task:

1. **ENI limit por conta/região**: cada tipo de instância EC2 tem um limite de ENIs por host, mas no Fargate o limite relevante é o limite de ENIs por subnet/VPC. O padrão é 5.000 ENIs por VPC. Para serviços com alta escala automática, esse limite pode ser atingido.

2. **VPC Flow Logs por task**: como cada task tem sua própria ENI, o VPC Flow Logs registra tráfego no nível de task. Você pode filtrar por IP para ver exatamente o tráfego de uma task específica — algo impossível no modelo bridge/host.

3. **Security groups granulares**: o security group é atribuído à ENI da task, não ao host. Dois serviços diferentes rodando em Fargate podem ter security groups completamente diferentes sem nenhuma relação entre si.

4. **Containers na mesma task compartilham a ENI**: todos os containers dentro de uma task compartilham o mesmo espaço de rede (mesma ENI, mesmo IP). A comunicação entre containers da mesma task é via `localhost`, não via IP da ENI.

---

### 2. Security groups no nível de task

[FATO] Security groups em ECS Fargate funcionam exatamente como em EC2: stateful, com regras de inbound e outbound separadas, referenciáveis entre si. A diferença é que você os aplica ao service/task em vez de à instância.

**Modelo recomendado para microserviços:**

```
┌─────────────┐    sg-alb         ┌─────────────┐
│     ALB     │──────────────────▶│  api-service│  sg-api
│  (sg-alb)   │                   │             │
└─────────────┘                   └──────┬──────┘
                                         │
                              sg-api → sg-db (5432)
                                         │
                                  ┌──────▼──────┐
                                  │  RDS Aurora │  sg-db
                                  └─────────────┘

Regras:
sg-api inbound:
  - port 3000 from sg-alb       (ALB fala com a task)

sg-api outbound:
  - port 443 to 0.0.0.0/0      (HTTPS para AWS APIs)
  - port 5432 to sg-db          (PostgreSQL para RDS)

sg-db inbound:
  - port 5432 from sg-api       (apenas api-service acessa o banco)
```

[CONSENSO] Referenciar security groups entre si (em vez de usar CIDRs) é a abordagem correta em VPCs com muitos serviços. Quando você usa `source: sg-api` em uma regra do `sg-db`, a regra automaticamente inclui qualquer nova task do api-service sem precisar atualizar o CIDR. É dinâmico por definição.

[FATO] Uma regra de outbound crítica que frequentemente é esquecida: para tasks em **subnets privadas**, você precisa de acesso de saída para chamar APIs AWS (ECR, CloudWatch, Secrets Manager, S3). As opções são:

```
Opção 1: NAT Gateway
  Task (privada) → NAT GW (pública) → Internet → AWS API
  Custo: ~$32/mês fixo + $0.045/GB processado
  Simplicidade: máxima (um único ponto de saída)

Opção 2: VPC Endpoints (Interface + Gateway)
  Task (privada) → VPC Endpoint ENI → AWS API (via PrivateLink)
  Custo: ~$7.30/mês por endpoint por AZ + $0.01/GB
  Segurança: tráfego nunca sai da rede AWS
  Complexidade: precisa criar endpoints individuais por serviço

Opção 3: Subnet pública com IP público
  Task (pública, assignPublicIp=ENABLED) → Internet Gateway → AWS API
  Custo: mínimo (IGW é gratuito)
  Segurança: task exposta à internet diretamente (evitar em produção)
```

[FATO] Para Fargate em subnets privadas, os VPC endpoints mínimos necessários para iniciar uma task são:

| Endpoint | Tipo | Para que serve |
|---|---|---|
| `com.amazonaws.REGION.ecr.api` | Interface | Pull de manifesto de imagem |
| `com.amazonaws.REGION.ecr.dkr` | Interface | Pull de layers de imagem |
| `com.amazonaws.REGION.s3` | Gateway | Download de layers do S3 (ECR usa S3) |
| `com.amazonaws.REGION.logs` | Interface | CloudWatch Logs (awslogs driver) |
| `com.amazonaws.REGION.secretsmanager` | Interface | Se usar Secrets Manager |
| `com.amazonaws.REGION.ssm` | Interface | Se usar Parameter Store |

---

### 3. Task IAM Role: como as credenciais chegam ao container

[FATO] O fluxo de credenciais IAM dentro de um container Fargate é gerenciado pelo ECS agent e funciona assim:

```
┌─────────────────────────────────────────────────────────────┐
│  Processo de injeção de credenciais (iniciado pelo ECS)     │
│                                                             │
│  1. Task é iniciada com taskRoleArn definido                │
│  2. ECS agent faz AssumeRole na Task Role                   │
│     → obtém credenciais temporárias (AccessKey + Secret     │
│       + SessionToken), válidas por 6 horas                  │
│  3. ECS agent armazena as credenciais em cache interno      │
│  4. ECS agent seta no container:                            │
│     AWS_CONTAINER_CREDENTIALS_RELATIVE_URI=/v2/credentials/TOKEN │
│                                                             │
│  5. Dentro do container, o AWS SDK faz GET para:            │
│     http://169.254.170.2/v2/credentials/TOKEN               │
│     → recebe credenciais temporárias em JSON                │
│                                                             │
│  6. A cada ~5 horas, o ECS agent renova automaticamente     │
└─────────────────────────────────────────────────────────────┘
```

[FATO] O endpoint `169.254.170.2` é um endereço link-local gerenciado pelo ECS agent, diferente do IMDS do EC2 (`169.254.169.254`). O ECS bloqueia o acesso ao IMDS do EC2 de dentro dos containers Fargate — containers não podem assumir a identidade do host subjacente.

[FATO] Você pode verificar qual identidade está sendo usada dentro de um container com:

```bash
# Dentro do container (via ECS Exec ou durante testes locais):
aws sts get-caller-identity

# Saída esperada para uma task com Task Role configurada:
{
    "UserId": "AROAEXAMPLEID:TASK-ID",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/my-task-role/TASK-ID"
}

# Se não houver Task Role configurada, o SDK retorna erro:
# An error occurred (NoCredentialProviders): no valid credential providers found
```

[FATO] O `ECS Exec` (`aws ecs execute-command`) permite abrir um shell interativo dentro de um container em execução, o que é muito útil para depurar IAM:

```bash
aws ecs execute-command \
  --cluster my-cluster \
  --task <task-id> \
  --container api \
  --interactive \
  --command "/bin/bash"

# Dentro do shell do container:
$ aws sts get-caller-identity
$ curl -s http://169.254.170.2$AWS_CONTAINER_CREDENTIALS_RELATIVE_URI | python3 -m json.tool
```

Para que o ECS Exec funcione, a task definition precisa ter `enableExecuteCommand: true` e a Task Role precisa da permissão `ssmmessages:CreateControlChannel` (entre outras).

---

### 4. Princípio do mínimo privilégio para Task Roles

[CONSENSO] Em ECS, o antipadrão mais comum é criar uma única task role com `AdministratorAccess` ou `PowerUserAccess` e reutilizá-la em todos os serviços. Isso viola o princípio do mínimo privilégio: se um container for comprometido (vulnerabilidade de código, SSRF, etc.), o atacante obtém acesso irrestrito à conta AWS.

[FATO] A política de mínimo privilégio para Task Roles exige:

1. **Uma Task Role por serviço** (não compartilhada entre serviços com funções diferentes).
2. **Recursos específicos por ARN** — nunca `"Resource": "*"` quando o ARN é conhecido em tempo de deploy.
3. **Condition keys** quando disponíveis, para restringir ainda mais.

Exemplo de Task Role para um serviço que lê de um bucket S3 específico e escreve em um DynamoDB específico:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::my-app-bucket/data/*"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "arn:aws:s3:::my-app-bucket",
      "Condition": {
        "StringLike": { "s3:prefix": ["data/*"] }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:UpdateItem",
        "dynamodb:Query"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/my-app-table"
    }
  ]
}
```

Equivalente CDK (usando métodos `grant*` que geram políticas mínimas automaticamente):

```typescript
// CDK gera as permissões mínimas corretas para cada operação
bucket.grantRead(taskRole, 'data/*');
table.grantReadWriteData(taskRole);

// Para Secrets Manager com path específico:
const secret = secretsmanager.Secret.fromSecretNameV2(this, 'DbSecret', 'prod/db');
secret.grantRead(taskRole);
```

---

## Exemplo prático

**Cenário completo:** Um serviço `api-service` em Fargate rodando em subnet privada que precisa:
- Receber tráfego do ALB na porta 3000.
- Chamar DynamoDB e Secrets Manager.
- Fazer pull de imagem do ECR privado.
- Enviar logs para CloudWatch.
- Ser acessível para debug via ECS Exec.

### CDK completo

```typescript
import { Stack, StackProps } from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as ecs from 'aws-cdk-lib/aws-ecs';
import * as iam from 'aws-cdk-lib/aws-iam';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';
import * as secretsmanager from 'aws-cdk-lib/aws-secretsmanager';
import * as ecr from 'aws-cdk-lib/aws-ecr';
import * as logs from 'aws-cdk-lib/aws-logs';
import { Construct } from 'constructs';

export class ApiServiceStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    const vpc = ec2.Vpc.fromLookup(this, 'Vpc', { vpcName: 'prod-vpc' });

    // ─── VPC Endpoints (para subnet privada sem NAT) ──────────────────────
    // Gateway endpoint para S3 (gratuito, necessário para ECR)
    vpc.addGatewayEndpoint('S3Endpoint', {
      service: ec2.GatewayVpcEndpointAwsService.S3,
    });
    // Interface endpoints para ECR, CloudWatch Logs, Secrets Manager
    vpc.addInterfaceEndpoint('EcrApiEndpoint', {
      service: ec2.InterfaceVpcEndpointAwsService.ECR,
    });
    vpc.addInterfaceEndpoint('EcrDkrEndpoint', {
      service: ec2.InterfaceVpcEndpointAwsService.ECR_DOCKER,
    });
    vpc.addInterfaceEndpoint('LogsEndpoint', {
      service: ec2.InterfaceVpcEndpointAwsService.CLOUDWATCH_LOGS,
    });
    vpc.addInterfaceEndpoint('SecretsManagerEndpoint', {
      service: ec2.InterfaceVpcEndpointAwsService.SECRETS_MANAGER,
    });

    // ─── Security Groups ──────────────────────────────────────────────────

    // SG do ALB (gerenciado externamente, referenciado aqui)
    const albSg = ec2.SecurityGroup.fromLookupByName(this, 'AlbSg', 'sg-alb', vpc);

    // SG da task
    const taskSg = new ec2.SecurityGroup(this, 'TaskSg', {
      vpc,
      description: 'Security group para api-service tasks',
      allowAllOutbound: false,   // mínimo privilégio: outbound explícito
    });
    // Inbound: apenas do ALB
    taskSg.addIngressRule(albSg, ec2.Port.tcp(3000), 'Tráfego do ALB');
    // Outbound: HTTPS para VPC endpoints e DynamoDB
    taskSg.addEgressRule(ec2.Peer.anyIpv4(), ec2.Port.tcp(443), 'HTTPS para AWS APIs');

    // ─── Task Role (identidade do código da aplicação) ────────────────────

    const taskRole = new iam.Role(this, 'TaskRole', {
      assumedBy: new iam.ServicePrincipal('ecs-tasks.amazonaws.com'),
      description: 'Task role para api-service — mínimo privilégio',
    });

    // Permissões da aplicação (usando grant* para gerar políticas mínimas)
    const table = dynamodb.Table.fromTableName(this, 'AppTable', 'api-app-table');
    table.grantReadWriteData(taskRole);

    const dbSecret = secretsmanager.Secret.fromSecretNameV2(
      this, 'DbSecret', 'prod/api/db-credentials'
    );
    dbSecret.grantRead(taskRole);

    // Permissões para ECS Exec (debug interativo)
    taskRole.addToPolicy(new iam.PolicyStatement({
      actions: [
        'ssmmessages:CreateControlChannel',
        'ssmmessages:CreateDataChannel',
        'ssmmessages:OpenControlChannel',
        'ssmmessages:OpenDataChannel',
      ],
      resources: ['*'],
    }));

    // ─── Task Definition ──────────────────────────────────────────────────

    const logGroup = new logs.LogGroup(this, 'LogGroup', {
      logGroupName: '/ecs/api-service',
      retention: logs.RetentionDays.ONE_MONTH,
    });

    const repo = ecr.Repository.fromRepositoryName(this, 'Repo', 'api-service');

    const taskDef = new ecs.FargateTaskDefinition(this, 'TaskDef', {
      cpu: 512,
      memoryLimitMiB: 1024,
      taskRole,
      // executionRole criada pelo CDK com permissões para ECR pull e CloudWatch logs
    });

    taskDef.addContainer('api', {
      image: ecs.ContainerImage.fromEcrRepository(repo, 'latest'),
      portMappings: [{ containerPort: 3000 }],
      logging: ecs.LogDrivers.awsLogs({
        streamPrefix: 'api',
        logGroup,
        mode: ecs.AwsLogDriverMode.NON_BLOCKING,
      }),
      secrets: {
        DB_PASSWORD: ecs.Secret.fromSecretsManager(dbSecret, 'password'),
        DB_HOST: ecs.Secret.fromSecretsManager(dbSecret, 'host'),
      },
    });

    // ─── ECS Service ──────────────────────────────────────────────────────

    const cluster = ecs.Cluster.fromClusterAttributes(this, 'Cluster', {
      clusterName: 'prod-cluster',
      vpc,
      securityGroups: [],
    });

    new ecs.FargateService(this, 'Service', {
      cluster,
      taskDefinition: taskDef,
      desiredCount: 3,
      securityGroups: [taskSg],
      // Subnets privadas — sem IP público
      assignPublicIp: false,
      vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS },
      enableExecuteCommand: true,   // habilita ECS Exec para debug
      deploymentCircuitBreaker: { rollback: true },
    });
  }
}
```

### Verificando identidade dentro do container

```bash
# Abrir shell no container em execução
aws ecs execute-command \
  --cluster prod-cluster \
  --task arn:aws:ecs:us-east-1:123456789012:task/prod-cluster/abc123 \
  --container api \
  --interactive \
  --command "/bin/sh"

# Dentro do container, verificar identidade da task role
$ aws sts get-caller-identity
{
    "UserId": "AROAEXAMPLE123:abc123",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/ApiServiceStack-TaskRole/abc123"
}

# Verificar variável de ambiente injetada pelo ECS
$ echo $AWS_CONTAINER_CREDENTIALS_RELATIVE_URI
/v2/credentials/a1b2c3d4-e5f6-7890-abcd-ef1234567890

# Inspecionar as credenciais brutas (útil para debug)
$ curl -s http://169.254.170.2$AWS_CONTAINER_CREDENTIALS_RELATIVE_URI
{
  "RoleArn": "arn:aws:iam::123456789012:role/ApiServiceStack-TaskRole",
  "AccessKeyId": "ASIA...",
  "SecretAccessKey": "...",
  "Token": "...",
  "Expiration": "2026-06-03T18:00:00Z"
}

# Testar acesso ao DynamoDB
$ aws dynamodb describe-table --table-name api-app-table --region us-east-1
```

---

## Armadilhas comuns

### Armadilha 1: `allowAllOutbound: true` no security group da task (padrão CDK)

**O erro:** O CDK cria security groups com `allowAllOutbound: true` por padrão. Para tasks em subnets privadas sem VPC endpoints, isso funciona se houver um NAT gateway — a task acessa qualquer endpoint AWS via NAT. Mas quando você remove o NAT gateway para economizar custos e adiciona VPC endpoints, o tráfego para serviços sem endpoint (ex: STS, EC2 Describe, CloudFormation) continua tentando ir para a internet, falha silenciosamente, e a task não consegue iniciar ou funcionar.

**Por que acontece:** Com `allowAllOutbound: true`, o security group permite o tráfego de saída, mas sem NAT e sem endpoint para o serviço específico, o tráfego não tem para onde ir na VPC e é descartado pela tabela de roteamento.

**Como reconhecer:** Tasks ficam presas no estado `PROVISIONING` ou falham com `CannotPullContainerError` indicando timeout no pull da imagem do ECR. Nos VPC Flow Logs, você vê tráfego REJECTED de saída para IPs públicos da AWS.

**Como evitar:** Use `allowAllOutbound: false` e adicione regras explícitas de outbound. Antes de remover um NAT gateway, audite quais endpoints AWS o serviço chama e crie VPC endpoints correspondentes. O comando `aws ec2 describe-vpc-endpoint-services` lista todos os serviços disponíveis como VPC endpoints na sua região.

---

### Armadilha 2: Task Role sem permissão, mas Execution Role com excesso de permissões

**O erro:** Uma task precisa ler do Secrets Manager. O desenvolvedor não configura `taskRoleArn` e em vez disso adiciona `secretsmanager:GetSecretValue` na Execution Role. A task inicia e consegue injetar o secret na inicialização via `secrets` da task definition (isso funciona, pois é o ECS agent que faz a chamada). Mas o código da aplicação que tenta chamar `GetSecretValue` diretamente (ex: para rotacionar credenciais em runtime) falha com `AccessDenied`, porque as credenciais injetadas no container são da Task Role (vazia), não da Execution Role.

**Por que acontece:** A Execution Role não é injetada no container — ela é usada exclusivamente pelo ECS agent antes e durante a inicialização. Uma vez que o container está rodando, apenas a Task Role é acessível via `AWS_CONTAINER_CREDENTIALS_RELATIVE_URI`.

**Como reconhecer:** O container inicia com sucesso (a injeção de secrets na inicialização funciona via Execution Role), mas chamadas de SDK dentro do código falham com `NoCredentialProviders` ou `AccessDenied` ao tentar acessar qualquer serviço AWS.

**Como evitar:** Regra definitiva — toda permissão que o **código da aplicação** precisa em runtime vai na **Task Role**. Toda permissão que o **ECS** precisa para fazer o container nascer vai na **Execution Role**.

---

### Armadilha 3: ENI limit atingido durante auto scaling

**O erro:** Você tem um serviço com `desiredCount=10` que escala até `maxCount=100` durante picos. Sua VPC tem 3 AZs e subnets privadas com /24 cada (253 IPs utilizáveis). Um auto scaling agressivo cria 100 tasks de uma vez, cada uma ocupando uma ENI em uma das subnets. Com 100 tasks + outras ENIs de serviços existentes + ENIs de instâncias RDS, Lambda ENIs, etc., você pode atingir o limite de IPs disponíveis na subnet ou o limite de ENIs por VPC.

**Por que acontece:** Diferente do EC2 (onde múltiplos containers compartilham uma ENI), no Fargate cada task consome uma ENI com um IP da subnet. Subnets /24 com 253 IPs são limitantes para serviços de alta escala.

**Como reconhecer:** Falhas de scaling com `SERVICE DEPLOYMENT FAILED: Tasks failed to start` e `EniLimitExceeded` ou `InsufficientPrivateIPAddressCapacity` nos eventos do serviço ECS.

**Como evitar:** Dimensione subnets para o pico de tasks × número de AZs com margem de 2x. Para serviços com potencial de dezenas de tasks, use subnets /22 ou maiores. Monitore o CloudWatch metric `SubnetsAvailableIPAddressCount` proativamente com alarmes antes de atingir o limite.

---

## Exercício de reflexão

Você está auditando a segurança de um sistema ECS Fargate existente e encontra a seguinte configuração:

- Todos os serviços usam a mesma Task Role com a policy `PowerUserAccess`.
- Todos os security groups das tasks têm `allowAllOutbound: true`.
- Os serviços rodam em subnets públicas com `assignPublicIp: ENABLED`.
- As tasks não têm `enableExecuteCommand` configurado.

Como você priorizaria as correções? Para cada problema, qual é o risco real e qual é a mudança mínima para mitigá-lo? Ao migrar para subnets privadas, quais VPC endpoints você criaria primeiro para garantir que as tasks continuem funcionando? Se você precisar manter pelo menos um ambiente operacional durante a migração, qual seria a sequência de mudanças?

---

## Recursos para aprofundar

### 1. Amazon ECS task networking (awsvpc mode)
**URL:** https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-networking-awsvpc.html
**O que encontrar:** Documentação completa do awsvpc mode: como ENIs são provisionadas, limites por instância EC2 (relevante para EC2 launch type), configuração de IP público vs privado, e integração com VPC Flow Logs.
**Por que é a fonte certa:** É a referência técnica primária do modelo de networking, mais detalhada que o guia de Fargate específico.

### 2. Amazon ECS task IAM role
**URL:** https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-iam-roles.html
**O que encontrar:** Como configurar a Task Role, o mecanismo de injeção de credenciais via `169.254.170.2`, como verificar credenciais dentro do container, e a diferença explícita entre Task Role e Execution Role.
**Por que é a fonte certa:** É a documentação canônica do mecanismo de credenciais IAM no ECS — o lugar para ir quando você tem dúvidas sobre `AccessDenied` em containers.

### 3. Best practices for connecting Amazon ECS to AWS services from inside your VPC
**URL:** https://docs.aws.amazon.com/AmazonECS/latest/developerguide/networking-connecting-vpc.html
**O que encontrar:** Comparação detalhada entre NAT Gateway, VPC Endpoints, e subnet pública para acesso a serviços AWS a partir de tasks ECS. Inclui tabela de custos e recomendações por caso de uso.
**Por que é a fonte certa:** É o guia de boas práticas oficial que trata exatamente do trade-off mais comum (NAT vs Endpoints), com dados de custo incluídos.

---

