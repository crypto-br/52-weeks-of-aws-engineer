# Sessão 12 — CDK: Context, feature flags e cdk.json production-grade

**Duração estimada:** 60 minutos
**Pré-requisitos:** session-011-cdk-custom-resources-aspects

---

## Objetivo

Ao final, você conseguirá usar `cdk.context.json` para cachear lookups (VPCs, AMIs) sem depender de resolução em runtime, configurar feature flags para controlar comportamentos de migração, e estruturar `cdk.json` para equipes (o que vai no gitignore vs o que é versionado).

---

## Contexto

[FATO] Uma das propriedades fundamentais de uma boa pipeline de infraestrutura é a **reprodutibilidade**: dado o mesmo código-fonte, dois engenheiros em máquinas diferentes ou dois builds de CI produzem exatamente o mesmo CloudFormation template. O CDK quebra essa propriedade se qualquer lookup de conta AWS é feito em tempo de síntese sem cache — porque a resposta pode mudar (uma nova AMI foi publicada, uma nova AZ foi habilitada) entre o build do desenvolvedor e o build do CI.

[FATO] O mecanismo que o CDK usa para tornar lookups reproduzíveis chama-se **Context**. Context é um dicionário chave-valor que a CLI do CDK popula automaticamente na primeira vez que um lookup é necessário e persiste em `cdk.context.json`. Nas sínteses seguintes, o valor cacheado é usado sem nova consulta à AWS. O arquivo `cdk.context.json` é versionado no repositório — essa é a garantia de reprodutibilidade.

[CONSENSO] Feature flags são uma sobreposição ao mesmo mecanismo de Context: são booleanos armazenados no `cdk.json` sob a chave `"context"` que controlam comportamentos do CDK que mudaram entre versões e que poderiam quebrar stacks existentes se ativados silenciosamente. Entendê-los é obrigatório ao migrar stacks de CDK v1 para v2, e ao atualizar projetos CDK v2 entre versões menores que introduziram mudanças opt-in.

---

## Conceitos principais

### 1. O ciclo de vida de um Context lookup

Quando seu código CDK chama algo como `ec2.Vpc.fromLookup(this, 'Vpc', { vpcName: 'prod-vpc' })`, o CDK precisa consultar a conta AWS para descobrir o ID da VPC, suas subnets e AZs. Esse processo tem dois modos:

```
                    ┌─────────────────────────────────────────────┐
                    │  cdk synth (primeira vez, sem cache)        │
                    │                                             │
  Código CDK        │  1. Tentativa de síntese                   │
  Vpc.fromLookup()  │     → CDK detecta lookup pendente           │
        │           │     → Sintetiza com valores DUMMY           │
        │           │     → NÃO gera template válido              │
        │           │  2. CLI faz describe-vpcs na AWS            │
        │           │     → armazena em cdk.context.json          │
        │           │  3. CDK tenta síntese novamente             │
        │           │     → lookup resolvido do cache             │
        │           │     → template válido gerado                │
        └───────────┘

                    ┌─────────────────────────────────────────────┐
                    │  cdk synth (com cache / CI)                 │
                    │                                             │
  Código CDK        │  1. CDK lê cdk.context.json                │
  Vpc.fromLookup()  │     → lookup já resolvido                  │
        │           │  2. Template gerado diretamente             │
        │           │     → zero chamadas à AWS necessárias       │
        └───────────┘
```

[FATO] No modo de CI (sem credenciais de conta, ou com `--no-lookups`), se o `cdk.context.json` não tiver o valor cacheado, a síntese falha com erro explícito. Isso é intencional — garante que o CI não depende de acesso de escrita ao cache para funcionar.

[FATO] O conteúdo de `cdk.context.json` após um lookup de VPC se parece com isto:

```json
{
  "vpc-provider:account=123456789012:filter.vpc-id=vpc-0abc:region=us-east-1:returnAsymmetricSubnets=true": {
    "vpcId": "vpc-0abc",
    "vpcCidrBlock": "10.0.0.0/16",
    "availabilityZones": ["us-east-1a", "us-east-1b", "us-east-1c"],
    "subnetGroups": [
      {
        "name": "Public",
        "type": "Public",
        "subnets": [
          { "subnetId": "subnet-pub1", "cidr": "10.0.0.0/24", "availabilityZone": "us-east-1a", "routeTableId": "rtb-1" }
        ]
      }
    ]
  }
}
```

