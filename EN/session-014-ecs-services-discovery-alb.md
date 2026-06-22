# Session 14 — ECS: Services, service discovery, and ALB Target Group integration

**Estimated duration:** 60 minutes
**Prerequisites:** session-013-ecs-task-definitions-logging

---

## Objective

By the end, you will be able to create an ECS Service that registers tasks in an ALB Target Group, configure health checks with grace period, use AWS Cloud Map for service-to-service discovery (without going through the load balancer), and understand when to use each approach.

---

## Context

[FACT] A **Task Definition** describes *what* to run. An **ECS Service** describes *how* to run it: how many replicas to maintain, how to do rolling updates, where to register tasks to receive traffic, and what to do when a task fails. The Service is ECS's self-healing mechanism — it continuously monitors the number of healthy tasks and creates new ones if the count drops below the `desiredCount`.

[FACT] ECS supports two types of integration for receiving traffic: **ALB/NLB Target Groups** for external traffic (and internal traffic going through the load balancer) and **AWS Cloud Map** for DNS-based service-to-service discovery without a load balancer. They are not mutually exclusive — a service can have both simultaneously.

[CONSENSUS] The choice between ALB and Cloud Map for internal communication is a trade-off between operability and efficiency. ALB offers TLS termination, path/header routing, circuit breaking, and native observability via access logs. Cloud Map is simpler and more efficient for high-frequency service-to-service communication where load balancer overhead matters and where dynamic discovery of multiple instances is needed without the indirection of a VIP.

---

## Key Concepts

### 1. Anatomy of an ECS Service

```
ECS Service
│
├── taskDefinition        → which revision to deploy
├── desiredCount          → how many tasks to keep running
├── launchType            → FARGATE | EC2 | EXTERNAL
│
├── deploymentConfiguration
│   ├── minimumHealthyPercent  → minimum % of healthy tasks during deploy
│   ├── maximumPercent         → maximum % of tasks during deploy
│   └── deploymentCircuitBreaker
│       ├── enable: true/false
│       └── rollback: true/false
│
├── loadBalancers[]
│   └── { targetGroupArn, containerName, containerPort }
│
├── serviceRegistries[]
│   └── { registryArn }   → Cloud Map service ARN
│
├── healthCheckGracePeriodSeconds  → ignore health checks for N seconds after start
│
└── networkConfiguration
    └── awsvpcConfiguration
        ├── subnets[]
        ├── securityGroups[]
        └── assignPublicIp: ENABLED | DISABLED
```

[FACT] The Service scheduler is a continuous loop. On each cycle, it:
1. Counts tasks in the `RUNNING` state that passed the health check.
2. If the count is less than `desiredCount`, starts new tasks.
3. If a task fails (exit code, OOM kill, health check failed), stops the task and starts a new one.
4. During a deploy (new task definition revision), executes the rolling update respecting `minimumHealthyPercent` and `maximumPercent`.

---

### 2. Rolling update: minimumHealthyPercent and maximumPercent

[FACT] The two deployment configuration parameters define the capacity envelope during a rolling update:

```
desiredCount = 4 tasks

Default values (REPLICA service):
  minimumHealthyPercent = 100%  → minimum: ceil(4 × 1.0) = 4 healthy tasks
  maximumPercent        = 200%  → maximum: floor(4 × 2.0) = 8 total tasks

Behavior with defaults:
  1. ECS starts 4 new tasks (new revision) → total: 8 tasks
  2. Waits for the 4 new ones to pass health check
  3. Stops the 4 old tasks
  4. Result: zero downtime, but momentarily doubled capacity

Aggressive values (faster deploy, lower cost):
  minimumHealthyPercent = 50%   → minimum: ceil(4 × 0.5) = 2 healthy tasks
  maximumPercent        = 150%  → maximum: floor(4 × 1.5) = 6 tasks

Aggressive behavior:
  1. Stops 2 old tasks → total: 2 tasks (50% capacity)
  2. Starts 4 new tasks → total: 6 tasks
  3. Waits for new ones to pass health check
  4. Stops the 2 remaining old tasks
  5. Result: temporary 50% capacity reduction during deploy
```

