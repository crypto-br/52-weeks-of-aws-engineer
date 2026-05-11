# Sessão session-011-cdk-custom-resources-aspects — CDK: CustomResources e Aspects

**Duração estimada:** 60 minutos
**Pré-requisitos:** session-010-cdk-pipelines-stages-shellsteps

---

## Objetivo

Ao final, você conseguirá criar um CustomResource que invoca uma Lambda para provisionar recursos não suportados nativamente pelo CloudFormation, e aplicar um Aspect para varrer todos os constructs de uma stack (ex: forçar encriptação em todos os buckets S3 automaticamente).

---

## Contexto

[FATO] CloudFormation conhece apenas os tipos de recurso que a AWS publicou — aproximadamente 1.200 tipos em maio de 2026. Todo recurso fora desse catálogo (configurações em APIs de terceiros, operações de inicialização de banco de dados, registros de DNS em provedores externos, bootstrapping de dados) precisa de um mecanismo de extensão. Esse mecanismo existe desde 2012 e se chama **Custom Resource**: um bloco `Type: Custom::*` no template que delega o ciclo de vida (Create/Update/Delete) a uma Lambda ou um tópico SNS.

[CONSENSO] O CDK oferece duas abstrações sobre Custom Resources: o `custom_resources.Provider` (um mini-framework com suporte a operações assíncronas, retry e timeout) e o `AwsCustomResource` (um wrapper que executa uma única chamada de SDK sem você precisar escrever código de Lambda). A escolha entre os dois segue uma heurística simples: se a lógica cabe em uma chamada de API, use `AwsCustomResource`; se precisa de lógica de negócio, polling ou múltiplas chamadas, use `Provider`.

[FATO] Aspects são um mecanismo de travessia da árvore de constructs introduzido no CDK v1 e mantido no v2. Eles implementam o padrão Visitor: você escreve uma função `visit(node)` e o CDK a invoca para cada nó da árvore durante a síntese, depois que todos os constructs foram instanciados. Isso permite inspecionar ou modificar a árvore de forma transversal — sem que cada construct precise saber sobre a regra que está sendo aplicada.

---

## Conceitos principais

### 1. CloudFormation Custom Resource: o protocolo base

Antes de ver a abstração CDK, entenda o que está por baixo. Quando CloudFormation precisa criar um Custom Resource, ele faz um HTTP POST para uma URL pré-signed S3 (ou invoca uma Lambda diretamente) com um payload JSON como este:

```json
{
  "RequestType": "Create",
  "ResponseURL": "https://s3.amazonaws.com/pre-signed-url...",
  "StackId": "arn:aws:cloudformation:...",
  "RequestId": "abc123",
  "ResourceType": "Custom::AcmeCertRegistration",
  "LogicalResourceId": "MyCert",
  "ResourceProperties": {
    "Domain": "app.example.com",
    "ServiceToken": "arn:aws:lambda:..."
  }
}
```

O handler (Lambda ou SNS) deve responder com um objeto JSON para a `ResponseURL` contendo `Status` (SUCCESS ou FAILED), `PhysicalResourceId`, e opcionalmente `Data` (atributos que ficam disponíveis via `Fn::GetAtt`).

[FATO] O **Physical Resource ID** é o campo mais crítico do protocolo. Ele identifica unicamente a instância do recurso externo. As regras são:

```
CREATE:  você gera o PhysicalResourceId (ex: "acme-cert-app.example.com")
UPDATE:  você retorna o mesmo ID se a operação é in-place,
         ou um ID diferente se é uma substituição.
         → Se o ID mudar, o CloudFormation envia um evento DELETE
           para o ID antigo logo em seguida!
DELETE:  CloudFormation envia o ID que foi armazenado no Create/Update.
         Você o usa para destruir o recurso externo.
```

[FATO] Se a Lambda falhar (exception não capturada) ou não responder em 60 minutos (timeout padrão de Custom Resource), o CloudFormation fica preso aguardando e eventualmente faz rollback após 1 hora. Por isso o CDK Provider existe: ele elimina a necessidade de escrever o código de resposta HTTP manualmente e gerencia o timeout.

---

### 2. custom_resources.Provider — o mini-framework CDK

[FATO] O `Provider` é um construct que cria toda a infraestrutura necessária para gerenciar um Custom Resource de forma robusta:

```
┌─────────────────────────────────────────────────────────────┐
│  Provider (construct)                                       │
│                                                             │
│  ┌──────────────┐    ┌──────────────────────────────────┐  │
│  │  onEvent     │    │  (se isComplete definido)        │  │
│  │  Lambda      │    │  ┌─────────────────────────────┐ │  │
│  │              │───▶│  │ Step Functions state machine│ │  │
│  │  CREATE/     │    │  │  ┌──────────┐  ┌─────────┐  │ │  │
│  │  UPDATE/     │    │  │  │isComplete│  │ waiter  │  │ │  │
│  │  DELETE      │    │  │  │  Lambda  │◀─│  loop   │  │ │  │
│  └──────────────┘    │  │  └──────────┘  └─────────┘  │ │  │
│                      │  └─────────────────────────────┘ │  │
│                      └──────────────────────────────────┘  │
│                                                             │
│  serviceToken: Lambda ARN ou Step Functions ARN             │
└─────────────────────────────────────────────────────────────┘
         │
         │ serviceToken
         ▼
┌─────────────────┐
│  CustomResource │  ◀─── CloudFormation trata como um recurso comum
│  (construct)    │
└─────────────────┘
```

A montagem mínima em TypeScript:

```typescript
import * as cr from 'aws-cdk-lib/custom-resources';
import { CustomResource, Duration } from 'aws-cdk-lib';
import * as lambda from 'aws-cdk-lib/aws-lambda';

// 1. Sua Lambda de handler
const onEventHandler = new lambda.Function(this, 'OnEvent', {
  runtime: lambda.Runtime.NODEJS_22_X,
  handler: 'index.handler',
  code: lambda.Code.fromInline(`
    exports.handler = async (event) => {
      console.log('Event:', JSON.stringify(event));
      const physicalId = event.PhysicalResourceId || 'my-resource-' + Date.now();
      
      if (event.RequestType === 'Delete') {
        // limpa o recurso externo aqui
        return { PhysicalResourceId: physicalId };
      }
      
      // cria ou atualiza
      const result = await doSomething(event.ResourceProperties);
      return {
        PhysicalResourceId: result.id,
        Data: { Endpoint: result.endpoint, ApiKey: result.apiKey },
      };
    };
  `),
});

// 2. Provider que registra o handler
const provider = new cr.Provider(this, 'Provider', {
  onEventHandler,
  // isCompleteHandler: opcionalmente uma segunda Lambda para polling
  // totalTimeout: Duration.minutes(30),  // máx para operações async
});

// 3. Custom Resource conectado ao Provider
const resource = new CustomResource(this, 'Resource', {
  serviceToken: provider.serviceToken,
  properties: {
    Domain: 'app.example.com',
    Version: '2',   // mudar este valor no futuro dispara um UPDATE
  },
  resourceType: 'Custom::AcmeCertRegistration',   // prefixo Custom:: obrigatório
});

// 4. Lendo atributos retornados pelo handler (via Data{})
const endpoint = resource.getAttString('Endpoint');
```

[FATO] A propriedade `serviceToken` do Provider pode ser o ARN de uma Lambda ou o ARN de um Step Functions Express Workflow (quando `isCompleteHandler` é definido). Quando há `isCompleteHandler`, o Provider cria uma state machine que chama `onEvent` uma vez, depois fica fazendo polling em `isComplete` a cada `queryInterval` (padrão 5 segundos) até retornar `{ IsComplete: true }` ou atingir o `totalTimeout`.

---

### 3. AwsCustomResource — o atalho para chamadas de SDK únicas

[FATO] `AwsCustomResource` + `AwsSdkCall` é uma abstração de alto nível que permite executar qualquer chamada de SDK AWS sem escrever código Lambda. Internamente, ele usa o `Provider` com uma Lambda genérica que executa chamadas de SDK dinamicamente.

Caso de uso clássico: buscar um parâmetro do SSM Parameter Store em uma região diferente da stack (o `ssm.StringParameter.valueFromLookup` só funciona em tempo de síntese, não em tempo de deploy):

