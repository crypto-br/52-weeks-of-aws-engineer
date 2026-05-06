# Sessão 006 — CDK: Stacks, environments e multi-account patterns

**Duração estimada:** 60 minutos
**Pré-requisitos:** session-005 — CDK: Constructs L1, L2, L3

---

## Objetivo

Ao final, você conseguirá criar múltiplos stacks em uma CDK App com environments distintos (dev/staging/prod em contas diferentes), usar `Stack.account` e `Stack.region` sem hardcode, e entender quando usar um stack por conta vs stacks nested.

---

## Contexto

[CONSENSO] A estrutura multi-conta é o padrão recomendado pela AWS para organizações maduras — uma conta por ambiente (dev, staging, prod), idealmente sob uma AWS Organization. O CDK tem suporte nativo a esse modelo via `environments` e `Stages`. Entender como o CDK resolve account e region é crítico para evitar dois erros opostos: hardcode de valores que quebram em outras contas, e stacks "environment-agnostic" que perdem funcionalidades importantes.

[FATO] No CDK, um **environment** é simplesmente o par `{ account: string, region: string }`. É o destino de deploy de um stack. Um stack sem environment explícito é chamado de "environment-agnostic" e tem limitações importantes.

---

## Conceitos principais

### 1. Environments — account + region sem hardcode

Todo stack pode (e em produção, deve) ter um environment associado:

```typescript
// bin/meu-app.ts

const app = new cdk.App();

// ❌ Hardcode — não faça isso
new MeuStack(app, 'MeuStack', {
  env: { account: '123456789012', region: 'us-east-1' },
});

// ✅ Environment dinâmico — usa o perfil/credenciais ativas no momento do synth
new MeuStack(app, 'MeuStack', {
  env: {
    account: process.env.CDK_DEFAULT_ACCOUNT,
    region:  process.env.CDK_DEFAULT_REGION,
  },
});

// ✅ Multi-conta explícita — o mais correto para pipelines
new MeuStack(app, 'DevStack', {
  env: { account: '111122223333', region: 'us-east-1' },
});

new MeuStack(app, 'ProdStack', {
  env: { account: '444455556666', region: 'us-east-1' },
});
```

**`CDK_DEFAULT_ACCOUNT` vs `CDK_DEFAULT_REGION`:**

[FATO] O CDK CLI popula automaticamente `CDK_DEFAULT_ACCOUNT` e `CDK_DEFAULT_REGION` com base nas credenciais AWS ativas (perfil SSO, variáveis de ambiente, instance profile — a mesma cadeia do CLI). Se você usa `--profile prod`, o CDK resolve essas variáveis para a conta associada ao perfil `prod`.

```bash
# Deploy em dev com perfil dev
cdk deploy --profile dev

# Deploy em prod com perfil prod
cdk deploy --profile prod

# O mesmo código, environments diferentes — sem nenhuma mudança no código
```

---

### 2. Environment-agnostic stacks — quando e por que evitar

Um stack sem `env` é sintetizado sem account/region específicos. O template resultante usa pseudo-parâmetros do CloudFormation (`AWS::AccountId`, `AWS::Region`).

```typescript
// Stack sem environment — environment-agnostic
new MeuStack(app, 'MeuStack');
// Gera: { "Account": { "Ref": "AWS::AccountId" } } no template
```

**O que você perde com environment-agnostic:**

```
❌ Context lookups não funcionam
   ec2.Vpc.fromLookup(...)        → falha: precisa saber a conta/região para consultar
   HostedZone.fromLookup(...)     → idem

❌ Service Principals podem estar errados
   Em regiões opt-in (ex: ap-east-1) o formato do service principal é diferente
   O CDK resolve isso automaticamente quando o environment está fixado

❌ Algumas validações do CDK são desabilitadas
   Verificações que dependem de dados da conta (limites, AMIs, etc.)

✅ O que ainda funciona
   Deploy em qualquer conta/região via cdk deploy --profile X
   Ideal para constructs reutilizáveis publicados como bibliotecas
```

[FATO] Para stacks de aplicação em produção, sempre fixe o environment. Para constructs reutilizáveis publicados como NPM packages, mantenha environment-agnostic — quem instancia define o environment.

---

### 3. `Stack.account` e `Stack.region` — referências sem hardcode dentro do stack

Dentro de um stack ou construct, você acessa account e region via `this.account` e `this.region` (ou `Stack.of(this).account`):