[FACT] For services with `desiredCount = 1` (common in development), the defaults `min=100%, max=200%` are the only ones that guarantee zero downtime: ECS starts the new task before stopping the old one. With `min=0%, max=100%`, the deploy stops the old task first, causing downtime.

---

### 3. Deployment Circuit Breaker

[FACT] The circuit breaker detects when a deploy is continuously failing and optionally rolls back automatically to the previous revision. It uses a sliding window of failed tasks:

```
Failure threshold = max(10, desiredCount × 2)
  → minimum: 3 failures   (documentation mentions minimum of 3 in this context)
  → maximum: 200 failures

If the count of failing tasks exceeds the threshold → circuit opens
  → if rollback=true: ECS reverts to the last successful revision
  → if rollback=false: deploy stops, service is in FAILED state
```

[CONSENSUS] In production, `deploymentCircuitBreaker: { enable: true, rollback: true }` is considered best practice. Without a circuit breaker, a deploy with a broken image (that crashes immediately) can loop indefinitely — ECS keeps trying to start tasks that fail, paying for the cost of each attempt and preventing the service from recovering.

---

### 4. Integration with ALB Target Groups

[FACT] The integration between ECS Service and ALB happens via **Target Group** in IP mode. In Fargate with `networkMode: awsvpc`, each task has its own private IP. ECS automatically registers and deregisters these IPs in the Target Group as tasks come up and go down.

```
Internet / VPC
      │
      ▼
┌──────────────┐
│     ALB      │  Listener: HTTPS:443, HTTP:80
│              │
│  Listener    │──rule: path /* ──▶ Target Group (type: IP)
└──────────────┘                        │
                                        │ registers/deregisters automatically
                              ┌─────────┴──────────┐
                              │  Task 1: 10.0.1.5  │
                              │  Task 2: 10.0.1.8  │  ← port 3000
                              │  Task 3: 10.0.1.12 │
                              └────────────────────┘
```

[FACT] When removing a task from circulation (during rolling update or scale-in), ECS performs **connection draining** (also called deregistration delay): sends the deregistration to the Target Group, waits for the ALB to finish active connections, and only then stops the container. The default deregistration delay is 300 seconds — configurable on the Target Group.

**Health Check Grace Period** is distinct from the Target Group's health check:

```
healthCheckGracePeriodSeconds (on the Service):
  → how long ECS IGNORES ALB health check failures after the task starts
  → prevents tasks in bootstrap from being killed prematurely
  → value zero: ECS acts immediately if the ALB marks the task as unhealthy

Health Check on the Target Group (on the ALB):
  → interval, threshold, path — defines when the ALB considers a task healthy
  → independent of the ECS grace period
```

[FACT] A classic error: the application takes 45 seconds to initialize (downloading configs, warming cache). The default `healthCheckGracePeriodSeconds` is zero. The ALB health check fails in the first 45 seconds. ECS kills the task. The new task also takes 45 seconds. Infinite loop. Solution: `healthCheckGracePeriodSeconds` should be slightly greater than the container health check's `startPeriod` + the application startup time.

---

### 5. AWS Cloud Map: DNS service discovery without load balancer

[FACT] AWS Cloud Map is a resource registration and discovery service. In the ECS context, it allows services to discover each other via DNS without going through a load balancer — the client resolves a DNS name and directly obtains the IPs of running tasks.

**Cloud Map components:**

```
Private DNS Namespace: internal.svc
│
├── Service: "api"
│   ├── DNS records: A records → IPs of api service tasks
│   └── Health check: synchronized with ECS health check
│
└── Service: "worker"
    ├── DNS records: A records → IPs of worker service tasks
    └── Health check: synchronized with ECS health check

DNS query: api.internal.svc → [10.0.1.5, 10.0.1.8, 10.0.1.12]
```

[FACT] When an ECS Service is configured with Cloud Map (`serviceRegistries`), ECS automatically registers an **instance** in Cloud Map for each task that passes the health check. When the task stops or fails, ECS removes the registration. The DNS TTL determines how long clients will cache stale IPs after a task is removed.

