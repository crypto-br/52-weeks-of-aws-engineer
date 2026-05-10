# Sessão 010 — CDK Pipelines: Stages customizados, ShellSteps e self-mutation em ação

**Duração estimada:** 60 minutos
**Pré-requisitos:** session-009 — CDK Pipelines: bootstrap cross-account, OIDC connection e estrutura da pipeline

---

## Objetivo

Ao final, você conseguirá adicionar um Stage com múltiplos stacks em sequência e em paralelo, inserir ShellSteps de validação entre stages (ex: smoke tests pós-deploy), observar o ciclo de self-mutation (a pipeline re-executa a si mesma quando o código da pipeline muda antes de qualquer deploy de aplicação), e debugar um deploy que falha na fase de publicação de assets.

---

## Contexto

[FATO] A sessão anterior cobriu a estrutura mínima da pipeline (Source → Build → UpdatePipeline). Esta sessão aprofunda os blocos que vêm depois: como organizar stages, como adicionar validações entre eles, e como usar outputs de stacks deployados em steps subsequentes.

[CONSENSO] O padrão recomendado para pipelines de produção é: deploy em dev → smoke tests automáticos → aprovação manual → deploy em prod → smoke tests automáticos. O CDK Pipelines tem suporte nativo a cada um desses passos via `pre`, `post`, `ShellStep` e `ManualApprovalStep`.

---

## Conceitos principais

### 1. Stages sequenciais e em paralelo — addStage e Wave

Por padrão, stages adicionados com `addStage` são executados **sequencialmente**:

```typescript
// Sequencial: dev termina antes de staging começar
pipeline.addStage(new AppStage(this, 'Dev',     { env: devEnv }));
pipeline.addStage(new AppStage(this, 'Staging', { env: stagingEnv }));
pipeline.addStage(new AppStage(this, 'Prod',    { env: prodEnv }));
```

Para stages em **paralelo**, use `Wave`:

```typescript
// Wave: eu-west-1 e ap-southeast-1 deployam ao mesmo tempo
const multiRegionWave = pipeline.addWave('MultiRegionDeploy');

multiRegionWave.addStage(new AppStage(this, 'ProdEU', {
  env: { account: PROD_ACCOUNT, region: 'eu-west-1' },
}));

multiRegionWave.addStage(new AppStage(this, 'ProdAP', {
  env: { account: PROD_ACCOUNT, region: 'ap-southeast-1' },
}));

// Depois da wave, um stage sequencial aguarda ambas terminarem
pipeline.addStage(new MonitoringStage(this, 'GlobalMonitoring', {
  env: { account: TOOLS_ACCOUNT, region: 'us-east-1' },
}));
```

**Diagrama de execução:**

```
Sequential:                         Wave (parallel):
  Dev  ──► Staging ──► Prod           ┌── ProdEU ──┐
  (uma de cada vez)                   │             ├──► GlobalMonitoring
                                      └── ProdAP ──┘
                                      (simultâneos)
```

**Ordem de stacks dentro de um Stage:**

[FATO] Dentro de um Stage com múltiplos stacks, o CDK determina a ordem automaticamente com base nas dependências entre os stacks. Stacks independentes são deployados em paralelo; stacks com dependências são ordenados corretamente.

```typescript
export class AppStage extends cdk.Stage {
  constructor(scope: Construct, id: string, props: cdk.StageProps) {
    super(scope, id, props);

    const network = new NetworkStack(this, 'Network');
    const data    = new DataStack(this, 'Data', { vpc: network.vpc });
    const app     = new AppStack(this, 'App',  { vpc: network.vpc, table: data.table });
    // CDK infere: Network → (Data e App em paralelo, depois App espera Data)
    // Network deploya primeiro; Data e App deployam depois em paralelo se não há dep entre eles
  }
}
```

Para forçar dependência explícita entre stacks no mesmo Stage:

```typescript
app.addDependency(data);   // garante que Data termina antes de App começar
```

---

### 2. ShellStep — validações entre stages

`ShellStep` é o bloco fundamental para inserir comandos shell em qualquer ponto da pipeline. Pode ser usado como `pre` (antes do deploy do stage) ou `post` (depois do deploy).