A chave é deterministicamente gerada a partir dos parâmetros do lookup. Se você mudar qualquer parâmetro (ex: o `vpcName`), o CDK gera uma chave diferente e faz um novo lookup.

---

### 2. Lookups disponíveis nativamente no CDK

[FATO] O CDK oferece os seguintes lookups de conta de primeira classe (todos cacheados em `cdk.context.json`):

| Método de lookup | O que consulta na AWS | Chave usada |
|---|---|---|
| `ec2.Vpc.fromLookup()` | `ec2:DescribeVpcs` + `ec2:DescribeSubnets` | filter params |
| `ec2.MachineImage.lookup()` | `ec2:DescribeImages` | filtros de AMI |
| `ssm.StringParameter.valueFromLookup()` | `ssm:GetParameter` | param name + account/region |
| `HostedZone.fromLookup()` | `route53:ListHostedZonesByName` | domínio |
| `ec2.SecurityGroup.fromLookupByName()` | `ec2:DescribeSecurityGroups` | nome + VPC ID |

[FATO] `ssm.StringParameter.valueFromLookup()` é diferente de `ssm.StringParameter.valueForStringParameter()`:

```typescript
// valueFromLookup: resolve em tempo de síntese, cacheia em cdk.context.json
// → valor concreto no template; NUNCA use para secrets!
const amiId = ssm.StringParameter.valueFromLookup(this, '/shared/ami-id');
// resultado: "ami-0abc123" (string concreta)

// valueForStringParameter: resolve em tempo de deploy (CloudFormation)
// → gera um token {{resolve:ssm:/shared/ami-id}} no template
// → roda uma chamada SSM no momento do deploy, não da síntese
const endpoint = ssm.StringParameter.valueForStringParameter(this, '/shared/db-endpoint');
// resultado: Token (resolvido pelo CloudFormation)
```

[CONSENSO] Para parâmetros que mudam com frequência (configurações, endpoints de ambiente), prefira `valueForStringParameter` — o template sempre usa o valor mais recente ao ser deployado. Para parâmetros estáveis que são referenciados em constructs que precisam do valor em tempo de síntese (ex: ID de uma VPC para fazer subnets lookup), `valueFromLookup` é necessário.

---

### 3. Gerenciamento do cdk.context.json em equipes

[FATO] Segundo a documentação oficial do CDK: `cdk.context.json` **deve ser versionado no repositório**. A razão é que ambientes de CI frequentemente não têm (e não devem ter) credenciais com permissão de describe na conta de produção. O cache é a âncora de reprodutibilidade.

[FATO] O comando `cdk context` gerencia o cache:

```bash
# lista todas as entradas do cache
cdk context

# remove uma entrada específica (força re-lookup na próxima síntese)
cdk context --reset "vpc-provider:account=123456789012:..."

# limpa todo o cache
cdk context --clear

# força síntese ignorando lookups (falha se algum lookup não estiver cacheado)
cdk synth --no-lookups
```

[CONSENSO] A política recomendada pela comunidade CDK para times é:

```
cdk.json          → versionado (configuração do projeto, feature flags)
cdk.context.json  → versionado (cache de lookups — necessário para CI)
cdk.out/          → gitignore (artefatos de síntese, gerados automaticamente)
node_modules/     → gitignore (dependências npm)
```

Nunca coloque `cdk.context.json` no `.gitignore`. Se fizer isso, todo desenvolvedor novo e todo build de CI vai fazer lookups na conta, o que:
1. Exige credenciais com permissões elevadas no CI.
2. Torna o build não-determinístico (um novo recurso na conta pode mudar o resultado).
3. Lentifica a síntese (chamadas de API extras).

---

### 4. Context como mecanismo de configuração explícita

Além dos lookups automáticos, você pode usar Context como um sistema de configuração intencional — passando valores ao app via `cdk.json`, variáveis de ambiente ou CLI:

```typescript
// lib/app.ts
const app = new App();

// Lê o ambiente alvo do context
// Pode ser passado via: cdk deploy --context env=production
const env = app.node.tryGetContext('env') ?? 'development';

const config = {
  development: { accountId: '111111111111', region: 'us-east-1', instanceType: 't3.micro' },
  staging:     { accountId: '222222222222', region: 'us-east-1', instanceType: 't3.small' },
  production:  { accountId: '333333333333', region: 'us-east-1', instanceType: 'm5.large' },
};

const targetEnv = config[env as keyof typeof config];
if (!targetEnv) throw new Error(`Ambiente inválido: ${env}`);

new ApiStack(app, `Api-${env}`, {
  env: { account: targetEnv.accountId, region: targetEnv.region },
  instanceType: new ec2.InstanceType(targetEnv.instanceType),
});
```

No `cdk.json`, você pode definir o valor padrão:

```json
{
  "app": "npx ts-node --prefer-ts-exts bin/app.ts",
  "context": {
    "env": "development"
  }
}
```

[FATO] A precedência de Context é, da mais alta para a mais baixa:
1. `--context key=value` na linha de comando da CLI
2. `~/.cdk.json` (arquivo global do usuário)
3. `cdk.context.json` (cache de lookups no projeto)
4. `cdk.json` (configuração do projeto)
5. `app.node.setContext()` no código

---

### 5. Feature Flags: o que são e quando importam

[FATO] Feature flags no CDK são booleanos de Context que têm o prefixo de módulo como namespace (ex: `@aws-cdk/aws-s3:serverAccessLogsUseBucketPolicy`). Eles controlam comportamentos que foram **alterados** em alguma versão do CDK mas que precisavam ser opt-in para não quebrar stacks existentes.

[FATO] No CDK v2, os feature flags históricos do v1 foram todos habilitados por padrão. Mas o CDK v2 continua introduzindo novos feature flags para mudanças que afetam stacks existentes. Quando você cria um novo projeto com `cdk init`, o `cdk.json` gerado inclui todos os feature flags da versão atual ativados — isso garante que novos projetos usam o comportamento mais moderno.

Exemplo de `cdk.json` gerado por `cdk init` (versão ~2.100+):

```json
{
  "app": "npx ts-node --prefer-ts-exts bin/myapp.ts",
  "watch": {
    "include": ["**"],
    "exclude": [
      "README.md",
      "cdk*.json",
      "**/*.d.ts",
      "**/*.js",
      "node_modules",
      "test"
    ]
  },
  "context": {
    "@aws-cdk/aws-lambda:recognizeLayerVersion": true,
    "@aws-cdk/core:checkSecretUsage": true,
    "@aws-cdk/core:target-partitions": ["aws", "aws-cn"],
    "@aws-cdk-containers/ecs-service-extensions:enableDefaultLogDriver": true,
    "@aws-cdk/aws-ec2:uniqueImdsv2TemplateName": true,
    "@aws-cdk/aws-ecs:arnFormatIncludesClusterName": true,
    "@aws-cdk/aws-iam:minimizePolicies": true,
    "@aws-cdk/core:validateSnapshotRemovalPolicy": true,
    "@aws-cdk/aws-codepipeline:crossAccountKeyAliasStackSafeResourceName": true,
    "@aws-cdk/aws-s3:createDefaultLoggingPolicy": true,
    "@aws-cdk/aws-sns-subscriptions:restrictSqsDescryption": true,
    "@aws-cdk/aws-apigateway:disableCloudWatchRole": true,
    "@aws-cdk/core:enablePartitionLiterals": true,
    "@aws-cdk/aws-events:eventsTargetQueueSameAccount": true,
    "@aws-cdk/aws-iam:standardizedServicePrincipals": true,
    "@aws-cdk/aws-ecs:disableExplicitDeploymentControllerForCircuitBreaker": true,
    "@aws-cdk/aws-iam:importedRoleStackSafeDefaultPolicyName": true,
    "@aws-cdk/aws-s3:serverAccessLogsUseBucketPolicy": true,
    "@aws-cdk/aws-route53-patters:useCertificate": true,
    "@aws-cdk/customresources:installLatestAwsSdkDefault": false,
    "@aws-cdk/aws-rds:databaseProxyUniqueResourceName": true,
    "@aws-cdk/aws-codedeploy:removeAlarmsFromDeploymentGroup": true,
    "@aws-cdk/aws-apigateway:authorizerChangeDeploymentLogicalId": true,
    "@aws-cdk/aws-ec2:launchTemplateDefaultUserData": true,
    "@aws-cdk/aws-secretsmanager:useAttachedSecretResourcePolicyForSecretTargetAttachments": true,
    "@aws-cdk/aws-redshift:columnId": true,
    "@aws-cdk/aws-cloudfront-origins:useOriginAccessControlForS3Origins": true,
    "@aws-cdk/core:enableCfnParameters": false
  }
}
```

