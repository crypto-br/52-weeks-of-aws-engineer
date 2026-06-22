# Session 12 — CDK: Context, feature flags, and production-grade cdk.json

**Estimated duration:** 60 minutes
**Prerequisites:** session-011-cdk-custom-resources-aspects

---

## Objective

By the end, you will be able to use `cdk.context.json` to cache lookups (VPCs, AMIs) without depending on runtime resolution, configure feature flags to control migration behaviors, and structure `cdk.json` for teams (what goes in gitignore vs what is versioned).

---

## Context

[FACT] One of the fundamental properties of a good infrastructure pipeline is **reproducibility**: given the same source code, two engineers on different machines or two CI builds produce exactly the same CloudFormation template. CDK breaks this property if any AWS account lookup is done at synthesis time without caching — because the response can change (a new AMI was published, a new AZ was enabled) between the developer's build and the CI build.

[FACT] The mechanism CDK uses to make lookups reproducible is called **Context**. Context is a key-value dictionary that the CDK CLI automatically populates the first time a lookup is needed and persists in `cdk.context.json`. On subsequent syntheses, the cached value is used without making a new query to AWS. The `cdk.context.json` file is versioned in the repository — this is the reproducibility guarantee.

[CONSENSUS] Feature flags are an overlay on the same Context mechanism: they are booleans stored in `cdk.json` under the `"context"` key that control CDK behaviors that changed between versions and that could break existing stacks if silently activated. Understanding them is mandatory when migrating stacks from CDK v1 to v2, and when updating CDK v2 projects between minor versions that introduced opt-in changes.

---

## Key Concepts

### 1. The lifecycle of a Context lookup

When your CDK code calls something like `ec2.Vpc.fromLookup(this, 'Vpc', { vpcName: 'prod-vpc' })`, CDK needs to query the AWS account to discover the VPC ID, its subnets, and AZs. This process has two modes:

```
                    ┌─────────────────────────────────────────────┐
                    │  cdk synth (first time, no cache)           │
                    │                                             │
  CDK Code          │  1. Synthesis attempt                      │
  Vpc.fromLookup()  │     → CDK detects pending lookup            │
        │           │     → Synthesizes with DUMMY values         │
        │           │     → Does NOT generate valid template      │
        │           │  2. CLI runs describe-vpcs on AWS           │
        │           │     → stores in cdk.context.json            │
        │           │  3. CDK retries synthesis                   │
        │           │     → lookup resolved from cache            │
        │           │     → valid template generated              │
        └───────────┘

                    ┌─────────────────────────────────────────────┐
                    │  cdk synth (with cache / CI)                │
                    │                                             │
  CDK Code          │  1. CDK reads cdk.context.json             │
  Vpc.fromLookup()  │     → lookup already resolved              │
        │           │  2. Template generated directly             │
        │           │     → zero AWS calls needed                 │
        └───────────┘
```

[FACT] In CI mode (without account credentials, or with `--no-lookups`), if `cdk.context.json` doesn't have the cached value, synthesis fails with an explicit error. This is intentional — it ensures CI doesn't depend on write access to the cache to function.

[FACT] The content of `cdk.context.json` after a VPC lookup looks like this:

```json
{
  "vpc-provider:account=123456789012:filter.vpc-id=vpc-0abc:region=us-east-1:returnAsymmetricSubnets=true": {
    "vpcId": "vpc-0abc",
    "vpcCidrBlock": "10.0.0.0/16",
    "availabilityZones": ["us-east-1a", "us-east-1b", "us-east-1c"],
    "subnetGroups": [
      {
        "name": "Public",
        "type": "Public",
        "subnets": [
          { "subnetId": "subnet-pub1", "cidr": "10.0.0.0/24", "availabilityZone": "us-east-1a", "routeTableId": "rtb-1" }
        ]
      }
    ]
  }
}
```

The key is deterministically generated from the lookup parameters. If you change any parameter (e.g., the `vpcName`), CDK generates a different key and does a new lookup.

---

### 2. Lookups natively available in CDK

[FACT] CDK offers the following first-class account lookups (all cached in `cdk.context.json`):

| Lookup method | What it queries on AWS | Key used |
|---|---|---|
| `ec2.Vpc.fromLookup()` | `ec2:DescribeVpcs` + `ec2:DescribeSubnets` | filter params |
| `ec2.MachineImage.lookup()` | `ec2:DescribeImages` | AMI filters |
| `ssm.StringParameter.valueFromLookup()` | `ssm:GetParameter` | param name + account/region |
| `HostedZone.fromLookup()` | `route53:ListHostedZonesByName` | domain |
| `ec2.SecurityGroup.fromLookupByName()` | `ec2:DescribeSecurityGroups` | name + VPC ID |

