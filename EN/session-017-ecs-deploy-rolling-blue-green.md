# Session — ECS: Deploy strategies — rolling update and blue/green with CodeDeploy

**Estimated duration:** 60 minutes
**Prerequisites:** session-016-ecs-capacity-providers-autoscaling, session-010-cdk-pipelines-stages-shellsteps

---

## Objective

By the end, you will be able to configure an ECS Service with deployment type `CODE_DEPLOY`, write the AppSpec for blue/green deployment, implement test hooks after the traffic shift, and configure automatic rollback based on CloudWatch Alarms.

---

## Context

[FACT] ECS supports three deployment types: **rolling update** (default, managed directly by ECS), **blue/green with CodeDeploy** (`CODE_DEPLOY`), and **external** (for third-party tool integrations). Rolling update is adequate for most services; blue/green is necessary when you need explicit verification before redirecting production traffic, zero-downtime with instant rollback capability, or testing in a production environment before the complete traffic shift.

[FACT] CodeDeploy's blue/green model for ECS is fundamentally different from rolling update: instead of gradually replacing tasks *within the same target group*, CodeDeploy creates a completely separate **replacement task set** (green), tests it, and then switches the ALB from pointing to the original task set (blue) to the green. After the stabilization period, the blue is destroyed. Rollback means switching the ALB back to blue — a seconds-long operation, without needing to create new tasks.

[CONSENSUS] Blue/green has higher operational cost than rolling update: it requires more components (two target groups, test listener, CodeDeploy deployment group, additional IAM role), doubles task capacity during deployment, and adds configuration complexity. It's justified for critical production services where the cost of a bad deploy (downtime, slow rollback) is high. For internal or less critical services, rolling update with circuit breaker is generally sufficient.

---

## Key Concepts

### 1. Blue/green architecture in ECS

[FACT] Blue/green deployment for ECS requires the following additional components compared to a rolling update service:

```
┌─────────────────────────────────────────────────────────────────┐
│  ALB                                                            │
│                                                                 │
│  Listener :443 (production)          Listener :8080 (test)     │
│       │                                   │                     │
│       ▼                                   ▼                     │
│  Target Group BLUE (active)    Target Group GREEN (new)        │
│  [tasks previous version]      [tasks new version]             │
└─────────────────────────────────────────────────────────────────┘

Before traffic shift:
  :443 → TG-BLUE (100% production traffic)
  :8080 → TG-GREEN (test traffic only)

During traffic shift (canary):
  :443 → TG-BLUE (90%) + TG-GREEN (10%)
  :8080 → TG-GREEN (100% test traffic)

After complete traffic shift:
  :443 → TG-GREEN (100%)
  :8080 → N/A (can be destroyed)

After stabilization (termination wait):
  TG-BLUE and its tasks are destroyed
```

[FACT] Required components:

1. **Two ALB Target Groups**: blue (active) and green (for the new task set).
2. **Two ALB Listeners**: production port (e.g., 443) and test port (e.g., 8080). The test listener is optional but recommended for validation hooks.
3. **CodeDeploy Application + Deployment Group**: with `computePlatform: ECS`.
4. **IAM Role for CodeDeploy**: `AWSCodeDeployRoleForECS` managed policy.
5. **ECS Service with `deploymentController: CODE_DEPLOY`**.

---

### 2. The AppSpec file

[FACT] The AppSpec is a YAML or JSON file that describes *what* to deploy and *how* to orchestrate the deployment. For ECS, it has two main blocks:

```yaml
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        # ARN or name of the task definition to deploy (new version)
        TaskDefinition: "arn:aws:ecs:us-east-1:123456789012:task-definition/api-service:42"
        LoadBalancerInfo:
          ContainerName: "api"        # container that receives traffic (portMappings)
          ContainerPort: 3000
        # Optional: network configuration override for the service
        # PlatformVersion: "LATEST"
        # NetworkConfiguration:
        #   AwsvpcConfiguration:
        #     Subnets: ["subnet-abc", "subnet-def"]
        #     SecurityGroups: ["sg-xyz"]
        #     AssignPublicIp: "DISABLED"
        # Optional: Capacity Provider strategy override
        # CapacityProviderStrategy:
        #   - CapacityProvider: "FARGATE"
        #     Base: 2
        #     Weight: 1

Hooks:
  # Hook BEFORE the ALB sends any production traffic to green
  - BeforeAllowTraffic:
      - location: "arn:aws:lambda:us-east-1:123456789012:function:PreTrafficHook"
        timeout: 300        # seconds — max 3600

  # Hook AFTER the ALB has directed production traffic to green
  - AfterAllowTraffic:
      - location: "arn:aws:lambda:us-east-1:123456789012:function:PostTrafficHook"
        timeout: 300
```

[FACT] Complete cycle of hooks available for ECS blue/green (in execution order):

```
1. BeforeInstall
   → before creating the replacement task set (green)
   → rarely used for ECS

2. AfterInstall
   → after the green task set is created and tasks are RUNNING
   → before any traffic

3. AfterAllowTestTraffic
   → after the TEST listener (:8080) points to green
   → before production traffic shift
   → IDEAL LOCATION for automated smoke tests

4. BeforeAllowTraffic
   → after smoke tests, before production traffic shift
   → final validation

5. AfterAllowTraffic
   → after production traffic shift is complete
   → metric monitoring, success alerts
```

[FACT] A hook is a Lambda that **must** call `codedeploy:PutLifecycleEventHookExecutionStatus` with `status: Succeeded` or `status: Failed`. If the Lambda doesn't call this API within the `timeout`, CodeDeploy considers the hook as failed and initiates rollback.

```python
# Lambda hook structure
import boto3

codedeploy = boto3.client('codedeploy')

def handler(event, context):
    deployment_id = event['DeploymentId']
    hook_execution_id = event['LifecycleEventHookExecutionId']
    
    try:
        # Execute tests here
        run_smoke_tests()
        
        # Signal success to CodeDeploy to continue
        codedeploy.put_lifecycle_event_hook_execution_status(
            deploymentId=deployment_id,
            lifecycleEventHookExecutionId=hook_execution_id,
            status='Succeeded'
        )
    except Exception as e:
        print(f"Hook failed: {e}")
        codedeploy.put_lifecycle_event_hook_execution_status(
            deploymentId=deployment_id,
            lifecycleEventHookExecutionId=hook_execution_id,
            status='Failed'
        )
        raise
```

---

### 3. Traffic shifting strategies

[FACT] CodeDeploy offers predefined and customizable traffic shifting configurations for ECS:

```
Predefined configurations (ECS):
  CodeDeployDefault.ECSAllAtOnce
    → 100% traffic immediately to green
    → No validation period with partial traffic
    → Manual rollback or by alarm

  CodeDeployDefault.ECSCanary10Percent5Minutes
    → 10% to green for 5 minutes, then 100%
    → Allows monitoring errors with real traffic before full shift

  CodeDeployDefault.ECSCanary10Percent15Minutes
    → 10% for 15 minutes, then 100%

  CodeDeployDefault.ECSLinear10PercentEvery1Minutes
    → +10% every 1 minute until 100% (10 increments)
    → More gradual, larger problem detection window

  CodeDeployDefault.ECSLinear10PercentEvery3Minutes
    → +10% every 3 minutes until 100% (30 minutes total)
```

[FACT] Custom traffic shifting configuration (via CDK or CLI):