```typescript
export class MeuStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props: cdk.StackProps) {
    super(scope, id, props);

    // ✅ Resolve em synth time se o environment está fixado
    const bucketName = `meu-app-${this.account}-${this.region}`;

    // ✅ Usando Sub para montar ARNs sem hardcode
    const paramPath = `/meu-app/${this.stackName}/config`;

    // ✅ Stack.of() — acessa o stack a partir de qualquer construct filho
    const account = cdk.Stack.of(this).account;
    const region  = cdk.Stack.of(this).region;

    // ✅ Tokens para valores resolvidos em deploy time (não em synth)
    // Quando o environment NÃO está fixado, this.account retorna um Token
    // (string como "${Token[AWS.AccountId.0]}") — não dá para usar em lógica condicional
    const isResolved = !cdk.Token.isUnresolved(this.account);
  }
}
```

**Token vs valor concreto — armadilha frequente:**

```typescript
// ❌ Não funciona quando environment-agnostic
// this.account retorna um Token, não '123456789012'
if (this.account === '123456789012') {  // sempre false com Token
  // lógica específica de conta
}

// ✅ Use props customizadas para lógica condicional
interface MeuStackProps extends cdk.StackProps {
  isProd: boolean;
}

export class MeuStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props: MeuStackProps) {
    super(scope, id, props);

    const removalPolicy = props.isProd
      ? cdk.RemovalPolicy.RETAIN
      : cdk.RemovalPolicy.DESTROY;
  }
}
```

---

### 4. Stage — agrupando stacks para multi-environment

`Stage` é o conceito do CDK para agrupar stacks que representam o mesmo sistema em um ambiente específico. É a unidade de promoção em pipelines (dev → staging → prod).

```typescript
// lib/application-stage.ts
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import { InfraStack } from './infra-stack';
import { AppStack } from './app-stack';

export class ApplicationStage extends cdk.Stage {
  constructor(scope: Construct, id: string, props: cdk.StageProps) {
    super(scope, id, props);

    // Todos os stacks deste sistema, juntos
    const infra = new InfraStack(this, 'Infra');
    const app   = new AppStack(this, 'App', {
      vpc: infra.vpc,
      cluster: infra.cluster,
    });
  }
}
```

```typescript
// bin/meu-app.ts
const app = new cdk.App();

// Stage de dev — conta de desenvolvimento
new ApplicationStage(app, 'Dev', {
  env: { account: '111122223333', region: 'us-east-1' },
});

// Stage de staging — conta separada
new ApplicationStage(app, 'Staging', {
  env: { account: '222233334444', region: 'us-east-1' },
});

// Stage de prod — conta de produção
new ApplicationStage(app, 'Prod', {
  env: { account: '333344445555', region: 'us-east-1' },
});
```

**O que o Stage gera:**

```
cdk ls
  Dev/Infra       → stack DevInfra na conta 111122223333
  Dev/App         → stack DevApp na conta 111122223333
  Staging/Infra   → stack StagingInfra na conta 222233334444
  Staging/App     → stack StagingApp na conta 222233334444
  Prod/Infra      → stack ProdInfra na conta 333344445555
  Prod/App        → stack ProdApp na conta 333344445555
```

**Deploy de um Stage completo:**

```bash
# Deploya todos os stacks do stage Dev
cdk deploy "Dev/*" --profile dev

# Deploya apenas um stack específico do stage
cdk deploy Dev/Infra --profile dev

# Diff de todo o stage Prod
cdk diff "Prod/*" --profile prod
```

---

### 5. Quando usar stack por conta vs stacks nested

Esta é uma decisão de arquitetura com trade-offs reais.

**Stack por conta (padrão recomendado):**

```
App
├── Dev/
│   ├── InfraStack   → VPC, subnets, SGs
│   └── AppStack     → ECS, Lambda, RDS
├── Staging/
│   ├── InfraStack
│   └── AppStack
└── Prod/
    ├── InfraStack
    └── AppStack
```

Vantagens:
- Isolamento real entre ambientes (contas separadas = limites de IAM, billing, CloudTrail)
- Falhas em dev não afetam prod
- Modelo alinhado com AWS Organizations e SCPs

**Múltiplos stacks na mesma conta (por ciclo de vida):**

```
App (mesma conta dev)
├── NetworkStack     → VPC, subnets (muda raramente)
├── DataStack        → RDS, DynamoDB (muda ocasionalmente)
└── AppStack         → Lambda, ECS, API GW (muda frequentemente)
```

Vantagens:
- Deploy parcial — apenas o AppStack muda no dia a dia
- Reduz risco: uma mudança no código de app não toca a infraestrutura de rede
- CloudFormation tem limite de 500 recursos por stack — dividir resolve isso

[CONSENSO] O critério principal para dividir em stacks não é tamanho — é **ciclo de vida e ownership**. Recursos que mudam juntos ficam no mesmo stack. Recursos que têm owners diferentes ficam em stacks diferentes. Recursos que nunca deveriam ser afetados por um deploy de aplicação ficam num stack separado.

**Stack por conta vs nested stacks:**

