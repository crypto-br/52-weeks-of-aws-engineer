# Sessão 009 — CDK Pipelines: bootstrap cross-account, OIDC connection e estrutura da pipeline

**Duração estimada:** 60 minutos
**Pré-requisitos:** session-008 — CDK: Testing com assertions

---

## Objetivo

Ao final, você conseguirá executar `cdk bootstrap` nas contas de deploy com trust policy apontando para a conta da pipeline, configurar a conexão com GitHub via AWS CodeStar Connections (OIDC — sem PAT), e criar uma CDK Pipeline com Source + Build + UpdatePipeline stages, entendendo o que cada stage faz antes do primeiro deploy real de aplicação.

---

## Contexto

[FATO] CDK Pipelines é um construct de alto nível (`aws-cdk-lib/pipelines`) que usa AWS CodePipeline como motor de execução. Ele adiciona uma camada de abstração opinionada sobre o CodePipeline, com duas funcionalidades centrais que o tornam diferente de uma pipeline comum: **self-mutation** (a pipeline atualiza a si mesma antes de qualquer deploy de aplicação) e **multi-account deployment nativo** (a pipeline roda numa conta e deploya em outras).

[CONSENSO] Para organizações com múltiplas contas AWS (dev, staging, prod), CDK Pipelines é o caminho mais direto para implementar um pipeline de CD completo usando apenas CDK — sem precisar configurar manualmente CodeBuild projects, IAM roles cross-account ou pipelines de múltiplos estágios no console.

---

## Conceitos principais

### 1. As contas envolvidas — a topologia multi-account

Uma CDK Pipeline sempre tem pelo menos dois tipos de conta:

```
┌─────────────────────────────────────────────────────────────┐
│  CONTA TOOLS / PIPELINE (onde a pipeline roda)              │
│  ┌──────────────────────────────────────┐                   │
│  │  CodePipeline                        │                   │
│  │    Source → Build → UpdatePipeline   │                   │
│  │    → [Stages da aplicação]           │                   │
│  │                                      │                   │
│  │  CodeBuild (synth + deploy)          │                   │
│  │  S3 (artifacts)                      │                   │
│  │  ECR (imagens)                       │                   │
│  └──────────────────────────────────────┘                   │
│           │           │           │                         │
└───────────┼───────────┼───────────┼─────────────────────────┘
            │ assume role           │ assume role
            ▼                       ▼
┌──────────────────┐    ┌──────────────────┐
│  CONTA DEV       │    │  CONTA PROD      │
│  (deploy target) │    │  (deploy target) │
│                  │    │                  │
│  CDKToolkit      │    │  CDKToolkit      │
│  (bootstrapped   │    │  (bootstrapped   │
│  com --trust)    │    │  com --trust)    │
└──────────────────┘    └──────────────────┘
```

A conta tools precisa ser capaz de **assumir roles** nas contas de deploy. Isso é configurado no bootstrap de cada conta de deploy via `--trust`.

---

### 2. Bootstrap cross-account — a sequência correta

O bootstrap padrão (`cdk bootstrap`) cria a CDKToolkit stack com roles que só a própria conta pode assumir. Para CDK Pipelines multi-account, você precisa modificar esse comportamento.

**Passo 1 — Bootstrap da conta tools (pipeline):**

```bash
# A conta tools não precisa de --trust especial
# mas precisa do bootstrap moderno
cdk bootstrap aws://PIPELINE_ACCOUNT_ID/us-east-1 \
  --profile pipeline
```

**Passo 2 — Bootstrap das contas de deploy com trust:**

```bash
# Conta de dev — confia na conta da pipeline para fazer deploys
cdk bootstrap aws://DEV_ACCOUNT_ID/us-east-1 \
  --profile dev \
  --trust PIPELINE_ACCOUNT_ID \
  --cloudformation-execution-policies arn:aws:iam::aws:policy/AdministratorAccess

# Conta de prod — mesma coisa
cdk bootstrap aws://PROD_ACCOUNT_ID/us-east-1 \
  --profile prod \
  --trust PIPELINE_ACCOUNT_ID \
  --cloudformation-execution-policies arn:aws:iam::aws:policy/AdministratorAccess
```

