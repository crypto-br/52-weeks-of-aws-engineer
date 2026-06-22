# Session 008 — CDK: Testing with assertions — fine-grained and snapshot tests

**Estimated duration:** 60 minutes
**Prerequisites:** session-007 — CDK: Assets — Lambda bundling, Docker images, and local files

---

## Objective

By the end, you will be able to write infrastructure unit tests with `aws-cdk-lib/assertions`, verify that resources exist with specific properties (`hasResourceProperties`), create snapshot tests, and interpret snapshot failures after infrastructure changes.

---

## Context

[CONSENSUS] Testing infrastructure as code is less about finding logic bugs and more about three things: **detecting regressions** (something that worked stopped working after a change), **documenting intentions** (the test describes what the construct must guarantee), and **gaining confidence to refactor** (you can reorganize code without fear of breaking behavior).

[FACT] CDK tests with `aws-cdk-lib/assertions` run locally and completely offline — they synthesize the CloudFormation template in memory and make assertions on the generated JSON. No AWS calls happen. A typical `npm test` runs in seconds.

[CONSENSUS] The official AWS documentation classifies CDK tests into three categories: fine-grained assertions (most used), snapshot tests (useful for refactoring), and validation tests (verify that invalid configurations throw exceptions). This session covers all three.

---

## Core concepts

### 1. Setup — Template.fromStack()

The entry point for any CDK test is `Template.fromStack()`, which synthesizes the stack in memory and returns a `Template` object with assertion methods.

```typescript
// test/my-stack.test.ts
import * as cdk from 'aws-cdk-lib';
import { Template, Match, Capture } from 'aws-cdk-lib/assertions';
import { MyStack } from '../lib/my-stack';

describe('MyStack', () => {
  let app: cdk.App;
  let stack: MyStack;
  let template: Template;

  beforeEach(() => {
    app = new cdk.App();
    stack = new MyStack(app, 'TestStack', {
      env: { account: '123456789012', region: 'us-east-1' },
    });
    template = Template.fromStack(stack);
  });

  // tests here
});
```

**Alternative: `Template.fromJSON()`** — for testing existing CloudFormation templates:

```typescript
import * as fs from 'fs';
const templateJson = JSON.parse(fs.readFileSync('path/to/template.json', 'utf8'));
const template = Template.fromJSON(templateJson);
```

---

### 2. Fine-grained assertions — the main methods

#### `hasResourceProperties` — verify properties of a resource

```typescript
// Verifies that an S3 bucket with versioning enabled exists
template.hasResourceProperties('AWS::S3::Bucket', {
  VersioningConfiguration: {
    Status: 'Enabled',
  },
});

// By default uses Match.objectLike — PARTIAL matching
// The bucket can have other properties beyond those verified
template.hasResourceProperties('AWS::S3::Bucket', {
  BucketEncryption: {
    ServerSideEncryptionConfiguration: [
      {
        ServerSideEncryptionByDefault: {
          SSEAlgorithm: 'AES256',
        },
      },
    ],
  },
});
```

#### `hasResource` — verify the complete resource (beyond Properties)

```typescript
// Verifies DeletionPolicy, DependsOn, Metadata, beyond Properties
template.hasResource('AWS::S3::Bucket', {
  DeletionPolicy: 'Retain',
  Properties: {
    VersioningConfiguration: { Status: 'Enabled' },
  },
});
```

#### `resourceCountIs` — verify resource count

```typescript
// Exactly 1 bucket must exist
template.resourceCountIs('AWS::S3::Bucket', 1);

// Exactly 2 Lambda functions must exist
template.resourceCountIs('AWS::Lambda::Function', 2);

// No EC2 resources should exist
template.resourceCountIs('AWS::EC2::Instance', 0);
```

#### `findResources` — inspect found resources

