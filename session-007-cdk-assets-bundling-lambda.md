# Sessão 007 — CDK: Assets — bundling de Lambda, Docker images e arquivos locais

**Duração estimada:** 60 minutos
**Pré-requisitos:** session-006 — CDK: Stacks, environments e multi-account patterns

---

## Objetivo

Ao final, você conseguirá fazer deploy de uma função Lambda com dependências bundled (via `aws_lambda_python_alpha` ou `NodejsFunction`), publicar uma imagem Docker via `DockerImageFunction`, e entender como assets são staged no S3/ECR pelo CDK.

---

## Contexto

[FATO] Assets são arquivos locais ou imagens Docker que o CDK precisa publicar na AWS antes de fazer o deploy dos stacks. Todo código de Lambda, toda imagem de container, e todo arquivo local referenciado num template CloudFormation passa pelo pipeline de assets.

[CONSENSO] Entender como assets funcionam é essencial para diagnosticar dois problemas comuns em produção: deploys lentos (re-upload desnecessário de assets que não mudaram) e falhas em pipelines CI/CD sem Docker instalado (bundling que depende de Docker mas o agente não tem).

---

## Conceitos principais

### 1. Como assets funcionam — o ciclo completo

Quando o CDK encontra um asset durante o `synth`, ele:

1. Calcula um **source hash** do conteúdo (SHA256)
2. Copia o asset para `cdk.out/<hash>/`
3. Registra no `manifest.json` do cloud assembly as instruções de publicação
4. Durante o `deploy`, verifica se o asset com esse hash já existe no destino (S3/ECR)
5. Se não existe, faz upload; se existe, pula — **idempotente por hash**

```
cdk synth
  └── cdk.out/
        ├── MeuStack.template.json
        ├── asset.a1b2c3d4.../         ← conteúdo do asset (zip ou Dockerfile)
        │     └── index.py
        ├── asset.e5f6g7h8.zip         ← asset já zipado
        └── manifest.json             ← instruções de upload

cdk deploy
  └── CDK CLI lê manifest.json
  └── Para cada asset:
        ├── Calcula hash local
        ├── Verifica se s3://cdk-assets-ACCOUNT-REGION/<hash> existe
        └── Se não existe: faz upload
        └── Passa a chave S3/ECR como parâmetro ao CloudFormation
```

**Tipos de assets:**

```
File assets     → arquivos/diretórios locais → zip → S3 (bucket do bootstrap)
Docker assets   → Dockerfile local → docker build → push → ECR (repo do bootstrap)
```

**O hash garante imutabilidade:**

```
Mesmo código + mesma versão de dependências = mesmo hash = sem re-upload
Mudança em qualquer arquivo do diretório = novo hash = novo upload
```

---

### 2. File assets — arquivos locais para S3

O tipo mais simples de asset: um arquivo ou diretório local que vai para o S3 do bootstrap.

```typescript
import * as s3_assets from 'aws-cdk-lib/aws-s3-assets';

// Um arquivo local referenciado no template
const asset = new s3_assets.Asset(this, 'ConfigFile', {
  path: path.join(__dirname, '../config/app-config.json'),
});

// O asset expõe: asset.s3BucketName, asset.s3ObjectKey, asset.httpUrl
console.log(asset.s3ObjectKey);  // hash-based key

// Exemplo: passar config para uma instância EC2 via UserData
const instance = new ec2.Instance(this, 'Server', { ... });
asset.grantRead(instance.role);  // permissão de leitura para a instância

instance.userData.addS3DownloadCommand({
  bucket: asset.bucket,
  bucketKey: asset.s3ObjectKey,
  localFile: '/etc/app/config.json',
});
```

**Diretório como asset (zipa automaticamente):**

```typescript
const dirAsset = new s3_assets.Asset(this, 'AppCode', {
  path: path.join(__dirname, '../src'),
  // exclude: ['*.test.ts', 'node_modules']  ← arquivos a ignorar no hash/zip
  exclude: ['**/*.test.ts', 'node_modules/**'],
});
```