[FACT] `ssm.StringParameter.valueFromLookup()` is different from `ssm.StringParameter.valueForStringParameter()`:

```typescript
// valueFromLookup: resolves at synthesis time, caches in cdk.context.json
// → concrete value in the template; NEVER use for secrets!
const amiId = ssm.StringParameter.valueFromLookup(this, '/shared/ami-id');
// result: "ami-0abc123" (concrete string)

// valueForStringParameter: resolves at deploy time (CloudFormation)
// → generates a token {{resolve:ssm:/shared/ami-id}} in the template
// → runs an SSM call at deploy time, not synthesis time
const endpoint = ssm.StringParameter.valueForStringParameter(this, '/shared/db-endpoint');
// result: Token (resolved by CloudFormation)
```

[CONSENSUS] For parameters that change frequently (configurations, environment endpoints), prefer `valueForStringParameter` — the template always uses the most recent value when deployed. For stable parameters referenced in constructs that need the value at synthesis time (e.g., a VPC ID for subnet lookups), `valueFromLookup` is necessary.

---

### 3. Managing cdk.context.json in teams

[FACT] According to the official CDK documentation: `cdk.context.json` **must be versioned in the repository**. The reason is that CI environments frequently don't have (and shouldn't have) credentials with describe permissions on the production account. The cache is the reproducibility anchor.

[FACT] The `cdk context` command manages the cache:

```bash
# list all cache entries
cdk context

# remove a specific entry (forces re-lookup on next synthesis)
cdk context --reset "vpc-provider:account=123456789012:..."

# clear the entire cache
cdk context --clear

# force synthesis ignoring lookups (fails if any lookup isn't cached)
cdk synth --no-lookups
```

[CONSENSUS] The recommended policy by the CDK community for teams is:

```
cdk.json          → versioned (project configuration, feature flags)
cdk.context.json  → versioned (lookup cache — needed for CI)
cdk.out/          → gitignore (synthesis artifacts, automatically generated)
node_modules/     → gitignore (npm dependencies)
```

Never put `cdk.context.json` in `.gitignore`. If you do, every new developer and every CI build will do lookups on the account, which:
1. Requires credentials with elevated permissions in CI.
2. Makes the build non-deterministic (a new resource in the account can change the result).
3. Slows down synthesis (extra API calls).

---

### 4. Context as an explicit configuration mechanism

Besides automatic lookups, you can use Context as an intentional configuration system — passing values to the app via `cdk.json`, environment variables, or CLI:

```typescript
// lib/app.ts
const app = new App();

// Reads the target environment from context
// Can be passed via: cdk deploy --context env=production
const env = app.node.tryGetContext('env') ?? 'development';

const config = {
  development: { accountId: '111111111111', region: 'us-east-1', instanceType: 't3.micro' },
  staging:     { accountId: '222222222222', region: 'us-east-1', instanceType: 't3.small' },
  production:  { accountId: '333333333333', region: 'us-east-1', instanceType: 'm5.large' },
};

const targetEnv = config[env as keyof typeof config];
if (!targetEnv) throw new Error(`Invalid environment: ${env}`);

new ApiStack(app, `Api-${env}`, {
  env: { account: targetEnv.accountId, region: targetEnv.region },
  instanceType: new ec2.InstanceType(targetEnv.instanceType),
});
```

In `cdk.json`, you can define the default value:

```json
{
  "app": "npx ts-node --prefer-ts-exts bin/app.ts",
  "context": {
    "env": "development"
  }
}
```

