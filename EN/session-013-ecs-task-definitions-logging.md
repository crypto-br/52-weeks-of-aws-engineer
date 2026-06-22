# Session 13 — ECS: Task Definitions — containers, volumes, logging, and resource limits

**Estimated duration:** 60 minutes
**Prerequisites:** session-004-cdk-v2-setup-bootstrap

---

## Objective

By the end, you will be able to write a complete task definition with multiple containers (sidecar pattern), configure the awslogs driver for CloudWatch, use EFS volumes mounted in multiple containers, and understand the CPU/memory limits per task vs per container.

---

## Context

[FACT] The **Task Definition** is the immutable blueprint of a task in ECS — the equivalent of a pod spec in Kubernetes. It defines which containers run together, which image each one uses, how much CPU and memory each receives, how logs are routed, which volumes are mounted, and which IAM role the task assumes. Each time you modify a task definition, ECS creates a new **revision** (immutable) with an incremented number. You deploy a specific revision, never edit an existing one.

[FACT] Containers in the same task definition share the same network lifecycle: in Fargate mode, each task has its own ENI with a private IP, and containers in the task communicate via `localhost`. This is what makes the **sidecar pattern** efficient in ECS — the sidecar doesn't need to cross the network to talk to the main container, it uses loopback.

[CONSENSUS] The ECS task definition model is deliberately simpler than the Kubernetes pod spec. It doesn't have concepts like complex dependency init containers, declarative readiness/liveness probes at the task level (only load balancer health checks), or resource requests distinct from resource limits. This simplicity is both an advantage (lower learning curve) and a limitation (less fine-grained control in advanced cases).

---

## Key Concepts

### 1. Task Definition structure

A task definition has two configuration levels: the **task level** (settings that apply to the task as a whole) and the **container level** (settings specific to each container within the task).

```
Task Definition (immutable revision)
│
├── taskRoleArn        → IAM role that the code inside containers uses
├── executionRoleArn   → IAM role that the ECS agent uses for image pull and logs
├── networkMode        → awsvpc | bridge | host (Fargate: awsvpc only)
├── cpu                → Total task CPU (required in Fargate)
├── memory             → Total task memory (required in Fargate)
├── volumes            → EFS volume definitions, bind mounts
│
└── containerDefinitions[]
    ├── [0] app container
    │   ├── image, cpu, memory, memoryReservation
    │   ├── portMappings
    │   ├── environment, secrets
    │   ├── logConfiguration (awslogs driver)
    │   ├── mountPoints (reference to volumes)
    │   ├── healthCheck
    │   └── essential: true
    │
    └── [1] sidecar container
        ├── image, cpu, memory
        ├── logConfiguration
        ├── mountPoints
        └── essential: false   ← task continues if the sidecar fails
```

[FACT] Two IAM fields that confuse beginners:

- **`taskRoleArn`** (Task Role): IAM identity of the *processes inside the containers*. If your app needs to call S3, DynamoDB, or any AWS service, this role needs the permissions.
- **`executionRoleArn`** (Execution Role): IAM identity of the *ECS agent*, which runs outside the containers. It's used to pull the image from ECR (`ecr:GetAuthorizationToken`, `ecr:BatchGetImage`) and to send logs to CloudWatch (`logs:CreateLogStream`, `logs:PutLogEvents`). Without this role, the container won't even start.

---

### 2. CPU and Memory: task level vs container level

[FACT] In Fargate, CPU and memory **must** be declared at the task level. Fargate provisions capacity based on these values. At the container level, the values are optional and have different semantics:

```
Task level (required in Fargate):
  cpu: 1024    → 1 vCPU allocated for the ENTIRE task
  memory: 2048 → 2 GB allocated for the ENTIRE task

Container level (optional):
  memory: 1536         → hard limit: the container is killed by the OOM killer if
                         exceeded. Must be ≤ task memory.
  memoryReservation: 768  → soft limit: reservation guarantee, but can use more
                            if memory is available in the task.
  cpu: 512             → relative CPU weight (CPU shares), not a hard limit.
```