**O que `--trust` faz na prática:**

[FATO] O `--trust PIPELINE_ACCOUNT_ID` modifica o trust policy das IAM roles do CDKToolkit na conta de deploy para incluir a conta da pipeline como principal autorizado:

```json
// Role: cdk-XXXX-deploy-role-DEV_ACCOUNT-us-east-1
// Trust Policy gerada pelo --trust:
{
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::PIPELINE_ACCOUNT_ID:root"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Isso permite que o CodeBuild rodando na conta da pipeline faça `sts:AssumeRole` na conta de deploy e execute o CloudFormation lá.

**`--cloudformation-execution-policies` — o que é e por que importa:**

[FATO] Esta flag define as políticas IAM que a role de execução do CloudFormation (a que cria os recursos de fato) terá na conta de deploy. `AdministratorAccess` é conveniente mas permissivo — para produção, considere criar uma policy customizada que só permite os recursos que sua aplicação usa.

```bash
# Versão mais restrita para prod (exemplo)
cdk bootstrap aws://PROD_ACCOUNT_ID/us-east-1 \
  --profile prod \
  --trust PIPELINE_ACCOUNT_ID \
  --cloudformation-execution-policies arn:aws:iam::PROD_ACCOUNT_ID:policy/CdkDeployPolicy
```

**Adicionando trust a um bootstrap existente:**

[FATO] Ao re-executar `cdk bootstrap` com `--trust`, você deve listar **todos** os accounts que devem ter trust — não apenas os novos. Accounts não listados no novo comando perdem o trust.

```bash
# Re-bootstrap adicionando uma segunda conta de pipeline (ex: após migração)
cdk bootstrap aws://DEV_ACCOUNT_ID/us-east-1 \
  --profile dev \
  --trust PIPELINE_ACCOUNT_ID_1 \
  --trust PIPELINE_ACCOUNT_ID_2 \  # ← novo; PIPELINE_ACCOUNT_ID_1 deve ser repetido
  --cloudformation-execution-policies arn:aws:iam::aws:policy/AdministratorAccess
```

---

### 3. CodeStar Connections — GitHub sem PAT

[FATO] A forma legacy de integrar CodePipeline com GitHub usava um Personal Access Token (PAT) armazenado no Secrets Manager. A forma moderna usa **AWS CodeStar Connections**, que implementa OAuth via uma AWS App instalada no GitHub — sem nenhum token estático.

**Por que CodeStar Connections é preferível:**

```
PAT (legado)                        CodeStar Connections (moderno)
─────────────────────────────────   ────────────────────────────────
Token estático com validade          OAuth gerenciado pela AWS
Precisa rotacionar manualmente       Sem rotação necessária
Armazenado no Secrets Manager        Sem segredo a gerenciar
Permissões amplas por repositório    Permissões por instalação da App
Webhook manual ou polling            Webhook automático via App
```

**Criando a connection (só pode ser feito via Console ou CLI — não via CDK):**

```bash
# Etapa 1: criar a connection (fica em PENDING até ser autorizada)
aws codestar-connections create-connection \
  --provider-type GitHub \
  --connection-name minha-org-github \
  --profile pipeline

# Retorna: { "ConnectionArn": "arn:aws:codestar-connections:us-east-1:..." }

# Etapa 2: autorizar no console
# Console → Developer Tools → Connections → [sua connection] → Update pending connection
# → Conecta com sua conta GitHub → autoriza a AWS App
# Status muda de PENDING → AVAILABLE
```

[FATO] A autorização da connection **obrigatoriamente** passa pelo Console — não existe forma de fazer isso via CLI ou CDK. Isso é intencional: a AWS exige interação humana para autorizar acesso OAuth ao GitHub.

**Verificar que a connection está disponível:**

```bash
aws codestar-connections get-connection \
  --connection-arn arn:aws:codestar-connections:us-east-1:ACCOUNT:connection/UUID \
  --profile pipeline \
  --query 'Connection.ConnectionStatus'