[FACT] Cloud Map supports two DNS record types for ECS:

| Record type | When to use | Returns |
|---|---|---|
| **A record** | `networkMode: awsvpc` (Fargate or EC2) | Task private IP |
| **SRV record** | `networkMode: bridge` or `host` (EC2) | IP + container port |

In Fargate (always `awsvpc`), use A records. SRV records are only needed in EC2 with bridge mode, where the host IP is the same for multiple tasks and the port is dynamic.

---

### 6. ALB vs Cloud Map: when to use each approach

```
                    ALB Target Group          AWS Cloud Map
                    ────────────────          ─────────────
Latency             +20-40ms (extra hop)      Zero overhead (DNS)
Cost                ~$18/month + LCUs         ~$1/month + queries
TLS termination     Yes (ACM integrated)      No (app manages)
Advanced routing    Yes (path, header, host)  No
Native metrics      Request count, latency    No (needs X-Ray)
Circuit breaking    Yes (5xx % via listener)  No
Multiple services   Yes (rules per path)      One DNS name per service
Port discovery      No (fixed port in TG)     SRV records (bridge mode)
```

[CONSENSUS] The practical rule adopted by most teams:

- **Use ALB** for: external traffic ingress, public APIs, any service that needs terminated TLS, path routing, or when you want request-level metrics at the application level.
- **Use Cloud Map** for: high-frequency service-to-service communication within the VPC where ALB latency matters, background services that don't receive external traffic, or when you need the client to know the IPs of all replicas (e.g., for client-side load balancing in gRPC).
- **Use both** for: services that receive external traffic via ALB AND are called internally by other services with low latency via Cloud Map.

---

## Practical Example

**Scenario:** A system with two services:
- `api-service`: receives external HTTPS traffic via ALB, exposes `/api/*`.
- `worker-service`: background processing, called only by `api-service` internally via Cloud Map.

### Architecture

```
Internet
    │ HTTPS:443
    ▼
┌───────────────────────────────────┐
│  ALB: api.example.com             │
│  Listener rule: /* → Target Group │
└───────────────┬───────────────────┘
                │ registers IPs
                ▼
     ┌─────────────────────┐
     │  api-service (ECS)  │──── Cloud Map DNS lookup
     │  3 tasks Fargate    │     worker.internal.svc
     │  port 3000          │────────────────▶ ┌─────────────────────┐
     └─────────────────────┘                  │ worker-service (ECS)│
                                              │ 2 tasks Fargate     │
                                              │ port 8080           │
                                              └─────────────────────┘
```

### CDK (TypeScript) — complete stack

