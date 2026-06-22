# Session 006 — CDK: Stacks, environments, and multi-account patterns

**Estimated duration:** 60 minutes
**Prerequisites:** session-005 — CDK: Constructs L1, L2, L3

---

## Objective

By the end, you will be able to create multiple stacks in a CDK App with distinct environments (dev/staging/prod in different accounts), use `Stack.account` and `Stack.region` without hardcoding, and understand when to use one stack per account vs nested stacks.

---

## Context

[CONSENSUS] The multi-account structure is the AWS-recommended pattern for mature organizations — one account per environment (dev, staging, prod), ideally under an AWS Organization. CDK has native support for this model via `environments` and `Stages`. Understanding how CDK resolves account and region is critical to avoid two opposite mistakes: hardcoding values that break in other accounts, and "environment-agnostic" stacks that lose important features.

[FACT] In CDK, an **environment** is simply the pair `{ account: string, region: string }`. It is the deploy target of a stack. A stack without an explicit environment is called "environment-agnostic" and has important limitations.

---

## Core concepts

### 1. Environments — account + region without hardcoding

Every stack can (and in production, should) have an associated environment:

```typescript
// bin/my-app.ts

const app = new cdk.App();

// ❌ Hardcode — don't do this
new MyStack(app, 'MyStack', {
  env: { account: '123456789012', region: 'us-east-1' },
});

// ✅ Dynamic environment — uses the profile/credentials active at synth time
new MyStack(app, 'MyStack', {
  env: {
    account: process.env.CDK_DEFAULT_ACCOUNT,
    region:  process.env.CDK_DEFAULT_REGION,
  },
});

// ✅ Explicit multi-account — the most correct for pipelines
new MyStack(app, 'DevStack', {
  env: { account: '111122223333', region: 'us-east-1' },
});

new MyStack(app, 'ProdStack', {
  env: { account: '444455556666', region: 'us-east-1' },
});
```

**`CDK_DEFAULT_ACCOUNT` vs `CDK_DEFAULT_REGION`:**

[FACT] The CDK CLI automatically populates `CDK_DEFAULT_ACCOUNT` and `CDK_DEFAULT_REGION` based on the active AWS credentials (SSO profile, environment variables, instance profile — the same chain as the CLI). If you use `--profile prod`, CDK resolves these variables to the account associated with the `prod` profile.

```bash
# Deploy to dev with dev profile
cdk deploy --profile dev

# Deploy to prod with prod profile
cdk deploy --profile prod

# The same code, different environments — with no code changes
```

---

### 2. Environment-agnostic stacks — when and why to avoid

A stack without `env` is synthesized without a specific account/region. The resulting template uses CloudFormation pseudo-parameters (`AWS::AccountId`, `AWS::Region`).

```typescript
// Stack without environment — environment-agnostic
new MyStack(app, 'MyStack');
// Generates: { "Account": { "Ref": "AWS::AccountId" } } in the template
```

**What you lose with environment-agnostic:**

```
❌ Context lookups don't work
   ec2.Vpc.fromLookup(...)        → fails: needs to know the account/region to query
   HostedZone.fromLookup(...)     → same

❌ Service Principals may be incorrect
   In opt-in regions (e.g., ap-east-1) the service principal format is different
   CDK resolves this automatically when the environment is pinned

❌ Some CDK validations are disabled
   Checks that depend on account data (limits, AMIs, etc.)

✅ What still works
   Deploy to any account/region via cdk deploy --profile X
   Ideal for reusable constructs published as libraries
```

[FACT] For application stacks in production, always pin the environment. For reusable constructs published as NPM packages, keep environment-agnostic — whoever instantiates defines the environment.

---

### 3. `Stack.account` and `Stack.region` — references without hardcoding inside the stack

Inside a stack or construct, you access account and region via `this.account` and `this.region` (or `Stack.of(this).account`):

```typescript
export class MyStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props: cdk.StackProps) {
    super(scope, id, props);

    // ✅ Resolves at synth time if the environment is pinned
    const bucketName = `my-app-${this.account}-${this.region}`;

    // ✅ Using Sub to build ARNs without hardcoding
    const paramPath = `/my-app/${this.stackName}/config`;

    // ✅ Stack.of() — accesses the stack from any child construct
    const account = cdk.Stack.of(this).account;
    const region  = cdk.Stack.of(this).region;

    // ✅ Tokens for values resolved at deploy time (not at synth)
    // When the environment is NOT pinned, this.account returns a Token
    // (string like "${Token[AWS.AccountId.0]}") — you can't use it in conditional logic
    const isResolved = !cdk.Token.isUnresolved(this.account);
  }
}
```

**Token vs concrete value — frequent pitfall:**

```typescript
// ❌ Doesn't work when environment-agnostic
// this.account returns a Token, not '123456789012'
if (this.account === '123456789012') {  // always false with Token
  // account-specific logic
}

// ✅ Use custom props for conditional logic
interface MyStackProps extends cdk.StackProps {
  isProd: boolean;
}

export class MyStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props: MyStackProps) {
    super(scope, id, props);

    const removalPolicy = props.isProd
      ? cdk.RemovalPolicy.RETAIN
      : cdk.RemovalPolicy.DESTROY;
  }
}
```