[FACT] The table of valid CPU × Memory combinations in Fargate (in CPU units and MiB):

| CPU (units) | vCPU | Valid Memory (MiB) |
|---|---|---|
| 256 | 0.25 | 512, 1024, 2048 |
| 512 | 0.5 | 1024–4096 (1024 increments) |
| 1024 | 1 | 2048–8192 (1024 increments) |
| 2048 | 2 | 4096–16384 (1024 increments) |
| 4096 | 4 | 8192–30720 (1024 increments) |
| 8192 | 8 | 16384–61440 (4096 increments) |
| 16384 | 16 | 32768–122880 (8192 increments) |

[FACT] The `cpu` at the container level in Fargate is treated as **CPU shares** (relative weight), not as a hard allocation. If one container is idle, another can use the unused CPU. This is different from the EC2 launch type, where container CPU is literally reserved on the host.

[CONSENSUS] AWS's recommendation for Fargate is: define `cpu` and `memory` at the task level, and use `memoryReservation` (soft limit) at the container level only if you need fine-grained control over how memory is divided between containers. Using `memory` (hard limit) on the container without understanding the OOM killer is a frequent pitfall — we'll see this in the pitfalls section.

---

### 3. awslogs driver: routing logs to CloudWatch

[FACT] The `awslogs` driver is ECS's native mechanism for sending container stdout/stderr to CloudWatch Logs. It's configured in the `logConfiguration` field of each container definition:

```json
{
  "logConfiguration": {
    "logDriver": "awslogs",
    "options": {
      "awslogs-group": "/ecs/my-app",
      "awslogs-region": "us-east-1",
      "awslogs-stream-prefix": "app",
      "awslogs-create-group": "true",
      "mode": "non-blocking",
      "max-buffer-size": "25m"
    }
  }
}
```

[FACT] Each container within a task generates a **separate log stream** in the CloudWatch Log Group, with the name in the format:

```
{awslogs-stream-prefix}/{container-name}/{ecs-task-id}
```

For example: `app/my-app-container/a1b2c3d4-5678-90ab-cdef-1234567890ab`

[FACT] The `mode: non-blocking` option deserves special attention. By default, the `awslogs` driver operates in **blocking** mode: if CloudWatch Logs is unavailable or slow, the container stops writing to stdout/stderr until the driver can send. In high-load applications, this can cause unexplainable stalls. With `mode: non-blocking` and `max-buffer-size`, the driver uses an in-memory buffer and discards logs if the buffer fills up, but never blocks the container.

```
Blocking mode (default):
  Container → awslogs driver → CloudWatch
             ↑ blocks here if CWL is slow

Non-blocking mode:
  Container → in-memory buffer → awslogs driver → CloudWatch
             ↑ never blocks       ↑ discards if buffer full
```

[FACT] The Execution Role needs specific permissions for the awslogs driver to work:

```json
{
  "Effect": "Allow",
  "Action": [
    "logs:CreateLogGroup",
    "logs:CreateLogStream",
    "logs:PutLogEvents",
    "logs:DescribeLogStreams"
  ],
  "Resource": "arn:aws:logs:*:*:*"
}
```

The `logs:CreateLogGroup` permission is only needed when `awslogs-create-group: true`. In production, create the Log Group with a retention policy via IaC and remove this permission from the Execution Role.

---

### 4. Sidecar pattern in ECS

[FACT] The sidecar pattern places an auxiliary container in the same task to separate responsibilities without code coupling. In ECS, the sidecar shares the same network (loopback) and can share volumes with the main container.

Common sidecar use cases in ECS:

```
┌─────────────────────────────────────────────────────────┐
│  Task (shared network namespace: localhost)             │
│                                                         │
│  ┌──────────────────┐    ┌──────────────────────────┐  │
│  │  App container   │    │  Sidecar container        │  │
│  │  (essential:true)│    │  (essential:false)        │  │
│  │                  │    │                           │  │
│  │  app:8080        │◀───│  nginx reverse proxy      │  │
│  │                  │    │  (TLS termination):443    │  │
│  └──────────────────┘    └──────────────────────────┘  │
│                                                         │
│  ┌──────────────────┐    ┌──────────────────────────┐  │
│  │  App container   │    │  CloudWatch Agent sidecar │  │
│  │  writing         │    │  reading metrics via      │  │
│  │  EMF metrics     │───▶│  UDP:25888                │  │
│  │  to UDP          │    │  and sending to CWL       │  │
│  └──────────────────┘    └──────────────────────────┘  │
│                                                         │
│  ┌──────────────────┐    ┌──────────────────────────┐  │
│  │  App container   │    │  AWS X-Ray daemon         │  │
│  │  sending traces  │    │  collecting traces via    │  │
│  │  to UDP:2000     │───▶│  UDP:2000                 │  │
│  └────────┬─────────┘    └──────────────────────────┘  │
│           │                                             │
│      shared EFS                                         │
│      volume                                             │
└─────────────────────────────────────────────────────────┘
```

[FACT] The `essential` field controls behavior when a container stops:

- `essential: true` → if this container stops (any exit code), the entire task is stopped and all other containers are terminated.
- `essential: false` → if this container stops, the task continues running. Useful for metrics collection or initialization sidecars.

[FACT] The `dependsOn` field allows defining startup order between containers:

```json
{
  "name": "app",
  "dependsOn": [
    {
      "containerName": "xray-daemon",
      "condition": "START"   // waits for xray to start before bringing up app
    }
  ]
}
```

Available conditions: `START` (container started), `COMPLETE` (container finished successfully — useful for init containers), `SUCCESS` (finished with exit code 0), `HEALTHY` (passed health check).

---

### 5. EFS Volumes in Task Definitions

[FACT] EFS volumes in task definitions have two components: the **volume definition** at the task level (which EFS file systems and access points to use) and the **mount points** at the container level (where to mount each volume inside the container).

```json
{
  "volumes": [
    {
      "name": "shared-data",
      "efsVolumeConfiguration": {
        "fileSystemId": "fs-0abc1234",
        "rootDirectory": "/",
        "transitEncryption": "ENABLED",
        "authorizationConfig": {
          "accessPointId": "fsap-0abc1234",
          "iam": "ENABLED"
        }
      }
    }
  ],
  "containerDefinitions": [
    {
      "name": "writer",
      "mountPoints": [
        {
          "sourceVolume": "shared-data",
          "containerPath": "/data/output",
          "readOnly": false
        }
      ]
    },
    {
      "name": "reader",
      "mountPoints": [
        {
          "sourceVolume": "shared-data",
          "containerPath": "/data/input",
          "readOnly": true
        }
      ]
    }
  ]
}
```

[FACT] For Fargate to mount an EFS volume, three requirements must be met:

1. **Platform version 1.4.0 or higher** (on Fargate Linux). Version 1.4.0 added EFS support.
2. **Compatible security groups**: the task's security group must have outbound access on port 2049 (NFS) to the EFS mount target's security group.
3. **Task Role with EFS permission**: if `iam: ENABLED` in `authorizationConfig`, the Task Role needs `elasticfilesystem:ClientMount` (and `ClientWrite` for writing).

[FACT] Ephemeral storage in Fargate: by default, each Fargate task has 20 GB of ephemeral storage (for container images + temporary data). You can increase it up to 200 GB via `ephemeralStorage.sizeInGiB`. This storage is destroyed when the task stops — it doesn't persist.

---

## Practical Example

**Scenario:** A Node.js API that needs:
- An AWS X-Ray daemon sidecar for distributed tracing.
- Structured logs sent to CloudWatch via `awslogs` in non-blocking mode.
- An EFS volume mounted to store user uploads persistently.