```typescript
import * as pipelines from 'aws-cdk-lib/pipelines';

// Pre-step: roda antes do deploy do stage
pipeline.addStage(new AppStage(this, 'Dev', { env: devEnv }), {
  pre: [
    new pipelines.ShellStep('LintAndTest', {
      commands: [
        'npm ci',
        'npm run lint',
        'npm test',
      ],
    }),
  ],
  post: [
    new pipelines.ShellStep('SmokeTest', {
      commands: [
        // Acessa a URL da aplicação e verifica o health endpoint
        'curl -f $APP_URL/health || exit 1',
        'echo "Smoke test passed"',
      ],
      envFromCfnOutputs: {
        APP_URL: appStageRef.appUrlOutput,  // output da stack (ver seção 4)
      },
    }),
  ],
});
```

**Casos de uso comuns para ShellStep:**

```
PRE-DEPLOY:
  ✅ Lint e testes unitários antes de deployar
  ✅ Validação de segurança (checkov, cfn-nag, cfn-guard)
  ✅ Geração de documentação ou artefatos
  ✅ ManualApprovalStep (tipo especial de pre-step)

POST-DEPLOY:
  ✅ Smoke tests (HTTP health checks, ping endpoints)
  ✅ Integration tests contra o ambiente recém-deployado
  ✅ Notificações (Slack, PagerDuty) de deploy bem-sucedido
  ✅ Invalidação de cache CDN
  ✅ Atualização de registro de deploy (JIRA, Backstage)
```

---

### 3. CodeBuildStep — ShellStep com mais controle

`CodeBuildStep` é uma versão mais poderosa do `ShellStep` que dá acesso a configurações do CodeBuild: imagem customizada, políticas IAM adicionais, variáveis de ambiente e timeout.

