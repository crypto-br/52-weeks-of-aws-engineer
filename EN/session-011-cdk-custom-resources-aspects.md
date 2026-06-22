# Session 11 — CDK: CustomResources and Aspects

**Estimated duration:** 60 minutes
**Prerequisites:** session-010-cdk-pipelines-stages-shellsteps

---

## Objective

By the end, you will be able to create a CustomResource that invokes a Lambda to provision resources not natively supported by CloudFormation, and apply an Aspect to traverse all constructs in a stack (e.g., force encryption on all S3 buckets automatically).

---

## Context

[FACT] CloudFormation only knows the resource types that AWS has published — approximately 1,200 types as of May 2026. Every resource outside that catalog (configurations in third-party APIs, database initialization operations, DNS records in external providers, data bootstrapping) needs an extension mechanism. This mechanism has existed since 2012 and is called **Custom Resource**: a `Type: Custom::*` block in the template that delegates the lifecycle (Create/Update/Delete) to a Lambda or an SNS topic.

[CONSENSUS] CDK offers two abstractions over Custom Resources: the `custom_resources.Provider` (a mini-framework with support for asynchronous operations, retry, and timeout) and `AwsCustomResource` (a wrapper that executes a single SDK call without you needing to write Lambda code). The choice between them follows a simple heuristic: if the logic fits in a single API call, use `AwsCustomResource`; if you need business logic, polling, or multiple calls, use `Provider`.

[FACT] Aspects are a construct tree traversal mechanism introduced in CDK v1 and maintained in v2. They implement the Visitor pattern: you write a `visit(node)` function and CDK invokes it for every node in the tree during synthesis, after all constructs have been instantiated. This allows inspecting or modifying the tree in a cross-cutting manner — without each construct needing to know about the rule being applied.

---

## Key Concepts

### 1. CloudFormation Custom Resource: the base protocol

Before seeing the CDK abstraction, understand what's underneath. When CloudFormation needs to create a Custom Resource, it makes an HTTP POST to a pre-signed S3 URL (or invokes a Lambda directly) with a JSON payload like this:

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

The handler (Lambda or SNS) must respond with a JSON object to the `ResponseURL` containing `Status` (SUCCESS or FAILED), `PhysicalResourceId`, and optionally `Data` (attributes available via `Fn::GetAtt`).

[FACT] The **Physical Resource ID** is the most critical field of the protocol. It uniquely identifies the external resource instance. The rules are:

```
CREATE:  you generate the PhysicalResourceId (e.g., "acme-cert-app.example.com")
UPDATE:  you return the same ID if the operation is in-place,
         or a different ID if it's a replacement.
         → If the ID changes, CloudFormation sends a DELETE event
           for the old ID immediately after!
DELETE:  CloudFormation sends the ID that was stored during Create/Update.
         You use it to destroy the external resource.
```

[FACT] If the Lambda fails (uncaught exception) or doesn't respond within 60 minutes (Custom Resource default timeout), CloudFormation gets stuck waiting and eventually rolls back after 1 hour. That's why the CDK Provider exists: it eliminates the need to write the HTTP response code manually and manages the timeout.

---

### 2. custom_resources.Provider — the CDK mini-framework

[FACT] The `Provider` is a construct that creates all the infrastructure needed to manage a Custom Resource robustly:

```
┌─────────────────────────────────────────────────────────────┐
│  Provider (construct)                                       │
│                                                             │
│  ┌──────────────┐    ┌──────────────────────────────────┐  │
│  │  onEvent     │    │  (if isComplete defined)         │  │
│  │  Lambda      │    │  ┌─────────────────────────────┐ │  │
│  │              │───▶│  │ Step Functions state machine│ │  │
│  │  CREATE/     │    │  │  ┌──────────┐  ┌─────────┐  │ │  │
│  │  UPDATE/     │    │  │  │isComplete│  │ waiter  │  │ │  │
│  │  DELETE      │    │  │  │  Lambda  │◀─│  loop   │  │ │  │
│  └──────────────┘    │  │  └──────────┘  └─────────┘  │ │  │
│                      │  └─────────────────────────────┘ │  │
│                      └──────────────────────────────────┘  │
│                                                             │
│  serviceToken: Lambda ARN or Step Functions ARN             │
└─────────────────────────────────────────────────────────────┘
         │
         │ serviceToken
         ▼
┌─────────────────┐
│  CustomResource │  ◀─── CloudFormation treats it as a regular resource
│  (construct)    │
└─────────────────┘
```

