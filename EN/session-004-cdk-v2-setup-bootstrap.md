# Session 004 — CDK v2: setup, bootstrap, and project structure

**Estimated duration:** 60 minutes
**Prerequisites:** session-003 — CloudFormation: changesets, drift detection, and stack policies

---

## Objective

By the end, you will be able to initialize a CDK v2 project in TypeScript or Python, execute `cdk bootstrap` in an account/region, understand what bootstrap provisions (S3 bucket, ECR, IAM roles), and execute `cdk synth`, `cdk diff`, and `cdk deploy` on a simple stack.

---

## Context

[FACT] The AWS CDK (Cloud Development Kit) is an open-source framework released in 2019 that allows defining infrastructure in general-purpose programming languages (TypeScript, Python, Java, Go, C#). CDK does not replace CloudFormation — it generates CloudFormation as its output artifact. Every `cdk deploy` ends with a CloudFormation changeset being executed.

[CONSENSUS] The main advantage of CDK over raw CloudFormation is not just syntax — it's abstraction with escape hatch. You write less for common cases (L2 constructs handle permissions, logs, and safe defaults automatically), but you can always drop down to the CloudFormation level when you need full control. This is the central proposition of the L1/L2/L3 construct model, which session 005 covers in depth.

[FACT] CDK v2 was released in December 2021. The main difference from v1 is that all AWS service constructs were consolidated into a single package (`aws-cdk-lib`) instead of separate packages per service. CDK v1 reached end-of-support in June 2023. Always use v2.

---

## Core concepts

### 1. Mental model: CDK is an infrastructure compiler

CDK transforms code into CloudFormation. The complete flow is:

```
Your code (TypeScript/Python)
        │
        ▼ cdk synth
Cloud Assembly (cdk.out/)
  ├── MyStack.template.json    ← generated CloudFormation template
  ├── asset.XXXX/              ← local files to upload (Lambda code, etc.)
  └── manifest.json            ← assembly metadata

        │
        ▼ cdk deploy
  Upload assets → S3/ECR (bootstrap bucket)
  Creates/updates CloudFormation stack via changeset
        │
        ▼
  AWS resources created
```

When a deploy fails, the real error is in CloudFormation — not in CDK. That's why knowing CloudFormation (sessions 002-003) is an essential prerequisite.

---

### 2. Installation and prerequisites

[FACT] The CDK CLI is an npm package. The minimum runtime is Node.js 18.x (even if you write in Python or Go — the CLI is always Node.js).

```bash
# Check Node version
node --version   # needs to be >= 18.x

# Install the CDK CLI globally
npm install -g aws-cdk

# Check installed version
cdk --version    # e.g., 2.176.0 (build xxxxxxx)

# For Python: install the lib in the project (not system-wide)
pip install aws-cdk-lib constructs
```

[CONSENSUS] Installing the CDK CLI globally (`-g`) is convenient for local use, but in CI/CD pipelines prefer installing locally in the project (`npm install aws-cdk`) to pin the version. The CLI version and the `aws-cdk-lib` version in `package.json` should be compatible — version differences between CLI and lib can cause warnings or synth errors.

---

### 3. CDK v2 project structure

**Initializing a new project:**

```bash
mkdir my-project && cd my-project

# TypeScript (most common in official examples)
cdk init app --language typescript

# Python
cdk init app --language python
```

**Generated structure for TypeScript:**

```
my-project/
├── bin/
│   └── my-project.ts        ← Entry point: instantiates the App and Stacks
├── lib/
│   └── my-project-stack.ts  ← Construct definitions (where you work)
├── test/
│   └── my-project.test.ts   ← Tests with assertions (session 008)
├── cdk.json                  ← CDK CLI configuration
├── package.json
├── tsconfig.json
└── node_modules/
```

**Generated structure for Python:**

```
my-project/
├── app.py                    ← Entry point
├── my_project/
│   ├── __init__.py
│   └── my_project_stack.py  ← Construct definitions
├── tests/
├── cdk.json
├── requirements.txt
└── .venv/                    ← virtualenv (create manually: python -m venv .venv)
```

**The entry point (bin/my-project.ts):**

```typescript
import * as cdk from 'aws-cdk-lib';
import { MyProjectStack } from '../lib/my-project-stack';

const app = new cdk.App();

new MyProjectStack(app, 'MyProjectStack', {
  env: {
    account: process.env.CDK_DEFAULT_ACCOUNT,
    region: process.env.CDK_DEFAULT_REGION,
  },
});
```

[FACT] `CDK_DEFAULT_ACCOUNT` and `CDK_DEFAULT_REGION` are environment variables automatically populated by the CDK CLI based on the active AWS profile. Hardcoding account/region is possible but not recommended — it prevents reusing the same code across multiple accounts.

**The stack (lib/my-project-stack.ts):**

```typescript
import * as cdk from 'aws-cdk-lib';
import * as s3 from 'aws-cdk-lib/aws-s3';
import { Construct } from 'constructs';

export class MyProjectStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // L2 Construct — high abstraction level
    new s3.Bucket(this, 'MyBucket', {
      versioned: true,
      encryption: s3.BucketEncryption.S3_MANAGED,
      removalPolicy: cdk.RemovalPolicy.DESTROY,   // equivalent to DeletionPolicy
      autoDeleteObjects: true,
    });
  }
}
```

**Construct hierarchy:**

```
App                           ← root of everything
  └── Stack (MyProjectStack)
        └── Bucket (MyBucket)
              └── BucketPolicy (automatically generated by L2)
```

Each node in the tree has a unique ID within its parent. CDK uses this path to generate logical IDs in CloudFormation: `MyProjectStackMyBucketXXXXXXXX`.

---

### 4. Bootstrap — what it is and what it provisions

**Why bootstrap is necessary:**

When CDK deploys assets (Lambda code, Docker images, local files), it needs a place to store them before passing to CloudFormation. Bootstrap creates this supporting infrastructure.

[FACT] `cdk bootstrap` deploys a CloudFormation stack called `CDKToolkit` in the account/region. This stack provisions:

```
CDKToolkit stack
├── S3 Bucket (cdk-xxxxxxxx-assets-ACCOUNT-REGION)
│     └── Stores synthesized templates and local assets
│         Versioned with 365-day lifecycle policy
│
├── ECR Repository (cdk-xxxxxxxx-container-assets-ACCOUNT-REGION)
│     └── Stores Docker images for Lambda/ECS
│
└── IAM Roles (6 roles created by default)
      ├── cdk-xxxxxxxx-lookup-role-ACCOUNT-REGION
      │     └── Allows CDK CLI to perform lookups (VPCs, AMIs, etc.)
      ├── cdk-xxxxxxxx-file-publishing-role-ACCOUNT-REGION
      │     └── Upload assets to S3
      ├── cdk-xxxxxxxx-image-publishing-role-ACCOUNT-REGION
      │     └── Push images to ECR
      ├── cdk-xxxxxxxx-deploy-role-ACCOUNT-REGION
      │     └── Create/update CloudFormation stacks
      ├── cdk-xxxxxxxx-cfn-exec-role-ACCOUNT-REGION
      │     └── Role that CloudFormation assumes to create resources
      └── cdk-xxxxxxxx-readonly-role-ACCOUNT-REGION (recent v2)
            └── Read permissions for diff and synth
```

**Running bootstrap:**

```bash
# Bootstrap in the current account/region (uses the default profile)
cdk bootstrap

# Bootstrap explicitly specifying account and region
cdk bootstrap aws://123456789012/us-east-1

# Bootstrap with a specific profile
cdk bootstrap --profile prod aws://123456789012/us-east-1

# Bootstrap multi-region (run once per region)
cdk bootstrap aws://123456789012/us-east-1
cdk bootstrap aws://123456789012/sa-east-1

# See what would be created without executing
cdk bootstrap --show-template
```

**Bootstrap qualifier:**

[FACT] The qualifier is a 9-character identifier (default: `hnb659fds`) that is part of the bootstrap resource names. It allows having multiple bootstraps in the same account/region — useful when different teams need asset isolation.

```bash
# Bootstrap with custom qualifier
cdk bootstrap --qualifier myteam

# In cdk.json, reference the qualifier
{
  "@aws-cdk/core:bootstrapQualifier": "myteam"
}
```

**When to re-run bootstrap:**

Bootstrap has a version number. When updating the CDK CLI to a major version, it may be necessary to update the bootstrap. CDK warns during `cdk deploy` if the bootstrap version is outdated:

```
❌ This CDK deployment requires bootstrap stack version '6', found '4'.
   Please run 'cdk bootstrap'.
```

Running `cdk bootstrap` on an already-bootstrapped account/region is safe — it updates existing resources without recreating them.

---

### 5. Main CDK CLI commands

**`cdk synth` — compile to CloudFormation:**

```bash
# Synthesize all stacks
cdk synth

# Synthesize a specific stack
cdk synth MyProjectStack

# View the generated template in terminal (without saving to cdk.out/)
cdk synth --quiet

# Output in cdk.out/ by default
ls cdk.out/
# MyProjectStack.template.json
# manifest.json
# tree.json
```

`cdk synth` is the equivalent of "compiling" — transforms code into CloudFormation. It detects configuration errors before any AWS call.

**`cdk diff` — see what will change:**

```bash
# Diff between local code and deployed stack
cdk diff

# Diff of a specific stack
cdk diff MyProjectStack

# Diff against a saved template (without deployed stack)
cdk diff --template cdk.out/MyProjectStack.template.json
```

`cdk diff` is the equivalent of the changeset from the previous session — but computed locally, without creating anything in AWS. It's faster than a real changeset, but less precise for resources with values resolved at runtime.

**`cdk deploy` — deploy:**

```bash
# Deploy with interactive approval of IAM changes
cdk deploy

# Deploy without approval (useful in pipelines)
cdk deploy --require-approval never

# Deploy multiple stacks
cdk deploy Stack1 Stack2

# Deploy all stacks
cdk deploy "*"

# Deploy with hotswap (for development — skips CloudFormation for Lambda changes)
cdk deploy --hotswap

# Deploy and show stack outputs
cdk deploy --outputs-file outputs.json
```

[FACT] `cdk deploy --hotswap` detects changes that are only in Lambda code or ECS definitions and applies them directly (without CloudFormation), reducing feedback time from ~2 minutes to ~10 seconds. **Never use hotswap in production** — it can leave CloudFormation state out of sync with actual state.

**`cdk destroy` — destroy the stack:**

```bash
cdk destroy MyProjectStack
# Asks for interactive confirmation

cdk destroy --force MyProjectStack
# Without confirmation
```

**Command flow diagram:**

```
CDK Code
    │
    ├─ cdk synth ──► cdk.out/ (CloudFormation templates + assets)
    │
    ├─ cdk diff  ──► Local comparison vs. deployed stack (no AWS calls)
    │
    ├─ cdk deploy
    │       ├── cdk synth (implicit)
    │       ├── Upload assets → S3/ECR (via bootstrap roles)
    │       ├── Creates CloudFormation changeset
    │       ├── Shows IAM changes for approval (if any)
    │       └── Executes changeset → waits for completion
    │
    └─ cdk destroy
            ├── Creates deletion changeset
            └── Executes → deletes all resources (respecting DeletionPolicy)
```

---

### 6. cdk.json — project configuration

`cdk.json` is the CDK CLI configuration file for the project. Every `cdk` command executed in the folder reads this file.

```json
{
  "app": "npx ts-node --prefer-ts-exts bin/my-project.ts",
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

**Main fields:**

```
app       → command to execute the entry point (compiler/runtime)
watch     → files to monitor for cdk watch (hot reload in dev)
context   → feature flags and persisted context values
```

[FACT] Values in `context` that start with `@aws-cdk/` are feature flags. They control behaviors that changed between CDK versions and allow gradual migration. New projects created with `cdk init` automatically receive the set of feature flags from the current version.

**What goes in gitignore:**

```
cdk.out/              ← always ignore (build artifact)
cdk.context.json      ← DEPENDS (see session 012 for the full discussion)
node_modules/         ← always ignore
.venv/                ← always ignore (Python)
```

---

## Practical example

Create a CDK project from scratch and do the first deploy:

```bash
# 1. Create and initialize the project
mkdir demo-cdk && cd demo-cdk
cdk init app --language typescript
npm install

# 2. Bootstrap (once per account/region)
cdk bootstrap --profile dev

# 3. Edit lib/demo-cdk-stack.ts
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

    // L2 handles permissions — generates the correct policy automatically
    bucket.grantReadWrite(appRole);

    // Outputs
    new cdk.CfnOutput(this, 'BucketName', { value: bucket.bucketName });
    new cdk.CfnOutput(this, 'AppRoleArn', { value: appRole.roleArn });
  }
}
EOF

