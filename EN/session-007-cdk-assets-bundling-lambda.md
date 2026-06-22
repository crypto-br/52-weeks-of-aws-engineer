# Session 007 — CDK: Assets — Lambda bundling, Docker images, and local files

**Estimated duration:** 60 minutes
**Prerequisites:** session-006 — CDK: Stacks, environments, and multi-account patterns

---

## Objective

By the end, you will be able to deploy a Lambda function with bundled dependencies (via `aws_lambda_python_alpha` or `NodejsFunction`), publish a Docker image via `DockerImageFunction`, and understand how assets are staged to S3/ECR by CDK.

---

## Context

[FACT] Assets are local files or Docker images that CDK needs to publish to AWS before deploying stacks. All Lambda code, every container image, and every local file referenced in a CloudFormation template goes through the asset pipeline.

[CONSENSUS] Understanding how assets work is essential for diagnosing two common problems in production: slow deploys (unnecessary re-upload of unchanged assets) and CI/CD pipeline failures without Docker installed (bundling that depends on Docker but the agent doesn't have it).

---

## Core concepts

### 1. How assets work — the complete cycle

When CDK encounters an asset during `synth`, it:

1. Calculates a **source hash** of the content (SHA256)
2. Copies the asset to `cdk.out/<hash>/`
3. Registers in the cloud assembly's `manifest.json` the publishing instructions
4. During `deploy`, checks if the asset with that hash already exists at the destination (S3/ECR)
5. If it doesn't exist, uploads; if it does, skips — **idempotent by hash**

```
cdk synth
  └── cdk.out/
        ├── MyStack.template.json
        ├── asset.a1b2c3d4.../         ← asset content (zip or Dockerfile)
        │     └── index.py
        ├── asset.e5f6g7h8.zip         ← already-zipped asset
        └── manifest.json             ← upload instructions

cdk deploy
  └── CDK CLI reads manifest.json
  └── For each asset:
        ├── Calculates local hash
        ├── Checks if s3://cdk-assets-ACCOUNT-REGION/<hash> exists
        └── If not: uploads
        └── Passes the S3/ECR key as parameter to CloudFormation
```

**Types of assets:**

```
File assets     → local files/directories → zip → S3 (bootstrap bucket)
Docker assets   → local Dockerfile → docker build → push → ECR (bootstrap repo)
```

**The hash guarantees immutability:**

```
Same code + same dependency version = same hash = no re-upload
Change in any file in the directory = new hash = new upload
```

---

### 2. File assets — local files to S3

The simplest type of asset: a local file or directory that goes to the bootstrap S3.

```typescript
import * as s3_assets from 'aws-cdk-lib/aws-s3-assets';

// A local file referenced in the template
const asset = new s3_assets.Asset(this, 'ConfigFile', {
  path: path.join(__dirname, '../config/app-config.json'),
});

// The asset exposes: asset.s3BucketName, asset.s3ObjectKey, asset.httpUrl
console.log(asset.s3ObjectKey);  // hash-based key

// Example: pass config to an EC2 instance via UserData
const instance = new ec2.Instance(this, 'Server', { ... });
asset.grantRead(instance.role);  // read permission for the instance

instance.userData.addS3DownloadCommand({
  bucket: asset.bucket,
  bucketKey: asset.s3ObjectKey,
  localFile: '/etc/app/config.json',
});
```

**Directory as asset (automatically zipped):**

```typescript
const dirAsset = new s3_assets.Asset(this, 'AppCode', {
  path: path.join(__dirname, '../src'),
  // exclude: files to ignore in the hash/zip
  exclude: ['**/*.test.ts', 'node_modules/**'],
});
```

---

### 3. Lambda with inline code — the simplest (no asset)

For small functions without external dependencies, code can be inline:

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

[FACT] `Code.fromInline` has a 4 KB limit. For anything larger, use `Code.fromAsset` or the bundling constructs.

**Lambda with local file (without bundling):**

```typescript
// Zips the directory and uploads to S3
const fn = new lambda.Function(this, 'Handler', {
  runtime: lambda.Runtime.PYTHON_3_12,
  handler: 'index.handler',
  code: lambda.Code.fromAsset(path.join(__dirname, '../lambda/handler')),
  // ⚠️ Without bundling: dependencies must already be in the directory
  // You are responsible for running pip install -t ./handler before synth
});
```

---

### 4. NodejsFunction — bundling TypeScript/JS with esbuild

[FACT] `NodejsFunction` is the high-level construct for TypeScript or JavaScript Lambdas. It uses **esbuild** to transpile and package the code locally (without Docker, very fast) or inside a Docker container (fallback when esbuild is not available).

```typescript
import * as lambda_nodejs from 'aws-cdk-lib/aws-lambda-nodejs';

const fn = new lambda_nodejs.NodejsFunction(this, 'ApiHandler', {
  // entry: entry point — CDK automatically locates it if the file
  // has the same name as the construct ID + is in the same directory
  entry: path.join(__dirname, '../lambda/api-handler.ts'),
  handler: 'handler',              // name of the exported function
  runtime: lambda.Runtime.NODEJS_22_X,
  architecture: lambda.Architecture.ARM_64,

  bundling: {
    minify: true,                  // production: yes; dev: optional
    sourceMap: true,               // source maps for debugging
    target: 'es2022',
    externalModules: [
      '@aws-sdk/*',                // AWS SDK v3 already included in Lambda runtime
    ],
    // Dependencies that should NOT be bundled (stay in node_modules)
    nodeModules: ['sharp'],        // modules with native binaries
  },

  environment: {
    TABLE_NAME: table.tableName,
    LOG_LEVEL: 'INFO',
  },

  timeout: cdk.Duration.seconds(30),
  memorySize: 256,
});
```

**What esbuild does:**

```
Before bundling (project directory):
  api-handler.ts
  utils/logger.ts
  utils/validator.ts
  node_modules/
    zod/
    axios/
    @aws-sdk/

After bundling (zip sent to Lambda):
  index.js     ← everything in a single file (tree-shaking included)
  # aws-sdk was excluded (already in the runtime)
  # zod and axios were included (bundled inline)
  # sharp was excluded (goes in separate node_modules)
```

**Local vs Docker bundling:**

```
LOCAL bundling (default when esbuild is available):
  ✅ Much faster (seconds vs minutes)
  ✅ No Docker required
  ⚠️  Modules with native binaries (sharp, bcrypt) may have wrong architecture
       → use nodeModules to keep them out of the main bundle

DOCKER bundling (fallback):
  ✅ Guarantees compatibility with Lambda environment (Amazon Linux 2)
  ✅ Correct for modules with native binaries
  ⚠️  Slow — each synth rebuilds the image
  ⚠️  Requires Docker in the build environment

Force Docker:
  bundling: { forceDockerBundling: true }
```

---

### 5. PythonFunction — bundling Python with dependencies

[FACT] `PythonFunction` is an **alpha** construct (`@aws-cdk/aws-lambda-python-alpha`) that manages the bundling of Python functions, installing dependencies in a Docker container compatible with Lambda.

```bash
# Install the alpha package separately
npm install @aws-cdk/aws-lambda-python-alpha
# or for Python:
pip install aws-cdk.aws-lambda-python-alpha
```

```typescript
import * as lambda_python from '@aws-cdk/aws-lambda-python-alpha';

const fn = new lambda_python.PythonFunction(this, 'DataProcessor', {
  entry: path.join(__dirname, '../lambda/processor'),
  // CDK automatically detects the dependency manager:
  // requirements.txt → pip
  // Pipfile          → pipenv
  // poetry.lock      → poetry
  // uv.lock          → uv

  runtime: lambda.Runtime.PYTHON_3_12,
  handler: 'handler',             // handler function in index.py (default)
  index: 'processor.py',          // if not index.py
  architecture: lambda.Architecture.ARM_64,

  bundling: {
    assetHashType: cdk.AssetHashType.OUTPUT,  // hash of the output, not the source
    // Custom image for bundling (if you need system dependencies)
    image: DockerImage.fromBuild(path.join(__dirname, '../docker/build-env')),
  },

  environment: {
    BUCKET_NAME: bucket.bucketName,
  },
});
```

**Function directory structure:**

```
lambda/processor/
├── processor.py          ← function code
├── requirements.txt      ← dependencies (or pyproject.toml, Pipfile, etc.)
└── utils/
    └── helpers.py

# requirements.txt:
boto3>=1.26.0
pandas==2.0.3
pydantic>=2.0
```

[FACT] `PythonFunction` runs `pip install` inside a Docker container with the Lambda base image (e.g., `public.ecr.aws/sam/build-python3.12`). This ensures that modules with C extensions (numpy, pandas) are compiled for the correct architecture.

**Why the alpha status matters:**

[FACT] `alpha` constructs in CDK have APIs that can change between versions without deprecation notice. For production, pin the alpha package version explicitly and monitor the changelog before upgrading. `PythonFunction` has been in alpha for a long time — it's widely used, but the API is not stable.

---

### 6. DockerImageFunction — Lambda with complete Docker image

When you need full control over the runtime (custom binaries, natively unsupported languages, multiple system files), use `DockerImageFunction`:

```typescript
import * as lambda from 'aws-cdk-lib/aws-lambda';

const fn = new lambda.DockerImageFunction(this, 'CustomRuntime', {
  code: lambda.DockerImageCode.fromImageAsset(
    path.join(__dirname, '../docker/my-function'),
    {
      // Build args for the Dockerfile
      buildArgs: {
        FUNCTION_VERSION: '1.2.3',
      },
      // Platform for Lambda ARM64
      platform: ecr_assets.Platform.LINUX_ARM64,
      // Custom Dockerfile (default: Dockerfile)
      file: 'Dockerfile.lambda',
    }
  ),
  architecture: lambda.Architecture.ARM_64,
  timeout: cdk.Duration.minutes(5),
  memorySize: 1024,
});
```

**Dockerfile for Lambda:**

```dockerfile
# Dockerfile in the docker/my-function/ folder
FROM public.ecr.aws/lambda/python:3.12

# Install system dependencies
RUN yum install -y libgomp && yum clean all

# Copy and install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy function code
COPY app/ ${LAMBDA_TASK_ROOT}/

# Handler
CMD ["app.handler"]
```

**Difference between `DockerImageFunction` and `PythonFunction`:**

```
PythonFunction
  → Uses Docker only for BUNDLING (installing deps)
  → The result is a zip sent to S3
  → Runtime is managed by AWS (Lambda managed runtime)
  → Lighter, faster cold start

DockerImageFunction
  → The Docker image IS the runtime
  → Image goes to ECR
  → You control 100% of the environment
  → Cold start slightly slower for large images
  → Use when: custom runtime, system binaries, > 250 MB zip
```

---

### 7. How assets are staged — the complete flow

```
cdk synth
  │
  ├─ NodejsFunction: esbuild runs locally
  │     └── cdk.out/asset.HASH_A/index.js   (generated bundle)
  │
  ├─ PythonFunction: docker build runs
  │     └── cdk.out/asset.HASH_B/            (installed libs)
  │
  └─ DockerImageFunction: docker build + tag
        └── cdk.out/asset.HASH_C/            (locally built image)

cdk deploy
  │
  ├─ asset.HASH_A → zip → s3://cdk-XXXX-assets-ACCOUNT-REGION/HASH_A.zip
  │                 ↑ only uploads if it doesn't already exist
  │
  ├─ asset.HASH_B → zip → s3://cdk-XXXX-assets-ACCOUNT-REGION/HASH_B.zip
  │
  └─ asset.HASH_C → docker push → ECR (bootstrap repo)
                    ↑ only pushes if the digest doesn't exist

CloudFormation deploy
  └── Receives as parameters:
        AssetBucketName:   cdk-XXXX-assets-ACCOUNT-REGION
        AssetObjectKey:    HASH_A.zip
        DockerImageUri:    ACCOUNT.dkr.ecr.REGION.amazonaws.com/cdk-XXXX-...:HASH_C
```

**Checking generated assets:**

```bash
# View all assets in the cloud assembly
cat cdk.out/manifest.json | jq '.artifacts | to_entries[] | select(.value.type == "aws:cloudformation:stack") | .value.metadata'

# View asset sizes before upload
du -sh cdk.out/asset.*

# Inspect the contents of a Lambda asset
unzip -l cdk.out/asset.XXXX.zip
```

---

## Practical example

Stack with three types of Lambda using different assets:

```typescript
import * as cdk from 'aws-cdk-lib';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as lambda_nodejs from 'aws-cdk-lib/aws-lambda-nodejs';
import * as lambda_python from '@aws-cdk/aws-lambda-python-alpha';
import * as path from 'path';

export class LambdaAssetsStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // 1. TypeScript with NodejsFunction (esbuild)
    const apiHandler = new lambda_nodejs.NodejsFunction(this, 'ApiHandler', {
      entry: path.join(__dirname, '../src/api/handler.ts'),
      runtime: lambda.Runtime.NODEJS_22_X,
      architecture: lambda.Architecture.ARM_64,
      bundling: {
        minify: true,
        externalModules: ['@aws-sdk/*'],
      },
    });

    // 2. Python with dependencies (PythonFunction)
    const processor = new lambda_python.PythonFunction(this, 'Processor', {
      entry: path.join(__dirname, '../src/processor'),
      runtime: lambda.Runtime.PYTHON_3_12,
      architecture: lambda.Architecture.ARM_64,
    });

    // 3. Custom container (DockerImageFunction)
    const mlInference = new lambda.DockerImageFunction(this, 'MLInference', {
      code: lambda.DockerImageCode.fromImageAsset(
        path.join(__dirname, '../docker/ml-inference'),
        { platform: ecr_assets.Platform.LINUX_ARM64 }
      ),
      architecture: lambda.Architecture.ARM_64,
      memorySize: 3008,
      timeout: cdk.Duration.minutes(5),
    });

    // Outputs for inspection
    new cdk.CfnOutput(this, 'ApiHandlerArn',   { value: apiHandler.functionArn });
    new cdk.CfnOutput(this, 'ProcessorArn',    { value: processor.functionArn });
    new cdk.CfnOutput(this, 'MLInferenceArn',  { value: mlInference.functionArn });
  }
}
```

```bash
# Before deploy, see what will be built
cdk synth 2>&1 | grep -E "Bundling|Building|Asset"

# Deploy and monitor asset upload
cdk deploy --verbose 2>&1 | grep -E "upload|push|asset"

# View the final size of each function in Lambda
aws lambda list-functions \
  --query 'Functions[?starts_with(FunctionName, `LambdaAssets`)].{Name:FunctionName,Size:CodeSize}' \
  --output table
```

---

## Common pitfalls

**1. Bundling that works locally but fails in CI**

`NodejsFunction` uses esbuild locally but falls back to Docker when esbuild is not available. If your CI pipeline doesn't have Docker, and also doesn't have esbuild installed, synth fails. Solution: ensure `esbuild` is a dev dependency in the project's `package.json` (`npm install --save-dev esbuild`), so CI installs it along with other deps.

**2. Asset hash doesn't change when a dependency changes**

If you use `AssetHashType.SOURCE` (default), the hash is calculated on the **input** files — not on the bundling output. This means updating a dependency version in `requirements.txt` **without** changing the function code doesn't generate a new hash and CDK doesn't re-upload. Use `AssetHashType.OUTPUT` in `PythonFunction` so the hash reflects the bundling result.

**3. `nodeModules` vs `externalModules` in NodejsFunction**

```
externalModules: ['sharp']   → sharp is excluded from the bundle and NOT included in the zip
                               (assumes sharp is already available in the runtime — it's not)
                               → Lambda will fail at runtime with "Cannot find module 'sharp'"

nodeModules: ['sharp']       → sharp is excluded from esbuild but included in the zip as node_modules/
                               → pip/npm install happens inside the Docker container
                               → Correct binary for Amazon Linux
```

Use `externalModules` only for the AWS SDK and modules you are certain exist in the runtime. Use `nodeModules` for modules with native binaries that need to be compiled for Lambda.

---

## Reflection exercise

You are implementing an image processing service in Python that uses the `Pillow` library (with C extensions) and `numpy`. The function needs to run on ARM64 to reduce cost.

You have three options: `PythonFunction` with default bundling, `PythonFunction` with custom build image, or `DockerImageFunction` with your own Dockerfile.

For each option, describe: what happens during `cdk synth`, what is sent to AWS, the impact on Lambda cold start, and the CI/CD environment requirements. Which would you choose for production and why? Is there any scenario where the answer would change?

---

## Resources for deeper learning

**Assets:**
- [Assets and the AWS CDK](https://docs.aws.amazon.com/cdk/v2/guide/assets.html) — covers both types (file and Docker), how the hash is calculated, and how CDK decides when to re-upload.

**NodejsFunction:**
- [aws-cdk-lib.aws_lambda_nodejs module](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_lambda_nodejs-readme.html) — all bundling options with esbuild: `minify`, `sourceMap`, `externalModules`, `nodeModules`, `define`, `banner`.

**PythonFunction:**
- [@aws-cdk/aws-lambda-python-alpha module](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-lambda-python-alpha-readme.html) — supported dependency managers, bundling options, and how to use a custom build image.

---
