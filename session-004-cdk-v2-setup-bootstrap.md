# Sessão 004 — CDK v2: setup, bootstrap e estrutura de projeto

**Duração estimada:** 60 minutos
**Pré-requisitos:** session-003 — CloudFormation: changesets, drift detection e stack policies

---

## Objetivo

Ao final, você conseguirá inicializar um projeto CDK v2 em TypeScript ou Python, executar `cdk bootstrap` em uma conta/região, entender o que o bootstrap provisiona (S3 bucket, ECR, IAM roles), e executar `cdk synth`, `cdk diff` e `cdk deploy` em um stack simples.

---

## Contexto

[FATO] O AWS CDK (Cloud Development Kit) é um framework open-source lançado em 2019 que permite definir infraestrutura em linguagens de programação de propósito geral (TypeScript, Python, Java, Go, C#). O CDK não substitui o CloudFormation — ele gera CloudFormation como artefato de saída. Todo `cdk deploy` termina em um changeset do CloudFormation sendo executado.

[CONSENSO] A principal vantagem do CDK sobre CloudFormation puro não é apenas sintaxe — é abstração com escape hatch. Você escreve menos para casos comuns (L2 constructs cuidam de permissões, logs, e defaults seguros automaticamente), mas sempre pode descer para o nível do CloudFormation quando precisar de controle total. Essa é a proposta central do modelo de constructs L1/L2/L3, que a sessão 005 cobre em profundidade.

[FATO] CDK v2 foi lançado em dezembro de 2021. A principal diferença em relação ao v1 é que todos os constructs de serviços AWS foram consolidados em um único pacote (`aws-cdk-lib`) em vez de pacotes separados por serviço. CDK v1 chegou ao end-of-support em junho de 2023. Use sempre v2.

---

## Conceitos principais

### 1. Modelo mental: CDK é um compilador de infraestrutura

O CDK transforma código em CloudFormation. O fluxo completo é:

```
Seu código (TypeScript/Python)
        │
        ▼ cdk synth
Cloud Assembly (cdk.out/)
  ├── MeuStack.template.json   ← CloudFormation template gerado
  ├── asset.XXXX/              ← arquivos locais a fazer upload (Lambda code, etc.)
  └── manifest.json            ← metadados do assembly

        │
        ▼ cdk deploy
  Upload assets → S3/ECR (bucket do bootstrap)
  Cria/atualiza stack CloudFormation via changeset
        │
        ▼
  Recursos AWS criados
```

Quando um deploy falha, o erro real está no CloudFormation — não no CDK. Por isso conhecer CloudFormation (sessões 002-003) é pré-requisito essencial.

---

### 2. Instalação e pré-requisitos

[FATO] O CDK CLI é um pacote npm. O runtime mínimo é Node.js 18.x (mesmo que você escreva em Python ou Go — o CLI é sempre Node.js).

```bash
# Verificar versão do Node
node --version   # precisa ser >= 18.x

# Instalar o CDK CLI globalmente
npm install -g aws-cdk

# Verificar versão instalada
cdk --version    # ex: 2.176.0 (build xxxxxxx)

# Para Python: instalar a lib no projeto (não no sistema)
pip install aws-cdk-lib constructs
```

[CONSENSO] Instalar o CDK CLI globalmente (`-g`) é conveniente para uso local, mas em pipelines CI/CD prefira instalar localmente no projeto (`npm install aws-cdk`) para fixar a versão. A versão do CLI e da `aws-cdk-lib` no `package.json` devem ser compatíveis — diferenças de versão entre CLI e lib podem causar warnings ou erros de synth.

---

### 3. Estrutura de um projeto CDK v2

**Inicializar um novo projeto:**

```bash
mkdir meu-projeto && cd meu-projeto

# TypeScript (mais comum em exemplos oficiais)
cdk init app --language typescript

# Python
cdk init app --language python
```

**Estrutura gerada para TypeScript:**

```
meu-projeto/
├── bin/
│   └── meu-projeto.ts      ← Entry point: instancia o App e os Stacks
├── lib/
│   └── meu-projeto-stack.ts ← Definição dos constructs (onde você trabalha)
├── test/
│   └── meu-projeto.test.ts ← Testes com assertions (sessão 008)
├── cdk.json                ← Configuração do CDK CLI
├── package.json
├── tsconfig.json
└── node_modules/
```

**Estrutura gerada para Python:**

```
meu-projeto/
├── app.py                  ← Entry point
├── meu_projeto/
│   ├── __init__.py
│   └── meu_projeto_stack.py ← Definição dos constructs
├── tests/
├── cdk.json
├── requirements.txt
└── .venv/                  ← virtualenv (criar manualmente: python -m venv .venv)
```

**O entry point (bin/meu-projeto.ts):**

```typescript
import * as cdk from 'aws-cdk-lib';
import { MeuProjetoStack } from '../lib/meu-projeto-stack';

const app = new cdk.App();

new MeuProjetoStack(app, 'MeuProjetoStack', {
  env: {
    account: process.env.CDK_DEFAULT_ACCOUNT,
    region: process.env.CDK_DEFAULT_REGION,
  },
});
```

[FATO] `CDK_DEFAULT_ACCOUNT` e `CDK_DEFAULT_REGION` são variáveis de ambiente populadas automaticamente pelo CDK CLI com base no perfil AWS ativo. Hardcodar account/region é possível mas não recomendado — impede reusar o mesmo código em múltiplas contas.

**A stack (lib/meu-projeto-stack.ts):**

```typescript
import * as cdk from 'aws-cdk-lib';
import * as s3 from 'aws-cdk-lib/aws-s3';
import { Construct } from 'constructs';

export class MeuProjetoStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // Construct L2 — nível de abstração alto
    new s3.Bucket(this, 'MeuBucket', {
      versioned: true,
      encryption: s3.BucketEncryption.S3_MANAGED,
      removalPolicy: cdk.RemovalPolicy.DESTROY,   // equivalente a DeletionPolicy
      autoDeleteObjects: true,
    });
  }
}
```

**Hierarquia de constructs:**

```
App                          ← raiz de tudo
  └── Stack (MeuProjetoStack)
        └── Bucket (MeuBucket)
              └── BucketPolicy (gerado automaticamente pelo L2)
```

Cada nó na árvore tem um ID único dentro do pai. O CDK usa esse path para gerar logical IDs no CloudFormation: `MeuProjetoStackMeuBucketXXXXXXXX`.

---

### 4. Bootstrap — o que é e o que provisiona

**Por que o bootstrap é necessário:**

Quando o CDK faz deploy de assets (código Lambda, imagens Docker, arquivos locais), ele precisa de um lugar para armazená-los antes de passar para o CloudFormation. O bootstrap cria essa infraestrutura de suporte.

[FATO] O `cdk bootstrap` implanta uma stack CloudFormation chamada `CDKToolkit` na conta/região. Esta stack provisiona:

```
CDKToolkit stack
├── S3 Bucket (cdk-xxxxxxxx-assets-ACCOUNT-REGION)
│     └── Armazena templates sintetizados e assets locais
│         Versionado e com lifecycle policy de 365 dias
│
├── ECR Repository (cdk-xxxxxxxx-container-assets-ACCOUNT-REGION)
│     └── Armazena imagens Docker para Lambda/ECS
│
└── IAM Roles (6 roles criadas por padrão)
      ├── cdk-xxxxxxxx-lookup-role-ACCOUNT-REGION
      │     └── Permite ao CDK CLI fazer lookups (VPCs, AMIs, etc.)
      ├── cdk-xxxxxxxx-file-publishing-role-ACCOUNT-REGION
      │     └── Upload de assets para o S3
      ├── cdk-xxxxxxxx-image-publishing-role-ACCOUNT-REGION
      │     └── Push de imagens para o ECR
      ├── cdk-xxxxxxxx-deploy-role-ACCOUNT-REGION
      │     └── Criação/atualização de stacks CloudFormation
      ├── cdk-xxxxxxxx-cfn-exec-role-ACCOUNT-REGION
      │     └── Role que o CloudFormation assume para criar recursos
      └── cdk-xxxxxxxx-readonly-role-ACCOUNT-REGION (v2 recente)
            └── Permissões de leitura para diff e synth
```

**Executando o bootstrap:**

```bash
# Bootstrap na conta/região atual (usa o perfil padrão)
cdk bootstrap

# Bootstrap explicitando conta e região
cdk bootstrap aws://123456789012/us-east-1

# Bootstrap com perfil específico
cdk bootstrap --profile prod aws://123456789012/us-east-1

# Bootstrap multi-região (executar uma vez por região)
cdk bootstrap aws://123456789012/us-east-1
cdk bootstrap aws://123456789012/sa-east-1

# Ver o que seria criado sem executar
cdk bootstrap --show-template
```

**Bootstrap qualifier:**

[FATO] O qualifier é um identificador de 9 caracteres (padrão: `hnb659fds`) que faz parte dos nomes dos recursos do bootstrap. Ele permite ter múltiplos bootstraps na mesma conta/região — útil quando times diferentes precisam de isolamento de assets.

```bash
# Bootstrap com qualifier customizado
cdk bootstrap --qualifier meutime

# No cdk.json, referenciar o qualifier
{
  "@aws-cdk/core:bootstrapQualifier": "meutime"
}
```

**Quando re-executar o bootstrap:**

O bootstrap tem um número de versão. Ao atualizar o CDK CLI para uma versão maior, pode ser necessário atualizar o bootstrap. O CDK avisa durante o `cdk deploy` se a versão do bootstrap está desatualizada:

```
❌ This CDK deployment requires bootstrap stack version '6', found '4'.
   Please run 'cdk bootstrap'.
```

Rodar `cdk bootstrap` numa conta/região já bootstrapada é seguro — ele atualiza os recursos existentes sem recriar.

---

### 5. Os comandos principais do CDK CLI

**`cdk synth` — compilar para CloudFormation:**

```bash
# Sintetizar todos os stacks
cdk synth

# Sintetizar um stack específico
cdk synth MeuProjetoStack

# Ver o template gerado no terminal (sem salvar em cdk.out/)
cdk synth --quiet

# Output em cdk.out/ por padrão
ls cdk.out/
# MeuProjetoStack.template.json
# manifest.json
# tree.json
```

`cdk synth` é o equivalente a "compilar" — transforma código em CloudFormation. Ele detecta erros de configuração antes de qualquer chamada à AWS.

**`cdk diff` — ver o que vai mudar:**

```bash
# Diff entre o código local e a stack deployada
cdk diff

# Diff de um stack específico
cdk diff MeuProjetoStack

# Diff contra um template salvo (sem stack deployada)
cdk diff --template cdk.out/MeuProjetoStack.template.json
```

`cdk diff` é o equivalente ao changeset da sessão anterior — mas calculado localmente, sem criar nada na AWS. É mais rápido que um changeset real, mas menos preciso para recursos com valores resolvidos em runtime.

**`cdk deploy` — implantar:**

```bash
# Deploy com aprovação interativa de IAM changes
cdk deploy

# Deploy sem aprovação (útil em pipelines)
cdk deploy --require-approval never

# Deploy de múltiplos stacks
cdk deploy Stack1 Stack2

# Deploy de todos os stacks
cdk deploy "*"

# Deploy com hotswap (para desenvolvimento — evita CloudFormation para mudanças em Lambda)
cdk deploy --hotswap

# Deploy e mostrar os outputs da stack
cdk deploy --outputs-file outputs.json
```

[FATO] `cdk deploy --hotswap` detecta mudanças que são apenas em código de Lambda ou definições de ECS e as aplica diretamente (sem CloudFormation), reduzindo o tempo de feedback de ~2 minutos para ~10 segundos. **Nunca use hotswap em produção** — ele pode deixar o estado do CloudFormation dessincronizado com o estado real.

**`cdk destroy` — destruir a stack:**

```bash
cdk destroy MeuProjetoStack
# Pede confirmação interativa

cdk destroy --force MeuProjetoStack
# Sem confirmação
```

**Diagrama do fluxo de comandos:**

```
Código CDK
    │
    ├─ cdk synth ──► cdk.out/ (CloudFormation templates + assets)
    │
    ├─ cdk diff  ──► Comparação local vs. stack deployada (sem chamadas AWS)
    │
    ├─ cdk deploy
    │       ├── cdk synth (implícito)
    │       ├── Upload assets → S3/ECR (via roles do bootstrap)
    │       ├── Cria changeset CloudFormation
    │       ├── Mostra IAM changes para aprovação (se houver)
    │       └── Executa changeset → aguarda conclusão
    │
    └─ cdk destroy
            ├── Cria changeset de deleção
            └── Executa → deleta todos os recursos (respeitando DeletionPolicy)
```

---

### 6. cdk.json — configuração do projeto

O `cdk.json` é o arquivo de configuração do CDK CLI para o projeto. Todo `cdk` executado na pasta lê esse arquivo.

```json
{
  "app": "npx ts-node --prefer-ts-exts bin/meu-projeto.ts",
  "watch": {
    "include": ["**"],
    "exclude": ["README.md", "cdk*.json", "**/*.d.ts", "**/*.js", "tsconfig.json",
                "package*.json", ".git/", "node_modules/", "cdk.out/"]
  },
  "context": {
    "@aws-cdk/aws-lambda:recognizeLayerVersion": true,
    "@aws-cdk/core:checkSecretUsage": true,
    "@aws-cdk/aws-iam:minimizePolicies": true,
    "@aws-cdk/aws-s3:serverAccessLogsUseBucketPolicy": true
  }
}
```

**Campos principais:**

```
app       → comando para executar o entry point (compilador/runtime)
watch     → arquivos a monitorar para cdk watch (hot reload em dev)
context   → feature flags e valores de contexto persistidos
```

[FATO] Os valores em `context` que começam com `@aws-cdk/` são feature flags. Eles controlam comportamentos que mudaram entre versões do CDK e permitem migração gradual. Projetos novos criados com `cdk init` recebem automaticamente o conjunto de feature flags da versão atual.

**O que vai no gitignore:**

```
cdk.out/              ← sempre ignorar (artefato de build)
cdk.context.json      ← DEPENDE (veja sessão 012 para a discussão completa)
node_modules/         ← sempre ignorar
.venv/                ← sempre ignorar (Python)
```

---

## Exemplo prático

Criar um projeto CDK do zero e fazer o primeiro deploy:

```bash
# 1. Criar e inicializar o projeto
mkdir demo-cdk && cd demo-cdk
cdk init app --language typescript
npm install

# 2. Bootstrap (uma vez por conta/região)
cdk bootstrap --profile dev

# 3. Editar lib/demo-cdk-stack.ts
cat > lib/demo-cdk-stack.ts << 'EOF'
import * as cdk from 'aws-cdk-lib';
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as iam from 'aws-cdk-lib/aws-iam';
import { Construct } from 'constructs';

export class DemoCdkStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const bucket = new s3.Bucket(this, 'DemoBucket', {
      versioned: true,
      encryption: s3.BucketEncryption.S3_MANAGED,
      blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
      removalPolicy: cdk.RemovalPolicy.DESTROY,
      autoDeleteObjects: true,
    });

    const appRole = new iam.Role(this, 'AppRole', {
      assumedBy: new iam.ServicePrincipal('lambda.amazonaws.com'),
      managedPolicies: [
        iam.ManagedPolicy.fromAwsManagedPolicyName('service-role/AWSLambdaBasicExecutionRole'),
      ],
    });

    // L2 cuida das permissões — gera a policy correta automaticamente
    bucket.grantReadWrite(appRole);

    // Outputs
    new cdk.CfnOutput(this, 'BucketName', { value: bucket.bucketName });
    new cdk.CfnOutput(this, 'AppRoleArn', { value: appRole.roleArn });
  }
}
EOF

# 4. Sintetizar — ver o CloudFormation gerado
cdk synth

# 5. Ver o que vai ser criado (diff contra stack inexistente)
cdk diff --profile dev

# 6. Deploy
cdk deploy --profile dev

# 7. Ver o CloudFormation template gerado (para comparar com o que você escreveria)
cat cdk.out/DemoCdkStack.template.json | jq '.Resources | keys'

# 8. Comparar o que o L2 gerou vs o que você escreveria manualmente
# O bucket.grantReadWrite(appRole) gera automaticamente uma IAM Policy
# com as permissões corretas — sem precisar escrever o ARN do bucket
cat cdk.out/DemoCdkStack.template.json | jq '.Resources | to_entries[] | select(.value.Type == "AWS::IAM::Policy")'
```

---

## Armadilhas comuns

**1. Esquecer de re-executar o bootstrap após atualização do CDK**

Após `npm install -g aws-cdk@latest`, se a nova versão exige uma versão mais recente do bootstrap, o próximo `cdk deploy` falha com a mensagem de versão incompatível. A solução é simples (`cdk bootstrap`), mas confunde quem não sabe por que está falhando.

**2. `RemovalPolicy.DESTROY` em produção**

O CDK usa `RemovalPolicy.RETAIN` como padrão para a maioria dos recursos stateful (S3, RDS, DynamoDB). Mudar para `DESTROY` faz o `cdk destroy` deletar o recurso e seus dados. Em projetos criados a partir de tutoriais, é comum esquecer esse valor copiado de exemplos de desenvolvimento ao promover para produção.

**3. Logical ID instável — o rename problem**

O CDK deriva o logical ID do CloudFormation a partir do path de constructs na árvore (`Stack > Construct > Resource`). Se você renomear um construct ou mover um recurso na hierarquia, o logical ID muda — e o CloudFormation interpreta isso como "deletar o antigo e criar um novo". Para recursos stateful (RDS, S3 com dados), isso é destrutivo. A solução é usar `overrideLogicalId()` para fixar o ID quando necessário.

```typescript
// Fixar o logical ID para evitar substituição acidental ao refatorar
const cfnBucket = bucket.node.defaultChild as s3.CfnBucket;
cfnBucket.overrideLogicalId('MeuBucketEstavel');
```

---

## Exercício de reflexão

Você está migrando uma stack CloudFormation existente para CDK. O template original tem um bucket S3 com logical ID `AppBucket`. Ao escrever o código CDK, você cria o bucket assim:

```typescript
new s3.Bucket(this, 'AppBucket', { versioned: true });
```

O CDK vai gerar um logical ID diferente do original (algo como `AppBucketXXXXXXXX` com hash). Se você fizer `cdk deploy` sobre a stack existente, o CloudFormation vai tentar criar um novo bucket e deletar o original.

Quais são as duas abordagens para resolver esse problema? Para cada uma, descreva os passos concretos e os trade-offs. Considere também: por que o CDK adiciona esse hash ao logical ID e qual problema ele resolve na vida normal de um projeto CDK.

---

## Recursos para aprofundar

**Setup e primeiro projeto:**
- [Getting started with the AWS CDK](https://docs.aws.amazon.com/cdk/v2/guide/getting_started.html) — pré-requisitos, instalação, e o tutorial `hello-world` com TypeScript e Python.

**Bootstrap em profundidade:**
- [AWS CDK bootstrapping](https://docs.aws.amazon.com/cdk/v2/guide/bootstrapping.html) — lista todos os recursos criados, versões do bootstrap, e quando re-executar.

**CLI reference:**
- [AWS CDK CLI reference](https://docs.aws.amazon.com/cdk/v2/guide/cli.html) — todos os comandos com flags, incluindo `--hotswap`, `--watch`, `--outputs-file` e opções de aprovação.

**CDK Workshop (prático):**
- [CDK Workshop](https://cdkworkshop.com) — tutorial hands-on com TypeScript ou Python que cobre App → Stack → Constructs → Pipeline em ~2 horas. Recomendado para consolidar essa sessão na prática.

---