# 4. Synthesize — see the generated CloudFormation
cdk synth

# 5. See what will be created (diff against nonexistent stack)
cdk diff --profile dev

# 6. Deploy
cdk deploy --profile dev

# 7. View the generated CloudFormation template (to compare with what you'd write)
cat cdk.out/DemoCdkStack.template.json | jq '.Resources | keys'

# 8. Compare what the L2 generated vs what you'd write manually
# The bucket.grantReadWrite(appRole) automatically generates an IAM Policy
# with the correct permissions — without needing to write the bucket ARN
cat cdk.out/DemoCdkStack.template.json | jq '.Resources | to_entries[] | select(.value.Type == "AWS::IAM::Policy")'
```

---

## Common pitfalls

**1. Forgetting to re-run bootstrap after CDK update**

After `npm install -g aws-cdk@latest`, if the new version requires a newer bootstrap version, the next `cdk deploy` fails with the incompatible version message. The solution is simple (`cdk bootstrap`), but it confuses those who don't know why it's failing.

**2. `RemovalPolicy.DESTROY` in production**

CDK uses `RemovalPolicy.RETAIN` as the default for most stateful resources (S3, RDS, DynamoDB). Changing to `DESTROY` makes `cdk destroy` delete the resource and its data. In projects created from tutorials, it's common to forget this value copied from development examples when promoting to production.

**3. Unstable logical ID — the rename problem**

CDK derives the CloudFormation logical ID from the construct path in the tree (`Stack > Construct > Resource`). If you rename a construct or move a resource in the hierarchy, the logical ID changes — and CloudFormation interprets this as "delete the old one and create a new one". For stateful resources (RDS, S3 with data), this is destructive. The solution is to use `overrideLogicalId()` to pin the ID when necessary.

```typescript
// Pin the logical ID to avoid accidental replacement when refactoring
const cfnBucket = bucket.node.defaultChild as s3.CfnBucket;
cfnBucket.overrideLogicalId('MyStableBucket');
```

---

## Reflection exercise

You are migrating an existing CloudFormation stack to CDK. The original template has an S3 bucket with logical ID `AppBucket`. When writing the CDK code, you create the bucket like this:

```typescript
new s3.Bucket(this, 'AppBucket', { versioned: true });
```

CDK will generate a different logical ID from the original (something like `AppBucketXXXXXXXX` with a hash). If you run `cdk deploy` on top of the existing stack, CloudFormation will try to create a new bucket and delete the original.

What are the two approaches to solve this problem? For each one, describe the concrete steps and trade-offs. Also consider: why does CDK add this hash to the logical ID and what problem does it solve in the normal life of a CDK project.

---

## Resources for deeper learning

**Setup and first project:**
- [Getting started with the AWS CDK](https://docs.aws.amazon.com/cdk/v2/guide/getting_started.html) — prerequisites, installation, and the `hello-world` tutorial with TypeScript and Python.

**Bootstrap in depth:**
- [AWS CDK bootstrapping](https://docs.aws.amazon.com/cdk/v2/guide/bootstrapping.html) — lists all created resources, bootstrap versions, and when to re-run.

**CLI reference:**
- [AWS CDK CLI reference](https://docs.aws.amazon.com/cdk/v2/guide/cli.html) — all commands with flags, including `--hotswap`, `--watch`, `--outputs-file`, and approval options.

**CDK Workshop (hands-on):**
- [CDK Workshop](https://cdkworkshop.com) — hands-on tutorial with TypeScript or Python covering App → Stack → Constructs → Pipeline in ~2 hours. Recommended to solidify this session in practice.

---
