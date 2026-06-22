# Session 010 — CDK Pipelines: Custom Stages, ShellSteps, and self-mutation in action

**Estimated duration:** 60 minutes
**Prerequisites:** session-009 — CDK Pipelines: cross-account bootstrap, OIDC connection, and pipeline structure

---

## Objective

By the end, you will be able to add a Stage with multiple stacks in sequence and in parallel, insert validation ShellSteps between stages (e.g., post-deploy smoke tests), observe the self-mutation cycle (the pipeline re-executes itself when the pipeline code changes before any application deploy), and debug a deploy that fails during the asset publishing phase.

---

## Context

[FACT] The previous session covered the minimal pipeline structure (Source → Build → UpdatePipeline). This session goes deeper into the blocks that come after: how to organize stages, how to add validations between them, and how to use outputs from deployed stacks in subsequent steps.

[CONSENSUS] The recommended pattern for production pipelines is: deploy to dev → automated smoke tests → manual approval → deploy to prod → automated smoke tests. CDK Pipelines has native support for each of these steps via `pre`, `post`, `ShellStep`, and `ManualApprovalStep`.

---

## Key Concepts

### 1. Sequential and parallel stages — addStage and Wave

By default, stages added with `addStage` are executed **sequentially**:

```typescript
// Sequential: dev finishes before staging begins
pipeline.addStage(new AppStage(this, 'Dev',     { env: devEnv }));
pipeline.addStage(new AppStage(this, 'Staging', { env: stagingEnv }));
pipeline.addStage(new AppStage(this, 'Prod',    { env: prodEnv }));
```

For **parallel** stages, use `Wave`:

```typescript
// Wave: eu-west-1 and ap-southeast-1 deploy at the same time
const multiRegionWave = pipeline.addWave('MultiRegionDeploy');

multiRegionWave.addStage(new AppStage(this, 'ProdEU', {
  env: { account: PROD_ACCOUNT, region: 'eu-west-1' },
}));

multiRegionWave.addStage(new AppStage(this, 'ProdAP', {
  env: { account: PROD_ACCOUNT, region: 'ap-southeast-1' },
}));

// After the wave, a sequential stage waits for both to finish
pipeline.addStage(new MonitoringStage(this, 'GlobalMonitoring', {
  env: { account: TOOLS_ACCOUNT, region: 'us-east-1' },
}));
```

**Execution diagram:**

```
Sequential:                         Wave (parallel):
  Dev  ──► Staging ──► Prod           ┌── ProdEU ──┐
  (one at a time)                     │             ├──► GlobalMonitoring
                                      └── ProdAP ──┘
                                      (simultaneous)
```

**Stack ordering within a Stage:**

[FACT] Within a Stage with multiple stacks, the CDK automatically determines the order based on dependencies between stacks. Independent stacks are deployed in parallel; stacks with dependencies are ordered correctly.

```typescript
export class AppStage extends cdk.Stage {
  constructor(scope: Construct, id: string, props: cdk.StageProps) {
    super(scope, id, props);

    const network = new NetworkStack(this, 'Network');
    const data    = new DataStack(this, 'Data', { vpc: network.vpc });
    const app     = new AppStack(this, 'App',  { vpc: network.vpc, table: data.table });
    // CDK infers: Network → (Data and App in parallel, then App waits for Data)
    // Network deploys first; Data and App deploy after in parallel if there's no dep between them
  }
}
```

To force explicit dependency between stacks in the same Stage:

```typescript
app.addDependency(data);   // ensures Data finishes before App starts
```

---

### 2. ShellStep — validations between stages

`ShellStep` is the fundamental building block for inserting shell commands at any point in the pipeline. It can be used as `pre` (before the stage deploy) or `post` (after the deploy).

```typescript
import * as pipelines from 'aws-cdk-lib/pipelines';

// Pre-step: runs before the stage deploy
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
        // Accesses the application URL and verifies the health endpoint
        'curl -f $APP_URL/health || exit 1',
        'echo "Smoke test passed"',
      ],
      envFromCfnOutputs: {
        APP_URL: appStageRef.appUrlOutput,  // stack output (see section 4)
      },
    }),
  ],
});
```

**Common use cases for ShellStep:**