```typescript
import { Stack, StackProps, Duration } from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as ecs from 'aws-cdk-lib/aws-ecs';
import * as elbv2 from 'aws-cdk-lib/aws-elasticloadbalancingv2';
import * as servicediscovery from 'aws-cdk-lib/aws-servicediscovery';
import { Construct } from 'constructs';

export class AppStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    const vpc = ec2.Vpc.fromLookup(this, 'Vpc', { vpcName: 'prod-vpc' });

    // ECS Cluster
    const cluster = new ecs.Cluster(this, 'Cluster', {
      vpc,
      containerInsights: true,   // enables detailed per-container metrics
    });

    // Cloud Map private DNS namespace
    const namespace = new servicediscovery.PrivateDnsNamespace(this, 'Namespace', {
      name: 'internal.svc',
      vpc,
    });

    // ─── Worker Service ───────────────────────────────────────────────────

    const workerTaskDef = new ecs.FargateTaskDefinition(this, 'WorkerTaskDef', {
      cpu: 256,
      memoryLimitMiB: 512,
    });
    workerTaskDef.addContainer('worker', {
      image: ecs.ContainerImage.fromRegistry('my-org/worker:latest'),
      portMappings: [{ containerPort: 8080 }],
      logging: ecs.LogDrivers.awsLogs({ streamPrefix: 'worker' }),
    });

    const workerSg = new ec2.SecurityGroup(this, 'WorkerSg', { vpc });

    const workerService = new ecs.FargateService(this, 'WorkerService', {
      cluster,
      taskDefinition: workerTaskDef,
      desiredCount: 2,
      securityGroups: [workerSg],
      // No load balancer — discovery only via Cloud Map
      cloudMapOptions: {
        name: 'worker',                     // DNS: worker.internal.svc
        cloudMapNamespace: namespace,
        dnsRecordType: servicediscovery.DnsRecordType.A,
        dnsTtl: Duration.seconds(10),       // Low TTL: failures detected quickly
      },
      deploymentCircuitBreaker: { rollback: true },
      minHealthyPercent: 50,
      maxHealthyPercent: 200,
    });

    // ─── API Service + ALB ────────────────────────────────────────────────

    const apiTaskDef = new ecs.FargateTaskDefinition(this, 'ApiTaskDef', {
      cpu: 512,
      memoryLimitMiB: 1024,
    });
    apiTaskDef.addContainer('api', {
      image: ecs.ContainerImage.fromRegistry('my-org/api:latest'),
      portMappings: [{ containerPort: 3000 }],
      logging: ecs.LogDrivers.awsLogs({ streamPrefix: 'api' }),
      environment: {
        // The api service discovers worker via Cloud Map
        WORKER_URL: 'http://worker.internal.svc:8080',
      },
    });

    // ALB
    const alb = new elbv2.ApplicationLoadBalancer(this, 'ALB', {
      vpc,
      internetFacing: true,
    });

    const listener = alb.addListener('HttpsListener', {
      port: 443,
      // certificates: [cert],  // ACM certificate in real production
      open: true,
    });

    const apiSg = new ec2.SecurityGroup(this, 'ApiSg', { vpc });

    const apiService = new ecs.FargateService(this, 'ApiService', {
      cluster,
      taskDefinition: apiTaskDef,
      desiredCount: 3,
      securityGroups: [apiSg],
      deploymentCircuitBreaker: { rollback: true },
      minHealthyPercent: 100,
      maxHealthyPercent: 200,
      // Grace period: app takes ~30s to initialize
      healthCheckGracePeriod: Duration.seconds(60),
    });

    // Register the service in the ALB Target Group
    apiService.registerLoadBalancerTargets({
      containerName: 'api',
      containerPort: 3000,
      newTargetGroupId: 'ApiTG',
      listener: ecs.ListenerConfig.applicationListener(listener, {
        protocol: elbv2.ApplicationProtocol.HTTP,
        healthCheck: {
          path: '/health',
          interval: Duration.seconds(15),
          healthyThresholdCount: 2,
          unhealthyThresholdCount: 3,
          timeout: Duration.seconds(5),
        },
        deregistrationDelay: Duration.seconds(30),  // reduced from 300s default
      }),
    });

    // API needs to talk to worker via port 8080
    workerSg.addIngressRule(apiSg, ec2.Port.tcp(8080), 'API to Worker');
  }
}
```

---

## Common Pitfalls

### Pitfall 1: healthCheckGracePeriodSeconds zero with slow-starting application

**The error:** The service has `healthCheckGracePeriodSeconds` at the default value (zero). The application takes 45 seconds to start (loads configurations, establishes database connections). The ALB health check starts immediately with a 30-second interval. On the first check (30s after start), the app isn't ready yet → returns 503. ECS marks the task as unhealthy and replaces it. The new task also takes 45 seconds. The service can never come up.

**Why it happens:** The `healthCheckGracePeriodSeconds` was designed exactly for this scenario. Without it, ECS treats initial health check failures the same way as failures on an already-stabilized task.

**How to recognize:** In the ECS console, the service's Events tab shows `service X: task Y is unhealthy in target-group Z` repeatedly, with tasks being replaced before reaching 60 seconds of life. In the ALB, access logs show only health check requests returning 5xx.

**How to avoid:** Measure your application's startup time under real conditions (not localhost). Configure `healthCheckGracePeriodSeconds` = startup time + 20s margin. For Java applications with JVM warmup or Python with ML model loading, this can be 120-180 seconds.

---