```typescript
import { AwsCustomResource, AwsSdkCall, PhysicalResourceId } from 'aws-cdk-lib/custom-resources';
import * as iam from 'aws-cdk-lib/aws-iam';

const getParam = new AwsCustomResource(this, 'GetParam', {
  onUpdate: {   // 'onUpdate' é chamado tanto em Create quanto em Update
    service: 'SSM',
    action: 'getParameter',
    parameters: {
      Name: '/shared/database-endpoint',
      WithDecryption: true,
    },
    region: 'us-east-1',                         // região diferente da stack!
    physicalResourceId: PhysicalResourceId.of(Date.now().toString()),
  },
  policy: AwsCustomResourcePolicy.fromSdkCalls({
    resources: AwsCustomResourcePolicy.ANY_RESOURCE,  // em prod: ARN específico
  }),
});

const dbEndpoint = getParam.getResponseField('Parameter.Value');
```

[CONSENSO] O `AwsCustomResource` exige que você defina explicitamente a `policy` — as permissões IAM que a Lambda genérica precisa para executar a chamada SDK. Usar `ANY_RESOURCE` é conveniente em desenvolvimento, mas em produção sempre restrinja ao ARN específico do recurso.

---

### 4. Aspects e IAspect: o padrão Visitor na árvore de constructs

[FATO] Um Aspect é qualquer objeto que implemente a interface `IAspect`:

```typescript
interface IAspect {
  visit(node: IConstruct): void;
}
```

Você o aplica a um escopo com `Aspects.of(scope).add(aspect)`. O CDK, durante a fase de síntese (após a construção de toda a árvore), percorre a árvore em **depth-first, pre-order** invocando `visit(node)` em cada construct:

```
App
└── Stack
    ├── Bucket1          ← visit() chamado aqui
    ├── Lambda1          ← e aqui
    │   └── Role         ← e aqui (filho de Lambda1)
    └── Queue1           ← e aqui
```

Se você aplicar o Aspect na `App`, ele visita todos os constructs da aplicação. Se aplicar em uma `Stack` específica, visita apenas os constructs daquela stack.

[FATO] Dentro de `visit()`, você pode:

1. **Inspecionar** o construct e adicionar anotações de erro ou aviso:
```typescript
import { Annotations } from 'aws-cdk-lib';
import * as s3 from 'aws-cdk-lib/aws-s3';

class BucketEncryptionChecker implements IAspect {
  visit(node: IConstruct): void {
    if (node instanceof s3.CfnBucket) {
      if (!node.bucketEncryption) {
        Annotations.of(node).addError(
          'Todos os buckets S3 devem ter encriptação configurada. ' +
          'Use BucketEncryption.S3_MANAGED ou KMS.'
        );
      }
    }
  }
}
```

2. **Mutar** o construct, adicionando propriedades ou chamando métodos:
```typescript
class EnforceS3Encryption implements IAspect {
  visit(node: IConstruct): void {
    if (node instanceof s3.CfnBucket) {
      // força encriptação SSE-S3 em qualquer bucket que não tenha uma já
      if (!node.bucketEncryption) {
        node.bucketEncryption = {
          serverSideEncryptionConfiguration: [{
            serverSideEncryptionByDefault: {
              sseAlgorithm: 'AES256',
            },
          }],
        };
      }
    }
  }
}
```

[FATO] `addError()` faz o `cdk synth` falhar com uma mensagem descritiva — o CloudAssembly não é gerado. `addWarning()` e `addInfo()` apenas emitem mensagens mas não interrompem a síntese. Isso torna Aspects adequados para **gates de compliance**: a pipeline CDK falha no `synth` antes mesmo de chegar ao deploy.

---

### 5. Aspects na prática: Tags, Annotations e limitações

[FATO] O próprio sistema de `Tags` do CDK é implementado internamente como um Aspect. Quando você chama `Tags.of(stack).add('Environment', 'production')`, o CDK registra um Aspect que, ao visitar cada construct, adiciona a tag ao CloudFormation resource correspondente.

[FATO] Ao visitar a árvore, o `node` recebido em `visit()` é tipado como `IConstruct`. Para agir apenas em um tipo específico de recurso, use `instanceof`:

```typescript
// Para constructs L2 (nível CDK):
if (node instanceof s3.Bucket) { ... }

// Para recursos L1 (nível CloudFormation):
if (node instanceof s3.CfnBucket) { ... }
```

[FATO] Existe uma diferença importante entre visitar o L2 e o L1:

- Visitar `s3.Bucket` (L2) dá acesso à API de alto nível do CDK (`bucket.addLifecycleRule()`, `bucket.grantRead()`). Porém, o L2 pode não existir se o recurso foi criado via L1 diretamente.
- Visitar `s3.CfnBucket` (L1) garante que você vê **todos** os buckets, mas você manipula propriedades CloudFormation diretamente (como no exemplo de `bucketEncryption` acima).

[CONSENSO] A convenção dominante na comunidade CDK é visitar o L1 em Aspects de compliance e mutação, porque é o denominador comum — todo recurso L2 eventualmente cria um L1.

**Limitação crítica:** [FATO] Você **não deve criar novos constructs** dentro de `visit()`. O CDK não garante a ordem de aplicação de Aspects em relação à construção da árvore, e adicionar constructs durante a visita pode causar comportamentos indefinidos (constructs ignorados, loops de síntese). Se precisar criar infraestrutura condicionalmente, crie-a no construtor e habilite/desabilite via propriedades.

---

## Exemplo prático

**Cenário:** Você tem uma stack de infraestrutura que cria buckets S3 em diferentes partes do código. A política de segurança da empresa exige que todo bucket tenha encriptação SSE-KMS (não SSE-S3) e que tenha o versionamento habilitado. Em vez de auditar cada `new s3.Bucket(...)` manualmente, você quer que o `cdk synth` falhe automaticamente se algum bucket violar as regras.

### Estrutura de arquivos

```
lib/
  aspects/
    bucket-compliance.ts   ← Aspect de validação
  stacks/
    storage-stack.ts       ← Stack com buckets
  app.ts                   ← Ponto de entrada
```

### `lib/aspects/bucket-compliance.ts`

```typescript
import { IAspect, Annotations } from 'aws-cdk-lib';
import * as s3 from 'aws-cdk-lib/aws-s3';
import { IConstruct } from 'constructs';

export class BucketComplianceAspect implements IAspect {
  visit(node: IConstruct): void {
    // Visita apenas recursos CfnBucket (L1) para cobrir todos os buckets
    if (!(node instanceof s3.CfnBucket)) return;

    // Regra 1: Encriptação KMS obrigatória
    const enc = node.bucketEncryption as s3.CfnBucket.BucketEncryptionProperty | undefined;
    const hasKms = enc?.serverSideEncryptionConfiguration?.some(
      (rule: any) =>
        rule.serverSideEncryptionByDefault?.sseAlgorithm === 'aws:kms'
    );
    if (!hasKms) {
      Annotations.of(node).addError(
        '[SECURITY] Bucket sem encriptação SSE-KMS. ' +
        'Configure encryptionKey ou use BucketEncryption.KMS.'
      );
    }

    // Regra 2: Versionamento obrigatório
    const versioning = node.versioningConfiguration as
      s3.CfnBucket.VersioningConfigurationProperty | undefined;
    if (versioning?.status !== 'Enabled') {
      Annotations.of(node).addWarning(
        '[COMPLIANCE] Bucket sem versionamento habilitado. ' +
        'Habilite com versioned: true.'
      );
    }
  }
}
```

### `lib/stacks/storage-stack.ts`

```typescript
import { Stack, StackProps } from 'aws-cdk-lib';
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as kms from 'aws-cdk-lib/aws-kms';
import { Construct } from 'constructs';

export class StorageStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    const key = new kms.Key(this, 'StorageKey', { enableKeyRotation: true });

    // Bucket compliant: KMS + versionamento
    new s3.Bucket(this, 'LogsBucket', {
      encryptionKey: key,
      encryption: s3.BucketEncryption.KMS,
      versioned: true,
    });

    // Bucket NÃO compliant: sem encriptação e sem versionamento
    // O Aspect vai emitir um error e um warning aqui
    new s3.Bucket(this, 'TempBucket');
  }
}
```

### `lib/app.ts`

```typescript
import { App, Aspects } from 'aws-cdk-lib';
import { StorageStack } from './stacks/storage-stack';
import { BucketComplianceAspect } from './aspects/bucket-compliance';

const app = new App();
const storageStack = new StorageStack(app, 'StorageStack');

// Aplica o Aspect na stack inteira
Aspects.of(storageStack).add(new BucketComplianceAspect());

app.synth();
```