```typescript
// CDK — custom deployment config
import * as codedeploy from 'aws-cdk-lib/aws-codedeploy';

const deploymentConfig = new codedeploy.EcsDeploymentConfig(this, 'Config', {
  trafficRouting: codedeploy.TrafficRouting.timeBasedCanary({
    interval: Duration.minutes(5),   // wait 5 min between increments
    percentage: 20,                   // 20% on the first increment
    // → 20% for 5 min, then 100%
  }),
  // Alternative: linear
  // trafficRouting: codedeploy.TrafficRouting.timeBasedLinear({
  //   interval: Duration.minutes(2),
  //   percentage: 10,  // +10% every 2 minutes
  // }),
});
```

---

### 4. Automatic rollback via CloudWatch Alarms

[FACT] CodeDeploy can monitor CloudWatch Alarms during deployment and trigger automatic rollback if any alarm enters the `ALARM` state. Alarms are associated with the deployment group, not the AppSpec.

```
Deployment group
│
├── autoRollback:
│   ├── onDeploymentFailure: true     (hook returned Failed)
│   ├── onAlarmThreshold: true        (CW Alarm triggered)
│   └── alarms:
│       ├── "HighErrorRate"           (e.g., 5xx > 5% for 2 min)
│       └── "HighLatency"            (e.g., P99 > 2s for 2 min)
│
└── terminationWait: 60 minutes       (time before destroying blue)
```

[FACT] The `terminationWait` is the period after the complete traffic shift during which CodeDeploy waits before destroying the blue task set. During this period, blue still exists and can be used for instant rollback. The default is 0 minutes (destroys immediately). In production, 30-60 minutes is recommended to have a post-deploy rollback window.

---

## Practical Example

**Complete scenario:** A critical `api-service` with:
- Blue/green via CodeDeploy with canary 10% for 5 minutes.
- `AfterAllowTestTraffic` hook that validates the green health check via test listener.
- Automatic rollback if `HighErrorRate` alarm triggers.
- Automatic rollback if the hook fails.

### Complete CDK

```typescript
import { Stack, StackProps, Duration } from 'aws-cdk-lib';
import * as ecs from 'aws-cdk-lib/aws-ecs';
import * as codedeploy from 'aws-cdk-lib/aws-codedeploy';
import * as elbv2 from 'aws-cdk-lib/aws-elasticloadbalancingv2';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as cloudwatch from 'aws-cdk-lib/aws-cloudwatch';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as iam from 'aws-cdk-lib/aws-iam';
import { Construct } from 'constructs';

export class BlueGreenStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    const vpc = ec2.Vpc.fromLookup(this, 'Vpc', { vpcName: 'prod-vpc' });
    const cluster = ecs.Cluster.fromClusterAttributes(this, 'Cluster', {
      clusterName: 'prod-cluster', vpc, securityGroups: [],
    });

    // ─── ALB with two target groups and two listeners ───────────────────────

    const alb = new elbv2.ApplicationLoadBalancer(this, 'ALB', {
      vpc, internetFacing: true,
    });

    // Target Group BLUE (active, starts with tasks from current version)
    const blueTG = new elbv2.ApplicationTargetGroup(this, 'BlueTG', {
      vpc,
      protocol: elbv2.ApplicationProtocol.HTTP,
      port: 3000,
      targetType: elbv2.TargetType.IP,
      healthCheck: {
        path: '/health',
        interval: Duration.seconds(15),
        healthyThresholdCount: 2,
      },
      deregistrationDelay: Duration.seconds(30),
    });

    // Target Group GREEN (empty, will be filled by CodeDeploy during deploy)
    const greenTG = new elbv2.ApplicationTargetGroup(this, 'GreenTG', {
      vpc,
      protocol: elbv2.ApplicationProtocol.HTTP,
      port: 3000,
      targetType: elbv2.TargetType.IP,
      healthCheck: {
        path: '/health',
        interval: Duration.seconds(15),
        healthyThresholdCount: 2,
      },
      deregistrationDelay: Duration.seconds(30),
    });

    // Production listener (:443) → starts pointing to BLUE
    const prodListener = alb.addListener('ProdListener', {
      port: 443,
      defaultTargetGroups: [blueTG],
      open: true,
    });

    // Test listener (:8080) → used by hooks for validation
    const testListener = alb.addListener('TestListener', {
      port: 8080,
      defaultTargetGroups: [greenTG],
      open: false,  // restricted (internal access only)
    });

    // ─── ECS Service with deploymentController: CODE_DEPLOY ────────────────

    const taskDef = new ecs.FargateTaskDefinition(this, 'TaskDef', {
      cpu: 512,
      memoryLimitMiB: 1024,
    });
    taskDef.addContainer('api', {
      image: ecs.ContainerImage.fromRegistry('my-org/api:v1'),
      portMappings: [{ containerPort: 3000 }],
      logging: ecs.LogDrivers.awsLogs({ streamPrefix: 'api' }),
    });

    const service = new ecs.FargateService(this, 'Service', {
      cluster,
      taskDefinition: taskDef,
      desiredCount: 3,
      deploymentController: {
        type: ecs.DeploymentControllerType.CODE_DEPLOY,  // key for blue/green
      },
      // Register in BLUE target group (green is managed by CodeDeploy)
      loadBalancers: [{
        targetGroup: blueTG,
        containerName: 'api',
        containerPort: 3000,
      }],
      vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS },
    });

    // ─── Hook Lambda (AfterAllowTestTraffic) ────────────────────────────────

    const hookFn = new lambda.Function(this, 'TestHook', {
      runtime: lambda.Runtime.PYTHON_3_12,
      handler: 'index.handler',
      timeout: Duration.seconds(300),
      code: lambda.Code.fromInline(`