### Task Definition in JSON (ECS RegisterTaskDefinition format)

```json
{
  "family": "api-service",
  "taskRoleArn": "arn:aws:iam::123456789012:role/api-task-role",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecs-execution-role",
  "networkMode": "awsvpc",
  "cpu": "512",
  "memory": "1024",
  "requiresCompatibilities": ["FARGATE"],
  "volumes": [
    {
      "name": "user-uploads",
      "efsVolumeConfiguration": {
        "fileSystemId": "fs-0abc1234",
        "transitEncryption": "ENABLED",
        "authorizationConfig": {
          "accessPointId": "fsap-0abc1234",
          "iam": "ENABLED"
        }
      }
    }
  ],
  "containerDefinitions": [
    {
      "name": "api",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/api:v1.2.3",
      "essential": true,
      "cpu": 384,
      "memoryReservation": 768,
      "portMappings": [
        { "containerPort": 3000, "protocol": "tcp" }
      ],
      "environment": [
        { "name": "NODE_ENV", "value": "production" },
        { "name": "AWS_XRAY_DAEMON_ADDRESS", "value": "localhost:2000" }
      ],
      "secrets": [
        {
          "name": "DATABASE_URL",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:prod/db-url"
        }
      ],
      "mountPoints": [
        {
          "sourceVolume": "user-uploads",
          "containerPath": "/app/uploads",
          "readOnly": false
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/api-service",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "api",
          "mode": "non-blocking",
          "max-buffer-size": "25m"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:3000/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      },
      "dependsOn": [
        { "containerName": "xray-daemon", "condition": "START" }
      ]
    },
    {
      "name": "xray-daemon",
      "image": "public.ecr.aws/xray/aws-xray-daemon:latest",
      "essential": false,
      "cpu": 64,
      "memoryReservation": 128,
      "portMappings": [
        { "containerPort": 2000, "protocol": "udp" }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/api-service",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "xray",
          "mode": "non-blocking",
          "max-buffer-size": "5m"
        }
      }
    }
  ]
}
```

### CDK Equivalent (TypeScript)