---

### 4. Stage — grouping stacks for multi-environment

`Stage` is CDK's concept for grouping stacks that represent the same system in a specific environment. It is the unit of promotion in pipelines (dev → staging → prod).

```typescript
// lib/application-stage.ts
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import { InfraStack } from './infra-stack';
import { AppStack } from './app-stack';

export class ApplicationStage extends cdk.Stage {
  constructor(scope: Construct, id: string, props: cdk.StageProps) {
    super(scope, id, props);

    // All stacks of this system, together
    const infra = new InfraStack(this, 'Infra');
    const app   = new AppStack(this, 'App', {
      vpc: infra.vpc,
      cluster: infra.cluster,
    });
  }
}
```

```typescript
// bin/my-app.ts
const app = new cdk.App();

// Dev stage — development account
new ApplicationStage(app, 'Dev', {
  env: { account: '111122223333', region: 'us-east-1' },
});

// Staging stage — separate account
new ApplicationStage(app, 'Staging', {
  env: { account: '222233334444', region: 'us-east-1' },
});

// Prod stage — production account
new ApplicationStage(app, 'Prod', {
  env: { account: '333344445555', region: 'us-east-1' },
});
```

**What Stage generates:**

```
cdk ls
  Dev/Infra       → stack DevInfra in account 111122223333
  Dev/App         → stack DevApp in account 111122223333
  Staging/Infra   → stack StagingInfra in account 222233334444
  Staging/App     → stack StagingApp in account 222233334444
  Prod/Infra      → stack ProdInfra in account 333344445555
  Prod/App        → stack ProdApp in account 333344445555
```

**Deploying a complete Stage:**

```bash
# Deploy all stacks in the Dev stage
cdk deploy "Dev/*" --profile dev

# Deploy only a specific stack from the stage
cdk deploy Dev/Infra --profile dev

# Diff of the entire Prod stage
cdk diff "Prod/*" --profile prod
```

---

### 5. When to use stack per account vs nested stacks

This is an architectural decision with real trade-offs.

**Stack per account (recommended pattern):**

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

Advantages:
- Real isolation between environments (separate accounts = IAM boundaries, billing, CloudTrail)
- Failures in dev don't affect prod
- Model aligned with AWS Organizations and SCPs

**Multiple stacks in the same account (by lifecycle):**

```
App (same dev account)
├── NetworkStack     → VPC, subnets (changes rarely)
├── DataStack        → RDS, DynamoDB (changes occasionally)
└── AppStack         → Lambda, ECS, API GW (changes frequently)
```

Advantages:
- Partial deploy — only the AppStack changes day-to-day
- Reduces risk: a change in app code doesn't touch network infrastructure
- CloudFormation has a 500 resource limit per stack — splitting resolves this

[CONSENSUS] The primary criterion for splitting into stacks is not size — it's **lifecycle and ownership**. Resources that change together belong in the same stack. Resources with different owners belong in different stacks. Resources that should never be affected by an application deploy belong in a separate stack.

**Stack per account vs nested stacks:**

[FACT] Nested stacks (via `NestedStack`) are CloudFormation child stacks of another stack. They are useful for overcoming the 500 resource limit, but create coupling — the parent stack is responsible for the lifecycle of children. In CDK, the recommended alternative is to use multiple independent stacks (with cross-stack references) instead of nested stacks.

```typescript
// ❌ Nested stack — avoid unless you need to overcome resource limits
import { NestedStack } from 'aws-cdk-lib';
class MyNestedStack extends NestedStack { ... }

// ✅ Independent stacks with cross-stack reference
class InfraStack extends cdk.Stack {
  public readonly vpc: ec2.Vpc;  // exposes the resource
  constructor(...) {
    this.vpc = new ec2.Vpc(this, 'Vpc');
  }
}

class AppStack extends cdk.Stack {
  constructor(scope, id, props: { vpc: ec2.Vpc } & cdk.StackProps) {
    // consumes the resource from another stack
    new ecs.Cluster(this, 'Cluster', { vpc: props.vpc });
  }
}
```

---

### 6. Cross-stack references in CDK

When a stack references a resource from another stack in CDK, the framework automatically creates a CloudFormation Export/Import underneath — but you don't need to manage this manually.

```typescript
// bin/my-app.ts
const app = new cdk.App();

const infra = new InfraStack(app, 'InfraStack', {
  env: { account: '111122223333', region: 'us-east-1' },
});

// Pass the object directly — CDK manages Export/Import automatically
const appStack = new AppStack(app, 'AppStack', {
  env: { account: '111122223333', region: 'us-east-1' },
  vpc: infra.vpc,              // direct reference to the object
  cluster: infra.cluster,
});
```

**What CDK generates underneath:**