import boto3
import urllib.request
import json
import os

codedeploy = boto3.client('codedeploy')
ALB_TEST_URL = os.environ.get('ALB_TEST_URL', '')

def handler(event, context):
    deployment_id = event['DeploymentId']
    hook_id = event['LifecycleEventHookExecutionId']
    
    try:
        # Call health endpoint via test listener (:8080)
        req = urllib.request.urlopen(f'{ALB_TEST_URL}/health', timeout=10)
        body = json.loads(req.read())
        
        if body.get('status') != 'ok':
            raise ValueError(f"Health check failed: {body}")
        
        print("Smoke test passed. Continuing deployment.")
        codedeploy.put_lifecycle_event_hook_execution_status(
            deploymentId=deployment_id,
            lifecycleEventHookExecutionId=hook_id,
            status='Succeeded'
        )
    except Exception as e:
        print(f"Smoke test FAILED: {e}")
        codedeploy.put_lifecycle_event_hook_execution_status(
            deploymentId=deployment_id,
            lifecycleEventHookExecutionId=hook_id,
            status='Failed'
        )
`),
      environment: {
        ALB_TEST_URL: `http://${alb.loadBalancerDnsName}:8080`,
      },
    });

    // The Lambda needs to call codedeploy:PutLifecycleEventHookExecutionStatus
    hookFn.addToRolePolicy(new iam.PolicyStatement({
      actions: ['codedeploy:PutLifecycleEventHookExecutionStatus'],
      resources: ['*'],
    }));

    // ─── CloudWatch Alarms for automatic rollback ──────────────────────────

    const errorRateAlarm = new cloudwatch.Alarm(this, 'HighErrorRate', {
      metric: new cloudwatch.MathExpression({
        expression: '(errors / requests) * 100',
        usingMetrics: {
          errors: new cloudwatch.Metric({
            namespace: 'AWS/ApplicationELB',
            metricName: 'HTTPCode_Target_5XX_Count',
            dimensionsMap: {
              LoadBalancer: alb.loadBalancerFullName,
              TargetGroup: blueTG.targetGroupFullName,
            },
            statistic: 'Sum',
          }),
          requests: new cloudwatch.Metric({
            namespace: 'AWS/ApplicationELB',
            metricName: 'RequestCount',
            dimensionsMap: {
              LoadBalancer: alb.loadBalancerFullName,
              TargetGroup: blueTG.targetGroupFullName,
            },
            statistic: 'Sum',
          }),
        },
        period: Duration.minutes(2),
      }),
      threshold: 5,           // rollback if 5xx > 5%
      evaluationPeriods: 2,
      comparisonOperator: cloudwatch.ComparisonOperator.GREATER_THAN_THRESHOLD,
      alarmName: 'api-HighErrorRate',
    });

    // ─── CodeDeploy Application and Deployment Group ───────────────────────

    const app = new codedeploy.EcsApplication(this, 'App', {
      applicationName: 'api-service',
    });

    const deploymentGroup = new codedeploy.EcsDeploymentGroup(this, 'DG', {
      application: app,
      deploymentGroupName: 'api-service-dg',
      service,
      blueGreenDeploymentConfig: {
        // Listeners and target groups for blue/green
        listener: prodListener,
        testListener,
        blueTargetGroup: blueTG,
        greenTargetGroup: greenTG,
        // Wait 60 min before destroying the blue task set
        terminationWaitTime: Duration.minutes(60),
      },
      deploymentConfig: codedeploy.EcsDeploymentConfig.CANARY_10PERCENT_5MINUTES,
      autoRollback: {
        failedDeployment: true,       // rollback if hook fails
        deploymentInAlarm: true,      // rollback if alarm triggers
        stoppedDeployment: false,
      },
      alarms: [errorRateAlarm],
    });
  }
}
```

### Generated AppSpec (for pipeline reference)

```yaml
# appspec.yaml — generated by the pipeline, replacing <TASK_DEF_ARN>
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: "<TASK_DEF_ARN>"
        LoadBalancerInfo:
          ContainerName: "api"
          ContainerPort: 3000

