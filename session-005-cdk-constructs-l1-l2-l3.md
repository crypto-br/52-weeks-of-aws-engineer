# Sessão 005 — CDK: Constructs L1, L2, L3 — o que são e como escolher

**Duração estimada:** 60 minutos
**Pré-requisitos:** session-004 — CDK v2: setup, bootstrap e estrutura de projeto

---

## Objetivo

Ao final, você conseguirá distinguir constructs L1 (`CfnBucket`), L2 (`Bucket`) e L3 (patterns como `ApplicationLoadBalancedFargateService`), navegar o Construct Hub para avaliar constructs de terceiros, e explicar por que L2 não é sempre preferível a L1.

---

## Contexto

[FATO] O modelo de três camadas de abstrações (L1/L2/L3) é o núcleo da proposta de valor do CDK. Ele define como a biblioteca de constructs é organizada e determina o nível de controle vs. conveniência em cada situação.

[CONSENSO] A maioria dos engenheiros que começa com CDK adota L2 por padrão e só desce para L1 quando algo não funciona. Essa abordagem funciona, mas deixa lacunas: você acaba não entendendo o que o L2 gera por baixo, o que dificulta debugging e torna as escape hatches misteriosas. Entender os três níveis — e quando cada um é apropriado — é o que separa o uso superficial do uso competente do CDK.

---

## Conceitos principais

### 1. A hierarquia de abstrações

```
L3 — Patterns (alta abstração)
  Composição de múltiplos recursos pré-configurados
  Ex: ApplicationLoadBalancedFargateService
       └── cria: ECS Cluster + Fargate Service + ALB + Target Group
                + Listener + Security Groups + IAM Roles + Log Group

L2 — Intent-Based (abstração média)
  Um recurso AWS com defaults seguros e helpers
  Ex: s3.Bucket
       └── encapsula: AWS::S3::Bucket (L1)
       └── gera automaticamente: BucketPolicy, AutoDeleteObjects Lambda
       └── expõe: bucket.grantRead(), bucket.grantReadWrite(), etc.

L1 — CloudFormation Layer (sem abstração)
  Mapeamento direto da spec do CloudFormation
  Ex: s3.CfnBucket
       └── equivale exatamente a: AWS::S3::Bucket no YAML
       └── toda propriedade do CloudFormation disponível
       └── nenhum default, nenhum helper
```

---

### 2. L1 — Constructs CloudFormation (prefixo `Cfn`)

[FATO] L1 constructs são **gerados automaticamente** a partir da especificação do CloudFormation. Se um recurso existe no CloudFormation, existe um L1 correspondente no CDK — com o prefixo `Cfn`.

```typescript
import * as s3 from 'aws-cdk-lib/aws-s3';

// L1: equivale byte-a-byte ao CloudFormation YAML
const bucket = new s3.CfnBucket(this, 'MeuBucket', {
  bucketName: 'meu-bucket-prod',
  versioningConfiguration: {
    status: 'Enabled',
  },
  bucketEncryption: {
    serverSideEncryptionConfiguration: [
      {
        serverSideEncryptionByDefault: {
          sseAlgorithm: 'AES256',
        },
      },
    ],
  },
  publicAccessBlockConfiguration: {
    blockPublicAcls: true,
    blockPublicPolicy: true,
    ignorePublicAcls: true,
    restrictPublicBuckets: true,
  },
});
```

**O que o L1 gera no CloudFormation:**

```json
{
  "MeuBucket": {
    "Type": "AWS::S3::Bucket",
    "Properties": {
      "BucketName": "meu-bucket-prod",
      "VersioningConfiguration": { "Status": "Enabled" },
      "BucketEncryption": { ... },
      "PublicAccessBlockConfiguration": { ... }
    }
  }
}
```

Exatamente o que você escreveria manualmente. Sem nada a mais, sem nada a menos.

**Características do L1:**
- Propriedades em `camelCase` (CDK) vs `PascalCase` (CloudFormation) — conversão automática
- Tipos estritos — o TypeScript detecta propriedades incorretas em tempo de compilação
- Cobertura 100% da spec do CloudFormation — se o CloudFormation suporta, o L1 suporta
- Sem defaults — você é responsável por cada propriedade