```
InfraStack Outputs:
  ExportsOutputRefVpcXXXXXX: vpc-0abc123   (Export: InfraStack:ExportsOutputRefVpcXXXXXX)

AppStack Resources:
  Cluster:
    Properties:
      VpcId: !ImportValue InfraStack:ExportsOutputRefVpcXXXXXX
```

[FACT] Cross-stack references in CDK **only work within the same account and region**. To reference resources cross-account, you need to use SSM Parameter Store, Secrets Manager, or pass values as concrete props (string/ARN) instead of CDK objects.

**Cross-account — correct pattern:**

```typescript
// InfraStack (account A) — exports via SSM
const vpc = new ec2.Vpc(this, 'Vpc');
new ssm.StringParameter(this, 'VpcIdParam', {
  parameterName: '/shared/vpc-id',
  stringValue: vpc.vpcId,
});

// AppStack (account B) — imports via lookup
const vpcId = ssm.StringParameter.valueForStringParameter(this, '/shared/vpc-id');
const vpc = ec2.Vpc.fromLookup(this, 'SharedVpc', { vpcId });
```

---

### 7. Architectural diagram — complete multi-account CDK App

```
CDK App (single repository)
│
├── bin/app.ts
│     ├── new ApplicationStage(app, 'Dev',     { env: devEnv })
│     ├── new ApplicationStage(app, 'Staging', { env: stagingEnv })
│     └── new ApplicationStage(app, 'Prod',    { env: prodEnv })
│
├── lib/
│     ├── application-stage.ts    ← Stage: groups the stacks
│     ├── network-stack.ts        ← VPC, subnets (changes rarely)
│     ├── data-stack.ts           ← RDS, DynamoDB (changes occasionally)
│     └── app-stack.ts            ← ECS, Lambda (changes frequently)
│
└── Bootstrap required in each account/region:
      cdk bootstrap aws://111122223333/us-east-1  (dev)
      cdk bootstrap aws://222233334444/us-east-1  (staging)
      cdk bootstrap aws://333344445555/us-east-1  (prod)

Deploy:
  cdk deploy "Dev/*"     --profile dev
  cdk deploy "Staging/*" --profile staging
  cdk deploy "Prod/*"    --profile prod
```

---

## Practical example

Complete CDK App with three stacks and two environments:

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

    // Exports the VPC ID via SSM for cross-account consumption
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
# List all generated stacks
cdk ls
# Dev/Network
# Dev/App
# Prod/Network
# Prod/App

# Deploy dev (bootstrap already done)
cdk deploy "Dev/*" --profile dev --require-approval never

# Diff prod before promoting
cdk diff "Prod/*" --profile prod

# Deploy prod with IAM change approval
cdk deploy "Prod/*" --profile prod
```

---

## Common pitfalls

**1. `Token.isUnresolved` — using `this.account` in conditional logic**

`this.account` returns a Token when the environment is not pinned, or during certain synth phases. Using it in comparisons (`if (this.account === '123...')`) always silently fails. Use typed props for per-environment conditional configurations — never compare `this.account` directly in business logic.

**2. Cross-stack reference between accounts — CDK object doesn't cross accounts**

Passing a CDK object (`vpc`, `cluster`, `bucket`) as a prop works only within the same account and region. CDK creates a CloudFormation Export/Import, which is regional. For cross-account, you need to pass strings (IDs, ARNs) and use `from*` methods to import the resource (`ec2.Vpc.fromVpcAttributes`, `s3.Bucket.fromBucketName`, etc.).

**3. Stage with `env` vs stacks with `env` — which takes precedence**

The Stage's `env` is inherited by all child stacks that don't define their own `env`. If a child stack defines a different `env`, the stack goes to a different account/region than the Stage — which can be intentional but is frequently a bug. Maintain consistency: either the Stage defines the `env` and stacks inherit, or each stack defines its own.

---

## Reflection exercise

You are designing the CDK structure for a platform with the following components: shared VPC (used by all services), RDS database (updated rarely, with critical data), API service (Lambda + API Gateway, deployed multiple times a day), and monitoring dashboard (CloudWatch dashboards + alarms).

You have three environments: dev (single account), staging (single account), and prod (single account, with multiple regions: us-east-1 and eu-west-1).

Design the stack and stage structure for this platform. For each decision — how many stacks, how to group them, how to pass references between them — justify based on lifecycle, blast radius risk, and operational requirements. What changes in the structure for multi-region prod?

---

## Resources for deeper learning

**Environments:**
- [Environments for the AWS CDK](https://docs.aws.amazon.com/cdk/v2/guide/environments.html) — covers environment-agnostic vs environment-bound, CDK_DEFAULT_ACCOUNT/REGION, and how the CLI resolves the variables.

**Stages:**
- [Introduction to AWS CDK stages](https://docs.aws.amazon.com/cdk/v2/guide/stages.html) — how to create and use Stages to group stacks by environment.

**Best practices:**
- [Best practices for developing and deploying cloud infrastructure with the AWS CDK](https://docs.aws.amazon.com/cdk/v2/guide/best-practices.html) — the "Constructs vs Stacks" section explains the lifecycle criterion for separating stacks, and "Environments" covers the recommended multi-account structure.

---