The minimal setup in TypeScript:

```typescript
import * as cr from 'aws-cdk-lib/custom-resources';
import { CustomResource, Duration } from 'aws-cdk-lib';
import * as lambda from 'aws-cdk-lib/aws-lambda';

// 1. Your handler Lambda
const onEventHandler = new lambda.Function(this, 'OnEvent', {
  runtime: lambda.Runtime.NODEJS_22_X,
  handler: 'index.handler',
  code: lambda.Code.fromInline(`
    exports.handler = async (event) => {
      console.log('Event:', JSON.stringify(event));
      const physicalId = event.PhysicalResourceId || 'my-resource-' + Date.now();
      
      if (event.RequestType === 'Delete') {
        // clean up the external resource here
        return { PhysicalResourceId: physicalId };
      }
      
      // create or update
      const result = await doSomething(event.ResourceProperties);
      return {
        PhysicalResourceId: result.id,
        Data: { Endpoint: result.endpoint, ApiKey: result.apiKey },
      };
    };
  `),
});

// 2. Provider that registers the handler
const provider = new cr.Provider(this, 'Provider', {
  onEventHandler,
  // isCompleteHandler: optionally a second Lambda for polling
  // totalTimeout: Duration.minutes(30),  // max for async operations
});

// 3. Custom Resource connected to the Provider
const resource = new CustomResource(this, 'Resource', {
  serviceToken: provider.serviceToken,
  properties: {
    Domain: 'app.example.com',
    Version: '2',   // changing this value in the future triggers an UPDATE
  },
  resourceType: 'Custom::AcmeCertRegistration',   // Custom:: prefix is required
});

// 4. Reading attributes returned by the handler (via Data{})
const endpoint = resource.getAttString('Endpoint');
```

[FACT] The Provider's `serviceToken` property can be the ARN of a Lambda or the ARN of a Step Functions Express Workflow (when `isCompleteHandler` is defined). When there's an `isCompleteHandler`, the Provider creates a state machine that calls `onEvent` once, then keeps polling `isComplete` every `queryInterval` (default 5 seconds) until it returns `{ IsComplete: true }` or reaches `totalTimeout`.

---

### 3. AwsCustomResource — the shortcut for single SDK calls

[FACT] `AwsCustomResource` + `AwsSdkCall` is a high-level abstraction that allows executing any AWS SDK call without writing Lambda code. Internally, it uses the `Provider` with a generic Lambda that executes SDK calls dynamically.

Classic use case: fetching a parameter from SSM Parameter Store in a different region from the stack (`ssm.StringParameter.valueFromLookup` only works at synthesis time, not deploy time):

```typescript
import { AwsCustomResource, AwsSdkCall, PhysicalResourceId } from 'aws-cdk-lib/custom-resources';
import * as iam from 'aws-cdk-lib/aws-iam';

const getParam = new AwsCustomResource(this, 'GetParam', {
  onUpdate: {   // 'onUpdate' is called on both Create and Update
    service: 'SSM',
    action: 'getParameter',
    parameters: {
      Name: '/shared/database-endpoint',
      WithDecryption: true,
    },
    region: 'us-east-1',                         // different region from the stack!
    physicalResourceId: PhysicalResourceId.of(Date.now().toString()),
  },
  policy: AwsCustomResourcePolicy.fromSdkCalls({
    resources: AwsCustomResourcePolicy.ANY_RESOURCE,  // in prod: specific ARN
  }),
});

const dbEndpoint = getParam.getResponseField('Parameter.Value');
```

[CONSENSUS] `AwsCustomResource` requires you to explicitly define the `policy` — the IAM permissions the generic Lambda needs to execute the SDK call. Using `ANY_RESOURCE` is convenient in development, but in production always restrict to the specific resource ARN.

---

### 4. Aspects and IAspect: the Visitor pattern on the construct tree