```typescript
// Returns an object with all found resources of the type
const buckets = template.findResources('AWS::S3::Bucket');
console.log(Object.keys(buckets));  // ['MyBucketXXXXXX']
console.log(Object.values(buckets)[0].Properties);  // bucket properties

// With filter
const encryptedBuckets = template.findResources('AWS::S3::Bucket', {
  Properties: {
    BucketEncryption: Match.anyValue(),
  },
});
```

#### `hasOutput` — verify stack outputs

```typescript
template.hasOutput('BucketName', {
  Value: { Ref: Match.stringLikeRegexp('MyBucket') },
  Export: { Name: Match.anyValue() },
});
```

---

### 3. Matchers — fine control over matching

The `Match` module provides matchers to control how comparisons are made.

```typescript
import { Match } from 'aws-cdk-lib/assertions';

// Match.objectLike — partial matching (default for hasResourceProperties)
// The actual object can have more keys than expected
template.hasResourceProperties('AWS::IAM::Role', {
  AssumeRolePolicyDocument: Match.objectLike({
    Statement: Match.arrayWith([
      Match.objectLike({
        Effect: 'Allow',
        Principal: { Service: 'lambda.amazonaws.com' },
      }),
    ]),
  }),
});

// Match.exact — EXACT matching — the object must be identical
template.hasResourceProperties('AWS::SQS::Queue', Match.exact({
  VisibilityTimeout: 300,
  MessageRetentionPeriod: 86400,
}));
// Will fail if the resource has any props beyond these two

// Match.arrayWith — array contains these elements (may have others)
template.hasResourceProperties('AWS::IAM::Policy', {
  PolicyDocument: {
    Statement: Match.arrayWith([
      {
        Effect: 'Allow',
        Action: 's3:GetObject',
        Resource: Match.anyValue(),
      },
    ]),
  },
});

// Match.arrayEquals — array is EXACTLY these elements, in this order
template.hasResourceProperties('AWS::Lambda::Function', {
  Layers: Match.arrayEquals([
    { Ref: 'MyLayerXXXXXX' },
  ]),
});

// Match.stringLikeRegexp — string that matches the regex
template.hasResourceProperties('AWS::Lambda::Function', {
  FunctionName: Match.stringLikeRegexp('^my-app-.*-handler$'),
});

// Match.anyValue — any non-null/undefined value
template.hasResourceProperties('AWS::S3::Bucket', {
  BucketName: Match.anyValue(),
});

// Match.absent — the property must NOT exist
template.hasResourceProperties('AWS::S3::Bucket', {
  WebsiteConfiguration: Match.absent(),
  PublicAccessBlockConfiguration: Match.absent(),   // confirms public access is blocked
});

// Match.not — negates any matcher
template.hasResourceProperties('AWS::Lambda::Function', {
  Runtime: Match.not(Match.stringLikeRegexp('nodejs14')),  // doesn't use Node 14
});

// Match.serializedJson — for properties that are JSON serialized as string
// (common in inline policies and environment variables that store JSON)
template.hasResourceProperties('AWS::Lambda::Function', {
  Environment: {
    Variables: {
      CONFIG: Match.serializedJson({
        timeout: 30,
        retries: 3,
      }),
    },
  },
});
```

---

### 4. Capture — capture values for additional assertions

`Capture` allows you to extract a value from the template to use in subsequent assertions — useful when the value is dynamic (hash, generated ARN) but you want to verify its structure.

```typescript
import { Capture } from 'aws-cdk-lib/assertions';

// Capture the role ARN to verify in another resource
const roleArnCapture = new Capture();

template.hasResourceProperties('AWS::Lambda::Function', {
  Role: roleArnCapture,
});

// Now roleArnCapture.asString() contains the captured value
// E.g., {"Fn::GetAtt": ["MyRoleXXXXXX", "Arn"]}
console.log(roleArnCapture.asString());

// Capture and verify a policy serialized as JSON
const policyCapture = new Capture();

template.hasResourceProperties('AWS::SQS::Queue', {
  RedrivePolicy: policyCapture,
});

// The value can be an object — use asObject()
const redrive = policyCapture.asObject();
expect(redrive.maxReceiveCount).toEqual(3);

// Capture with matcher — ensures the captured value has the right structure
const bucketCapture = new Capture(Match.objectLike({ 'Fn::GetAtt': Match.anyValue() }));

template.hasResourceProperties('AWS::Lambda::Function', {
  Environment: {
    Variables: {
      BUCKET_NAME: bucketCapture,
    },
  },
});
```

