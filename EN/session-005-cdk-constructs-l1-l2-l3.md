# Session 005 — CDK: Constructs L1, L2, L3 — what they are and how to choose

**Estimated duration:** 60 minutes
**Prerequisites:** session-004 — CDK v2: setup, bootstrap, and project structure

---

## Objective

By the end, you will be able to distinguish L1 constructs (`CfnBucket`), L2 (`Bucket`), and L3 (patterns like `ApplicationLoadBalancedFargateService`), navigate the Construct Hub to evaluate third-party constructs, and explain why L2 is not always preferable to L1.

---

## Context

[FACT] The three-layer abstraction model (L1/L2/L3) is the core of CDK's value proposition. It defines how the construct library is organized and determines the level of control vs. convenience in each situation.

[CONSENSUS] Most engineers who start with CDK adopt L2 by default and only drop to L1 when something doesn't work. This approach works, but it leaves gaps: you end up not understanding what the L2 generates underneath, which makes debugging difficult and escape hatches mysterious. Understanding the three levels — and when each is appropriate — is what separates superficial from competent CDK usage.

---

## Core concepts

### 1. The abstraction hierarchy

```
L3 — Patterns (high abstraction)
  Composition of multiple pre-configured resources
  E.g., ApplicationLoadBalancedFargateService
       └── creates: ECS Cluster + Fargate Service + ALB + Target Group
                + Listener + Security Groups + IAM Roles + Log Group

L2 — Intent-Based (medium abstraction)
  A single AWS resource with safe defaults and helpers
  E.g., s3.Bucket
       └── encapsulates: AWS::S3::Bucket (L1)
       └── automatically generates: BucketPolicy, AutoDeleteObjects Lambda
       └── exposes: bucket.grantRead(), bucket.grantReadWrite(), etc.

L1 — CloudFormation Layer (no abstraction)
  Direct mapping of the CloudFormation spec
  E.g., s3.CfnBucket
       └── corresponds exactly to: AWS::S3::Bucket in YAML
       └── every CloudFormation property available
       └── no defaults, no helpers
```

---

### 2. L1 — CloudFormation Constructs (`Cfn` prefix)

[FACT] L1 constructs are **automatically generated** from the CloudFormation specification. If a resource exists in CloudFormation, there is a corresponding L1 in CDK — with the `Cfn` prefix.

```typescript
import * as s3 from 'aws-cdk-lib/aws-s3';

// L1: byte-for-byte equivalent to CloudFormation YAML
const bucket = new s3.CfnBucket(this, 'MyBucket', {
  bucketName: 'my-bucket-prod',
  versioningConfiguration: {
    status: 'Enabled',
  },
  bucketEncryption: {
    serverSideEncryptionConfiguration: [
      {
        serverSideEncryptionByDefault: {
          sseAlgorithm: 'AES256',
        },
      },
    ],
  },
  publicAccessBlockConfiguration: {
    blockPublicAcls: true,
    blockPublicPolicy: true,
    ignorePublicAcls: true,
    restrictPublicBuckets: true,
  },
});
```

**What L1 generates in CloudFormation:**

```json
{
  "MyBucket": {
    "Type": "AWS::S3::Bucket",
    "Properties": {
      "BucketName": "my-bucket-prod",
      "VersioningConfiguration": { "Status": "Enabled" },
      "BucketEncryption": { ... },
      "PublicAccessBlockConfiguration": { ... }
    }
  }
}
```

Exactly what you would write manually. Nothing more, nothing less.

**L1 characteristics:**
- Properties in `camelCase` (CDK) vs `PascalCase` (CloudFormation) — automatic conversion
- Strict types — TypeScript detects incorrect properties at compile time
- 100% coverage of the CloudFormation spec — if CloudFormation supports it, L1 supports it
- No defaults — you are responsible for every property

---

### 3. L2 — Intent-based Constructs

L2 is where CDK delivers its greatest value. Beyond encapsulating L1, it adds:

**a) Safe and opinionated defaults:**

```typescript
import * as s3 from 'aws-cdk-lib/aws-s3';

// L2: much less code, sensible defaults
const bucket = new s3.Bucket(this, 'MyBucket', {
  versioned: true,
  encryption: s3.BucketEncryption.S3_MANAGED,
  blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
  removalPolicy: cdk.RemovalPolicy.RETAIN,  // default is already RETAIN for S3
});
// The L2 automatically applies: complete block public access by default (v2)
```

**b) Helper methods (the biggest differentiator):**