---

### 3. L2 — Constructs intent-based

L2 é onde o CDK entrega seu maior valor. Além de encapsular o L1, ele adiciona:

**a) Defaults seguros e opinionados:**

```typescript
import * as s3 from 'aws-cdk-lib/aws-s3';

// L2: muito menos código, defaults sensatos
const bucket = new s3.Bucket(this, 'MeuBucket', {
  versioned: true,
  encryption: s3.BucketEncryption.S3_MANAGED,
  blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
  removalPolicy: cdk.RemovalPolicy.RETAIN,  // padrão já é RETAIN para S3
});
// O L2 aplica automaticamente: block public access completo por padrão (v2)
```

**b) Helper methods (o maior diferencial):**

```typescript
const bucket = new s3.Bucket(this, 'AppBucket');
const fn = new lambda.Function(this, 'Processor', { ... });
const role = new iam.Role(this, 'AppRole', { ... });

// Sem L2 (L1 puro): você escreveria a IAM Policy manualmente com o ARN correto
// Com L2: o construct sabe quais permissões são necessárias
bucket.grantRead(fn);           // s3:GetObject + s3:ListBucket na policy da fn
bucket.grantReadWrite(role);    // s3:GetObject + s3:PutObject + s3:DeleteObject + s3:ListBucket
bucket.grantPut(fn);            // apenas s3:PutObject

// Outros exemplos de helpers:
queue.grantSendMessages(fn);
table.grantReadData(fn);
topic.grantPublish(role);
fn.grantInvoke(role);
```

**c) Integrações entre recursos:**

```typescript
import * as events from 'aws-cdk-lib/aws-events';
import * as targets from 'aws-cdk-lib/aws-events-targets';

const rule = new events.Rule(this, 'DailyRule', {
  schedule: events.Schedule.cron({ hour: '8', minute: '0' }),
});

// Adiciona a Lambda como target: cria a permissão e o target automaticamente
rule.addTarget(new targets.LambdaFunction(fn));
```

**O que o L2 gera que você não vê diretamente:**

```typescript
// Quando você escreve:
new s3.Bucket(this, 'MeuBucket', {
  removalPolicy: cdk.RemovalPolicy.DESTROY,
  autoDeleteObjects: true,      // ← essa linha
});

// O CDK gera em CloudFormation:
// 1. AWS::S3::Bucket (o bucket em si)
// 2. AWS::IAM::Role (para a Lambda de auto-delete)
// 3. AWS::Lambda::Function (a Lambda que deleta os objetos)
// 4. AWS::Lambda::Permission (permissão para o CloudFormation invocar)
// 5. AWS::CloudFormation::CustomResource (que aciona a Lambda no delete)
// 6. AWS::S3::BucketPolicy (se você usar grants)
```

[CONSENSO] Esse "efeito colateral" do L2 é o principal motivo pelo qual L2 não é sempre a melhor escolha. Às vezes você não quer esses recursos extras — especialmente em ambientes com políticas de segurança restritas sobre Lambdas e IAM roles geradas automaticamente.

---

### 4. Quando L2 não é preferível a L1

Esta é a parte contraintuitiva da sessão. L2 não é sempre melhor.

**Caso 1: A propriedade que você precisa não está exposta no L2**

L2 constructs são mantidos pela equipe da AWS e nem sempre acompanham a velocidade dos lançamentos de novos recursos do CloudFormation. Quando o CloudFormation lança uma nova propriedade, o L1 a expõe automaticamente (é gerado da spec), mas o L2 pode levar semanas ou meses para expô-la via método.

```typescript
// Ex hipotético: nova propriedade lançada ontem no CloudFormation
// O L2 ainda não tem um prop para isso

// Opção A: usar L1 diretamente
const bucket = new s3.CfnBucket(this, 'Bucket', {
  novaPropriedade: 'valor',    // disponível no L1 imediatamente
});

// Opção B: usar L2 + escape hatch (veja seção 6)
const bucket = new s3.Bucket(this, 'Bucket');
const cfnBucket = bucket.node.defaultChild as s3.CfnBucket;
cfnBucket.addPropertyOverride('NovaPropriedade', 'valor');
```

**Caso 2: Os recursos extras do L2 são indesejados**