```typescript
import { Stack, StackProps, Duration, RemovalPolicy } from 'aws-cdk-lib';
import * as ecs from 'aws-cdk-lib/aws-ecs';
import * as iam from 'aws-cdk-lib/aws-iam';
import * as logs from 'aws-cdk-lib/aws-logs';
import * as secretsmanager from 'aws-cdk-lib/aws-secretsmanager';
import * as efs from 'aws-cdk-lib/aws-efs';
import { Construct } from 'constructs';

export class ApiTaskStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    // Log Group with explicit retention (avoids accumulating logs indefinitely)
    const logGroup = new logs.LogGroup(this, 'LogGroup', {
      logGroupName: '/ecs/api-service',
      retention: logs.RetentionDays.ONE_MONTH,
      removalPolicy: RemovalPolicy.DESTROY,
    });

    // Task Role: permissions for the application code
    const taskRole = new iam.Role(this, 'TaskRole', {
      assumedBy: new iam.ServicePrincipal('ecs-tasks.amazonaws.com'),
    });
    taskRole.addManagedPolicy(
      iam.ManagedPolicy.fromAwsManagedPolicyName('AWSXRayDaemonWriteAccess')
    );
    // taskRole.addToPolicy(new iam.PolicyStatement({ ... s3, dynamodb, etc. }));

    // Task Definition
    const taskDef = new ecs.FargateTaskDefinition(this, 'TaskDef', {
      cpu: 512,
      memoryLimitMiB: 1024,
      taskRole,
      // executionRole is automatically created by CDK with minimal permissions
    });

    // EFS Volume
    const fileSystem = efs.FileSystem.fromFileSystemAttributes(this, 'EFS', {
      fileSystemId: 'fs-0abc1234',
      securityGroup: /* EFS sg */ undefined as any,
    });
    const accessPoint = new efs.AccessPoint(this, 'AccessPoint', {
      fileSystem,
      path: '/uploads',
      posixUser: { uid: '1000', gid: '1000' },
      createAcl: { ownerUid: '1000', ownerGid: '1000', permissions: '755' },
    });

    // Add volume to task definition
    taskDef.addVolume({
      name: 'user-uploads',
      efsVolumeConfiguration: {
        fileSystemId: fileSystem.fileSystemId,
        transitEncryption: 'ENABLED',
        authorizationConfig: {
          accessPointId: accessPoint.accessPointId,
          iam: 'ENABLED',
        },
      },
    });

    // X-Ray Sidecar
    const xrayContainer = taskDef.addContainer('xray-daemon', {
      image: ecs.ContainerImage.fromRegistry('public.ecr.aws/xray/aws-xray-daemon:latest'),
      cpu: 64,
      memoryReservationMiB: 128,
      essential: false,
      logging: ecs.LogDrivers.awsLogs({
        streamPrefix: 'xray',
        logGroup,
        mode: ecs.AwsLogDriverMode.NON_BLOCKING,
        maxBufferSize: cdk.Size.mebibytes(5),
      }),
    });
    xrayContainer.addPortMappings({ containerPort: 2000, protocol: ecs.Protocol.UDP });

    // Main container
    const appContainer = taskDef.addContainer('api', {
      image: ecs.ContainerImage.fromEcrRepository(/* repo */ undefined as any, 'v1.2.3'),
      cpu: 384,
      memoryReservationMiB: 768,
      essential: true,
      environment: {
        NODE_ENV: 'production',
        AWS_XRAY_DAEMON_ADDRESS: 'localhost:2000',
      },
      secrets: {
        DATABASE_URL: ecs.Secret.fromSecretsManager(
          secretsmanager.Secret.fromSecretNameV2(this, 'DbSecret', 'prod/db-url')
        ),
      },
      logging: ecs.LogDrivers.awsLogs({
        streamPrefix: 'api',
        logGroup,
        mode: ecs.AwsLogDriverMode.NON_BLOCKING,
        maxBufferSize: cdk.Size.mebibytes(25),
      }),
      healthCheck: {
        command: ['CMD-SHELL', 'curl -f http://localhost:3000/health || exit 1'],
        interval: Duration.seconds(30),
        timeout: Duration.seconds(5),
        retries: 3,
        startPeriod: Duration.seconds(60),
      },
    });
    appContainer.addPortMappings({ containerPort: 3000 });
    appContainer.addMountPoints({
      sourceVolume: 'user-uploads',
      containerPath: '/app/uploads',
      readOnly: false,
    });
    appContainer.addContainerDependencies({
      container: xrayContainer,
      condition: ecs.ContainerDependencyCondition.START,
    });

    // Allow the task role to access EFS
    fileSystem.grantRootAccess(taskRole);
  }
}
```

---

## Common Pitfalls

### Pitfall 1: Confusing Task Role with Execution Role

**The error:** The application needs to read a secret from Secrets Manager. You add the `secretsmanager:GetSecretValue` permission to the **Execution Role** instead of the **Task Role**. The container starts normally (the Execution Role has what it needs for image pull and logs), but when the code tries to call `GetSecretValue`, it receives `AccessDenied`.

**Why it happens:** The Execution Role is used by the ECS agent **before** the container starts (to pull the image and set up logs). The Task Role is injected as credentials inside the container via the metadata endpoint (169.254.170.2). They are two completely different execution contexts.

**How to recognize:** `AccessDenied` errors on AWS SDK calls inside the application code, while the container starts without problems. The ECS startup log shows no errors.

**How to avoid:** Mnemonic rule: "Execution Role = what ECS needs to make the container born. Task Role = what the code inside the container needs to do its work."

---

### Pitfall 2: Silent OOM kill due to hard limit on the container