```
PRE-DEPLOY:
  ✅ Lint and unit tests before deploying
  ✅ Security validation (checkov, cfn-nag, cfn-guard)
  ✅ Documentation or artifact generation
  ✅ ManualApprovalStep (special type of pre-step)

POST-DEPLOY:
  ✅ Smoke tests (HTTP health checks, ping endpoints)
  ✅ Integration tests against the newly deployed environment
  ✅ Notifications (Slack, PagerDuty) of successful deploy
  ✅ CDN cache invalidation
  ✅ Deploy registry update (JIRA, Backstage)
```

---

### 3. CodeBuildStep — ShellStep with more control

`CodeBuildStep` is a more powerful version of `ShellStep` that gives access to CodeBuild configurations: custom image, additional IAM policies, environment variables, and timeout.

```typescript
import * as codebuild from 'aws-cdk-lib/aws-codebuild';

const integrationTest = new pipelines.CodeBuildStep('IntegrationTest', {
  commands: [
    'npm ci',
    'npm run test:integration',
  ],

  // Static environment variables
  env: {
    ENVIRONMENT: 'dev',
    LOG_LEVEL: 'debug',
  },

  // Custom CodeBuild image
  buildEnvironment: {
    buildImage: codebuild.LinuxBuildImage.STANDARD_7_0,
    computeType: codebuild.ComputeType.MEDIUM,
    privileged: false,
  },

  // Additional IAM policies for the CodeBuild project
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

  // CodeBuild project timeout
  timeout: cdk.Duration.minutes(30),

  // Variables from deployed stack outputs (see section 4)
  envFromCfnOutputs: {
    API_ENDPOINT: apiStack.endpointOutput,
  },
});

pipeline.addStage(new AppStage(this, 'Dev', { env: devEnv }), {
  post: [integrationTest],
});
```

---

### 4. Outputs from deployed stacks in ShellSteps

A very common pattern is: deploy the stack → get the endpoint/URL of what was created → use it in the smoke test. CDK Pipelines has native support via `envFromCfnOutputs`.

**Step 1 — expose the output in the Stage:**

```typescript
// lib/app-stage.ts
export class AppStage extends cdk.Stage {
  // Expose outputs that will be used in steps
  public readonly apiEndpointOutput: CfnOutput;
  public readonly loadBalancerDnsOutput: CfnOutput;

  constructor(scope: Construct, id: string, props: cdk.StageProps) {
    super(scope, id, props);

    const appStack = new AppStack(this, 'App');

    // The outputs are CfnOutput from the stack
    this.apiEndpointOutput = appStack.apiEndpoint;       // CfnOutput defined in AppStack
    this.loadBalancerDnsOutput = appStack.lbDnsName;     // CfnOutput defined in AppStack
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

**Step 2 — use the outputs in the pipeline:**

```typescript
// lib/pipeline-stack.ts
const devStage = new AppStage(this, 'Dev', { env: devEnv });