```typescript
// autoDeleteObjects gera uma Lambda + Role + CustomResource
// Em ambientes com SCPs que bloqueiam criação de Lambdas em certas regiões,
// ou com revisão obrigatória de IAM roles, isso é um problema operacional

// L1 puro: apenas o bucket, nada mais
const bucket = new s3.CfnBucket(this, 'Bucket', {
  bucketName: 'meu-bucket',
  // sem recursos extras
});
```

**Caso 3: Migração de CloudFormation existente**

Ao importar um recurso existente (via `cloudformation import`), você precisa que o logical ID no CDK corresponda exatamente ao do template original. L1 com `overrideLogicalId` dá controle total; L2 com hash pode exigir mais trabalho.

**Caso 4: Auditoria e compliance de template**

Em organizações com revisão de segurança do template CloudFormation antes do deploy, L2 gera recursos "implícitos" que aparecem no template e precisam ser justificados. L1 produz templates previsíveis e auditáveis.

**Regra prática:**

```
Prefira L2 quando:
  → Você quer velocidade de desenvolvimento
  → Os defaults são adequados para o seu caso
  → Os helpers de permissão (grant*) simplificam muito o código
  → Você está em fase de prototipação

Prefira L1 quando:
  → Você precisa de uma propriedade que o L2 ainda não expõe
  → Os recursos extras gerados pelo L2 são indesejados
  → Você precisa de logical IDs fixos e previsíveis
  → O template final precisa ser auditável sem surpresas
```

---

### 5. L3 — Patterns

L3 constructs compõem múltiplos L2 em um padrão arquitetural reutilizável. O exemplo canônico é `ApplicationLoadBalancedFargateService`:

```typescript
import * as ecs_patterns from 'aws-cdk-lib/aws-ecs-patterns';
import * as ecs from 'aws-cdk-lib/aws-ecs';

const cluster = new ecs.Cluster(this, 'Cluster', { vpc });

// Uma linha substitui ~150 linhas de CloudFormation
const service = new ecs_patterns.ApplicationLoadBalancedFargateService(this, 'Service', {
  cluster,
  cpu: 256,
  memoryLimitMiB: 512,
  desiredCount: 2,
  taskImageOptions: {
    image: ecs.ContainerImage.fromRegistry('nginx:latest'),
    containerPort: 80,
  },
  publicLoadBalancer: true,
});

// O L3 criou por baixo:
// - ECS Cluster (se não fornecido)
// - Task Definition + Container Definition
// - Fargate Service
// - Application Load Balancer
// - ALB Listener (porta 80/443)
// - Target Group com health check
// - Security Groups (ALB → Service)
// - IAM Roles (task role + execution role)
// - CloudWatch Log Group
```

**Acessando os componentes internos de um L3:**

```typescript
// O L3 expõe os sub-constructs para customização
service.targetGroup.configureHealthCheck({
  path: '/health',
  healthyHttpCodes: '200',
});

service.loadBalancer.addSecurityGroup(mySG);
service.taskDefinition.addContainer('sidecar', { ... });

// Para customizações mais profundas: escape hatch no sub-recurso
const cfnService = service.service.node.defaultChild as ecs.CfnService;
cfnService.addPropertyOverride('DeploymentConfiguration.MaximumPercent', 200);
```

**Outros L3 patterns nativos relevantes:**

```typescript
// Fila SQS + Lambda consumer pré-configurados
new sqs_event_sources.SqsEventSource(queue, { batchSize: 10 });

// API Gateway + Lambda integrado
new apigw.LambdaRestApi(this, 'Api', { handler: fn });

// SNS topic com email subscription
new sns_subscriptions.EmailSubscription('ops@empresa.com');
```

---

### 6. Escape hatches — descendo do L2 para o L1

Quando um L2 não expõe o que você precisa, você acessa o L1 subjacente via escape hatch. Esse é o mecanismo que garante que o CDK nunca te bloqueia — você sempre pode descer um nível.

