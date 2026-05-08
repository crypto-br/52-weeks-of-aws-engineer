# Sessão 008 — CDK: Testing com assertions — fine-grained e snapshot tests

**Duração estimada:** 60 minutos
**Pré-requisitos:** session-007 — CDK: Assets — bundling de Lambda, Docker images e arquivos locais

---

## Objetivo

Ao final, você conseguirá escrever testes unitários de infra com `aws-cdk-lib/assertions`, verificar que recursos existem com propriedades específicas (`hasResourceProperties`), criar snapshot tests e interpretar falhas de snapshot após mudanças de infra.

---

## Contexto

[CONSENSO] Testar infraestrutura como código é menos sobre encontrar bugs de lógica e mais sobre três coisas: **detectar regressões** (algo que funcionava parou de funcionar após uma mudança), **documentar intenções** (o teste descreve o que o construct deve garantir), e **ganhar confiança para refatorar** (você pode reorganizar o código sem medo de quebrar o comportamento).

[FATO] Os testes de CDK com `aws-cdk-lib/assertions` rodam localmente e completamente offline — eles sintetizam o template CloudFormation em memória e fazem assertions sobre o JSON gerado. Nenhuma chamada à AWS acontece. Um `npm test` típico roda em segundos.

[CONSENSO] A documentação oficial da AWS classifica os testes CDK em três categorias: fine-grained assertions (mais usados), snapshot tests (úteis para refatoração) e validation tests (verificam que configurações inválidas lançam exceções). Esta sessão cobre as três.

---

## Conceitos principais

### 1. Setup — Template.fromStack()

O ponto de entrada de qualquer teste CDK é `Template.fromStack()`, que sintetiza o stack em memória e retorna um objeto `Template` com os métodos de assertion.

```typescript
// test/meu-stack.test.ts
import * as cdk from 'aws-cdk-lib';
import { Template, Match, Capture } from 'aws-cdk-lib/assertions';
import { MeuStack } from '../lib/meu-stack';

describe('MeuStack', () => {
  let app: cdk.App;
  let stack: MeuStack;
  let template: Template;

  beforeEach(() => {
    app = new cdk.App();
    stack = new MeuStack(app, 'TestStack', {
      env: { account: '123456789012', region: 'us-east-1' },
    });
    template = Template.fromStack(stack);
  });

  // testes aqui
});
```

**Alternativa: `Template.fromJSON()`** — para testar templates CloudFormation existentes:

```typescript
import * as fs from 'fs';
const templateJson = JSON.parse(fs.readFileSync('path/to/template.json', 'utf8'));
const template = Template.fromJSON(templateJson);
```

---

### 2. Fine-grained assertions — os métodos principais

#### `hasResourceProperties` — verificar propriedades de um recurso

```typescript
// Verifica que existe um bucket S3 com versioning habilitado
template.hasResourceProperties('AWS::S3::Bucket', {
  VersioningConfiguration: {
    Status: 'Enabled',
  },
});

// Por padrão usa Match.objectLike — matching PARCIAL
// O bucket pode ter outras propriedades além das verificadas
template.hasResourceProperties('AWS::S3::Bucket', {
  BucketEncryption: {
    ServerSideEncryptionConfiguration: [
      {
        ServerSideEncryptionByDefault: {
          SSEAlgorithm: 'AES256',
        },
      },
    ],
  },
});
```

#### `hasResource` — verificar o recurso completo (além de Properties)

```typescript
// Verifica DeletionPolicy, DependsOn, Metadata, além de Properties
template.hasResource('AWS::S3::Bucket', {
  DeletionPolicy: 'Retain',
  Properties: {
    VersioningConfiguration: { Status: 'Enabled' },
  },
});
```

#### `resourceCountIs` — verificar quantidade de recursos

```typescript
// Deve existir exatamente 1 bucket
template.resourceCountIs('AWS::S3::Bucket', 1);

// Deve existir exatamente 2 funções Lambda
template.resourceCountIs('AWS::Lambda::Function', 2);

// Não deve existir nenhum recurso EC2
template.resourceCountIs('AWS::EC2::Instance', 0);
```

#### `findResources` — inspecionar recursos encontrados