[FATO] O flag `@aws-cdk/customresources:installLatestAwsSdkDefault` merece atenção: quando `true`, a Lambda interna do `AwsCustomResource` usa o SDK AWS mais recente disponível (não o bundled com o runtime Node). Isso pode causar falhas se a API que você está chamando mudou. O padrão mudou para `false` em versões recentes justamente por isso — preferível usar o SDK bundled e previsível.

---

### 6. Estrutura production-grade do cdk.json

[CONSENSO] Para projetos de time, o `cdk.json` deve ser tratado como um arquivo de configuração do projeto, não apenas um arquivo gerado pelo `cdk init`. Uma estrutura completa comentada:

```json
{
  // comando de entrada do app CDK
  "app": "npx ts-node --prefer-ts-exts bin/app.ts",

  // configuração do modo watch (cdk watch / cdk deploy --watch)
  "watch": {
    "include": ["**"],
    "exclude": [
      "README.md",
      "cdk*.json",
      "**/*.d.ts",
      "**/*.js",
      "node_modules",
      "test"
    ]
  },

  // configuração do build antes de cdk deploy/diff/synth
  // "build": "npm run build",  ← descomente se não usar ts-node diretamente

  // todo context vai aqui
  "context": {
    // ---- Configuração do projeto ----
    "env": "development",          // substituído via --context no CI/CD
    "owner": "platform-team",      // metadado para tagging
    "costCenter": "infra-001",

    // ---- Feature flags CDK ----
    // (os flags abaixo são os padrão para projetos novos)
    "@aws-cdk/aws-iam:minimizePolicies": true,
    "@aws-cdk/aws-s3:serverAccessLogsUseBucketPolicy": true,
    "@aws-cdk/customresources:installLatestAwsSdkDefault": false,
    // ... outros flags conforme a versão do CDK usada

    // ---- Lookups cacheados ----
    // (preenchidos automaticamente pelo CDK — não edite manualmente)
    // "vpc-provider:account=...": { ... }
  }
}
```

[CONSENSO] Separar contexto de configuração de projeto (como `env`, `owner`) dos feature flags melhora a legibilidade, mas não é uma estrutura imposta pelo CDK — é apenas organização por comentários.

**O que vai no `.gitignore`:**

```gitignore
# Artefatos de síntese (gerados pelo cdk synth)
cdk.out/

# Dependências Node
node_modules/

# TypeScript compilado (se não usar ts-node direto)
# *.js
# *.d.ts
```

**O que NÃO vai no `.gitignore`:**

```
cdk.json          ← configuração do projeto, feature flags
cdk.context.json  ← cache de lookups (crítico para CI reprodutível)
```

---

## Exemplo prático

**Cenário:** Você tem uma CDK app multi-ambiente que precisa:
1. Usar a VPC existente de cada conta (não criar uma nova).
2. Ter comportamento determinístico no CI sem credenciais de describe.
3. Suportar `cdk deploy --context env=staging` para deployar em staging sem mudar código.

### Estrutura do projeto

```
bin/
  app.ts                 ← entry point, lê context 'env'
lib/
  stacks/
    api-stack.ts         ← usa Vpc.fromLookup()
config/
  environments.ts        ← mapa de configuração por ambiente
cdk.json                 ← feature flags + default env
cdk.context.json         ← cache de lookups (versionado)
```

### `config/environments.ts`