```typescript
const bucket = new s3.Bucket(this, 'Bucket', {
  versioned: true,
});

// Acessar o L1 subjacente
const cfnBucket = bucket.node.defaultChild as s3.CfnBucket;

// Modificar uma propriedade existente do L1
cfnBucket.lifecycleConfiguration = {
  rules: [
    {
      status: 'Enabled',
      transitions: [
        {
          storageClass: 'GLACIER',
          transitionInDays: 90,
        },
      ],
    },
  ],
};

// Adicionar uma propriedade que o L2 não expõe (usando override direto)
cfnBucket.addPropertyOverride('ObjectLockEnabled', true);

// Adicionar metadata ao recurso CloudFormation
cfnBucket.addOverride('Metadata', {
  'aws:cdk:path': 'MeuStack/Bucket/Resource',
  'Comment': 'Auditado em 2026-05-11',
});

// Remover uma propriedade gerada pelo L2
cfnBucket.addPropertyDeletionOverride('Tags');

// Fixar o logical ID (para migração de stacks existentes)
cfnBucket.overrideLogicalId('MeuBucketOriginal');
```

**Hierarquia de escape hatches (do menos para o mais invasivo):**

```
1. Props do L2                → use sempre que disponível
   bucket.versioned = true

2. Methods do L2              → para integrações e permissões
   bucket.grantRead(fn)

3. node.defaultChild + props  → para props L1 não expostas no L2
   cfnBucket.lifecycleConfiguration = { ... }

4. addPropertyOverride        → para props L1 sem tipo no CDK
   cfnBucket.addPropertyOverride('NewProp', value)

5. addOverride                → para metadados, DependsOn, etc.
   cfnBucket.addOverride('DependsOn', ['OutroRecurso'])

6. CfnResource puro (L1)      → quando L2 não serve de jeito nenhum
   new s3.CfnBucket(this, 'Bucket', { ... })
```

---

### 7. Construct Hub — avaliando constructs de terceiros