### Resultado de `cdk synth`

```
[Error at /StorageStack/TempBucket/Resource] [SECURITY] Bucket sem encriptação 
SSE-KMS. Configure encryptionKey ou use BucketEncryption.KMS.

[Warning at /StorageStack/TempBucket/Resource] [COMPLIANCE] Bucket sem 
versionamento habilitado. Habilite com versioned: true.

Found errors
```

A síntese falha (`Found errors`). O `TempBucket` precisa ser corrigido antes que qualquer deploy aconteça.

---

### Bônus: CustomResource combinado com AwsCustomResource

Problema real: você precisa, ao criar a stack, registrar um webhook em um serviço externo (ex: GitHub) usando a API REST desse serviço. O `AwsCustomResource` cobre chamadas SDK AWS, mas não chamadas HTTP arbitrárias. Aqui você usa o `Provider` completo.

```typescript
// lib/constructs/github-webhook.ts
import { Construct, IConstruct } from 'constructs';
import { CustomResource, SecretValue } from 'aws-cdk-lib';
import * as cr from 'aws-cdk-lib/custom-resources';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as path from 'path';

export class GithubWebhook extends Construct {
  constructor(scope: Construct, id: string, props: {
    repo: string;
    webhookUrl: string;
    githubTokenSecret: string; // ARN do Secrets Manager
  }) {
    super(scope, id);

    const handler = new lambda.Function(this, 'Handler', {
      runtime: lambda.Runtime.NODEJS_22_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset(path.join(__dirname, '..', 'lambda', 'github-webhook')),
      environment: {
        GITHUB_TOKEN_SECRET: props.githubTokenSecret,
      },
    });

    // Permissão para ler o secret
    // handler.addToRolePolicy(new iam.PolicyStatement({ ... }));

    const provider = new cr.Provider(this, 'Provider', {
      onEventHandler: handler,
    });

    new CustomResource(this, 'Resource', {
      serviceToken: provider.serviceToken,
      properties: {
        Repo: props.repo,
        WebhookUrl: props.webhookUrl,
        // Incluir um hash do token faz o webhook ser re-registrado se o token mudar
        TokenHash: SecretValue.secretsManager(props.githubTokenSecret).toString(),
      },
      resourceType: 'Custom::GithubWebhook',
    });
  }
}
```

---

## Armadilhas comuns

### Armadilha 1: Mudar o PhysicalResourceId em um Update causa Delete do recurso antigo

**O erro:** Você tem um Custom Resource que gera um ID baseado em algumas propriedades. Em uma atualização, você muda uma propriedade que faz o ID ser diferente. O CloudFormation envia um `UPDATE` para o novo ID e depois um `DELETE` para o ID antigo — sua Lambda vai tentar deletar um recurso que acabou de ser criado com nome diferente, o que é frequentemente correto, mas pode ser um bug silencioso se você não estava esperando o `DELETE`.

**Por que acontece:** O protocolo CloudFormation de Custom Resource especifica que, se o `PhysicalResourceId` muda em um Update, o recurso antigo foi "substituído" e deve ser deletado.

**Como reconhecer:** Nos logs da Lambda (CloudWatch), você vê dois eventos: `RequestType: Update` seguido de `RequestType: Delete` com o ID anterior.

**Como evitar:** Seja deliberado sobre quando mudar o `PhysicalResourceId`. Se a operação é sempre in-place (ex: atualizar configuração de um recurso existente), retorne sempre o mesmo ID. Se a operação cria um novo recurso (ex: registrar um certificado com um domínio diferente), mudar o ID é correto — mas garanta que o `DELETE` handler saiba lidar com a tentativa de deletar algo que pode não existir mais.

---

### Armadilha 2: Aspect visitando L2 não pega buckets criados via L1

**O erro:** Você escreve `if (node instanceof s3.Bucket)` no seu Aspect, mas parte da infraestrutura usa `new s3.CfnBucket(...)` diretamente. O Aspect passa por esses recursos sem detectá-los.

**Por que acontece:** `s3.Bucket` e `s3.CfnBucket` são classes diferentes na hierarquia. O `instanceof` do TypeScript verifica o tipo exato, não a relação com o recurso CloudFormation subjacente.