pipeline.addStage(devStage, {
  post: [
    new pipelines.ShellStep('SmokeTests', {
      commands: [
        // API_ENDPOINT and LB_DNS are automatically populated
        // with the CfnOutput values after the deploy
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

**What CDK does under the hood:**

[FACT] The `envFromCfnOutputs` creates an implicit dependency: the ShellStep only starts after the entire stage is deployed, and the CodeBuild project receives the variables via CodePipeline (which reads the outputs from the deployed CloudFormation stack).

---

### 5. Self-mutation in action — observing the cycle

This section documents the self-mutation behavior so you can recognize what's happening when the pipeline restarts.

**Scenario: you add a new Stage to the pipeline**

```
State before: pipeline has Dev and Prod
You add Staging between Dev and Prod and push
```

```
Execution #N (with the new code):

  Source:          ✅ Downloads code with the new Staging stage
  Build:           ✅ cdk synth → cloud assembly with Dev + Staging + Prod
  UpdatePipeline:  ⚠️  Current pipeline: Dev → Prod
                       Cloud assembly: Dev → Staging → Prod
                       DIFFERENT → applies change → RESTARTS

Execution #N+1 (same branch, updated pipeline):

  Source:          ✅ Same code
  Build:           ✅ Same cloud assembly
  UpdatePipeline:  ✅ Current pipeline = cloud assembly → no change → PROCEEDS

  Assets:          Upload assets
  Dev:             Deploy to dev account
  SmokeTests:      Post-deploy dev tests
  Staging:         Deploy to staging account  ← new stage working
  Approve:         Manual approval
  Prod:            Deploy to prod account
```

**How to observe the restart in the Console:**

```
CodePipeline → MyAppPipeline → Executions
  Execution #N:   Status: Superseded  ← was replaced by the restart
  Execution #N+1: Status: Succeeded   ← the one that actually executed everything
```

[FACT] When the pipeline restarts due to self-mutation, the previous execution gets the status `Superseded` (not `Failed`). If you see `Superseded`, the pipeline worked correctly — it's not an error.

**Forcing a manual restart:**

```bash
# Equivalent to the "Release Change" button in the console
aws codepipeline start-pipeline-execution \
  --name MyAppPipeline \
  --profile pipeline
```

---

### 6. Debugging failures in asset publishing

The asset publishing phase (`Assets` stage) is where CDK uploads Lambda code and Docker images. Failures here have specific causes.

**Assets stage structure:**

```
Assets
  ├── Publish-Asset-HASH_A (Lambda zip)
  ├── Publish-Asset-HASH_B (another Lambda zip)
  └── Publish-Asset-HASH_C (Docker image)
```

[FACT] Each asset has its own parallel CodeBuild action. A failure in one does not immediately cancel the others — they can fail in parallel.

**Common errors and diagnosis:**

**Error 1: `AccessDenied` when uploading to S3**

```
Error: AccessDenied: Access Denied (Service: S3, Status Code: 403)

Cause: The CodeBuild for asset publishing doesn't have permission to write
       to the bootstrap bucket of the target account.

Diagnosis:
  1. Verify the target account was bootstrapped with --trust
  2. Verify the role cdk-XXXX-file-publishing-role exists in the target account
  3. Verify the role's trust policy includes the pipeline account

Fix:
  cdk bootstrap aws://DEST_ACCOUNT/REGION \
    --profile dest \
    --trust PIPELINE_ACCOUNT \
    --cloudformation-execution-policies arn:aws:iam::aws:policy/AdministratorAccess
```

**Error 2: Docker not available for image assets**

```
Error: Cannot connect to the Docker daemon at unix:///var/run/docker.sock

Cause: The CodeBuild project for asset publishing doesn't have Docker enabled.

Fix in CDK:
  const pipeline = new pipelines.CodePipeline(this, 'Pipeline', {
    synth,
    assetPublishingCodeBuildDefaults: {
      buildEnvironment: {
        privileged: true,  // enables Docker in CodeBuild
      },
    },
  });

⚠️  If the pipeline is already deployed:
  1. Set privileged: true
  2. Push and wait for self-mutation to apply the change
  3. Only then add the Docker asset
  Reason: changing privileged on an existing pipeline requires recreating the
          CodeBuild project via self-mutation before using Docker
```

**Error 3: `Cannot find module` in Lambda after deploy**

```
Error: Runtime.ImportModuleError: Cannot find module 'sharp'

Cause: Module with native binary was bundled by esbuild for the wrong
       architecture (see session 007 — externalModules vs nodeModules).

Diagnosis:
  1. Check if 'sharp' is in externalModules (wrong) or nodeModules (correct)
  2. Check if the NodejsFunction architecture matches the compiled module

Fix:
  bundling: {
    nodeModules: ['sharp'],   // compiles inside Docker for Amazon Linux
  }
```

**Inspecting logs from a failed asset publishing:**

```bash
# Find the build ID of the failed CodeBuild
aws codepipeline get-pipeline-state \
  --name MyAppPipeline \
  --profile pipeline \
  --query 'stageStates[?stageName==`Assets`].actionStates[?currentRevision!=null].latestExecution.externalExecutionId' \
  --output text

# View the CodeBuild logs
aws codebuild batch-get-builds \
  --ids BUILD_ID \
  --profile pipeline \
  --query 'builds[0].logs.deepLink'
# Opens CloudWatch Logs with the complete build logs
```

---

### 7. Complete pipeline with all features

```typescript
// lib/pipeline-stack.ts
export class PipelineStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props: cdk.StackProps) {
    super(scope, id, props);

    const source = pipelines.CodePipelineSource.connection(
      'my-org/my-repo', 'main',
      { connectionArn: CONNECTION_ARN }
    );

    const synth = new pipelines.ShellStep('Synth', {
      input: source,
      commands: ['npm ci', 'npm run build', 'npx cdk synth'],
    });

    const pipeline = new pipelines.CodePipeline(this, 'Pipeline', {
      pipelineName: 'MyAppPipeline',
      synth,
      selfMutation: true,
      publishAssetsInParallel: true,
      codeBuildDefaults: {
        buildEnvironment: {
          buildImage: codebuild.LinuxBuildImage.STANDARD_7_0,
        },
      },
    });

    // ── Dev Stage ─────────────────────────────────────────────────────
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

    // ── Wave: Multi-region staging (parallel) ─────────────────────────
    const stagingWave = pipeline.addWave('Staging', {
      pre: [new pipelines.ManualApprovalStep('ApproveStaging')],
    });

    stagingWave.addStage(new AppStage(this, 'StagingUS', {
      env: { account: STAGING_ACCOUNT, region: 'us-east-1' },
    }));

    stagingWave.addStage(new AppStage(this, 'StagingEU', {
      env: { account: STAGING_ACCOUNT, region: 'eu-west-1' },
    }));

    // ── Prod Stage ────────────────────────────────────────────────────
    const prodStage = new AppStage(this, 'Prod', { env: PROD_ENV });

    pipeline.addStage(prodStage, {
      pre: [
        new pipelines.ManualApprovalStep('ApproveProd'),
        // Automatic gating: blocks if IAM permissions broadened
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

## Common Pitfalls

**1. `envFromCfnOutputs` with output from the wrong stack**

If you pass a `CfnOutput` from a stack that was not deployed **in this stage**, the pipeline fails with an invalid reference error. The `envFromCfnOutputs` only accepts outputs from stacks within the stage that precedes the step. To use an output from another stage, you need to pass it via SSM Parameter Store or Secrets Manager.

**2. Wave with stages that have cross-dependencies**

Stages within a Wave are deployed in parallel — which presupposes they are independent. If Stage A depends on an output from Stage B (both in the same Wave), this creates a circular dependency that the CDK detects with an error. Move dependent stages outside the Wave.

**3. Modifying `privileged` on an existing pipeline without doing self-mutation first**

If you enable Docker assets on a project that already has a deployed pipeline, but don't wait for self-mutation to apply `privileged: true` before adding the Docker asset, the CodeBuild project will still have `privileged: false` on the execution where the asset appears for the first time. The failure will happen on that execution. The solution: make two commits — first one with only `privileged: true`, wait for the pipeline to self-mutate; then a second commit with the Docker asset.

---

## Reflection Exercise

You have a pipeline with dev → staging → prod. The smoke tests for the `dev` stage are failing intermittently: 70% of the time they pass, 30% they fail with `curl: (7) Failed to connect`. You suspect the tests begin before the application is fully ready after the deploy.

What are the possible strategies to solve this? Consider at least three different approaches (from ShellStep adjustments to changes in the health check architecture). For each one, describe the trade-off between implementation complexity, reliability, and impact on total pipeline time. Which would you implement first and why?

---

## Resources for Further Study

**Complete Pipelines readme:**
- [aws-cdk-lib.pipelines module](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.pipelines-readme.html) — complete reference for ShellStep, CodeBuildStep, Wave, envFromCfnOutputs, ConfirmPermissionsBroadening, and all configuration options of the CodePipeline construct.

**CDK Pipelines guide:**
- [Continuous integration and delivery (CI/CD) using CDK Pipelines](https://docs.aws.amazon.com/cdk/v2/guide/cdk_pipeline.html) — troubleshooting section covers the most common asset publishing and self-mutation errors.

**Original blog post:**
- [CDK Pipelines: Continuous Delivery for AWS CDK Applications](https://aws.amazon.com/blogs/developer/cdk-pipelines-continuous-delivery-for-aws-cdk-applications/) — complete walk-through with the reasoning behind the self-mutation design and the Source → Build → UpdatePipeline progression.

---