---

### 5. Validation tests — expected errors

Beyond verifying what is created, you should test that invalid configurations throw exceptions:

```typescript
test('should throw error when port is out of range', () => {
  const app = new cdk.App();
  const stack = new cdk.Stack(app, 'TestStack');

  expect(() => {
    new MyWebServer(stack, 'Server', {
      port: 70000,  // invalid port
    });
  }).toThrow('Port must be between 1024 and 65535');
});

test('should throw error when bucket name has uppercase', () => {
  const app = new cdk.App();
  const stack = new cdk.Stack(app, 'TestStack');

  expect(() => {
    new s3.Bucket(stack, 'Bucket', {
      bucketName: 'MyInvalidBucket',  // uppercase not allowed
    });
  }).toThrow(/bucket name/i);
});
```

---

### 6. Snapshot tests — the template as baseline

Snapshot tests store the complete CloudFormation template in a `.snap` file and compare on each future execution.

```typescript
test('complete stack snapshot', () => {
  const app = new cdk.App();
  const stack = new MyStack(app, 'SnapshotStack');
  const template = Template.fromStack(stack);

  // On first run: creates the .snap file
  // On subsequent runs: compares with the saved snapshot
  expect(template.toJSON()).toMatchSnapshot();
});
```

**Snapshot file structure (Jest):**

```
test/
├── my-stack.test.ts
└── __snapshots__/
    └── my-stack.test.ts.snap   ← automatically generated, commit to git
```

**Updating snapshots after intentional change:**

```bash
# Update all snapshots
npx jest --updateSnapshot

# Update snapshot of a specific test
npx jest --updateSnapshot --testNamePattern="complete stack snapshot"

# Shorthand
npx jest -u
```

**What to do when a snapshot fails:**

```
1. Read the diff — Jest shows exactly what changed
2. Evaluate if the change is:
   a) Intentional (you altered the code) → npx jest -u → commit the new snapshot
   b) Side effect of a CDK update → review if the new behavior is ok
   c) Unexpected regression → find the cause and fix the code
3. NEVER run -u without reading the diff
```

---

### 7. Fine-grained vs snapshot — when to use each

[FACT, per official documentation] AWS recommends using fine-grained assertions as the base of tests and snapshot tests as a complement for refactoring.

```
Fine-grained assertions
  ✅ Detect regressions of specific behavior
  ✅ Are explicit about what is being verified
  ✅ Don't break from unrelated changes in the template
  ✅ Document construct intentions
  ✅ Work as contract tests for reusable constructs
  ⚠️  More verbose to write
  ⚠️  Don't cover the entire template

Snapshot tests
  ✅ Cover the entire template without effort
  ✅ Detect any change — including ones you didn't anticipate
  ✅ Useful during refactoring: ensures refactoring doesn't change output
  ⚠️  Break with CDK updates that change the template for internal reasons
  ⚠️  Don't distinguish between intentional change and regression
  ⚠️  Can accumulate false positives if the team runs -u automatically
  ⚠️  Not useful for detecting regressions in new constructs
```

**Recommended combined strategy:**

```
For each construct:
  1. Fine-grained: verifies critical invariants
     (correct IAM permissions, encryption enabled, correct deletion policy)
  2. Fine-grained: verifies that invalid configurations throw errors
  3. Snapshot: captures the complete template as baseline for refactoring
```

---

## Practical example

Application stack with complete test coverage:

```typescript
// lib/app-stack.ts
export class AppStack extends cdk.Stack {
  public readonly bucket: s3.Bucket;
  public readonly handler: lambda_nodejs.NodejsFunction;

  constructor(scope: Construct, id: string, props: AppStackProps) {
    super(scope, id, props);

    this.bucket = new s3.Bucket(this, 'AppBucket', {
      versioned: true,
      encryption: s3.BucketEncryption.S3_MANAGED,
      blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
      removalPolicy: props.isProd ? cdk.RemovalPolicy.RETAIN : cdk.RemovalPolicy.DESTROY,
    });

    this.handler = new lambda_nodejs.NodejsFunction(this, 'Handler', {
      entry: path.join(__dirname, '../src/handler.ts'),
      runtime: lambda.Runtime.NODEJS_22_X,
      environment: {
        BUCKET_NAME: this.bucket.bucketName,
        IS_PROD: props.isProd ? 'true' : 'false',
      },
    });

    this.bucket.grantReadWrite(this.handler);
  }
}

// test/app-stack.test.ts
import { Template, Match, Capture } from 'aws-cdk-lib/assertions';

describe('AppStack', () => {
  const makeStack = (isProd = false) => {
    const app = new cdk.App();
    const stack = new AppStack(app, 'TestStack', {
      env: { account: '123456789012', region: 'us-east-1' },
      isProd,
    });
    return Template.fromStack(stack);
  };

  describe('S3 Bucket', () => {
    test('bucket has versioning enabled', () => {
      const template = makeStack();
      template.hasResourceProperties('AWS::S3::Bucket', {
        VersioningConfiguration: { Status: 'Enabled' },
      });
    });

    test('bucket has encryption enabled', () => {
      const template = makeStack();
      template.hasResourceProperties('AWS::S3::Bucket', {
        BucketEncryption: {
          ServerSideEncryptionConfiguration: Match.arrayWith([
            Match.objectLike({
              ServerSideEncryptionByDefault: { SSEAlgorithm: 'AES256' },
            }),
          ]),
        },
      });
    });

    test('bucket in dev has DeletionPolicy Delete', () => {
      const template = makeStack(false);
      template.hasResource('AWS::S3::Bucket', {
        DeletionPolicy: 'Delete',
      });
    });

    test('bucket in prod has DeletionPolicy Retain', () => {
      const template = makeStack(true);
      template.hasResource('AWS::S3::Bucket', {
        DeletionPolicy: 'Retain',
      });
    });

    test('bucket does NOT have public access', () => {
      const template = makeStack();
      template.hasResourceProperties('AWS::S3::Bucket', {
        PublicAccessBlockConfiguration: {
          BlockPublicAcls: true,
          BlockPublicPolicy: true,
          IgnorePublicAcls: true,
          RestrictPublicBuckets: true,
        },
      });
    });
  });

  describe('Lambda Function', () => {
    test('function has Node 22 runtime', () => {
      const template = makeStack();
      template.hasResourceProperties('AWS::Lambda::Function', {
        Runtime: 'nodejs22.x',
      });
    });

    test('function has BUCKET_NAME variable in environment', () => {
      const template = makeStack();
      const bucketRef = new Capture();

      template.hasResourceProperties('AWS::Lambda::Function', {
        Environment: {
          Variables: {
            BUCKET_NAME: bucketRef,
          },
        },
      });

      // The value should be a reference to the bucket, not hardcoded
      expect(JSON.stringify(bucketRef.asObject())).toContain('Ref');
    });

    test('function has read and write permission on the bucket', () => {
      const template = makeStack();

      // An IAM Policy with S3 permissions must exist
      template.hasResourceProperties('AWS::IAM::Policy', {
        PolicyDocument: {
          Statement: Match.arrayWith([
            Match.objectLike({
              Effect: 'Allow',
              Action: Match.arrayWith([
                's3:GetObject*',
                's3:PutObject*',
              ]),
            }),
          ]),
        },
      });
    });
  });

  describe('Resource count', () => {
    test('should create exactly 1 bucket', () => {
      const template = makeStack();
      template.resourceCountIs('AWS::S3::Bucket', 1);
    });

    test('should create exactly 1 Lambda function', () => {
      const template = makeStack();
      template.resourceCountIs('AWS::Lambda::Function', 1);
    });
  });

  describe('Snapshot', () => {
    test('template does not change unexpectedly', () => {
      const template = makeStack();
      expect(template.toJSON()).toMatchSnapshot();
    });
  });
});
```