```typescript
import * as codebuild from 'aws-cdk-lib/aws-codebuild';

const integrationTest = new pipelines.CodeBuildStep('IntegrationTest', {
  commands: [
    'npm ci',
    'npm run test:integration',
  ],

  // Variáveis de ambiente estáticas
  env: {
    ENVIRONMENT: 'dev',
    LOG_LEVEL: 'debug',
  },

  // Imagem do CodeBuild customizada
  buildEnvironment: {
    buildImage: codebuild.LinuxBuildImage.STANDARD_7_0,
    computeType: codebuild.ComputeType.MEDIUM,
    privileged: false,
  },

  // Políticas IAM adicionais para o CodeBuild project
  rolePolicyStatements: [
    new iam.PolicyStatement({
      effect: iam.Effect.ALLOW,
      actions: ['ssm:GetParameter'],
      resources: [`arn:aws:ssm:${this.region}:${this.account}:parameter/test/*`],
    }),
    new iam.PolicyStatement({
      effect: iam.Effect.ALLOW,
      actions: ['secretsmanager:GetSecretValue'],
      resources: [`arn:aws:secretsmanager:${this.region}:${this.account}:secret:test-*`],
    }),
  ],

  // Timeout do CodeBuild project
  timeout: cdk.Duration.minutes(30),

  // Variáveis vindas de outputs de stacks deployados (ver seção 4)
  envFromCfnOutputs: {
    API_ENDPOINT: apiStack.endpointOutput,
  },
});

pipeline.addStage(new AppStage(this, 'Dev', { env: devEnv }), {
  post: [integrationTest],
});
```

---

### 4. Outputs de stacks deployados em ShellSteps

Um padrão muito comum é: deploy o stack → obter o endpoint/URL do que foi criado → usá-lo no smoke test. O CDK Pipelines tem suporte nativo via `envFromCfnOutputs`.

**Passo 1 — expor o output no Stage:**

```typescript
// lib/app-stage.ts
export class AppStage extends cdk.Stage {
  // Expor os outputs que serão usados em steps
  public readonly apiEndpointOutput: CfnOutput;
  public readonly loadBalancerDnsOutput: CfnOutput;

  constructor(scope: Construct, id: string, props: cdk.StageProps) {
    super(scope, id, props);

    const appStack = new AppStack(this, 'App');

    // Os outputs são CfnOutput do stack
    this.apiEndpointOutput = appStack.apiEndpoint;       // CfnOutput definido em AppStack
    this.loadBalancerDnsOutput = appStack.lbDnsName;     // CfnOutput definido em AppStack
  }
}

// lib/app-stack.ts
export class AppStack extends cdk.Stack {
  public readonly apiEndpoint: CfnOutput;
  public readonly lbDnsName: CfnOutput;

  constructor(scope: Construct, id: string, props: cdk.StackProps) {
    super(scope, id, props);

    const api = new apigw.RestApi(this, 'Api');

    this.apiEndpoint = new CfnOutput(this, 'ApiEndpoint', {
      value: api.url,
    });

    const lb = new elbv2.ApplicationLoadBalancer(this, 'LB', { vpc, internetFacing: true });

    this.lbDnsName = new CfnOutput(this, 'LbDnsName', {
      value: lb.loadBalancerDnsName,
    });
  }
}
```

**Passo 2 — usar os outputs no pipeline:**

```typescript
// lib/pipeline-stack.ts
const devStage = new AppStage(this, 'Dev', { env: devEnv });

pipeline.addStage(devStage, {
  post: [
    new pipelines.ShellStep('SmokeTests', {
      commands: [
        // API_ENDPOINT e LB_DNS são populados automaticamente
        // com os valores dos CfnOutputs após o deploy
        'curl -f https://$API_ENDPOINT/health',
        'curl -f http://$LB_DNS/ping',
        'echo "All smoke tests passed"',
      ],
      envFromCfnOutputs: {
        API_ENDPOINT: devStage.apiEndpointOutput,
        LB_DNS:       devStage.loadBalancerDnsOutput,
      },
    }),
  ],
});
```

**O que o CDK faz por baixo:**

[FATO] O `envFromCfnOutputs` cria uma dependência implícita: o ShellStep só começa após o stage inteiro estar deployado, e o CodeBuild project recebe as variáveis via CodePipeline (que lê os outputs do CloudFormation stack deployado).

---

### 5. Self-mutation em ação — observando o ciclo

Esta seção documenta o comportamento do self-mutation para que você reconheça o que está acontecendo quando a pipeline reinicia.

**Cenário: você adiciona um novo Stage à pipeline**

```
Estado antes: pipeline tem Dev e Prod
Você adiciona Staging entre Dev e Prod e faz push
```

```
Execução #N (com o novo código):

  Source:          ✅ Baixa código com o novo stage Staging
  Build:           ✅ cdk synth → cloud assembly com Dev + Staging + Prod
  UpdatePipeline:  ⚠️  Pipeline atual: Dev → Prod
                       Cloud assembly: Dev → Staging → Prod
                       DIFERENTE → aplica mudança → REINICIA

Execução #N+1 (mesma branch, pipeline atualizada):

  Source:          ✅ Mesmo código
  Build:           ✅ Mesmo cloud assembly
  UpdatePipeline:  ✅ Pipeline atual = cloud assembly → sem mudança → AVANÇA

  Assets:          Upload de assets
  Dev:             Deploy na conta dev
  SmokeTests:      Testes pós-deploy dev
  Staging:         Deploy na conta staging  ← novo stage funcionando
  Approve:         Aprovação manual
  Prod:            Deploy na conta prod
```

**Como observar o restart no Console:**

```
CodePipeline → MeuAppPipeline → Executions
  Execution #N:   Status: Superseded  ← foi substituída pelo restart
  Execution #N+1: Status: Succeeded   ← a que realmente executou tudo
```

[FATO] Quando a pipeline reinicia devido ao self-mutation, a execução anterior fica com status `Superseded` (não `Failed`). Se você ver `Superseded`, a pipeline funcionou corretamente — não é um erro.

**Forçando um restart manual:**

```bash
# Equivalente ao botão "Release Change" no console
aws codepipeline start-pipeline-execution \
  --name MeuAppPipeline \
  --profile pipeline
```

---

### 6. Debugando falhas na publicação de assets

A fase de publicação de assets (`Assets` stage) é onde o CDK faz upload do código Lambda e imagens Docker. Falhas aqui têm causas específicas.

**Estrutura do stage Assets:**

```
Assets
  ├── Publish-Asset-HASH_A (Lambda zip)
  ├── Publish-Asset-HASH_B (outro Lambda zip)
  └── Publish-Asset-HASH_C (imagem Docker)
```

[FATO] Cada asset tem seu próprio CodeBuild action paralelo. Uma falha em um não cancela os outros imediatamente — eles podem falhar em paralelo.

**Erros comuns e diagnóstico:**

**Erro 1: `AccessDenied` ao fazer upload para S3**

```
Error: AccessDenied: Access Denied (Service: S3, Status Code: 403)

Causa: O CodeBuild do asset publishing não tem permissão para escrever no bucket
       do bootstrap da conta de destino.

Diagnóstico:
  1. Verifique se a conta de destino foi bootstrapada com --trust
  2. Verifique se a role cdk-XXXX-file-publishing-role existe na conta de destino
  3. Verifique se a trust policy da role inclui a conta da pipeline

Fix:
  cdk bootstrap aws://DEST_ACCOUNT/REGION \
    --profile dest \
    --trust PIPELINE_ACCOUNT \
    --cloudformation-execution-policies arn:aws:iam::aws:policy/AdministratorAccess
```

**Erro 2: Docker not available para image assets**

```
Error: Cannot connect to the Docker daemon at unix:///var/run/docker.sock

Causa: O CodeBuild project do asset publishing não tem Docker habilitado.

Fix no CDK:
  const pipeline = new pipelines.CodePipeline(this, 'Pipeline', {
    synth,
    assetPublishingCodeBuildDefaults: {
      buildEnvironment: {
        privileged: true,  // habilita Docker no CodeBuild
      },
    },
  });

⚠️  Se a pipeline já está deployada:
  1. Sete privileged: true
  2. Faça push e aguarde o self-mutation aplicar a mudança
  3. Só então adicione o asset Docker
  Motivo: mudar privileged numa pipeline existente exige recrear o CodeBuild project
          via self-mutation antes de usar Docker
```

**Erro 3: `Cannot find module` no Lambda após deploy**

```
Error: Runtime.ImportModuleError: Cannot find module 'sharp'

Causa: Módulo com binário nativo foi bundled pelo esbuild para a arquitetura errada
       (ver sessão 007 — externalModules vs nodeModules).

Diagnóstico:
  1. Verifique se 'sharp' está em externalModules (errado) ou nodeModules (correto)
  2. Verifique se a arquitetura do NodejsFunction bate com o módulo compilado

Fix:
  bundling: {
    nodeModules: ['sharp'],   // compila dentro do Docker para Amazon Linux
  }
```

**Inspecionando logs de um asset publishing que falhou:**

```bash
# Encontrar o build ID do CodeBuild que falhou
aws codepipeline get-pipeline-state \
  --name MeuAppPipeline \
  --profile pipeline \
  --query 'stageStates[?stageName==`Assets`].actionStates[?currentRevision!=null].latestExecution.externalExecutionId' \
  --output text

# Ver os logs do CodeBuild
aws codebuild batch-get-builds \
  --ids BUILD_ID \
  --profile pipeline \
  --query 'builds[0].logs.deepLink'
# Abre o CloudWatch Logs com os logs completos do build
```

---

### 7. Pipeline completa com todos os recursos

```typescript
// lib/pipeline-stack.ts
export class PipelineStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props: cdk.StackProps) {
    super(scope, id, props);

    const source = pipelines.CodePipelineSource.connection(
      'minha-org/meu-repo', 'main',
      { connectionArn: CONNECTION_ARN }
    );

    const synth = new pipelines.ShellStep('Synth', {
      input: source,
      commands: ['npm ci', 'npm run build', 'npx cdk synth'],
    });

    const pipeline = new pipelines.CodePipeline(this, 'Pipeline', {
      pipelineName: 'MeuAppPipeline',
      synth,
      selfMutation: true,
      publishAssetsInParallel: true,
      codeBuildDefaults: {
        buildEnvironment: {
          buildImage: codebuild.LinuxBuildImage.STANDARD_7_0,
        },
      },
    });

    // ── Stage Dev ─────────────────────────────────────────────────────
    const devStage = new AppStage(this, 'Dev', { env: DEV_ENV });

    pipeline.addStage(devStage, {
      pre: [
        new pipelines.ShellStep('UnitTests', {
          commands: ['npm ci', 'npm test'],
        }),
      ],
      post: [
        new pipelines.CodeBuildStep('IntegrationTests', {
          commands: [
            'npm ci',
            'npm run test:integration',
          ],
          envFromCfnOutputs: {
            API_URL: devStage.apiEndpointOutput,
          },
          rolePolicyStatements: [
            new iam.PolicyStatement({
              actions: ['execute-api:Invoke'],
              resources: ['*'],
            }),
          ],
        }),
      ],
    });

    // ── Wave: Staging multi-região (paralelo) ─────────────────────────
    const stagingWave = pipeline.addWave('Staging', {
      pre: [new pipelines.ManualApprovalStep('ApproveStaging')],
    });

    stagingWave.addStage(new AppStage(this, 'StagingUS', {
      env: { account: STAGING_ACCOUNT, region: 'us-east-1' },
    }));

    stagingWave.addStage(new AppStage(this, 'StagingEU', {
      env: { account: STAGING_ACCOUNT, region: 'eu-west-1' },
    }));

    // ── Stage Prod ────────────────────────────────────────────────────
    const prodStage = new AppStage(this, 'Prod', { env: PROD_ENV });

    pipeline.addStage(prodStage, {
      pre: [
        new pipelines.ManualApprovalStep('ApproveProd'),
        // Gating automático: bloqueia se IAM permissions aumentaram
        new pipelines.ConfirmPermissionsBroadening('CheckPermissions', {
          stage: prodStage,
        }),
      ],
      post: [
        new pipelines.ShellStep('ProdSmokeTests', {
          commands: ['curl -f $API_URL/health || exit 1'],
          envFromCfnOutputs: {
            API_URL: prodStage.apiEndpointOutput,
          },
        }),
      ],
    });
  }
}
```

---

## Armadilhas comuns

**1. `envFromCfnOutputs` com output de stack errado**

Se você passa um `CfnOutput` de um stack que não foi deployado **neste stage**, a pipeline falha com erro de referência inválida. O `envFromCfnOutputs` só aceita outputs de stacks dentro do stage que precede o step. Para usar um output de outro stage, você precisa passá-lo via SSM Parameter Store ou Secrets Manager.

**2. Wave com stages que têm dependências cruzadas**

Stages dentro de uma Wave são deployados em paralelo — o que pressupõe que são independentes. Se o Stage A depende de um output do Stage B (ambos na mesma Wave), isso cria uma dependência circular que o CDK detecta com erro. Mova os estágios dependentes para fora da Wave.

**3. Modificar `privileged` numa pipeline existente sem fazer self-mutation primeiro**

Se você habilitar Docker assets num projeto que já tem uma pipeline deployada, mas não esperar o self-mutation aplicar o `privileged: true` antes de adicionar o asset Docker, o CodeBuild project ainda terá `privileged: false` na execução em que o asset aparece pela primeira vez. A falha vai acontecer nessa execução. A solução: faça dois commits — primeiro um com apenas `privileged: true`, espere a pipeline se mutar; depois um segundo commit com o asset Docker.

---

## Exercício de reflexão

Você tem uma pipeline com dev → staging → prod. Os smoke tests do stage `dev` estão falhando intermitentemente: 70% das vezes passam, 30% falham com `curl: (7) Failed to connect`. Você suspeita que os testes começam antes da aplicação estar completamente pronta após o deploy.

Quais são as estratégias possíveis para resolver isso? Considere pelo menos três abordagens diferentes (desde ajustes no ShellStep até mudanças na arquitetura do health check). Para cada uma, descreva o trade-off entre complexidade de implementação, confiabilidade e impacto no tempo total da pipeline. Qual você implementaria primeiro e por quê?

---

## Recursos para aprofundar

**Pipelines readme completo:**
- [aws-cdk-lib.pipelines module](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.pipelines-readme.html) — referência completa de ShellStep, CodeBuildStep, Wave, envFromCfnOutputs, ConfirmPermissionsBroadening e todas as opções de configuração do CodePipeline construct.

**CDK Pipelines guide:**
- [Continuous integration and delivery (CI/CD) using CDK Pipelines](https://docs.aws.amazon.com/cdk/v2/guide/cdk_pipeline.html) — seção de troubleshooting cobre os erros mais comuns de asset publishing e self-mutation.

**Blog post original:**
- [CDK Pipelines: Continuous Delivery for AWS CDK Applications](https://aws.amazon.com/blogs/developer/cdk-pipelines-continuous-delivery-for-aws-cdk-applications/) — walk-through completo com o raciocínio por trás do design do self-mutation e a progressão Source → Build → UpdatePipeline.

---