[FACT] An Aspect is any object that implements the `IAspect` interface:

```typescript
interface IAspect {
  visit(node: IConstruct): void;
}
```

You apply it to a scope with `Aspects.of(scope).add(aspect)`. CDK, during the synthesis phase (after the entire tree is constructed), traverses the tree in **depth-first, pre-order** invoking `visit(node)` on each construct:

```
App
└── Stack
    ├── Bucket1          ← visit() called here
    ├── Lambda1          ← and here
    │   └── Role         ← and here (child of Lambda1)
    └── Queue1           ← and here
```

If you apply the Aspect to the `App`, it visits all constructs in the application. If applied to a specific `Stack`, it visits only the constructs in that stack.

[FACT] Inside `visit()`, you can:

1. **Inspect** the construct and add error or warning annotations:
```typescript
import { Annotations } from 'aws-cdk-lib';
import * as s3 from 'aws-cdk-lib/aws-s3';

class BucketEncryptionChecker implements IAspect {
  visit(node: IConstruct): void {
    if (node instanceof s3.CfnBucket) {
      if (!node.bucketEncryption) {
        Annotations.of(node).addError(
          'All S3 buckets must have encryption configured. ' +
          'Use BucketEncryption.S3_MANAGED or KMS.'
        );
      }
    }
  }
}
```

2. **Mutate** the construct by adding properties or calling methods:
```typescript
class EnforceS3Encryption implements IAspect {
  visit(node: IConstruct): void {
    if (node instanceof s3.CfnBucket) {
      // forces SSE-S3 encryption on any bucket that doesn't already have one
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

[FACT] `addError()` makes `cdk synth` fail with a descriptive message — the CloudAssembly is not generated. `addWarning()` and `addInfo()` only emit messages but don't interrupt synthesis. This makes Aspects suitable for **compliance gates**: the CDK pipeline fails at `synth` before even reaching the deploy.

---

### 5. Aspects in practice: Tags, Annotations, and limitations

[FACT] CDK's own `Tags` system is internally implemented as an Aspect. When you call `Tags.of(stack).add('Environment', 'production')`, CDK registers an Aspect that, when visiting each construct, adds the tag to the corresponding CloudFormation resource.

[FACT] When traversing the tree, the `node` received in `visit()` is typed as `IConstruct`. To act only on a specific resource type, use `instanceof`:

```typescript
// For L2 constructs (CDK level):
if (node instanceof s3.Bucket) { ... }

// For L1 resources (CloudFormation level):
if (node instanceof s3.CfnBucket) { ... }
```

[FACT] There's an important difference between visiting L2 and L1:

- Visiting `s3.Bucket` (L2) gives access to CDK's high-level API (`bucket.addLifecycleRule()`, `bucket.grantRead()`). However, the L2 may not exist if the resource was created via L1 directly.
- Visiting `s3.CfnBucket` (L1) guarantees you see **all** buckets, but you manipulate CloudFormation properties directly (like in the `bucketEncryption` example above).

[CONSENSUS] The dominant convention in the CDK community is to visit L1 in compliance and mutation Aspects, because it's the common denominator — every L2 resource eventually creates an L1.

**Critical limitation:** [FACT] You **must not create new constructs** inside `visit()`. CDK doesn't guarantee the order of Aspect application relative to tree construction, and adding constructs during visitation can cause undefined behavior (ignored constructs, synthesis loops). If you need to create infrastructure conditionally, create it in the constructor and enable/disable via properties.

---

## Practical Example

**Scenario:** You have an infrastructure stack that creates S3 buckets in different parts of the code. The company's security policy requires every bucket to have SSE-KMS encryption (not SSE-S3) and versioning enabled. Instead of auditing each `new s3.Bucket(...)` manually, you want `cdk synth` to fail automatically if any bucket violates the rules.

### File structure

```
lib/
  aspects/
    bucket-compliance.ts   ← Validation Aspect
  stacks/
    storage-stack.ts       ← Stack with buckets
  app.ts                   ← Entry point
```

### `lib/aspects/bucket-compliance.ts`

```typescript
import { IAspect, Annotations } from 'aws-cdk-lib';
import * as s3 from 'aws-cdk-lib/aws-s3';
import { IConstruct } from 'constructs';