[FATO] O [Construct Hub](https://constructs.dev) é o repositório público de constructs CDK de terceiros, mantido pela AWS. Constructs são publicados via npm (TypeScript/JavaScript), PyPI (Python), Maven (Java) e NuGet (C#).

**Como avaliar um construct de terceiro:**

```
Critérios técnicos:
  ✅ Compatível com CDK v2 (aws-cdk-lib, não @aws-cdk/*)
  ✅ Versão recente com releases regulares
  ✅ Testes na suite de CI (GitHub Actions, etc.)
  ✅ README com exemplos claros de uso
  ✅ Escape hatches documentados

Critérios de risco:
  ⚠️  Mantenedor único vs. organização
  ⚠️  Número de dependentes (downloads/semana)
  ⚠️  Issues abertas sem resposta
  ⚠️  Compatibilidade com sua versão de CDK
  ⚠️  Licença compatível com o uso pretendido
```

**Alternativa ao construct de terceiro:** sempre verifique se o CDK nativo tem um L3 equivalente antes de adicionar uma dependência externa. Constructs de terceiros têm custo de manutenção — quando o CDK lança uma versão incompatível, você depende do terceiro atualizar.

---

## Exemplo prático

Comparação lado a lado dos três níveis para o mesmo recurso:

```typescript
import * as cdk from 'aws-cdk-lib';
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as iam from 'aws-cdk-lib/aws-iam';

export class ComparacaoStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string) {
    super(scope, id);

    const role = new iam.Role(this, 'AppRole', {
      assumedBy: new iam.ServicePrincipal('lambda.amazonaws.com'),
    });

    // ── L1: controle total, verboso ────────────────────────────────────
    const bucketL1 = new s3.CfnBucket(this, 'BucketL1', {
      versioningConfiguration: { status: 'Enabled' },
      bucketEncryption: {
        serverSideEncryptionConfiguration: [{
          serverSideEncryptionByDefault: { sseAlgorithm: 'AES256' },
        }],
      },
    });
    // Para dar permissão: você escreve a IAM Policy completa manualmente
    new iam.CfnPolicy(this, 'PolicyL1', {
      policyName: 'BucketAccess',
      roles: [role.roleName],
      policyDocument: {
        Version: '2012-10-17',
        Statement: [{
          Effect: 'Allow',
          Action: ['s3:GetObject', 's3:PutObject'],
          Resource: `${bucketL1.attrArn}/*`,
        }],
      },
    });

    // ── L2: conciso, helpers automáticos ──────────────────────────────
    const bucketL2 = new s3.Bucket(this, 'BucketL2', {
      versioned: true,
      encryption: s3.BucketEncryption.S3_MANAGED,
    });
    bucketL2.grantReadWrite(role);  // policy gerada automaticamente

    // ── L2 + escape hatch: melhor dos dois mundos ────────────────────
    const bucketHybrid = new s3.Bucket(this, 'BucketHybrid', {
      versioned: true,
    });
    // Adicionar propriedade não exposta pelo L2
    const cfn = bucketHybrid.node.defaultChild as s3.CfnBucket;
    cfn.addPropertyOverride('ObjectLockEnabled', true);
    bucketHybrid.grantReadWrite(role);  // ainda usa o helper do L2
  }
}
```

**Executar e comparar os templates gerados:**

```bash
cdk synth

# Ver quantos recursos cada abordagem gerou
cat cdk.out/ComparacaoStack.template.json | jq '[.Resources | to_entries[] | .value.Type] | group_by(.) | map({type: .[0], count: length})'

# Inspecionar a policy gerada pelo grantReadWrite do L2
cat cdk.out/ComparacaoStack.template.json | \
  jq '.Resources | to_entries[] | select(.value.Type == "AWS::IAM::Policy") | .value.Properties.PolicyDocument'
```

---

## Armadilhas comuns

**1. Assumir que o L2 tem todas as propriedades do CloudFormation**

O L2 é uma abstração mantida manualmente. Novos recursos do CloudFormation aparecem no L1 imediatamente (geração automática), mas no L2 dependem de um PR sendo mergeado. Antes de concluir que "o CDK não suporta X", verifique se o L1 tem o que você precisa — na maioria dos casos, tem.

**2. Usar escape hatch quando um método do L2 existe**

`cfnBucket.addPropertyOverride('BucketName', 'meu-bucket')` funciona, mas `new s3.Bucket(this, 'B', { bucketName: 'meu-bucket' })` é mais legível, tipado e valida o valor. Use escape hatch apenas quando o L2 realmente não expõe o que você precisa.

**3. Constructs L3 de terceiros como caixa preta**

Um construct L3 de terceiro pode criar dezenas de recursos. Se você não leu o código ou o README com atenção, você pode ter surpresas: recursos com `RemovalPolicy.RETAIN` que você não controla, IAM policies mais amplas do que o necessário, ou configurações de networking que conflitam com sua VPC. Sempre faça `cdk synth` e revise o template gerado antes de adotar um L3 de terceiro em produção.

---

## Exercício de reflexão

Você está implementando um bucket S3 para armazenar logs de auditoria com as seguintes restrições de segurança: Object Lock habilitado em modo GOVERNANCE com retenção de 7 anos, notificação para uma fila SQS a cada `s3:ObjectCreated:*`, e uma bucket policy que nega qualquer operação que não use TLS.

Ao tentar implementar com L2, você descobre que `Object Lock` não está disponível como prop direto no construct `s3.Bucket` do CDK v2.

Descreva a estratégia completa de implementação: qual combinação de L1, L2 e escape hatches você usaria para cada requisito, e por que. Existe algum requisito que é mais simples em L1 puro do que em L2 + escape hatch? Se sim, qual e por quê?

---

## Recursos para aprofundar

**Modelo de constructs:**
- [AWS CDK Constructs](https://docs.aws.amazon.com/cdk/v2/guide/constructs.html) — documentação oficial com a definição dos três níveis e exemplos em TypeScript e Python.

**Escape hatches:**
- [Customize constructs from the AWS Construct Library](https://docs.aws.amazon.com/cdk/v2/guide/cfn_layer.html) — todos os métodos de escape hatch com exemplos: `node.defaultChild`, `addPropertyOverride`, `addOverride`, `addPropertyDeletionOverride`.

**L3 patterns ECS:**
- [aws-cdk-lib.aws_ecs_patterns module](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_ecs_patterns-readme.html) — lista todos os patterns nativos disponíveis com exemplos completos.

**Construct Hub:**
- [constructs.dev](https://constructs.dev) — busque por serviço ou caso de uso. Use os filtros de linguagem e compatibilidade com CDK v2.

---