```typescript
import { Environment } from 'aws-cdk-lib';

export interface EnvConfig {
  env: Environment;
  vpcName: string;
  hostedZoneName: string;
  instanceType: string;
}

export const environments: Record<string, EnvConfig> = {
  development: {
    env: { account: '111111111111', region: 'us-east-1' },
    vpcName: 'dev-vpc',
    hostedZoneName: 'dev.example.com',
    instanceType: 't3.micro',
  },
  staging: {
    env: { account: '222222222222', region: 'us-east-1' },
    vpcName: 'staging-vpc',
    hostedZoneName: 'staging.example.com',
    instanceType: 't3.small',
  },
  production: {
    env: { account: '333333333333', region: 'us-east-1' },
    vpcName: 'prod-vpc',
    hostedZoneName: 'example.com',
    instanceType: 'm5.large',
  },
};
```

### `lib/stacks/api-stack.ts`

```typescript
import { Stack, StackProps } from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as route53 from 'aws-cdk-lib/aws-route53';
import { Construct } from 'constructs';
import { EnvConfig } from '../../config/environments';

interface ApiStackProps extends StackProps {
  config: EnvConfig;
}

export class ApiStack extends Stack {
  constructor(scope: Construct, id: string, props: ApiStackProps) {
    super(scope, id, props);

    // Lookup da VPC existente na conta
    // → na primeira execução: CDK consulta a AWS e salva em cdk.context.json
    // → nas execuções seguintes (inclusive CI): lê do cache
    const vpc = ec2.Vpc.fromLookup(this, 'Vpc', {
      vpcName: props.config.vpcName,
    });

    // Lookup da Hosted Zone
    const zone = route53.HostedZone.fromLookup(this, 'Zone', {
      domainName: props.config.hostedZoneName,
    });

    // O resto usa vpc e zone como se fossem constructs normais
    // (mesmo sendo lookups, têm os métodos/propriedades esperados)
  }
}
```

### `bin/app.ts`

```typescript
import { App } from 'aws-cdk-lib';
import { ApiStack } from '../lib/stacks/api-stack';
import { environments } from '../config/environments';

const app = new App();

const envName = app.node.tryGetContext('env') ?? 'development';
const config = environments[envName];

if (!config) {
  throw new Error(
    `Ambiente '${envName}' não encontrado. ` +
    `Ambientes disponíveis: ${Object.keys(environments).join(', ')}`
  );
}

new ApiStack(app, `Api-${envName}`, {
  env: config.env,
  config,
  // Tags globais aplicadas a todos os recursos desta stack
  tags: {
    Environment: envName,
    ManagedBy: 'CDK',
  },
});
```

### Workflow de onboarding de um novo ambiente

```bash
# 1. Primeiro deploy em staging: CDK vai fazer lookups na conta de staging
AWS_PROFILE=staging-profile cdk deploy --context env=staging

# → CDK consulta a AWS, popula cdk.context.json com as entradas de staging
# → commit o cdk.context.json atualizado

git add cdk.context.json
git commit -m "chore: cache context lookups for staging environment"

# 2. Builds de CI em staging agora são reproduzíveis sem credenciais de describe
# (o CI usa as entradas cacheadas)
```

---

## Armadilhas comuns

### Armadilha 1: Colocar cdk.context.json no .gitignore

**O erro:** Um desenvolvedor vê `cdk.context.json` como "arquivo gerado" e o adiciona ao `.gitignore`. O projeto funciona localmente porque cada dev tem credenciais AWS. O CI começa a falhar com erros como:

```
Error: Cannot retrieve value from context provider vpc-provider since account/region
are not specified at the stack level. Configure "env" for the stack...
```

Ou pior: o CI funciona mas produz templates diferentes em dias diferentes porque um novo recurso apareceu na conta (nova AMI, nova AZ).

**Por que acontece:** Sem o cache versionado, o CI precisa de credenciais de describe para fazer lookups em tempo real, e esses lookups são não-determinísticos.

**Como evitar:** `cdk.context.json` sempre no git. O único momento legítimo de não versionar é em projetos de prova de conceito estritamente locais.

---

### Armadilha 2: Cache de lookups obsoleto causando erros silenciosos

**O erro:** Você deletou e recriou a VPC `prod-vpc` na AWS (por qualquer razão). O `cdk.context.json` ainda tem as subnet IDs antigas. O `cdk synth` passa sem erro (usa o cache), o deploy começa, e no meio do deploy o CloudFormation tenta referenciar `subnet-old123` que não existe mais.