**The error:** You define `memory: 512` (hard limit) for the main container in a task with `memory: 1024`. Under high load, the Node.js process exceeds 512 MiB. The Linux OOM killer kills the process. ECS registers the container as terminated with `OOMKilled: true` in the event, but the task definition marks the container as `essential: true`, so the entire task is stopped and immediately restarted. The result is a loop of silent crashes that appears to the user as "unstable service".

**Why it happens:** The container memory hard limit (`memory`) is enforced by the Linux kernel via cgroups. When the process exceeds the limit, it's killed without warning. ECS automatically restarts the task, creating the loop.

**How to recognize:** `StoppedReason: Essential container in task exited` events in the ECS console with frequency. CloudWatch Metrics showing MemoryUtilization spikes followed by abrupt drops to zero. `aws ecs describe-tasks` showing `exitCode: 137` (signal 9 = SIGKILL from OOM killer).

**How to avoid:** Prefer `memoryReservation` (soft limit) instead of `memory` (hard limit) at the container level. Use CloudWatch Container Insights metrics to discover the P99 memory usage and size the hard limit with a 20-30% safety margin.

---

### Pitfall 3: awslogs blocking the container in default (blocking) mode

**The error:** Your ECS service experiences CloudWatch Logs latency (overloaded region, brief outage). The `awslogs` driver in blocking mode stops all containers trying to write to stdout. The application hangs on `console.log()` or equivalent. The ALB health check fails. ECS replaces the task. The log of the problem disappears because the driver couldn't send it.

**Why it happens:** The `awslogs` driver in default (blocking) mode applies backpressure to the process when it can't send to CloudWatch. It's a correct throughput behavior, but catastrophic for availability.

**How to avoid:** Always configure `mode: non-blocking` and `max-buffer-size: 25m` (or a value appropriate for your workload) in all production containers. Accept that in case of CloudWatch unavailability you may lose some logs — this is preferable to losing availability.

---

## Reflection Exercise

You're designing a task definition for an image processing service: a main container that receives images, processes them with FFmpeg (CPU-intensive), and writes results to an EFS volume. A second container reads the processed results and uploads them to S3, acting as an export sidecar. The task runs on Fargate.

How would you divide resources (CPU and memory) between the two containers and at the task level? Would the export sidecar be `essential: true` or `false`? Would you use `dependsOn` and with which condition? How would you configure logs for each container (same Log Group or separate groups)? And if the FFmpeg process is unstable and frequently aborts with exit code 1 — which health check field or dependency would you use to detect this before ECS marks the task as unhealthy?

---

## Resources for Further Study

### 1. Amazon ECS task definition parameters (Fargate)
**URL:** https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definition_parameters.html
**What to find:** Complete reference of all task definition fields for Fargate: container definitions, volumes, network mode, resource limits. This is the document you consult when you're unsure about a specific field.
**Why it's the right source:** It's AWS's canonical reference — more precise than any tutorial.

### 2. Using the awslogs log driver
**URL:** https://docs.aws.amazon.com/AmazonECS/latest/developerguide/using_awslogs.html
**What to find:** Complete configuration of the `awslogs` driver, including non-blocking mode options, multi-line log patterns for capturing stack traces in a single entry, and the IAM permissions needed on the Execution Role.
**Why it's the right source:** Documents all driver options, including the less obvious ones like `awslogs-multiline-pattern` which is essential for Java/Python stack traces.

### 3. ECS Best Practices Guide
**URL:** https://docs.aws.amazon.com/AmazonECS/latest/bestpracticesguide/intro.html
**What to find:** Separate chapters on task sizing, logging patterns, sidecar patterns, and persistent volume usage. It's more prescriptive than the developer guide — it says *what to do*, not just *how it works*.
**Why it's the right source:** It's the official best practices guide from the ECS team, consolidating lessons from users in production, covering trade-offs the developer guide doesn't explicitly discuss.

---