```typescript
const bucket = new s3.Bucket(this, 'AppBucket');
const fn = new lambda.Function(this, 'Processor', { ... });
const role = new iam.Role(this, 'AppRole', { ... });

// Without L2 (pure L1): you would write the IAM Policy manually with the correct ARN
// With L2: the construct knows which permissions are needed
bucket.grantRead(fn);           // s3:GetObject + s3:ListBucket on fn's policy
bucket.grantReadWrite(role);    // s3:GetObject + s3:PutObject + s3:DeleteObject + s3:ListBucket
bucket.grantPut(fn);            // only s3:PutObject

// Other examples of helpers:
queue.grantSendMessages(fn);
table.grantReadData(fn);
topic.grantPublish(role);
fn.grantInvoke(role);
```

**c) Integrations between resources:**

```typescript
import * as events from 'aws-cdk-lib/aws-events';
import * as targets from 'aws-cdk-lib/aws-events-targets';

const rule = new events.Rule(this, 'DailyRule', {
  schedule: events.Schedule.cron({ hour: '8', minute: '0' }),
});

// Adds Lambda as target: creates the permission and target automatically
rule.addTarget(new targets.LambdaFunction(fn));
```

**What L2 generates that you don't directly see:**

```typescript
// When you write:
new s3.Bucket(this, 'MyBucket', {
  removalPolicy: cdk.RemovalPolicy.DESTROY,
  autoDeleteObjects: true,      // ← this line
});

// CDK generates in CloudFormation:
// 1. AWS::S3::Bucket (the bucket itself)
// 2. AWS::IAM::Role (for the auto-delete Lambda)
// 3. AWS::Lambda::Function (the Lambda that deletes objects)
// 4. AWS::Lambda::Permission (permission for CloudFormation to invoke)
// 5. AWS::CloudFormation::CustomResource (that triggers the Lambda on delete)
// 6. AWS::S3::BucketPolicy (if you use grants)
```

[CONSENSUS] This "side effect" of L2 is the main reason why L2 is not always the best choice. Sometimes you don't want those extra resources — especially in environments with restrictive security policies about automatically generated Lambdas and IAM roles.

---

### 4. When L2 is not preferable to L1

This is the counterintuitive part of the session. L2 is not always better.

**Case 1: The property you need is not exposed by L2**

L2 constructs are maintained by the AWS team and don't always keep up with the speed of new CloudFormation resource launches. When CloudFormation launches a new property, L1 exposes it immediately (generated from the spec), but L2 may take weeks or months to expose it via a method.

```typescript
// Hypothetical example: new property launched yesterday in CloudFormation
// The L2 doesn't have a prop for this yet

// Option A: use L1 directly
const bucket = new s3.CfnBucket(this, 'Bucket', {
  newProperty: 'value',    // available in L1 immediately
});

// Option B: use L2 + escape hatch (see section 6)
const bucket = new s3.Bucket(this, 'Bucket');
const cfnBucket = bucket.node.defaultChild as s3.CfnBucket;
cfnBucket.addPropertyOverride('NewProperty', 'value');
```

**Case 2: The extra L2 resources are unwanted**

```typescript
// autoDeleteObjects generates a Lambda + Role + CustomResource
// In environments with SCPs that block Lambda creation in certain regions,
// or with mandatory review of IAM roles, this is an operational problem

// Pure L1: just the bucket, nothing more
const bucket = new s3.CfnBucket(this, 'Bucket', {
  bucketName: 'my-bucket',
  // no extra resources
});
```

**Case 3: Migrating from existing CloudFormation**

When importing an existing resource (via `cloudformation import`), you need the CDK logical ID to correspond exactly to the original template. L1 with `overrideLogicalId` gives full control; L2 with hash may require more work.

**Case 4: Template auditing and compliance**

In organizations with security review of the CloudFormation template before deploy, L2 generates "implicit" resources that appear in the template and need to be justified. L1 produces predictable and auditable templates.

**Practical rule:**

```
Prefer L2 when:
  → You want development speed
  → The defaults are adequate for your case
  → The permission helpers (grant*) greatly simplify the code
  → You are in the prototyping phase

Prefer L1 when:
  → You need a property that L2 doesn't expose yet
  → The extra resources generated by L2 are unwanted
  → You need fixed and predictable logical IDs
  → The final template needs to be auditable without surprises
```

---

### 5. L3 — Patterns

L3 constructs compose multiple L2s into a reusable architectural pattern. The canonical example is `ApplicationLoadBalancedFargateService`:

```typescript
import * as ecs_patterns from 'aws-cdk-lib/aws-ecs-patterns';
import * as ecs from 'aws-cdk-lib/aws-ecs';

const cluster = new ecs.Cluster(this, 'Cluster', { vpc });

// One line replaces ~150 lines of CloudFormation
const service = new ecs_patterns.ApplicationLoadBalancedFargateService(this, 'Service', {
  cluster,
  cpu: 256,
  memoryLimitMiB: 512,
  desiredCount: 2,
  taskImageOptions: {
    image: ecs.ContainerImage.fromRegistry('nginx:latest'),
    containerPort: 80,
  },
  publicLoadBalancer: true,
});

// The L3 created underneath:
// - ECS Cluster (if not provided)
// - Task Definition + Container Definition
// - Fargate Service
// - Application Load Balancer
// - ALB Listener (port 80/443)
// - Target Group with health check
// - Security Groups (ALB → Service)
// - IAM Roles (task role + execution role)
// - CloudWatch Log Group
```

**Accessing the internal components of an L3:**

```typescript
// The L3 exposes sub-constructs for customization
service.targetGroup.configureHealthCheck({
  path: '/health',
  healthyHttpCodes: '200',
});

service.loadBalancer.addSecurityGroup(mySG);
service.taskDefinition.addContainer('sidecar', { ... });

// For deeper customizations: escape hatch on the sub-resource
const cfnService = service.service.node.defaultChild as ecs.CfnService;
cfnService.addPropertyOverride('DeploymentConfiguration.MaximumPercent', 200);
```

**Other relevant native L3 patterns:**

```typescript
// SQS queue + Lambda consumer pre-configured
new sqs_event_sources.SqsEventSource(queue, { batchSize: 10 });

// API Gateway + Lambda integrated
new apigw.LambdaRestApi(this, 'Api', { handler: fn });

// SNS topic with email subscription
new sns_subscriptions.EmailSubscription('ops@company.com');
```

---

### 6. Escape hatches — dropping from L2 to L1

When an L2 doesn't expose what you need, you access the underlying L1 via escape hatch. This is the mechanism that ensures CDK never blocks you — you can always drop down a level.

```typescript
const bucket = new s3.Bucket(this, 'Bucket', {
  versioned: true,
});

// Access the underlying L1
const cfnBucket = bucket.node.defaultChild as s3.CfnBucket;

// Modify an existing L1 property
cfnBucket.lifecycleConfiguration = {
  rules: [
    {
      status: 'Enabled',
      transitions: [
        {
          storageClass: 'GLACIER',
          transitionInDays: 90,
        },
      ],
    },
  ],
};

// Add a property that L2 doesn't expose (using direct override)
cfnBucket.addPropertyOverride('ObjectLockEnabled', true);

// Add metadata to the CloudFormation resource
cfnBucket.addOverride('Metadata', {
  'aws:cdk:path': 'MyStack/Bucket/Resource',
  'Comment': 'Audited on 2026-05-11',
});

// Remove a property generated by L2
cfnBucket.addPropertyDeletionOverride('Tags');

// Pin the logical ID (for migrating existing stacks)
cfnBucket.overrideLogicalId('MyOriginalBucket');
```

**Escape hatch hierarchy (from least to most invasive):**

```
1. L2 props                   → use whenever available
   bucket.versioned = true

2. L2 methods                 → for integrations and permissions
   bucket.grantRead(fn)

3. node.defaultChild + props  → for L1 props not exposed in L2
   cfnBucket.lifecycleConfiguration = { ... }

4. addPropertyOverride        → for L1 props without CDK type
   cfnBucket.addPropertyOverride('NewProp', value)

5. addOverride                → for metadata, DependsOn, etc.
   cfnBucket.addOverride('DependsOn', ['OtherResource'])

6. Pure CfnResource (L1)     → when L2 doesn't work at all
   new s3.CfnBucket(this, 'Bucket', { ... })
```

---

### 7. Construct Hub — evaluating third-party constructs