---

### 3. Lambda com código inline — o mais simples (sem asset)

Para funções pequenas sem dependências externas, o código pode ser inline:

```typescript
import * as lambda from 'aws-cdk-lib/aws-lambda';

const fn = new lambda.Function(this, 'SimpleHandler', {
  runtime: lambda.Runtime.PYTHON_3_12,
  handler: 'index.handler',
  code: lambda.Code.fromInline(`
def handler(event, context):
    return {'statusCode': 200, 'body': 'ok'}
  `),
});
```

[FATO] `Code.fromInline` tem limite de 4 KB. Para qualquer coisa maior, use `Code.fromAsset` ou os constructs de bundling.

**Lambda com arquivo local (sem bundling):**

```typescript
// Zipa o diretório e faz upload para S3
const fn = new lambda.Function(this, 'Handler', {
  runtime: lambda.Runtime.PYTHON_3_12,
  handler: 'index.handler',
  code: lambda.Code.fromAsset(path.join(__dirname, '../lambda/handler')),
  // ⚠️ Sem bundling: dependências precisam já estar no diretório
  // Você é responsável por rodar pip install -t ./handler antes do synth
});
```

---

### 4. NodejsFunction — bundling TypeScript/JS com esbuild

[FATO] `NodejsFunction` é o construct de alto nível para Lambdas em TypeScript ou JavaScript. Ele usa o **esbuild** para transpilar e empacotar o código localmente (sem Docker, muito rápido) ou dentro de um container Docker (fallback quando esbuild não está disponível).

```typescript
import * as lambda_nodejs from 'aws-cdk-lib/aws-lambda-nodejs';

const fn = new lambda_nodejs.NodejsFunction(this, 'ApiHandler', {
  // entry: ponto de entrada — o CDK localiza automaticamente se o arquivo
  // tem o mesmo nome que o construct ID + está no mesmo diretório
  entry: path.join(__dirname, '../lambda/api-handler.ts'),
  handler: 'handler',              // nome da função exportada
  runtime: lambda.Runtime.NODEJS_22_X,
  architecture: lambda.Architecture.ARM_64,

  bundling: {
    minify: true,                  // produção: sim; dev: opcional
    sourceMap: true,               // mapa de fontes para debugging
    target: 'es2022',
    externalModules: [
      '@aws-sdk/*',                // AWS SDK v3 já incluído no runtime da Lambda
    ],
    // Dependências que NÃO devem ser bundled (ficam em node_modules)
    nodeModules: ['sharp'],        // módulos com binários nativos
  },

  environment: {
    TABLE_NAME: table.tableName,
    LOG_LEVEL: 'INFO',
  },

  timeout: cdk.Duration.seconds(30),
  memorySize: 256,
});
```

**O que o esbuild faz:**

```
Antes do bundling (diretório do projeto):
  api-handler.ts
  utils/logger.ts
  utils/validator.ts
  node_modules/
    zod/
    axios/
    @aws-sdk/

Depois do bundling (zip enviado para a Lambda):
  index.js     ← tudo num único arquivo (tree-shaking incluso)
  # aws-sdk foi excluído (já está no runtime)
  # zod e axios foram incluídos (bundled inline)
  # sharp foi excluído (vai em node_modules separado)
```

**Bundling local vs Docker:**

```
Bundling LOCAL (padrão quando esbuild disponível):
  ✅ Muito mais rápido (segundos vs minutos)
  ✅ Sem necessidade de Docker
  ⚠️  Módulos com binários nativos (sharp, bcrypt) podem ter arquitetura errada
       → use nodeModules para deixá-los fora do bundle principal

Bundling em DOCKER (fallback):
  ✅ Garante compatibilidade com o ambiente Lambda (Amazon Linux 2)
  ✅ Correto para módulos com binários nativos
  ⚠️  Lento — cada synth rebuilda a imagem
  ⚠️  Requer Docker no ambiente de build

Forçar Docker:
  bundling: { forceDockerBundling: true }
```