[FATO] Nested stacks (via `NestedStack`) são stacks CloudFormation filhos de outro stack. São úteis para superar o limite de 500 recursos, mas criam acoplamento — o stack pai é responsável pelo ciclo de vida dos filhos. No CDK, a alternativa recomendada é usar múltiplos stacks independentes (com referências cross-stack) em vez de nested stacks.

```typescript
// ❌ Nested stack — evite salvo para superar limite de recursos
import { NestedStack } from 'aws-cdk-lib';
class MinhaNestedStack extends NestedStack { ... }

// ✅ Stacks independentes com referência cross-stack
class InfraStack extends cdk.Stack {
  public readonly vpc: ec2.Vpc;  // expõe o recurso
  constructor(...) {
    this.vpc = new ec2.Vpc(this, 'Vpc');
  }
}

class AppStack extends cdk.Stack {
  constructor(scope, id, props: { vpc: ec2.Vpc } & cdk.StackProps) {
    // consome o recurso de outro stack
    new ecs.Cluster(this, 'Cluster', { vpc: props.vpc });
  }
}
```

---

### 6. Referências cross-stack no CDK

Quando um stack referencia um recurso de outro stack no CDK, o framework automaticamente cria um CloudFormation Export/Import por baixo — mas você não precisa gerenciar isso manualmente.

```typescript
// bin/meu-app.ts
const app = new cdk.App();

const infra = new InfraStack(app, 'InfraStack', {
  env: { account: '111122223333', region: 'us-east-1' },
});

// Passa o objeto diretamente — o CDK gerencia o Export/Import automaticamente
const appStack = new AppStack(app, 'AppStack', {
  env: { account: '111122223333', region: 'us-east-1' },
  vpc: infra.vpc,              // referência direta ao objeto
  cluster: infra.cluster,
});
```

**O que o CDK gera por baixo:**

```
InfraStack Outputs:
  ExportsOutputRefVpcXXXXXX: vpc-0abc123   (Export: InfraStack:ExportsOutputRefVpcXXXXXX)

AppStack Resources:
  Cluster:
    Properties:
      VpcId: !ImportValue InfraStack:ExportsOutputRefVpcXXXXXX
```

[FATO] Referências cross-stack no CDK **só funcionam dentro da mesma conta e região**. Para referenciar recursos cross-account, você precisa usar SSM Parameter Store, Secrets Manager, ou passar valores como props concretos (string/ARN) em vez de objetos CDK.

**Cross-account — padrão correto:**

```typescript
// InfraStack (conta A) — exporta via SSM
const vpc = new ec2.Vpc(this, 'Vpc');
new ssm.StringParameter(this, 'VpcIdParam', {
  parameterName: '/shared/vpc-id',
  stringValue: vpc.vpcId,
});

// AppStack (conta B) — importa via lookup
const vpcId = ssm.StringParameter.valueForStringParameter(this, '/shared/vpc-id');
const vpc = ec2.Vpc.fromLookup(this, 'SharedVpc', { vpcId });
```

---

### 7. Diagrama arquitetural — App CDK multi-conta completo

```
CDK App (repositório único)
│
├── bin/app.ts
│     ├── new ApplicationStage(app, 'Dev',     { env: devEnv })
│     ├── new ApplicationStage(app, 'Staging', { env: stagingEnv })
│     └── new ApplicationStage(app, 'Prod',    { env: prodEnv })
│
├── lib/
│     ├── application-stage.ts    ← Stage: agrupa os stacks
│     ├── network-stack.ts        ← VPC, subnets (muda raramente)
│     ├── data-stack.ts           ← RDS, DynamoDB (muda ocasionalmente)
│     └── app-stack.ts            ← ECS, Lambda (muda frequentemente)
│
└── Bootstrap necessário em cada conta/região:
      cdk bootstrap aws://111122223333/us-east-1  (dev)
      cdk bootstrap aws://222233334444/us-east-1  (staging)
      cdk bootstrap aws://333344445555/us-east-1  (prod)

Deploy:
  cdk deploy "Dev/*"     --profile dev
  cdk deploy "Staging/*" --profile staging
  cdk deploy "Prod/*"    --profile prod
```

---

## Exemplo prático

App CDK completo com três stacks e dois environments:

```typescript
// lib/network-stack.ts
export class NetworkStack extends cdk.Stack {
  public readonly vpc: ec2.Vpc;

  constructor(scope: Construct, id: string, props: cdk.StackProps) {
    super(scope, id, props);

    this.vpc = new ec2.Vpc(this, 'Vpc', {
      maxAzs: 2,
      natGateways: 1,
      subnetConfiguration: [
        { name: 'public',  subnetType: ec2.SubnetType.PUBLIC,           cidrMask: 24 },
        { name: 'private', subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS, cidrMask: 24 },
      ],
    });

    // Exporta o VPC ID via SSM para consumo cross-account
    new ssm.StringParameter(this, 'VpcIdParam', {
      parameterName: `/${this.stackName}/vpc-id`,
      stringValue: this.vpc.vpcId,
    });
  }
}

// lib/app-stack.ts
interface AppStackProps extends cdk.StackProps {
  vpc: ec2.Vpc;
  isProd: boolean;
}

export class AppStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props: AppStackProps) {
    super(scope, id, props);

    const cluster = new ecs.Cluster(this, 'Cluster', { vpc: props.vpc });

    new s3.Bucket(this, 'AssetsBucket', {
      versioned: props.isProd,
      removalPolicy: props.isProd
        ? cdk.RemovalPolicy.RETAIN
        : cdk.RemovalPolicy.DESTROY,
      autoDeleteObjects: !props.isProd,
    });
  }
}

// lib/application-stage.ts
export class ApplicationStage extends cdk.Stage {
  constructor(scope: Construct, id: string, props: cdk.StageProps & { isProd: boolean }) {
    super(scope, id, props);

    const network = new NetworkStack(this, 'Network');
    new AppStack(this, 'App', {
      vpc: network.vpc,
      isProd: props.isProd,
    });
  }
}

// bin/app.ts
const app = new cdk.App();

new ApplicationStage(app, 'Dev', {
  env: { account: '111122223333', region: 'us-east-1' },
  isProd: false,
});

new ApplicationStage(app, 'Prod', {
  env: { account: '333344445555', region: 'us-east-1' },
  isProd: true,
});
```

```bash
# Listar todos os stacks gerados
cdk ls
# Dev/Network
# Dev/App
# Prod/Network
# Prod/App

# Deploy de dev (bootstrap já feito)
cdk deploy "Dev/*" --profile dev --require-approval never

# Diff de prod antes de promover
cdk diff "Prod/*" --profile prod

# Deploy de prod com aprovação de mudanças IAM
cdk deploy "Prod/*" --profile prod
```

---

## Armadilhas comuns

**1. `Token.isUnresolved` — usar `this.account` em lógica condicional**

`this.account` retorna um Token quando o environment não está fixado, ou em certas fases do synth. Usar em comparações (`if (this.account === '123...')`) sempre falha silenciosamente. Use props tipadas para configurações condicionais por ambiente — nunca compare `this.account` diretamente em lógica de negócio.

**2. Cross-stack reference entre accounts — objeto CDK não cruza conta**

Passar um objeto CDK (`vpc`, `cluster`, `bucket`) como prop funciona apenas dentro da mesma conta e região. O CDK cria um Export/Import do CloudFormation, que é regional. Para cross-account, você precisa passar strings (IDs, ARNs) e usar métodos `from*` para importar o recurso (`ec2.Vpc.fromVpcAttributes`, `s3.Bucket.fromBucketName`, etc.).

**3. Stage com `env` vs stacks com `env` — quem prevalece**

O `env` do Stage é herdado por todos os stacks filhos que não definem `env` próprio. Se um stack filho define um `env` diferente, o stack vai para uma conta/região diferente do Stage — o que pode ser intencional mas é frequentemente um bug. Mantenha consistência: ou o Stage define o `env` e os stacks herdam, ou cada stack define o seu.

---

## Exercício de reflexão

Você está desenhando a estrutura CDK para uma plataforma com os seguintes componentes: VPC compartilhada (usada por todos os serviços), banco de dados RDS (atualizado raramente, com dados críticos), serviço de API (Lambda + API Gateway, deployado várias vezes ao dia), e dashboard de monitoramento (CloudWatch dashboards + alarms).

Você tem três ambientes: dev (conta única), staging (conta única) e prod (conta única, com múltiplas regiões: us-east-1 e eu-west-1).

Desenhe a estrutura de stacks e stages para essa plataforma. Para cada decisão — quantos stacks, como agrupá-los, como passar referências entre eles — justifique com base em ciclo de vida, risco de blast radius e requisitos operacionais. O que muda na estrutura para prod multi-região?

---

## Recursos para aprofundar

**Environments:**
- [Environments for the AWS CDK](https://docs.aws.amazon.com/cdk/v2/guide/environments.html) — cobre environment-agnostic vs environment-bound, CDK_DEFAULT_ACCOUNT/REGION e como o CLI resolve as variáveis.

**Stages:**
- [Introduction to AWS CDK stages](https://docs.aws.amazon.com/cdk/v2/guide/stages.html) — como criar e usar Stages para agrupar stacks por ambiente.

**Best practices:**
- [Best practices for developing and deploying cloud infrastructure with the AWS CDK](https://docs.aws.amazon.com/cdk/v2/guide/best-practices.html) — seção "Constructs vs Stacks" explica o critério de ciclo de vida para separar stacks, e "Environments" cobre a estrutura multi-conta recomendada.

---