export class BucketComplianceAspect implements IAspect {
  visit(node: IConstruct): void {
    // Only visits CfnBucket resources (L1) to cover all buckets
    if (!(node instanceof s3.CfnBucket)) return;

    // Rule 1: KMS encryption required
    const enc = node.bucketEncryption as s3.CfnBucket.BucketEncryptionProperty | undefined;
    const hasKms = enc?.serverSideEncryptionConfiguration?.some(
      (rule: any) =>
        rule.serverSideEncryptionByDefault?.sseAlgorithm === 'aws:kms'
    );
    if (!hasKms) {
      Annotations.of(node).addError(
        '[SECURITY] Bucket without SSE-KMS encryption. ' +
        'Configure encryptionKey or use BucketEncryption.KMS.'
      );
    }

    // Rule 2: Versioning required
    const versioning = node.versioningConfiguration as
      s3.CfnBucket.VersioningConfigurationProperty | undefined;
    if (versioning?.status !== 'Enabled') {
      Annotations.of(node).addWarning(
        '[COMPLIANCE] Bucket without versioning enabled. ' +
        'Enable with versioned: true.'
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

    // Compliant bucket: KMS + versioning
    new s3.Bucket(this, 'LogsBucket', {
      encryptionKey: key,
      encryption: s3.BucketEncryption.KMS,
      versioned: true,
    });

    // NON-compliant bucket: no encryption and no versioning
    // The Aspect will emit an error and a warning here
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

// Apply the Aspect to the entire stack
Aspects.of(storageStack).add(new BucketComplianceAspect());

app.synth();
```

### `cdk synth` output

```
[Error at /StorageStack/TempBucket/Resource] [SECURITY] Bucket without SSE-KMS 
encryption. Configure encryptionKey or use BucketEncryption.KMS.

[Warning at /StorageStack/TempBucket/Resource] [COMPLIANCE] Bucket without 
versioning enabled. Enable with versioned: true.

Found errors
```

Synthesis fails (`Found errors`). The `TempBucket` needs to be fixed before any deploy happens.

---

### Bonus: CustomResource combined with AwsCustomResource

Real problem: you need, when creating the stack, to register a webhook in an external service (e.g., GitHub) using that service's REST API. `AwsCustomResource` covers AWS SDK calls, but not arbitrary HTTP calls. Here you use the full `Provider`.

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
    githubTokenSecret: string; // Secrets Manager ARN
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

    // Permission to read the secret
    // handler.addToRolePolicy(new iam.PolicyStatement({ ... }));

    const provider = new cr.Provider(this, 'Provider', {
      onEventHandler: handler,
    });

    new CustomResource(this, 'Resource', {
      serviceToken: provider.serviceToken,
      properties: {
        Repo: props.repo,
        WebhookUrl: props.webhookUrl,
        // Including a hash of the token causes the webhook to be re-registered if the token changes
        TokenHash: SecretValue.secretsManager(props.githubTokenSecret).toString(),
      },
      resourceType: 'Custom::GithubWebhook',
    });
  }
}
```

---

## Common Pitfalls

### Pitfall 1: Changing the PhysicalResourceId on an Update causes Delete of the old resource

**The error:** You have a Custom Resource that generates an ID based on some properties. In an update, you change a property that makes the ID different. CloudFormation sends an `UPDATE` for the new ID and then a `DELETE` for the old ID — your Lambda will try to delete a resource that was just created with a different name, which is often correct but can be a silent bug if you weren't expecting the `DELETE`.

**Why it happens:** The CloudFormation Custom Resource protocol specifies that if the `PhysicalResourceId` changes on an Update, the old resource was "replaced" and should be deleted.

**How to recognize:** In the Lambda logs (CloudWatch), you see two events: `RequestType: Update` followed by `RequestType: Delete` with the previous ID.

**How to avoid:** Be deliberate about when to change the `PhysicalResourceId`. If the operation is always in-place (e.g., updating configuration of an existing resource), always return the same ID. If the operation creates a new resource (e.g., registering a certificate with a different domain), changing the ID is correct — but ensure the `DELETE` handler knows how to deal with attempting to delete something that may no longer exist.

---

### Pitfall 2: Aspect visiting L2 doesn't catch buckets created via L1

**The error:** You write `if (node instanceof s3.Bucket)` in your Aspect, but part of the infrastructure uses `new s3.CfnBucket(...)` directly. The Aspect passes through those resources without detecting them.

**Why it happens:** `s3.Bucket` and `s3.CfnBucket` are different classes in the hierarchy. TypeScript's `instanceof` checks the exact type, not the relationship with the underlying CloudFormation resource.

**How to recognize:** `cdk synth` passes without errors for a `CfnBucket` that should be intercepted. A manual review or a unit test (using `Template.hasResourceProperties`) can reveal the problem.

**How to avoid:** In compliance Aspects that need to cover 100% of resources of a type, always visit L1 (`CfnBucket`, `CfnFunction`, etc.). If you need to visit L2 for some reason, add a separate check for L1 as well.

---

### Pitfall 3: Creating constructs inside visit() causes undefined behavior

**The error:** Inside `visit(node)`, you identify that a bucket doesn't have a log bucket and try to create a `new s3.Bucket(this, 'AccessLogs', ...)` to configure it automatically. On the first run it seems to work, but on subsequent deployments CDK emits warnings about non-deterministic synthesis, or the new bucket isn't included in already-applied Aspects.

**Why it happens:** The synthesis phase traverses the tree once. Adding new constructs during that traversal is like adding elements to a list while iterating over it — the behavior is undefined by the CDK specification.

**How to avoid:** Never instantiate `new SomeConstruct(...)` inside `visit()`. If you need to create resources conditionally, create them in the constructor and use flags or props to control creation. If you need to detect the absence of something and fix it, consider creating a custom L2 construct that encapsulates the rules in the constructor, instead of using an Aspect for mutation.

---

## Reflection Exercise

You're building an internal platform where different teams create independent CDK stacks. The security policy mandates:

1. Every S3 bucket must use SSE-KMS with a KMS key that has automatic rotation enabled.
2. Every SNS topic must have encryption at rest enabled.
3. Every Lambda function must have a DLQ (Dead Letter Queue) configured.
4. No Security Group can have an ingress rule with cidr `0.0.0.0/0` on port 22.

How would you structure Aspects to implement these four rules? For each rule, decide: would you visit L1 or L2? Would you use `addError()` (blocking) or `addWarning()` (informational)? How would you organize the Aspects in code — a single Aspect with all checks, or one Aspect per rule? Consider how this solution would scale if the company grew to 20 compliance rules and 50 teams with independent stacks. Are there cases where an automatic mutation Aspect (silently fixing instead of reporting the error) would be preferable? What are the risks of that approach in a team environment?

---

## Resources for Further Study

### 1. CDK v2 Guide — Custom Resources (cfn_layer)
**URL:** https://docs.aws.amazon.com/cdk/v2/guide/cfn_layer.html#develop-customize-expose
**What to find:** The "Develop a custom resource provider" section explains CDK's escape hatch layer for interacting directly with CloudFormation Custom Resources, including how to expose attributes via `Data` and how to use the escape hatch `node.defaultChild` for customizations.
**Why it's the right source:** It's the official CDK documentation for the "expose" approach — the closest point to creating your own resource types via CDK.

### 2. aws-cdk-lib.custom_resources module README
**URL:** https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.custom_resources-readme.html
**What to find:** Complete documentation of `Provider`, `AwsCustomResource`, and `AwsSdkCall`. Includes diagrams of the Provider's internal architecture (with Step Functions), examples of `isCompleteHandler` for asynchronous resources, and the complete list of response properties.
**Why it's the right source:** It's the canonical module reference — more detailed than the narrative guide.

### 3. CDK v2 Guide — Aspects
**URL:** https://docs.aws.amazon.com/cdk/v2/guide/aspects.html
**What to find:** Examples of Aspects for validation (with `addError`) and for tag application. Explains the execution order (depth-first, pre-order) and how to apply Aspects to different scopes (App, Stack, individual Construct).
**Why it's the right source:** It's the official feature document, with examples in TypeScript, Python, and Java, and clarifies the guarantees and limitations of the mechanism.

---