```typescript
// Retorna um objeto com todos os recursos do tipo encontrados
const buckets = template.findResources('AWS::S3::Bucket');
console.log(Object.keys(buckets));  // ['MeuBucketXXXXXX']
console.log(Object.values(buckets)[0].Properties);  // propriedades do bucket

// Com filtro
const encryptedBuckets = template.findResources('AWS::S3::Bucket', {
  Properties: {
    BucketEncryption: Match.anyValue(),
  },
});
```

#### `hasOutput` — verificar outputs da stack

```typescript
template.hasOutput('BucketName', {
  Value: { Ref: Match.stringLikeRegexp('MeuBucket') },
  Export: { Name: Match.anyValue() },
});
```

---

### 3. Matchers — controle fino sobre o matching

O módulo `Match` fornece matchers para controlar como as comparações são feitas.

```typescript
import { Match } from 'aws-cdk-lib/assertions';

// Match.objectLike — matching parcial (padrão do hasResourceProperties)
// O objeto real pode ter mais chaves do que o esperado
template.hasResourceProperties('AWS::IAM::Role', {
  AssumeRolePolicyDocument: Match.objectLike({
    Statement: Match.arrayWith([
      Match.objectLike({
        Effect: 'Allow',
        Principal: { Service: 'lambda.amazonaws.com' },
      }),
    ]),
  }),
});

// Match.exact — matching EXATO — o objeto deve ser idêntico
template.hasResourceProperties('AWS::SQS::Queue', Match.exact({
  VisibilityTimeout: 300,
  MessageRetentionPeriod: 86400,
}));
// Vai falhar se o recurso tiver qualquer prop além dessas duas

// Match.arrayWith — array contém esses elementos (pode ter outros)
template.hasResourceProperties('AWS::IAM::Policy', {
  PolicyDocument: {
    Statement: Match.arrayWith([
      {
        Effect: 'Allow',
        Action: 's3:GetObject',
        Resource: Match.anyValue(),
      },
    ]),
  },
});

// Match.arrayEquals — array é EXATAMENTE esses elementos, nessa ordem
template.hasResourceProperties('AWS::Lambda::Function', {
  Layers: Match.arrayEquals([
    { Ref: 'MinhaLayerXXXXXX' },
  ]),
});

// Match.stringLikeRegexp — string que bate com o regex
template.hasResourceProperties('AWS::Lambda::Function', {
  FunctionName: Match.stringLikeRegexp('^meu-app-.*-handler$'),
});

// Match.anyValue — qualquer valor não-null/undefined
template.hasResourceProperties('AWS::S3::Bucket', {
  BucketName: Match.anyValue(),
});

// Match.absent — a propriedade NÃO deve existir
template.hasResourceProperties('AWS::S3::Bucket', {
  WebsiteConfiguration: Match.absent(),
  PublicAccessBlockConfiguration: Match.absent(),   // confirma que public access está bloqueado
});

// Match.not — nega qualquer matcher
template.hasResourceProperties('AWS::Lambda::Function', {
  Runtime: Match.not(Match.stringLikeRegexp('nodejs14')),  // não usa Node 14
});

// Match.serializedJson — para propriedades que são JSON serializado como string
// (comum em policies inline e environment variables que guardam JSON)
template.hasResourceProperties('AWS::Lambda::Function', {
  Environment: {
    Variables: {
      CONFIG: Match.serializedJson({
        timeout: 30,
        retries: 3,
      }),
    },
  },
});
```

---

### 4. Capture — capturar valores para assertions adicionais

`Capture` permite extrair um valor do template para usá-lo em assertions subsequentes — útil quando o valor é dinâmico (hash, ARN gerado) mas você quer verificar sua estrutura.

```typescript
import { Capture } from 'aws-cdk-lib/assertions';

// Capturar o ARN da role para verificar em outro recurso
const roleArnCapture = new Capture();

template.hasResourceProperties('AWS::Lambda::Function', {
  Role: roleArnCapture,
});

// Agora roleArnCapture.asString() contém o valor capturado
// Ex: {"Fn::GetAtt": ["MeuRoleXXXXXX", "Arn"]}
console.log(roleArnCapture.asString());

// Capturar e verificar uma política serializada como JSON
const policyCapture = new Capture();

template.hasResourceProperties('AWS::SQS::Queue', {
  RedrivePolicy: policyCapture,
});

// O valor pode ser um objeto — use asObject()
const redrive = policyCapture.asObject();
expect(redrive.maxReceiveCount).toEqual(3);

// Capture com matcher — garante que o valor capturado tem a estrutura certa
const bucketCapture = new Capture(Match.objectLike({ 'Fn::GetAtt': Match.anyValue() }));

template.hasResourceProperties('AWS::Lambda::Function', {
  Environment: {
    Variables: {
      BUCKET_NAME: bucketCapture,
    },
  },
});
```