---

### 5. PythonFunction — bundling Python com dependências

[FATO] `PythonFunction` é um construct **alpha** (`@aws-cdk/aws-lambda-python-alpha`) que gerencia o bundling de funções Python, instalando dependências em um container Docker compatível com Lambda.

```bash
# Instalar o pacote alpha separadamente
npm install @aws-cdk/aws-lambda-python-alpha
# ou para Python:
pip install aws-cdk.aws-lambda-python-alpha
```

```typescript
import * as lambda_python from '@aws-cdk/aws-lambda-python-alpha';

const fn = new lambda_python.PythonFunction(this, 'DataProcessor', {
  entry: path.join(__dirname, '../lambda/processor'),
  // O CDK detecta automaticamente o gerenciador de dependências:
  // requirements.txt → pip
  // Pipfile          → pipenv
  // poetry.lock      → poetry
  // uv.lock          → uv

  runtime: lambda.Runtime.PYTHON_3_12,
  handler: 'handler',             // função handler em index.py (padrão)
  index: 'processor.py',          // se não for index.py
  architecture: lambda.Architecture.ARM_64,

  bundling: {
    assetHashType: cdk.AssetHashType.OUTPUT,  // hash do output, não do source
    // Image customizada para o bundling (se precisar de dependências de sistema)
    image: DockerImage.fromBuild(path.join(__dirname, '../docker/build-env')),
  },

  environment: {
    BUCKET_NAME: bucket.bucketName,
  },
});
```

**Estrutura do diretório da função:**

```
lambda/processor/
├── processor.py          ← código da função
├── requirements.txt      ← dependências (ou pyproject.toml, Pipfile, etc.)
└── utils/
    └── helpers.py

# requirements.txt:
boto3>=1.26.0
pandas==2.0.3
pydantic>=2.0
```

[FATO] O `PythonFunction` roda `pip install` dentro de um container Docker com a imagem base do Lambda (ex: `public.ecr.aws/sam/build-python3.12`). Isso garante que módulos com C extensions (numpy, pandas) sejam compilados para a arquitetura correta.

**Por que o status alpha importa:**

[FATO] Constructs `alpha` no CDK têm APIs que podem mudar entre versões sem aviso de deprecation. Para produção, versione o pacote alpha explicitamente e monitore o changelog antes de atualizar. O `PythonFunction` está em alpha há bastante tempo — é amplamente usado, mas a API não é estável.

---

### 6. DockerImageFunction — Lambda com imagem Docker completa

Quando você precisa de controle total sobre o runtime (binários customizados, linguagens não suportadas nativamente, múltiplos arquivos de sistema), use `DockerImageFunction`:

```typescript
import * as lambda from 'aws-cdk-lib/aws-lambda';

const fn = new lambda.DockerImageFunction(this, 'CustomRuntime', {
  code: lambda.DockerImageCode.fromImageAsset(
    path.join(__dirname, '../docker/my-function'),
    {
      // Build args para o Dockerfile
      buildArgs: {
        FUNCTION_VERSION: '1.2.3',
      },
      // Platform para Lambda ARM64
      platform: ecr_assets.Platform.LINUX_ARM64,
      // Arquivo Dockerfile customizado (padrão: Dockerfile)
      file: 'Dockerfile.lambda',
    }
  ),
  architecture: lambda.Architecture.ARM_64,
  timeout: cdk.Duration.minutes(5),
  memorySize: 1024,
});
```

**Dockerfile para Lambda:**

```dockerfile
# Dockerfile na pasta docker/my-function/
FROM public.ecr.aws/lambda/python:3.12

# Instalar dependências de sistema
RUN yum install -y libgomp && yum clean all

# Copiar e instalar dependências Python
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copiar código da função
COPY app/ ${LAMBDA_TASK_ROOT}/

# Handler
CMD ["app.handler"]
```