# "AVAILABLE" ← pronto para uso
# "PENDING"   ← ainda precisa ser autorizada no console
```

---

### 4. Estrutura da CDK Pipeline — os stages automáticos

Uma CDK Pipeline mínima tem 3 stages gerados automaticamente antes de qualquer stage de aplicação:

```
GitHub                     CodePipeline (conta tools)
  │                              │
  │ push para main               │
  ├──────────────────────────────►
  │                         ┌────▼──────────┐
  │                         │  Source       │  Baixa o código do GitHub
  │                         │               │  via CodeStar Connection
  │                         └────┬──────────┘
  │                              │
  │                         ┌────▼──────────┐
  │                         │  Build        │  npm ci + cdk synth
  │                         │  (Synth)      │  Gera o cloud assembly
  │                         └────┬──────────┘
  │                              │
  │                         ┌────▼──────────┐
  │                         │ UpdatePipeline│  Aplica mudanças na
  │                         │ (Self-Mutate) │  própria pipeline
  │                         └────┬──────────┘
  │                              │
  │                    ┌─────────▼──────────┐
  │                    │  Assets            │  Upload S3/ECR
  │                    └─────────┬──────────┘
  │                              │
  │                    ┌─────────▼──────────┐
  │                    │  Stage: Dev        │  Deploy nas contas
  │                    │  Stage: Prod       │  de aplicação
  │                    └────────────────────┘
```

**O que cada stage automático faz:**

**Source:**
- Baixa o código do repositório GitHub via CodeStar Connection
- Armazena o artefato no S3 (artifact bucket da pipeline)
- Triggado por push no branch configurado

**Build (Synth):**
- Roda `npm ci` (ou equivalente) para instalar dependências
- Executa `cdk synth` para gerar o cloud assembly (`cdk.out/`)
- O cloud assembly é o artefato que alimenta todos os stages seguintes

**UpdatePipeline (Self-Mutate):**
- Compara a definição da pipeline no cloud assembly com a pipeline atual
- Se a pipeline mudou: aplica as mudanças e **reinicia a execução do zero**
- Se não mudou: avança para os stages de aplicação

---

### 5. Self-mutation — o mecanismo que muda tudo

Self-mutation é a característica mais importante e mais confusa do CDK Pipelines.

[FATO] Quando você muda o código da pipeline (adiciona um stage, muda um ShellStep, etc.) e faz push, o ciclo é:

```
Push para main
  │
  ▼
Source → Build → UpdatePipeline
                      │
                  Pipeline mudou?
                  │           │
                 Sim          Não
                  │           │
                  ▼           ▼
           Pipeline        Continua
           se atualiza      para os
           e REINICIA       stages de
           do zero          aplicação
                  │
                  ▼
           Source → Build → UpdatePipeline
                                 │
                             Agora igual
                                 │
                                 ▼
                            Assets → Dev → Prod
```

**Implicações práticas do self-mutation:**

1. **A primeira execução nunca deploya a aplicação** — ela apenas atualiza a pipeline (que acabou de ser criada) e reinicia. A segunda execução é que deploya.

2. **Mudanças na pipeline são atômicas** — você não pode ter "a pipeline em produção está na versão antiga enquanto eu testo a versão nova". Toda mudança passa pela pipeline antes de chegar à aplicação.

3. **Debugging fica mais lento** — para testar uma mudança na definição da pipeline, você precisa fazer push, esperar a pipeline rodar, ver o resultado. Não tem como testar localmente.

4. **`selfMutation: false` para desenvolvimento** — durante o desenvolvimento inicial da pipeline, você pode desabilitar o self-mutation para iterar mais rápido. Mas nunca em produção.

---

### 6. Criando a pipeline — código completo

```typescript
// lib/pipeline-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as pipelines from 'aws-cdk-lib/pipelines';
import { Construct } from 'constructs';
import { ApplicationStage } from './application-stage';

export class PipelineStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props: cdk.StackProps) {
    super(scope, id, props);

    // 1. Fonte: GitHub via CodeStar Connection
    const source = pipelines.CodePipelineSource.connection(
      'minha-org/meu-repo',    // owner/repo no GitHub
      'main',                   // branch
      {
        connectionArn: 'arn:aws:codestar-connections:us-east-1:PIPELINE_ACCOUNT:connection/UUID',
        triggerOnPush: true,    // padrão: true
      }
    );