[FACT] The [Construct Hub](https://constructs.dev) is the public repository for third-party CDK constructs, maintained by AWS. Constructs are published via npm (TypeScript/JavaScript), PyPI (Python), Maven (Java), and NuGet (C#).

**How to evaluate a third-party construct:**

```
Technical criteria:
  ✅ Compatible with CDK v2 (aws-cdk-lib, not @aws-cdk/*)
  ✅ Recent version with regular releases
  ✅ Tests in CI suite (GitHub Actions, etc.)
  ✅ README with clear usage examples
  ✅ Documented escape hatches

Risk criteria:
  ⚠️  Single maintainer vs. organization
  ⚠️  Number of dependents (downloads/week)
  ⚠️  Open issues without response
  ⚠️  Compatibility with your CDK version
  ⚠️  License compatible with intended use
```

**Alternative to third-party construct:** always check whether native CDK has an equivalent L3 before adding an external dependency. Third-party constructs have maintenance cost — when CDK releases an incompatible version, you depend on the third party to update.

---

## Practical example

Side-by-side comparison of the three levels for the same resource:

```typescript
import * as cdk from 'aws-cdk-lib';
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as iam from 'aws-cdk-lib/aws-iam';

export class ComparisonStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string) {
    super(scope, id);

    const role = new iam.Role(this, 'AppRole', {
      assumedBy: new iam.ServicePrincipal('lambda.amazonaws.com'),
    });

    // ── L1: full control, verbose ────────────────────────────────────
    const bucketL1 = new s3.CfnBucket(this, 'BucketL1', {
      versioningConfiguration: { status: 'Enabled' },
      bucketEncryption: {
        serverSideEncryptionConfiguration: [{
          serverSideEncryptionByDefault: { sseAlgorithm: 'AES256' },
        }],
      },
    });
    // To grant permission: you write the complete IAM Policy manually
    new iam.CfnPolicy(this, 'PolicyL1', {
      policyName: 'BucketAccess',
      roles: [role.roleName],
      policyDocument: {
        Version: '2012-10-17',
        Statement: [{
          Effect: 'Allow',
          Action: ['s3:GetObject', 's3:PutObject'],
          Resource: `${bucketL1.attrArn}/*`,
        }],
      },
    });

    // ── L2: concise, automatic helpers ──────────────────────────────
    const bucketL2 = new s3.Bucket(this, 'BucketL2', {
      versioned: true,
      encryption: s3.BucketEncryption.S3_MANAGED,
    });
    bucketL2.grantReadWrite(role);  // policy generated automatically

    // ── L2 + escape hatch: best of both worlds ────────────────────
    const bucketHybrid = new s3.Bucket(this, 'BucketHybrid', {
      versioned: true,
    });
    // Add property not exposed by L2
    const cfn = bucketHybrid.node.defaultChild as s3.CfnBucket;
    cfn.addPropertyOverride('ObjectLockEnabled', true);
    bucketHybrid.grantReadWrite(role);  // still uses the L2 helper
  }
}
```

**Run and compare the generated templates:**

```bash
cdk synth

# See how many resources each approach generated
cat cdk.out/ComparisonStack.template.json | jq '[.Resources | to_entries[] | .value.Type] | group_by(.) | map({type: .[0], count: length})'

# Inspect the policy generated by L2's grantReadWrite
cat cdk.out/ComparisonStack.template.json | \
  jq '.Resources | to_entries[] | select(.value.Type == "AWS::IAM::Policy") | .value.Properties.PolicyDocument'
```

---

## Common pitfalls

**1. Assuming L2 has all CloudFormation properties**

L2 is a manually maintained abstraction. New CloudFormation resources appear in L1 immediately (automatic generation), but in L2 they depend on a PR being merged. Before concluding that "CDK doesn't support X", check if L1 has what you need — in most cases, it does.

**2. Using escape hatch when an L2 method exists**

`cfnBucket.addPropertyOverride('BucketName', 'my-bucket')` works, but `new s3.Bucket(this, 'B', { bucketName: 'my-bucket' })` is more readable, typed, and validates the value. Use escape hatch only when L2 truly doesn't expose what you need.

**3. Third-party L3 constructs as black boxes**

A third-party L3 construct can create dozens of resources. If you haven't read the code or the README carefully, you may have surprises: resources with `RemovalPolicy.RETAIN` that you don't control, IAM policies broader than necessary, or networking configurations that conflict with your VPC. Always run `cdk synth` and review the generated template before adopting a third-party L3 in production.

---

## Reflection exercise

You are implementing an S3 bucket to store audit logs with the following security requirements: Object Lock enabled in GOVERNANCE mode with 7-year retention, notification to an SQS queue on every `s3:ObjectCreated:*`, and a bucket policy that denies any operation that doesn't use TLS.

When trying to implement with L2, you discover that `Object Lock` is not available as a direct prop on the CDK v2 `s3.Bucket` construct.

Describe the complete implementation strategy: which combination of L1, L2, and escape hatches you would use for each requirement, and why. Is there any requirement that is simpler in pure L1 than in L2 + escape hatch? If so, which one and why?

---

## Resources for deeper learning

**Construct model:**
- [AWS CDK Constructs](https://docs.aws.amazon.com/cdk/v2/guide/constructs.html) — official documentation with the definition of the three levels and examples in TypeScript and Python.

**Escape hatches:**
- [Customize constructs from the AWS Construct Library](https://docs.aws.amazon.com/cdk/v2/guide/cfn_layer.html) — all escape hatch methods with examples: `node.defaultChild`, `addPropertyOverride`, `addOverride`, `addPropertyDeletionOverride`.

**L3 ECS patterns:**
- [aws-cdk-lib.aws_ecs_patterns module](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_ecs_patterns-readme.html) — lists all available native patterns with complete examples.

**Construct Hub:**
- [constructs.dev](https://constructs.dev) — search by service or use case. Use the language and CDK v2 compatibility filters.

---