**Diferença entre `DockerImageFunction` e `PythonFunction`:**

```
PythonFunction
  → Usa Docker apenas para BUNDLING (instalar deps)
  → O resultado é um zip enviado para S3
  → Runtime é gerenciado pela AWS (Lambda managed runtime)
  → Mais leve, mais rápido no cold start

DockerImageFunction
  → A imagem Docker É o runtime
  → Imagem vai para ECR
  → Você controla 100% do ambiente
  → Cold start ligeiramente mais lento para imagens grandes
  → Use quando: runtime customizado, binários de sistema, > 250 MB zip
```

---

### 7. Como assets são staged — o fluxo completo

```
cdk synth
  │
  ├─ NodejsFunction: esbuild roda localmente
  │     └── cdk.out/asset.HASH_A/index.js   (bundle gerado)
  │
  ├─ PythonFunction: docker build roda
  │     └── cdk.out/asset.HASH_B/            (libs instaladas)
  │
  └─ DockerImageFunction: docker build + tag
        └── cdk.out/asset.HASH_C/            (imagem buildada localmente)

cdk deploy
  │
  ├─ asset.HASH_A → zip → s3://cdk-XXXX-assets-ACCOUNT-REGION/HASH_A.zip
  │                 ↑ só faz upload se ainda não existe
  │
  ├─ asset.HASH_B → zip → s3://cdk-XXXX-assets-ACCOUNT-REGION/HASH_B.zip
  │
  └─ asset.HASH_C → docker push → ECR (repo do bootstrap)
                    ↑ só faz push se o digest não existe

CloudFormation deploy
  └── Recebe como parâmetros:
        AssetBucketName:   cdk-XXXX-assets-ACCOUNT-REGION
        AssetObjectKey:    HASH_A.zip
        DockerImageUri:    ACCOUNT.dkr.ecr.REGION.amazonaws.com/cdk-XXXX-...:HASH_C
```

**Verificando os assets gerados:**

```bash
# Ver todos os assets no cloud assembly
cat cdk.out/manifest.json | jq '.artifacts | to_entries[] | select(.value.type == "aws:cloudformation:stack") | .value.metadata'

# Ver o tamanho dos assets antes do upload
du -sh cdk.out/asset.*

# Inspecionar o conteúdo de um asset Lambda
unzip -l cdk.out/asset.XXXX.zip
```

---

## Exemplo prático

Stack com três tipos de Lambda usando assets diferentes:

```typescript
import * as cdk from 'aws-cdk-lib';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as lambda_nodejs from 'aws-cdk-lib/aws-lambda-nodejs';
import * as lambda_python from '@aws-cdk/aws-lambda-python-alpha';
import * as path from 'path';

export class LambdaAssetsStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // 1. TypeScript com NodejsFunction (esbuild)
    const apiHandler = new lambda_nodejs.NodejsFunction(this, 'ApiHandler', {
      entry: path.join(__dirname, '../src/api/handler.ts'),
      runtime: lambda.Runtime.NODEJS_22_X,
      architecture: lambda.Architecture.ARM_64,
      bundling: {
        minify: true,
        externalModules: ['@aws-sdk/*'],
      },
    });

    // 2. Python com dependências (PythonFunction)
    const processor = new lambda_python.PythonFunction(this, 'Processor', {
      entry: path.join(__dirname, '../src/processor'),
      runtime: lambda.Runtime.PYTHON_3_12,
      architecture: lambda.Architecture.ARM_64,
    });

    // 3. Container customizado (DockerImageFunction)
    const mlInference = new lambda.DockerImageFunction(this, 'MLInference', {
      code: lambda.DockerImageCode.fromImageAsset(
        path.join(__dirname, '../docker/ml-inference'),
        { platform: ecr_assets.Platform.LINUX_ARM64 }
      ),
      architecture: lambda.Architecture.ARM_64,
      memorySize: 3008,
      timeout: cdk.Duration.minutes(5),
    });

    // Outputs para inspecionar
    new cdk.CfnOutput(this, 'ApiHandlerArn',   { value: apiHandler.functionArn });
    new cdk.CfnOutput(this, 'ProcessorArn',    { value: processor.functionArn });
    new cdk.CfnOutput(this, 'MLInferenceArn',  { value: mlInference.functionArn });
  }
}
```