---

### 5. Validation tests — erros esperados

Além de verificar o que é criado, você deve testar que configurações inválidas lançam exceções:

```typescript
test('deve lançar erro quando porta fora do range', () => {
  const app = new cdk.App();
  const stack = new cdk.Stack(app, 'TestStack');

  expect(() => {
    new MeuWebServer(stack, 'Server', {
      port: 70000,  // porta inválida
    });
  }).toThrow('Port must be between 1024 and 65535');
});

test('deve lançar erro quando bucket name tem maiúsculas', () => {
  const app = new cdk.App();
  const stack = new cdk.Stack(app, 'TestStack');

  expect(() => {
    new s3.Bucket(stack, 'Bucket', {
      bucketName: 'MeuBucketInvalido',  // maiúsculas não permitidas
    });
  }).toThrow(/bucket name/i);
});
```

---

### 6. Snapshot tests — o template como baseline

Snapshot tests armazenam o template CloudFormation completo em um arquivo `.snap` e comparam em cada execução futura.

```typescript
test('snapshot do stack completo', () => {
  const app = new cdk.App();
  const stack = new MeuStack(app, 'SnapshotStack');
  const template = Template.fromStack(stack);

  // Na primeira execução: cria o arquivo .snap
  // Nas execuções seguintes: compara com o snapshot salvo
  expect(template.toJSON()).toMatchSnapshot();
});
```

**Estrutura dos arquivos de snapshot (Jest):**

```
test/
├── meu-stack.test.ts
└── __snapshots__/
    └── meu-stack.test.ts.snap   ← gerado automaticamente, comitar no git
```

**Atualizando snapshots após mudança intencional:**

```bash
# Atualiza todos os snapshots
npx jest --updateSnapshot

# Atualiza snapshot de um teste específico
npx jest --updateSnapshot --testNamePattern="snapshot do stack"

# Shorthand
npx jest -u
```

**O que fazer quando um snapshot falha:**

```
1. Leia o diff — o Jest mostra exatamente o que mudou
2. Avalie se a mudança é:
   a) Intencional (você alterou o código) → npx jest -u → comita o novo snapshot
   b) Colateral de uma atualização do CDK → revise se o novo comportamento é ok
   c) Regressão inesperada → encontre a causa e corrija o código
3. NUNCA faça -u sem ler o diff
```

---

### 7. Fine-grained vs snapshot — quando usar cada um

[FATO, conforme documentação oficial] A AWS recomenda usar fine-grained assertions como base dos testes e snapshot tests como complemento para refatoração.

```
Fine-grained assertions
  ✅ Detectam regressões de comportamento específico
  ✅ São explícitos sobre o que está sendo verificado
  ✅ Não quebram por mudanças não relacionadas no template
  ✅ Documentam as intenções do construct
  ✅ Funcionam como testes de contrato para constructs reutilizáveis
  ⚠️  Mais verbosos para escrever
  ⚠️  Não cobrem o template inteiro

Snapshot tests
  ✅ Cobrem o template inteiro sem esforço
  ✅ Detectam qualquer mudança — inclusive as que você não antecipou
  ✅ Úteis durante refatoração: garante que a refatoração não muda o output
  ⚠️  Quebram com atualizações do CDK que mudam o template por razões internas
  ⚠️  Não distinguem entre mudança intencional e regressão
  ⚠️  Podem acumular falsos positivos se o time fizer -u automaticamente
  ⚠️  Não são úteis para detectar regressões em constructs novos
```

**Estratégia combinada recomendada:**

```
Para cada construct:
  1. Fine-grained: verifica as invariantes críticas
     (IAM permissions corretas, encryption habilitada, deletion policy correta)
  2. Fine-grained: verifica que configurações inválidas lançam erro
  3. Snapshot: captura o template completo como baseline para refatoração
```

---

## Exemplo prático