Hooks:
  - AfterAllowTestTraffic:
      - location: "arn:aws:lambda:us-east-1:123456789012:function:BlueGreenStack-TestHook"
        timeout: 300
```

### Timeline of a successful deployment

```
t=0m    CodeDeploy creates green task set with new task definition
t=2m    Green tasks pass TG health check
t=2m    Listener :8080 points to green TG (test traffic available)
t=2m    AfterAllowTestTraffic hook is invoked
t=3m    Hook validates /health via :8080 → Succeeded
t=3m    BeforeAllowTraffic (no hook configured, proceeds)
t=3m    Traffic shift: :443 goes to 10% green / 90% blue
t=8m    No alarm triggered during observation period (5 min)
t=8m    Traffic shift: :443 goes to 100% green / 0% blue
t=8m    AfterAllowTraffic hook invoked (if configured)
t=8m    Deployment marked as Succeeded
t=68m   terminationWaitTime elapsed → blue task set destroyed
```

---

## Common Pitfalls

### Pitfall 1: Hook Lambda without calling PutLifecycleEventHookExecutionStatus

**The error:** You create a hook Lambda that executes tests but returns normally (no exception). The Lambda finishes successfully (exit 200), but CodeDeploy doesn't receive confirmation via `PutLifecycleEventHookExecutionStatus`. After the configured `timeout` (e.g., 300 seconds), CodeDeploy considers the hook as `Failed` and initiates rollback. The deployment is reverted even with all tests passing.

**Why it happens:** CodeDeploy doesn't use the Lambda's exit code to determine hook success/failure. It *exclusively* waits for the explicit call to the `PutLifecycleEventHookExecutionStatus` API. Returning from the Lambda without calling this API is indistinguishable from a timeout for CodeDeploy.

**How to recognize:** In the CodeDeploy console, the deployment shows `Lifecycle event hook timed out` or `Lifecycle event hook failed`. Lambda logs show normal execution with no errors. The Lambda was invoked, executed, and returned — but the deployment reverted.

**How to avoid:** Use a `try/finally` block to ensure `PutLifecycleEventHookExecutionStatus` is always called, even in case of exception. Never return from the Lambda without having called this API.

---

### Pitfall 2: terminationWait zero — no post-deploy rollback window

**The error:** `terminationWaitTime` is configured as 0 (default). A successful deploy destroys the blue task set immediately after the traffic shift. Thirty minutes later, a latent problem appears in the new version (e.g., a batch job that runs every hour starts failing). To revert, you need to create a new deployment with the previous version — which takes time and may have downtime.

**Why it happens:** With `terminationWaitTime = 0`, CodeDeploy destroys the blue task set as soon as the traffic shift is complete. There's no blue to instantly revert to.

**How to avoid:** Configure `terminationWaitTime` of 30-60 minutes for critical services. During this period, a rollback is literally an ALB redirection operation — seconds, without creating new tasks. The cost is maintaining double the tasks for that period, which is acceptable for important services. If you want to save, 30 minutes is sufficient for most post-deploy problems to become visible.

---

### Pitfall 3: Alarm configured on the wrong target group causes false positives

**The error:** You configure the rollback alarm monitoring metrics from `blueTG` (original target group). During deployment, CodeDeploy is sending 10% of traffic to `greenTG` and 90% to `blueTG`. If `blueTG` had a high error rate *before* the deployment (which motivated deploying the new version), the alarm is already active and CodeDeploy immediately reverts the just-started deployment.

**Why it happens:** The alarm monitors the wrong target group. During canary traffic shift, what you want to monitor are errors on the *new* target group (green), not the old one (blue).

**How to recognize:** Deployments that revert immediately upon starting the traffic shift, even when the new version is working correctly. Deployment group logs show `Alarm entered ALARM state` as the rollback reason.

**How to avoid:** Configure rollback alarms to monitor `greenTG` (or the ALB as a whole, which includes both). Alternatively, monitor business metrics (e.g., conversion rate, P99 latency) that reflect service health regardless of which target group is active.

---

## Reflection Exercise

You're migrating a payments service from rolling update to blue/green. The service has a characteristic: the new version includes a database schema migration that is compatible with both code versions (old and new run against the new schema). The deployment typically takes 3 minutes to complete the traffic shift.

How would you structure the hooks? Which lifecycle event (`BeforeAllowTraffic`, `AfterAllowTestTraffic`, etc.) would be most appropriate for each type of validation — health check verification, payment flow test via test listener, and post-shift error rate monitoring? What `terminationWaitTime` would you choose, considering that database schema rollback is not trivial? And if a high latency alarm triggered 45 minutes after the successful deployment (after blue has already been destroyed) — what would be the rollback procedure in that case?

---

## Resources for Further Study

### 1. CodeDeploy blue/green deployments for Amazon ECS
**URL:** https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-type-bluegreen.html
**What to find:** Complete overview of the `CODE_DEPLOY` deployment type for ECS: required components, deployment flow, differences from rolling update, and limitations (e.g., doesn't support Capacity Provider with base > 0 for the green task set in all configurations).
**Why it's the right source:** It's the official entry point for understanding the complete model.

### 2. AppSpec 'hooks' section for ECS
**URL:** https://docs.aws.amazon.com/codedeploy/latest/userguide/reference-appspec-file-structure-hooks.html
**What to find:** Complete reference of all lifecycle hooks available for ECS, execution order, how the hook Lambda is invoked (payload), and the format of the `PutLifecycleEventHookExecutionStatus` call.
**Why it's the right source:** It's the technical reference that defines exactly the protocol between CodeDeploy and the hook Lambda.

### 3. Working with deployment configurations in CodeDeploy
**URL:** https://docs.aws.amazon.com/codedeploy/latest/userguide/deployment-configurations.html
**What to find:** Complete list of predefined configurations for ECS (Canary, Linear, AllAtOnce), how to create custom configurations, and how traffic shifting works in each mode.
**Why it's the right source:** It's the reference for choosing or creating the correct traffic shifting strategy for each risk context.

---