### Pitfall 2: High DNS TTL in Cloud Map causing calls to dead tasks

**The error:** You configure `dnsTtl: Duration.seconds(60)` in Cloud Map (or use the default, which is 60 seconds). During a rolling update of `worker-service`, a task is removed from Cloud Map. The `api-service` still has the dead task's IP in DNS cache for up to 60 seconds and keeps trying to connect to it. Calls fail with `connection refused` during this period.

**Why it happens:** DNS is a distributed cache. The TTL defines how long resolvers (and the JVM, and Node.js) keep the value. If the TTL is 60s, a client that just resolved the name can keep using the IP for up to 60s after it's removed from Cloud Map.

**How to recognize:** `ECONNREFUSED` or `connection reset` errors in `api-service` lasting 30-60 seconds during deployments of `worker-service`, even with health check configured.

**How to avoid:** Use `dnsTtl: Duration.seconds(10)` for services that do frequent rolling updates. Lower values reduce propagation time but increase DNS query volume (minimal cost in Cloud Map, generally acceptable). Additionally, implement retry with backoff on the client to absorb the transition period.

---

### Pitfall 3: High deregistration delay causing deploy slowdowns

**The error:** The Target Group has the default deregistration delay of 300 seconds. During a rolling update of a service with `desiredCount=4`, ECS needs to stop 4 old tasks. For each task, ECS waits 300 seconds of connection draining before stopping it. A complete rolling update takes: 4 tasks × 300 seconds = 20 minutes, even though the new version has been 100% healthy for 19 minutes.

**Why it happens:** 300 seconds was the default chosen by AWS for workloads with long-lived connections (WebSockets, streaming). For REST APIs with short connections (< 1 second), it's an excessive value.

**How to recognize:** Deploys that take much longer than expected. The ECS console shows tasks in `DEREGISTERING` state for long periods. The Target Group panel shows targets in `draining` state.

**How to avoid:** Configure `deregistrationDelay` on the Target Group according to the connection type. For typical REST APIs, 15-30 seconds is sufficient. For WebSockets, keep it at 300s or more. You can configure this via CDK in the `HealthCheck` when registering the Target Group.

---

## Reflection Exercise

You're designing a microservices system with 5 services in ECS Fargate:
- `gateway`: receives external traffic (HTTPS via ALB)
- `auth`: verifies JWT tokens, called by `gateway` for each request
- `orders`: business logic, called by `gateway`
- `inventory`: queried by `orders` to check stock
- `notifications`: sends emails/SMS, triggered by `orders` asynchronously via SQS (not called directly)

For which services would you use ALB? For which would you use Cloud Map? Does `notifications` need either of these discovery mechanisms? What would be the impact of a 60-second DNS TTL on the `auth` service, given that it's called on every gateway request? How would you configure `healthCheckGracePeriodSeconds` for each service, knowing that `auth` uses a lightweight Node.js image (initialization in 2 seconds) and `orders` uses a Java Spring Boot app (initialization in 40 seconds)?

---

## Resources for Further Study

### 1. Amazon ECS service definition parameters
**URL:** https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service_definition_parameters.html
**What to find:** Complete reference of all ECS Service parameters: deployment configuration, load balancer integration, service registries, network configuration. This is the base document when you need to understand the exact meaning of a field.
**Why it's the right source:** It's the official reference — more precise than the console or tutorials.

### 2. Deployment circuit breaker
**URL:** https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-circuit-breaker.html
**What to find:** How the circuit breaker detects deploy failures, activation thresholds, rollback behavior, and limitations (only works with rolling update, not with blue/green).
**Why it's the right source:** Documents the detection algorithm — essential for understanding when the circuit breaker will and won't protect you.

### 3. Use service discovery to connect Amazon ECS services with DNS names
**URL:** https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-discovery.html
**What to find:** How to configure Cloud Map with ECS, difference between A and SRV records, health check synchronization between ECS and Cloud Map, and limitations (maximum 1000 instances per Cloud Map service).
**Why it's the right source:** It's the official ECS + Cloud Map integration guide, with configuration examples via console, CLI, and CloudFormation.

---