Stack de aplicação com cobertura de testes completa:

```typescript
// lib/app-stack.ts
export class AppStack extends cdk.Stack {
  public readonly bucket: s3.Bucket;
  public readonly handler: lambda_nodejs.NodejsFunction;

  constructor(scope: Construct, id: string, props: AppStackProps) {
    super(scope, id, props);

    this.bucket = new s3.Bucket(this, 'AppBucket', {
      versioned: true,
      encryption: s3.BucketEncryption.S3_MANAGED,
      blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
      removalPolicy: props.isProd ? cdk.RemovalPolicy.RETAIN : cdk.RemovalPolicy.DESTROY,
    });

    this.handler = new lambda_nodejs.NodejsFunction(this, 'Handler', {
      entry: path.join(__dirname, '../src/handler.ts'),
      runtime: lambda.Runtime.NODEJS_22_X,
      environment: {
        BUCKET_NAME: this.bucket.bucketName,
        IS_PROD: props.isProd ? 'true' : 'false',
      },
    });

    this.bucket.grantReadWrite(this.handler);
  }
}

// test/app-stack.test.ts
import { Template, Match, Capture } from 'aws-cdk-lib/assertions';

describe('AppStack', () => {
  const makeStack = (isProd = false) => {
    const app = new cdk.App();
    const stack = new AppStack(app, 'TestStack', {
      env: { account: '123456789012', region: 'us-east-1' },
      isProd,
    });
    return Template.fromStack(stack);
  };

  describe('S3 Bucket', () => {
    test('bucket tem versioning habilitado', () => {
      const template = makeStack();
      template.hasResourceProperties('AWS::S3::Bucket', {
        VersioningConfiguration: { Status: 'Enabled' },
      });
    });

    test('bucket tem encryption habilitada', () => {
      const template = makeStack();
      template.hasResourceProperties('AWS::S3::Bucket', {
        BucketEncryption: {
          ServerSideEncryptionConfiguration: Match.arrayWith([
            Match.objectLike({
              ServerSideEncryptionByDefault: { SSEAlgorithm: 'AES256' },
            }),
          ]),
        },
      });
    });

    test('bucket em dev tem DeletionPolicy Delete', () => {
      const template = makeStack(false);
      template.hasResource('AWS::S3::Bucket', {
        DeletionPolicy: 'Delete',
      });
    });

    test('bucket em prod tem DeletionPolicy Retain', () => {
      const template = makeStack(true);
      template.hasResource('AWS::S3::Bucket', {
        DeletionPolicy: 'Retain',
      });
    });

    test('bucket NÃO tem acesso público', () => {
      const template = makeStack();
      template.hasResourceProperties('AWS::S3::Bucket', {
        PublicAccessBlockConfiguration: {
          BlockPublicAcls: true,
          BlockPublicPolicy: true,
          IgnorePublicAcls: true,
          RestrictPublicBuckets: true,
        },
      });
    });
  });

  describe('Lambda Function', () => {
    test('função tem runtime Node 22', () => {
      const template = makeStack();
      template.hasResourceProperties('AWS::Lambda::Function', {
        Runtime: 'nodejs22.x',
      });
    });

    test('função tem variável BUCKET_NAME no environment', () => {
      const template = makeStack();
      const bucketRef = new Capture();

      template.hasResourceProperties('AWS::Lambda::Function', {
        Environment: {
          Variables: {
            BUCKET_NAME: bucketRef,
          },
        },
      });

      // O valor deve ser uma referência ao bucket, não hardcode
      expect(JSON.stringify(bucketRef.asObject())).toContain('Ref');
    });

    test('função tem permissão de leitura e escrita no bucket', () => {
      const template = makeStack();

      // Deve existir uma IAM Policy com as permissões S3
      template.hasResourceProperties('AWS::IAM::Policy', {
        PolicyDocument: {
          Statement: Match.arrayWith([
            Match.objectLike({
              Effect: 'Allow',
              Action: Match.arrayWith([
                's3:GetObject*',
                's3:PutObject*',
              ]),
            }),
          ]),
        },
      });
    });
  });

  describe('Contagem de recursos', () => {
    test('deve criar exatamente 1 bucket', () => {
      const template = makeStack();
      template.resourceCountIs('AWS::S3::Bucket', 1);
    });

    test('deve criar exatamente 1 função Lambda', () => {
      const template = makeStack();
      template.resourceCountIs('AWS::Lambda::Function', 1);
    });
  });

  describe('Snapshot', () => {
    test('template não muda inesperadamente', () => {
      const template = makeStack();
      expect(template.toJSON()).toMatchSnapshot();
    });
  });
});
```