```bash
# Antes do deploy, ver o que vai ser construído
cdk synth 2>&1 | grep -E "Bundling|Building|Asset"

# Deploy e monitorar o upload dos assets
cdk deploy --verbose 2>&1 | grep -E "upload|push|asset"

# Ver o tamanho final de cada função no Lambda
aws lambda list-functions \
  --query 'Functions[?starts_with(FunctionName, `LambdaAssets`)].{Name:FunctionName,Size:CodeSize}' \
  --output table
```

---

## Armadilhas comuns

**1. Bundling que funciona localmente mas falha no CI**

O `NodejsFunction` usa esbuild localmente mas cai para Docker quando esbuild não está disponível. Se seu pipeline CI não tem Docker, e também não tem esbuild instalado, o synth falha. Solução: garanta `esbuild` como dev dependency no `package.json` do projeto (`npm install --save-dev esbuild`), assim o CI o instala junto com as demais deps.

**2. Hash do asset não muda quando dependência muda**

Se você usa `AssetHashType.SOURCE` (padrão), o hash é calculado sobre os arquivos de **entrada** — não sobre o output do bundling. Isso significa que atualizar uma versão de dependência no `requirements.txt` **sem** mudar o código da função não gera um novo hash e o CDK não faz re-upload. Use `AssetHashType.OUTPUT` em `PythonFunction` para que o hash reflita o resultado do bundling.

**3. `nodeModules` vs `externalModules` no NodejsFunction**

```
externalModules: ['sharp']   → sharp é excluído do bundle e NÃO incluído no zip
                               (assume que sharp já está disponível no runtime — não está)
                               → A Lambda vai falhar em runtime com "Cannot find module 'sharp'"

nodeModules: ['sharp']       → sharp é excluído do esbuild mas incluído no zip como node_modules/
                               → pip/npm install acontece dentro do container Docker
                               → Binário correto para Amazon Linux
```

Use `externalModules` apenas para o AWS SDK e módulos que você tem certeza que existem no runtime. Use `nodeModules` para módulos com binários nativos que precisam ser compilados para Lambda.

---

## Exercício de reflexão

Você está implementando um serviço de processamento de imagens em Python que usa a biblioteca `Pillow` (com C extensions) e `numpy`. A função precisa rodar em ARM64 para reduzir custo.

Você tem três opções: `PythonFunction` com bundling padrão, `PythonFunction` com imagem de build customizada, ou `DockerImageFunction` com Dockerfile próprio.

Para cada opção, descreva: o que acontece durante o `cdk synth`, o que é enviado para a AWS, o impacto no cold start da Lambda, e os requisitos do ambiente de CI/CD. Qual você escolheria para produção e por quê? Existe algum cenário onde a resposta mudaria?

---

## Recursos para aprofundar

**Assets:**
- [Assets and the AWS CDK](https://docs.aws.amazon.com/cdk/v2/guide/assets.html) — cobre os dois tipos (file e Docker), como o hash é calculado e como o CDK decide quando fazer re-upload.

**NodejsFunction:**
- [aws-cdk-lib.aws_lambda_nodejs module](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_lambda_nodejs-readme.html) — todas as opções de bundling com esbuild: `minify`, `sourceMap`, `externalModules`, `nodeModules`, `define`, `banner`.

**PythonFunction:**
- [@aws-cdk/aws-lambda-python-alpha module](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-lambda-python-alpha-readme.html) — gerenciadores de dependência suportados, opções de bundling e como usar imagem de build customizada.

---