**Como reconhecer:** `cdk synth` passa sem erros para um `CfnBucket` que deveria ser interceptado. Uma revisão manual ou um teste de unidade (usando `Template.hasResourceProperties`) pode revelar o problema.

**Como evitar:** Em Aspects de compliance que precisam cobrir 100% dos recursos de um tipo, sempre visite o L1 (`CfnBucket`, `CfnFunction`, etc.). Se você precisa visitar o L2 por algum motivo, adicione também uma checagem separada para o L1.

---

### Armadilha 3: Criar constructs dentro de visit() causa comportamento indefinido

**O erro:** Dentro de `visit(node)`, você identifica que um bucket não tem um bucket de log e tenta criar um `new s3.Bucket(this, 'AccessLogs', ...)` para configurar automaticamente. Na primeira execução parece funcionar, mas em deployments subsequentes o CDK emite warnings sobre síntese não-determinística, ou o novo bucket não é incluído nos Aspects já aplicados.

**Por que acontece:** A fase de síntese percorre a árvore uma vez. Adicionar novos constructs durante essa travessia é como adicionar elementos a uma lista enquanto você a itera — o comportamento é indefinido pela especificação do CDK.

**Como evitar:** Nunca instancie `new SomeConstruct(...)` dentro de `visit()`. Se precisar criar recursos condicionalmente, crie-os no construtor e use flags ou props para controlar a criação. Se precisar detectar a ausência de algo e corrigir, considere criar um construct L2 customizado que encapsula as regras no construtor, em vez de usar um Aspect para mutação.

---

## Exercício de reflexão

Você está construindo uma plataforma interna onde times diferentes criam stacks CDK independentes. A política de segurança determina:

1. Todo bucket S3 deve usar SSE-KMS com uma KMS key que tenha rotação automática habilitada.
2. Todo tópico SNS deve ter encriptação em repouso habilitada.
3. Toda Lambda function deve ter um DLQ (Dead Letter Queue) configurado.
4. Nenhum Security Group pode ter uma regra de ingress com cidr `0.0.0.0/0` na porta 22.

Como você estruturaria Aspects para implementar essas quatro regras? Para cada regra, decida: você visitaria o L1 ou o L2? Você usaria `addError()` (bloqueante) ou `addWarning()` (informativo)? Como você organizaria os Aspects no código — um único Aspect com todos os checks, ou um Aspect por regra? Considere como essa solução escalaria se a empresa crescesse para 20 regras de compliance e 50 times com stacks independentes. Há casos em que um Aspect de mutação automática (corrigir silenciosamente em vez de reportar o erro) seria preferível? Quais são os riscos dessa abordagem em um ambiente de time?

---

## Recursos para aprofundar

### 1. CDK v2 Guide — Custom Resources (cfn_layer)
**URL:** https://docs.aws.amazon.com/cdk/v2/guide/cfn_layer.html#develop-customize-expose
**O que encontrar:** Seção "Develop a custom resource provider" explica a camada de escape do CDK para interagir diretamente com CloudFormation Custom Resources, incluindo como expor atributos via `Data` e como usar o escape hatch `node.defaultChild` para customizações.
**Por que é a fonte certa:** É a documentação oficial do CDK para a abordagem "expose" — o ponto mais próximo de criar seus próprios tipos de recursos via CDK.

### 2. aws-cdk-lib.custom_resources module README
**URL:** https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.custom_resources-readme.html
**O que encontrar:** Documentação completa do `Provider`, `AwsCustomResource`, e `AwsSdkCall`. Inclui diagramas da arquitetura interna do Provider (com Step Functions), exemplos de `isCompleteHandler` para recursos assíncronos, e a lista completa de propriedades do response.
**Por que é a fonte certa:** É a referência canônica do módulo — mais detalhada que o guia narrativo.

### 3. CDK v2 Guide — Aspects
**URL:** https://docs.aws.amazon.com/cdk/v2/guide/aspects.html
**O que encontrar:** Exemplos de Aspects para validação (com `addError`) e para aplicação de tags. Explica a ordem de execução (depth-first, pre-order) e como aplicar Aspects em diferentes escopos (App, Stack, Construct individual).
**Por que é a fonte certa:** É o documento oficial da feature, com exemplos em TypeScript, Python e Java, e esclarece as garantias e limitações do mecanismo.

---