**Rodando os testes:**

```bash
# Rodar todos os testes
npx jest

# Rodar com cobertura
npx jest --coverage

# Rodar em modo watch (reexecuta ao salvar)
npx jest --watch

# Atualizar snapshots
npx jest --updateSnapshot

# Rodar apenas um arquivo
npx jest test/app-stack.test.ts

# Rodar um teste específico pelo nome
npx jest --testNamePattern="bucket tem versioning"

# Ver output detalhado de um teste que falhou
npx jest --verbose
```

**Interpretando uma falha de snapshot:**

```
● AppStack › Snapshot › template não muda inesperadamente

  expect(received).toMatchSnapshot()

  Snapshot name: `AppStack Snapshot template não muda inesperadamente 1`

  - Snapshot  - 5 lines
  + Received  + 5 lines

    "MeuHandlerXXXXXX": {
      "Type": "AWS::Lambda::Function",
      "Properties": {
  -     "Runtime": "nodejs20.x",    ← valor no snapshot salvo
  +     "Runtime": "nodejs22.x",    ← valor atual no código
        "Handler": "index.handler",

  # Interpretação:
  # Você atualizou o runtime de nodejs20 para nodejs22.
  # Mudança intencional → npx jest -u para aceitar o novo baseline.
```

---

## Armadilhas comuns

**1. `toMatchSnapshot` sem revisão do diff — o "snapshot debt"**

Equipes que rodam `npx jest -u` automaticamente em CI (para evitar falhas de snapshot após atualizações do CDK) acumulam snapshots que nunca foram revisados. O snapshot deixa de ser um teste e vira documentação desatualizada. Regra: `npx jest -u` só em commits onde a mudança no snapshot é o objetivo explícito, e o diff deve ser revisado antes do merge.

**2. `Match.objectLike` vs `Match.exact` — falsas aprovações**

`hasResourceProperties` usa `Match.objectLike` por padrão — o template pode ter mais propriedades do que as verificadas. Se você quer garantir que uma propriedade **não existe**, use `Match.absent()`. Se quer que o objeto seja **exatamente** o que você declarou, use `Match.exact()`. Não assuma que passar no `hasResourceProperties` significa que o recurso não tem propriedades adicionais indesejadas.

**3. Testar logical IDs em vez de propriedades**

Evite escrever testes que dependem do logical ID gerado pelo CDK (ex: `{ Ref: 'MeuBucketA1B2C3D4' }`). Logical IDs mudam quando você renomeia constructs ou reorganiza a hierarquia. Use `Match.anyValue()` ou `Capture` para isolar o teste do logical ID.

---

## Exercício de reflexão

Você está escrevendo testes para um construct `SecureBucket` que deve garantir: bucket sempre criptografado com KMS (não SSE-S3), public access sempre bloqueado, versioning habilitado, e um Bucket Policy que nega operações sem TLS (`aws:SecureTransport: false`).

Escreva os testes fine-grained para cada um desses requisitos. Para cada teste, explique: o que exatamente está sendo verificado, qual matcher é mais apropriado e por que, e o que o teste **não** verifica (seus limites). Por fim, o snapshot test desse construct é útil ou redundante dado os fine-grained tests? Justifique.

---

## Recursos para aprofundar

**Testing guide:**
- [Test AWS CDK applications](https://docs.aws.amazon.com/cdk/v2/guide/testing.html) — cobre os três tipos de teste (fine-grained, snapshot, validation) com exemplos em TypeScript e Python.

**Assertions API:**
- [aws-cdk-lib.assertions module](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.assertions-readme.html) — referência completa de todos os métodos do `Template` e de todos os `Match.*` matchers com exemplos de cada um.

**Artigo AWS:**
- [Testing Infrastructure with the AWS CDK](https://aws.amazon.com/blogs/developer/testing-infrastructure-with-the-aws-cloud-development-kit/) — walk-through completo com exemplos práticos dos três tipos de teste.

---