**Running the tests:**

```bash
# Run all tests
npx jest

# Run with coverage
npx jest --coverage

# Run in watch mode (re-executes on save)
npx jest --watch

# Update snapshots
npx jest --updateSnapshot

# Run only one file
npx jest test/app-stack.test.ts

# Run a specific test by name
npx jest --testNamePattern="bucket has versioning"

# See detailed output of a failed test
npx jest --verbose
```

**Interpreting a snapshot failure:**

```
● AppStack › Snapshot › template does not change unexpectedly

  expect(received).toMatchSnapshot()

  Snapshot name: `AppStack Snapshot template does not change unexpectedly 1`

  - Snapshot  - 5 lines
  + Received  + 5 lines

    "MyHandlerXXXXXX": {
      "Type": "AWS::Lambda::Function",
      "Properties": {
  -     "Runtime": "nodejs20.x",    ← value in saved snapshot
  +     "Runtime": "nodejs22.x",    ← current value in code
        "Handler": "index.handler",

  # Interpretation:
  # You updated the runtime from nodejs20 to nodejs22.
  # Intentional change → npx jest -u to accept the new baseline.
```

---

## Common pitfalls

**1. `toMatchSnapshot` without reviewing the diff — "snapshot debt"**

Teams that run `npx jest -u` automatically in CI (to avoid snapshot failures after CDK updates) accumulate snapshots that were never reviewed. The snapshot stops being a test and becomes outdated documentation. Rule: `npx jest -u` only in commits where the snapshot change is the explicit goal, and the diff must be reviewed before merge.

**2. `Match.objectLike` vs `Match.exact` — false approvals**

`hasResourceProperties` uses `Match.objectLike` by default — the template can have more properties than those verified. If you want to guarantee that a property **does not exist**, use `Match.absent()`. If you want the object to be **exactly** what you declared, use `Match.exact()`. Don't assume that passing `hasResourceProperties` means the resource has no additional unwanted properties.

**3. Testing logical IDs instead of properties**

Avoid writing tests that depend on the logical ID generated by CDK (e.g., `{ Ref: 'MyBucketA1B2C3D4' }`). Logical IDs change when you rename constructs or reorganize the hierarchy. Use `Match.anyValue()` or `Capture` to isolate the test from the logical ID.

---

## Reflection exercise

You are writing tests for a `SecureBucket` construct that must guarantee: bucket always encrypted with KMS (not SSE-S3), public access always blocked, versioning enabled, and a Bucket Policy that denies operations without TLS (`aws:SecureTransport: false`).

Write the fine-grained tests for each of these requirements. For each test, explain: what exactly is being verified, which matcher is most appropriate and why, and what the test **does not** verify (its limits). Finally, is the snapshot test for this construct useful or redundant given the fine-grained tests? Justify.

---

## Resources for deeper learning

**Testing guide:**
- [Test AWS CDK applications](https://docs.aws.amazon.com/cdk/v2/guide/testing.html) — covers the three test types (fine-grained, snapshot, validation) with examples in TypeScript and Python.

**Assertions API:**
- [aws-cdk-lib.assertions module](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.assertions-readme.html) — complete reference of all `Template` methods and all `Match.*` matchers with examples of each.

**AWS article:**
- [Testing Infrastructure with the AWS CDK](https://aws.amazon.com/blogs/developer/testing-infrastructure-with-the-aws-cloud-development-kit/) — complete walk-through with practical examples of all three test types.

---