    // 2. Synth step: instala deps e executa cdk synth
    const synth = new pipelines.ShellStep('Synth', {
      input: source,
      commands: [
        'npm ci',               // instala dependências (lockfile)
        'npm run build',        // compila TypeScript (se necessário)
        'npx cdk synth',        // gera o cloud assembly
      ],
      // primaryOutputDirectory: 'cdk.out',  // padrão quando cdk synth é o último comando
    });

    // 3. Definição da pipeline
    const pipeline = new pipelines.CodePipeline(this, 'Pipeline', {
      pipelineName: 'MeuAppPipeline',
      synth,

      // CodeBuild image para o synth (padrão: aws/codebuild/standard:7.0)
      codeBuildDefaults: {
        buildEnvironment: {
          buildImage: codebuild.LinuxBuildImage.STANDARD_7_0,
        },
      },

      // Self-mutation habilitado (padrão: true)
      selfMutation: true,

      // Publicar assets em paralelo (mais rápido para múltiplos assets)
      publishAssetsInParallel: true,

      // Docker necessário? (se algum asset usa Docker)
      dockerEnabledForSynth: false,
      dockerEnabledForSelfMutation: false,
    });

    // 4. Adicionar stages de aplicação
    pipeline.addStage(new ApplicationStage(this, 'Dev', {
      env: { account: 'DEV_ACCOUNT_ID', region: 'us-east-1' },
    }));

    // Prod com aprovação manual antes do deploy
    pipeline.addStage(new ApplicationStage(this, 'Prod', {
      env: { account: 'PROD_ACCOUNT_ID', region: 'us-east-1' },
    }), {
      pre: [new pipelines.ManualApprovalStep('ApproveProductionDeploy')],
    });
  }
}

// bin/app.ts
const app = new cdk.App();

// A pipeline stack roda na conta tools
new PipelineStack(app, 'PipelineStack', {
  env: { account: 'PIPELINE_ACCOUNT_ID', region: 'us-east-1' },
});
```

**Primeiro deploy — o bootstrap da pipeline:**

```bash
# 1. Bootstrap de todas as contas (uma vez)
cdk bootstrap aws://PIPELINE_ACCOUNT_ID/us-east-1 --profile pipeline

cdk bootstrap aws://DEV_ACCOUNT_ID/us-east-1 \
  --profile dev \
  --trust PIPELINE_ACCOUNT_ID \
  --cloudformation-execution-policies arn:aws:iam::aws:policy/AdministratorAccess

cdk bootstrap aws://PROD_ACCOUNT_ID/us-east-1 \
  --profile prod \
  --trust PIPELINE_ACCOUNT_ID \
  --cloudformation-execution-policies arn:aws:iam::aws:policy/AdministratorAccess

# 2. Deploy inicial da pipeline stack (só precisa rodar uma vez)
cdk deploy PipelineStack --profile pipeline

# A partir daqui, todo push para main atualiza e executa a pipeline automaticamente
```

---

### 7. O que acontece antes do primeiro deploy de aplicação

Sequência detalhada na primeira execução após o `cdk deploy PipelineStack`:

```
Execução #1 (após cdk deploy PipelineStack):

  Source:          Baixa o código do GitHub
  Build:           npm ci + cdk synth → gera cloud assembly
  UpdatePipeline:  Compara pipeline atual vs cloud assembly
                   → A pipeline foi criada pelo cdk deploy manual
                   → Cloud assembly tem a mesma definição
                   → Sem mudanças → avança

  Assets:          Faz upload dos assets para S3/ECR
  Dev:             Deploya ApplicationStage na conta dev ✅
  Approve:         Aguarda aprovação manual
  Prod:            Deploya ApplicationStage na conta prod ✅

───────────────────────────────────────────────────────────────

Depois, você faz uma mudança na pipeline (ex: adiciona um stage):

  push para main

Execução #2:

  Source:          Baixa o novo código
  Build:           cdk synth com o novo stage
  UpdatePipeline:  Pipeline mudou → aplica mudança → REINICIA