[FACT] The Context precedence is, from highest to lowest:
1. `--context key=value` on the CLI command line
2. `~/.cdk.json` (user's global file)
3. `cdk.context.json` (project lookup cache)
4. `cdk.json` (project configuration)
5. `app.node.setContext()` in code

---

### 5. Feature Flags: what they are and when they matter

[FACT] Feature flags in CDK are Context booleans with a module prefix as namespace (e.g., `@aws-cdk/aws-s3:serverAccessLogsUseBucketPolicy`). They control behaviors that were **changed** in some CDK version but needed to be opt-in to avoid breaking existing stacks.

[FACT] In CDK v2, the historical feature flags from v1 were all enabled by default. But CDK v2 continues introducing new feature flags for changes that affect existing stacks. When you create a new project with `cdk init`, the generated `cdk.json` includes all feature flags from the current version activated — this ensures new projects use the most modern behavior.

Example of `cdk.json` generated by `cdk init` (version ~2.100+):

```json
{
  "app": "npx ts-node --prefer-ts-exts bin/myapp.ts",
  "watch": {
    "include": ["**"],
    "exclude": [
      "README.md",
      "cdk*.json",
      "**/*.d.ts",
      "**/*.js",
      "node_modules",
      "test"
    ]
  },
  "context": {
    "@aws-cdk/aws-lambda:recognizeLayerVersion": true,
    "@aws-cdk/core:checkSecretUsage": true,
    "@aws-cdk/core:target-partitions": ["aws", "aws-cn"],
    "@aws-cdk-containers/ecs-service-extensions:enableDefaultLogDriver": true,
    "@aws-cdk/aws-ec2:uniqueImdsv2TemplateName": true,
    "@aws-cdk/aws-ecs:arnFormatIncludesClusterName": true,
    "@aws-cdk/aws-iam:minimizePolicies": true,
    "@aws-cdk/core:validateSnapshotRemovalPolicy": true,
    "@aws-cdk/aws-codepipeline:crossAccountKeyAliasStackSafeResourceName": true,
    "@aws-cdk/aws-s3:createDefaultLoggingPolicy": true,
    "@aws-cdk/aws-sns-subscriptions:restrictSqsDescryption": true,
    "@aws-cdk/aws-apigateway:disableCloudWatchRole": true,
    "@aws-cdk/core:enablePartitionLiterals": true,
    "@aws-cdk/aws-events:eventsTargetQueueSameAccount": true,
    "@aws-cdk/aws-iam:standardizedServicePrincipals": true,
    "@aws-cdk/aws-ecs:disableExplicitDeploymentControllerForCircuitBreaker": true,
    "@aws-cdk/aws-iam:importedRoleStackSafeDefaultPolicyName": true,
    "@aws-cdk/aws-s3:serverAccessLogsUseBucketPolicy": true,
    "@aws-cdk/aws-route53-patters:useCertificate": true,
    "@aws-cdk/customresources:installLatestAwsSdkDefault": false,
    "@aws-cdk/aws-rds:databaseProxyUniqueResourceName": true,
    "@aws-cdk/aws-codedeploy:removeAlarmsFromDeploymentGroup": true,
    "@aws-cdk/aws-apigateway:authorizerChangeDeploymentLogicalId": true,
    "@aws-cdk/aws-ec2:launchTemplateDefaultUserData": true,
    "@aws-cdk/aws-secretsmanager:useAttachedSecretResourcePolicyForSecretTargetAttachments": true,
    "@aws-cdk/aws-redshift:columnId": true,
    "@aws-cdk/aws-cloudfront-origins:useOriginAccessControlForS3Origins": true,
    "@aws-cdk/core:enableCfnParameters": false
  }
}
```

[FACT] The flag `@aws-cdk/customresources:installLatestAwsSdkDefault` deserves attention: when `true`, the internal Lambda of `AwsCustomResource` uses the latest available AWS SDK (not the one bundled with the Node runtime). This can cause failures if the API you're calling changed. The default changed to `false` in recent versions for this reason — it's preferable to use the bundled and predictable SDK.

---

### 6. Production-grade cdk.json structure

[CONSENSUS] For team projects, `cdk.json` should be treated as a project configuration file, not merely a file generated by `cdk init`. A complete commented structure:

```json
{
  // CDK app entry command
  "app": "npx ts-node --prefer-ts-exts bin/app.ts",

  // watch mode configuration (cdk watch / cdk deploy --watch)
  "watch": {
    "include": ["**"],
    "exclude": [
      "README.md",
      "cdk*.json",
      "**/*.d.ts",
      "**/*.js",
      "node_modules",
      "test"
    ]
  },

  // build configuration before cdk deploy/diff/synth
  // "build": "npm run build",  ← uncomment if not using ts-node directly

  // all context goes here
  "context": {
    // ---- Project configuration ----
    "env": "development",          // overridden via --context in CI/CD
    "owner": "platform-team",      // metadata for tagging
    "costCenter": "infra-001",

    // ---- CDK feature flags ----
    // (the flags below are the defaults for new projects)
    "@aws-cdk/aws-iam:minimizePolicies": true,
    "@aws-cdk/aws-s3:serverAccessLogsUseBucketPolicy": true,
    "@aws-cdk/customresources:installLatestAwsSdkDefault": false,
    // ... other flags as per the CDK version used

    // ---- Cached lookups ----
    // (automatically populated by CDK — don't edit manually)
    // "vpc-provider:account=...": { ... }
  }
}
```

[CONSENSUS] Separating project configuration context (like `env`, `owner`) from feature flags improves readability, but it's not a structure enforced by CDK — it's just organization by comments.

**What goes in `.gitignore`:**

```gitignore
# Synthesis artifacts (generated by cdk synth)
cdk.out/

# Node dependencies
node_modules/

# Compiled TypeScript (if not using ts-node directly)
# *.js
# *.d.ts
```

**What does NOT go in `.gitignore`:**

```
cdk.json          ← project configuration, feature flags
cdk.context.json  ← lookup cache (critical for reproducible CI)
```

---

## Practical Example

**Scenario:** You have a multi-environment CDK app that needs to:
1. Use the existing VPC from each account (not create a new one).
2. Have deterministic behavior in CI without describe credentials.
3. Support `cdk deploy --context env=staging` to deploy to staging without changing code.

### Project structure

```
bin/
  app.ts                 ← entry point, reads context 'env'
lib/
  stacks/
    api-stack.ts         ← uses Vpc.fromLookup()
config/
  environments.ts        ← configuration map per environment
cdk.json                 ← feature flags + default env
cdk.context.json         ← lookup cache (versioned)
```

### `config/environments.ts`

```typescript
import { Environment } from 'aws-cdk-lib';

export interface EnvConfig {
  env: Environment;
  vpcName: string;
  hostedZoneName: string;
  instanceType: string;
}

export const environments: Record<string, EnvConfig> = {
  development: {
    env: { account: '111111111111', region: 'us-east-1' },
    vpcName: 'dev-vpc',
    hostedZoneName: 'dev.example.com',
    instanceType: 't3.micro',
  },
  staging: {
    env: { account: '222222222222', region: 'us-east-1' },
    vpcName: 'staging-vpc',
    hostedZoneName: 'staging.example.com',
    instanceType: 't3.small',
  },
  production: {
    env: { account: '333333333333', region: 'us-east-1' },
    vpcName: 'prod-vpc',
    hostedZoneName: 'example.com',
    instanceType: 'm5.large',
  },
};
```

### `lib/stacks/api-stack.ts`

```typescript
import { Stack, StackProps } from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as route53 from 'aws-cdk-lib/aws-route53';
import { Construct } from 'constructs';
import { EnvConfig } from '../../config/environments';

interface ApiStackProps extends StackProps {
  config: EnvConfig;
}

export class ApiStack extends Stack {
  constructor(scope: Construct, id: string, props: ApiStackProps) {
    super(scope, id, props);

    // Lookup of the existing VPC in the account
    // → on first run: CDK queries AWS and saves to cdk.context.json
    // → on subsequent runs (including CI): reads from cache
    const vpc = ec2.Vpc.fromLookup(this, 'Vpc', {
      vpcName: props.config.vpcName,
    });

    // Hosted Zone lookup
    const zone = route53.HostedZone.fromLookup(this, 'Zone', {
      domainName: props.config.hostedZoneName,
    });

    // The rest uses vpc and zone as if they were normal constructs
    // (even though they're lookups, they have the expected methods/properties)
  }
}
```

### `bin/app.ts`

```typescript
import { App } from 'aws-cdk-lib';
import { ApiStack } from '../lib/stacks/api-stack';
import { environments } from '../config/environments';

const app = new App();

const envName = app.node.tryGetContext('env') ?? 'development';
const config = environments[envName];

if (!config) {
  throw new Error(
    `Environment '${envName}' not found. ` +
    `Available environments: ${Object.keys(environments).join(', ')}`
  );
}

new ApiStack(app, `Api-${envName}`, {
  env: config.env,
  config,
  // Global tags applied to all resources in this stack
  tags: {
    Environment: envName,
    ManagedBy: 'CDK',
  },
});
```

### New environment onboarding workflow

```bash
# 1. First deploy to staging: CDK will do lookups on the staging account
AWS_PROFILE=staging-profile cdk deploy --context env=staging

# → CDK queries AWS, populates cdk.context.json with staging entries
# → commit the updated cdk.context.json

git add cdk.context.json
git commit -m "chore: cache context lookups for staging environment"

# 2. CI builds for staging are now reproducible without describe credentials
# (CI uses the cached entries)
```

---

## Common Pitfalls

### Pitfall 1: Putting cdk.context.json in .gitignore

**The error:** A developer sees `cdk.context.json` as a "generated file" and adds it to `.gitignore`. The project works locally because every dev has AWS credentials. CI starts failing with errors like:

```
Error: Cannot retrieve value from context provider vpc-provider since account/region
are not specified at the stack level. Configure "env" for the stack...
```

Or worse: CI works but produces different templates on different days because a new resource appeared in the account (new AMI, new AZ).

**Why it happens:** Without the versioned cache, CI needs describe credentials to do real-time lookups, and those lookups are non-deterministic.

**How to avoid:** `cdk.context.json` always in git. The only legitimate time not to version it is in strictly local proof-of-concept projects.

---

### Pitfall 2: Stale lookup cache causing silent errors

**The error:** You deleted and recreated the VPC `prod-vpc` on AWS (for whatever reason). The `cdk.context.json` still has the old subnet IDs. `cdk synth` passes without error (uses the cache), the deploy starts, and mid-deploy CloudFormation tries to reference `subnet-old123` which no longer exists.

**Why it happens:** CDK never automatically invalidates the cache — it's a write-once cache, invalidated only manually. It has no way to know the external resource changed.

**How to recognize:** Deploy errors like `subnet-id does not exist` or `security group sg-XXX not found` when you haven't changed the CDK code.

**How to avoid:** After any manual infrastructure change that affects lookups (recreating VPCs, changing resource names, etc.), run `cdk context --clear` locally, re-synthesize, and commit the updated `cdk.context.json`.

---

### Pitfall 3: New feature flag in CDK version silently changes behavior

**The error:** You update `aws-cdk-lib` from `2.80.0` to `2.120.0`. The `package.json` and `package-lock.json` are updated, but `cdk.json` doesn't receive the new feature flags from version 2.120.0. Some constructs start having different behavior than expected (a new IAM resource appears, the naming logic of a resource changes), but no error is emitted.

**Why it happens:** New feature flags are not retroactively added to the `cdk.json` of existing projects — only to the one generated by `cdk init` with the new version. Existing projects keep the configuration from the project creation version.

**How to recognize:** After a version update, run `cdk diff` in a non-production environment and review every change. If there are unexpected changes (new resources, name changes), check the changelogs and feature flags of the version.

**How to avoid:** When updating CDK, compare your project's `cdk.json` with the `cdk.json` that `cdk init` would generate in the new version. The version documentation lists the new feature flags introduced. Activate them deliberately after understanding the impact.

---

## Reflection Exercise

You're migrating a CDK app from a single developer to a team repository with an automated CI/CD pipeline. The app has three stacks, and two of them use `Vpc.fromLookup()` and `MachineImage.lookup()`. CI runs in a Docker container without access to production AWS accounts (per security policy — only the deploy pipeline has the credentials, not the CI build).

How would you structure the process to ensure that:
1. CI produces templates identical to what the developer produced locally.
2. When the underlying infrastructure changes (e.g., the VPC is recreated), there's a defined and controlled process to update the cache.
3. New developers don't need production account credentials to work on the code.
4. Feature flags are updated deliberately and reviewed when CDK is updated.

Describe the commit workflow, the versioned and non-versioned files, and the cache update process you would adopt.

---

## Resources for Further Study

### 1. Context values and the AWS CDK
**URL:** https://docs.aws.amazon.com/cdk/v2/guide/context.html
**What to find:** Complete documentation of the Context mechanism: available lookup types, how the cache works, value precedence, `cdk context` commands, and the distinction between `valueFromLookup` and `valueForStringParameter`.
**Why it's the right source:** It's the official and canonical mechanism guide — more complete than any blog post.

### 2. AWS CDK feature flags
**URL:** https://docs.aws.amazon.com/cdk/v2/guide/featureflags.html
**What to find:** Complete list of all CDK v2 feature flags, what each one controls, when it was introduced, and the behavior with and without the flag. Essential when migrating or updating CDK versions.
**Why it's the right source:** It's the authoritative reference — CDK changelogs on GitHub reference this document for each introduced flag.

### 3. CDK Best Practices — Context and project
**URL:** https://docs.aws.amazon.com/cdk/v2/guide/best-practices.html
**What to find:** Sections on project structure for teams, context management in CI/CD pipelines, and the policy of what to version vs what to ignore in `.gitignore`.
**Why it's the right source:** It's the official best practices guide from the CDK team, consolidating lessons learned from users in production.

---