**Por que acontece:** O CDK nunca invalida o cache automaticamente — é um cache write-once, invalidado apenas manualmente. Ele não tem como saber que o recurso externo mudou.

**Como reconhecer:** Erros de deploy do tipo `subnet-id does not exist` ou `security group sg-XXX not found` quando você não mudou o código CDK.

**Como evitar:** Após qualquer mudança manual de infraestrutura que afete lookups (recriar VPCs, mudar names de recursos, etc.), execute `cdk context --clear` localmente, re-sintetize, commite o `cdk.context.json` atualizado.

---

### Armadilha 3: Feature flag novo na versão do CDK muda comportamento silenciosamente

**O erro:** Você atualiza `aws-cdk-lib` de `2.80.0` para `2.120.0`. O `package.json` e `package-lock.json` são atualizados, mas o `cdk.json` não recebe os novos feature flags da versão 2.120.0. Alguns constructs passam a ter comportamento diferente do esperado (um novo recurso IAM aparece, a lógica de nomeação de um recurso muda), mas nenhum erro é emitido.

**Por que acontece:** Feature flags novos não são adicionados retroativamente ao `cdk.json` de projetos existentes — só ao gerado por `cdk init` com a nova versão. Projetos existentes ficam com a configuração da versão de criação do projeto.

**Como reconhecer:** Depois de uma atualização de versão, execute `cdk diff` em um ambiente de não-produção e revise cada mudança. Se houver mudanças não esperadas (novos recursos, mudanças de nomes), verifique os changelogs e feature flags da versão.

**Como evitar:** Ao atualizar o CDK, compare o `cdk.json` do seu projeto com o `cdk.json` que `cdk init` geraria na nova versão. A documentação da versão lista os novos feature flags introduzidos. Ative-os deliberadamente após entender o impacto.

---

## Exercício de reflexão

Você está migrando uma CDK app de um único desenvolvedor para um repositório de time com uma pipeline de CI/CD automatizada. A app tem três stacks, e dois deles usam `Vpc.fromLookup()` e `MachineImage.lookup()`. O CI roda em um container Docker sem acesso às contas AWS de produção (por política de segurança — apenas a pipeline de deploy tem as credenciais, não o build de CI).

Como você estruturaria o processo para garantir que:
1. O CI produz templates idênticos aos que o desenvolvedor produziu localmente.
2. Quando a infraestrutura subjacente muda (ex: a VPC é recriada), há um processo definido e controlado para atualizar o cache.
3. Novos desenvolvedores não precisam ter credenciais das contas de produção para trabalhar no código.
4. Os feature flags são atualizados de forma deliberada e revisada quando o CDK é atualizado.

Descreva o workflow de commit, os arquivos versionados e não versionados, e o processo de atualização de cache que você adotaria.

---

## Recursos para aprofundar

### 1. Context values and the AWS CDK
**URL:** https://docs.aws.amazon.com/cdk/v2/guide/context.html
**O que encontrar:** Documentação completa do mecanismo de Context: tipos de lookups disponíveis, como o cache funciona, precedência de valores, comandos `cdk context`, e a distinção entre `valueFromLookup` e `valueForStringParameter`.
**Por que é a fonte certa:** É o guia oficial e canônico do mecanismo — mais completo que qualquer blog post.

### 2. AWS CDK feature flags
**URL:** https://docs.aws.amazon.com/cdk/v2/guide/featureflags.html
**O que encontrar:** Lista completa de todos os feature flags do CDK v2, o que cada um controla, quando foi introduzido, e o comportamento com e sem o flag. Essencial ao migrar ou atualizar versões do CDK.
**Por que é a fonte certa:** É a referência autoritativa — os changelogs do CDK no GitHub referenciam este documento para cada flag introduzido.

### 3. CDK Best Practices — Context e projeto
**URL:** https://docs.aws.amazon.com/cdk/v2/guide/best-practices.html
**O que encontrar:** Seções sobre estrutura de projeto para times, gerenciamento de context em pipelines CI/CD, e a política de o que versionar vs o que ignorar no `.gitignore`.
**Por que é a fonte certa:** É o guia de boas práticas oficial da equipe do CDK, consolidando lições aprendidas de usuários em produção.

---