Execução #3 (após restart):

  Source → Build → UpdatePipeline  (sem mudança agora)
  Assets → Dev → Approve → Prod
```

---

## Exemplo prático — verificando a pipeline após deploy

```bash
# Ver o status da pipeline
aws codepipeline get-pipeline-state \
  --name MeuAppPipeline \
  --profile pipeline \
  --query 'stageStates[*].{Stage:stageName,Status:latestExecution.status}' \
  --output table

# Ver os logs do Synth step
aws codebuild batch-get-builds \
  --ids $(aws codepipeline get-pipeline-state \
    --name MeuAppPipeline \
    --query 'stageStates[?stageName==`Build`].actionStates[0].latestExecution.externalExecutionId' \
    --output text) \
  --query 'builds[0].logs.deepLink'

# Forçar nova execução manualmente
aws codepipeline start-pipeline-execution \
  --name MeuAppPipeline \
  --profile pipeline

# Ver a definição atual da pipeline
aws codepipeline get-pipeline \
  --name MeuAppPipeline \
  --profile pipeline \
  | jq '.pipeline.stages[].name'
```

---

## Armadilhas comuns

**1. Connection em status PENDING — pipeline trava no Source**

Se você criar a connection via CLI mas esquecer de autorizá-la no Console, a pipeline vai travar no stage Source com erro `Connection is not authorized`. Verifique sempre o status da connection antes de criar a pipeline.

**2. Bootstrap sem `--trust` — deploy falha com AccessDenied**

Se você esquecer o `--trust PIPELINE_ACCOUNT_ID` no bootstrap das contas de deploy, a pipeline vai rodar com sucesso até o stage de deploy e falhar com `AccessDenied` ao tentar assumir a role na conta de destino. O erro não é óbvio — ele aparece nos logs do CodeBuild como falha de `sts:AssumeRole`.

**3. Self-mutation e desenvolvimento da pipeline — loop confuso**

Durante o desenvolvimento inicial, cada mudança na pipeline exige um push, espera pelo Source + Build + UpdatePipeline, e reinício. Se a pipeline está quebrando no Synth, você fica num loop: push → falha → fix → push → falha. Para sair mais rápido: teste o `cdk synth` localmente antes de fazer push, e use `selfMutation: false` temporariamente enquanto está construindo a pipeline pela primeira vez.

**4. `dockerEnabledForSynth: false` com assets Docker**

Se algum dos seus Lambda assets usa `DockerImageFunction` ou `PythonFunction` com bundling em Docker, o Synth step precisa de Docker. Sem `dockerEnabledForSynth: true`, o synth vai falhar com erro de Docker não disponível. Habilitar Docker no CodeBuild aumenta o tempo de startup do build.

---

## Exercício de reflexão

Você tem uma CDK Pipeline rodando na conta `tools` que deploya em `dev` e `prod`. A pipeline está funcionando bem. Então você precisa adicionar um terceiro ambiente `staging` (nova conta) entre dev e prod.

Descreva a sequência exata de passos necessários — incluindo quais comandos rodar em qual conta, em qual ordem, e o que acontece com a pipeline durante essa transição. Em particular: quando a pipeline precisa ser pausada? Existe risco de downtime em dev ou prod durante a adição do novo stage? O que acontece se você adicionar o stage no código e fazer push sem ter feito o bootstrap da conta staging antes?

---

## Recursos para aprofundar

**CDK Pipelines:**
- [Continuous integration and delivery (CI/CD) using CDK Pipelines](https://docs.aws.amazon.com/cdk/v2/guide/cdk_pipeline.html) — guia completo com exemplos de multi-account, self-mutation e ShellSteps.

**Bootstrap cross-account:**
- [Bootstrap your environment for use with the AWS CDK](https://docs.aws.amazon.com/cdk/v2/guide/bootstrapping-env.html) — opções `--trust` e `--cloudformation-execution-policies` com exemplos de comandos.

**CodeStar Connections:**
- [GitHub connections](https://docs.aws.amazon.com/codepipeline/latest/userguide/connections-github.html) — processo de criação e autorização da connection no Console, incluindo os requisitos de permissão IAM para criar connections.

---

